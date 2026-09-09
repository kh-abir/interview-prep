# 02. Kubernetes Architecture, Workloads & Operational Mechanics

> **Target Role**: Staff / Senior Backend & Infrastructure Engineer  
> **Module**: 08-docker-devops-aws / 02-kubernetes-fundamentals  
> **Key Focus**: Control Plane & Data Plane Architecture, Pod Internals (`pause` container), Service Routing (`kube-proxy` iptables vs IPVS), Health Probes & Cascading Failure Prevention, Zero-Downtime Deployments (`preStop` hooks), HPA Autoscaling Algorithms, and Helm Package Engineering.

---

## Table of Contents
1. [Kubernetes Architecture & Core Workload Objects](#1-kubernetes-architecture--core-workload-objects)
   - [1.1 Definition & Core Concept](#11-definition--core-concept)
   - [1.2 Internal Mechanics: Control Plane, Kubelet & The Pause Container](#12-internal-mechanics-control-plane-kubelet--the-pause-container)
   - [1.3 Production Manifests: Pod, Deployment, ConfigMap, Secret & PVC](#13-production-manifests-pod-deployment-configmap-secret--pvc)
   - [1.4 Production Outages & Debugging](#14-production-outages--debugging)
   - [1.5 Trade-offs & Decision Matrix](#15-trade-offs--decision-matrix)
   - [1.6 Senior Interview Q&A](#16-senior-interview-qa)
2. [Kubernetes Networking, Services & Ingress Controllers](#2-kubernetes-networking-services--ingress-controllers)
   - [2.1 Definition & Core Concept](#21-definition--core-concept)
   - [2.2 Internal Mechanics: kube-proxy (iptables vs IPVS) & Packet Flow](#22-internal-mechanics-kube-proxy-iptables-vs-ipvs--packet-flow)
   - [2.3 Production Service & Ingress Configurations](#23-production-service--ingress-configurations)
   - [2.4 Production Outages & Debugging](#24-production-outages--debugging)
   - [2.5 Trade-offs & Decision Matrix](#25-trade-offs--decision-matrix)
   - [2.6 Senior Interview Q&A](#26-senior-interview-qa)
3. [Pod Lifecycle, Health Probes & Cascading Failure Prevention](#3-pod-lifecycle-health-probes--cascading-failure-prevention)
   - [3.1 Definition & Core Concept](#31-definition--core-concept)
   - [3.2 Internal Mechanics: Startup, Readiness & Liveness Probes](#32-internal-mechanics-startup-readiness--liveness-probes)
   - [3.3 Production Probe Configurations & App Handlers](#33-production-probe-configurations--app-handlers)
   - [3.4 Production Outages & Debugging](#34-production-outages--debugging)
   - [3.5 Trade-offs & Decision Matrix](#35-trade-offs--decision-matrix)
   - [3.6 Senior Interview Q&A](#36-senior-interview-qa)
4. [Zero-Downtime Deployments & Graceful Termination](#4-zero-downtime-deployments--graceful-termination)
   - [4.1 Definition & Core Concept](#41-definition--core-concept)
   - [4.2 Internal Mechanics: RollingUpdate Math & Endpoint Deregistration Races](#42-internal-mechanics-rollingupdate-math--endpoint-deregistration-races)
   - [4.3 Production Zero-Downtime Deployment Manifest](#43-production-zero-downtime-deployment-manifest)
   - [4.4 Production Outages & Debugging](#44-production-outages--debugging)
   - [4.5 Trade-offs & Decision Matrix](#45-trade-offs--decision-matrix)
   - [4.6 Senior Interview Q&A](#46-senior-interview-qa)
5. [Horizontal Pod Autoscaling (HPA) & Helm Engineering](#5-horizontal-pod-autoscaling-hpa--helm-engineering)
   - [5.1 Definition & Core Concept](#51-definition--core-concept)
   - [5.2 Internal Mechanics: HPA Controller Algorithm & Helm Templating Engine](#52-internal-mechanics-hpa-controller-algorithm--helm-templating-engine)
   - [5.3 Production HPA & Helm Chart Definitions](#53-production-hpa--helm-chart-definitions)
   - [5.4 Production Outages & Debugging](#54-production-outages--debugging)
   - [5.5 Trade-offs & Decision Matrix](#55-trade-offs--decision-matrix)
   - [5.6 Senior Interview Q&A](#56-senior-interview-qa)

---

# 1. Kubernetes Architecture & Core Workload Objects

### 1.1 Definition & Core Concept
Kubernetes (K8s) is a distributed cluster operating system that orchestrates containerized workloads across pools of physical or virtual machines:
- **Control Plane**: Manages cluster state, schedules workloads, and detects/corrects drift via reconciliation loops. Consists of `etcd` (distributed key-value store), `kube-apiserver` (the REST gateway), `kube-scheduler` (assigns Pods to nodes), and `kube-controller-manager` (runs ReplicaSet, Node, and Endpoint controllers).
- **Data Plane (Worker Nodes)**: Executes application containers. Runs `kubelet` (node agent communicating with CRI), `kube-proxy` (manages network routing rules), and the Container Runtime (e.g., `containerd`).
- **Core Objects**:
  - **Pod**: The atomic, indivisible unit of scheduling. One or more containers sharing network, IPC, and storage namespaces.
  - **Deployment**: Declarative controller managing rolling updates and rollbacks of stateless Pods via underlying **ReplicaSets**.
  - **ConfigMap & Secret**: Inject configuration parameters and sensitive credentials.
  - **PersistentVolume (PV) & PersistentVolumeClaim (PVC)**: Decouple storage provisioning from consumption.

```
+-----------------------------------------------------------------------------------+
|                               CONTROL PLANE                                       |
|                                                                                   |
|   +-------------+       +-------------------+       +-------------------------+   |
|   |    etcd     |<----->|  kube-apiserver   |<----->| kube-controller-manager |   |
|   |  (Raft DB)  |       |  (JSON REST API)  |       | (Reconciliation Loops)  |   |
|   +-------------+       +-------------------+       +-------------------------+   |
|                                  ^                                                |
|                                  |                                                |
|                                  v                                                |
|                         +------------------+                                      |
|                         |  kube-scheduler  |                                      |
|                         | (Filters & Ranks)|                                      |
|                         +------------------+                                      |
+-----------------------------------------------------------------------------------+
                                   | (HTTPS gRPC)
          +------------------------+------------------------+
          |                                                 |
          v                                                 v
+-----------------------------------+     +-----------------------------------+
|       WORKER NODE 1               |     |       WORKER NODE 2               |
|                                   |     |                                   |
|   +---------------------------+   |     |   +---------------------------+   |
|   |          kubelet          |   |     |   |          kubelet          |   |
|   +---------------------------+   |     |   +---------------------------+   |
|     | (CRI gRPC)                  |     |     | (CRI gRPC)                  |
|     v                             |     |     v                             |
|   +---------------------------+   |     |   +---------------------------+   |
|   |        containerd         |   |     |   |        containerd         |   |
|   +---------------------------+   |     |   +---------------------------+   |
|     |                             |     |     |                             |
|     v                             |     |     v                             |
|   +---------------------------+   |     |   +---------------------------+   |
|   | Pod (pause + App Container)|  |     |   | Pod (pause + App Container)|  |
|   +---------------------------+   |     |   +---------------------------+   |
|                                   |     |                                   |
|   +---------------------------+   |     |   +---------------------------+   |
|   |   kube-proxy (iptables)   |   |     |   |   kube-proxy (iptables)   |   |
|   +---------------------------+   |     |   +---------------------------+   |
+-----------------------------------+     +-----------------------------------+
```

---

### 1.2 Internal Mechanics: Control Plane, Kubelet & The Pause Container

#### 1. Pod Internals & The `pause` Container (`k8s.gcr.io/pause`)
A Pod is not a single container. When a Pod is scheduled to a worker node:
1. `kubelet` instructs the CRI runtime to create and run an **infrastructure container called `pause`**.
2. The `pause` container executes a tiny assembly loop that makes the `pause(2)` system call to sleep indefinitely and handle `SIGCHLD`.
3. The `pause` container holds the **Linux Network (`CLONE_NEWNET`) and IPC (`CLONE_NEWIPC`) namespaces**.
4. When application containers (e.g., Rails web app and Envoy sidecar) are launched inside that Pod, they join the **exact same network and IPC namespaces** created by `pause` (`setns` into `/proc/<pause_pid>/ns/net`).
5. **Architectural Result**:
   - All containers in the same Pod share the exact same IP address (`eth0`).
   - Containers communicate with each other over `localhost` (`127.0.0.1`) at native kernel memory speed.
   - Ports cannot collide: if container A binds to 8080, container B cannot bind to 8080.
   - If an application container crashes and restarts, the network IP and open sockets remain intact because the `pause` container never exited.

```
+---------------------------------------------------------------------------------+
|                                    POD BOUNDARY                                 |
|                                                                                 |
|   +-------------------------------------------------------------------------+   |
|   |                       pause container (PID 1)                           |   |
|   |          Holds: Network Namespace (IP: 10.244.1.45) & IPC Namespace     |   |
|   +-------------------------------------------------------------------------+   |
|            ^                                                       ^            |
|            | (joins NET & IPC ns)                                  |            |
|            v                                                       v            |
|   +---------------------------------+     +---------------------------------+   |
|   |    Container 1: Web (Puma)      |     |   Container 2: Envoy / Datadog  |   |
|   |    Listens on 127.0.0.1:3000    |<--->|   Forwards from 127.0.0.1:3000  |   |
|   +---------------------------------+     +---------------------------------+   |
|            \                                                       /            |
|             +-----------------------+-----------------------------+             |
|                                     v                                           |
|                      Shared Volume: emptyDir / PVC                              |
+---------------------------------------------------------------------------------+
```

#### 2. ConfigMap & Secret: Volume Mount vs. Environment Variable
Injected configuration parameters can be supplied via Environment Variables or Volume Mounts:

| Feature | Injected via `envFrom` / `valueFrom` | Injected via Volume Mount (`mountPath: /etc/secrets`) |
|:---|:---|:---|
| **Live Updates (Hot-reloading)** | **NO**. Process environment is immutable once initialized via `execve`. Requires Pod restart to pick up changes. | **YES**. `kubelet` periodically refreshes mounted projections from `etcd` (atomic update via symlink rotation). |
| **Exposure Risks** | **HIGH**. Child processes inherit env vars; crash dumps, `/proc/<pid>/environ`, and application debug endpoints often leak all environment variables. | **LOW**. Encapsulated in the filesystem; protected by file access permissions (`defaultMode: 0400`). |
| **Payload Size Limits** | Limited by OS environment block limit (`ARG_MAX`, typically 2MB). | Standard etcd object limit (1MB default per Secret/ConfigMap). |

---

### 1.3 Production Manifests: Pod, Deployment, ConfigMap, Secret & PVC

```yaml
# ------------------------------------------------------------------------------
# 1. ConfigMap: Application Non-sensitive Config
# ------------------------------------------------------------------------------
apiVersion: v1
kind: ConfigMap
metadata:
  name: api-config
  namespace: production
data:
  RAILS_ENV: "production"
  PORT: "3000"
  CACHE_TTL_SECONDS: "3600"
---
# ------------------------------------------------------------------------------
# 2. Secret: Production Credentials (Encrypted / Base64)
# ------------------------------------------------------------------------------
apiVersion: v1
kind: Secret
metadata:
  name: api-secrets
  namespace: production
type: Opaque
data:
  # echo -n "postgres://user:pass@pg-prod:5432/db" | base64
  DATABASE_URL: cG9zdGdyZXM6Ly91c2VyOnBhc3NAcGctcHJvZDoxNDMyL2Ri
  # echo -n "supersecretmasterkey123" | base64
  SECRET_KEY_BASE: c3VwZXJzZWNyZXRtYXN0ZXJrZXkxMjM=
---
# ------------------------------------------------------------------------------
# 3. PersistentVolumeClaim: Fast NVMe EBS Storage (gp3)
# ------------------------------------------------------------------------------
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: uploads-pvc
  namespace: production
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: gp3-ebs
  resources:
    requests:
      storage: 100Gi
---
# ------------------------------------------------------------------------------
# 4. Production Deployment: High Availability & Security Context
# ------------------------------------------------------------------------------
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-deployment
  namespace: production
  labels:
    app.kubernetes.io/name: api-service
    app.kubernetes.io/part-of: core-platform
spec:
  replicas: 4
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: api-service
  template:
    metadata:
      labels:
        app: api-service
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 10001
        runAsGroup: 10001
        fsGroup: 10001
        seccompProfile:
          type: RuntimeDefault
      containers:
        - name: web
          image: 123456789012.dkr.ecr.us-east-1.amazonaws.com/api-service:c4f1e0a
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 3000
              name: http
          envFrom:
            - configMapRef:
                name: api-config
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: api-secrets
                  key: DATABASE_URL
          volumeMounts:
            - name: secrets-volume
              mountPath: /etc/app-secrets
              readOnly: true
            - name: storage-volume
              mountPath: /app/storage
          resources:
            requests:
              cpu: "500m"
              memory: "512Mi"
            limits:
              cpu: "1000m"
              memory: "1024Mi"
      volumes:
        - name: secrets-volume
          secret:
            secretName: api-secrets
            defaultMode: 0400
        - name: storage-volume
          persistentVolumeClaim:
            claimName: uploads-pvc
```

---

### 1.4 Production Outages & Debugging

#### Outage Case Study: The Base64 Secret Leak & OOM Thrashed Worker Nodes
- **Context**: A team checked Kubernetes Secret manifests into a public Git repo believing Base64 was encryption. Later, during heavy batch processing, nodes began entering `NotReady` states.
- **Root Cause**:
  1. Base64 is merely an encoding scheme (`base64 -d`), **not encryption**. Anyone with read access to the repo or YAML retrieved plaintext production database credentials.
  2. The Pods had no `resources.limits.memory` configured. A memory leak in one Ruby worker consumed 60 GB of RAM on a 64 GB worker node. The Linux kernel OOM killer began killing critical host processes including `containerd` and `kubelet`, dropping the entire node offline.
- **Remediation**:
  1. Rotated all leaked database credentials immediately.
  2. Implemented **Sealed Secrets** or **AWS Secrets Manager Operator (External Secrets Operator - ESO)** to inject credentials at runtime without plaintext Git commits.
  3. Enforced namespace `LimitRange` and `ResourceQuota` policies requiring all Pods to define both `requests` and `limits`.

---

### 1.5 Trade-offs & Decision Matrix

| Storage / Config Model | Performance | Security | Dynamic Reload | Production Recommendation |
|:---|:---|:---|:---|:---|
| **Secret as Env Var** | Zero disk overhead | High risk of leakage in process dumps | Requires Pod restart | Avoid for high-security credentials. Acceptable for trivial tokens. |
| **Secret as Volume Mount** | Microsecond disk read (backed by tmpfs) | Highly secure (restricted Unix permissions) | Atomic update via symlink swap | **Production Standard** for SSL certs, private keys, API credentials. |
| **EBS PersistentVolume (gp3)** | High IOPS (3,000 baseline) | Bound to single AWS Availability Zone | `ReadWriteOnce` only (1 node at a time) | Production standard for stateful single-instance DBs. |
| **EFS PersistentVolume (NFS)** | Lower IOPS, higher latency | Shared across entire VPC & AZs | `ReadWriteMany` (hundreds of Pods concurrently) | Shared media uploads, legacy file shares. |

---

### 1.6 Senior Interview Q&A

#### Q: How does Kubernetes guarantee atomic Secret and ConfigMap updates when mounted as volumes, and why does an application reading a mounted file never see partial writes?
**Answer**:
When `kubelet` syncs an updated Secret or ConfigMap from `etcd`, it does not write directly to the target file. Instead, it uses **symlink swapping**:
1. It creates a new timestamped directory: `..data_2026_09_09_12_00_00`.
2. It writes the updated files into this new directory.
3. It creates a temporary symlink `..data_tmp` pointing to `..data_2026_09_09_12_00_00`.
4. It calls `rename(2)` to atomically swap the symlink `..data` to point to `..data_tmp`.
5. The container's application files (`my-secret.txt`) are symlinks pointing through `..data/my-secret.txt`.
Because the Linux `rename(2)` system call is atomic at the VFS kernel level, any reading process will either read the entire old file or the entire new file, completely preventing torn reads or partial content corruption.

---

# 2. Kubernetes Networking, Services & Ingress Controllers

### 2.1 Definition & Core Concept
Kubernetes enforces a flat network model: **Every Pod gets a unique, routable IP address**, and any Pod can communicate with any other Pod without NAT.
Because Pods are ephemeral (created and destroyed during rollouts or crashes, changing their IPs), the **Service** object acts as a stable Layer 4 virtual abstraction:
- **ClusterIP (Default)**: Virtual internal IP accessible only from within the cluster.
- **NodePort**: Allocates a static port (default: 30000-32767) on every worker node's physical IP address.
- **LoadBalancer**: Provisions an external cloud load balancer (e.g., AWS NLB/ALB) routing to the NodePort/ClusterIP.
- **Ingress Controller**: Layer 7 reverse proxy (NGINX, Traefik, AWS ALB Controller) routing external HTTP/HTTPS traffic to Services based on hostnames (`api.example.com`) and paths (`/v1`).

---

### 2.2 Internal Mechanics: kube-proxy (iptables vs IPVS) & Packet Flow

`kube-proxy` runs on every worker node, watching the `kube-apiserver` for Service and EndpointSlice mutations.

```
Client Pod (10.244.1.15)
       |
       | Requests ClusterIP: http://10.96.0.10:80
       v
Linux Kernel Netfilter / IPVS (on Worker Node)
       |
       |-- [iptables mode]: Evaluates sequential chain of probabilistic rules
       |-- [IPVS mode]: O(1) Hash table lookup (Round Robin / Least Conns)
       v
Rewrites Destination IP (DNAT): 10.96.0.10 -> 10.244.2.88 (Pod 2 IP)
       |
       v
Routed across CNI overlay / VPC subnet to Target Pod
```

#### iptables Mode vs. IPVS Mode:

##### 1. iptables Mode:
- `kube-proxy` writes sequential chains (`KUBE-SERVICES`, `KUBE-SVC-XXX`, `KUBE-SEP-XXX`).
- Load balancing is simulated using probabilistic matching modules (`statistic --mode random --probability 0.33333`).
- **Complexity**: $O(N)$ lookup time where $N$ is the number of backend services/endpoints.
- **Performance Wall**: When a cluster scales to >5,000 services or 40,000 pods, iptables rule sync freezes node CPUs, and packet processing introduces millisecond delays.

##### 2. IPVS Mode (IP Virtual Server):
- Utilizes the Linux Netfilter IPVS transport-layer load balancer running inside the Linux kernel.
- **Complexity**: $O(1)$ lookup using kernel **hash tables**.
- Supports advanced load balancing algorithms: Round-Robin (`rr`), Least Connections (`lc`), Destination Hashing (`dh`), Source Hashing (`sh`).
- Scalable to tens of thousands of services with zero degradation.

---

### 2.3 Production Service & Ingress Configurations

```yaml
# ------------------------------------------------------------------------------
# 1. ClusterIP Service targeting Application Pods
# ------------------------------------------------------------------------------
apiVersion: v1
kind: Service
metadata:
  name: api-service
  namespace: production
spec:
  type: ClusterIP
  selector:
    app: api-service
  ports:
    - name: http
      port: 80
      targetPort: 3000
---
# ------------------------------------------------------------------------------
# 2. Production Ingress-NGINX with TLS Termination & Rate Limiting
# ------------------------------------------------------------------------------
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-ingress
  namespace: production
  annotations:
    kubernetes.io/ingress.class: "nginx"
    cert-manager.io/cluster-issuer: "letsencrypt-production"
    nginx.ingress.kubernetes.io/proxy-body-size: "20m"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "60"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "60"
    nginx.ingress.kubernetes.io/limit-rps: "100"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  tls:
    - hosts:
        - api.enterprise.com
      secretName: api-enterprise-tls
  rules:
    - host: api.enterprise.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: api-service
                port:
                  number: 80
```

---

### 2.4 Production Outages & Debugging

#### Outage Case Study: The 5,000-Service iptables Freeze
- **Context**: A large-scale microservices platform scaled to 6,200 Services. Node CPU usage surged to 100% on worker nodes across the cluster. Pod network calls began failing with random connection timeouts.
- **Investigation**:
  1. Profiling the Linux kernel via `perf top` showed 80% CPU time spent in `iptables-restore` and kernel lock contention inside `nf_conntrack`.
  2. Every time an Endpoint changed, `kube-proxy` regenerated a 12 MB iptables ruleset and locked the kernel table to write it.
- **Resolution**:
  1. Switched `kube-proxy` from `iptables` to `ipvs`:
     ```yaml
     # kube-proxy ConfigMap
     mode: "ipvs"
     ipvs:
       scheduler: "rr"
     ```
  2. Loaded Linux kernel modules on all nodes: `modprobe ip_vs ip_vs_rr ip_vs_wrr ip_vs_sh nf_conntrack`.
  3. CPU consumption dropped from 100% to 2%, and network routing latency dropped to <0.2ms.

---

### 2.5 Trade-offs & Decision Matrix

| Routing Model | Protocol Layer | Scale Limit | Advanced Routing (Canary/Headers) | Production Best Fit |
|:---|:---|:---|:---|:---|
| **kube-proxy (iptables)** | Layer 4 (TCP/UDP) | ~2,000 Services | No | Small to mid-size clusters. |
| **kube-proxy (IPVS)** | Layer 4 (TCP/UDP) | >50,000 Services | No | Large-scale enterprise clusters. |
| **Ingress-NGINX** | Layer 7 (HTTP/HTTPS) | High | Yes (annotations, regex, rate limiting) | Production standard for web APIs and microservices. |
| **AWS ALB Ingress Controller** | Layer 7 (HTTP/HTTPS) | Cloud managed | Yes (integrates natively with AWS WAF & ACM) | AWS-native deployments prioritizing serverless L7 offloading. |

---

### 2.6 Senior Interview Q&A

#### Q: How does kube-proxy handle connection draining when a Pod is terminating, and why do clients sometimes get HTTP 502 Bad Gateway during rollouts?
**Answer**:
When a Pod begins termination, two asynchronous events happen in parallel:
1. **Kubelet** sends `SIGTERM` to the container and starts the grace period timer.
2. **Endpoint Controller** removes the Pod IP from the Service's `EndpointSlice` and notifies `kube-proxy` on all nodes to prune the iptables/IPVS rules.
Because iptables rule distribution across hundreds of nodes can take between 1 to 5 seconds, **worker nodes continue forwarding new incoming HTTP requests to the terminating Pod while it is already shutting down or dead**. The application returns `ECONNREFUSED`, manifesting as HTTP 502 at the ingress load balancer.
To fix this, a `preStop` hook with `sleep 5` must be added to the Pod spec, forcing the container to delay `SIGTERM` processing until the network routing tables across the cluster have completely removed its IP.

---

# 3. Pod Lifecycle, Health Probes & Cascading Failure Prevention

### 3.1 Definition & Core Concept
Kubernetes provides three distinct health probes executed periodically by `kubelet`:
1. **Startup Probe**: Determines if the application container has completed its initial boot sequence. All other probes are disabled until the startup probe succeeds. Protects slow-starting legacy apps from being killed prematurely.
2. **Readiness Probe**: Determines if the Pod is ready to accept incoming network traffic. If it fails, `kube-proxy` immediately strips the Pod IP from the Service Endpoints. The container is **NOT** restarted.
3. **Liveness Probe**: Determines if the container process is healthy and active. If it fails, `kubelet` terminates the container and initiates a restart based on `restartPolicy`.

```
Pod State: Container Created
       |
       v
+-----------------------------+
|    Startup Probe Running    | <----+
+-----------------------------+      | (Fails: Retries until failureThreshold)
       | (Succeeds!)                 | If exceeded: RESTART CONTAINER
       v
+-------------------------------------------------------------+
|                      Container Running                      |
|                                                             |
|   +--------------------------+  +------------------------+  |
|   |  Readiness Probe Checks  |  |  Liveness Probe Checks |  |
|   +--------------------------+  +------------------------+  |
|          |              |              |             |      |
|     (Passes)        (Fails)        (Passes)      (Fails)    |
|        |                |              |             |      |
|        v                v              v             v      |
|   [Keep in Service] [Remove from    [Keep Running] [SIGKILL |
|                      Endpoints]                     Restart]|
+-------------------------------------------------------------+
```

---

### 3.2 Internal Mechanics: Startup, Readiness & Liveness Probes

#### The Cascading Restart Loop (Thundering Herd Outage)
The single most common Kubernetes production disaster is a **misconfigured Liveness Probe that checks downstream dependencies (e.g., PostgreSQL or Redis)**:

```
[Traffic Spike] 
      │
      ▼
[Postgres CPU Hits 100%] 
      │
      ▼
[App DB queries slow down from 5ms to 8000ms]
      │
      ▼
[Liveness Probe checks /healthz (which pings DB)] 
      │
      ▼
[Liveness Probe times out after 3s on Pod 1] 
      │
      ▼
[Kubelet sends SIGKILL and restarts Pod 1] 
      │
      ▼
[Pod 1 is OFFLINE! Total capacity drops from 4 Pods to 3 Pods]
      │
      ▼
[Incoming traffic is redistributed across remaining 3 Pods]
      │
      ▼
[Remaining Pods get overwhelmed -> Liveness Probes fail -> Pods 2, 3, 4 KILLED]
      │
      ▼
=========================================================
== COMPLETE CLUSTER-WIDE CASCADING SYSTEM COLLAPSE     ==
=========================================================
```

##### Core Architecture Rule:
- **Liveness Probes must ONLY verify internal process liveliness** (e.g., event loop not blocked, deadlocks absent, memory within margins). **NEVER check external databases, caches, or third-party APIs in a liveness probe**.
- **Readiness Probes** can check if local database connection pools are active to shed traffic gracefully, but must degrade softly without causing node evictions.

---

### 3.3 Production Probe Configurations & App Handlers

#### 1. Production Kubernetes Manifest Spec
```yaml
spec:
  containers:
    - name: api
      image: api-service:v2.4.0
      ports:
        - containerPort: 3000
      # 1. Startup Probe: Allows up to 60 seconds (30 * 2s) for cold boot & migrations
      startupProbe:
        httpGet:
          path: /health/startup
          port: 3000
        initialDelaySeconds: 2
        periodSeconds: 2
        timeoutSeconds: 2
        failureThreshold: 30

      # 2. Readiness Probe: Checks if local process has free worker threads
      readinessProbe:
        httpGet:
          path: /health/ready
          port: 3000
        periodSeconds: 5
        timeoutSeconds: 2
        successThreshold: 1
        failureThreshold: 3

      # 3. Liveness Probe: Pure internal loop ping (Deadlock / crash detection only)
      livenessProbe:
        httpGet:
          path: /health/live
          port: 3000
        periodSeconds: 10
        timeoutSeconds: 3
        failureThreshold: 3
```

#### 2. Node.js Express Probe Implementation
```typescript
import express, { Request, Response } from 'express';

const app = express();
let isShuttingDown = false;

// Signal handling hook
process.on('SIGTERM', () => {
  isShuttingDown = true;
});

// 1. Startup: App has completed initialization
app.get('/health/startup', (_req: Request, res: Response) => {
  res.status(200).send('OK');
});

// 2. Liveness: Internal loop health only (Zero external calls!)
app.get('/health/live', (_req: Request, res: Response) => {
  if (isShuttingDown) {
    return res.status(503).send('Shutting down');
  }
  // Event loop tick check
  return res.status(200).send('Alive');
});

// 3. Readiness: Checks if ready to receive customer traffic
app.get('/health/ready', (req: Request, res: Response) => {
  if (isShuttingDown) {
    return res.status(503).send('Not accepting connections');
  }
  // Optional: check internal connection pool availability
  res.status(200).send('Ready');
});
```

---

### 3.4 Production Outages & Debugging

#### Outage Case Study: The 1-Second Timeout Probe Loop
- **Context**: An enterprise Rails application experienced random container restarts during peak business hours. `kubectl describe pod` revealed:
  `Liveness probe failed: Get "http://10.244.1.20:3000/health": context deadline exceeded (Client.Timeout exceeded while awaiting headers)`.
- **Root Cause**: The probe had `timeoutSeconds: 1`. Puma had 5 worker threads. During peak load, all 5 threads were saturated processing 800ms database queries. The incoming `/health` HTTP probe was queued in the Puma socket backlog, exceeding the 1-second timeout. `kubelet` killed perfectly healthy containers under load, exacerbating the queue.
- **Resolution**:
  1. Increased `timeoutSeconds` to 3.
  2. Moved the health check endpoint to a lightweight internal rack middleware executed **before** Rails controller initialization.
  3. Increased `failureThreshold` from 1 to 3.

---

### 3.5 Trade-offs & Decision Matrix

| Probe Mechanism | Transport | Resource Cost | Failure Action | Best Used For |
|:---|:---|:---|:---|:---|
| **HTTP Get Probe** | HTTP/1.1 or HTTP/2 | Low | Status Code $\ge 400$ triggers failure | Standard web services and microservice APIs. |
| **TCP Socket Probe** | TCP 3-way handshake | Ultra-low | Socket connection refused | Raw TCP daemons, databases, cache proxies. |
| **Exec Probe** | Shell execution (`/bin/sh -c`) | **Extremely High** (Forks a process inside container) | Non-zero exit code triggers failure | State verification where HTTP is unavailable. **Avoid at high frequencies**. |

---

### 3.6 Senior Interview Q&A

#### Q: Why can an `exec` probe cause node-level performance collapse compared to an `httpGet` probe?
**Answer**:
When `kubelet` runs an `httpGet` probe, it makes a lightweight asynchronous socket call from the host's networking stack to the container's IP address.
When an `exec` probe is configured, `kubelet` communicates with `containerd` via CRI to invoke `runc exec`. This requires:
1. Allocating a new kernel process (`fork`).
2. Joining the container's namespaces (`setns`).
3. Spawning a shell (`/bin/sh`).
4. Parsing and executing the command.
5. Destroying the process and reaping child exit codes.
If 100 Pods run an `exec` probe every 5 seconds, the host kernel forks 1,200 new processes per minute purely for health checks, causing kernel CPU context-switch churn, PID table exhaustion, and severe CFS scheduling latency for actual application workloads.

---

# 4. Zero-Downtime Deployments & Graceful Termination

### 4.1 Definition & Core Concept
Zero-downtime deployment in Kubernetes requires coordinating two distinct subsystems:
1. **The RollingUpdate Strategy**: Replacing old Pods with new Pods incrementally, strictly constrained by `maxSurge` (how many extra Pods can be created above `replicas`) and `maxUnavailable` (how many Pods can be taken down during the rollout).
2. **Graceful Termination Coordination**: Ensuring that when an old Pod is terminated, it does not abort active HTTP requests or reject incoming packets while kube-proxy updates routing tables.

---

### 4.2 Internal Mechanics: RollingUpdate Math & Endpoint Deregistration Races

#### 1. RollingUpdate Math
Given `replicas: 10`, `maxSurge: 25%`, `maxUnavailable: 0`:
- $\text{Max Allowed Pods} = \text{replicas} + \lceil\text{replicas} \times 0.25\rceil = 10 + 3 = 13$
- $\text{Min Available Pods} = \text{replicas} - 0 = 10$
- **Step 1**: K8s spins up 3 new version Pods.
- **Step 2**: K8s waits for all 3 Pods to pass **Readiness Probes**.
- **Step 3**: Only once ready, K8s initiates termination of 3 old version Pods.
- **Step 4**: Repeat until all 10 replicas run the new image. **Zero capacity loss occurs throughout the rollout**.

#### 2. The Graceful Termination Timeline

```
Time (ms)  Event Description
0ms        API Server receives rollout update. Pod marked as 'Terminating'.
10ms       Endpoints controller detects 'Terminating', queues IP removal.
10ms       Kubelet simultaneously executes container 'preStop' hook.
           ========================================================================
           == PRE-STOP HOOK RUNS: sleep 5                                        ==
           == The container continues serving HTTP traffic normally!             ==
           ========================================================================
1500ms     Kube-proxy on all cluster nodes receives EndpointSlice update.
           Iptables rules updated: no NEW connections will reach this Pod.
5000ms     'sleep 5' finishes.
5001ms     Kubelet sends SIGTERM to container PID 1.
5002ms     App stops listening on 3000; enters graceful shutdown.
           Active inflight requests finish processing (draining connection pool).
7200ms     App finishes last HTTP request, closes DB sockets, exits with code 0.
           (If app hangs past terminationGracePeriodSeconds, Kubelet sends SIGKILL).
```

---

### 4.3 Production Zero-Downtime Deployment Manifest

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payments-api
  namespace: production
spec:
  replicas: 8
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 25%         # 2 extra pods can be created during deployment
      maxUnavailable: 0     # ZERO pods are killed until new pods are ready!
  selector:
    matchLabels:
      app: payments-api
  template:
    metadata:
      labels:
        app: payments-api
    spec:
      terminationGracePeriodSeconds: 45 # Total time allowed for shutdown
      containers:
        - name: app
          image: payments-api:v3.1.2
          lifecycle:
            preStop:
              exec:
                # Critical Production Rule: Sleep 5s to allow iptables propagation
                command: ["/bin/sh", "-c", "sleep 5"]
          ports:
            - containerPort: 3000
          readinessProbe:
            httpGet:
              path: /health/ready
              port: 3000
            periodSeconds: 3
            failureThreshold: 2
```

---

### 4.4 Production Outages & Debugging

#### Outage Case Study: The Broken Rollout & Automated Rollback
- **Context**: A bad migration was pushed. The new image crashed immediately on boot (`CrashLoopBackOff`).
- **Debugging & Rollback Workflow**:
  ```bash
  # Check deployment status
  kubectl rollout status deployment/payments-api -n production
  # Output: Waiting for deployment "payments-api" rollout to finish: 2 of 8 updated replicas are available...

  # Inspect rollout history
  kubectl rollout history deployment/payments-api -n production

  # View cause of failure
  kubectl describe pod -l app=payments-api -n production | grep -E "Exit Code|Reason"

  # INSTANT ROLLBACK to previous stable revision:
  kubectl rollout undo deployment/payments-api -n production

  # Verify rollback completion
  kubectl rollout status deployment/payments-api -n production
  # Output: deployment "payments-api" successfully rolled out
  ```

---

### 4.5 Trade-offs & Decision Matrix

| Deployment Strategy | Cost Overhead | Zero-Downtime Guarantee | Rollback Latency | Complexity |
|:---|:---|:---|:---|:---|
| **RollingUpdate (`maxUnavailable: 0`)** | +25% compute during rollout | **100%** (if `preStop` sleep is applied) | Seconds (`rollout undo`) | Low (native K8s) |
| **Recreate** | 0% | **0% (Full outage)** | Minutes | Trivial (Development only) |
| **Blue/Green (Service Swap)** | +100% compute during rollout | **100%** | Sub-second (Service selector update) | Moderate |
| **Canary (Argo Rollouts)** | Variable (5-10%) | **100%** | Automated based on error rate metrics | Advanced |

---

### 4.6 Senior Interview Q&A

#### Q: What happens if `terminationGracePeriodSeconds` expires before your application finishes draining active requests?
**Answer**:
`kubelet` enforces an immutable hard deadline. When a container enters `Terminating`:
1. The `preStop` hook executes.
2. `SIGTERM` is transmitted to PID 1.
3. A kernel timer counts down from `terminationGracePeriodSeconds` (default: 30s).
If the timer reaches zero and the application process has not terminated (e.g., waiting on a long-running report query or deadlocked thread), `kubelet` immediately issues `SIGKILL` (Signal 9).
The kernel forcibly reclaims the process's pages, all open TCP sockets are abruptly reset (`TCP RST`), and clients receive broken connection errors. In production, `terminationGracePeriodSeconds` must always be calibrated higher than the maximum expected application request timeout (e.g., if HTTP timeout is 30s, grace period should be 45s or 60s).

---

# 5. Horizontal Pod Autoscaling (HPA) & Helm Engineering

### 5.1 Definition & Core Concept
- **Horizontal Pod Autoscaler (HPA)**: Controller that automatically scales the replica count of a Deployment, StatefulSet, or ReplicaSet based on observed CPU/memory metrics or external custom business metrics (e.g., SQS queue depth, HTTP requests per second).
- **Helm**: The package manager for Kubernetes. Packages YAML manifests into parameterized, version-controlled archives called **Charts**, evaluating Go templates with input values (`values.yaml`).

---

### 5.2 Internal Mechanics: HPA Controller Algorithm & Helm Templating Engine

#### 1. The HPA Mathematical Formula
The HPA controller queries the `metrics.k8s.io` API (served by Metrics Server) every 15 seconds (default: `--horizontal-pod-autoscaler-sync-period=15s`).
It computes desired replicas using the formula:

$$\text{DesiredReplicas} = \left\lceil \text{CurrentReplicas} \times \left( \frac{\text{CurrentMetricValue}}{\text{TargetMetricValue}} \right) \right\rceil$$

##### Real-World Calculation Example:
- `CurrentReplicas`: 4
- `TargetMetricValue`: 50% CPU utilization (based on CPU `requests`)
- `CurrentMetricValue`: 80% CPU utilization
$$\text{DesiredReplicas} = \left\lceil 4 \times \left( \frac{80}{50} \right) \right\rceil = \lceil 4 \times 1.6 \rceil = \lceil 6.4 \rceil = 7$$
The HPA controller updates the Deployment spec to 7 replicas.
- **Cool-down / Stabilization Window**: By default, scaling down has a 300-second stabilization window to prevent flapping/thrashing ("oscillation") under bursty traffic.

---

### 5.3 Production HPA & Helm Chart Definitions

#### 1. Production HPA Spec with Scaling Policies
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api-deployment
  minReplicas: 4
  maxReplicas: 30
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 65
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 75
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 0 # Immediate scale up during traffic spikes
      policies:
        - type: Percent
          value: 100
          periodSeconds: 15          # Double capacity every 15s if saturated
    scaleDown:
      stabilizationWindowSeconds: 300 # Wait 5 minutes before scale down to prevent flapping
      policies:
        - type: Percent
          value: 10
          periodSeconds: 60          # Shed max 10% capacity per minute
```

#### 2. Helm Chart Architecture & Templates

```
my-api-chart/
├── Chart.yaml          # Metadata: chart version, app version, dependencies
├── values.yaml         # Default configuration variables
└── templates/
    ├── _helpers.tpl    # Go template reusable template helpers
    ├── deployment.yaml # Parameterized Deployment manifest
    ├── service.yaml    # Parameterized Service manifest
    └── hpa.yaml        # Autoscaler manifest
```

##### Production `deployment.yaml` Template:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "my-api.fullname" . }}
  labels:
    {{- include "my-api.labels" . | nindent 4 }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "my-api.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "my-api.selectorLabels" . | nindent 8 }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - containerPort: {{ .Values.service.port }}
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
```

---

### 5.4 Production Outages & Debugging

#### Outage Case Study: HPA Thrashing on Unconfigured Requests
- **Context**: An engineer enabled HPA on a deployment. The HPA stayed in `<unknown>` status and never scaled, despite traffic surging by 10x.
- **Root Cause**: The Pod specification omitted `resources.requests.cpu`. HPA calculates utilization as:
  $$\text{Utilization} = \frac{\text{Actual Usage}}{\text{Requested Value}}$$
  Because `requests.cpu` was null, division by zero prevented HPA from computing utilization.
- **Resolution**: Defined deterministic CPU and memory requests on all container specs.

---

### 5.5 Trade-offs & Decision Matrix

| Tool / Metric Type | Responsiveness | Infrastructure Cost | Operational Complexity | Production Recommendation |
|:---|:---|:---|:---|:---|
| **HPA on CPU** | Reactive (30-60s delay) | Low | Low | Baseline for compute-bound APIs. |
| **HPA on Memory** | Very Slow (Memory is sticky) | Moderate | Low | Use as safety boundary; avoid for primary scaling trigger. |
| **HPA on Custom Metrics (Prometheus)** | Predictive / Fast (e.g., SQS queue size) | Optimized | High (Requires Prometheus Adapter or KEDA) | **Mandatory** for event-driven async workers (Sidekiq, Celery). |
| **KEDA (K8s Event-driven Autoscaling)** | Instant (scales 0 -> N) | Minimal | Moderate | Standard for message queue consumers. |

---

### 5.6 Senior Interview Q&A

#### Q: How does Helm track releases, and what happens when a Helm upgrade fails midway through a release?
**Answer**:
Helm 3 stores release history directly inside Kubernetes **Secrets** within the target namespace, named `sh.helm.release.v1.<release_name>.v<revision>`.
If an upgrade fails (e.g., an invalid image or crash-looping pod fails readiness probes when `--wait` is passed):
1. The release status is marked as `FAILED`.
2. Existing modified resources may be left in an intermediate state.
3. Helm does **NOT** automatically roll back by default unless invoked with `--atomic`.
4. Using `--atomic` forces Helm to purge created resources and roll back to the last successful release revision stored in the prior Secret.
5. In production CI/CD, always execute:
   `helm upgrade --install <release> <chart> --wait --timeout 5m --atomic`.
