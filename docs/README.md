# Senior & Staff Backend Developer — Interview Preparation Knowledge Base

> **Candidate Profile**: 5+ Years Backend Experience (Ruby on Rails, Node.js/TypeScript, Next.js, Python, PostgreSQL, Redis, Docker, AWS) · ICPC Regionalist · 3,000+ Algorithmic Problems Solved · FinTech, B2B SaaS & Apple MDM Systems.  
> **Target Role**: Senior / Staff Backend Developer @ Vivasoft Ltd.  
> **Core Focus**: Zero-to-Senior depth, OS/hardware internal mechanics, hands-on production code, real-world outage debugging, and staff-level architectural trade-offs.

---

## 🗺️ Master Documentation Roadmap

```
docs/
├── README.md                                          # Master Index & Navigation Matrix (You Are Here)
│
├── 00-core-cs/                                        # Core Computer Science Foundation
│   ├── 01-memory-architecture.md                      # Stack vs Heap, GC Internals, Cache Lines, Bitmasks & Bloom Filters
│   └── 02-os-fundamentals-concurrency.md              # Processes vs Threads, GVL/libuv, Mutex & Deadlocks, epoll Async I/O
│
├── 01-ruby-on-rails/                                  # Primary Backend Stack (Selling Point)
│   ├── 01-ruby-language-fundamentals.md               # Object Model, Metaprogramming, Enumerable, GVL/Ractors, Memory GC
│   ├── 02-rails-framework-deep-dive.md                # ActiveRecord Internals, 1B Row Migration, Sidekiq, Hotwire, Cascading Timeouts
│   └── 03-rails-error-patterns-debugging.md           # Production Outages, Deadlocks, Connection Timeouts, Memory Bloat, rbspy
│
├── 02-nodejs-express-ts/                              # Primary Backend Stack (JD-Critical)
│   ├── 01-nodejs-runtime-architecture.md              # V8 JIT & Hidden Classes, Event Loop Phases, libuv Pool, Stream Backpressure
│   ├── 02-express-deep-dive.md                        # Middleware Onion, Async Error Boundaries, Layered Architecture, Prisma/Knex
│   ├── 03-typescript-for-backend.md                   # Advanced Types, Discriminated Unions, Declaration Merging, tsconfig
│   └── 04-nodejs-debugging-error-patterns.md          # Event Loop Blocks, ReDoS, Heap Snapshots, Clinic.js, Graceful Shutdown
│
├── 03-react-nextjs/                                   # Primary Frontend Stack (Selling Point)
│   ├── 01-react-fundamentals-hooks.md                 # Fiber Reconciler, Hooks Deep Dive, Zustand/TanStack Query, Virtualization
│   └── 02-nextjs-app-router-rsc.md                    # RSC Wire Protocol, Streaming SSR & Suspense, Caching Matrix, Core Web Vitals
│
├── 04-python-laravel-flutter/                         # Working Knowledge Stack (JD-Critical Polyglot)
│   ├── 01-python-backend.md                           # CPython Internals, FastAPI/Pydantic, Django ORM, GIL & Asyncio
│   ├── 02-laravel-php.md                              # Eloquent ORM Hydration, Service Container IoC, Queues, Sanctum/Passport
│   └── 03-flutter-mobile.md                           # Dart Sound Null Safety, Three Trees, Layout Constraints, GoRouter, Riverpod/Bloc
│
├── 05-databases/                                      # Databases — SQL & NoSQL (JD-Critical)
│   ├── 01-postgresql-deep-dive.md                     # Query Lifecycle, B-tree/BRIN/GIN, HOT Updates, EXPLAIN BUFFERS, RLS Multi-Tenancy
│   ├── 02-mysql-deep-dive.md                          # InnoDB Clustered Index, utf8mb4 4-Byte Flaw, Locking & Isolation
│   ├── 03-mongodb-deep-dive.md                        # BSON Architecture, Embed vs Reference, Aggregation Pipelines, Sharding
│   └── 04-redis-deep-dive.md                          # Data Types, Sliding Window Log via Lua, Cache Stampede, Redis Cluster & Redlock
│
├── 06-competitive-programming-dsa/                    # Algorithmic Mastery (3,000+ Solved, ICPC)
│   ├── 01-data-structures-internals.md                # HashDoS, Monotonic Queues in Prod, Radix Trie Routers, Consistent Hashing
│   ├── 02-algorithmic-paradigms.md                    # Search on Answer, 6 DP Archetypes, KMP/Aho-Corasick, Number Theory
│   └── 03-complexity-and-interview-strategy.md        # Amortized Analysis, Constraint Matrix, 45-Minute Interview Blueprint
│
├── 07-system-design/                                  # Alex Xu Level System Design (JD-Critical)
│   ├── 01-framework-and-staff-mindset.md              # 4-Step Framework, Challenging PRD, Latency Numbers, Build vs Buy
│   ├── 02-core-distributed-components.md              # Rate Limiters, Snowflake IDs, URL Shorteners, Video Streaming, Chat, Ledgers
│   ├── 03-distributed-systems-patterns.md             # CAP/PACELC, Replication & Raft, CQRS, Saga Orchestration, Outbox CDC
│   └── 04-microservices-and-case-studies.md           # Strangler Fig, Kafka vs RabbitMQ, Real-World: Uber-Lite, Flash Sale, Multi-Tenant SaaS
│
├── 08-docker-devops-aws/                              # Infrastructure, DevOps & Cloud (JD-Critical)
│   ├── 01-docker-internals-and-recipes.md             # Namespaces, Cgroups v2, OverlayFS, Multi-Stage Rails & Node Dockerfiles
│   ├── 02-kubernetes-fundamentals.md                  # Pod Architecture, Probes & Cascading Restarts, HPA Scaling, Zero-Downtime Rollouts
│   ├── 03-cicd-and-deployment-strategies.md           # GitHub Actions, Blue-Green vs Canary Deployments, GitOps ArgoCD
│   ├── 04-aws-cloud-architecture.md                   # VPC Subnet Isolation, ECS Fargate vs EC2, ALB/NLB, S3 Pre-signed URLs, RDS Multi-AZ
│   └── 05-production-troubleshooting-and-git.md       # strace Syscalls, tcpdump & SNAT Exhaustion, Live Profiling, git bisect/reflog
│
├── 09-networking-web-internals/                       # Protocols & Web Architecture
│   ├── 01-url-lifecycle-and-nginx.md                  # End-to-End DNS, TCP/TFO, TLS 1.3, NGINX Slow Client Buffering, Browser CRP
│   ├── 02-http-protocols-and-headers.md               # HTTP/1.1 vs HTTP/2 vs HTTP/3 QUIC, TCP vs UDP, Status Codes & Caching Headers
│   ├── 03-https-tls-and-security.md                   # Symmetric vs Asymmetric, TLS 1.3 1-RTT & 0-RTT Replays, mTLS Zero-Trust, OCSP
│   └── 04-realtime-and-api-design.md                  # WebSockets vs SSE vs gRPC, Cursor vs Offset Pagination, Versioning, GraphQL
│
├── 10-ai-workflow-integrations/                       # Modern Agentic Workflows (Selling Point)
│   ├── 01-ai-assisted-engineering.md                  # Antigravity/Claude/Cursor, MCP Standard, Spec-First Loop, Property-Based Testing
│   ├── 02-llm-api-and-rag-systems.md                  # OpenAI/Anthropic/Gemini APIs, Streaming SSE, pgvector HNSW, Hybrid Search, ReAct
│   └── 03-llm-core-concepts.md                        # Scaled Dot-Product Attention, KV Cache Memory, 4-bit Quantization, LoRA/QLoRA
│
├── 11-security-reliability-observability/             # Enterprise Production Standards
│   ├── 01-application-security.md                     # OWASP Top 10, SSRF Defense, JWT Rotation & Breach Detection, AWS KMS Envelopes
│   ├── 02-reliability-patterns.md                     # Circuit Breakers (Opossum), Jittered Retries, Bulkhead Isolation, Little's Law
│   └── 03-observability-and-incidents.md              # OpenTelemetry Tracing, RED & USE Frameworks, SLO Error Budgets, SEV-1 IC Protocol
│
├── 12-behavioral-past-projects/                       # Leadership, Mentorship & Real Projects
│   ├── 01-project-case-studies.md                     # Technonext 10x Scaling ($0 Infra), Multi-Tenant Auth, Gauntlet FinTech, Apple MDM
│   └── 02-staff-leadership-and-communication.md       # STAR Execution, Force Multiplier Mindset, Socratic Mentoring, Client Negotiations
│
└── 13-resume-defense/                                 # Resume Line-by-Line Technical Defense
    └── 01-resume-claims-and-technical-qa.md           # 5+ Years Defense, 3000+ CP Problems, FinTech Ledgers, Kubernetes/GraphQL Q&A
```

