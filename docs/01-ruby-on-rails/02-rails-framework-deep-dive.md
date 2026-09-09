# 02 — Rails Framework Deep Dive (Zero to Senior)

> **Context**: Primary Backend Engine (5+ years production experience in FinTech & SaaS). Covers ORM internals, zero-downtime database patterns, background job architecture, and production deployment.

---

## 1. MVC Request Lifecycle & RESTful Routing

### 1.1 Request Lifecycle Flow
```
Browser / Client
      │ (HTTP Request)
      ▼
NGINX Reverse Proxy (Slow-client buffering)
      │ (Unix Socket / TCP)
      ▼
Puma Web Server (Reactor thread -> Worker process -> Thread pool)
      │ (Rack ENV Hash)
      ▼
Rack Middleware Stack (ActionDispatch, Cookies, SSL, Rack::Attack)
      │
      ▼
ActionDispatch::Routing::RouteSet (Matches URI & HTTP verb to Controller#Action)
      │
      ▼
ApplicationController (Callbacks: before_action, Authentication, Pundit auth)
      │
      ▼
Controller Action (Orchestrates Services, Models, DB queries)
      │
      ▼
ActiveRecord Models (PG query via ConnectionPool -> Serializes to Blueprinter/JSON)
      │
      ▼
ActionController::Metal#render (Serializes payload, sets Headers & Status code)
      │
      ▼
Rack Response Array: [status, headers_hash, [body_string]]
```

### 1.2 Senior Routing Patterns
```ruby
# config/routes.rb
Rails.application.routes.draw do
  namespace :api, defaults: { format: :json } do
    namespace :v1 do
      # Nested resources with shallow scoping to prevent deep URL anti-patterns
      resources :organizations, only: %i[index show create update] do
        resources :projects, shallow: true do
          resources :tasks, only: %i[index create]
        end
      end

      # Non-RESTful RPC member/collection routes
      resources :payments, only: %i[create show] do
        member do
          post :capture
          post :refund
        end
        collection do
          get :reconciliation_summary
        end
      end
    end
  end

  # Health check endpoint bypassing sessions & heavy middleware
  get "/up" => "rails/health#show", as: :rails_health_check
end
```

---

## 2. ActiveRecord Internals & Database Mastery

### 2.1 The ORM Pattern & Heap Allocation Realities
Every row loaded via `User.all` or `User.where(...)` instantiates an instance of `User < ActiveRecord::Base`.
- An instantiated ActiveRecord object consumes **1.5 KB to 3.5 KB of heap memory** (attribute hashes, dirty tracking mutation maps, type-casting wrappers).
- Loading 50,000 records into memory will allocate ~150 MB of Ruby heap objects and trigger major GC pauses.

#### `pluck` vs. `select` Allocation Profiling
- `User.select(:id, :email)`: Executes `SELECT id, email FROM users`, then allocates 50,000 `User` objects on the heap.
- `User.pluck(:id, :email)`: Bypasses ActiveRecord model instantiation entirely. The PG adapter extracts raw values into lightweight native Ruby arrays `[[1, "a@b.com"], ...]`.

```ruby
# Benchmark: Pluck vs Select Memory Overhead
require 'benchmark/ips'

# Assuming 10,000 user rows in DB
Benchmark.ips do |x|
  x.report("ActiveRecord select") do
    User.limit(10_000).select(:id, :email).to_a
  end

  x.report("ActiveRecord pluck") do
    User.limit(10_000).pluck(:id, :email)
  end

  x.compare!
end
# Result: `pluck` is 8x-12x faster and consumes 92% less memory.
```

### 2.2 Memory Bloat Prevention with `find_each`
Never call `.all.each` or `.where(...).each` on unbounded tables in production.
- `find_each(batch_size: 1000)`: Uses primary-key cursor pagination (`WHERE id > last_seen_id ORDER BY id ASC LIMIT 1000`).
- Loads 1,000 records, processes them, and allows the Young Generation GC to sweep the previous batch before allocating the next batch.

