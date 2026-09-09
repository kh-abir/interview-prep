# 04. Redis Architecture, Data Structures, Rate Limiting & Resilience Deep Dive

> **Target Role**: Staff / Senior Backend Engineer (Rails, Node.js/TypeScript, Distributed Systems)  
> **Module**: 05-databases / 04-redis-deep-dive  
> **Key Focus**: In-Memory Data Structures (SDS, Quicklist, SkipList, Streams, HyperLogLog, Bitmaps), Production Rate Limiting (Sliding Window Log via ZSET + Lua), Caching Patterns & Stampede Mitigation, Persistence (RDB vs AOF everysec vs Hybrid), Eviction Policies, Redis Cluster (16384 Hash Slots), and Redlock Distributed Locking Realities.

---

## Table of Contents
1. [In-Memory Data Structures & Engine Internals](#1-in-memory-data-structures--engine-internals)
   - [1.1 Definition & Core Concept](#11-definition--core-concept)
   - [1.2 Internal Mechanics & Engine Realities](#12-internal-mechanics--engine-realities)
   - [1.3 Production Code & Real-World Usage](#13-production-code--real-world-usage)
   - [1.4 Production Outages & Debugging](#14-production-outages--debugging)
   - [1.5 Trade-offs & Decision Matrix](#15-trade-offs--decision-matrix)
   - [1.6 Senior Interview Q&A](#16-senior-interview-qa)
2. [Production Rate Limiting: Sliding Window Log via ZSET + Lua](#2-production-rate-limiting-sliding-window-log-via-zset--lua)
   - [2.1 Definition & Core Concept](#21-definition--core-concept)
   - [2.2 Internal Mechanics & Engine Realities](#22-internal-mechanics--engine-realities)
   - [2.3 Production Code & Real-World Usage](#23-production-code--real-world-usage)
   - [2.4 Production Outages & Debugging](#24-production-outages--debugging)
   - [2.5 Trade-offs & Decision Matrix](#25-trade-offs--decision-matrix)
   - [2.6 Senior Interview Q&A](#26-senior-interview-qa)
3. [Caching Patterns, Resilience & Persistence](#3-caching-patterns-resilience--persistence)
   - [3.1 Definition & Core Concept](#31-definition--core-concept)
   - [3.2 Internal Mechanics & Engine Realities](#32-internal-mechanics--engine-realities)
   - [3.3 Production Code & Real-World Usage](#33-production-code--real-world-usage)
   - [3.4 Production Outages & Debugging](#34-production-outages--debugging)
   - [3.5 Trade-offs & Decision Matrix](#35-trade-offs--decision-matrix)
   - [3.6 Senior Interview Q&A](#36-senior-interview-qa)
4. [High Availability, Redis Cluster & Distributed Locking (Redlock)](#4-high-availability-redis-cluster--distributed-locking-redlock)
   - [4.1 Definition & Core Concept](#41-definition--core-concept)
   - [4.2 Internal Mechanics & Engine Realities](#42-internal-mechanics--engine-realities)
   - [4.3 Production Code & Real-World Usage](#43-production-code--real-world-usage)
   - [4.4 Production Outages & Debugging](#44-production-outages--debugging)
   - [4.5 Trade-offs & Decision Matrix](#45-trade-offs--decision-matrix)
   - [4.6 Senior Interview Q&A](#46-senior-interview-qa)

---

# 1. In-Memory Data Structures & Engine Internals

### 1.1 Definition & Core Concept
Redis (Remote Dictionary Server) is an open-source, in-memory data structure store used as a database, cache, streaming engine, and message broker.

Unlike traditional disk-first relational or document stores, Redis holds its entire primary dataset in RAM. Operations are executed primarily by a **Single-Threaded Event Loop** (`aeEventLoop`) using I/O multiplexing (`epoll` on Linux, `kqueue` on macOS). This design eliminates context switching, CPU lock contention, and mutex deadlocks, delivering sub-millisecond execution latencies and over 100,000 operations per second per CPU core. (Since Redis 6.0, multi-threaded I/O offloads socket read/write operations, while command execution remains strictly single-threaded).

---

### 1.2 Internal Mechanics & Engine Realities

```
+-------------------------------------------------------------------------------+
| Redis Server Process (Single-Threaded Command Execution Core)                 |
|                                                                               |
|  [ Linux epoll Event Loop ] ──► Dispatcher ──► Command Table (dict)           |
|                                                   │                           |
|      ┌────────────────────────┬───────────────────┴───────┬────────────────┐  |
|      ▼                        ▼                           ▼                ▼  |
|  [ SDS Strings ]      [ Quicklist / Lists ]       [ SkipList / ZSET ]  [ ... ]|
|  len, alloc, buf      linked list of listpacks    hash table + skiplist       |
+-------------------------------------------------------------------------------+
```

#### 1. Strings: Simple Dynamic Strings (SDS)
Redis does not use standard C null-terminated strings (`char*`). It implements a custom dynamic string structure called **SDS (Simple Dynamic String)**:
- Structure:
  ```c
  struct __attribute__ ((__packed__)) sdshdr64 {
      uint64_t len;        // Used bytes (string length)
      uint64_t alloc;      // Allocated bytes (excluding header & null terminator)
      unsigned char flags; // Header type (sdshdr5, 8, 16, 32, or 64)
      char buf[];          // Actual character byte buffer
  };
  ```
- **$O(1)$ Length Retrieval**: `strlen` is an immediate header lookup ($O(1)$ vs $O(N)$ scanning in C).
- **Binary Safety**: Because SDS tracks length explicitly via `len`, strings can safely store arbitrary binary blobs, serialized protobufs, compressed images, and null bytes (`\0`).
- **Buffer Preallocation & Lazy Freeing**: Appending to an SDS doubles capacity (or allocates +1MB if size $> 1\text{MB}$), amortizing reallocations and reducing memory fragmentation.

#### 2. Hashes: Listpack vs. Dict (Hashtable)
Redis maps fields to values using two distinct internal encodings:
- **`listpack`** (replaced legacy `ziplist` in Redis 7.0): A contiguous memory block encoding key-value pairs sequentially without pointer overhead. Used when the hash has few elements ($\le 512$) and small values ($\le 64$ bytes). Highly cache-friendly and compact.
- **`hashtable` (`dict`)**: Switched automatically when elements or values exceed configured thresholds (`hash-max-listpack-entries`). Uses two hash tables (`ht[0]`, `ht[1]`) to perform **Incremental Rehashing**, moving buckets gradually during subsequent command executions to avoid stop-the-world pauses.

#### 3. Lists: Quicklist
- Implemented as a **Quicklist**—a doubly linked list of `listpack` memory nodes.
- Balances memory density (contiguous listpack chunks) with $O(1)$ push/pop operations at both ends (`LPUSH`, `RPOP`, `BRPOP`) without requiring massive contiguous reallocations.

#### 4. Sets: Intset vs. Dict
- **`intset`**: When a set contains solely 16, 32, or 64-bit signed integers, Redis stores them in a tightly packed, sorted contiguous binary array ($O(\log N)$ binary search).
- **`dict`**: Switched automatically when string values are introduced or member count exceeds `set-max-intset-entries` (default 512).

#### 5. Sorted Sets (ZSET): Hash Table + Skip List
Sorted Sets maintain elements ordered by floating-point scores. To deliver both $O(1)$ point lookups and $O(\log N)$ range scans, Redis pairs two data structures:
1. **Hash Table (`dict`)**: Maps member $\to$ score, guaranteeing $O(1)$ lookup for `ZSCORE`.
2. **Skip List (`zskiplist`)**: A probabilistic hierarchy of linked lists where nodes have variable heights (up to 32 levels, assigned via geometric distribution $p = 0.25$).
   - Forward pointers traverse ranges in $O(\log N)$ time.
   - Backward pointers at level 0 enable descending traversal (`ZREVRANGE`).

```
Skip List Search Path for Score = 25:
Level 3: [ Head ] ────────────────────────────────────────► [ Node: 30 ]
             │                                                    │
Level 2: [ Head ] ──────────────► [ Node: 15 ] ──────────► [ Node: 30 ]
             │                         │                          │
Level 1: [ Head ] ──► [ Node: 8 ] ──► [ Node: 15 ] ──► [ Node: 25 ] (Found!)
```

#### 6. Streams: Radix Tree + Listpack
Introduced in Redis 5.0, Streams provide an append-only log tailored for event-driven messaging:
- Data is stored in a **Radix Tree** (Rax) indexing 64-bit millisecond timestamps and sequence numbers (`<millisecondsTime>-<sequenceNumber>`).
- Supports **Consumer Groups**: Multiple workers consume different messages from the same stream without competition.
- **Pending Entries List (PEL)**: Tracks messages delivered to a consumer but not yet acknowledged via `XACK`. Unacknowledged messages can be claimed by other workers via `XCLAIM` if a consumer crashes.

#### 7. HyperLogLog: Fixed 12KB Probabilistic Cardinality
- Estimates the unique cardinality of massive datasets (e.g., billions of unique visitors) with a fixed memory footprint of exactly **12 KB** and a standard theoretical error rate of **0.81%**.
- Algorithm: Hashes elements to a 64-bit integer, uses the first 14 bits to address one of $2^{14} = 16,384$ registers, and records the maximum count of leading zeros in the remaining 50 bits. Cardinality is estimated using harmonic means across registers (Flajolet-Martin algorithm).

#### 8. Bitmaps: Bit-Level Array Operations
- Bitmaps are not an independent data type; they are bit-level operations executed on standard SDS strings.
- Using `SETBIT`, `GETBIT`, `BITCOUNT`, and `BITOP` (AND, OR, XOR), you can track binary flags across 100 million users in just **11.9 Megabytes of RAM** ($\frac{100,000,000 \text{ bits}}{8 \times 1024 \times 1024} \approx 11.92 \text{ MB}$).

---

### 1.3 Production Code & Real-World Usage

#### 1. Daily Active User (DAU) & Retention Tracking via Bitmaps (Node.js)
```typescript
import Redis from 'ioredis';

const redis = new Redis(process.env.REDIS_URL!);

export class UserActivityService {
  /**
   * Records a user active event on a specific date using SETBIT
   */
  static async recordActivity(userId: number, dateStr: string): Promise<void> {
    const key = `active:users:${dateStr}`;
    // Mark the bit at offset = userId
    await redis.setbit(key, userId, 1);
  }

  /**
   * Computes exact unique active users across a single day via BITCOUNT
   */
  static async getDailyActiveCount(dateStr: string): Promise<number> {
    return await redis.bitcount(`active:users:${dateStr}`);
  }

  /**
   * Computes users active on BOTH Date 1 AND Date 2 using BITOP AND
   */
  static async getRetainedUsers(date1: string, date2: string): Promise<number> {
    const key1 = `active:users:${date1}`;
    const key2 = `active:users:${date2}`;
    const destKey = `active:retained:${date1}:${date2}`;

    // Atomic server-side bitwise AND: stores result in destKey
    await redis.bitop('AND', destKey, key1, key2);
    // Set 24-hour expiration on transient calculation key
    await redis.expire(destKey, 86400);

    return await redis.bitcount(destKey);
  }
}
```

#### 2. Event-Driven Messaging with Redis Streams & Consumer Groups
```typescript
export class StreamProcessor {
  private static STREAM_KEY = 'events:orders';
  private static GROUP_NAME = 'order_workers';

  static async setup(): Promise<void> {
    try {
      // Create consumer group starting at current tip ($); creates stream if not exists
      await redis.xgroup('CREATE', this.STREAM_KEY, this.GROUP_NAME, '$', 'MKSTREAM');
    } catch (err: any) {
      if (!err.message.includes('BUSYGROUP')) throw err;
    }
  }

  static async produceOrderEvent(orderId: string, total: number): Promise<string> {
    // XADD stream * (auto-generate ID) field value ...
    return await redis.xadd(
      this.STREAM_KEY, 
      '*', 
      'orderId', orderId, 
      'total', total.toString(), 
      'timestamp', Date.now().toString()
    ) as string;
  }

  static async consumeOrders(consumerName: string): Promise<void> {
    while (true) {
      // XREADGROUP GROUP group consumer BLOCK ms COUNT n STREAMS key >
      const response: any = await redis.xreadgroup(
        'GROUP', this.GROUP_NAME, consumerName,
        'BLOCK', 5000,
        'COUNT', 10,
        'STREAMS', this.STREAM_KEY, '>'
      );

      if (!response) continue;

      for (const [stream, messages] of response) {
        for (const [id, fields] of messages) {
          try {
            console.log(`Processing order message: ${id}`);
            // Business logic execution...
            
            // CRITICAL: Acknowledge processing completion to clear from PEL
            await redis.xack(this.STREAM_KEY, this.GROUP_NAME, id);
          } catch (error) {
            console.error(`Failed to process message ${id}. Will be re-claimed.`, error);
          }
        }
      }
    }
  }
}
```

---

### 1.4 Production Outages & Debugging

#### Real-World Outage: The Single-Threaded Event Loop Freeze via `KEYS *`
- **Incident Summary**: A major payment gateway suffered an instant global outage with API gateways returning HTTP 504 Gateway Timeouts. Redis response latency spiked from 0.4ms to 45 seconds.
- **Root Cause**: An admin executed `KEYS user:session:*` in production to count active logins. Because Redis evaluates commands sequentially on a single thread and the database contained 35 million keys, the engine spent 42 seconds traversing the entire global `dict` table. During these 42 seconds, all incoming commands (session checks, distributed lock acquisitions, rate limiters) queued indefinitely in the TCP socket backlog, triggering thread pool exhaustion across 200 microservices.
- **Investigation Commands**:
  ```bash
  # Check slow query log (queries exceeding slowlog-log-slower-than threshold)
  redis-cli SLOWLOG GET 10

  # Inspect latency distribution
  redis-cli --latency -h 127.0.0.1 -p 6379

  # Find large keys safely without blocking the event loop
  redis-cli --bigkeys
  ```
- **Remediation**:
  1. Disabled dangerous commands in `redis.conf`:
     ```ini
     rename-command KEYS ""
     rename-command FLUSHALL ""
     rename-command FLUSHDB ""
     ```
  2. Replaced key scanning in scripts with non-blocking cursor iteration: `SCAN cursor MATCH pattern COUNT 1000`.
  3. Replaced synchronous deletion of large collections (`DEL`) with asynchronous background unlinking (`UNLINK`).

---

### 1.5 Trade-offs & Decision Matrix

| Data Structure | Read Complexity | Write Complexity | Memory Overhead | Primary Production Use Case |
| :--- | :--- | :--- | :--- | :--- |
| **String (SDS)** | $O(1)$ | $O(1)$ | Low (variable SDS headers) | Caching JSON, atomic counters (`INCR`), locks. |
| **Hash** | $O(1)$ | $O(1)$ | Extremely low with `listpack` | Object storage with partial field mutations. |
| **List (Quicklist)** | $O(1)$ at ends; $O(N)$ index | $O(1)$ push/pop | Moderate | Producer-consumer job queues, recent feeds. |
| **Set** | $O(1)$ member lookup | $O(1)$ insert/delete | Compact with `intset` | Unique tags, friend circles, set intersections (`SINTER`). |
| **Sorted Set (ZSET)** | $O(1)$ score; $O(\log N)$ rank | $O(\log N)$ | Higher (Hash table + SkipList pointers) | Leaderboards, rate limiting, priority scheduling. |
| **Streams** | $O(1)$ append; $O(\log N)$ read | $O(1)$ append | Moderate (Radix tree + listpack) | Durable event logs, multi-consumer queues with ACK. |
| **HyperLogLog** | $O(1)$ cardinality estimate | $O(1)$ add | Fixed **12 KB** always | Unique visitor counting (DAU/MAU) for billions of items. |
| **Bitmap** | $O(1)$ bit lookup; $O(N)$ count | $O(1)$ bit set | Extremely low (1 bit per entity) | Daily user activity streaks, feature flag toggles. |

---

### 1.6 Senior Interview Q&A

#### Q: "Why did Salvatore Sanfilippo (antirez) choose a Skip List instead of a Self-Balancing Binary Search Tree (such as Red-Black or AVL tree) for Redis Sorted Sets?"
> **Answer**:  
> 1. **Range Query Performance**: In a Skip List, leaf nodes at Level 0 form a contiguous singly linked list. Performing range scans (e.g., `ZRANGEBYSCORE min max` or `ZREVRANGE`) takes $O(\log N)$ to find the starting node and then traverses contiguous pointers horizontally in $O(M)$ time. In a Red-Black Tree, traversing a range requires in-order tree traversals across parent and sibling pointers, destroying CPU cache locality.
> 2. **Algorithmic Simplicity & Mutation Overhead**: Inserting into a Red-Black tree can trigger complex tree rebalancing, node rotations, and color flips across multiple levels, which requires locking or complex traversal state. In a Skip List, insertion requires only updating local pointer links across the randomly chosen height levels, making the code simpler and less error-prone.
> 3. **Memory Tunability**: The probability parameter $p$ can be adjusted. Redis uses $p = 0.25$, meaning on average each node has only 1.33 pointers, making it more memory-efficient than a balanced binary tree that strictly requires two child pointers plus a parent pointer and color metadata per node.

---

# 2. Production Rate Limiting: Sliding Window Log via ZSET + Lua

### 2.1 Definition & Core Concept
Rate limiting protects APIs against denial-of-service, brute-force attacks, and resource starvation.

While naive algorithms (Fixed Window Counter) suffer from boundary burst vulnerabilities (e.g., an attacker can send twice the limit within a 2-second interval bridging a window reset), the **Sliding Window Log** guarantees that at any arbitrary point in time $T$, the number of requests executed in the interval $[T - \text{window}, T]$ strictly does not exceed the limit.

---

### 2.2 Internal Mechanics & Engine Realities

```
Sliding Window Log using Redis Sorted Set (ZSET):
Key: "rate_limit:user_123"
ZSET Members & Scores:
+-------------------+-------------------+-------------------+
| Member: UUID_1    | Member: UUID_2    | Member: UUID_3    |
| Score: 1718000100 | Score: 1718000450 | Score: 1718000890 |  <-- Millisecond Timestamps
+-------------------+-------------------+-------------------+
                      ▲
                      │ Current Time: 1718001000, Window: 1000ms
                      └─ Step 1: ZREMRANGEBYSCORE key -inf (1718001000 - 1000) -> Purges expired entries!
                         Step 2: ZCARD key -> Counts active requests in window
                         Step 3: If count < limit -> ZADD key 1718001000 UUID_4, EXPIRE key window
```

#### 1. The Algorithm Step-by-Step:
1. **Purge Expired**: Remove all entries in the Sorted Set with scores strictly less than the window cutoff (`now - window_size_in_ms`) via `ZREMRANGEBYSCORE`.
2. **Count Active**: Count the number of remaining elements in the set via `ZCARD`.
3. **Threshold Evaluation**:
   - If `count < max_limit`: Add the current request timestamp with a unique identifier via `ZADD`, refresh key TTL via `EXPIRE`, and permit the request (HTTP 200).
   - If `count >= max_limit`: Reject the request (HTTP 429 Too Many Requests).

#### 2. The Absolute Requirement for Lua Script Atomicity
If an application executes `ZREMRANGEBYSCORE`, `ZCARD`, and `ZADD` as independent network commands:
- Under high concurrency (e.g., 50 parallel requests from the same user), all 50 threads can execute `ZCARD` concurrently, all read `count = 9` (with limit 10), and all 50 proceed to execute `ZADD`.
- The rate limit is bypassed completely due to a classic **Time-of-Check to Time-of-Use (TOCTOU)** race condition.
- **The Solution**: Redis executes Lua scripts **atomically on its single execution thread**. No other Redis operation can interleave between the check and the add, guaranteeing mathematically strict rate limiting.

---

### 2.3 Production Code & Real-World Usage

#### 1. Atomic Sliding Window Log Lua Script
```lua
-- KEYS[1]: Rate limit key (e.g., "ratelimit:ip:192.168.1.1")
-- ARGV[1]: Current timestamp in milliseconds (e.g., 1718000000000)
-- ARGV[2]: Window size in milliseconds (e.g., 60000 for 1 minute)
-- ARGV[3]: Maximum permitted requests within window (e.g., 100)
-- ARGV[4]: Unique request identifier (e.g., "req_uuid_v4")

local key = KEYS[1]
local now = tonumber(ARGV[1])
local window = tonumber(ARGV[2])
local limit = tonumber(ARGV[3])
local requestId = ARGV[4]

local clearBefore = now - window

-- 1. Remove timestamps that fall outside the active sliding window
redis.call('ZREMRANGEBYSCORE', key, '-inf', clearBefore)

-- 2. Count current valid requests within window
local currentRequests = redis.call('ZCARD', key)

if currentRequests < limit then
    -- 3. Within threshold: Add current request to Sorted Set
    redis.call('ZADD', key, now, requestId)
    -- Set expiration to auto-reclaim memory if client goes idle
    redis.call('PEXPIRE', key, window + 1000)
    -- Return [allowed (1), remaining_tokens]
    return {1, limit - currentRequests - 1}
else
    -- 4. Rate limit breached: Return [blocked (0), 0]
    return {0, 0}
end
```

#### 2. Production Express / TypeScript Middleware with Cached `EVALSHA`
```typescript
import { Request, Response, NextFunction } from 'express';
import Redis from 'ioredis';
import crypto from 'crypto';

const redis = new Redis(process.env.REDIS_URL!);

const SLIDING_WINDOW_LUA = `
local key = KEYS[1]
local now = tonumber(ARGV[1])
local window = tonumber(ARGV[2])
local limit = tonumber(ARGV[3])
local requestId = ARGV[4]

local clearBefore = now - window
redis.call('ZREMRANGEBYSCORE', key, '-inf', clearBefore)
local currentRequests = redis.call('ZCARD', key)

if currentRequests < limit then
    redis.call('ZADD', key, now, requestId)
    redis.call('PEXPIRE', key, window + 1000)
    return {1, limit - currentRequests - 1}
else
    return {0, 0}
end
`;

// Pre-load script into Redis script cache to retrieve SHA1 hash
let scriptSha: string;
async function initScript() {
  scriptSha = await redis.script('LOAD', SLIDING_WINDOW_LUA) as string;
}
initScript();

export function createSlidingWindowLimiter(windowMs: number, maxRequests: number) {
  return async (req: Request, res: Response, next: NextFunction) => {
    const identifier = req.ip || req.headers['x-forwarded-for'] || 'unknown';
    const key = `ratelimit:${identifier}`;
    const now = Date.now();
    const requestId = `${now}:${crypto.randomBytes(4).toString('hex')}`;

    try {
      // Execute via EVALSHA for optimal network performance
      let result: [number, number];
      try {
        result = await redis.evalsha(
          scriptSha, 1, key, now, windowMs, maxRequests, requestId
        ) as [number, number];
      } catch (err: any) {
        if (err.message.includes('NOSCRIPT')) {
          // Cache miss: reload and retry via EVAL
          result = await redis.eval(
            SLIDING_WINDOW_LUA, 1, key, now, windowMs, maxRequests, requestId
          ) as [number, number];
        } else {
          throw err;
        }
      }

      const [allowed, remaining] = result;

      // Attach standard RFC rate limit headers
      res.setHeader('X-RateLimit-Limit', maxRequests);
      res.setHeader('X-RateLimit-Remaining', remaining);

      if (allowed === 1) {
        return next();
      }

      res.setHeader('Retry-After', Math.ceil(windowMs / 1000));
      return res.status(429).json({
        error: 'Too Many Requests',
        message: `Exceeded request limit of ${maxRequests} per ${windowMs / 1000}s. Please retry later.`
      });
    } catch (err) {
      console.error('Rate limiting failure. Failing open to preserve availability.', err);
      return next(); // Fail-open pattern in production
    }
  };
}
```

---

### 2.4 Production Outages & Debugging

#### Real-World Outage: Memory Exhaustion via Rate Limiting High-Volume DDoS
- **Incident Summary**: A botnet targeted a public API with 250,000 requests per second across 1.2 million distinct spoofed IP addresses. While the Lua rate limiter successfully returned HTTP 429 to the attackers, the Redis cluster exhausted all 32 GB of RAM within 15 minutes, triggering `OOM command not allowed when used memory > 'maxmemory'`.
- **Root Cause**: The sliding window log creates a ZSET entry for every single request. During a massive attack, storing 250k UUID strings per second across millions of keys consumed over 30 MB of RAM per second. Although keys had TTLs, the incoming rate exceeded eviction velocity.
- **Remediation**:
  1. For ultra-high-throughput DDoS mitigation, migrated edge filtering to a **Token Bucket** or **Sliding Window Counter** algorithm implemented using atomic `INCR` commands and Redis Hashes, consuming a fixed 48 bytes per IP regardless of request volume.
  2. Pushed Layer 7 rate limiting to edge proxies (Cloudflare / AWS WAF), reserving Redis rate limiting for authenticated user IDs and API tokens.

---

### 2.5 Trade-offs & Decision Matrix

| Rate Limiting Algorithm | Memory Footprint | Boundary Accuracy | Precision Under Burst | Implementation Complexity |
| :--- | :--- | :--- | :--- | :--- |
| **Fixed Window Counter** | $O(1)$ (Single integer counter) | Low (Vulnerable to $2\times$ burst at boundary) | Poor | Trivial (`INCR` + `EXPIRE`) |
| **Sliding Window Counter**| $O(1)$ (2 counters: current & previous) | High (~99% accurate approximation) | Good | Moderate |
| **Token Bucket** | $O(1)$ (Tokens + last fill timestamp) | High (Permits controlled bursts) | Excellent | Moderate (Lua script) |
| **Sliding Window Log** | $O(N)$ (Stores timestamp per request) | **100% Mathematically Exact** | Flawless | Higher (ZSET + Lua) |

---

### 2.6 Senior Interview Q&A

#### Q: "Why is `EVALSHA` preferred over `EVAL` in production applications, and how does your application handle Redis master failover when using cached scripts?"
> **Answer**:  
> 1. `EVAL` sends the full plaintext Lua script string over the network socket on every single invocation. For a 2KB script called 10,000 times per second, this generates 20 MB/sec of unnecessary network bandwidth consumption.
> 2. `EVALSHA` computes the SHA1 digest of the script and sends only the 40-character hash. If the script is present in the Redis script cache, Redis executes it immediately.
> 3. **Handling Failover**: When a Redis primary crashes and a replica is promoted, the new master has a cold, empty script cache.
> 4. If the client sends `EVALSHA`, the server responds with: `NOSCRIPT No matching script. Please use EVAL.`.
> 5. A production Redis driver must catch the `NOSCRIPT` error, automatically fallback to sending the full script via `EVAL` (which executes the script and populates the new master's script cache), and resume subsequent calls via `EVALSHA`.

---

# 3. Caching Patterns, Resilience & Persistence

### 3.1 Definition & Core Concept
In enterprise architectures, Redis functions as a secondary high-speed cache layer fronting primary relational databases (PostgreSQL, MySQL). 

System resilience depends on correctly implementing caching topologies (Cache-Aside vs. Write-Through), preventing cache stampedes, choosing appropriate on-disk persistence guarantees (RDB vs. AOF), and selecting appropriate memory eviction policies under `maxmemory` constraints.

---

### 3.2 Internal Mechanics & Engine Realities

#### 1. The Cache-Aside Pattern with TTL Jitter
```
Application Read Flow (Cache-Aside):
Client Request ──► [ Check Redis ] ──Hit──► Return Data
                         │
                        Miss
                         ▼
                   [ Query Database ]
                         │
                         ▼
                   [ Write to Redis with Jittered TTL ] ──► Return Data
```

- **The Synchronized Expiration Catastrophe**:
  If 100,000 products are batch-updated at midnight with a static TTL of 3,600 seconds (`EXPIRE key 3600`), all 100,000 keys expire at exactly 01:00:00 AM. Subsequent incoming traffic experiences a 100% cache miss rate simultaneously, sending an avalanche of queries directly to the relational database and taking it down.
- **TTL Jitter**: To eliminate synchronized expiration, applications add pseudo-random entropy:
  $$\text{TTL} = \text{Base\_TTL} + \text{UniformRandom}(-\text{Jitter}, +\text{Jitter})$$

#### 2. Thundering Herd / Cache Stampede Mitigation
When an exceptionally hot key (e.g., the front page banner or a flash sale inventory item) expires or is invalidated, thousands of concurrent requests miss the cache simultaneously and all execute the expensive database query in parallel.

##### Mitigation Strategies:
1. **Distributed Mutex Locking (Single-Flight)**:
   The first thread encountering a cache miss acquires an exclusive Redis lock (`SET lock:key token NX EX 5`). Only the lock owner queries the database and populates Redis. Concurrent threads wait or poll, reading the cached result once populated.
2. **Probabilistic Early Expiration (The XFetch Algorithm)**:
   Instead of waiting for the key to expire, worker threads compute a probabilistic formula on every read based on the computational cost ($\Delta$), a tuning constant ($\beta$), and remaining TTL:
   $$-\beta \times \Delta \times \ln(\text{random}()) > \text{Remaining\_TTL}$$
   As the expiration time approaches, the probability that a reader will asynchronously recompute and refresh the cache in the background approaches 1.0, ensuring the key **never actually expires** for end-users.

#### 3. Persistence Models: RDB vs. AOF vs. Hybrid
Redis provides two independent on-disk persistence engines:

```
+-------------------------------------------------------------------------------+
| Redis Persistence Engines                                                     |
|                                                                               |
| 1. RDB (Snapshot)  ──► fork() child process writes point-in-time dump.rdb     |
| 2. AOF (Log File)  ──► Writes every mutating command to appendonly.aof        |
|    - appendfsync always:   fsync() per write (Slowest, zero data loss)        |
|    - appendfsync everysec: fsync() once per second (Industry standard)        |
|    - appendfsync no:       Delegates to OS dirty page flush (Risk of loss)    |
| 3. Hybrid (Redis 4+) ──► RDB preamble + incremental AOF tail (Fast & Durable) |
+-------------------------------------------------------------------------------+
```

- **RDB (Redis Database Snapshot)**:
  - Writes a point-in-time compact binary image of memory to disk via `bgsave`.
  - Mechanism: Invokes Linux `fork()`. The child process traverses memory and writes `dump.rdb`.
  - **Copy-on-Write (COW) Danger**: The parent and child share physical memory pages. As the parent receives new writes, the Linux kernel duplicates the modified 4KB pages. On a write-heavy database, COW can double Redis memory usage, triggering OS OOM-Killer crashes.
- **AOF (Append-Only File)**:
  - Appends every write command to `appendonly.aof` using the Redis serialization protocol (RESP).
  - `appendfsync everysec` flushes the OS write buffer to disk every second on a dedicated background thread, guaranteeing a maximum of **1 second of data loss** upon power failure.
  - Background Rewrite (`bgrewriteaof`): Compresses the log by writing the minimal commands necessary to reconstruct the current memory state.
- **Hybrid Persistence (Default in Modern Redis)**:
  - Combines the rapid recovery speed of RDB with the granular durability of AOF. The rewritten AOF file starts with an RDB binary snapshot header, followed by incremental RESP log commands appended since the snapshot started.

#### 4. Memory Eviction Policies (`maxmemory`)
When RAM usage reaches `maxmemory`, Redis applies an eviction policy:
- `noeviction`: Returns errors on write commands (`OOM command not allowed`).
- **`allkeys-lru`**: Evicts the Least Recently Used keys across all keys.
- **`volatile-lru`**: Evicts LRU keys only among those with an active TTL.
- **`allkeys-lfu`**: Evicts the Least Frequently Used keys across all keys.
- **`volatile-lfu`**: Evicts LFU keys only among those with an active TTL.
- **Engine Reality**: Redis does **NOT** maintain an exact doubly linked list for LRU (which would consume 16 bytes of pointer overhead per key and require mutex locking on every read). Instead, Redis uses **Approximated LRU/LFU**: it randomly samples $N$ keys (default `maxmemory-samples 5`) and evicts the best candidate among the sample. At $N = 10$, approximated LRU is mathematically indistinguishable from true LRU.

---

### 3.3 Production Code & Real-World Usage

#### 1. Resilient Cache-Aside with Jitter & Mutex Stampede Protection (TypeScript)
```typescript
import Redis from 'ioredis';
import crypto from 'crypto';

const redis = new Redis(process.env.REDIS_URL!);

interface CacheOptions {
  ttlSeconds: number;
  jitterPercent?: number; // e.g., 0.15 for +/- 15%
  lockTimeoutMs?: number;
}

export async function fetchWithCache<T>(
  key: string,
  fetchFromDb: () => Promise<T>,
  options: CacheOptions
): Promise<T> {
  const { ttlSeconds, jitterPercent = 0.2, lockTimeoutMs = 5000 } = options;

  // 1. Check Cache
  const cached = await redis.get(key);
  if (cached) {
    return JSON.parse(cached) as T;
  }

  // 2. Cache Miss: Mitigate Cache Stampede via Distributed Mutex
  const lockKey = `lock:${key}`;
  const lockToken = crypto.randomBytes(16).toString('hex');
  
  // Acquire lock: SET lockKey lockToken NX PX lockTimeoutMs
  const acquired = await redis.set(lockKey, lockToken, 'PX', lockTimeoutMs, 'NX');

  if (acquired === 'OK') {
    try {
      // Primary thread queries DB
      const data = await fetchFromDb();

      // Compute TTL with random Jitter
      const jitterRange = ttlSeconds * jitterPercent;
      const jitterDelta = (Math.random() * 2 - 1) * jitterRange; // Uniform between [-range, +range]
      const finalTtl = Math.max(1, Math.floor(ttlSeconds + jitterDelta));

      // Populate Cache
      await redis.set(key, JSON.stringify(data), 'EX', finalTtl);
      return data;
    } finally {
      // Safely release lock using Lua script (guarantees token ownership)
      const releaseLua = `
        if redis.call('get', KEYS[1]) == ARGV[1] then
          return redis.call('del', KEYS[1])
        else
          return 0
        end
      `;
      await redis.eval(releaseLua, 1, lockKey, lockToken);
    }
  } else {
    // Concurrent threads wait briefly and retry cache read
    await new Promise((resolve) => setTimeout(resolve, 50));
    return fetchWithCache(key, fetchFromDb, options);
  }
}
```

#### 2. Production `redis.conf` Tuning
```ini
# Memory Capacity & Eviction
maxmemory 16gb
maxmemory-policy allkeys-lru
maxmemory-samples 10             # Higher sampling matches true LRU precision

# Persistence Configuration (Hybrid AOF + RDB)
save 900 1                       # RDB snapshot fallback
save 300 10
appendonly yes
appendfilename "appendonly.aof"
appendfsync everysec             # Industry standard durability (max 1s loss)
no-appendfsync-on-rewrite yes    # Prevents fsync latency spikes during AOF rewrite
aof-use-rdb-preamble yes         # Hybrid persistence mode

# Multi-Threaded I/O (Redis 6+)
io-threads 4                     # Tune to available CPU cores (typically core count - 1)
io-threads-do-reads yes
```

---

### 3.4 Production Outages & Debugging

#### Real-World Outage: The Copy-on-Write (COW) Kernel OOM Massacre
- **Incident Summary**: An e-commerce Redis instance provisioned with 30 GB of RAM on a 32 GB server abruptly died under high traffic. System monitoring showed the Redis process vanished with no application log errors.
- **Root Cause**: 
  - The instance had `save 60 10000` enabled. During peak flash sale writes, Redis triggered a background `bgsave`.
  - The Linux kernel cloned the page table for the child process.
  - Because write volume was extraordinarily high, incoming writes mutated hundreds of thousands of keys, forcing the Linux kernel to execute Copy-on-Write (COW), allocating new physical 4KB memory pages for every modified key.
  - Total process memory footprint expanded to 34 GB, exceeding physical RAM and swap.
  - The Linux kernel OOM-Killer executed, sending `SIGKILL` to the parent Redis daemon.
- **Remediation**:
  1. Set Linux memory overcommit policy in `/etc/sysctl.conf`:
     ```ini
     vm.overcommit_memory = 1
     ```
  2. Provisioned physical RAM with at least **30-40% headroom** above `maxmemory` (`maxmemory 20gb` on a 32gb instance).
  3. Disabled periodic RDB snapshots on the master node, delegating RDB snapshot generation entirely to a dedicated read-only replica.

---

### 3.5 Trade-offs & Decision Matrix

| Dimension | RDB Snapshots | AOF (`everysec`) | Hybrid Mode |
| :--- | :--- | :--- | :--- |
| **Data Loss Window** | Minutes (Time since last snapshot) | $\le 1$ second | $\le 1$ second |
| **Startup / Recovery Time** | Ultra-fast (Direct memory load) | Slow (Replays millions of commands) | Fast (Loads RDB header, replays tail) |
| **Disk File Size** | Highly compact binary dump | Larger text-based log | Compact |
| **Runtime Performance Impact** | High during `fork()` and COW | Low, constant background disk I/O | Low |

---

### 3.6 Senior Interview Q&A

#### Q: "What is the difference between `allkeys-lru` and `allkeys-lfu` in Redis, and when should you choose LFU over LRU?"
> **Answer**:  
> 1. **LRU (Least Recently Used)**: Evicts keys based on **time of last access**. Each object maintains a 24-bit timestamp clock. If a key was accessed 1 second ago, it is retained; if it was accessed 1 hour ago, it is evicted.
> 2. **LFU (Least Frequently Used)**: Evicts keys based on **frequency of access**. The 24-bit field is split into:
>    - 16 bits: Last decrement time.
>    - 8 bits: Logistic Counter (an approximate logarithmic frequency counter that increments probabilistically and decays over time).
> 3. **The Defect of LRU**: A large, one-off batch scan or crawling script can read millions of rarely accessed keys into memory once. Under LRU, these cold keys appear "hot" (recent access timestamp) and evict truly essential, high-frequency keys from cache.
> 4. **When to Choose LFU**: Choose `allkeys-lfu` when your workload has long-term power-law access distributions (Pareto 80/20 rule), ensuring that keys accessed thousands of times are not evicted by occasional transient scan traffic.

---

# 4. High Availability, Redis Cluster & Distributed Locking (Redlock)

### 4.1 Definition & Core Concept
Redis scales horizontally through **Redis Cluster**, a decentralized, multi-master sharding architecture that automatically partitions data across multiple nodes without requiring an intermediary proxy.

For distributed concurrency synchronization across independent nodes, the **Redlock Algorithm** provides a consensus-based distributed lock protocol designed to overcome single-point-of-failure risks in single-master setups.

---

### 4.2 Internal Mechanics & Engine Realities

```
Redis Cluster 16,384 Hash Slots Distribution:
+------------------------+------------------------+------------------------+
| Master Node A          | Master Node B          | Master Node C          |
| Slots: 0 - 5460        | Slots: 5461 - 10922    | Slots: 10923 - 16383   |
+------------------------+------------------------+------------------------+
            ▲
            │ CRC16(key) % 16384 = Slot 2840 -> Routed to Node A!
            │ If sent to Node B -> Responds with: "-MOVED 2840 10.0.0.1:6379"
```

#### 1. Hash Slots & Hash Tags
- Redis Cluster partitions the keyspace into exactly **16,384 Hash Slots**.
- Every key is assigned to a slot using the mathematical formula:
  $$\text{Slot} = \text{CRC16}(\text{key}) \pmod{16384}$$
- **Hash Tags `{...}`**: Multi-key operations (`MGET`, transactions, Lua scripts) require that all involved keys reside on the **exact same hash slot**.
  - If a key contains `{...}`, only the text inside the curly braces is hashed.
  - Example: Keys `{user:101}:profile` and `{user:101}:orders` hash solely on `"user:101"`, guaranteeing they map to the exact same hash slot and node.
- **Client Redirection**:
  - `MOVED`: The slot has migrated permanently to another master. The client updates its internal slot-to-node cache and retries.
  - `ASK`: The slot is currently migrating. The client sends an `ASKING` command followed by the query to the target node for a single request, retaining its original cache.

#### 2. The Redlock Algorithm & The Martin Kleppmann Debate
To achieve distributed locking without a single point of failure, Salvatore Sanfilippo created **Redlock**:
- Mechanism:
  1. Deploys $N$ completely independent Redis masters (typically 5) with zero replication links between them.
  2. Client generates a unique random token and records the start timestamp $T_1$.
  3. Attempts to acquire the lock on all $N$ instances sequentially using identical keys and tokens with a small timeout ($5\text{ms}–50\text{ms}$).
  4. The lock is considered acquired if the client obtains the lock on a **strict majority ($> N/2 = 3$)** of nodes AND the elapsed time $(T_2 - T_1)$ is less than the lock validity time.
  5. If acquisition fails, the client releases all instances (even those it failed to acquire on).

```
Martin Kleppmann's Critical Flaw Analysis:
Client 1 acquires Redlock ──► Holds lock on majority
         │
         ▼
[ Stop-The-World GC Pause (e.g. 15 seconds) / VM Pause ]
         │
         ▼ Lock expires on Redis nodes while Client 1 is paused!
Client 2 acquires Redlock ──► Writes to Storage
         │
         ▼
Client 1 wakes up from GC pause (thinks it still has lock!) ──► Writes to Storage (DATA CORRUPTION!)
```

##### Martin Kleppmann's Critique:
1. **Clock Drift & NTP Jumps**: If a server's hardware clock jumps forward due to unsynchronized NTP updates, the lock expires prematurely on that node, violating mutual exclusion.
2. **Stop-the-World Pauses**: If an application thread suffers a GC pause, hypervisor descheduling, or network hiccup after acquiring the lock, the lock validity expires while the thread is asleep. Upon waking, the thread believes it holds the lock and performs conflicting writes.
3. **Absence of Fencing Tokens**: Redlock does not provide monotonically increasing fencing tokens (like ZooKeeper `zxid` or etcd `revision`). Without fencing tokens, the underlying storage layer cannot reject writes from stale clients.
4. **Conclusion**: For business-critical operations where data corruption is unacceptable (financial ledgers, billing), use consensus-backed systems like **ZooKeeper**, **etcd**, or database transactional locks. Redlock is suitable only for non-critical operational coordination (e.g., preventing duplicate background email sending).

---

### 4.3 Production Code & Real-World Usage

#### 1. Safe Single-Instance Distributed Lock with Auto-Renewal (TypeScript)
```typescript
import Redis from 'ioredis';
import crypto from 'crypto';

const redis = new Redis(process.env.REDIS_URL!);

export class DistributedLock {
  private key: string;
  private token: string;
  private ttlMs: number;
  private renewalTimer: NodeJS.Timeout | null = null;

  constructor(resourceName: string, ttlMs: number = 10000) {
    this.key = `lock:${resourceName}`;
    this.token = crypto.randomUUID();
    this.ttlMs = ttlMs;
  }

  async acquire(): Promise<boolean> {
    // SET key token NX PX ttlMs
    const status = await redis.set(this.key, this.token, 'PX', this.ttlMs, 'NX');
    if (status === 'OK') {
      this.startHeartbeatRenewal();
      return true;
    }
    return false;
  }

  /**
   * Heartbeat / Watchdog timer: Automatically extends lock TTL
   * while the long-running task is actively executing
   */
  private startHeartbeatRenewal(): void {
    const interval = Math.floor(this.ttlMs / 3);
    this.renewalTimer = setInterval(async () => {
      const renewLua = `
        if redis.call('get', KEYS[1]) == ARGV[1] then
          return redis.call('pexpire', KEYS[1], ARGV[2])
        else
          return 0
        end
      `;
      try {
        const renewed = await redis.eval(renewLua, 1, this.key, this.token, this.ttlMs);
        if (renewed !== 1) {
          this.stopHeartbeatRenewal();
        }
      } catch (err) {
        this.stopHeartbeatRenewal();
      }
    }, interval);
  }

  private stopHeartbeatRenewal(): void {
    if (this.renewalTimer) {
      clearInterval(this.renewalTimer);
      this.renewalTimer = null;
    }
  }

  /**
   * Safely release lock using Lua: Only delete if token matches
   */
  async release(): Promise<boolean> {
    this.stopHeartbeatRenewal();
    const releaseLua = `
      if redis.call('get', KEYS[1]) == ARGV[1] then
        return redis.call('del', KEYS[1])
      else
        return 0
      end
    `;
    const result = await redis.eval(releaseLua, 1, this.key, this.token);
    return result === 1;
  }
}
```

---

### 4.4 Production Outages & Debugging

#### Real-World Outage: The CrossSlot Lua Exception in Redis Cluster
- **Incident Summary**: An engineering team migrated a monolithic Redis instance to a 6-node Redis Cluster. Immediately after deployment, the checkout service crashed with:
  `UnhandledPromiseRejection: ReplyError: CROSSSLOT Keys in request don't hash to the same slot`.
- **Root Cause**: The checkout pipeline executed a multi-key Lua script operating on `user:9812:cart` and `inventory:item:4401`. In Redis Cluster, `user:9812:cart` hashed to Slot 4,110 (Node A), while `inventory:item:4401` hashed to Slot 12,890 (Node C). Redis Cluster strictly forbids operations involving keys located on different nodes or different slots within the same command or Lua script.
- **Remediation**:
  Refactored key naming using **Hash Tags**:
  `{cart:9812}:user` and `{cart:9812}:inventory`. By enclosing the tenant/cart ID in curly braces `{...}`, Redis hashes solely on `"cart:9812"`, guaranteeing that both keys map to the exact same hash slot and allow atomic execution.

---

### 4.5 Trade-offs & Decision Matrix

| Architecture | Scalability | Failover Mechanism | Multi-Key Support | Client Routing Complexity |
| :--- | :--- | :--- | :--- | :--- |
| **Standalone Master-Replica** | Vertical only | Manual | Full support across all keys | Trivial |
| **Redis Sentinel** | Vertical (High Availability) | Automated failover via Sentinel quorum | Full support across all keys | Moderate (Queries Sentinel for master) |
| **Redis Cluster** | Horizontal (Shards data across 16,384 slots)| Automated failover via cluster gossip | Restricted to same hash slot (`{...}`) | High (Cluster-aware client with MOVED handling) |

---

### 4.6 Senior Interview Q&A

#### Q: "Why does Redis Cluster use exactly 16,384 hash slots instead of 65,536 ($2^{16}$)? What was the specific engineering trade-off?"
> **Answer**:  
> Salvatore Sanfilippo specifically addressed this design decision:
> 1. **Heartbeat Packet Overhead**: Redis Cluster nodes continuously exchange gossip ping/pong packets every second to detect node failures. Each heartbeat packet includes the node's view of slot ownership as a **bitmap**.
>    - With 16,384 slots ($16\text{K}$ bits), the bitmap occupies exactly **2 Kilobytes** of memory ($\frac{16384}{8} = 2048\text{ bytes}$).
>    - With 65,536 slots ($64\text{K}$ bits), the bitmap would occupy **8 Kilobytes**, quadrupling gossip packet network traffic across hundreds of nodes.
> 2. **Cluster Scalability Realities**: Redis Cluster is architected to scale realistically up to ~1,000 master nodes. 16,384 slots provide an average of 16 slots per node at 1,000 nodes, which is more than sufficient granularity for balancing, rendering 65,536 slots completely unnecessary.
> 3. **Bitmap Compression**: At 16K slots, slot allocations compress cleanly within the binary protocol frame, optimizing memory and network bandwidth.

#### Q: "Why can't Redlock guarantee safety in the presence of long process pauses (e.g. Stop-The-World GC pauses), and what is a Fencing Token?"
> **Answer**:  
> 1. **The Vulnerability**: 
>    - Client 1 acquires Redlock with a 10-second lease.
>    - Immediately after acquisition, Client 1 enters a 15-second Stop-The-World GC pause (or hypervisor CPU freeze).
>    - While Client 1 is paused, the 10-second lease expires on the Redis instances.
>    - Client 2 requests the lock, successfully acquires it from the majority, and begins modifying shared storage.
>    - Client 1 wakes up from its GC pause. It has no awareness that 15 seconds elapsed, believes it still holds the valid lock, and executes its write to shared storage, **overwriting Client 2's data and causing split-brain data corruption**.
> 2. **Fencing Tokens**:
>    - A **Fencing Token** is a strictly monotonically increasing integer counter (e.g., 101, 102, 103) issued by the lock service with every successful lock acquisition.
>    - When Client 1 acquires the lock, it receives Token 101.
>    - When Client 2 acquires the lock after Client 1's pause, it receives Token 102.
>    - When Client 2 writes to the database or storage layer, it passes Token 102; the storage layer records that the highest observed token is now 102.
>    - When Client 1 wakes up and sends its write containing stale Token 101, the storage layer compares $101 < 102$ and **rejects the write**.
>    - Because Redis locks cannot generate monotonically increasing fencing tokens across independent uncoordinated masters, Redlock cannot prevent this class of distributed data corruption.
