# 01 — Node.js Runtime Architecture (Zero to Senior)

> **Context**: Primary Backend Engine (JD-CRITICAL). Covers V8 internals, event loop mechanics, libuv thread pool, stream backpressure, and production memory leak hunting.

---

## 1. V8 Engine Internals & Mechanical Sympathy

### 1.1 JIT Compilation Pipeline: Ignition & TurboFan
V8 does not interpret JavaScript naively. It uses a two-tier adaptive compilation pipeline:
```
JavaScript Source
       │
       ▼
Parser (AST: Abstract Syntax Tree)
       │
       ▼
Ignition Bytecode Generator (Fast startup, low memory footprint)
       │
       ▼ (Execution + Profiler feedback / Type feedback vectors)
       ├── [Hot Code + Stable Types] ────────► TurboFan Optimizing Compiler (Machine Code)
       │                                              │
       └── [Type Polymorphism / De-opt] ◄─────────────┘ (De-optimization deopt bailout)
```

1. **Hidden Classes (Shapes)**:
   - When an object is created `const obj = { x: 1 }`, V8 assigns hidden class `C0`.
   - Adding `obj.y = 2` transitions it to hidden class `C1`.
   - **Performance Trap**: Initializing properties in different orders (`{ x, y }` vs `{ y, x }`) generates separate hidden classes, destroying **Inline Caching (IC)** and causing a 5x-10x slowdown in hot loops.
   - **Production Rule**: Always initialize object fields in the exact same order, preferably inside constructor functions or factory classes.

2. **Inline Caching (IC)**:
   - *Monomorphic*: Accesses property on objects with 1 hidden class (fastest, direct memory offset).
   - *Polymorphic*: 2-4 hidden classes (switch table lookup).
   - *Megamorphic*: 5+ hidden classes (falls back to global hash table lookup, slowest).

---

## 2. Event Loop Phases & Microtask Priority

### 2.1 The 6 Phases of the libuv Event Loop
Node.js runs single-threaded JavaScript execution orchestrated by the **libuv event loop**.

```
   ┌───────────────────────────┐
┌─►│          timers           │  setTimeout(), setInterval()
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │     pending callbacks     │  I/O callbacks deferred from previous loop
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │       idle, prepare       │  Internal libuv use only
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │           poll            │  Retrieve new I/O events (epoll/kqueue); blocks if queue empty
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │           check           │  setImmediate() callbacks run here
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │      close callbacks      │  socket.on('close', ...)
└────────────────┴─────────────┘
```

### 2.2 Microtask Queue Priority & Starvation Risk
Between **every single phase** of the event loop, Node.js exhausts the **Microtask Queues**:
1. **`process.nextTick` Queue**: Runs first, with absolute priority.
2. **Promise Microtask Queue**: Runs native Promise resolutions and async/await continuations.

```javascript
// Starvation Demonstration
function recursiveNextTick() {
  process.nextTick(recursiveNextTick); // STARVES the event loop!
}
// If called, timers, I/O callbacks, and HTTP requests NEVER fire!
```

#### `setImmediate` vs. `process.nextTick` vs. `setTimeout(fn, 0)`:
- `process.nextTick`: Executes immediately after current synchronous JavaScript stack finishes, before entering the next event loop phase.
- `setTimeout(fn, 0)`: Scheduled in the *timers* phase. Minimum delay is actually clamped to 1ms.
- `setImmediate`: Scheduled in the *check* phase, specifically designed to run after I/O polling completes.

---

## 3. libuv Thread Pool & `UV_THREADPOOL_SIZE`

### 3.1 What Runs on the libuv Thread Pool?
JavaScript executes on 1 thread. Network I/O (TCP/HTTP/HTTPS) is delegated directly to the OS kernel via non-blocking sockets (`epoll` on Linux, `kqueue` on macOS).
However, 3 operations **cannot** be done asynchronously by standard POSIX kernels and run on libuv's background thread pool:
1. **File System Operations** (`fs.*`)
2. **DNS Lookups** (`dns.lookup` uses `getaddrinfo(3)`, blocking the thread pool)
3. **Crypto** (`crypto.pbkdf2`, `crypto.scrypt`, hashing)
4. **Zlib compression**

