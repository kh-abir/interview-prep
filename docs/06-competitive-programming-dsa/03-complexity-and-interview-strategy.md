# 03 — Complexity Analysis & Interview Problem-Solving Strategy

> **Context**: ICPC Regionalist Meta-Playbook. Bridges theoretical Big-O analysis and the 45-minute technical coding interview pacing formula.

---

## 1. Formal Complexity Analysis & Amortized Mathematics

### 1.1 Asymptotic Notations Defined
- **$O(g(n))$ (Big-O)**: Asymptotic upper bound. $f(n) \in O(g(n))$ if $\exists c > 0, n_0 > 0$ such that $0 \le f(n) \le c \cdot g(n)$ for all $n \ge n_0$.
- **$\Omega(g(n))$ (Big-Omega)**: Asymptotic lower bound ($f(n) \ge c \cdot g(n)$).
- **$\Theta(g(n))$ (Big-Theta)**: Asymptotically tight bound ($c_1 \cdot g(n) \le f(n) \le c_2 \cdot g(n)$).

### 1.2 Amortized Analysis: The 3 Proof Techniques

```
Operation 1..N:   Cost 1 each
Operation N+1:    Array doubles -> Cost N (re-allocation + copy)
Total Cost:       N + N = 2N operations over N insertions
Amortized Cost:   2N / N = 2 = O(1) amortized!
```

1. **Aggregate Method**:
   Compute total cost $T(n)$ for a sequence of $n$ operations. The amortized cost per operation is $T(n) / n$.
   - *Example*: Dynamic array resizing by doubling factor 2. After $n$ appends, total elements copied is $\sum_{i=0}^{\log_2 n} 2^i < 2n$. Amortized cost is $O(1)$ per append.