```ruby
# Production Data Migration / Batch Export
def export_inactive_accounts
  CSV.open("tmp/inactive_accounts.csv", "wb") do |csv|
    csv << ["ID", "Email", "Last Login"]

    # Prevents OOM by bounding memory consumption to 1,000 models
    User.where("last_sign_in_at < ?", 1.year.ago)
        .find_each(batch_size: 1000) do |user|
      csv << [user.id, user.email, user.last_sign_in_at]
    end
  end
end
```

### 2.3 N+1 Queries: Mechanical Dissection of Solutions
Given: `users = User.limit(100)` and `users.each { |u| u.profile.bio }`. This triggers 1 query for users, and 100 queries for profiles ($1 + N$).

| Method | SQL Execution Strategy | When to Choose |
|---|---|---|
| `preload(:profile)` | Executes 2 separate queries: `SELECT * FROM users` + `SELECT * FROM profiles WHERE user_id IN (...)` | Best for simple 1:1 and 1:N associations where you do NOT filter by the association in the `WHERE` clause. |
| `includes(:profile)` | Defaults to `preload` (2 queries). If `.where("profiles.bio = ?", ...)` is added, automatically falls back to `eager_load` (`LEFT OUTER JOIN`). | General-purpose eager loading. |
| `eager_load(:profile)` | Executes 1 single massive query with `LEFT OUTER JOIN` | Necessary when filtering or ordering by columns in the joined association. Higher memory consumption due to duplicate row Cartesian products. |
| `joins(:profile)` | Executes an `INNER JOIN`. Does **not** load association models into memory. | Use when filtering records based on child table attributes, but not accessing child fields in views. |

---

## 3. High-Scale Patterns & 1-Billion-Row Migration Protocol

### 3.1 The 1-Billion-Row Zero-Downtime Migration Pattern
**The Problem**: Running `ALTER TABLE users ADD COLUMN status VARCHAR DEFAULT 'active'` or `ADD INDEX` directly on a table with 100M+ rows locks the table (`AccessExclusiveLock`), queuing all subsequent web requests until connection pools exhaust and cause a SEV-1 outage.

**The 5-Step Zero-Downtime Protocol**:

```
Step 1: Add column as NULLABLE without default value (locks for < 1ms)
   │
Step 2: Add Dual-Write application logic (write to both old and new columns)
   │
Step 3: Asynchronous batched backfill via Sidekiq (1,000 rows/batch with 100ms sleep)
   │
Step 4: Switch read path in application to the new column
   │
Step 5: Add NOT NULL constraint with NOT VALID, followed by VALIDATE CONSTRAINT
```

```ruby
# db/migrate/20260909000001_safe_add_column_to_large_table.rb
class SafeAddColumnToLargeTable < ActiveRecord::Migration[7.1]
  disable_ddl_transaction! # Required for concurrent index creation!

  def up
    # Step 1: Add column without default (instantaneous metadata change in PG 11+)
    add_column :transactions, :settlement_status, :string

    # Step 2: Add index concurrently (does NOT lock writes!)
    add_index :transactions, :settlement_status, algorithm: :concurrently
  end

  def down
    remove_index :transactions, :settlement_status, algorithm: :concurrently
    remove_column :transactions, :settlement_status
  end
end
```

```ruby
# app/workers/backfill_settlement_status_worker.rb
class BackfillSettlementStatusWorker
  include Sidekiq::Worker
  sidekiq_options queue: :low_priority, retry: 5

  def perform(start_id, end_id)
    Transaction.where(id: start_id..end_id, settlement_status: nil)
               .update_all("settlement_status = status")
    
    # Sleep to allow PostgreSQL replication lag to drain
    sleep 0.1
  end
end
```

### 3.2 Concurrency Control: Optimistic vs. Pessimistic Locking