### 3.2 Thread Pool Starvation & Tuning
Default thread pool size is **4 threads**. If 4 concurrent clients hash passwords via `crypto.pbkdf2`, all 4 threads are occupied. Any concurrent file read or `dns.lookup` will stall in queue.

```bash
# Production fix: Must be exported BEFORE Node process initializes!
export UV_THREADPOOL_SIZE=16
node server.js
```

```javascript
// Demonstration: Concurrency bottleneck with default thread pool
const crypto = require('crypto');

const start = Date.now();
for (let i = 0; i < 8; i++) {
  crypto.pbkdf2('secret', 'salt', 100000, 64, 'sha512', () => {
    console.log(`Hash ${i + 1} completed: ${Date.now() - start}ms`);
  });
}
// With UV_THREADPOOL_SIZE=4: Hashes 1-4 finish in ~100ms. Hashes 5-8 finish in ~200ms.
// With UV_THREADPOOL_SIZE=8: All 8 finish simultaneously in ~110ms.
```

---

## 4. Node.js Streams & Backpressure Management

### 4.1 The Memory Hazard of Naive Buffering
Loading a 500MB video or CSV via `fs.readFile()` copies 500MB directly into V8 heap memory. Under 10 concurrent requests, this consumes 5GB RAM and triggers an immediate Out-Of-Memory (`ENOMEM`) crash.

### 4.2 Stream Backpressure Mechanics
When a Readable stream produces data faster than a Writable stream can consume (e.g. fast SSD reading data going over slow 3G cellular socket), `writable.write(chunk)` returns `false`.
If the developer ignores `false` and keeps pushing data, unconsumed chunks queue in the internal buffer (`highWaterMark`, default 16KB for streams, 64KB for `fs`), leading to heap exhaustion.

```javascript
// Production Solution: pipeline with automatic backpressure and error cleanup
const fs = require('fs');
const zlib = require('zlib');
const { pipeline } = require('stream/promises');

async function compressLargeFile(sourcePath, destPath) {
  try {
    await pipeline(
      fs.createReadStream(sourcePath, { highWaterMark: 64 * 1024 }),
      zlib.createGzip(),
      fs.createWriteStream(destPath)
    );
    console.log('Compression pipeline completed successfully.');
  } catch (err) {
    console.error('Pipeline failed and auto-destroyed stream handles:', err);
    throw err;
  }
}
```

---

## 5. V8 Memory Architecture & Leak Hunting

### 5.1 V8 Heap Layout
```
Total Process Memory (RSS: Resident Set Size)
├── C++ Bindings & Native Buffers (Allocated via Buffer.allocUnsafe - outside V8 heap!)
└── V8 Heap (Controlled via --max-old-space-size)
    ├── New Space (Young Generation: 1-64MB)
    │   ├── Semi-space: From Space
    │   └── Semi-space: To Space  (Scavenger GC: fast Cheney copying)
    └── Old Space (Old Generation)
        ├── Old Pointer Space (Surviving objects pointing to other objects)
        ├── Old Data Space (Raw data: strings, boxed numbers)
        └── Large Object Space (Objects exceeding single page limit)
```

### 5.2 The 4 Classic Production Memory Leaks
1. **Accidental Global Caches**: `const cache = {}` without TTL or LRU eviction limits.
2. **Closures Retaining Outer Scope**: An inner function retaining an outer scope variable containing large buffers or request objects.
3. **Forgotten Timers / Intervals**: `setInterval()` referencing variables in closure without `clearInterval()`.
4. **Unremoved Event Listeners**: Listening on a long-lived singleton emitter `eventEmitter.on('data', handler)` inside a short-lived request lifecycle.

