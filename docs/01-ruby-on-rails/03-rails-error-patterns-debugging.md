# 03 — Rails Error Patterns & Production Debugging

> **Context**: Real-world triage, post-mortem playbooks, and production debugging tools for Senior/Staff Rails Engineers.

---

## 1. The Anatomy of Common Production Errors & Fixes

### 1.1 `PG::UniqueViolation` & The Model Validation Race Condition
**The Myth**: `validates :email, uniqueness: true` prevents duplicate accounts.
**The Reality**: Rails validation runs an initial `SELECT 1 FROM users WHERE email = 'test@example.com' LIMIT 1`. If two concurrent requests arrive simultaneously (e.g., rapid double-click on registration form), both `SELECT` checks see zero records, both proceed to `INSERT`, and one crashes with `PG::UniqueViolation` (or worse, corrupts data if no DB constraint exists).

```
Thread A: SELECT 1 FROM users WHERE email = 'x' -> Empty
Thread B: SELECT 1 FROM users WHERE email = 'x' -> Empty
Thread A: INSERT INTO users (email) VALUES ('x') -> SUCCEEDS
Thread B: INSERT INTO users (email) VALUES ('x') -> CRASH (PG::UniqueViolation)
```

#### The Production Fix: Database Unique Constraint + Upsert / Rescue
```ruby
# 1. db/migrate/20260909000002_add_unique_index_to_users_email.rb
class AddUniqueIndexToUsersEmail < ActiveRecord::Migration[7.1]
  disable_ddl_transaction!

  def change
    add_index :users, :email, unique: true, algorithm: :concurrently
  end
end

# 2. app/services/user_registration_service.rb
class UserRegistrationService
  def self.call(email:, attributes:)
    User.create!(attributes.merge(email: email))
  rescue ActiveRecord::RecordNotUnique, PG::UniqueViolation
    # Return existing record or raise clean domain error
    User.find_by!(email: email)
  end
end
```

---

### 1.2 Database Deadlocks (`ActiveRecord::Deadlocked`)
**The Cause**: Two concurrent transactions update the same resources in opposite orders.
```
Transaction 1: Locks Account A -> Waits to acquire lock on Account B
Transaction 2: Locks Account B -> Waits to acquire lock on Account A
PostgreSQL Engine: Detects circular dependency after `deadlock_timeout` (1s) -> Aborts Transaction 2
```

#### The Fix: Strict Global Lock Ordering
Always acquire resource locks in a deterministic order (e.g., sorted by Primary Key ID).

```ruby
# app/services/transfer_service.rb
class TransferService
  def self.execute(from_account_id, to_account_id, amount)
    # Enforce strict ID sorting to guarantee lock acquisition order
    first_id, second_id = [from_account_id, to_account_id].sort

    Account.transaction do
      # Locks will ALWAYS be acquired in ascending ID order!
      first_acc  = Account.lock.find(first_id)
      second_acc = Account.lock.find(second_id)

      from_acc = (first_acc.id == from_account_id) ? first_acc : second_acc
      to_acc   = (first_acc.id == to_account_id)   ? first_acc : second_acc

      raise InsufficientBalance if from_acc.balance < amount

      from_acc.update!(balance: from_acc.balance - amount)
      to_acc.update!(balance: to_acc.balance + amount)
    end
  rescue ActiveRecord::Deadlocked
    # Automated retry with exponential jitter for unexpected edge cases
    retry_count ||= 0
    if (retry_count += 1) <= 3
      sleep(rand(0.05..0.2) * retry_count)
      retry
    else
      raise
    end
  end
end
```

---

### 1.3 `ActiveRecord::ConnectionTimeoutError`
**The Symptom**: Requests spike to 5000ms latency, followed by ALB 502/504 errors. Logs show:
`could not obtain a connection from the pool within 5.000 seconds (waited 5.002s); all 5 connections are in use`

#### Troubleshooting & Resolution Checklist:
1. **Check Pool Size vs Threads**: Verify that `pool` in `config/database.yml` equals or exceeds `RAILS_MAX_THREADS`.
   ```yaml
   production:
     adapter: postgresql
     pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>
     checkout_timeout: 5.0
   ```
2. **Sidekiq Connection Leak**: If a Sidekiq job creates internal threads or calls external APIs while holding an active DB connection, the pool starves:
   ```ruby
   # ANTI-PATTERN:
   Account.transaction do
     account = Account.lock.find(1)
     # CRITICAL BUG: Network call takes 10s while holding PostgreSQL row lock & connection!
     response = HTTParty.post("https://payment-provider.com/charge")
     account.update!(charged: true)
   end

   # CORRECT PATTERN:
   # Do HTTP call OUTSIDE the database transaction!
   response = HTTParty.post("https://payment-provider.com/charge")
   if response.success?
     Account.transaction do
       Account.lock.find(1).update!(charged: true)
     end
   end
   ```

---

