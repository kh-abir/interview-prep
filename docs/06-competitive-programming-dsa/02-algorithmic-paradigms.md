# 02 — Algorithmic Paradigms & Advanced Techniques

> **Context**: ICPC Regionalist Level (3,000+ problems solved). Deep-dive on paradigms, mathematical proofs, and advanced string/graph algorithms.

---

## 1. Binary Search: The "Search on Answer" Meta-Pattern

### 1.1 The Monotonicity Principle
Classic binary search finds an element in a sorted array. The **Search on Answer** pattern solves optimization problems:
*"Find the minimum (or maximum) value $X$ such that condition $f(X)$ is satisfied."*

If $f(X)$ is monotonic ($f(X) = \text{true} \implies f(Y) = \text{true}$ for all $Y > X$), binary search finds the optimal value in $O(\log(\text{range}) \cdot \text{Cost}(f))$.

### 1.2 Production Scenario: Minimum Server Capacity to Finish Batch Jobs
```python
def can_finish(capacity: int, weights: list[int], max_days: int) -> bool:
    """Predicate f(X): Can we ship all payloads with capacity X within max_days?"""
    days = 1
    current_load = 0
    for w in weights:
        if w > capacity:
            return False
        if current_load + w > capacity:
            days += 1
            current_load = w
        else:
            current_load += w
    return days <= max_days

def ship_within_days(weights: list[int], days: int) -> int:
    low = max(weights)        # Cannot have capacity less than heaviest item
    high = sum(weights)       # Maximum possible capacity needed (1 day)
    optimal_capacity = high

    while low <= high:
        mid = (low + high) // 2
        if can_finish(mid, weights, days):
            optimal_capacity = mid
            high = mid - 1   # Try smaller capacity
        else:
            low = mid + 1    # Need more capacity

    return optimal_capacity
```

---

## 2. Dynamic Programming: State Space Design

### 2.1 The 6 Core DP Archetypes

| DP Archetype | Typical State Definition | Recurrence Relation | Space Optimization |
|---|---|---|---|
| **1D Linear** | `dp[i]`: optimal answer for prefix $0 \dots i$ | `dp[i] = max(dp[i-1], dp[i-2] + val[i])` | Rolling variables: $O(N) \to O(1)$ |
| **2D Grid / Sequence** | `dp[i][j]`: subproblem over prefixes $s[0..i]$ and $t[0..j]$ | `dp[i][j] = (s[i]==t[j]) ? dp[i-1][j-1] + 1 : max(dp[i-1][j], dp[i][j-1])` | Rolling 2 rows: $O(M \cdot N) \to O(\min(M, N))$ |
| **0/1 Knapsack** | `dp[w]`: max value achievable with exact weight $w$ | `dp[w] = max(dp[w], dp[w - weight[i]] + val[i])` | Reverse 1D array traversal prevents double-use |
| **Interval DP** | `dp[i][j]`: optimal answer for subarray from index $i$ to $j$ | `dp[i][j] = min_{i \le k < j}(dp[i][k] + dp[k+1][j] + cost)` | $O(N^2)$ space |
| **Bitmask DP** | `dp[mask][u]`: cost to visit subset of nodes `mask` ending at node `u` | `dp[mask | (1<<v)][v] = min(dp[mask][u] + dist[u][v])` | $O(2^N \cdot N)$ |
| **Tree DP & Rerooting** | `dp[u]`: answer considering subtree rooted at `u` | Bottom-up post-order DFS + Top-down rerooting pass | $O(N)$ space |

### 2.2 Bitmask DP: The Traveling Salesperson Problem (TSP)
```python
def tsp(n: int, dist: list[list[int]]) -> int:
    """Finds minimum Hamiltonian cycle in O(2^N * N^2) time."""
    memo = {}

    def solve(mask: int, u: int) -> int:
        if mask == (1 << n) - 1:
            return dist[u][0] # Return to origin
        if (mask, u) in memo:
            return memo[(mask, u)]

        ans = float('inf')
        for v in range(n):
            if not (mask & (1 << v)):
                ans = min(ans, dist[u][v] + solve(mask | (1 << v), v))

        memo[(mask, u)] = ans
        return ans

    return solve(1, 0) # Start at node 0 with mask 00...001
```

---

## 3. Greedy Algorithms & The Exchange Argument

### 3.1 When Does Greedy Work?
A greedy algorithm makes the locally optimal choice at each stage. It is valid **if and only if** the problem exhibits:
1. **Greedy Choice Property**: A global optimum can be arrived at by selecting a local optimum.
2. **Optimal Substructure**: An optimal solution contains within it optimal solutions to subproblems.

### 3.2 The Exchange Argument Proof Technique
To mathematically prove a greedy strategy is optimal:
1. Assume there exists an optimal schedule $O$ that differs from greedy schedule $G$.
2. Find the first point of divergence between $O$ and $G$.
3. "Exchange" the choice in $O$ with the greedy choice from $G$.
4. Prove that the objective function value of the modified $O'$ is **greater than or equal to** $O$.
5. By induction, $O$ can be transformed into $G$ without degrading quality, proving $G$ is optimal.

---

## 4. Advanced Graph Algorithms

### 4.1 0-1 BFS in $O(V + E)$
When edges have weights of only **$0$ or $1$** (e.g. grid navigation where turning costs 1 and straight costs 0):
- Standard Dijkstra requires a Priority Queue: $O((V + E)\log V)$.
- **0-1 BFS with Double-Ended Queue (Deque)**:
  - If edge weight is $0$: Push neighbor to the **front** of the deque.
  - If edge weight is $1$: Push neighbor to the **back** of the deque.
  - Guarantees vertices are processed in strictly ascending distance order in pure **$O(V + E)$ linear time**.

