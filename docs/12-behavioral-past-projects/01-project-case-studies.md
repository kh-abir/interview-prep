# 01 — Production Case Studies & Architectural Wins

> **Context**: Core Resume Stories & Behavioral Deep-Dives. Demonstrates high-stakes production leadership across FinTech, SaaS, MDM, and high-scale optimization.

---

## 1. Technonext: 10x Scaling with $0 Infrastructure Cost

### 1.1 The Situation & Metrics
- **Context**: B2B SaaS platform handling multi-enterprise organizational workflows.
- **The Outage**: At **1,000 concurrent active users**, PostgreSQL CPU spiked to 100%, query response times surged from 80ms to over 6,500ms, and ALB health check timeouts began killing application servers in cascading failure loops.
- **The Constraint**: Executive leadership denied requests for immediate vertical database scaling ($1,500/month RDS upgrade). The mandate: **Scale to 10,000+ users on the existing 2 vCPU / 4GB RAM database tier ($0 infra cost expansion)**.

### 1.2 Root Cause Analysis
Using `pg_stat_statements` and `EXPLAIN (ANALYZE, BUFFERS)`:
1. **Un-indexed Polymorphic Queries**: A critical dashboard audit query scanned 4.2 million rows via a `Seq Scan`, performing over **38,000 disk page reads** per request.
2. **PostgreSQL Connection Thrashing**: 300 Puma application worker threads connected directly to PostgreSQL. Each PostgreSQL backend process consumed ~8MB RAM, consuming 2.4GB RAM purely in connection memory and causing aggressive context-switching.
3. **Cache Invalidation Storms**: Cache TTL was set to static 60 seconds; when high-cardinality keys expired simultaneously, hundreds of identical expensive queries hit the database simultaneously (Thundering Herd).

### 1.3 Architectural Actions Taken
```
[ 10,000 Clients ]
        │
        ▼
[ AWS ALB ]
        │
        ▼
[ Puma Application Servers ] (300 threads)
        │
        ▼
[ PgBouncer ] (Transaction Pooling: 300 app conns ──► 25 dedicated Postgres server processes!)
        │
        ▼
[ PostgreSQL Primary ] (B-tree Indexes + Partial Indexes + FILLFACTOR 85)
        ▲
        │ (Cache-Aside with Jittered TTL)
[ Redis Cluster ]
```

1. **PgBouncer in Transaction Pooling Mode**:
   Multiplexed 300 application connections into **25 dedicated PostgreSQL server connections**. Dropped DB idle memory by 85% and eliminated OS connection fork overhead.
2. **Targeted Compound & Partial Indexing**:
   Replaced sequential table scans with index-only scans:
   ```sql
   CREATE INDEX CONCURRENTLY idx_org_audits_active 
   ON audit_logs (organization_id, created_at DESC) 
   WHERE status = 'ACTIVE';
   ```
   Query execution dropped from **4,200ms to 1.8ms** (99.9% latency reduction).
3. **Redis Cache-Aside with Probabilistic TTL Jitter**:
   Added randomized $\pm 15\%$ jitter to TTLs to eliminate simultaneous cache stampedes.
4. **Result**: Sustained **10,000+ concurrent active users** at **< 120ms p99 latency** on the exact same hardware tier, saving over $18,000/year in infrastructure costs.

---

## 2. Multi-Tenant Authorization Engine: 4-Layer Security

### 2.1 The 4-Layer Defense Architecture
To guarantee absolute tenant isolation across a shared database architecture without human developer error:

```
[ Layer 1: API Gateway Layer ] ──► Validates JWT signature & extracts Tenant UUID claim
              │
              ▼
[ Layer 2: Middleware Context ]  ──► Binds Tenant ID to Thread-Local / AsyncLocalStorage
              │
              ▼
[ Layer 3: Application Policy ]  ──► Pundit / CASL RBAC evaluation (Module & Action permissions)
              │
              ▼
[ Layer 4: PostgreSQL RLS ]      ──► Database engine enforces SET LOCAL app.current_tenant = X
```

### 2.2 PostgreSQL Row-Level Security (RLS) Implementation
Even if a junior engineer writes an un-scoped query `SELECT * FROM invoices` in an API endpoint, **PostgreSQL guarantees cross-organization data can NEVER be returned**:

```sql
-- Enable RLS on sensitive table
ALTER TABLE invoices ENABLE ROW LEVEL SECURITY;

-- Enforce tenant isolation policy
CREATE POLICY invoice_tenant_isolation_policy ON invoices
FOR ALL
USING (organization_id = NULLIF(current_setting('app.current_tenant', true), '')::uuid);
```