### 1.4 Memory Bloat from Large CSV/Report Exports
**The Symptom**: Worker memory jumps from 300MB to 2.5GB during an end-of-month CSV export, triggering Linux kernel OOM killer (`Out of memory: Kill process (puma)`).

#### Solution: Streaming HTTP Responses with `ActionController::Live`
```ruby
# app/controllers/reports_controller.rb
class ReportsController < ApplicationController
  include ActionController::Live

  def export_transactions
    response.headers['Content-Type'] = 'text/csv'
    response.headers['Content-Disposition'] = 'attachment; filename="transactions.csv"'
    response.headers['X-Accel-Buffering'] = 'no' # Disable NGINX proxy buffering!

    writer = response.stream
    writer.write "ID,Amount,Status,Created At\n"

    # Stream chunks of 1000 records without buffering all in memory
    Transaction.where(created_at: 1.month.ago..Time.current)
               .find_each(batch_size: 1000) do |tx|
      writer.write "#{tx.id},#{tx.amount},#{tx.status},#{tx.created_at.iso8601}\n"
    end
  ensure
    response.stream.close # Must always close the stream!
  end
end
```

---

## 2. Production Debugging & Profiling Tooling

### 2.1 Live Process CPU Profiling with `rbspy`
`rbspy` is a non-invasive sampling profiler for Ruby that attaches to any running Ruby process via OS `process_vm_readv` system calls. It has **near-zero overhead** and requires no code modifications or gem dependencies.

```bash
# 1. Identify high-CPU Puma worker PID
ps aux | grep puma | grep -v cluster

# 2. Record 30 seconds of call stack execution from PID 4821
sudo rbspy record --pid 4821 --duration 30 --file /tmp/puma_flamegraph.svg

# 3. Inspect top hot methods in real-time terminal summary
sudo rbspy top --pid 4821
```

```
% self   % total   name
 34.20     34.20   ActiveRecord::ConnectionAdapters::PostgreSQL::Type::DateTime#cast
 22.15     56.35   ActiveSupport::JSON::Encoding::JSONGemEncoder#encode
 15.10     71.45   Psych::Parser#parse
```
*Actionable Insight*: High time in `DateTime#cast` indicates un-indexed date parsing or over-fetching redundant timestamp columns in large query loops.

---

### 2.2 Heap Allocation Analysis with `memory_profiler`
```ruby
# In rails console or isolated staging benchmark
require 'memory_profiler'

report = MemoryProfiler.report do
  100.times do
    DashboardMetricsService.call(tenant_id: 42)
  end
end

# Find retained objects (memory that never gets garbage collected)
report.pretty_print(
  to_file: "tmp/dashboard_leak.txt",
  scale_bytes: true,
  normalize_paths: true
)
```

---

### 2.3 Production Safe Debugging: `rails console --sandbox`
When debugging live data issues on production instances:
```bash
# Starts Rails console with an automatic rollback on exit!
RAILS_ENV=production bundle exec rails console --sandbox

# Output:
# Any modifications you make will be rolled back on exit!
# irb(main):001:0> u = User.first
# irb(main):002:0> u.update!(status: 'active')
# irb(main):003:0> exit
# Rolling back transaction!
```

---

## 3. Senior Interview Q&A Cheatsheet

### Q1: "How do you systematically triage a sudden spike in Puma 502 Bad Gateway errors?"
> **Answer**:
> 1. **Check NGINX Error Log**: Look for `connect() to unix:/var/run/puma.sock failed (11: Resource temporarily unavailable)` or `upstream prematurely closed connection`.
> 2. **Check Puma Backlog Queue**: A 502 typically means the Puma socket listen backlog is completely full because all worker threads are locked or saturated.
> 3. **Inspect Active DB Queries**: Run `SELECT pid, now() - query_start AS duration, query, state FROM pg_stat_activity WHERE state != 'idle' ORDER BY duration DESC;` to see if an un-indexed query or lock contention is holding Puma threads hostage.
> 4. **Check Out-Of-Memory (OOM)**: Run `dmesg -T | grep -i oom` to see if the Linux kernel killed Puma worker processes due to memory exhaustion.

### Q2: "Why is `dependent: :destroy` dangerous on high-volume associations, and what is the alternative?"
> **Answer**: `dependent: :destroy` instantiates every child record in memory and executes their callbacks one by one (`N` queries + `N` instantiations). For an account with 100,000 transactions, destroying the account will attempt to load 100,000 models, blowing up Ruby heap memory and locking the database for seconds.
> **Alternatives**:
> 1. `dependent: :delete_all`: Issues a single direct SQL query `DELETE FROM transactions WHERE account_id = ?` (bypasses model instantiation and callbacks).
> 2. **Database Foreign Key Cascades**: `add_foreign_key :transactions, :accounts, on_delete: :cascade` handles deletion entirely inside the database engine at C-level speed without application involvement.
> 3. **Soft Deletion / Async Purging**: Mark record `deleted_at: Time.current` and enqueue a background Sidekiq batch worker to delete rows in small batches during off-peak hours.
