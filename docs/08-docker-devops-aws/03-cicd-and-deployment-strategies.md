# 03. CI/CD Pipelines, Deployment Strategies & GitOps Engineering

> **Target Role**: Staff / Senior Backend & Infrastructure Engineer  
> **Module**: 08-docker-devops-aws / 03-cicd-and-deployment-strategies  
> **Key Focus**: Production Pipeline Architecture, GitHub Actions Automation, Security Gate Enforcement (Trivy), Deployment Strategies (Rolling, Blue-Green, Canary), Feature Flag Decoupling, GitOps with ArgoCD, and Immutable Image Tagging Governance.

---

## Table of Contents
1. [Production CI/CD Pipeline Architecture & GitHub Actions](#1-production-cicd-pipeline-architecture--github-actions)
   - [1.1 Definition & Core Concept](#11-definition--core-concept)
   - [1.2 Internal Mechanics: DAG Execution, Runner Isolation & Secret Handling](#12-internal-mechanics-dag-execution-runner-isolation--secret-handling)
   - [1.3 Complete Production GitHub Actions Workflow](#13-complete-production-github-actions-workflow)
   - [1.4 Production Outages & Debugging](#14-production-outages--debugging)
   - [1.5 Trade-offs & Decision Matrix](#15-trade-offs--decision-matrix)
   - [1.6 Senior Interview Q&A](#16-senior-interview-qa)
2. [Advanced Deployment Strategies: Rolling, Blue-Green & Canary](#2-advanced-deployment-strategies-rolling-blue-green--canary)
   - [2.1 Definition & Core Concept](#21-definition--core-concept)
   - [2.2 Internal Mechanics: Traffic Weighting & Routing Plane Manipulations](#22-internal-mechanics-traffic-weighting--routing-plane-manipulations)
   - [2.3 Production Implementation Configurations](#23-production-implementation-configurations)
   - [2.4 Production Outages & Debugging](#24-production-outages--debugging)
   - [2.5 Trade-offs & Decision Matrix](#25-trade-offs--decision-matrix)
   - [2.6 Senior Interview Q&A](#26-senior-interview-qa)
3. [Feature Flags: Decoupling Deployment from Release](#3-feature-flags-decoupling-deployment-from-release)
   - [3.1 Definition & Core Concept](#31-definition--core-concept)
   - [3.2 Internal Mechanics: Evaluation Engine, Local In-Memory Caching & Polling/SSE](#32-internal-mechanics-evaluation-engine-local-in-memory-caching--pollingsse)
   - [3.3 Production Code Implementations (TypeScript & Ruby)](#33-production-code-implementations-typescript--ruby)
   - [3.4 Production Outages & Debugging](#34-production-outages--debugging)
   - [3.5 Trade-offs & Decision Matrix](#35-trade-offs--decision-matrix)
   - [3.6 Senior Interview Q&A](#36-senior-interview-qa)
4. [GitOps Architecture with ArgoCD](#4-gitops-architecture-with-argocd)
   - [4.1 Definition & Core Concept](#41-definition--core-concept)
   - [4.2 Internal Mechanics: The Reconciliation Loop, Drift Detection & Self-Healing](#42-internal-mechanics-the-reconciliation-loop-drift-detection--self-healing)
   - [4.3 Production ArgoCD Application & Sync Policy Manifests](#43-production-argocd-application--sync-policy-manifests)
   - [4.4 Production Outages & Debugging](#44-production-outages--debugging)
   - [4.5 Trade-offs & Decision Matrix](#45-trade-offs--decision-matrix)
   - [4.6 Senior Interview Q&A](#46-senior-interview-qa)
5. [Docker Image Tagging Governance & The `:latest` Anti-Pattern](#5-docker-image-tagging-governance--the-latest-anti-pattern)
   - [5.1 Definition & Core Concept](#51-definition--core-concept)
   - [5.2 Internal Mechanics: Content-Addressable Blobs & Immutable SHA Digests](#52-internal-mechanics-content-addressable-blobs--immutable-sha-digests)
   - [5.3 Production CI Tagging & Verification Automation](#53-production-ci-tagging--verification-automation)
   - [5.4 Production Outages & Debugging](#54-production-outages--debugging)
   - [5.5 Trade-offs & Decision Matrix](#55-trade-offs--decision-matrix)
   - [5.6 Senior Interview Q&A](#56-senior-interview-qa)

---

# 1. Production CI/CD Pipeline Architecture & GitHub Actions

### 1.1 Definition & Core Concept
Continuous Integration and Continuous Deployment (CI/CD) automates the transition from committed code to verified production execution. A staff-level pipeline enforces a strict **Directed Acyclic Graph (DAG)** pipeline topology with fail-fast gates:
1. **Static Analysis & Linting**: Syntax, type checking (`tsc`), style (`rubocop`, `eslint`).
2. **Automated Unit & Integration Testing**: Test execution against ephemeral backing services (PostgreSQL, Redis) in parallel containers.
3. **Container Image Build & Multi-Arch Compilation**: BuildKit caching with layer re-use.
4. **Static Application Security Testing (SAST) & Vulnerability Scanning**: Scanning image layers for Common Vulnerabilities and Exposures (CVEs) with **Trivy**.
5. **Image Attestation & Push**: Publishing signed, immutable images to AWS ECR.
6. **Deployment & GitOps Sync**: Triggering blue-green/canary or ArgoCD reconciliation.
7. **Automated Smoke Testing**: Probing edge ingress endpoints to verify live routing before marking deployment as successful.

```
[Git Commit: SHA 4a7c1b]
          │
          ▼
┌──────────────────┐
│  Lint & Security │
│  (ESLint/Trivy)  │
└─────────┬────────┘
          │ (Pass)
          ▼
┌──────────────────┐
│  Unit & Integr.  │ ◄─── Spin up ephemeral Postgres & Redis
│  Tests (Parallel)│
└─────────┬────────┘
          │ (Pass)
          ▼
┌──────────────────┐
│ Docker BuildKit  │ ◄─── Cache mount: GitHub Actions Cache / ECR Registry Cache
│ (Multi-stage)    │
└─────────┬────────┘
          │ (Built)
          ▼
┌──────────────────┐
│ Vulnerability    │ ◄─── Trivy: Block deployment on CRITICAL/HIGH CVEs
│ Scan Gate        │
└─────────┬────────┘
          │ (No Critical CVEs)
          ▼
┌──────────────────┐
│ Push to AWS ECR  │ ◄─── Tag: 4a7c1b (Immutable SHA)
└─────────┬────────┘
          │
          ▼
┌──────────────────┐
│ Deploy Workload  │ ◄─── Update GitOps Repo / ECS Task Definition
└─────────┬────────┘
          │
          ▼
┌──────────────────┐
│ Smoke Test Probe │ ◄─── Query https://api.prod.com/healthz
└──────────────────┘
```

---

### 1.2 Internal Mechanics: DAG Execution, Runner Isolation & Secret Handling

#### 1. GitHub Actions Execution Topology
- **Runners**: Ephemeral virtual machines (Ubuntu 22.04 LTS). Each job (`job_id`) executes on a clean host with isolated disk, memory, and networking.
- **Dependencies (`needs`)**: Jobs execute in parallel unless explicitly serialized via `needs: [job_a, job_b]`.
- **OpenID Connect (OIDC) Authentication**: Avoids long-lived AWS IAM Access Keys. The GitHub Actions runner generates an ephemeral JWT token signed by GitHub's CA. AWS Security Token Service (STS) validates the token via `sts:AssumeRoleWithWebIdentity` and exchanges it for a 1-hour temporary AWS session credential.

---

### 1.3 Complete Production GitHub Actions Workflow

```yaml
# .github/workflows/production-pipeline.yml
name: Production Deployment Pipeline

on:
  push:
    branches:
      - main
    paths-ignore:
      - '**.md'
      - 'docs/**'

permissions:
  id-token: write   # Required for AWS OIDC authentication
  contents: read    # Checkout repository code
  security-events: write # Upload Trivy SARIF results to GitHub Security tab

env:
  AWS_REGION: "us-east-1"
  ECR_REPOSITORY: "production-core-api"
  ROLE_TO_ASSUME: "arn:aws:iam::123456789012:role/github-actions-ecr-deployer"

jobs:
  # ----------------------------------------------------------------------------
  # JOB 1: Lint & Code Quality
  # ----------------------------------------------------------------------------
  lint:
    name: Code Quality & Type Check
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Code
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'npm'

      - name: Install Dependencies
        run: npm ci

      - name: Run Typecheck
        run: npm run typecheck

      - name: Run Linter
        run: npm run lint

  # ----------------------------------------------------------------------------
  # JOB 2: Automated Tests with Ephemeral Database Services
  # ----------------------------------------------------------------------------
  test:
    name: Automated Test Suite
    runs-on: ubuntu-latest
    needs: [lint]
    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_DB: app_test
          POSTGRES_USER: test_user
          POSTGRES_PASSWORD: test_password
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 5s
          --health-timeout 3s
          --health-retries 5

      redis:
        image: redis:7-alpine
        ports:
          - 6379:6379
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 5s
          --health-timeout 3s
          --health-retries 5

    steps:
      - name: Checkout Code
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'npm'

      - name: Install Dependencies
        run: npm ci

      - name: Run Tests with Coverage
        env:
          DATABASE_URL: postgres://test_user:test_password@localhost:5432/app_test
          REDIS_URL: redis://localhost:6379/0
          NODE_ENV: test
        run: npm test -- --coverage

  # ----------------------------------------------------------------------------
  # JOB 3: Build & Security Vulnerability Gate
  # ----------------------------------------------------------------------------
  build-and-scan:
    name: Build & Security Scan
    runs-on: ubuntu-latest
    needs: [test]
    outputs:
      image_tag: ${{ steps.set_tag.outputs.tag }}
      ecr_registry: ${{ steps.login-ecr.outputs.registry }}
    steps:
      - name: Checkout Code
        uses: actions/checkout@v4

      - name: Derive Immutable Commit Tag
        id: set_tag
        run: |
          TAG=$(git rev-parse --short HEAD)
          echo "tag=${TAG}" >> "$GITHUB_OUTPUT"

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Configure AWS Credentials via OIDC
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ env.ROLE_TO_ASSUME }}
          aws-region: ${{ env.AWS_REGION }}
          audience: sts.amazonaws.com

      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2

      - name: Build Container Image (Local Tarball for Trivy)
        uses: docker/build-push-action@v5
        with:
          context: .
          load: true # Load into local docker daemon for scanning
          tags: ${{ steps.login-ecr.outputs.registry }}/${{ env.ECR_REPOSITORY }}:${{ steps.set_tag.outputs.tag }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

      - name: Run Trivy Vulnerability Scanner
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ steps.login-ecr.outputs.registry }}/${{ env.ECR_REPOSITORY }}:${{ steps.set_tag.outputs.tag }}
          format: 'table'
          exit-code: '1' # Fails the pipeline if CRITICAL vulnerabilities exist
          ignore-unfixed: true
          vuln-type: 'os,library'
          severity: 'CRITICAL,HIGH'

      - name: Push Container Image to Amazon ECR
        run: |
          docker push ${{ steps.login-ecr.outputs.registry }}/${{ env.ECR_REPOSITORY }}:${{ steps.set_tag.outputs.tag }}

  # ----------------------------------------------------------------------------
  # JOB 4: Deployment & Verification Smoke Test
  # ----------------------------------------------------------------------------
  deploy:
    name: Deploy to Production & Smoke Test
    runs-on: ubuntu-latest
    needs: [build-and-scan]
    steps:
      - name: Checkout Ops/GitOps Repository
        uses: actions/checkout@v4
        with:
          repository: enterprise-org/gitops-deployments
          token: ${{ secrets.GITOPS_DEPLOY_PAT }}

      - name: Update Target Image Digest in GitOps Repository
        run: |
          NEW_TAG="${{ needs.build-and-scan.outputs.image_tag }}"
          sed -i "s/tag: .*/tag: \"${NEW_TAG}\"/g" environments/production/values.yaml
          git config user.name "github-actions-bot"
          git config user.email "bot@enterprise.com"
          git commit -am "chore(prod): promote image tag to ${NEW_TAG} [skip ci]"
          git push origin main

      - name: Wait for GitOps Sync & Ingress Stabilization
        run: sleep 30

      - name: Execute Live Production Smoke Test
        run: |
          echo "Executing HTTP Healthcheck..."
          STATUS=$(curl -s -o /dev/null -w "%{http_code}" https://api.enterprise.com/health/live)
          if [ "$STATUS" -ne 200 ]; then
            echo "Smoke test failed with HTTP $STATUS! Initiating incident trigger..."
            exit 1
          fi
          echo "Deployment successful: Ingress returned HTTP 200."
```

---

### 1.4 Production Outages & Debugging

#### Outage Case Study: The Expired AWS Secret Key Blackout
- **Context**: On a Friday evening, an emergency hotfix failed to deploy. The CI pipeline threw:
  `AWS Error: The security token included in the request is expired or invalid`.
- **Root Cause**: The repository relied on a static IAM user's `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` stored in GitHub Secrets. A corporate 90-day key rotation policy automatically deleted the IAM key, halting all production releases across 20 services.
- **Resolution**: Eliminated static secrets permanently by migrating the entire pipeline to **AWS IAM OIDC Federation** (`aws-actions/configure-aws-credentials` with `role-to-assume`). Now authentication uses short-lived, self-rotating 15-minute STS tokens that never expire or require manual rotation.

---

### 1.5 Trade-offs & Decision Matrix

| CI Architecture | Pipeline Speed | Infrastructure Maintenance | Security Posture | Production Verdict |
|:---|:---|:---|:---|:---|
| **GitHub Hosted Runners** | Fast spin-up; autoscaled by GitHub | Zero maintenance | Moderate (public runner pool) | Standard for SaaS microservices. |
| **Self-Hosted Runners (K8s ARC)** | Ultra-fast (persistent local BuildKit cache) | High (patching, scaling, node management) | Exceptional (runs within private VPC) | **Enterprise standard** for large monolithic repos and strict data residency. |
| **Monolithic Single Job** | High latency (sequential steps) | Trivial YAML | Poor (no fail-fast feedback) | Anti-pattern; avoid. |
| **DAG-Based Parallel Jobs** | Minimum total pipeline duration | Moderate | High (isolated job permissions via GITHUB_TOKEN) | **Production standard**. |

---

### 1.6 Senior Interview Q&A

#### Q: How does OIDC-based authentication work between GitHub Actions and AWS IAM, and why does it eliminate the risk of credential leakage?
**Answer**:
OIDC eliminates the need to store long-lived cloud credentials in repository secrets:
1. When a job runs, GitHub's internal token minting service issues an **OIDC JSON Web Token (JWT)** signed by GitHub's private key.
2. The JWT contains claims: `iss` (`https://token.actions.githubusercontent.com`), `aud` (`sts.amazonaws.com`), and `sub` (`repo:org/repo:ref:refs/heads/main`).
3. The runner sends this JWT to AWS STS via the `AssumeRoleWithWebIdentity` API call.
4. AWS STS downloads GitHub's public JSON Web Key Set (JWKS), verifies the signature, and evaluates the IAM Role's **Trust Policy Condition**:
   ```json
   "Condition": {
     "StringEquals": {
       "token.actions.githubusercontent.com:sub": "repo:my-org/my-api:ref:refs/heads/main"
     }
   }
   ```
5. If valid, STS generates temporary, ephemeral credentials (access key, secret key, and session token) valid for 15 to 60 minutes.
No static keys exist anywhere to be leaked, compromised, or rotated.

---

# 2. Advanced Deployment Strategies: Rolling, Blue-Green & Canary

### 2.1 Definition & Core Concept
Modern deployments isolate users from failures during application upgrades:
- **Rolling Update**: Replaces instances sequentially. Zero infrastructure cost overhead, but runs mixed software versions simultaneously during rollout.
- **Blue-Green Deployment**: Maintains two complete, identical production environments ("Blue" = active live version; "Green" = new candidate version). Traffic cutover occurs via an instant router/load balancer swap.
- **Canary Deployment**: Routes a small percentage (e.g., 2% -> 10% -> 50% -> 100%) of real user traffic to the candidate version, monitoring error rates, CPU, and latency before promoting or rolling back.

```
                    CANARY TRAFFIC SPLIT (Layer 7 ALB / Envoy)
                                       │
                    ┌──────────────────┴──────────────────┐
                    │                                     │
                    ▼ (95% Traffic)                       ▼ (5% Traffic)
      ┌───────────────────────────┐         ┌───────────────────────────┐
      │  STABLE FLEET (v1.0)      │         │  CANARY FLEET (v1.1)      │
      │  - 19 Pods / Tasks        │         │  - 1 Pod / Task           │
      │  - Error Rate: 0.01%      │         │  - Error Rate: 4.8% (SPIKE)
      └───────────────────────────┘         └─────────────┬─────────────┘
                                                          │
                                                (Automated Prometheus
                                                 Error Budget Breach!)
                                                          │
                                                          ▼
                                            [INSTANT CANARY TEARDOWN]
                                            (Zero impact on 95% of users)
```

---

### 2.2 Internal Mechanics: Traffic Weighting & Routing Plane Manipulations

#### 1. Blue-Green Router Cutover
In Blue-Green, both environments connect to the production database:
1. `Blue` fleet is receiving 100% of traffic via the ingress router (e.g., AWS ALB Target Group or Kubernetes Service selector `version: blue`).
2. Deployment pipeline provisions the `Green` fleet running the new version.
3. Automated integration tests execute directly against the `Green` fleet's internal endpoint.
4. Once verified, the router switches the target group pointer:
   `ALB Listener -> TargetGroup-Green (Weight: 100)`.
5. If errors spike within the next 5 minutes, the router pointer is reverted to `TargetGroup-Blue` in $<1\text{ second}$.

#### 2. Canary Automated Error Budget Rollback
Canary deployments analyze **SLIs (Service Level Indicators)**. If the Canary error budget burns faster than the allowed threshold, traffic is immediately shunted back to 0%.
Common evaluation query (PromQL):
$$\text{ErrorRate} = \frac{\sum(\text{rate}(\text{http\_requests\_total}\{\text{status}=\sim"5..", \text{version}="canary"\}[2\text{m}]))}{\sum(\text{rate}(\text{http\_requests\_total}\{\text{version}="canary"\}[2\text{m}]))}$$
If $\text{ErrorRate} > 0.01$ (1%), the automated canary orchestrator (Argo Rollouts / Flagger) aborts the deployment.

---

### 2.3 Production Implementation Configurations

#### 1. Argo Rollouts Canary Specification
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: api-canary-rollout
  namespace: production
spec:
  replicas: 10
  strategy:
    canary:
      canaryService: api-canary-svc
      stableService: api-stable-svc
      trafficRouting:
        alb:
          ingress: api-ingress
          servicePort: 80
      steps:
        # Step 1: Shift 5% traffic to canary
        - setWeight: 5
        - pause: { duration: 10m }
        # Step 2: Shift 20% traffic to canary
        - setWeight: 20
        - pause: { duration: 30m }
        # Step 3: Shift 50% traffic
        - setWeight: 50
        - pause: { duration: 15m }
      analysis:
        templates:
          - templateName: error-rate-analysis
        args:
          - name: service-name
            value: api-canary-svc
---
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: error-rate-analysis
  namespace: production
spec:
  metrics:
    - name: success-rate
      interval: 1m
      successCondition: result[0] >= 0.995 # Must maintain 99.5% success rate
      failureLimit: 3
      provider:
        prometheus:
          address: http://prometheus-k8s.monitoring:9090
          query: |
            sum(rate(http_requests_total{status!~"5.*", service="api-canary-svc"}[1m]))
            /
            sum(rate(http_requests_total{service="api-canary-svc"}[1m]))
```

---

### 2.4 Production Outages & Debugging

#### Outage Case Study: The Blue-Green Destructive Database Migration
- **Context**: A team executed a Blue-Green deployment. The new "Green" version contained a database migration that dropped the column `users.phone_number` and created `users.mobile`.
- **Failure**: The migration ran while "Blue" was still live receiving 100% of customer traffic. The instant Green ran `db:migrate`, the existing "Blue" application crashed on every checkout attempt with `PG::UndefinedColumn: ERROR: column "phone_number" does not exist`.
- **Resolution**: Enforced the **Expand-and-Contract (Parallel Run) Database Pattern**:
  1. *Phase 1 (Expand)*: Add `mobile` column as nullable; duplicate writes to both `phone_number` and `mobile`. Deploy code.
  2. *Phase 2 (Backfill)*: Migrate existing legacy data asynchronously.
  3. *Phase 3 (Switch)*: Read exclusively from `mobile`.
  4. *Phase 4 (Contract)*: Drop legacy `phone_number` column only *after* all old application versions are decommissioned.

---

### 2.5 Trade-offs & Decision Matrix

| Strategy | Extra Infra Cost | Blast Radius | Rollback Speed | Database Schema Compatibility |
|:---|:---|:---|:---|:---|
| **Rolling Update** | 0% - 25% | Moderate (mixed version state) | Slow (rolling undo takes minutes) | Requires backward & forward compatibility |
| **Blue-Green** | **+100%** during cutover | Low | **Instant** (<1 second router flip) | Requires strict non-breaking migrations |
| **Canary** | Minimal (1 extra instance) | **Near-Zero** (only 1-5% of users affected) | Fast (automated analysis abort) | Requires backward & forward compatibility |

---

### 2.6 Senior Interview Q&A

#### Q: How do you handle sticky sessions (session affinity) during a Canary deployment?
**Answer**:
Standard weighted round-robin Canary routing breaks if an application depends on in-memory user sessions, as a user could ping the Canary (v1.1) on request 1, and the Stable (v1.0) on request 2.
Production solutions include:
1. **Stateless Session Tokens**: Store user sessions in signed JWTs or shared distributed Redis clusters, completely decoupling session data from individual backend hosts.
2. **Hash-Based Cookie / Header Routing**: Instead of random probabilistic routing, configure the Ingress/ALB or Envoy router to hash a specific HTTP header (e.g., `X-User-ID` or session cookie):
   $$\text{Hash}(\text{UserID}) \pmod{100} < 5 \implies \text{Route to Canary}$$
   This guarantees that a given user is consistently pinned to either the Canary or the Stable fleet for the entire duration of the evaluation window.

---

# 3. Feature Flags: Decoupling Deployment from Release

### 3.1 Definition & Core Concept
- **Deployment**: Moving software artifacts to production servers and spinning up processes. (An operational event).
- **Release**: Exposing new features or business logic to end users. (A business decision).
**Feature Flags (Feature Toggles)** decouple these events by wrapping code paths in conditional checks evaluated dynamically at runtime without deploying new code or restarting containers.

---

### 3.2 Internal Mechanics: Evaluation Engine, Local In-Memory Caching & Polling/SSE

Feature flag architectures (LaunchDarkly, Flipt, Unleash) must satisfy sub-millisecond evaluation latency:

```
+-----------------------------+
|    Feature Flag Server      |
|  (LaunchDarkly / Unleash)   |
+-----------------------------+
               │
               │ Server-Sent Events (SSE) / Streaming gRPC Connection
               ▼
+-------------------------------------------------------------+
|                     Application Worker                      |
|                                                             |
|   ┌─────────────────────────────────────────────────────┐   |
|   │         In-Memory Flag Cache (Concurrent Map)       │   |
|   │  - "new-checkout-flow": { enabled: true, rollout: 15}│   |
|   └─────────────────────────────────────────────────────┘   |
|                              ▲                              |
|                              │ In-memory lookup (< 0.05ms!) |
|   ┌──────────────────────────┴──────────────────────────┐   |
|   │ app.post('/checkout', (req, res) => {               │   |
|   │   if (flags.isEnabled('new-checkout-flow', user)) { │   |
|   │     return runV2Engine(req);                        │   |
|   │   }                                                 │   |
|   │   return runV1Engine(req);                          │   |
|   │ });                                                 │   |
|   └─────────────────────────────────────────────────────┘   |
+-------------------------------------------------------------+
```

1. **Local Evaluation**: The client SDK downloads all flag targeting rules upon initialization and stores them in local memory.
2. **Real-time Sync**: The SDK maintains a persistent HTTP **Server-Sent Events (SSE)** connection to the flag server. When an engineer toggles a flag in the UI, a delta event updates the local memory cache in $<100\text{ms}$.
3. **Consistent Percentage Hashing**: To roll out a feature to 10% of users, the SDK computes:
   $$\text{MurmurHash3}(\text{UserKey} + \text{FlagKey}) \pmod{100} < 10$$
   This ensures deterministic evaluation across multiple backend instances without central network round-trips.

---

### 3.3 Production Code Implementations (TypeScript & Ruby)

#### 1. TypeScript Production Wrapper with Circuit Breaker Fallback
```typescript
import { Unleash, initialize } from 'unleash-client';

class FeatureFlagService {
  private static instance: FeatureFlagService;
  private client: Unleash;

  private constructor() {
    this.client = initialize({
      url: process.env.UNLEASH_API_URL || 'http://unleash:4242/api/',
      appName: 'production-core-api',
      instanceId: process.env.HOSTNAME || 'worker-1',
      customHeaders: {
        Authorization: process.env.UNLEASH_API_TOKEN || '',
      },
      refreshInterval: 15000, // Sync every 15s fallback
    });

    this.client.on('error', (err) => {
      console.error('[FeatureFlag] Unleash SDK error. Falling back to default values:', err);
    });
  }

  public static getInstance(): FeatureFlagService {
    if (!FeatureFlagService.instance) {
      FeatureFlagService.instance = new FeatureFlagService();
    }
    return FeatureFlagService.instance;
  }

  public isEnabled(flagName: string, userId: string, fallback: boolean = false): boolean {
    try {
      // Evaluates locally in RAM with zero network overhead
      return this.client.isEnabled(flagName, { userId }, fallback);
    } catch (err) {
      // Fail closed / use safe fallback in case of internal evaluation panic
      return fallback;
    }
  }
}

export const flags = FeatureFlagService.getInstance();
```

#### 2. Ruby on Rails Implementation
```ruby
# app/services/feature_flag.rb
class FeatureFlag
  def self.enabled?(flag_key, user_id, default: false)
    # Check Redis cache first, or evaluate hash
    context = { user_id: user_id.to_s }
    
    # Fast MurmurHash rollout simulation
    bucket = Digest::MD5.hexdigest("#{flag_key}:#{user_id}")[0..7].to_i(16) % 100
    threshold = ENV.fetch("FLAG_#{flag_key.upcase}_PERCENT", "0").to_i

    bucket < threshold
  rescue StandardError => e
    Rails.logger.error("FeatureFlag evaluation error for #{flag_key}: #{e.message}")
    default
  end
end
```

---

### 3.4 Production Outages & Debugging

#### Outage Case Study: The Synchronous Flag Server Cascading Collapse
- **Context**: A team used an internal feature flag service that made an outbound HTTP REST call (`fetch('http://flags-svc/eval')`) on every incoming web request.
- **Failure**: The feature flag service suffered an AWS hardware degradation, increasing response times from 2ms to 2,500ms. Because the application made synchronous HTTP calls on every request, all Puma worker threads and Node event loop timers backed up waiting for flag responses. The entire primary user-facing application became completely unresponsive.
- **Remediation**: Replaced all synchronous HTTP evaluations with **In-Memory SDK Evaluation** (LaunchDarkly/Unleash architecture) where rules are cached locally in heap memory, guaranteeing zero network latency on request evaluation.

---

### 3.5 Trade-offs & Decision Matrix

| Approach | Evaluation Latency | Blast Radius Control | Technical Debt Cost | Production Recommendation |
|:---|:---|:---|:---|:---|
| **Hardcoded Conditionals (`ENV`)** | Microseconds | Static (Requires redeploy to toggle) | Low | Configuration only (e.g., ports, limits). |
| **Database-Backed Flags** | 5-15ms (DB query) | High risk of DB connection exhaustion | Moderate | Avoid for high-traffic path evaluations. |
| **In-Memory Distributed Flags (SDK)** | **<0.05ms** | Instant runtime kill-switch | High (flag rot if not pruned) | **Production standard** for product releases. |

---

### 3.6 Senior Interview Q&A

#### Q: How do you prevent "Feature Flag Rot" and technical debt from accumulating in a mature engineering organization?
**Answer**:
Feature flag rot occurs when temporary release toggles remain permanently in the codebase, creating combinatorial testing matrices and dead code paths.
Production governance entails:
1. **Flag Expiration Metadata (TTL)**: Every flag is registered in Git or UI with an expiration date (e.g., 30 days) and an owning engineer.
2. **Automated CI Alerts**: CI pipelines check flag ages. If a flag has been at 100% rollout for >14 days, CI raises a warning or opens an automated Jira ticket to remove the dead toggle branch.
3. **AST-Based Removal Tools**: Utilizing automated AST parsers (such as Uber's Piranha) that automatically scan source code, delete the conditional check, remove the flag reference, and submit a PR deleting the legacy path.

---

# 4. GitOps Architecture with ArgoCD

### 4.1 Definition & Core Concept
GitOps is an operational paradigm where **Git repositories act as the single source of truth for the desired infrastructure and application state**.
- **The Core Inversion**: Instead of CI pipelines pushing changes to Kubernetes using external credentials (`kubectl apply` via CI runner), an in-cluster GitOps operator (**ArgoCD**) periodically pulls declarative manifests from Git and reconciles the live cluster state.

```
+---------------------------------------------------------------------------------+
|                                 GITOPS REPOSITORY                               |
|                         (Single Source of Truth)                                |
|   environments/production/                                                      |
|   ├── values.yaml (image: api:4a7c1b, replicas: 10)                             |
+---------------------------------------------------------------------------------+
                                         │
                                         │ Git Polling (Every 3m) / Webhook Trigger
                                         ▼
+---------------------------------------------------------------------------------+
|                       KUBERNETES CLUSTER (PRODUCTION)                           |
|                                                                                 |
|   +-------------------------------------------------------------------------+   |
|   |                      ArgoCD Reconciliation Loop                         |   |
|   |                                                                         |   |
|   |  [ Git Desired State ]  <======== COMPARE ========>  [ Cluster Live ]   |   |
|   |  - Image: 4a7c1b                                     - Image: 1d2e3f    |   |
|   +-------------------------------------------------------------------------+   |
|                                         │                                       |
|                                         │ (DETECTS DRIFT!)                      |
|                                         ▼                                       |
|                         [ AUTOMATED SELF-HEALING SYNC ]                         |
|                                         │                                       |
|                                         v                                       |
|   +-------------------------------------------------------------------------+   |
|   |                      Live Production Deployment                         |   |
|   |                      Updated to Image: 4a7c1b                           |   |
|   +-------------------------------------------------------------------------+   |
+---------------------------------------------------------------------------------+
```

---

### 4.2 Internal Mechanics: The Reconciliation Loop, Drift Detection & Self-Healing

1. **Reconciliation Loop**: ArgoCD runs a continuous control loop (default interval: 3 minutes, or triggered immediately via Git push webhooks).
2. **Drift Detection**: It queries the Git repo and the Kubernetes API server, computing a deep JSON-level diff between:
   - **Target State (Git)**: Parameterized Helm or Kustomize templates rendered into plain manifests.
   - **Live State (etcd)**: The active resources running on the cluster.
3. **Self-Healing (`selfHeal: true`)**:
   If an engineer executes `kubectl edit` or `kubectl delete` directly against the production cluster (manual out-of-band mutation), ArgoCD detects the discrepancy within seconds and **automatically overwrites the live cluster state**, restoring the exact specification defined in Git.

---

### 4.3 Production ArgoCD Application & Sync Policy Manifests

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: production-api
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: 'git@github.com:enterprise-org/gitops-manifests.git'
    targetRevision: main
    path: environments/production
    helm:
      valueFiles:
        - values.yaml
  destination:
    server: 'https://kubernetes.default.svc'
    namespace: production
  syncPolicy:
    automated:
      prune: true     # Deletes live resources if removed from Git
      selfHeal: true  # Overwrites unauthorized manual kubectl edits
      allowEmpty: false
    syncOptions:
      - CreateNamespace=true
      - PruneLast=true
      - ApplyOutOfSyncOnly=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
```

---

### 4.4 Production Outages & Debugging

#### Outage Case Study: The Rogue Developer Manual Fix & Self-Healing Rollback
- **Context**: During an outage, an on-call engineer used `kubectl scale deployment api --replicas=20` to absorb a DDoS attack. 2 minutes later, the cluster scaled back down to 4 replicas automatically, collapsing the service.
- **Root Cause**: ArgoCD had `selfHeal: true` enabled. The Git repository had `replicas: 4`. When the engineer manually altered the replica count in etcd, ArgoCD detected drift and "healed" the cluster by resetting the live state to match Git's desired state.
- **Staff-Level Protocol**: During emergencies with GitOps:
  1. Temporarily pause auto-sync:
     `argocd app set production-api --sync-policy none`
  2. Perform emergency remediation.
  3. Commit the change back to Git to make it the official desired state before re-enabling automated sync.

---

### 4.5 Trade-offs & Decision Matrix

| Deployment Paradigm | CI Access Requirements | Audit Trail | Disaster Recovery | Production Recommendation |
|:---|:---|:---|:---|:---|
| **Push-Based (CI `kubectl apply`)** | CI runner must hold admin cluster credentials | Scattered across CI job logs | Manual reconfiguration | Acceptable for non-prod; high security risk in prod. |
| **Pull-Based GitOps (ArgoCD)** | **Zero cluster credentials outside VPC** | 100% tracked in Git commit history | Instant cluster rebuilding (`git clone` & sync) | **Production standard** for Kubernetes platforms. |

---

### 4.6 Senior Interview Q&A

#### Q: How do you prevent sensitive secrets from being committed in plaintext to a GitOps repository?
**Answer**:
Production GitOps workflows employ one of three secret management patterns:
1. **Sealed Secrets (Bitnami)**: Secrets are encrypted locally via an asymmetric public key using the `kubeseal` CLI. The resulting encrypted `SealedSecret` manifest is committed to Git safely. Only the controller running inside the private cluster holds the private key to decrypt it into a native Kubernetes Secret.
2. **External Secrets Operator (ESO)**: Git holds an `ExternalSecret` manifest referencing a secret name in AWS Secrets Manager or HashiCorp Vault. The in-cluster operator queries the AWS API via IAM Roles for Service Accounts (IRSA) and writes the secret into etcd.
3. **SOPS (Mozilla Secrets OPerationS)**: Encrypts values in YAML files using AWS KMS keys before Git commit; ArgoCD decrypts them during render time via a SOPS plugin.

---

# 5. Docker Image Tagging Governance & The `:latest` Anti-Pattern

### 5.1 Definition & Core Concept
Docker image tags are mutable pointers to content-addressable image manifests:
- **Mutable Tags (`:latest`, `:staging`)**: Pointers that can be reassigned to completely different image layers over time.
- **Immutable Tags (Git SHA `:<git-sha>`, SemVer `:v1.4.2`)**: Tags mapped 1:1 to a specific cryptographic Git commit, ensuring deterministic reproduction.
- **Digest Pinning (`@sha256:...`)**: Bypasses tag names entirely, pointing directly to the cryptographic SHA-256 manifest hash.

---

### 5.2 Internal Mechanics: Content-Addressable Blobs & Immutable SHA Digests

```
Image Repository: 123456789.dkr.ecr.us-east-1.amazonaws.com/api

Tag Pointer (MUTABLE):       :latest ──────────────┐
                                                    │
Tag Pointer (IMMUTABLE):     :c4f1e0a ────────────►│ (Manifest List / OCI Descriptor)
                                                    ▼
Manifest Digest (IMMUTABLE): sha256:7b10fae39b405...
                             ├── Config: sha256:1a2b...
                             └── Layers:
                                 ├── sha256:d4e5... (OS)
                                 └── sha256:8f9a... (Code)
```

#### Why `:latest` is a Catastrophic Production Anti-Pattern:
1. **Broken Rollback Guarantees**:
   If version A is deployed with `:latest` and fails, running `kubectl rollout undo` does nothing, because both the current and the previous deployment specifications specify `image: api:latest`. K8s cannot distinguish between them.
2. **Kubelet `imagePullPolicy` Caching Trap**:
   If a worker node already has an image tagged `:latest` locally:
   - If `imagePullPolicy: IfNotPresent` is set, `kubelet` skips pulling from the registry! Some worker nodes will run the new code, while other worker nodes run code from three weeks ago, causing **cluster split-brain behavior**.
3. **Zero Auditability**:
   Inspecting a running Pod tells you only that it is running `:latest`. It is impossible to identify which line of Git code is running without breaking into the container.

---

### 5.3 Production CI Tagging & Verification Automation

```bash
#!/usr/bin/env bash
set -euo pipefail

# 1. Derive deterministic tag based on short Git SHA and commit timestamp
GIT_SHA=$(git rev-parse --short=8 HEAD)
TIMESTAMP=$(git log -1 --format=%cd --date=format:%Y%m%d%H%M)
IMAGE_TAG="${TIMESTAMP}-${GIT_SHA}"

echo "Building production image with immutable tag: ${IMAGE_TAG}"

# 2. Build and push image
docker build -t "123456789.dkr.ecr.us-east-1.amazonaws.com/api:${IMAGE_TAG}" .
docker push "123456789.dkr.ecr.us-east-1.amazonaws.com/api:${IMAGE_TAG}"

# 3. Extract the immutable SHA-256 Digest from the registry response
IMAGE_DIGEST=$(docker inspect --format='{{index .RepoDigests 0}}' "123456789.dkr.ecr.us-east-1.amazonaws.com/api:${IMAGE_TAG}")

echo "Pinned cryptographic reference: ${IMAGE_DIGEST}"
# Output: 123456789.dkr.ecr.us-east-1.amazonaws.com/api@sha256:e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855
```

---

### 5.4 Production Outages & Debugging

#### Outage Case Study: The Multi-Node Mixed-Version Split-Brain
- **Context**: A microservice was deployed with `image: core-service:latest`. Autoscaling triggered under load, adding 6 new worker nodes.
- **Symptom**: 50% of users received API errors complaining that a new database column was missing, while 50% experienced no issues.
- **Root Cause**:
  - Existing worker nodes had pulled `:latest` 10 hours earlier (Version 1).
  - Newly provisioned nodes pulled `:latest` from the registry (Version 2).
  - Because the tag name was identical, Kubernetes treated all Pods as identical, routing traffic randomly across two incompatible versions of the software.
- **Remediation**:
  Updated CI/CD to reject all builds targeting `:latest`. Every container image is now tagged with `${GIT_COMMIT_SHA}` and enforced in Kubernetes manifests.

---

### 5.5 Trade-offs & Decision Matrix

| Tagging Strategy | Reproducibility | Rollback Safety | Auditability | Production Suitability |
|:---|:---|:---|:---|:---|
| **`:latest`** | Zero | Broken | Impossible | **Strictly Forbidden** in production. |
| **Semantic Versioning (`:v1.2.3`)** | High | High | Good (Maps to release tag) | Production standard for public libraries & SDKs. |
| **Git Commit SHA (`:4a7c1b8d`)** | **Absolute** | **Sub-second** | **100% traceable to commit** | **Production standard** for backend web services. |
| **Pinned Digest (`@sha256:...`)** | Cryptographically absolute | Perfect | Requires registry query to map to Git | Recommended for mission-critical banking & defense infra. |

---

### 5.6 Senior Interview Q&A

#### Q: How do you enforce immutable container image tags in Amazon ECR or Google Artifact Registry?
**Answer**:
By default, container registries allow clients to overwrite existing tags by pushing a new image with the same tag name.
In production, you must enable **Tag Immutability** on the registry repository:
```bash
# Enforce tag immutability on AWS ECR
aws ecr put-image-tag-mutability \
    --repository-name production-core-api \
    --image-tag-mutability IMMUTABLE \
    --region us-east-1
```
Once configured:
- If a CI pipeline or malicious actor attempts to push an image with an existing tag (e.g., re-pushing `v1.2.0`), ECR rejects the push with `ImageTagAlreadyExistsException`.
- This guarantees that once a Git SHA or SemVer tag is pushed and verified by security scanning, its underlying bits can never be tampered with or replaced.
