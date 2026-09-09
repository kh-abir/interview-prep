# 04 — Node.js Debugging & Production Error Patterns

> **Context**: Production Incident Triage, SRE diagnostics, and root-cause analysis for Senior Node.js / TypeScript systems.

---

## 1. The Anatomy of Common Production Errors & Fixes

### 1.1 `ECONNREFUSED` vs. `ETIMEDOUT`

| Error Code | System Layer | Root Cause | Production Remediation |
|---|---|---|---|
| **`ECONNREFUSED`** | TCP Layer (OS) | The target host is reachable, but **no process is listening on the target port**, or the target service crashed, or security group blocks the port with an explicit TCP RST. | Verify downstream container/database status. Implement circuit breakers (`opossum`) to prevent stampeding a recovering service. |
| **`ETIMEDOUT`** | Network / IP Layer | No response packet was received within the OS socket timeout (typically 75s-120s) because a firewall silently drops packets (black hole), or network interface routing failed. | Check AWS Security Group egress rules, VPC NAT Gateway routes, or set explicit application-level connection timeouts (`timeout: 3000`). |

```typescript
// Production Axios Client with Jittered Retry and Circuit Breaker
import axios from 'axios';
import axiosRetry from 'axios-retry';

export const apiClient = axios.create({
  timeout: 4000, // 4-second timeout to prevent holding event loop connections
});

axiosRetry(apiClient, {
  retries: 3,
  retryDelay: (retryCount) => {
    // Exponential backoff with full jitter to avoid thundering herd
    const delay = Math.pow(2, retryCount) * 1000;
    const jitter = Math.random() * 200;
    return delay + jitter;
  },
  retryCondition: (error) => {
    // Retry only on network timeouts and idempotent 5xx errors
    return (
      axiosRetry.isNetworkOrIdempotentRequestError(error) ||
      error.code === 'ECONNRESET' ||
      error.code === 'ETIMEDOUT'
    );
  },
});
```

---

### 1.2 `ERR_HTTP_HEADERS_SENT`
**The Symptom**: Server throws unhandled exception:
`Error [ERR_HTTP_HEADERS_SENT]: Cannot set headers after they are sent to the client`

```typescript
// ANTI-PATTERN: Multiple responses in control flow
app.post('/api/charge', async (req, res, next) => {
  if (!req.body.amount) {
    res.status(400).json({ error: 'Amount required' }); // FORGOT TO RETURN!
  }

  // Code continues executing!
  const charge = await chargeCard(req.body.amount);
  res.json({ success: true, charge }); // CRASH: Headers already sent!
});

// PRODUCTION FIX: Always prefix responses with `return`
app.post('/api/charge', async (req, res, next) => {
  if (!req.body.amount) {
    return res.status(400).json({ error: 'Amount required' });
  }

  const charge = await chargeCard(req.body.amount);
  return res.json({ success: true, charge });
});
```

---

### 1.3 `ENOMEM` & V8 Heap Exhaustion
**The Symptom**: Kubernetes pod terminates with status code `137` (OOMKilled) or Node.js logs:
`FATAL ERROR: Ineffective mark-compacts near heap limit Allocation failed - JavaScript heap out of memory`

#### Resolution Protocol:
1. **Configure Heap Limit**: By default, Node.js limits heap memory to ~1.4GB on 64-bit systems. Tune it explicitly to fit within container cgroup limits:
   ```bash
   # In Kubernetes / Docker container with 4GB RAM limit:
   # Reserve ~700MB for OS + C++ native buffers
   node --max-old-space-size=3300 dist/server.js
   ```
2. **Diagnose with Node Diagnostic Reports**:
   Configure Node to dump a diagnostic JSON file automatically when memory approaches exhaustion:
   ```bash
   node --report-on-fatalerror --report-on-signal=SIGUSR2 dist/server.js
   ```

---

### 1.4 Event Loop Blockage & Catastrophic Regex Backtracking (ReDoS)
A regular expression with nested quantifiers evaluated against an adversarial string can trigger exponential backtracking $O(2^N)$, locking the CPU core at 100% and completely freezing all other concurrent users.