### 5.3 Hunting Memory Leaks with Heap Snapshots
```javascript
// Triggering programmatic heap snapshot on high memory threshold
const v8 = require('v8');
const fs = require('fs');

function checkMemoryThreshold() {
  const mem = process.memoryUsage();
  const heapUsedMB = mem.heapUsed / 1024 / 1024;

  if (heapUsedMB > 1200) { // Alert at 1.2GB
    const snapshotStream = v8.getHeapSnapshot();
    const fileName = `/tmp/heap-${Date.now()}.heapsnapshot`;
    const fileStream = fs.createWriteStream(fileName);
    snapshotStream.pipe(fileStream);
    console.error(`ALERT: Heap snapshot dumped to ${fileName}`);
  }
}
setInterval(checkMemoryThreshold, 10000);
```

#### Chrome DevTools Retainers Tree Analysis:
1. Open Chrome -> `chrome://inspect` -> Click "Open dedicated DevTools for Node".
2. Load 2 `.heapsnapshot` files (taken 10 minutes apart under load).
3. Select "Comparison" view -> Sort by **# Alloc** and **Size Delta**.
4. Expand the suspect object class -> Look at the **Retainers Tree**.
5. Identify the GC Root (usually a global variable, module-level array, or un-cleared closure) preventing garbage collection.

---

## 6. Multi-Processing: `cluster` vs. `worker_threads`

| Feature | `cluster` Module | `worker_threads` Module |
|---|---|---|
| **Architecture** | Multiple OS processes | Multiple OS threads within 1 process |
| **Memory Isolation** | Full isolation; zero memory sharing | Shared heap (`SharedArrayBuffer`), independent V8 isolates |
| **Communication** | IPC serialized message passing | Fast message channels or shared memory pointers |
| **Crash Blast Radius** | Process crashes without killing siblings | Uncaught exception in thread can crash entire host process |
| **Primary Use Case** | Scale HTTP servers across CPU cores | Heavy CPU computation (image processing, crypto, ML inference) |

### 6.1 Production Clustering with Node.js Cluster Module
```javascript
// cluster_server.js
const cluster = require('cluster');
const http = require('http');
const os = require('os');

if (cluster.isPrimary) {
  const numCPUs = os.cpus().length;
  console.log(`Primary master ${process.pid} running. Forking ${numCPUs} workers...`);

  for (let i = 0; i < numCPUs; i++) {
    cluster.fork();
  }

  cluster.on('exit', (worker, code, signal) => {
    console.warn(`Worker ${worker.process.pid} died. Forking replacement...`);
    cluster.fork();
  });
} else {
  // Workers share the same TCP port via OS SO_REUSEPORT or IPC socket handoff
  http.createServer((req, res) => {
    res.writeHead(200);
    res.end(`Handled by worker PID ${process.pid}\n`);
  }).listen(3000);

  console.log(`Worker ${process.pid} started`);
}
```

---

## 7. Senior Interview Q&A Cheatsheet

### Q1: "Why does parsing a 60MB JSON string lock up an Express web server?"
> **Answer**: V8's `JSON.parse()` executes **synchronously on the main JavaScript execution thread**. A 60MB payload parse can take 300ms to 800ms of uninterrupted CPU time. Because Node.js uses a single-threaded event loop, while `JSON.parse` is running, the event loop cannot proceed to the *poll* phase to accept new TCP connections or process responses for other connected clients. Every concurrent request stalls.
> **Remedy**: Stream the JSON using a streaming parser (e.g. `JSONStream` or `stream-json`), offload parsing to a `worker_threads` pool, or validate and reject oversized request bodies at the NGINX / API Gateway layer.

### Q2: "What is the difference between `dns.lookup` and `dns.resolve` in Node.js?"
> **Answer**: `dns.lookup` calls the underlying OS synchronous C library function `getaddrinfo(3)`. It runs on the **libuv thread pool** (default 4 threads), respects `/etc/hosts`, but easily causes thread starvation under high concurrent outgoing HTTP requests.
> In contrast, `dns.resolve` uses **c-ares**, an asynchronous C DNS resolver that performs non-blocking network I/O directly over UDP sockets on the event loop. It does not touch the libuv thread pool and scales to tens of thousands of concurrent resolutions, but bypasses `/etc/hosts`.