```ruby
# 1. Optimistic Locking (Collision is rare, high throughput)
# Requires integer column: `lock_version`
account = Account.find(1)
account.balance += 50
account.save! # Raises ActiveRecord::StaleObjectError if another thread updated it first!

# 2. Pessimistic Locking (Collision is frequent, financial transactions)
# Executes: SELECT * FROM accounts WHERE id = 1 FOR UPDATE
Account.transaction do
  account = Account.lock.find(1)
  raise InsufficientFunds if account.balance < withdrawal_amount

  account.balance -= withdrawal_amount
  account.save!

  AuditLog.create!(account_id: account.id, delta: -withdrawal_amount)
end
```

---

## 4. Background Processing: ActiveJob & Sidekiq Internals

### 4.1 Sidekiq Architecture
- **Engine**: Redis in-memory storage + threaded Ruby worker processes.
- **Data Structures**:
  - `queues`: Redis Sets containing queue names.
  - `queue:critical`: Redis List. Jobs pushed via `LPUSH`, pulled by workers via `BRPOP`.
  - `schedule` / `retry`: Redis Sorted Sets (`ZSET`) where score = Unix timestamp of scheduled execution time.
- **Thread Safety**: Workers share memory within the same process. Code executed in `perform` **must be thread-safe** (no mutating class variables `@@` or shared global state).

### 4.2 Idempotency & Unique Jobs Pattern
Network timeouts between Sidekiq and Redis or third-party payment gateways can cause jobs to re-run. Every job must be **strictly idempotent**.

```ruby
class CaptureStripePaymentWorker
  include Sidekiq::Worker
  sidekiq_options queue: :payments, retry: 10, dead: true

  def perform(payment_id, idempotency_key)
    payment = Payment.find(payment_id)
    return if payment.completed? # Fast exit if already processed

    # Acquire distributed lock or record processing attempt
    Payment.transaction do
      payment.lock!
      return if payment.completed?

      # Pass idempotency key directly to payment gateway
      charge = Stripe::PaymentIntent.capture(
        payment.external_transaction_id,
        {},
        { idempotency_key: idempotency_key }
      )

      payment.update!(status: :completed, captured_at: Time.current)
    end
  rescue Stripe::CardError => e
    # Non-retryable business error: Mark payment failed, do not let Sidekiq retry
    payment.update!(status: :failed, error_message: e.message)
  end
end
```

---

## 5. Modern Rails: Hotwire (Turbo + Stimulus) vs. React API Mode

### 5.1 Architecture Decision Matrix

| Feature | Hotwire (Turbo + Stimulus) | Rails API + Next.js / React |
|---|---|---|
| **Architecture** | Server-rendered HTML fragments over HTTP/WebSockets | Client-rendered SPA/SSR calling JSON REST/GraphQL APIs |
| **Team Velocity** | 3x faster for monolithic full-stack teams (single codebase) | Slower (2 repositories, schema sync, serialization overhead) |
| **Client Bundle Size** | ~40 KB (Turbo + Stimulus) | 200 KB - 1 MB+ JavaScript |
| **Mobile Integration** | Turbo Native (wraps web views with native chrome) | Native / Flutter / React Native via JSON APIs |
| **Optimal Use Case** | B2B SaaS, Admin dashboards, internal workflow tools | High-interactivity consumer apps, multi-platform mobile apps |

### 5.2 Turbo Streams in Action
```ruby
# app/controllers/comments_controller.rb
class CommentsController < ApplicationController
  def create
    @comment = @post.comments.create!(comment_params)

    respond_to do |format|
      format.turbo_stream # Renders create.turbo_stream.erb
      format.html { redirect_to @post }
    end
  end
end
```

```erb
<%# app/views/comments/create.turbo_stream.erb %>
<%= turbo_stream.prepend "comments_list", partial: "comments/comment", locals: { comment: @comment } %>
<%= turbo_stream.replace "new_comment_form", partial: "comments/form", locals: { post: @post, comment: Comment.new } %>
```

---

## 6. Authentication, Authorization & Security Best Practices