---

## 🎯 High-Impact Interview Cheatsheet (Quick Links)

1. **ActiveRecord Allocation Bloat & Pluck**: [`01-ruby-on-rails/02-rails-framework-deep-dive.md`](file:///home/abrakadebra/projects/interview-prep/docs/01-ruby-on-rails/02-rails-framework-deep-dive.md)
2. **The 1-Billion-Row Migration Pattern**: [`01-ruby-on-rails/02-rails-framework-deep-dive.md`](file:///home/abrakadebra/projects/interview-prep/docs/01-ruby-on-rails/02-rails-framework-deep-dive.md)
3. **Cascading Timeout Stack Rule**: [`01-ruby-on-rails/02-rails-framework-deep-dive.md`](file:///home/abrakadebra/projects/interview-prep/docs/01-ruby-on-rails/02-rails-framework-deep-dive.md)
4. **Node.js libuv Thread Pool Starvation**: [`02-nodejs-express-ts/01-nodejs-runtime-architecture.md`](file:///home/abrakadebra/projects/interview-prep/docs/02-nodejs-express-ts/01-nodejs-runtime-architecture.md)
5. **PostgreSQL HOT Updates & FILLFACTOR**: [`05-databases/01-postgresql-deep-dive.md`](file:///home/abrakadebra/projects/interview-prep/docs/05-databases/01-postgresql-deep-dive.md)
6. **Redis Sliding Window Log via Lua**: [`05-databases/04-redis-deep-dive.md`](file:///home/abrakadebra/projects/interview-prep/docs/05-databases/04-redis-deep-dive.md)
7. **Monotonic Queues in Production**: [`06-competitive-programming-dsa/01-data-structures-internals.md`](file:///home/abrakadebra/projects/interview-prep/docs/06-competitive-programming-dsa/01-data-structures-internals.md)
8. **Consistent Hashing Virtual Nodes Ring**: [`06-competitive-programming-dsa/01-data-structures-internals.md`](file:///home/abrakadebra/projects/interview-prep/docs/06-competitive-programming-dsa/01-data-structures-internals.md)
9. **NGINX Slow Client Buffering Mechanics**: [`09-networking-web-internals/01-url-lifecycle-and-nginx.md`](file:///home/abrakadebra/projects/interview-prep/docs/09-networking-web-internals/01-url-lifecycle-and-nginx.md)
10. **Keyset / Cursor vs Offset Pagination**: [`09-networking-web-internals/04-realtime-and-api-design.md`](file:///home/abrakadebra/projects/interview-prep/docs/09-networking-web-internals/04-realtime-and-api-design.md)
11. **AWS SNAT Port Exhaustion Diagnosis**: [`08-docker-devops-aws/05-production-troubleshooting-and-git.md`](file:///home/abrakadebra/projects/interview-prep/docs/08-docker-devops-aws/05-production-troubleshooting-and-git.md)
12. **Technonext 10x Scaling Case Study**: [`12-behavioral-past-projects/01-project-case-studies.md`](file:///home/abrakadebra/projects/interview-prep/docs/12-behavioral-past-projects/01-project-case-studies.md)
