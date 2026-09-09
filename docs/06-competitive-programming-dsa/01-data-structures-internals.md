# 01 — Data Structures: Internals & Production Systems

> **Context**: ICPC Regionalist Foundation (3,000+ problems solved). Bridges competitive programming algorithms to high-scale production systems (HTTP routers, rate limiters, storage engines).

---

## 1. Hash Tables & The HashDoS Vulnerability

### 1.1 Collision Resolution Mechanics
```
Key: "user_session_491" ──► Hash Function (MurmurHash3 / SipHash) ──► 64-bit Integer
                                                                          │ modulo Array Capacity (N)
                                                                          ▼
                                                                Bucket Index [7]
```

| Strategy | Collision Resolution | Worst Case | Cache Friendliness |
|---|---|---|---|
| **Chaining** (Java `HashMap`, Ruby Hash) | Linked list at each bucket. Java converts list to **Red-Black Tree** when bucket length $\ge 8$ and table capacity $\ge 64$. | $O(\log N)$ with tree; $O(N)$ with raw linked list. | Poor (pointer chasing across heap). |
| **Open Addressing** (Python `dict`, Go `runtime.map`) | Linear probing ($h + i$), Quadratic probing ($h + i^2$), or Double Hashing ($h_1 + i \cdot h_2$). | $O(N)$ under clustering. | **Superior** (contiguous array scanning benefits from 64-byte L1 cache lines). |

### 1.2 HashDoS Attack & Mitigation
- **Vulnerability**: If an application uses an unseeded, deterministic hash function, an attacker can generate thousands of distinct strings that produce the **exact same bucket index**. A POST request with 50,000 colliding JSON keys degrades hash table insertion from $O(1)$ to $O(N^2)$, freezing the server CPU at 100%.
- **Production Defense**: Cryptographically secure, randomized-seed hash functions:
  - Ruby / Python / Go: Use **SipHash-2-4** with a randomized per-process seed generated at boot time.
  - Express / Node.js: Limit incoming parameter keys (`express.urlencoded({ parameterLimit: 1000 })`).

---

## 2. Monotonic Stacks & Monotonic Queues in Production

### 2.1 Monotonic Stack: Next Greater Element in $O(N)$
Maintains elements in strictly decreasing or increasing order. When a new element violates monotonicity, older elements are popped.

```python
def next_greater_elements(arr: list[int]) -> list[int]:
    n = len(arr)
    result = [-1] * n
    stack = []  # stores indices

    for i in range(n):
        while stack and arr[i] > arr[stack[-1]]:
            idx = stack.pop()
            result[idx] = arr[i]
        stack.append(i)
    return result
```

### 2.2 Production Monotonic Queue: Rolling 5-Minute Sliding Window Max
**Real-World Scenario**: Calculate the maximum concurrent request spike in a sliding 5-minute rolling window for a DDoS mitigation engine in real-time.
- A naive re-scan takes $O(W)$ per incoming metric.
- A **Monotonic Deque** achieves **$O(1)$ amortized time per insertion**.

```typescript
// Monotonic Queue maintaining strictly decreasing order of metric values
class RollingWindowMax {
  // Deque stores: { timestamp: number, value: number }
  private deque: { timestamp: number; value: number }[] = [];

  constructor(private readonly windowDurationMs: number) {}

  public recordMetric(timestamp: number, value: number): void {
    // 1. Purge entries older than window expiration from front
    while (this.deque.length > 0 && this.deque[0].timestamp <= timestamp - this.windowDurationMs) {
      this.deque.shift();
    }

    // 2. Maintain strictly decreasing order: purge smaller values from back
    while (this.deque.length > 0 && this.deque[this.deque.length - 1].value <= value) {
      this.deque.pop();
    }

    // 3. Insert current metric
    this.deque.push({ timestamp, value });
  }

  public getMax(): number | null {
    if (this.deque.length === 0) return null;
    // Front element is always the maximum in current active window!
    return this.deque[0].value;
  }
}
```

---

## 3. Trees in Production Infrastructure

### 3.1 Radix Trie: The Backbone of High-Performance HTTP Routers
- Standard Trie stores one character per edge.
- **Radix Trie (Compressed Trie)** merges single-child nodes together (e.g. `/api/v1/` becomes one edge).
- **Why It Beats Regex Linear Scans**:
  - Express with 200 routes: evaluates 200 regular expressions linearly $O(N)$.
  - Next.js App Router / Go Chi Router / Rails Journey: traverses the Radix Trie in $O(L)$ where $L$ is URL path length, independent of whether the application has 10 or 10,000 routes.

```
                  /api/v1/
                  ├── users/
                  │   └── :id (param node)
                  │       ├── /profile (GET)
                  │       └── /orders  (GET)
                  └── organizations/
                      └── :org_id
```