2. **Accounting (Banker's) Method**:
   Assign an amortized charge $\hat{c}_i$ to each operation. Cheap operations deposit "credit" in the bank; expensive operations draw down credit to pay for their execution.
3. **Potential Method (Physicist's)**:
   Define potential function $\Phi(D)$ mapping data structure state $D$ to a real number.
   Amortized cost $\hat{c}_i = c_i + \Phi(D_i) - \Phi(D_{i-1})$.
   - *Example*: **Union-Find / DSU** with Path Compression and Union by Rank achieves amortized time $O(\alpha(N))$ per operation, where $\alpha(N)$ is the **Inverse Ackermann Function** ($\alpha(10^{80}) \le 4$, effectively constant).

---

## 2. Constraint-Driven Algorithm Selection Matrix

In competitive programming and coding interviews, **input constraints dictate the required algorithm before writing a single line of code**.

| Input Constraint ($N$) | Target Time Complexity | Typical Required Paradigms / Algorithms |
|---|---|---|
| **$N \le 10$** | $O(N!)$ or $O(N^6)$ | Full Permutations, Exhaustive Brute Force Backtracking |
| **$N \le 20$** | $O(2^N \cdot N)$ | Bitmask Dynamic Programming, Meet-in-the-Middle ($2^{N/2}$) |
| **$N \le 100$** | $O(N^4)$ or $O(N^3)$ | 2D Grid DP, Floyd-Warshall All-Pairs Shortest Path |
| **$N \le 500$** | $O(N^3)$ | Matrix Chain Multiplication, Interval DP, Hopcroft-Karp Matching |
| **$N \le 5,000$** | $O(N^2)$ | 2D DP, Two Pointers, Bubble/Insertion Sort, String LCS / Edit Distance |
| **$N \le 10^5$** | $O(N \log N)$ | MergeSort, QuickSort, Binary Search on Answer, Segment Tree, Fenwick Tree, Dijkstra with Priority Queue, Greedy with Sorting |
| **$N \le 10^6$** | $O(N)$ | Monotonic Stack/Queue, Sliding Window, Prefix Sums, Hash Table, KMP, BFS/DFS, Two Pointers |
| **$N \ge 10^9$ or $10^{18}$** | $O(\log N)$ or $O(1)$ | Binary Search, Matrix Exponentiation, Fast Modular Exponentiation, Math / Number Theory |

*Rule of Thumb*: A modern CPU performs approximately **$10^8$ basic operations per second**. An algorithm requiring $> 3 \cdot 10^8$ operations will trigger a Time Limit Exceeded (TLE) or HTTP 504 Gateway Timeout.

---

## 3. The 45-Minute Technical Coding Interview Execution Blueprint

```
[ 00:00 - 05:00 ] Step 1: Clarify & Extract Constraints
[ 05:00 - 10:00 ] Step 2: Propose Brute Force -> Optimize -> State Big-O
[ 10:00 - 32:00 ] Step 3: Implement Production-Grade Code (Zero hand-waving)
[ 32:00 - 40:00 ] Step 4: Dry-Run with Edge Cases & Trace Variables
[ 40:00 - 45:00 ] Step 5: Staff Trade-offs & Production System Mapping
```

### 3.1 Step 1: Clarification Checklist (Minutes 0-5)
1. **Input Bounds**: "What is the maximum value of $N$? Can $N = 0$ or the array be empty?"
2. **Data Types**: "Can values be negative? Are duplicates allowed? Can numbers overflow 32-bit signed integers ($> 2 \cdot 10^9$)?"
3. **Return Expectations**: "Should it return in-place or allocate a new data structure? How should ties or missing targets be reported?"

### 3.2 Step 2: Approach & Complexity Alignment (Minutes 5-10)
- **State the Brute Force First**: "A naive approach checks all pairs in $O(N^2)$ time with $O(1)$ space."
- **Identify Bottleneck**: "The bottleneck is the linear inner lookup. We can trade $O(N)$ space for $O(1)$ lookup using a Hash Map, reducing runtime to $O(N)$."
- **Confirm Before Coding**: *"Does this $O(N)$ time and $O(N)$ space approach sound optimal to you, or would you like me to consider other constraints before I begin implementing?"*

### 3.3 Step 3: Implementation Rules (Minutes 10-32)
- Use descriptive variable names (`window_start`, `current_frequency`, `target_sum`), not single-letter variables (`i`, `j`, `k`, `x`).
- Decompose complex sub-logic into helper functions.
- Guard against null/empty edge cases at the very top of the function.

### 3.4 Step 4: Dry-Run & Edge Case Verification (Minutes 32-40)
Never say *"I'm done"*. Walk through test cases manually by tracing variable states:
1. **Empty / Minimal Input**: `nums = []`, `nums = [1]`.
2. **Extreme Values**: All negative `[-5, -10, -2]`, integer overflow limits (`INT_MAX`, `INT_MIN`).
3. **Duplicates**: `[2, 2, 2, 2, 2]`.
4. **Already Sorted / Reverse Sorted**: Verifies pivot worst-cases.

---

## 4. Senior Interview Q&A Cheatsheet

### Q1: "What is the difference between Auxiliary Space and Total Space Complexity?"
> **Answer**:
> - **Total Space Complexity**: Measures all memory consumed by the algorithm, including input storage.
> - **Auxiliary Space Complexity**: Measures only the **extra or temporary memory** allocated by the algorithm outside the original input.
> - *Example*: In-place QuickSort takes $O(N)$ total space (to store the array of size $N$), but its auxiliary space is $O(\log N)$ on average (recursion call stack frames). An algorithm that creates a duplicate copy of the array has $O(N)$ auxiliary space.

### Q2: "How do you detect whether an algorithm will cause a Call Stack Overflow in production?"
> **Answer**: Every thread in Linux / V8 / CRuby has a fixed stack memory allocation (typically 1MB-8MB). Each nested function call pushes an activation record (stack frame: return address, parameters, local variables, ~48-128 bytes).
> If recursion depth $D$ exceeds ~10,000 to 20,000 frames (e.g. DFS on a skewed linear graph $N = 10^5$), it blows past stack allocation and triggers `StackOverflowError` / segmentation fault.
> **Production Fix**: Convert recursion to an **iterative loop backed by an explicit heap-allocated array stack** (`const stack = [root]`), which is bounded only by available system RAM (gigabytes).