---

## 3. Gauntlet FinTech: Plaid + Stripe + Dwolla & Double-Entry Ledger

### 3.1 ACH Transfer & Verification Flow
```
User ──► Plaid Link ──► Exchanges public_token for processor_token (Dwolla/Stripe)
                          │
                          ▼
             Dwolla ACH Transfer Initiated (Idempotency Key)
                          │
                          ▼ Webhook: transfer:pending
             Pending Ledger Entry Created
                          │
                          ▼ Webhook: transfer:completed (Signature Verified)
             Funds Cleared -> Settled in Double-Entry Ledger
```

### 3.2 Double-Entry Bookkeeping Ledger
Financial systems cannot use naive `user.balance += 50` updates. Every monetary movement must be recorded as an immutable pair of equal and opposite debits and credits:

$$\sum \text{Debits} = \sum \text{Credits}$$

```ruby
# app/services/ledger_service.rb
class LedgerService
  def self.record_transfer!(from_account:, to_account:, amount_cents:, currency:, reference:)
    ActiveRecord::Base.transaction do
      entry = JournalEntry.create!(
        reference: reference,
        description: "ACH Transfer settlement",
        posted_at: Time.current
      )

      # 1. Credit the source account (liability/asset shift)
      entry.postings.create!(
        account: from_account,
        amount_cents: -amount_cents, # Debit/Credit polarity
        currency: currency
      )

      # 2. Debit the destination account
      entry.postings.create!(
        account: to_account,
        amount_cents: amount_cents,
        currency: currency
      )

      # Integrity assertion: Total entry sum MUST equal zero!
      raise CorruptedLedgerError unless entry.postings.sum(:amount_cents).zero?
    end
  end
end
```

### 3.3 Resilient Webhook Handling Protocol
1. **Signature Verification**: Validate HMAC-SHA256 signature against webhook secret before parsing body.
2. **Idempotency Store**: Insert webhook event ID into `processed_webhooks` table with a unique constraint. If duplicate arrives, return HTTP 200 immediately.
3. **Async Processing**: Return HTTP 200 within 200ms, pushing payload to Sidekiq/SQS to prevent third-party timeout retries.

---

## 4. Auro24 Apple Mobile Device Management (MDM)

### 4.1 Enterprise Architecture Cutover
- **Migration**: Monolithic legacy codebase migrated to a decoupled **Rails API Backend + Next.js App Router Frontend**.
- **Scale**: Onboarding **2,000+ enterprise Apple devices** (macOS, iOS, iPadOS).
- **Core Engineering**:
  - Implemented Apple MDM protocol handling **SCEP (Simple Certificate Enrollment Protocol)** payloads for device identity.
  - Automated device policy enforcement commands delivered via **Apple Push Notification service (APNs)** over HTTP/2.
  - Zero-downtime database cutover using dual-write and asynchronous Sidekiq backfills.

---

## 5. Senior Interview Q&A Cheatsheet

### Q1: "Describe a catastrophic production bug you diagnosed and resolved under pressure."
> **Answer**:
> "During peak traffic at Technonext, our application servers suddenly began spewing ALB 502 Bad Gateway errors. Initial monitoring showed PostgreSQL was responsive, but Puma threads were completely saturated.
> I attached `rbspy` to a live high-CPU worker and ran `netstat -nat | grep ESTABLISHED | wc -l`. We discovered an **AWS SNAT Port Exhaustion event**: a recently deployed microservice integration made outgoing HTTP requests to a partner API using a fresh TCP connection on every request without HTTP keep-alive. Under high concurrent traffic, the instance exhausted all 55,000 ephemeral outbound TCP ports, forcing all subsequent network calls to hang until timeout, queuing Puma worker threads.
> **Mitigation**: I immediately scaled the NAT Gateway, enabled HTTP keep-alive connection pooling using an `http.Agent` with persistent sockets, and added an egress circuit breaker. Response times returned to baseline within 4 minutes."

### Q2: "How do you resolve disagreements with non-technical founders on technical debt vs. new features?"
> **Answer**:
> "I never frame technical debt as an abstract aesthetic preference. I translate technical trade-offs into **financial cost, risk exposure, and team delivery velocity**.
> For example, when leadership wanted to add features instead of fixing database indexing, I showed that our database CPU was at 90%, and at current growth, the platform would crash during Black Friday, resulting in an estimated $40,000 revenue loss. I proposed an MVP compromise: 80% sprint time on feature delivery, and 20% dedicated to high-impact indexing and connection pooling that took 2 days but eliminated 95% of database load. Leaders respect data-driven risk management."