### 6.1 Pundit Policy Architecture
```ruby
# app/policies/document_policy.rb
class DocumentPolicy < ApplicationPolicy
  class Scope < Scope
    def resolve
      if user.admin?
        scope.all
      else
        # Multi-tenant isolation at policy query level
        scope.where(organization_id: user.organization_id)
      end
    end
  end

  def update?
    user.admin? || (record.organization_id == user.organization_id && record.author_id == user.id)
  end

  def destroy?
    user.admin?
  end
end
```

### 6.2 Security Vulnerabilities & Mitigations

| Vulnerability | Attack Vector | Rails Defense Mechanism |
|---|---|---|
| **Mass Assignment** | Malicious client adds `is_admin: true` to JSON payload | **Strong Parameters**: `params.require(:user).permit(:name, :email)` |
| **SQL Injection** | `User.where("name = '#{params[:name]}'")` | **Parameterized Queries**: `User.where("name = ?", params[:name])` or hash `User.where(name: params[:name])` |
| **XSS** | User inputs `<script>stealCookie()</script>` | ERB auto-escapes all output. **Never** use `raw()` or `html_safe` on untrusted user strings. |
| **CSRF** | Forged POST request from third-party malicious website | `protect_from_forgery with: :exception` validates `X-CSRF-Token` header. |

---

## 7. Production Architecture & The Cascading Timeout Rule

### 7.1 The Golden Rule of Stack Timeouts
In a distributed web system, **timeouts must decrease monotonically deeper into the stack**. If an upstream component has a shorter timeout than a downstream component, the upstream drops the connection while the database continues burning CPU executing dead work.

```
Client Browser Timeout: 30s
       │
AWS ALB (Load Balancer) Timeout: 60s
       │
NGINX `proxy_read_timeout`: 60s
       │
Puma Rack Timeout: 15s (rack-timeout / Puma worker killer)
       │
PostgreSQL `statement_timeout`: 10s
```
*Why this works*: If a catastrophic query runs, PostgreSQL terminates it at **10s**, returning an error through Puma and NGINX back to the client. The ALB never times out, worker threads are freed immediately, and cascading 502/504 storms are prevented.

### 7.2 Puma Thread & Worker Tuning Formula
```ruby
# config/puma.rb
# In Docker / ECS container with 2 vCPUs and 2GB RAM
workers Integer(ENV.fetch("WEB_CONCURRENCY") { 2 }) # 1 worker per physical CPU core
threads_count = Integer(ENV.fetch("RAILS_MAX_THREADS") { 5 }) # 5 threads per worker
threads threads_count, threads_count

preload_app! # Enables Copy-on-Write (CoW) memory savings across forked workers

# Database connection pool must equal or exceed total threads per worker!
# database.yml: pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>
```

---

## 8. Senior Interview Q&A Cheatsheet

### Q1: "Explain how PostgreSQL `includes` works and when it can actually hurt performance."
> **Answer**: `includes` defaults to issuing two separate queries (`preload`), which is efficient for 1:N relations. However, if you add an association condition in `.where()`, Rails forces a `LEFT OUTER JOIN` (`eager_load`). On large tables, a `LEFT OUTER JOIN` generates a Cartesian product of rows that the database must transfer and Rails must parse into models. If joining an order with 20 items, 20 duplicated order rows travel over the network. If you only need to filter without displaying association data, `joins` (`INNER JOIN`) is far superior as it avoids loading redundant child records into memory.

### Q2: "How do you detect and fix database connection pool exhaustion in Rails?"
> **Answer**: Connection pool exhaustion surfaces as `ActiveRecord::ConnectionTimeoutError: could not obtain a connection from the pool within 5.000 seconds`. It happens when Puma worker threads exceed `pool` size in `database.yml`, or when long-running Sidekiq threads hold open checked-out connections while waiting on external network calls. Fixes:
> 1. Set `database.yml` pool size $\ge$ `RAILS_MAX_THREADS`.
> 2. Ensure non-database long-running calls (e.g. external HTTP calls or file uploads) do not hold database connections open.
> 3. Implement PgBouncer in transaction pooling mode in front of PostgreSQL to multiplex thousands of Rails thread connections into a small, fixed pool of dedicated Postgres backend processes.
