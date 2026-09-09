# 05. Production Troubleshooting, OS Introspection & Advanced Git

> **Target Role**: Staff / Senior Backend & Infrastructure Engineer  
> **Module**: 08-docker-devops-aws / 05-production-troubleshooting-and-git  
> **Key Focus**: Linux OS Introspection (`strace`, `ptrace`), Network Packet Sniffing (`tcpdump`, Wireshark), Diagnosing AWS SNAT Port Exhaustion, Live Process Profiling (`rbspy`, Node `--inspect` Flamegraphs), Interactive Rebase (`git rebase -i`), Automated Regression Hunting (`git bisect`), and Disaster Recovery (`git reflog`).

---

## Table of Contents
1. [System Call Tracing with strace](#1-system-call-tracing-with-strace)
   - [1.1 Definition & Core Concept](#11-definition--core-concept)
   - [1.2 Internal Mechanics: ptrace Syscall, Context Switches & Trap Handlers](#12-internal-mechanics-ptrace-syscall-context-switches--trap-handlers)
   - [1.3 Production CLI Recipes & Diagnostic Patterns](#13-production-cli-recipes--diagnostic-patterns)
   - [1.4 Production Outages & Debugging](#14-production-outages--debugging)
   - [1.5 Trade-offs & Decision Matrix](#15-trade-offs--decision-matrix)
   - [1.6 Senior Interview Q&A](#16-senior-interview-qa)
2. [Network Packet Sniffing & AWS SNAT Port Exhaustion](#2-network-packet-sniffing--aws-snat-port-exhaustion)
   - [2.1 Definition & Core Concept](#21-definition--core-concept)
   - [2.2 Internal Mechanics: Libpcap, Conntrack, TCP State Machine & Ephemeral Port Limits](#22-internal-mechanics-libpcap-conntrack-tcp-state-machine--ephemeral-port-limits)
   - [2.3 Production tcpdump Scripts & SNAT Diagnostic Workflows](#23-production-tcpdump-scripts--snat-diagnostic-workflows)
   - [2.4 Production Outages & Debugging](#24-production-outages--debugging)
   - [2.5 Trade-offs & Decision Matrix](#25-trade-offs--decision-matrix)
   - [2.6 Senior Interview Q&A](#26-senior-interview-qa)
3. [Live Production Profiling & Flamegraphs (Ruby & Node.js)](#3-live-production-profiling--flamegraphs-ruby--nodejs)
   - [3.1 Definition & Core Concept](#31-definition--core-concept)
   - [3.2 Internal Mechanics: Sampling Stack Traces vs Instrumentation & V8 Inspector](#32-internal-mechanics-sampling-stack-traces-vs-instrumentation--v8-inspector)
   - [3.3 Production Profiling Recipes (rbspy & Node --inspect)](#33-production-profiling-recipes-rbspy--node---inspect)
   - [3.4 Production Outages & Debugging](#34-production-outages--debugging)
   - [3.5 Trade-offs & Decision Matrix](#35-trade-offs--decision-matrix)
   - [3.6 Senior Interview Q&A](#36-senior-interview-qa)
4. [Advanced Git Mastery: Interactive Rebase & Commit Hygiene](#4-advanced-git-mastery-interactive-rebase--commit-hygiene)
   - [4.1 Definition & Core Concept](#41-definition--core-concept)
   - [4.2 Internal Mechanics: The Git Directed Acyclic Graph (DAG) & Cherry-Pick Engine](#42-internal-mechanics-the-git-directed-acyclic-graph-dag--cherry-pick-engine)
   - [4.3 Production Interactive Rebase Workflow](#43-production-interactive-rebase-workflow)
   - [4.4 Production Outages & Debugging](#44-production-outages--debugging)
   - [4.5 Trade-offs & Decision Matrix](#45-trade-offs--decision-matrix)
   - [4.6 Senior Interview Q&A](#46-senior-interview-qa)
5. [Automated Regression Hunting & Recovery (git bisect & git reflog)](#5-automated-regression-hunting--recovery-git-bisect--git-reflog)
   - [5.1 Definition & Core Concept](#51-definition--core-concept)
   - [5.2 Internal Mechanics: Binary Search Math & The Reflog Journal Architecture](#52-internal-mechanics-binary-search-math--the-reflog-journal-architecture)
   - [5.3 Production Automated Bisect Script & Reflog Recovery](#53-production-automated-bisect-script--reflog-recovery)
   - [5.4 Production Outages & Debugging](#54-production-outages--debugging)
   - [5.5 Trade-offs & Decision Matrix](#55-trade-offs--decision-matrix)
   - [5.6 Senior Interview Q&A](#56-senior-interview-qa)

---

# 1. System Call Tracing with strace

### 1.1 Definition & Core Concept
Every user-space application (Ruby, Node.js, Python, Go) must communicate with the Linux kernel to perform I/O operations (reading files, opening network sockets, accepting HTTP requests, querying clocks).
`strace` is an OS-level diagnostic utility that intercepts and records the system calls called by a process and the signals received by the process. It is the ultimate diagnostic weapon for troubleshooting **hung, deadlocked, or zombie processes** where logs are silent.

---

### 1.2 Internal Mechanics: ptrace Syscall, Context Switches & Trap Handlers

```
+-----------------------------------------------------------------------------+
|                                strace (Tracer)                              |
+-----------------------------------------------------------------------------+
                                       │
                                       │ ptrace(PTRACE_SYSCALL, pid, ...)
                                       ▼
+-----------------------------------------------------------------------------+
|                                 Linux Kernel                                |
|   - Sets TIF_SYSCALL_TRACE flag on process task_struct                      |
|   - Intercepts SYSENTER / SYSCALL trap                                      |
|   - Halts target process -> Sends SIGTRAP to strace                         |
|   - Wakes up strace -> strace reads registers (%rax, %rdi)                  |
|   - Kernel resumes target process                                           |
+-----------------------------------------------------------------------------+
                                       │
                                       │ Intercepts Syscall Entry & Syscall Exit
                                       ▼
+-----------------------------------------------------------------------------+
|                          Target Process (Puma / Node)                       |
|   Executing: read(fd, buf, count)                                           |
+-----------------------------------------------------------------------------+
```

1. **The `ptrace(2)` Mechanism**: `strace` attaches to the target process using `ptrace(PTRACE_ATTACH, pid)`. The kernel sets the `TIF_SYSCALL_TRACE` flag in the target thread's `task_struct`.
2. **Double Trap Overhead**: For *every single system call*, the target process is halted twice:
   - Once before the syscall executes (inspecting arguments).
   - Once after the syscall finishes (inspecting return value).
3. **Production Overhead Warning**: `strace` can degrade application throughput by 10x to 100x. **Never run `strace` indiscriminately across an entire production cluster; attach only to a single isolated worker PID**.

---

### 1.3 Production CLI Recipes & Diagnostic Patterns

#### 1. Tracing a Frozen Process with Timing & Profiling
```bash
# 1. Profile syscall execution time and count (Aggregated summary)
# -c: Count time and calls per syscall
# -p: Target PID
strace -c -p 18492
# Output on Ctrl+C:
# % time     seconds  usecs/call     calls    errors syscall
# ------ ----------- ----------- --------- --------- ----------------
#  94.20    1.420100      710050         2           epoll_wait
#   5.80    0.087400         145       602           read
# ------ ----------- ----------- --------- --------- ----------------
# 100.00    1.507500                   604           total

# 2. Trace slow system calls with microsecond timestamps and duration
# -t: Wall-clock time
# -T: Show time spent inside each system call
# -e: Filter specific syscall family (network, file, etc.)
# -s: String truncation limit (default 32 bytes -> increase to 512)
strace -t -T -s 512 -e trace=network -p 18492
# Output:
# 14:02:11.102304 connect(9, {sa_family=AF_INET, sin_port=htons(5432), sin_addr=inet_addr("10.0.10.45")}, 16) = -1 EINPROGRESS <0.000120>
# 14:02:11.102500 epoll_wait(4, [{events=EPOLLOUT, data={u32=9, u64=9}}], 1024, 5000) = 1 <4.998200>
# Analysis: Application spent 4.99 seconds blocked waiting for PostgreSQL socket to become writable!
```

#### 2. Identifying File Descriptor Leaks & Blocked Sockets
```bash
# Filter purely for file open/close and network operations
strace -f -e trace=open,openat,close,socket,connect -p $(pgrep -f "puma")
```

---

### 1.4 Production Outages & Debugging

#### Outage Case Study: The Silent `epoll_wait` Database Lockup
- **Context**: A Rails worker thread hung indefinitely at 100% CPU. Application logs showed no activity.
- **Investigation**:
  1. Attached `strace`:
     ```bash
     sudo strace -p 4821 -T
     ```
  2. Observed:
     ```
     read(14, 0x55d0f128c, 8192) = -1 EAGAIN (Resource temporarily unavailable) <0.000012>
     futex(0x7f9a2b, FUTEX_WAIT_BITSET_PRIVATE, 2, NULL, ...) ... [BLOCKED]
     ```
  3. The process was trapped in a **Pthread Futex Deadlock** caused by a third-party C-extension holding a mutex across a forked process boundary.
- **Resolution**: Identified the offending gem (`mini_racer`), enabled fork-safety flags, and eliminated thread deadlocks.

---

### 1.5 Trade-offs & Decision Matrix

| Diagnostic Tool | Kernel Mechanism | Overhead | Granularity | Production Safety |
|:---|:---|:---|:---|:---|
| **`strace`** | `ptrace` system call traps | **Extremely High (10x-100x slowdown)** | Every single system call | Use on single PID only during incidents. |
| **`perf`** | Hardware CPU performance counters | Minimal (<1%) | CPU instruction / stack sampling | Safe for continuous production profiling. |
| **eBPF (`bpftrace`)** | Kernel BPF JIT hooks | Near-Zero (<0.5%) | Filtered kernel event aggregation | **Modern Production Standard**. |

---

### 1.6 Senior Interview Q&A

#### Q: Why does running `strace` on a high-throughput multi-threaded Node.js or Puma process sometimes cause timeouts in upstream load balancers?
**Answer**:
`strace` relies on `ptrace(2)`. Every time a thread executes a system call (e.g., `epoll_wait`, `read`, `write`, `gettimeofday`), the Linux kernel pauses the thread, forces a context switch to user-space `strace`, writes data, and then context-switches back to resume the application thread.
In a multi-threaded service handling 3,000 req/sec, tens of thousands of syscalls occur each second. The CPU context switches skyrocket from hundreds per second to over 100,000 per second. This saturates the CPU cache, stalls the application's event loop/worker threads, and inflates $p99$ response times from 10ms to >5,000ms, triggering upstream AWS ALB 504 Gateway Timeouts.

---

# 2. Network Packet Sniffing & AWS SNAT Port Exhaustion

### 2.1 Definition & Core Concept
When distributed applications encounter obscure connection resets (`ECONNRESET`), socket timeouts, or intermittent DNS failures, log files only show the application's perspective.
`tcpdump` is a command-line packet analyzer utilizing `libpcap` to capture raw frames traversing Linux network interfaces (`eth0`, `veth`, `docker0`).
- **AWS SNAT Port Exhaustion**: A classic cloud failure where an application initiating thousands of concurrent outbound API connections through an **AWS NAT Gateway** exhausts the pool of 55,000 available ephemeral source ports, causing random 5-second connection drops.

```
+---------------------------------------------------------------------------------+
|                   VPC PRIVATE SUBNET (Hundreds of Containers)                   |
|   App Instance 1 (10.0.10.12)  ──────┐                                          |
|   App Instance 2 (10.0.10.15)  ──────┼──► Concurrent Outbound HTTPS Calls       |
|   App Instance 3 (10.0.10.18)  ──────┘    (e.g., Stripe, SendGrid, Auth0)       |
+---------------------------------------------------------------------------------+
                                       │
                                       ▼
+---------------------------------------------------------------------------------+
|                        AWS NAT GATEWAY (Single Public IP)                       |
|   - Translates Private IPs -> Public IP (54.21.9.12)                            |
|   - Ephemeral Source Port Pool: Exactly 55,000 ports per destination IP:port   |
|   ===========================================================================   |
|   == CONCURRENT OUTBOUND CONNECTIONS EXCEED 55,000!                         ==   |
|   == SNAT PORT EXHAUSTION OCCURS: PACKETS SILENTLY DROPPED BY AWS!          ==   |
|   ===========================================================================   |
+---------------------------------------------------------------------------------+
                                       │ (Packets Dropped)
                                       ▼
                                 PUBLIC INTERNET
```

---

### 2.2 Internal Mechanics: Libpcap, Conntrack, TCP State Machine & Ephemeral Port Limits

#### 1. The TCP 3-Way Handshake at Packet Level
When an application connects to `api.stripe.com:443`:
1. **Client -> Server `[SYN]`**: Sequence number $X$.
2. **Server -> Client `[SYN, ACK]`**: Acknowledges $X+1$, sends sequence $Y$.
3. **Client -> Server `[ACK]`**: Acknowledges $Y+1$. Connection is `ESTABLISHED`.

#### 2. The SNAT Port Math & Connection Reuse
A TCP connection is uniquely identified by the **5-Tuple**:
$$\text{Tuple} = \{\text{Source IP}, \text{Source Port}, \text{Destination IP}, \text{Destination Port}, \text{Protocol}\}$$
- The AWS NAT Gateway has **1 Public Elastic IP**.
- When communicating with a single remote endpoint (e.g., `api.stripe.com` on IP `52.20.10.5` port `443`):
  - Destination IP is fixed (`52.20.10.5`).
  - Destination Port is fixed (`443`).
  - Protocol is fixed (`TCP`).
  - Source IP is fixed (NAT Gateway IP).
- **The only variable is the Source Port**.
- The Linux kernel limits ephemeral source ports to range `1024` to `65535` ($\approx 55,000$ concurrent ports).
- If microservices make $>55,000$ concurrent requests without HTTP Keep-Alive connection pooling, or close connections rapidly leaving them in `TIME_WAIT` (default: 60s), **all 55,000 ports are consumed**.
- NAT Gateway silently drops subsequent `SYN` packets until a port frees up, triggering client-side 5-second connection timeouts.

---

### 2.3 Production tcpdump Scripts & SNAT Diagnostic Workflows

#### 1. Capturing Raw Traffic on Linux Interface
```bash
# Capture HTTPS traffic on eth0, capture full packet (-s 0), write to file
sudo tcpdump -i eth0 -nn -s 0 -w /tmp/traffic_dump.pcap "tcp port 443"

# Inspect packet capture directly on server terminal
tcpdump -r /tmp/traffic_dump.pcap -nn "tcp[tcpflags] & (tcp-syn|tcp-rst) != 0"
# Look for TCP RST (Connection Refused / Aborted):
# 14:15:02.192038 IP 10.0.10.12.54128 > 52.20.10.5.443: Flags [S], seq 3829104
# 14:15:02.194510 IP 52.20.10.5.443 > 10.0.10.12.54128: Flags [R.], seq 0, ack 3829105 (RST packet received!)
```

#### 2. Diagnosing AWS SNAT Port Exhaustion in CloudWatch
Query the following AWS CloudWatch metrics for the NAT Gateway:
- `ErrorPortAllocation`: The number of times the NAT Gateway could not allocate an ephemeral source port. **If this metric is $>0$, you have active SNAT port exhaustion!**
- `PacketsDropCount`: Number of packets dropped by the NAT Gateway due to allocation limits.

---

### 2.4 Production Outages & Debugging

#### Outage Case Study: The 3rd-Party Webhook SNAT Exhaustion Cascade
- **Context**: An e-commerce platform dispatched webhooks to a single partner's endpoint (`hooks.partner.com:443`). During Black Friday, 120,000 webhooks were fired within 2 minutes.
- **Symptom**: Suddenly, **ALL outbound internet traffic failed across the entire company**: Stripe payments timed out, Slack alerts failed, AWS DynamoDB calls stalled.
- **Root Cause**: The webhook dispatcher opened a fresh TCP/TLS socket for every single HTTP POST without connection reuse (`keep-alive: false`). All 55,000 ephemeral ports on the single NAT Gateway were locked in `TIME_WAIT`. Stripe and all other outbound traffic traversing the same NAT Gateway were starved of ports.
- **Remediation**:
  1. Enabled **HTTP Keep-Alive Connection Pooling** in the Node.js/Ruby HTTP client:
     ```typescript
     import https from 'https';
     import axios from 'axios';

     // Reuse TCP sockets across requests!
     const agent = new https.Agent({
       keepAlive: true,
       maxSockets: 100,
       maxFreeSockets: 10,
       timeout: 60000,
     });

     const httpClient = axios.create({ httpsAgent: agent });
     ```
  2. For AWS VPC architecture: Attached multiple Elastic IPs to the NAT Gateway (or provisioned a secondary NAT Gateway) to expand ephemeral port pools.

---

### 2.5 Trade-offs & Decision Matrix

| Outbound Architecture | SNAT Limit per Endpoint | Cost | Maintenance | Production Recommendation |
|:---|:---|:---|:---|:---|
| **Single NAT Gateway** | 55,000 concurrent sockets | Cheap (~$32/mo + data) | Low | Dev/Staging only. |
| **Multi-AZ NAT Gateways (3x)** | $3 \times 55,000 = 165,000$ | Moderate (~$100/mo) | Low | **Production Baseline**. |
| **VPC Endpoints (PrivateLink)** | **UNLIMITED (Bypasses NAT!)** | $0.01/hr per AZ | Zero | **Mandatory** for S3, DynamoDB, ECR, SQS, Secrets Manager. |

---

### 2.6 Senior Interview Q&A

#### Q: Why should AWS services (like S3 and DynamoDB) NEVER route through an AWS NAT Gateway?
**Answer**:
1. **Cost Inefficiency**: AWS NAT Gateways charge **$0.045 per GB of data processed** in addition to the base hourly rate. Routing massive S3 database backups or video uploads through a NAT Gateway generates astronomical bandwidth bills.
2. **SNAT Exhaustion Risk**: High-volume parallel S3 or DynamoDB requests burn ephemeral ports on the NAT Gateway, risking port exhaustion for mission-critical external APIs (Stripe, payment gateways).
3. **The Solution (VPC Gateway Endpoints)**: AWS provides **Free VPC Gateway Endpoints** for S3 and DynamoDB. Enabling a Gateway Endpoint updates the VPC subnet route tables to route all S3/DynamoDB traffic directly over the AWS private fiber backbone, achieving near-zero latency, zero data processing fees, and bypassing the NAT Gateway completely.

---

# 3. Live Production Profiling & Flamegraphs (Ruby & Node.js)

### 3.1 Definition & Core Concept
When an application exhibits 100% CPU utilization or excessive memory consumption in production, traditional APM metrics only tell you *that* the server is slow, not *which line of code* is consuming CPU cycles.
- **Flamegraphs**: Visual representation of hierarchical call stacks where the x-axis represents the percentage of total CPU time spent in each function, and the y-axis represents call stack depth.
- **Non-Invasive Profilers (`rbspy`)**: Read process memory externally without modifying code or stopping the runtime.

```
+-----------------------------------------------------------------------------+
|                              FLAMEGRAPH VISUALIZATION                       |
|                                                                             |
|   +---------------------------------------------------------------------+   |
|   |                     Rack::Sendfile#call (100%)                      |   |
|   +---------------------------------------------------------------------+   |
|   |                 ActionController::Base#process (92%)                |   |
|   +---------------------------------------------------------------------+   |
|   |         OrderController#show (85%)         | Other (7%)             |   |
|   +--------------------------------------------+------------------------+   |
|   | ActiveSupport::JSON.decode (78%) [WIDE!!]  | DB Query (7%)          |   |
|   +--------------------------------------------+------------------------+   |
|   == WIDE BLOCK: ActiveSupport::JSON.decode burns 78% of all CPU time! ==   |
+-----------------------------------------------------------------------------+
```

---

### 3.2 Internal Mechanics: Sampling Stack Traces vs Instrumentation & V8 Inspector

#### 1. Sampling Profiling vs. Instrumentation
- **Instrumentation (e.g., Ruby `set_trace_func`)**: Hooks into every single method entry and exit. Introduces massive runtime overhead (3x-10x slowdown); unacceptable in production.
- **Statistical Sampling (`rbspy`)**: Reads the target process's memory space via `process_vm_readv(2)` at fixed intervals (e.g., 100 Hz = 100 times per second). Samples take $<10\mu s$ of CPU time, inducing $<1\%$ overhead.

#### 2. Node.js V8 Profiler Mechanics
Node.js exposes V8's internal sampling profiler via the V8 Inspector protocol (`--inspect` or `v8-profiler-next`). V8 uses an OS timer thread (`SIGPROF`) to record the instruction pointer (`%rip`) and maps it back to JavaScript function closures and source lines.

---

### 3.3 Production Profiling Recipes (rbspy & Node --inspect)

#### 1. Profiling Live Production Ruby Processes with `rbspy`
```bash
# 1. Install rbspy binary on production host
sudo curl -sL https://github.com/rbspy/rbspy/releases/latest/download/rbspy-x86_64-unknown-linux-musl.tar.gz | sudo tar -xz -C /usr/local/bin

# 2. Record 30 seconds of CPU samples from a running Puma worker without restarting it
sudo rbspy record --pid $(pgrep -f "puma.*worker: 0") --duration 30 --file /tmp/ruby_profile.flamegraph.svg

# 3. Download /tmp/ruby_profile.flamegraph.svg and open in browser
```

#### 2. Profiling Live Node.js Microservices On-Demand
```typescript
import v8Profiler from 'v8-profiler-next';
import fs from 'fs';

// Expose secure internal diagnostic endpoint to trigger a 10-second CPU flamegraph
export function setupProfilingRoute(app: any) {
  app.post('/admin/diagnostics/cpu-profile', async (req: any, res: any) => {
    const title = `profile-${Date.now()}`;
    v8Profiler.startProfiling(title, true);

    setTimeout(() => {
      const profile = v8Profiler.stopProfiling(title);
      profile.export((error, result) => {
        fs.writeFileSync(`/tmp/${title}.cpuprofile`, result!);
        profile.delete();
        res.json({ status: 'completed', path: `/tmp/${title}.cpuprofile` });
      });
    }, 10000); // 10 seconds sample window
  });
}
```

---

### 3.4 Production Outages & Debugging

#### Outage Case Study: The Inefficient Regex CPU Spike
- **Context**: Following a release, CPU utilization hit 100% on 20 Node.js tasks. Autoscaling maxed out, and responses timed out.
- **Diagnosis**:
  1. Triggered a 10-second V8 CPU profile.
  2. Imported the `.cpuprofile` into Chrome DevTools (Performance -> Load Profile).
  3. The Flamegraph displayed an overwhelmingly wide block: `RegExp.exec` inside an email validation function accounting for 91.4% of all CPU ticks.
  4. Root Cause: **Catastrophic Backtracking (ReDoS)** on nested quantifiers: `^([a-zA-Z0-9_.-]+)+@...`.
- **Resolution**: Replaced the nested regex with a linear-time parser, instantly dropping CPU usage from 100% to 4%.

---

### 3.5 Trade-offs & Decision Matrix

| Profiler | Target Runtime | Overhead | Invasive? | Production Suitability |
|:---|:---|:---|:---|:---|
| **`rbspy`** | Ruby (MRI) | < 1% | No (External memory reader) | **Gold Standard for Rails in production**. |
| **`v8-profiler-next`** | Node.js | < 2% | Minimal (C++ addon triggered via API) | **Gold Standard for production Node.js**. |
| **Chrome DevTools `--inspect`** | Node.js | High (Keeps open WebSocket) | Yes (Exposes debug port) | Avoid in public prod; bind to `127.0.0.1` only. |

---

### 3.6 Senior Interview Q&A

#### Q: How does `rbspy` read the call stack of a Ruby process without attaching a debugger or pausing the thread?
**Answer**:
`rbspy` leverages the Linux kernel system call `process_vm_readv(2)`.
1. It reads the virtual memory space of the target process without interrupting its CPU execution.
2. It locates the global `rb_current_execution_context_struct` in the Ruby VM binary's symbols.
3. It resolves the `ec->cfp` (Control Frame Pointer), which points to the current Ruby stack frame.
4. It iterates backward through the linked list of `rb_control_frame_struct` structures in memory, extracting method names, class names, and line numbers.
5. Because memory reading happens in microseconds via direct kernel page mapping, the target Ruby process continues executing at full speed with virtually zero performance impact.

---

# 4. Advanced Git Mastery: Interactive Rebase & Commit Hygiene

### 4.1 Definition & Core Concept
In enterprise engineering organizations, Git history is not merely an automated log of file changes; it is a **curated, searchable historical documentation audit trail**.
- **Interactive Rebase (`git rebase -i`)**: Allows engineers to rewrite commit history before merging to `main`: squashing messy "WIP" or "fixed typo" commits, rewording commit messages to follow conventional commits (`feat:`, `fix:`), and splitting monolithic changes into atomic units.

---

### 4.2 Internal Mechanics: The Git Directed Acyclic Graph (DAG) & Cherry-Pick Engine

```
BEFORE INTERACTIVE REBASE:
[Commit A] ──► [Commit B: "WIP"] ──► [Commit C: "fix bug"] ──► [Commit D: "add tests"]
                                                                      ▲
                                                                      │ (HEAD)

REBASE INSTRUCTION LIST:
pick A
pick B "WIP"
fixup C "fix bug"   <── Folds C into B, discarding C's message
squash D "add tests" <── Folds D into B, combining messages

AFTER INTERACTIVE REBASE (Clean Linear History):
[Commit A] ───────────────────────► [Commit B': "feat(auth): add OAuth2 provider with tests"]
                                                                      ▲
                                                                      │ (HEAD)
```

1. Git identifies the common ancestor commit (`BASE`).
2. Git unwinds the commits on the current branch back to `BASE`.
3. It reads the interactive instruction list and executes `git cherry-pick` sequentially for each line.
4. For `squash` or `fixup`, it merges the tree objects and amends the commit object.
5. **The Golden Rule of Rebasing**: **Never rebase commits that have already been pushed and shared on public release branches (`main`, `master`, `production`)**. Rebasing alters cryptographic commit SHAs; rewriting public history forces teammate repositories out of sync.

---

### 4.3 Production Interactive Rebase Workflow

```bash
# Rebase the last 4 commits on the current branch
git rebase -i HEAD~4

# Git opens your editor ($EDITOR) with the rebase instruction sheet:
# ------------------------------------------------------------------------------
# Commands:
# p, pick <commit> = use commit
# r, reword <commit> = use commit, but edit the commit message
# e, edit <commit> = use commit, but stop for amending
# s, squash <commit> = use commit, but meld into previous commit
# f, fixup <commit> = like "squash", but discard this commit's log message
# d, drop <commit> = remove commit
# ------------------------------------------------------------------------------

# Modify the instructions to clean up messy WIPs:
pick 7a1b0cd feat(billing): add stripe webhook handler
fixup 8f2c3de fix typo in webhook secret parser
squash 9b4d1ef add unit tests for webhook signature validation

# Save and exit. Git automatically collapses the 3 commits into 1 atomic commit!

# Push updated branch to remote (Requires --force-with-lease for safety):
git push origin feature/stripe-webhooks --force-with-lease
```

> [!CAUTION]
> **Always use `--force-with-lease` instead of `--force`!**  
> `--force` unconditionally overwrites the remote branch. If a teammate pushed a commit to your branch while you were rebasing, `--force` will destroy their work permanently.  
> `--force-with-lease` checks if the remote ref matches your local tracking ref; if anyone else pushed to the branch, the push is safely rejected.

---

### 4.4 Production Outages & Debugging

#### Outage Case Study: The Merge Conflict Resolution Loop
- **Context**: During an interactive rebase of a 20-commit branch against `main`, an engineer encountered 15 consecutive merge conflicts, accidentally introducing a regression by resolving an import conflict incorrectly.
- **Staff Remediation Technique**:
  1. Abort the messy rebase immediately:
     ```bash
     git rebase --abort
     ```
  2. Use `git rerere` (Reuse Recorded Resolution) to automate recurring conflict resolutions:
     ```bash
     git config --global rerere.enabled true
     ```
  3. Squash the feature branch into logical atomic commits *before* rebasing against `main`. Resolving conflicts against 2 clean commits is drastically safer than resolving against 20 micro-commits.

---

### 4.5 Trade-offs & Decision Matrix

| Strategy | History Cleanliness | Branch Representation | Ease of Bisecting | Production Suitability |
|:---|:---|:---|:---|:---|
| **Merge Commits (`git merge`)** | Cluttered with non-linear merge bubbles | Preserves exact branch topology | Difficult (Reverts are non-trivial) | Open-source multi-contributor projects. |
| **Squash and Merge (GitHub PR)** | Perfectly linear; 1 commit per PR | Loses individual commit details | **Exceptional** | **Standard for microservices & web products**. |
| **Rebase and Merge** | Linear; preserves all curated atomic commits | Linear | Excellent | Standard for systems software & large monorepos. |

---

### 4.6 Senior Interview Q&A

#### Q: What is the difference between `git merge --squash` and `git rebase -i`?
**Answer**:
- `git merge --squash <feature-branch>` takes all the changes on the feature branch, stages them into the working index on top of the current branch as a single uncommitted change set, and does not preserve individual commit authors or history.
- `git rebase -i` provides fine-grained, selective control over the commit DAG. It allows you to squash some commits while keeping others separate, edit commit messages individually, reorder commits, or drop specific intermediate commits entirely.

---

# 5. Automated Regression Hunting & Recovery (git bisect & git reflog)

### 5.1 Definition & Core Concept
- **`git bisect`**: A debugging tool that uses a **Binary Search ($O(\log N)$)** algorithm through the commit history to find the exact commit that introduced a bug.
- **`git reflog`**: A chronological journal recording every movement of the `HEAD` pointer on your local machine (commits, checkouts, rebases, resets, pulls). It serves as the ultimate safety net to recover seemingly "lost" commits or deleted branches.

---

### 5.2 Internal Mechanics: Binary Search Math & The Reflog Journal Architecture

#### 1. Binary Search Math in `git bisect`
If a bug was introduced somewhere between a known good release (1,000 commits ago) and the current bad release:
- Sequential inspection: Requires testing up to 1,000 commits.
- Binary search:
  $$\text{Steps} = \lceil \log_2(1000) \rceil \approx 10\text{ steps}$$
Testing only 10 commits isolates the exact breaking commit.

#### 2. The Reflog Journal Architecture
In Git, branches are merely lightweight 41-byte pointer files in `.git/refs/heads/`.
When you run `git reset --hard HEAD~5`, Git does not delete the commit objects. The commits remain in `.git/objects/`. Only the pointer moves.
The **Reflog** (`.git/logs/HEAD`) records every update to `HEAD`:
```
c4f1e0a HEAD@{0}: reset: moving to HEAD~5
9b4d1ef HEAD@{1}: commit: feat(checkout): add payment gateway
7a1b0cd HEAD@{2}: commit: fix(db): add missing index
```
Until Git runs garbage collection (`git gc`, default 30-90 days), any unreferenced commit recorded in the reflog can be completely recovered.

---

### 5.3 Production Automated Bisect Script & Reflog Recovery

#### 1. Fully Automated `git bisect run` with Test Script
Instead of manually checking out commits and testing, write an automated test script and let Git find the bug autonomously:

```bash
# Start bisect session
git bisect start

# Mark current HEAD as bad
git bisect bad HEAD

# Mark known working release tag as good
git bisect good v2.1.0
# Output: Bisecting: 256 revisions left to test after this (roughly 8 steps)

# Execute automated binary search using exit code of test runner!
# Convention: Exit 0 = GOOD, Exit 1-127 = BAD, Exit 125 = SKIP commit (unbuildable)
git bisect run npm test -- tests/unit/payment-calculator.test.ts

# Git automatically checks out intermediate commits, runs the test suite,
# and terminates with the pinpointed guilty commit:
# ------------------------------------------------------------------------------
# 8f2c3de14a2b is the first bad commit
# Author: Jane Dev <jane@enterprise.com>
# Date:   Wed Sep 2 14:22:01 2026 -0400
#     refactor(math): alter rounding logic for tax calculations
# ------------------------------------------------------------------------------

# Reset repository back to original HEAD
git bisect reset
```

#### 2. Disaster Recovery via `git reflog`
**Scenario**: An engineer ran `git reset --hard origin/main` on the wrong terminal, wiping out 3 days of unpushed commits.

```bash
# 1. Inspect the chronological HEAD journal
git reflog
# Output:
# 4a7c1b8 HEAD@{0}: reset: moving to origin/main
# 8f9b2c3 HEAD@{1}: commit: feat(reporting): complete financial audit report (LOST!)
# 1d2e3f4 HEAD@{2}: commit: wip on reporting queries

# 2. Instantly recover the lost commit state into a new emergency branch
git checkout -b recovery-branch HEAD@{1}

# 3. Verify files: ALL WORK IS 100% RESTORED!
git log -n 1
```

---

### 5.4 Production Outages & Debugging

#### Outage Case Study: The Intermittent Flaky Commit in Bisect
- **Context**: During a `git bisect run`, the test suite failed on an intermediate commit because an unrelated npm dependency broke the build, causing bisect to falsely mark the commit as "bad".
- **Resolution**: Utilized **Exit Code 125**:
  ```bash
  # bisect-runner.sh
  npm install
  if [ $? -ne 0 ]; then
    # Cannot build this commit -> Tell Git to skip it!
    exit 125
  fi
  npm test -- tests/core.test.ts
  ```
  Passing exit code 125 instructs `git bisect` to pick an adjacent commit without marking the current commit as good or bad.

---

### 5.5 Trade-offs & Decision Matrix

| Tool / Recovery Method | Time to Pinpoint / Recover | Skill Level Required | Risk of Data Loss | Production Guidance |
|:---|:---|:---|:---|:---|
| **Manual Git Log Hunting** | Hours / Days | Low | Zero | Inefficient for large codebases. |
| **`git bisect run <script>`** | **Seconds to Minutes** | Moderate | Zero | **Mandatory** for isolating regressions across releases. |
| **`git reflog`** | **< 30 Seconds** | High | Zero | **Primary recovery safety net** for local terminal accidents. |

---

### 5.6 Senior Interview Q&A

#### Q: If you accidentally run `git branch -D feature-branch` and you have not pushed it to remote, how do you recover the deleted branch?
**Answer**:
1. When a branch is deleted via `git branch -D`, Git merely deletes the reference pointer file under `.git/refs/heads/feature-branch`. The commit objects and tree blobs remain untouched in the `.git/objects/` database.
2. Run `git reflog` or `git reflog show HEAD`.
3. Locate the last commit SHA that was made on that branch before deletion (e.g., `checkout: moving from feature-branch to main` or the last commit message).
4. Recreate the branch pointing to that commit SHA:
   ```bash
   git checkout -b feature-branch <COMMIT_SHA>
   ```
5. The branch and all its commits are completely restored.
