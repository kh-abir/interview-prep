# 01. Docker Internals, Image Engineering & Production Recipes

> **Target Role**: Staff / Senior Backend & Infrastructure Engineer  
> **Module**: 08-docker-devops-aws / 01-docker-internals-and-recipes  
> **Key Focus**: Linux Kernel Primitives (Namespaces, Cgroups v1/v2, OverlayFS, Seccomp, Capabilities), OCI Runtimes (`containerd`, `runc`), Image Optimization, Container Networking & Storage, Production Multi-Stage Dockerfiles (Rails & Next.js).

---

## Table of Contents
1. [Linux Primitives: The Foundation of Containers](#1-linux-primitives-the-foundation-of-containers)
   - [1.1 Definition & Core Concept](#11-definition--core-concept)
   - [1.2 Internal Mechanics: Kernel Subsystems & Syscalls](#12-internal-mechanics-kernel-subsystems--syscalls)
   - [1.3 Production Shell Commands & C Program Verification](#13-production-shell-commands--c-program-verification)
   - [1.4 Production Outages & Debugging](#14-production-outages--debugging)
   - [1.5 Trade-offs & Decision Matrix](#15-trade-offs--decision-matrix)
   - [1.6 Senior Interview Q&A](#16-senior-interview-qa)
2. [Container Runtime Architecture & Lifecycle States](#2-container-runtime-architecture--lifecycle-states)
   - [2.1 Definition & Core Concept](#21-definition--core-concept)
   - [2.2 Internal Mechanics: Docker Daemon to containerd to runc](#22-internal-mechanics-docker-daemon-to-containerd-to-runc)
   - [2.3 Production CLI & Inspection Workflows](#23-production-cli--inspection-workflows)
   - [2.4 Production Outages & Debugging](#24-production-outages--debugging)
   - [2.5 Trade-offs & Decision Matrix](#25-trade-offs--decision-matrix)
   - [2.6 Senior Interview Q&A](#26-senior-interview-qa)
3. [Image Engineering, Caching & Container Security](#3-image-engineering-caching--container-security)
   - [3.1 Definition & Core Concept](#31-definition--core-concept)
   - [3.2 Internal Mechanics: OCI Image Spec, Merkle Trees & Layer Hashes](#32-internal-mechanics-oci-image-spec-merkle-trees--layer-hashes)
   - [3.3 Production Dockerfile Optimization Patterns](#33-production-dockerfile-optimization-patterns)
   - [3.4 Production Outages & Debugging](#34-production-outages--debugging)
   - [3.5 Trade-offs & Decision Matrix](#35-trade-offs--decision-matrix)
   - [3.6 Senior Interview Q&A](#36-senior-interview-qa)
4. [Container Networking & Storage Architecture](#4-container-networking--storage-architecture)
   - [4.1 Definition & Core Concept](#41-definition--core-concept)
   - [4.2 Internal Mechanics: Veth Pairs, Iptables NAT & Storage Drivers](#42-internal-mechanics-veth-pairs-iptables-nat--storage-drivers)
   - [4.3 Production Docker Compose Patterns & Healthchecks](#43-production-docker-compose-patterns--healthchecks)
   - [4.4 Production Outages & Debugging](#44-production-outages--debugging)
   - [4.5 Trade-offs & Decision Matrix](#45-trade-offs--decision-matrix)
   - [4.6 Senior Interview Q&A](#46-senior-interview-qa)
5. [Staff-Level Production Dockerfiles (Rails & Next.js)](#5-staff-level-production-dockerfiles-rails--nextjs)
   - [5.1 Definition & Core Concept](#51-definition--core-concept)
   - [5.2 Internal Mechanics: Asset Compilers, Dynamic Linkers & Memory Allocators](#52-internal-mechanics-asset-compilers-dynamic-linkers--memory-allocators)
   - [5.3 Production Multi-Stage Implementations](#53-production-multi-stage-implementations)
   - [5.4 Production Outages & Debugging](#54-production-outages--debugging)
   - [5.5 Trade-offs & Decision Matrix](#55-trade-offs--decision-matrix)
   - [5.6 Senior Interview Q&A](#56-senior-interview-qa)

---

# 1. Linux Primitives: The Foundation of Containers

### 1.1 Definition & Core Concept
A "container" does not exist as a physical or standalone kernel entity in Linux. There is no `container_struct` inside the Linux kernel. A container is simply a standard Linux process governed by three foundational kernel technologies:
1. **Linux Namespaces**: Partition and virtualize kernel resources, giving the process the illusion of owning its own dedicated system (processes, network adapters, mount points, users).
2. **Control Groups (cgroups)**: Meter, throttle, and bound hardware resource consumption (CPU, RAM, block I/O, network bandwidth, process count).
3. **Union Mount Filesystems (OverlayFS)**: Stack multiple directories into a single unified view, enabling layered images with copy-on-write (CoW) mechanics.

Security boundaries are reinforced by **Seccomp** (restricting system calls that the process can execute) and **Linux Capabilities** (slicing root privilege `UID 0` into distinct granular permissions).

```
+---------------------------------------------------------------------------------+
|                               Container Process                                 |
|               (e.g., Ruby Puma worker / Node.js Express server)                 |
+---------------------------------------------------------------------------------+
       |                              |                              |
       v                              v                              v
+------------------+       +----------------------+       +-----------------------+
|    NAMESPACES    |       |       CGROUPS        |       |       SECURITY        |
|  - PID (PIDs)    |       |  - cpu.max (Throttl) |       |  - Seccomp Filter     |
|  - NET (IP/veth) |       |  - memory.max (OOM)  |       |  - Cap drop (SYS_ADM) |
|  - MNT (Rootfs)  |       |  - memory.current    |       |  - AppArmor / SELinux |
|  - UTS (Hostname)|       |  - io.weight (Disk)  |       +-----------------------+
|  - IPC (Shmem)   |       |  - pids.max (Forkbmb)|                   |
|  - USER (UID/GID)|       +----------------------+                   v
|  - CGROUP (Tree) |                  |                   +-----------------------+
+------------------+                  v                   |       OverlayFS       |
       |                   +----------------------+       | [Upperdir] Read-Write |
       +------------------>|     Linux Kernel     |<------+ [Workdir]  Internal   |
                           |   x86-64 Hardware    |       | [Lowerdir] Read-Only  |
                           +----------------------+       +-----------------------+
```

---

### 1.2 Internal Mechanics: Kernel Subsystems & Syscalls

#### 1. The 7 Linux Namespaces
Namespaces are created via the `clone(2)` system call using specific bit flags, or joined dynamically using `setns(2)` and `unshare(2)`:

| Namespace | `clone` Flag | Virtualized Kernel Resource | Production Impact & Isolation Behavior |
|:---|:---|:---|:---|
| **PID** | `CLONE_NEWPID` | Process IDs | Container main process sees itself as PID 1. Signals like `SIGKILL` are dropped if unhandled by PID 1. Subprocesses map to separate PIDs on the host. |
| **NET** | `CLONE_NEWNET` | Network devices, IP routing tables, port bindings, firewall rules | Container receives its own loopback interface (`lo`) and virtual ethernet adapter (`eth0`). Container port 3000 does not collide with host port 3000. |
| **MNT** | `CLONE_NEWNS` | Filesystem mount points | Container gets an isolated root filesystem (`/`) via `pivot_root(2)`, shielding the host's `/var`, `/etc`, and `/proc`. |
| **UTS** | `CLONE_NEWUTS` | Hostname and NIS domain name | Allows the container to have a dedicated hostname (e.g., `web-srv-01`) without mutating the host's `/etc/hostname`. |
| **IPC** | `CLONE_NEWIPC` | System V IPC, POSIX message queues, shared memory segments | Prevents cross-container shared memory exploits (`shmget`, `mq_open`). |
| **USER** | `CLONE_NEWUSER`| User and Group ID mappings | Enables root inside container (`UID 0`) to map to an unprivileged user (`UID 10001`) on the host. Breaks container breakout exploits. |
| **CGROUP**| `CLONE_NEWCGROUP`| Cgroup directory root view | Virtualizes `/proc/cgroups` and `/sys/fs/cgroup` so the container cannot discover the host's cgroup hierarchy. |

#### 2. Cgroups v1 vs. Cgroups v2
Linux Control Groups manage resource allocation. 

```
Cgroups v1 (Per-controller Orthogonal Trees)
/sys/fs/cgroup/
├── cpu/           --> Process X belongs to /cpu/group_a
├── memory/        --> Process X belongs to /memory/group_b (Inconsistent hierarchy!)
└── blkio/

Cgroups v2 (Unified Single Tree Hierarchy)
/sys/fs/cgroup/
├── system.slice/
└── docker-container_abc.scope/
    ├── cgroup.controllers: "cpu memory io pids"
    ├── cpu.max: "200000 100000" (2 CPUs: 200ms quota per 100ms period)
    ├── memory.max: "1073741824" (1 GiB hard limit)
    ├── memory.current: "452984832"
    └── pids.max: "256" (Prevents fork bombs)
```

##### Cgroups v1 CPU Throttling:
Controlled by two files:
- `cpu.cfs_period_us`: The enforcement window in microseconds (default: $100,000 \mu s = 100\text{ms}$).
- `cpu.cfs_quota_us`: The maximum runtime the container's processes can cumulatively consume across all CPU cores within that period.
For a 2.5 CPU limit: `cpu.cfs_quota_us` = 250,000, `cpu.cfs_period_us` = 100,000.
If a multi-threaded process runs on 4 cores and burns 250ms of CPU time in the first 30ms of the period, the Linux CFS (Completely Fair Scheduler) **completely suspends execution** for the remaining 70ms of the window. This causes massive tail latency spikes ($p99$).

##### Cgroups v2 Improvements:
- Unified hierarchy: CPU, Memory, and I/O controllers govern the exact same cgroup slice.
- `cpu.max`: Syntax is `quota period` (e.g., `200000 100000`).
- `memory.max`: Hard limit. When `memory.current` exceeds `memory.max`, the Linux kernel triggers the **OOM (Out Of Memory) Killer**.
- `memory.high`: Throttling threshold. Slows down allocations and invokes page reclaim before triggering OOM killing.
- True writeback caching accounting: In v1, buffered disk I/O in page cache was charged to memory, bypassing `blkio` throttling. In v2, unified accounting tracks disk writes directly to the owning cgroup.

##### The OOM Killer Scoring:
The kernel calculates `oom_score` based on `oom_score_adj` (set via `/proc/<pid>/oom_score_adj`, from -1000 to +1000) and physical RAM consumed. When memory is exhausted:
1. Kernel invokes `mem_cgroup_out_of_memory()`.
2. Inspects processes within the container cgroup.
3. Sends `SIGKILL` (signal 9) to the process with the highest score. The process cannot catch, block, or handle `SIGKILL`.

#### 3. OverlayFS: Union Filesystem Mechanics
OverlayFS constructs an illusion of a single filesystem by combining multiple directories:
- **`lowerdir`**: Read-only layers (base image, dependencies). Multiple layers are separated by colons (`lowerdir=/layer3:/layer2:/layer1`).
- **`upperdir`**: Read-write layer. Any file created, modified, or written by the running container lives here.
- **`workdir`**: Internal kernel staging directory used to prepare atomic file operations before writing to `upperdir`.
- **`merged`**: The mount target presented to the container process as root `/`.

```
           +-------------------------------------------------------+
Container  |            Merged Directory (`/`)                     |
Access     |   reads: reads upperdir if exists, else lowerdir      |
           |   writes: writes directly to upperdir                 |
           +-------------------------------------------------------+
                                      |
                 +--------------------+--------------------+
                 |                                         |
                 v                                         v
   +---------------------------+             +---------------------------+
   |  Upperdir (Read-Write)    |             |  Lowerdir (Read-Only)     |
   |  - Modded files (`app.rb`)|             |  Layer 2: Ruby 3.3.0 gems |
   |  - New logs (`puma.log`)  |             |  Layer 1: Debian Bookworm |
   |  - Whiteout device (char  |             +---------------------------+
   |    0,0 for deleted files) |
   +---------------------------+
```

##### Copy-on-Write (CoW) Workflow:
1. **File Read**: If `/app/server.js` is unmodified, the kernel reads directly from `lowerdir`. Zero copy overhead.
2. **File Modification**: The instant an application opens `/app/server.js` with write flags (`O_WRONLY` or `O_RDWR`), OverlayFS triggers a **Copy-Up** operation. The entire file is copied byte-for-byte from `lowerdir` to `upperdir`. Subsequent writes modify the copy in `upperdir`.
3. **File Deletion**: When an application deletes `/etc/nginx/nginx.conf` present in `lowerdir`, OverlayFS cannot delete a file in a read-only layer. Instead, it creates a **Whiteout file** (character device with major:minor number 0:0) in `upperdir`. The kernel masks the file from directory listings in `merged`.

#### 4. Seccomp (Secure Computing Mode) & Linux Capabilities
- **Linux Capabilities (`cap_set_proc(2)`)**: Traditional Unix security is all-or-nothing (`root` vs `non-root`). Linux decomposes superuser privileges into distinct bits:
  - `CAP_CHOWN`: Change file ownership.
  - `CAP_NET_BIND_SERVICE`: Bind sockets to privileged ports (< 1024).
  - `CAP_SYS_ADMIN`: Overly broad "god-mode" capability (mount filesystems, load kernel modules, trace arbitrary processes).
  - Production containers drop `CAP_SYS_ADMIN`, `CAP_NET_ADMIN`, and `CAP_SYS_RAWIO` by default.
- **Seccomp Filters**: Berkley Packet Filter (BPF) programs evaluated upon every single system call.
  - Default Docker Seccomp profile blocks >40 dangerous syscalls out of ~350, including `reboot`, `ptrace`, `kexec_load`, `sys_chroot`, and `add_key`.
  - Blocks attackers from exploiting kernel zero-days even if they achieve `UID 0` within the container.

---

### 1.3 Production Shell Commands & C Program Verification

#### 1. Verifying Cgroups v2 and OOM Triggers
```bash
# Check if host or container is running Cgroups v2
mount -t cgroup2

# Inspect container cgroup directory
CONTAINER_ID=$(docker run -d --memory="256m" --cpus="1.5" --name test-oom alpine sleep 3600)
CGROUP_PATH="/sys/fs/cgroup/system.slice/docker-${CONTAINER_ID}.scope"

# On modern systemd-based hosts:
cat ${CGROUP_PATH}/memory.max
# Output: 268435456 (256 * 1024 * 1024)

cat ${CGROUP_PATH}/cpu.max
# Output: 150000 100000 (150ms per 100ms period = 1.5 CPUs)

# Check OOM kill count on the cgroup
grep oom ${CGROUP_PATH}/memory.events
# Output:
# oom 1
# oom_kill 1
```

#### 2. Manual OverlayFS Mount Reproduction
```bash
# Step-by-step reproduction of Docker storage layer mechanics in pure bash
mkdir -p /tmp/overlay-demo/{lower,upper,work,merged}

# Create base file in lower layer
echo "Database configuration: host=localhost" > /tmp/overlay-demo/lower/database.yml

# Mount OverlayFS
sudo mount -t overlay overlay \
  -o lowerdir=/tmp/overlay-demo/lower,upperdir=/tmp/overlay-demo/upper,workdir=/tmp/overlay-demo/work \
  /tmp/overlay-demo/merged

# Inspect merged view
cat /tmp/overlay-demo/merged/database.yml
# Output: Database configuration: host=localhost

# Modify the file from the merged mount (Triggers Copy-Up)
echo "Database configuration: host=production-pg-cluster.internal" > /tmp/overlay-demo/merged/database.yml

# Inspect lower vs upper
cat /tmp/overlay-demo/lower/database.yml
# Output: Database configuration: host=localhost (UNMODIFIED!)

cat /tmp/overlay-demo/upper/database.yml
# Output: Database configuration: host=production-pg-cluster.internal

# Cleanup
sudo umount /tmp/overlay-demo/merged
rm -rf /tmp/overlay-demo
```

---

### 1.4 Production Outages & Debugging

#### Outage Case Study: The High-Latency CFS Quota Throttling Loop
- **Context**: A Node.js microservice handling 4,000 req/sec was provisioned with `--cpus="1.0"`. Node.js is single-threaded for its event loop, so the team reasoned that 1 CPU core was sufficient.
- **Symptom**: Average response times were 12ms, but $p99$ response times surged to 105ms. CPU utilization reported by Prometheus was only 48%.
- **Root Cause**: Node.js internally uses `libuv` with 4 worker threads for DNS lookups, compression, and crypto operations (`crypto.pbkdf2`). During traffic bursts, the event loop and the worker threads ran simultaneously across 4 physical cores. Within the first 25ms of the CFS 100ms window, the 4 threads accumulated $4 \times 25\text{ms} = 100\text{ms}$ of CPU time, depleting the quota. The CFS scheduler paused the container's processes for the remaining 75ms. Every incoming request arriving during that freeze suffered a 75ms latency penalty.
- **Resolution**:
  1. Inspect throttling metrics:
     ```bash
     cat /sys/fs/cgroup/cpu/cpu.stat
     # nr_periods 12040
     # nr_throttled 6850  <-- Over 50% of windows were throttled!
     # throttled_time 485938592934
     ```
  2. In Cgroups v2:
     ```bash
     cat /sys/fs/cgroup/.../cpu.stat
     # usage_usec 48392019
     # throttled_usec 24891024
     # nr_throttled 6850
     ```
  3. Action: Set CPU limits to integer multiples matching internal thread concurrency (or remove hard CPU limits and rely strictly on CPU shares/weights while setting memory hard limits).

---

### 1.5 Trade-offs & Decision Matrix

| Mechanism | Setting / Primitive | Pros | Cons / Latent Hazards | Production Guidance |
|:---|:---|:---|:---|:---|
| **CPU Enforcement** | Hard Limits (`--cpus=2`) | Enforces deterministic noisy-neighbor boundaries. Prevents host starvation. | CFS quota period slicing triggers aggressive thread suspension and $p99$ spikes. | Prefer generous quotas or CPU shares (`--cpu-shares=1024`) for latency-critical services. |
| **Memory Enforcement** | Hard Limit (`--memory=4g`) | Predictable capacity planning. Protects host kernel from unmitigated paging. | Triggering `memory.max` causes immediate SIGKILL (exit 137). Zero recovery hooks. | Always monitor `memory.current` vs `memory.high` and alert at 80% threshold. |
| **User Mapping** | User Namespaces (`userns-remap`) | Complete immunity to container breakout root privilege escalation. | Incompatible with some volume bind mounts and storage drivers without UID shifting. | Mandatory for multi-tenant container platforms; recommended for edge clusters. |
| **Storage Driver** | OverlayFS (`overlay2`) | Blazing fast container spin-up, zero-copy read sharing across containers. | High disk write churn causes metadata fragmentation and copy-up latency for large files. | Mount dedicated named volumes or bind mounts for write-heavy workloads (Postgres, Redis). |

---

### 1.6 Senior Interview Q&A

#### Q: Explain what happens at the kernel and container runtime level when an application hits its memory limit. What is exit code 137?
**Answer**:
When a process inside a container allocates heap memory (via `brk` or `mmap`), the Linux virtual memory manager allocates pages. In Cgroups v2, this increments `memory.current` in the container's cgroup directory. 
1. As usage crosses `memory.high`, the kernel proactively invokes asynchronous page reclamation (flushing clean file-backed page caches to disk).
2. If memory continues to grow and hits `memory.max`, synchronous direct reclaim is executed in the allocating thread's context.
3. If reclaim fails to free memory, `mem_cgroup_out_of_memory()` is triggered.
4. The kernel's OOM killer calculates badness scores:
   $$\text{points} = \text{RAM consumed} + \text{oom\_score\_adj}$$
5. The container's primary offending process is sent an uncatchable `SIGKILL` (signal number 9).
6. When a Unix process terminates due to a signal, its exit code returned to the parent (`waitpid`) is:
   $$\text{Exit Code} = 128 + \text{Signal Number} = 128 + 9 = 137$$
7. Docker / containerd captures this exit code, marks the container state as `Dead` / `Exited (137)`, and if configured with `restart: on-failure`, attempts a container recreation.

---

# 2. Container Runtime Architecture & Lifecycle States

### 2.1 Definition & Core Concept
Modern containerization follows a decoupled, layered micro-architecture standardized by the **Open Container Initiative (OCI)**:
- **Docker Daemon (`dockerd`)**: High-level developer UX daemon managing networks, volumes, image builds, and exposing a REST API (`/var/run/docker.sock`).
- **`containerd`**: Cloud-native runtime daemon (governed by CNCF). Handles image distribution, storage management, snapshotting, and container lifecycle supervisor coordination.
- **`containerd-shim`**: A tiny daemon process running per container. It retains open standard I/O streams (stdin, stdout, stderr) and container exit codes, allowing `containerd` and `dockerd` to restart or upgrade without killing the running containers.
- **`runc`**: The reference implementation of the OCI runtime-spec (`opencontainers/runtime-spec`). A short-lived CLI binary that accepts an unpacked root filesystem directory and a `config.json` specification, configures the namespaces, cgroups, seccomp filters, mounts the rootfs via `pivot_root`, and calls `execve(2)` to start the application. Once the process runs, `runc` exits.

```
+-----------------------------------------------------------------------------+
|                                docker CLI                                   |
+-----------------------------------------------------------------------------+
                                       |
                                       | (REST API via /var/run/docker.sock)
                                       v
+-----------------------------------------------------------------------------+
|                                dockerd                                      |
+-----------------------------------------------------------------------------+
                                       |
                                       | (gRPC via /run/containerd/containerd.sock)
                                       v
+-----------------------------------------------------------------------------+
|                               containerd                                    |
+-----------------------------------------------------------------------------+
          |                                                 |
          v                                                 v
+--------------------+                            +--------------------+
|  containerd-shim   |                            |  containerd-shim   |
+--------------------+                            +--------------------+
          |                                                 |
          | (forks & executes)                              | (forks & executes)
          v                                                 v
+--------------------+                            +--------------------+
|       runc         | (creates namespaces,       |       runc         |
| (Exits immediately)|  cgroups, executes app)    | (Exits immediately)|
+--------------------+                            +--------------------+
          |                                                 |
          v                                                 v
+--------------------+                            +--------------------+
| Container Proc (1) |                            | Container Proc (2) |
|    (Puma Web)      |                            |   (Sidekiq Worker) |
+--------------------+                            +--------------------+
```

---

### 2.2 Internal Mechanics: Docker Daemon to containerd to runc

#### 1. Step-by-Step Lifecycle of `docker run -d -p 80:80 nginx`
1. **API Handling**: `docker` CLI constructs a POST request to `/containers/create` on `dockerd`.
2. **Image Pull & Unpack**: If not cached locally, `dockerd` instructs `containerd` via gRPC to pull the image layers from registry.
3. **Snapshot Mount**: `containerd` uses its snapshotter (`overlayfs`) to prepare `lowerdir`, `upperdir`, and mounts the combined layer into a root directory.
4. **OCI Bundle Generation**: `containerd` generates an OCI-compliant directory bundle containing:
   - `rootfs/`: The extracted filesystem.
   - `config.json`: Specification containing UID/GID, environment variables, mounts, capabilities, seccomp profile, and namespace configuration.
5. **Shim Creation**: `containerd` forks a new process: `containerd-shim-runc-v2`.
6. **Execution via `runc`**: The shim invokes `runc create <container-id>`:
   - `runc` creates namespaces using `clone(CLONE_NEWPID | CLONE_NEWNET | CLONE_NEWNS ...)`.
   - Sets cgroup limits in `/sys/fs/cgroup/...`.
   - Applies Seccomp BPF filters.
   - Changes root directory using `pivot_root(2)` into `rootfs/`.
   - Drops Linux capabilities.
7. **Shim Handover & Container Start**: The shim calls `runc start <container-id>`. `runc` invokes `execve("/usr/sbin/nginx", ...)` replacing itself with NGINX as PID 1 inside the PID namespace. `runc` terminates. The shim monitors NGINX's exit status.

#### 2. Container Lifecycle States
An OCI container traverses deterministic state transitions:

```
                  +--------------+
                  |   Created    |
                  +--------------+
                         |
                         | (start / runc start)
                         v
     (pause)      +--------------+      (SIGSTOP)
  +-------------> |   Running    | <-----------------+
  |               +--------------+                   |
  | (unpause)            |                           |
  +----------------------+                           |
                         | (SIGTERM / timeout / SIGKILL)
                         v
                  +--------------+
                  |   Stopped    |
                  +--------------+
                         |
                         | (docker rm)
                         v
                  +--------------+
                  |   Deleted    |
                  +--------------+
```

---

### 2.3 Production CLI & Inspection Workflows

#### 1. Direct OCI Runtime Inspection
```bash
# Locate the containerd-shim and PID 1 of a running container
docker run -d --name web-target nginx:alpine
CONTAINER_PID=$(docker inspect --format '{{.State.Pid}}' web-target)
echo "Container Host PID: ${CONTAINER_PID}"

# Inspect the processes tree from host terminal
pstree -p -s ${CONTAINER_PID}
# Output: systemd(1)---containerd(1201)---containerd-shim(18920)---nginx(18950)---nginx(18990)

# Inspect namespaces attached to this process
ls -l /proc/${CONTAINER_PID}/ns/
# lrwxrwxrwx 1 root root 0 net -> 'net:[4026532582]'
# lrwxrwxrwx 1 root root 0 mnt -> 'mnt:[4026532580]'
# lrwxrwxrwx 1 root root 0 pid -> 'pid:[4026532583]'

# Execute an interactive debugging shell directly inside the container's namespaces
# WITHOUT using 'docker exec' (Bypasses Docker Daemon completely!)
sudo nsenter --target ${CONTAINER_PID} --mount --uts --ipc --net --pid /bin/sh
```

#### 2. Production Logging Drivers & Log Rotation
By default, Docker uses the `json-file` logging driver with **unlimited file growth**, which leads to disk exhaustion outages:
```json
// /etc/docker/daemon.json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "50m",
    "max-file": "5"
  },
  "default-ulimits": {
    "nofile": {
      "Name": "nofile",
      "Hard": 65535,
      "Soft": 65535
    }
  },
  "live-restore": true
}
```
> [!IMPORTANT]
> Setting `"live-restore": true` allows containers to remain running even if `dockerd` crashes or is upgraded during OS patching.

---

### 2.4 Production Outages & Debugging

#### Outage Case Study: The Docker Daemon Zombie Deadlock
- **Context**: A cluster node running 45 containers stopped responding to health checks. New deployments failed with `docker: Error response from daemon: context deadline exceeded`.
- **Diagnosis**: 
  1. `systemctl status docker` showed daemon was active, but `docker ps` hung indefinitely.
  2. Inspecting the process list (`ps aux | grep containerd-shim`) revealed thousands of zombie processes (`[sh] <defunct>`).
  3. A misconfigured cron script inside one container was executing `docker exec <container> /bin/sh -c "backup.sh"` every 5 seconds. The script ran a background process and exited without reaping child PIDs.
  4. Container PID 1 did not have a signal handler or zombie reaper installed (was running raw binary instead of an init system like `tini` or `dumb-init`).
  5. The host process table reached `/proc/sys/kernel/pid_max`, preventing `dockerd` from creating pipes or threads.
- **Resolution**:
  1. Force-killed the defunct shims using `kill -9`.
  2. Updated all container entrypoints to prepend `tini`:
     ```dockerfile
     RUN apk add --no-cache tini
     ENTRYPOINT ["/sbin/tini", "--"]
     CMD ["ruby", "bin/rails", "server"]
     ```
  3. Enforced PID limits per container: `--pids-limit 500`.

---

### 2.5 Trade-offs & Decision Matrix

| Container Runtime | Scope & Target | Pros | Cons / Overhead | Production Recommendation |
|:---|:---|:---|:---|:---|
| **Docker Engine (`dockerd`)** | Developer laptops, local Compose stacks | Unified UX, integrated build tools (`BuildKit`), easy CLI. | Monolithic overhead; daemon crashes can affect control plane operations if `live-restore` is off. | Best for local development and CI build nodes. |
| **containerd** | Production Kubernetes / ECS | Lean, resource-efficient, direct integration via CRI (Container Runtime Interface). | No built-in image build tooling; requires external CLI tools (`nerdctl`, `crictl`). | Production standard for K8s worker nodes and bare metal. |
| **runc** | Low-level OCI reference | Pure Linux primitives, minimal footprint, standardized. | Does not handle networking, image pulling, or long-running stream retention. | Low-level substrate; never used directly in production without containerd. |
| **gVisor (`runsc`)** | Multi-tenant untrusted code execution | Intercepts all syscalls in userspace Sentry kernel; near-complete host isolation. | Significant performance penalty for syscall-intensive apps (database I/O, heavy network). | Use for multi-tenant SaaS running customer-submitted code. |

---

### 2.6 Senior Interview Q&A

#### Q: Why is `containerd-shim` necessary? Why doesn't `containerd` monitor the container process directly?
**Answer**:
`containerd-shim` serves three critical architectural purposes:
1. **Daemonless Containers (Surviving Daemon Restarts)**: If `containerd` maintained the open file descriptors (pipes) for container `stdin`, `stdout`, and `stderr`, restarting `containerd` (for an upgrade or due to a crash) would break the pipes, sending `SIGPIPE` to every container on the host and terminating the entire fleet. The shim acts as an independent sub-parent process that outlives daemon restarts.
2. **Reaping Zombies as Sub-reaper**: The shim calls `prctl(PR_SET_CHILD_SUBREAPER, 1)`. If the container's internal processes fork child processes that get orphaned, the shim adopts and reaps them if the container's PID 1 fails to do so.
3. **Exit Code Retention**: When the container process terminates, the shim captures the exit status and saves it for `containerd` to query asynchronously via `wait`, even if `containerd` was offline at the precise millisecond the container exited.

---

# 3. Image Engineering, Caching & Container Security

### 3.1 Definition & Core Concept
An OCI container image is a cryptographically verifiable, content-addressable archive consisting of:
1. A **Manifest JSON** listing config digests and layer tarballs.
2. An **Image Configuration JSON** storing environment variables, exposed ports, entrypoint instructions, and architecture.
3. A collection of **Layer Tarballs (`.tar.gz`)**, each representing a delta of the filesystem generated by a Dockerfile build step.

Optimization revolves around three fundamentals:
- **Layer Caching**: Arranging commands so that slowly changing dependencies precede frequently changing application source code.
- **Multi-Stage Builds**: Compiling binaries and downloading build tools in heavy "builder" stages, then extracting *only* compiled artifacts into ultra-lean "runtime" stages.
- **Attack Surface Minimization**: Running as a non-root user, removing package managers, compilers, and shells from the final runtime image.

---

### 3.2 Internal Mechanics: OCI Image Spec, Merkle Trees & Layer Hashes

```
+------------------------------------------------------------------------------------+
|                               OCI Image Manifest                                   |
|  schemaVersion: 2                                                                  |
|  config: sha256:7b10fae... (Entrypoint, Env, User, Architecture)                   |
|  layers:                                                                           |
|    - sha256:a1b2c3d... (Size: 28MB)  --> Base OS (Debian Slim / Alpine)            |
|    - sha256:f4e5d6c... (Size: 140MB) --> Language Runtime + System Shared Libs     |
|    - sha256:8a9b0c1... (Size: 85MB)  --> Vendor Dependencies (gems / node_modules) |
|    - sha256:1d2e3f4... (Size: 4MB)   --> Compiled App Source Code                  |
+------------------------------------------------------------------------------------+
```

#### 1. Layer Hash Generation and Cache Invalidation
When BuildKit evaluates a Dockerfile:
- For `RUN <command>`: Docker evaluates the exact string command. If the previous layer hash matches and the command string is identical, it uses the cached layer. **Docker does not check if external repositories modified packages** (e.g., `apt-get update` will NOT re-run unless its layer cache is invalidated).
- For `COPY <src> <dest>`: Docker calculates a **SHA-256 checksum of each file's content** in `<src>`. If any file's hash changes, that step's cache is invalidated.
- **Cascading Invalidation**: Once any layer cache is invalidated, **every subsequent instruction in the Dockerfile is executed from scratch**, completely bypassing the cache.

#### 2. `CMD` vs. `ENTRYPOINT` (Shell Form vs. Exec Form)
This is a critical production nuance:

| Form | Syntax | Kernel Execution Equivalent | Signal Forwarding (`SIGTERM`) |
|:---|:---|:---|:---|
| **Exec Form** | `ENTRYPOINT ["executable", "param1"]` | `execve("/bin/executable", ["param1"], ...)` | **YES**. Application runs as PID 1 directly. Receives `SIGTERM` on shutdown. |
| **Shell Form**| `ENTRYPOINT executable param1` | `execve("/bin/sh", ["-c", "executable param1"], ...)` | **NO**. `/bin/sh` runs as PID 1. Most shells do **not** forward `SIGTERM` to child processes! |

##### The Shell Form Trap:
If you write `CMD npm start`, Docker executes `/bin/sh -c "npm start"`. 
- `/bin/sh` is PID 1.
- `node` is PID 2.
- When Docker stops the container (`docker stop`), it sends `SIGTERM` to PID 1.
- `/bin/sh` ignores the signal. Node.js never gets notified.
- Docker waits 10 seconds (default timeout), gives up, and sends `SIGKILL` to PID 1.
- The Node process is abruptly terminated: active database connections drop, inflight HTTP requests are aborted, and data corruption can occur.

---

### 3.3 Production Dockerfile Optimization Patterns

#### 1. Optimal Layer Caching Sequence
```dockerfile
# ANTI-PATTERN: Invalidates cache on every single code change!
FROM node:20-slim
WORKDIR /app
COPY . .
RUN npm ci
CMD ["node", "dist/server.js"]

# PRODUCTION PATTERN: Isolates dependencies into immutable cached layer
FROM node:20-slim
WORKDIR /app
# 1. Copy ONLY dependency manifests first
COPY package.json package-lock.json ./
# 2. Install dependencies (layer stays cached unless packages change!)
RUN npm ci --only=production
# 3. Copy source code (changes frequently, but fast to build)
COPY . .
CMD ["node", "dist/server.js"]
```

#### 2. Base Image Deep Dive: Alpine vs. Slim vs. Distroless

| Image Variant | Typical Size | C Standard Library | Security Vulnerabilities (CVEs) | Production Caveats |
|:---|:---|:---|:---|:---|
| **Debian Slim** (`node:20-slim`) | ~120 MB | `glibc` | Low | High compatibility. Pre-compiled wheels/gems run flawlessly. Recommended default for enterprise. |
| **Alpine Linux** (`node:20-alpine`) | ~40 MB | `musl libc` | Minimal | **Musl memory allocator performs poorly with multi-threaded allocators**. C-extensions (Numpy, Nokogiri, GRPC) must be compiled from C source, slowing builds. DNS resolver does not support TCP fallback or glibc search path conventions. |
| **Google Distroless** (`gcr.io/distroless/nodejs20-debian12`) | ~50 MB | `glibc` | Near Zero | Contains NO package manager (`apt`), NO shell (`/bin/sh`). Cannot run `docker exec -it /bin/sh`. Exceptional production security. |

---

### 3.4 Production Outages & Debugging

#### Outage Case Study: Alpine `musl` DNS Lookup Timeout
- **Context**: A high-volume Ruby API containerized on `ruby:3.3-alpine` experienced periodic 5-second connection timeouts when resolving internal AWS RDS endpoints (`rds.postgres.internal`).
- **Investigation**:
  1. Captured network packets inside the container using `tcpdump`.
  2. Discovered that glibc queries DNS `A` (IPv4) and `AAAA` (IPv6) records sequentially or over a single socket, while Alpine's `musl libc` sends `A` and `AAAA` queries **concurrently over separate UDP sockets with identical source ports**.
  3. The intermediate AWS NAT Gateway / Linux bridge iptables `conntrack` dropped the second packet due to conntrack race conditions, waiting for the 5-second DNS UDP retransmission timeout.
- **Resolution**:
  Switched the base image from `ruby:3.3-alpine` to `ruby:3.3-slim` (Debian glibc), instantly resolving DNS conntrack drops and reducing $p99$ response times by 35%.

---

### 3.5 Trade-offs & Decision Matrix

| Strategy | Implementation | Pros | Cons | Decision Rule |
|:---|:---|:---|:---|:---|
| **Distroless** | `FROM gcr.io/distroless/cc-debian12` | Zero shell, zero attack vectors, passes enterprise security audits. | Difficult to troubleshoot live containers; requires ephemeral debug containers. | Deploy to internet-facing public web APIs. |
| **Non-Root User** | `USER 10001:10001` | Prevents filesystem modification of system directories; blocks privilege escalation. | Cannot bind ports < 1024 without `CAP_NET_BIND_SERVICE`; file permission issues with mounted host volumes. | Mandatory for all production containers (listen on ports > 1024, e.g., 3000, 8080). |
| **Read-Only Root Filesystem** | `docker run --read-only` | Eliminates malware persistence; attackers cannot drop binaries in `/tmp` or `/var`. | Requires explicit `tmpfs` mounts for directories like `/tmp`, `/app/tmp`, and `/run`. | Mandatory for PCI-DSS and financial transaction workloads. |

---

### 3.6 Senior Interview Q&A

#### Q: How does BuildKit differ from the legacy Docker build engine, and how do you leverage secret mounts in multi-stage builds?
**Answer**:
Docker BuildKit introduces an execution DAG (Directed Acyclic Graph) engine that resolves build stages concurrently:
1. **Parallel Stage Execution**: Independent build stages (e.g., building frontend React assets and building backend Go binaries) run simultaneously on multiple CPU cores.
2. **Dead Code Elimination**: Any build stage whose artifacts are not referenced by the final target stage is skipped entirely.
3. **Secure Secret Injection (`--mount=type=secret`)**:
   In legacy Docker, passing API keys or private SSH keys via `ARG` or `ENV` baked the secrets permanently into the intermediate layer tarballs (discoverable via `docker history`).
   BuildKit provides ephemeral in-memory secret mounting:
   ```dockerfile
   RUN --mount=type=secret,id=npmrc,target=/root/.npmrc \
       npm ci
   ```
   The secret file is mounted in-memory only during the execution of that specific `RUN` command. It is **never recorded in the layer metadata or committed to the image snapshot**.

---

# 4. Container Networking & Storage Architecture

### 4.1 Definition & Core Concept
Container networking isolates process network stacks while providing controlled ingress, egress, and inter-container communication:
- **Bridge Network (Default)**: A virtual software switch (`docker0`) created on the Linux host. Containers get a virtual network adapter connected to the bridge.
- **Host Network**: Strips away network namespace isolation (`--network host`). The container shares the host's exact network stack, IP address, and port space.
- **Overlay Network**: Multi-host VXLAN encapsulation enabling containers on different physical nodes to communicate over a flat virtual subnet.

Container storage bridges the transient nature of OverlayFS with persistent state:
- **Bind Mounts**: Direct mapping of a host filesystem path into the container.
- **Named Volumes**: Docker-managed persistent storage residing in `/var/lib/docker/volumes/`.
- **tmpfs Mounts**: Ephemeral in-memory storage backed by host RAM (`tmpfs`). Data is wiped upon container stop.

```
Host Network Namespace (Physical eth0: 192.168.1.50)
+---------------------------------------------------------------------------------+
|                                 docker0 (Bridge)                                |
|                              IP: 172.17.0.1/16                                  |
+---------------------------------------------------------------------------------+
          |                                                 |
          | (vethA <---> vethB)                             | (vethC <---> vethD)
          v                                                 v
+-----------------------+                         +-----------------------+
|  Container 1 Net NS   |                         |  Container 2 Net NS   |
|  eth0: 172.17.0.2     |                         |  eth0: 172.17.0.3     |
|  (Puma Web App)       |                         |  (PostgreSQL DB)      |
+-----------------------+                         +-----------------------+
```

---

### 4.2 Internal Mechanics: Veth Pairs, Iptables NAT & Storage Drivers

#### 1. Bridge Network Mechanics
When a container connects to a Docker bridge network:
1. **Virtual Ethernet Pair (`veth`)**: The kernel creates a linked pair of virtual ethernet interfaces (like a virtual patch cord).
   - One end (`vethXXXXX`) stays in the host network namespace attached to the bridge switch (`docker0` or custom `br-XXXXX`).
   - The peer end is moved inside the container's private network namespace and renamed to `eth0`.
2. **IP Allocation & Embedded DNS**: Docker assigns a private IP (e.g., `172.18.0.4`) via internal IPAM (IP Address Management). In user-defined bridge networks, Docker runs an embedded DNS server at `127.0.0.11` that automatically resolves container names (e.g., `postgres` -> `172.18.0.3`).
3. **Port Publishing via Iptables (DNAT)**:
   When running `docker run -p 8080:80 nginx`:
   The Docker daemon inserts a **DNAT (Destination Network Address Translation)** rule into the host's `PREROUTING` and `DOCKER` iptables chains:
   ```bash
   # Inbound packet destined for host port 8080:
   # MATCH: -d 0.0.0.0/0 -p tcp --dport 8080
   # ACTION: DNAT to container private IP 172.17.0.2:80
   iptables -t nat -A DOCKER -p tcp --dport 8080 -j DNAT --to-destination 172.17.0.2:80
   ```
4. **Outbound Internet Access (SNAT / MASQUERADE)**:
   When the container makes an outbound HTTPS call to an external API:
   The kernel evaluates the `POSTROUTING` nat chain:
   ```bash
   # Outbound packet leaving host eth0:
   # MATCH: -s 172.17.0.0/16 ! -o docker0
   # ACTION: MASQUERADE (Rewrite source IP from 172.17.0.2 to host physical IP)
   iptables -t nat -A POSTROUTING -s 172.17.0.0/16 ! -o docker0 -j MASQUERADE
   ```

#### 2. Storage Subsystems Comparison

```
+-------------------------------------------------------------------------------+
|                            Linux Host Filesystem                              |
|                                                                               |
|  [ /var/lib/docker/volumes/pgdata/_data ] ----------> Named Volume (Managed)  |
|                                                            |                  |
|  [ /home/deploy/rails-app/config/master.key ] -------> Bind Mount (Direct)    |
|                                                            |                  |
|  [ Host RAM / Swap ] --------------------------------> tmpfs Mount (RAM)      |
+-------------------------------------------------------------------------------+
                                                             |
                                                             v
                                             +-------------------------------+
                                             |     Running OCI Container     |
                                             +-------------------------------+
```

---

### 4.3 Production Docker Compose Patterns & Healthchecks

A production-grade `docker-compose.yml` for local development and integration environments:

```yaml
version: '3.8'

services:
  postgres:
    image: postgres:16-alpine
    container_name: production_pg
    environment:
      POSTGRES_DB: app_production
      POSTGRES_USER: app_user
      POSTGRES_PASSWORD_FILE: /run/secrets/pg_password
    secrets:
      - pg_password
    volumes:
      - pg_data:/var/lib/postgresql/data
    networks:
      - backend_net
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app_user -d app_production"]
      interval: 5s
      timeout: 3s
      retries: 5
      start_period: 10s
    deploy:
      resources:
        limits:
          cpus: '2.0'
          memory: 2048M

  redis:
    image: redis:7-alpine
    container_name: production_redis
    command: ["redis-server", "--appendonly", "yes", "--requirepass", "secure_redis_pass"]
    volumes:
      - redis_data:/data
    networks:
      - backend_net
    healthcheck:
      test: ["CMD", "redis-cli", "-a", "secure_redis_pass", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5

  web:
    build:
      context: .
      dockerfile: Dockerfile
      target: runner
    container_name: production_app
    environment:
      DATABASE_URL: postgres://app_user:secure_pg_pass@postgres:5432/app_production
      REDIS_URL: redis://:secure_redis_pass@redis:6379/0
      PORT: 3000
    ports:
      - "3000:3000"
    networks:
      - backend_net
    # Critical Production Pattern: Prevent boot race condition!
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    restart: unless-stopped
    deploy:
      resources:
        limits:
          cpus: '1.5'
          memory: 1024M

secrets:
  pg_password:
    file: ./secrets/postgres_password.txt

volumes:
  pg_data:
    driver: local
  redis_data:
    driver: local

networks:
  backend_net:
    driver: bridge
```

---

### 4.4 Production Outages & Debugging

#### Outage Case Study: The Docker Compose Boot Crash Loop
- **Context**: On spinning up microservices with `docker compose up`, the web application crashed with `PG::ConnectionBad: could not connect to server: Connection refused`.
- **Root Cause**: The web container had `depends_on: [postgres]`. However, standard `depends_on` only waits for the postgres container to reach the `Running` state (the process has started). PostgreSQL, however, initializes storage, validates transaction logs, and runs recovery for 3 to 8 seconds before opening port 5432. The web app executed database migrations immediately, failed to connect, and terminated.
- **Resolution**: Implemented the health check pattern above with `condition: service_healthy`. Docker Compose pauses starting the web container until `pg_isready` returns exit code 0.

---

### 4.5 Trade-offs & Decision Matrix

| Mount Type | Read/Write Throughput | Persistence Scope | Host Coupling | Best Used For |
|:---|:---|:---|:---|:---|
| **Named Volume** | Native host filesystem performance | Persists across container deletions; managed by Docker engine | Decoupled from host paths (`/var/lib/docker/...`) | Database storage (PostgreSQL, MySQL, Redis), persistent logs. |
| **Bind Mount** | Native host performance | Tied directly to absolute host path | Highly coupled; dependent on host directory permissions | Local source code live-reloading during development; host certificates. |
| **tmpfs Mount** | Extreme (RAM speed, zero disk I/O) | Destroyed on container stop | None | Storing sensitive tokens, master keys, session encryption cookies. |

---

### 4.6 Senior Interview Q&A

#### Q: How does Docker implement service discovery on custom bridge networks, and why doesn't it work on the default `bridge` network?
**Answer**:
1. **User-Defined Bridge Networks**: Docker activates an **embedded DNS server** listening on `127.0.0.11` inside each container. When a container resolves `postgres`, the local resolver dispatches the UDP query to `127.0.0.11`. The Docker daemon intercepts this request, queries its internal service discovery registry for the container's IP on that network, and responds with the IP address.
2. **Default `bridge` Network**: For legacy backward compatibility (dating back to Docker 0.x), the default bridge disables the embedded DNS server. Resolution can only occur via explicit, fragile `--link` flags that hardcode static entries into the container's `/etc/hosts`. Production environments must never run workloads on the default `bridge` network.

---

# 5. Staff-Level Production Dockerfiles (Rails & Next.js)

### 5.1 Definition & Core Concept
Production-grade Dockerfiles for Ruby on Rails and Node.js/Next.js must satisfy strict enterprise criteria:
1. **Minimal Attack Surface**: Zero build tools (`gcc`, `make`, `python`) or package managers in the runtime image.
2. **Non-Root Execution**: Container processes run under a dedicated unprivileged user (`UID 10001`).
3. **Optimized Layer Caching**: Dependency installation is isolated from source code changes.
4. **Memory Allocation Tuning**: Injecting high-performance memory allocators (`jemalloc`) to eliminate heap fragmentation in Ruby and Node.
5. **Deterministic Builds**: Strict lockfile enforcement (`bundle --frozen`, `npm ci`).

---

### 5.2 Internal Mechanics: Asset Compilers, Dynamic Linkers & Memory Allocators

#### 1. Why `jemalloc` in Ruby and Node.js?
The standard GNU C Library allocator (`glibc ptmalloc`) scales poorly with multi-threaded runtimes (Puma, Node worker threads). When threads release memory, `ptmalloc` holds onto virtual memory pages due to heap fragmentation, causing the container's `memory.current` to balloon until the Linux OOM killer issues `SIGKILL`.
`jemalloc` splits allocations into strict size classes and returns unused dirty pages to the kernel via `madvise(MADV_DONTNEED)`. Preloading `jemalloc` (`LD_PRELOAD=/usr/lib/x86_64-linux-gnu/libjemalloc.so.2`) typically reduces Ruby and Node production memory footprints by 20% to 40%.

---

### 5.3 Production Multi-Stage Implementations

#### 1. Production Ruby on Rails 7.x / 8.x Dockerfile
```dockerfile
# syntax=docker/dockerfile:1
# ------------------------------------------------------------------------------
# STAGE 1: Base Runtime Environment
# ------------------------------------------------------------------------------
FROM ruby:3.3.4-slim AS base

WORKDIR /rails

# Set production environment variables
ENV RAILS_ENV="production" \
    BUNDLE_DEPLOYMENT="1" \
    BUNDLE_PATH="/usr/local/bundle" \
    BUNDLE_WITHOUT="development:test" \
    MALLOC_CONF="dirty_decay_ms:1000,narenas:2"

# Install essential runtime dependencies and jemalloc
RUN apt-get update -qq && \
    apt-get install --no-install-recommends -y \
      curl \
      libpq5 \
      libjemalloc2 \
      tzdata && \
    rm -rf /var/lib/apt/lists /var/cache/apt/archives

# Enable jemalloc globally
ENV LD_PRELOAD="/usr/lib/x86_64-linux-gnu/libjemalloc.so.2"

# ------------------------------------------------------------------------------
# STAGE 2: Build Dependencies & Asset Compilation
# ------------------------------------------------------------------------------
FROM base AS build

# Install build dependencies required for compiling native C gems (Nokogiri, pg)
RUN apt-get update -qq && \
    apt-get install --no-install-recommends -y \
      build-essential \
      git \
      libpq-dev \
      pkg-config \
      node-gyp \
      python3 && \
    rm -rf /var/lib/apt/lists /var/cache/apt/archives

# Install application gems using BuildKit cache mounts
COPY Gemfile Gemfile.lock ./
RUN --mount=type=cache,target=/usr/local/bundle/cache \
    bundle install && \
    rm -rf /usr/local/bundle/bundler/gems/*/.git

# Copy application source code
COPY . .

# Precompile Bootsnap code for faster container boot times
RUN bundle exec bootsnap precompile --gemfile app/ lib/

# Precompile Rails assets (Provide dummy SECRET_KEY_BASE for compilation)
RUN SECRET_KEY_BASE_DUMMY=1 ./bin/rails assets:precompile

# Remove temporary build artifacts
RUN rm -rf tmp/cache spec test

# ------------------------------------------------------------------------------
# STAGE 3: Final Minimalist Runtime Stage
# ------------------------------------------------------------------------------
FROM base AS runner

# Create dedicated non-root user and group
RUN groupadd --system --gid 10001 rails && \
    useradd rails --uid 10001 --gid 10001 --create-home --shell /bin/bash

# Copy installed gems and application code from build stage
COPY --from=build --chown=rails:rails /usr/local/bundle /usr/local/bundle
COPY --from=build --chown=rails:rails /rails /rails

# Switch to non-root user
USER 10001:10001

# Expose HTTP port
EXPOSE 3000

# Configure healthcheck
HEALTHCHECK --interval=10s --timeout=3s --retries=3 --start-period=15s \
  CMD curl -f http://localhost:3000/up || exit 1

# Launch application server using exec form
ENTRYPOINT ["/rails/bin/docker-entrypoint"]
CMD ["./bin/thrust", "./bin/rails", "server"]
```

#### 2. Production Next.js (Standalone Output) Dockerfile
```dockerfile
# syntax=docker/dockerfile:1
# ------------------------------------------------------------------------------
# STAGE 1: Dependency Installation
# ------------------------------------------------------------------------------
FROM node:20-bookworm-slim AS deps
WORKDIR /app

# Install build dependencies if needed
RUN apt-get update -qq && \
    apt-get install --no-install-recommends -y libc6 && \
    rm -rf /var/lib/apt/lists /var/cache/apt/archives

# Copy package manifests
COPY package.json package-lock.json ./

# Install clean dependencies using BuildKit cache mount
RUN --mount=type=cache,target=/root/.npm \
    npm ci --include=dev

# ------------------------------------------------------------------------------
# STAGE 2: Source Compilation & Asset Bundling
# ------------------------------------------------------------------------------
FROM node:20-bookworm-slim AS builder
WORKDIR /app

COPY --from=deps /app/node_modules ./node_modules
COPY . .

# Disable Next.js telemetry during build
ENV NEXT_TELEMETRY_DISABLED=1
ENV NODE_ENV=production

# Compile Next.js with standalone output mode configured in next.config.js
RUN npm run build

# ------------------------------------------------------------------------------
# STAGE 3: Lean Production Runner
# ------------------------------------------------------------------------------
FROM node:20-bookworm-slim AS runner
WORKDIR /app

ENV NODE_ENV=production
ENV NEXT_TELEMETRY_DISABLED=1
ENV PORT=3000
ENV HOSTNAME="0.0.0.0"

# Create non-root system user
RUN groupadd --system --gid 10001 nodejs && \
    useradd --system --uid 10001 --gid 10001 nextjs

# Copy static assets and standalone server package
# Note: output: 'standalone' bundles only the required node_modules automatically!
COPY --from=builder /app/public ./public
COPY --from=builder --chown=nextjs:nodejs /app/.next/standalone ./
COPY --from=builder --chown=nextjs:nodejs /app/.next/static ./.next/static

# Switch to unprivileged user
USER 10001:10001

EXPOSE 3000

HEALTHCHECK --interval=10s --timeout=3s --retries=3 --start-period=10s \
  CMD node -e "require('http').get('http://localhost:3000/api/health', (r) => {if (r.statusCode !== 200) process.exit(1);})"

# Exec form launching the minimal standalone Node.js server (~80MB final image!)
CMD ["node", "server.js"]
```

---

### 5.4 Production Outages & Debugging

#### Outage Case Study: Next.js Missing Static CSS/JS 404s in Production
- **Context**: After deploying a standalone Next.js container, pages loaded with raw, unstyled HTML. Chrome console threw 404 errors for `/_next/static/chunks/main-app.js`.
- **Root Cause**: In Next.js, setting `output: 'standalone'` copies the minimal server code and imported runtime dependencies into `.next/standalone`. However, **it explicitly omits `.next/static` and `public/` by design** to allow offloading them to a CDN. If your container serves requests directly without an external S3/CloudFront CDN, omitting `COPY --from=builder /app/.next/static ./.next/static` causes the standalone server to return 404s for all assets.
- **Resolution**: Ensured the Dockerfile explicitly copies `.next/static` into the standalone root directory as demonstrated in Stage 3 above.

---

### 5.5 Trade-offs & Decision Matrix

| Optimization Technique | Performance Benefit | Security Benefit | Build Complexity | Production Verdict |
|:---|:---|:---|:---|:---|
| **Multi-Stage Builds** | Drastic image reduction (e.g., 1.2 GB -> 95 MB) | Eliminates compilers (`gcc`), headers, and package managers | Moderate | **Mandatory** across all production systems. |
| **`jemalloc` Preload** | 25-40% reduction in long-running heap bloat | N/A | Low (one `ENV` variable) | **Mandatory** for Ruby MRI and high-concurrency Node.js. |
| **Next.js Standalone** | Extracts only used `node_modules`; drops devDependencies | Eliminates dev-tool CVEs | Requires `output: 'standalone'` in config | **Mandatory** for self-hosted containerized Next.js. |
| **Non-Root `USER 10001`** | Prevents host port usurpation (<1024) | Mitigates container escape vulnerability chains | Moderate (must manage directory ownership) | **Mandatory** for CIS compliance and Kubernetes security policies. |

---

### 5.6 Senior Interview Q&A

#### Q: In a Ruby on Rails Dockerfile, why is `SECRET_KEY_BASE_DUMMY=1` passed during `assets:precompile`? What failure mode does this prevent?
**Answer**:
Rails asset precompilation (`bin/rails assets:precompile`) initializes the Rails application environment, loading `config/application.rb` and all initializers to evaluate routes, engines, and view helpers.
In production mode (`RAILS_ENV=production`), Rails requires `SECRET_KEY_BASE` to be present; otherwise, initialization raises a fatal `ArgumentError: Missing required secret_key_base`.
If developers attempt to pass the actual production `SECRET_KEY_BASE` during image build time via `ARG` or `ENV`, that master cryptographic secret gets **permanently baked into the image layers**, allowing anyone with image pull access to decrypt session cookies and forge signed authentication tokens.
Passing `SECRET_KEY_BASE_DUMMY=1` instructs Rails to use a temporary, fake cryptographic key purely to satisfy application boot requirements during static asset compilation, ensuring zero secrets are leaked into the built artifact.
