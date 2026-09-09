# 03 — Observability, Telemetry & SEV-1 Incident Protocol

> **Context**: SRE & Staff Engineering Leadership (Handbook Ch 4 & 15). Covers OpenTelemetry distributed tracing, RED/USE metrics, SLO error budgets, and the SEV-1 Incident Commander protocol.

---

## 1. The Three Pillars of Observability

```
                       ┌───────────────────────────────┐
                       │  Unified Telemetry (OTel)     │
                       └──────────────┬────────────────┘
                                      │
           ┌──────────────────────────┼──────────────────────────┐
           ▼                          ▼                          ▼
     [ Traces ]                 [ Metrics ]                 [ Logs ]
- OpenTelemetry spans      - Prometheus Counters       - Structured JSON
- W3C Trace Context        - Gauges, Histograms        - Correlation IDs (trace_id)
- Distributed call graph   - Aggregated timeseries     - High cardinality events
```

### 1.1 OpenTelemetry Distributed Tracing & W3C Trace Context
In a microservices architecture, a single user request can touch 10 independent services. Without distributed tracing, isolating the specific service causing a 3-second latency spike is impossible.

#### W3C `traceparent` Header Format:
```
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
              │                  │                               │       │
           Version            Trace ID                        Span ID  Flags (01 = sampled)
```

```typescript
// OpenTelemetry Node.js Instrumentation Setup
import { NodeSDK } from '@opentelemetry/sdk-node';
import { getNodeAutoInstrumentations } from '@opentelemetry/auto-instrumentations-node';
import { OTLPTraceExporter } from '@opentelemetry/exporter-trace-otlp-http';

const sdk = new NodeSDK({
  traceExporter: new OTLPTraceExporter({
    url: 'http://otel-collector:4318/v1/traces',
  }),
  instrumentations: [
    getNodeAutoInstrumentations({
      '@opentelemetry/instrumentation-fs': { enabled: false }, // Avoid trace spam from internal file reads
    }),
  ],
});

sdk.start();
```

---

## 2. Monitoring Frameworks: RED vs. USE Methods

### 2.1 The RED Method (For Request-Driven Services)
Invented by Tom Wilkie. Evaluates software services from the user's perspective:
1. **Rate**: Number of requests per second receiving traffic.
2. **Errors**: Number of failed requests per second (HTTP 5xx, failed jobs).
3. **Duration**: Distribution of time requests take (p50, p95, p99 latency histograms).

### 2.2 The USE Method (For Physical / Infrastructure Resources)
Invented by Brendan Gregg. Evaluates hardware resources (CPU, Memory, Disk, Network):
1. **Utilization**: Percentage of time the resource was busy (e.g., CPU % busy, disk I/O time).
2. **Saturation**: Degree to which the resource has extra work queued that it cannot process (e.g., CPU run-queue length, disk queue depth).
3. **Errors**: Count of error events (e.g., network interface CRC packet errors, memory ECC errors).

---

## 3. Reliability Metrics: SLI, SLO, and SLA

```
SLI (What we measure)   ──► "99.2% of HTTP requests return in < 250ms"
SLO (Our internal goal) ──► "99.9% of HTTP requests must return in < 250ms over 30 days"
SLA (Legal contract)    ──► "99.5% uptime guaranteed, or 15% billing credit refunded"
Error Budget            ──► 100% - SLO = 0.1% allowable failure window
```

### Error Budget Policy:
If a service burns more than **50% of its monthly error budget in the first 7 days**, all feature deployments are frozen. Engineering focus is diverted 100% to reliability, bug fixes, and architectural hardening until the budget recovers.

---

## 4. Production Incident Management: The SEV-1 Protocol

### 4.1 Incident Severity Tiers
- **SEV-1 (Critical)**: Catastrophic production outage affecting all or core revenue paths (e.g. checkout completely down, data corruption). Response time: **< 5 minutes**, 24/7.
- **SEV-2 (Major)**: Significant feature degraded for high volume of users with no immediate workaround. Response time: **< 15 minutes**.
- **SEV-3 (Minor)**: Non-critical feature issue affecting small subset of users; workaround exists. Handled during normal business hours.

### 4.2 Incident Response Roles
```
                ┌───────────────────────────────────┐
                │   Incident Commander (IC)         │
                │ - Holds single decision authority │
                │ - Delegates triage tasks          │
                │ - Shields responders from noise   │
                └─────────────────┬─────────────────┘
                                  │
         ┌────────────────────────┴────────────────────────┐
         ▼                                                 ▼
┌─────────────────────────────────┐       ┌─────────────────────────────────┐
│     Operations / Tech Lead      │       │     Communications Lead         │
│ - Hands-on triage & mitigation  │       │ - Updates status page           │
│ - Executes rollbacks / scaling  │       │ - Informs executive stakeholders│
└─────────────────────────────────┘       └─────────────────────────────────┘
```

### 4.3 The Non-Negotiable Rule: Mitigation First, Debugging Second
**The Priority is ALWAYS restoring service, NEVER finding the perfect root cause during the fire.**
- *Immediate Mitigations*:
  1. Roll back to the previous known stable Docker image tag / Git commit.
  2. Shed non-critical traffic (disable background jobs, drain queues).
  3. Scale up container count / increase database compute resources.
  4. Flip feature flag to disable the offending subsystem.
- Do **NOT** keep production broken for 2 hours while attempting to attach interactive debuggers or run local reproductions. Restore service first; inspect logs and metrics in staging later.

### 4.4 The Blameless Post-Mortem Culture
Post-mortems determine **how the system allowed the failure to occur**, never pointing fingers at individuals:
- Never state: *"Engineer X deployed bad code."*
- State: *"The CI pipeline lacked integration tests verifying database migration compatibility with zero-downtime rolling restarts, allowing an incompatible schema change to pass into production."*

---

## 5. Senior Interview Q&A Cheatsheet

### Q1: "Why should you alert on SLO Error Budget Burn Rate rather than static threshold alerts like 'CPU > 80%'?"
> **Answer**:
> 1. **High CPU is often completely normal**: An optimized batch worker running at 95% CPU is efficient, not broken. Alerting on static resource thresholds generates alert fatigue and wakes engineers for non-issues.
> 2. **Symptom-based SLO Burn Rate alerts directly correlate to customer pain**: If a service's error rate is burning through 2% of the monthly error budget in 1 hour, users are actively failing transactions. Alerting on burn rate catches slow-bleed degradation and catastrophic outages with mathematically calibrated urgency, while remaining completely silent during safe, transient traffic spikes.

### Q2: "What is the role of an Incident Commander during a live SEV-1 production outage?"
> **Answer**: The Incident Commander (IC) serves as the **operational coordinator and final decision-maker**. The IC does **not** write code or debug logs directly. Their role is to:
> 1. Formulate hypotheses and assign specific diagnostic tasks to designated technical responders.
> 2. Keep the incident channel focused by preventing chaotic debates or speculation.
> 3. Make the call on aggressive mitigations (e.g. initiating database failover, full rollback, or customer traffic diversion).
> 4. Ensure an assigned Communications Lead handles status page updates and client messaging so responders can focus entirely on technical restoration.
