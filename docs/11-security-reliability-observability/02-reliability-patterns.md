# 02 — Distributed Reliability & Resilience Patterns

> **Context**: SRE & Staff Backend Systems Architecture. Covers Circuit Breakers, Jittered Retries, Bulkheads, Adaptive Load Shedding, and Chaos Engineering.

---

## 1. The Circuit Breaker Pattern

### 1.1 State Machine Mechanics
When a downstream microservice or third-party payment gateway degrades, continuing to send traffic causes thread starvation, connection pool depletion, and cascading failures across the entire cluster.

```
       ┌──────────────────────┐
       │        CLOSED        │◄─────────────────────────┐
       │ (Normal Operation)   │                          │
       └──────────┬───────────┘                          │
                  │ Failure Rate > Threshold (e.g. 50%)  │ Success Rate > 90%
                  ▼                                      │
       ┌──────────────────────┐                          │
       │         OPEN         │                          │
       │ (Fast Fail: Fallback)│                          │
       └──────────┬───────────┘                          │
                  │ Sleep Window Expired (e.g. 30s)      │
                  ▼                                      │
       ┌──────────────────────┐                          │
       │      HALF-OPEN       ├──────────────────────────┘
       │ (Trial Probe Traffic)│
       └──────────────────────┘
                  │
                  ▼ Probe Fails
       [ Returns to OPEN ]
```

### 1.2 Production Circuit Breaker Implementation (Opossum)
```typescript
import CircuitBreaker from 'opossum';
import axios from 'axios';
import { logger } from '../utils/logger';

async function fetchThirdPartyRates(currency: string) {
  const { data } = await axios.get(`https://api.forex-rates.com/v1/latest?base=${currency}`, {
    timeout: 2000, // 2-second strict network timeout
  });
  return data;
}

const circuitOptions: CircuitBreaker.Options = {
  timeout: 2500,               // If action takes longer than 2.5s, trigger failure
  errorThresholdPercentage: 50,// Open circuit if 50% of requests fail
  resetTimeout: 30000,         // Wait 30 seconds in OPEN state before trying HALF-OPEN
  rollingCountTimeout: 10000,  // 10-second rolling statistical window
  volumeThreshold: 10,         // Minimum 10 requests in window before evaluating threshold
};

export const forexBreaker = new CircuitBreaker(fetchThirdPartyRates, circuitOptions);

// Fallback execution when circuit is OPEN or call times out
forexBreaker.fallback((currency: string) => {
  logger.warn({ currency }, 'Forex Circuit Breaker OPEN! Serving stale cached fallback rates.');
  return getStaleCachedRates(currency);
});

forexBreaker.on('open', () => logger.fatal('ALERT: Forex Circuit Breaker has flipped to OPEN!'));
forexBreaker.on('halfOpen', () => logger.info('Forex Circuit Breaker is HALF-OPEN. Probing...'));
forexBreaker.on('close', () => logger.info('Forex Circuit Breaker has CLOSED. Normal operation restored.'));
```

---

## 2. Retry Strategies & Full Jitter Mathematics

### 2.1 The Thundering Herd Hazard of Naive Retries
If 5,000 requests fail simultaneously and all retry with standard exponential backoff ($2^i \cdot 1000\text{ms}$), all 5,000 clients retry at the exact same millisecond ($2\text{s}, 4\text{s}, 8\text{s}$), repeatedly hammering and re-killing the recovering upstream service.

### 2.2 Full Jitter Formula (AWS Architecture Recommendation)
To break synchronized waves of traffic, add randomness to spread retry spikes uniformly across the interval:

$$T_{\text{sleep}} = \text{random}(0, \ \min(M, \ B \cdot 2^i))$$

```typescript
// Full Jitter Exponential Backoff Calculation
export function calculateFullJitterSleep(
  attempt: number,
  baseMs = 100,
  maxBackoffMs = 10000
): number {
  const exponentialCap = Math.min(maxBackoffMs, baseMs * Math.pow(2, attempt));
  // Uniform random distribution between 0 and exponentialCap
  return Math.random() * exponentialCap;
}
```

---

## 3. The Bulkhead Pattern: Resource Isolation

Named after the watertight bulkheads in ships: if one compartment floods, the ship remains afloat.
- **Problem**: In a web server, a slow analytics reporting query consumes all 50 database connections in the connection pool. High-priority user checkout queries stall and time out.
- **Solution**: Split shared resource pools into dedicated, independent quotas:
  - `Pool A (Checkout / Core API)`: 35 connections reserved.
  - `Pool B (Reporting / Admin Dashboard)`: 10 connections max.
  - `Pool C (Async Background Workers)`: 5 connections max.
  - If reporting queries exhaust their 10 connections, checkout queries operate without degraded latency.

---

## 4. Adaptive Load Shedding & Concurrency Limits

### 4.1 Little's Law & Concurrency Limits
Traditional rate limiters cap requests per minute (RPM), which ignores request processing time.
By **Little's Law**:

$$\text{Concurrency } (L) = \text{Throughput } (\lambda) \times \text{Latency } (W)$$

If downstream database latency increases from 10ms to 500ms, the number of concurrent in-flight requests holding server memory multiplies by 50x.
- **Adaptive Load Shedding**: Monitor queue latency or CPU saturation. When median response time exceeds the SLA threshold, the server immediately sheds load by rejecting non-critical requests with **HTTP 503 Service Unavailable** and a `Retry-After: 5` header before they touch backend worker threads.

---

## 5. Senior Interview Q&A Cheatsheet

### Q1: "What is Deadline Propagation and why is it essential in microservice call trees?"
> **Answer**: In a distributed call chain ($A \to B \to C \to D$), if service $A$ sets a user timeout of 3.0 seconds, but service $B$ takes 2.9 seconds before calling $C$:
> Without deadline propagation, service $C$ and $D$ will start expensive computations that will take another 2 seconds. Meanwhile, service $A$ has already timed out and disconnected the client. Services $C$ and $D$ are burning database and CPU resources computing a response that nobody is waiting for.
> **Deadline Propagation** injects the remaining allowable time into the request header (e.g. `grpc-timeout: 100m` or `X-Request-Deadline`). When service $C$ receives the request, it checks remaining time: if $< 10\text{ms}$, it immediately aborts the call chain without executing downstream work.

### Q2: "What is the difference between Load Balancing and Load Shedding?"
> **Answer**:
> - **Load Balancing**: Distributes incoming traffic evenly across a pool of healthy servers (Round Robin, Least Connections, Consistent Hashing). It attempts to accommodate all requests.
> - **Load Shedding**: An internal defense mechanism executed by an individual overloaded server. When the server detects that system capacity (CPU, memory, or thread pool queue delay) has crossed safe operational thresholds, it intentionally rejects low-priority incoming requests to protect the performance and availability of existing in-flight high-priority requests.