```javascript
// CATASTROPHIC REGEX PATTERN:
// Regex: /^(a+)+$/
// Target string: "aaaaaaaaaaaaaaaaaaaaaaaaaaaaaa!"
// Triggers over 1 billion backtracking steps, freezing the event loop for ~15 seconds!

// PRODUCTION FIX: Use `re2` (Google's linear-time automata regex engine)
const RE2 = require('re2');
const safeRegex = new RE2('^(a+)+$');
// Executes in strictly O(N) linear time, completely immune to ReDoS!
```

---

## 2. Advanced Diagnostic & Profiling Tooling

### 2.1 Generating Live Diagnostic Reports
Node.js includes a native diagnostic reporting system that captures JavaScript call stacks, native C++ thread stacks, libuv active handles, and memory statistics into a single structured JSON document.

```javascript
// Triggering diagnostic report on critical event or latency spike
const process = require('process');

function dumpDiagnosticReport() {
  const filename = process.report.writeReport('/tmp/node-diagnostic-report.json');
  console.log(`Diagnostic report generated at: ${filename}`);
}

// In terminal: trigger without restarting process
// kill -USR2 <PID>
```

### 2.2 Clinic.js Diagnostics Suite
`clinic` is the gold-standard diagnostic suite for production Node.js applications:
1. **`clinic doctor`**: Analyzes event loop delay, memory growth, and CPU usage. Automatically diagnoses whether a bottleneck is I/O-bound, CPU-bound, or event loop starvation.
2. **`clinic flame`**: Generates interactive Flamegraphs of CPU execution stacks to locate hot functions.
3. **`clinic bubbleprof`**: Visualizes asynchronous execution latency across async/await boundaries.

```bash
# Benchmark server under load and generate interactive HTML diagnostic report
clinic doctor --on-port 'autocannon -c 100 -d 30 http://localhost:3000/users' -- node dist/server.js
```

---

## 3. Production Process Resilience & Crash Boundaries

### 3.1 Uncaught Exceptions vs. Unhandled Rejections
```typescript
// src/index.ts
import { logger } from './utils/logger';

// 1. Unhandled Promise Rejections (e.g. forgotten .catch() on non-awaited promise)
process.on('unhandledRejection', (reason: unknown, promise: Promise<unknown>) => {
  logger.error({ reason }, 'Unhandled Promise Rejection detected');
  // Send alert to Sentry / Datadog
});

// 2. Uncaught Exceptions (e.g. synchronous throw outside try/catch)
// CRITICAL: The application state is now CORRUPT! You MUST shut down!
process.on('uncaughtException', (error: Error) => {
  logger.fatal({ error }, 'Uncaught Exception! Initiating emergency shutdown...');
  
  // Close HTTP server and flush logs before exiting
  server.close(() => {
    process.exit(1); // Process orchestrator (Kubernetes/PM2) will spin up a fresh instance
  });

  // If server.close() hangs, force kill after 3 seconds
  setTimeout(() => process.exit(1), 3000).unref();
});
```

---

## 4. Senior Interview Q&A Cheatsheet

### Q1: "How do you detect which line of code is blocking the Node.js event loop in production?"
> **Answer**:
> 1. Use the `perf_hooks` module with `monitorEventLoopDelay({ resolution: 20 })` to measure 99th percentile event loop lag in real-time.
> 2. Use `blocked-at` gem/npm package, which tracks call stacks when event loop delay exceeds a threshold (e.g., >100ms).
> 3. Generate CPU flamegraphs using `clinic flame` or by passing `--prof` to Node, running the profiler, and processing tick logs with `node --prof-process isolate-*.log`.
> 4. Inspect libuv active handles via `process._getActiveHandles()` and `process._getActiveRequests()` to see queued tasks.

### Q2: "What is the difference between `process.exit(0)` and `process.exit(1)` in containerized environments?"
> **Answer**: An exit code of `0` signals clean, intentional completion (success). An exit code of `1` signals an abnormal termination or unhandled error. In Kubernetes or Docker Compose with `restart: on-failure`, an exit code of `1` automatically instructs the container runtime to restart the pod/container. If a crashed process exits with `0`, the orchestrator treats it as a completed batch job and will **not** restart it, leaving your service completely dead.