```python
from collections import deque

def zero_one_bfs(graph: dict[int, list[tuple[int, int]]], start: int, n: int) -> list[int]:
    dist = [float('inf')] * n
    dist[start] = 0
    dq = deque([start])

    while dq:
        u = dq.popleft()
        for v, weight in graph[u]:
            if dist[u] + weight < dist[v]:
                dist[v] = dist[u] + weight
                if weight == 0:
                    dq.appendleft(v)
                else:
                    dq.append(v)
    return dist
```

### 4.2 Tarjan's Strongly Connected Components & Bridges
Finds critical network bottlenecks (bridges / single points of failure) in a single DFS pass ($O(V + E)$) using discovery times `tin[u]` and lowest reachable ancestor `low[u]`.
- Edge $(u, v)$ is a **bridge** if and only if `low[v] > tin[u]`.

---

## 5. Advanced String Algorithms

### 5.1 Knuth-Morris-Pratt (KMP) & The Prefix Function
- Computes the longest proper prefix of $P[0..i]$ that is also a suffix: the $\pi$-table in $O(M)$ time.
- Matches pattern $P$ in text $T$ in **$O(N + M)$ time without ever backtracking in the text**.

```python
def compute_kmp_pi(pattern: str) -> list[int]:
    m = len(pattern)
    pi = [0] * m
    j = 0
    for i in range(1, m):
        while j > 0 and pattern[i] != pattern[j]:
            j = pi[j - 1]
        if pattern[i] == pattern[j]:
            j += 1
        pi[i] = j
    return pi

def kmp_search(text: str, pattern: str) -> list[int]:
    pi = compute_kmp_pi(pattern)
    matches = []
    j = 0
    for i in range(len(text)):
        while j > 0 and text[i] != pattern[j]:
            j = pi[j - 1]
        if text[i] == pattern[j]:
            j += 1
        if j == len(pattern):
            matches.append(i - len(pattern) + 1)
            j = pi[j - 1]
    return matches
```

### 5.2 Aho-Corasick Multi-Pattern Search
- Builds a Trie over $K$ search keywords, augmented with **failure links** (analogous to KMP $\pi$-table).
- Finds all occurrences of all $K$ patterns in text $T$ simultaneously in **$O(|T| + \sum |P_i| + \text{matches})$** time.
- **Production Use Case**: High-speed firewall packet inspection, antivirus malware signature scanning, and profanity filtering.

---

## 6. Number Theory & Combinatorics

### 6.1 Modular Arithmetic & Fermat's Little Theorem
In CP and cryptography, computations are performed modulo prime $p = 10^9 + 7$ or $998244353$.
- Division by $b$ modulo prime $p$: $a / b \equiv a \cdot b^{-1} \pmod p$.
- By **Fermat's Little Theorem**: If $p$ is prime and $\gcd(b, p) = 1$, then $b^{p-1} \equiv 1 \pmod p$.
- Therefore: **$b^{-1} \equiv b^{p-2} \pmod p$**.

```python
MOD = 1_000_000_007

def fast_power(base: int, exp: int) -> int:
    """Binary Exponentiation in O(log exp)."""
    res = 1
    base %= MOD
    while exp > 0:
        if exp % 2 == 1:
            res = (res * base) % MOD
        base = (base * base) % MOD
        exp //= 2
    return res

def mod_inverse(n: int) -> int:
    return fast_power(n, MOD - 2)

def nCr(n: int, r: int, fact: list[int], inv_fact: list[int]) -> int:
    if r < 0 or r > n:
        return 0
    return fact[n] * inv_fact[r] % MOD * inv_fact[n - r] % MOD
```

---

## 7. Senior Interview Q&A Cheatsheet

### Q1: "Why does QuickSort run in $O(N^2)$ worst-case, and how do modern runtimes prevent this?"
> **Answer**: QuickSort degrades to $O(N^2)$ when the chosen pivot consistently splits the array into unbalanced partitions of size $0$ and $N-1$ (e.g. sorted input with first element as pivot).
> Modern runtimes (C++ `std::sort`, V8, Go) prevent this using **Introsort**:
> 1. Starts with QuickSort using a median-of-three or Tukey's ninther pivot selection.
> 2. Tracks recursion stack depth: if depth exceeds $2 \cdot \lfloor \log_2 N \rfloor$, it automatically switches to **HeapSort** to guarantee strict $O(N\log N)$ worst-case time.
> 3. Switches to **InsertionSort** for small partitions ($N \le 16$) to exploit L1 CPU cache locality.

### Q2: "How does Matrix Exponentiation compute the $N$-th Fibonacci number in $O(\log N)$ time?"
> **Answer**: Express linear recurrence in matrix form:
> $$\begin{pmatrix} F_{n+1} \\ F_n \end{pmatrix} = \begin{pmatrix} 1 & 1 \\ 1 & 0 \end{pmatrix} \begin{pmatrix} F_n \\ F_{n-1} \end{pmatrix} = \begin{pmatrix} 1 & 1 \\ 1 & 0 \end{pmatrix}^n \begin{pmatrix} F_1 \\ F_0 \end{pmatrix}$$
> By using **Binary Exponentiation** to compute the matrix power $M^n$, we perform repeated matrix multiplications in $O(K^3 \log N)$ where matrix dimension $K = 2$. For $N = 10^{18}$, this executes in under **100 microseconds**.