### 3.2 Interval Trees in Scheduling Systems (Airbnb, Calendly, Uber)
- **Problem**: Given $N$ calendar bookings, find all overlapping bookings for a proposed time interval $[t_{start}, t_{end}]$.
- Naive scan: $O(N)$ for every search.
- **Augmented Interval Tree**: An augmented Red-Black tree where each node stores $[low, high]$ and `max_subtree_high`.
- **Query Performance**: Detects overlapping intervals in $O(\log N + K)$ time where $K$ is the number of overlaps.

---

## 4. Consistent Hashing: Distributed Partitioning

### 4.1 Modulo Hashing Failure at Scale
- Naive partitioning: `node_id = hash(key) % N`.
- **The Catastrophe**: When adding node $N+1$ or when 1 node crashes:
  Almost **100% of existing keys are remapped** to different nodes ($k \pmod N \ne k \pmod{N+1}$).
  The cache completely misses simultaneously, stampeding downstream primary databases into a SEV-1 outage.

### 4.2 Consistent Hashing Ring with Virtual Nodes
- Map both servers and keys onto a $0$ to $2^{32}-1$ integer ring.
- A key is assigned to the **first server encountered traveling clockwise**.
- When node $C$ is added between $A$ and $B$, only keys between $A$ and $C$ move. Exactly **$1/N$ of keys are redistributed**.
- **Virtual Nodes**: Each physical server is assigned 100-256 virtual replicas on the ring (`server1#1`, `server1#2`). Eliminates hotspots and guarantees uniform key distribution across servers (Cassandra, DynamoDB, Redis Cluster).

```typescript
// Production Consistent Hashing Ring Implementation
import crypto from 'crypto';

export class ConsistentHashRing {
  private ring: Map<number, string> = new Map();
  private sortedHashes: number[] = [];

  constructor(
    private readonly replicaCount: number = 150
  ) {}

  private hashKey(key: string): number {
    return crypto.createHash('md5').update(key).digest().readUInt32BE(0);
  }

  public addNode(node: string): void {
    for (let i = 0; i < this.replicaCount; i++) {
      const virtualNodeName = `${node}#${i}`;
      const hash = this.hashKey(virtualNodeName);
      this.ring.set(hash, node);
      this.sortedHashes.push(hash);
    }
    this.sortedHashes.sort((a, b) => a - b);
  }

  public removeNode(node: string): void {
    for (let i = 0; i < this.replicaCount; i++) {
      const virtualNodeName = `${node}#${i}`;
      const hash = this.hashKey(virtualNodeName);
      this.ring.delete(hash);
    }
    this.sortedHashes = Array.from(this.ring.keys()).sort((a, b) => a - b);
  }

  public getNode(key: string): string {
    if (this.sortedHashes.length === 0) throw new Error('Hash ring is empty');

    const hash = this.hashKey(key);
    // Binary search (lower_bound) for the first server hash >= key hash
    let low = 0;
    let high = this.sortedHashes.length - 1;
    let targetIdx = 0;

    while (low <= high) {
      const mid = Math.floor((low + high) / 2);
      if (this.sortedHashes[mid] >= hash) {
        targetIdx = mid;
        high = mid - 1;
      } else {
        low = mid + 1;
      }
    }

    const matchedHash = this.sortedHashes[targetIdx];
    return this.ring.get(matchedHash)!;
  }
}
```

---

## 5. Senior Interview Q&A Cheatsheet

### Q1: "Why does an $O(N)$ sequential array scan often execute faster than an $O(\log N)$ balanced Binary Search Tree lookup for small $N$ ($N < 200$)?"
> **Answer**: **Cache Locality & Hardware Realities**.
> 1. Arrays reside in contiguous physical RAM. When CPU fetches `array[0]`, the hardware prefetcher loads the entire **64-byte L1 cache line** into L1 data cache (1ns latency). Subsequent sequential elements are already in CPU cache.
> 2. BST nodes are dynamically allocated across the heap. Traversing pointers causes repeated **L1/L2 cache misses**, forcing 100ns round-trips to main RAM.
> 3. CPU **branch predictors** excel at predictable sequential loop scans, avoiding the ~15 CPU cycle pipeline stall penalty of pointer-chasing branches.

### Q2: "How does an LRU Cache achieve $O(1)$ `get` and $O(1)$ `put` operations?"
> **Answer**: By combining a **Hash Map** and a **Doubly Linked List**:
> 1. The Doubly Linked List maintains access recency (Head = Most Recently Used, Tail = Least Recently Used). Node insertion and removal take $O(1)$ when pointing to node references.
> 2. The Hash Map maps keys directly to Doubly Linked List node memory pointers, providing $O(1)$ key lookups.
> 3. On `get(key)`: Locate node in $O(1)$ via hash map, unlink node from its position, and splice it to the Head in $O(1)$.
> 4. On `put(key, val)`: If key exists, update value and move to Head. If capacity exceeded, remove Tail node from list and delete its key from the Hash Map in $O(1)$.
