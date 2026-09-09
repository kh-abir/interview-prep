# Interview Preparation — Deep Knowledge Checklist

> **Target Role**: Backend Developer @ Vivasoft Ltd (80k-140k BDT)
> **Your Profile**: 5+ years Rails/Node.js/Next.js · ICPC Regionalist · 3000+ problems · FinTech + SaaS + MDM
>
> **JD MUST-HAVEs**: ① 2000+ CP problems ✅ (you: 3000+) ② Alex Xu-level System Design
> **JD Requirements**: Node.js, Python, Java/C# · RESTful APIs · Cache · Microservices · AWS/Docker · DB (SQL + NoSQL) · Frontend (React) · Client communication · Mentoring

---

> **How to use**: Go through each checkbox topic. Study from zero → senior depth. Mark `[x]` when confident. Topics marked *(JD-CRITICAL)* directly map to job requirements. Topics marked *(SELLING POINT)* are your competitive advantages to highlight.

---

## 0. Core CS Foundation *(from your Senior SWE Handbook — Chapters 1 & 5)*

### 0.1 Memory Architecture

- [ ] **Stack vs. Heap**
  - Stack: LIFO, fixed per thread (~8MB), stores primitives, pointers, stack frames; `StackOverflowError` on deep recursion
  - Heap: dynamic pool, objects/arrays/closures allocated here; allocation overhead (free block scanning), fragmentation
  - Ruby/JS: pass-by-value of reference (pointer to heap object is copied, not the object)
  - Avoid `str += "a"` in hot loops — creates new heap object per iteration; use `StringIO` / array join

- [ ] **Garbage collection**
  - Mark-and-Sweep: traverse roots → mark reachable → sweep unmarked — stop-the-world pauses
  - Generational GC: Young/Nursery (frequent, fast) vs. Old gen (rare, expensive) — V8, Ruby MRI
  - Major GC pause causing ALB timeouts (1-3s): memory leak promoting objects to old space
  - GC tuning: `RUBY_GC_HEAP_INIT_SLOTS`, V8 `--max-old-space-size`

- [ ] **Cache locality & hardware realities**
  - CPU cache: L1 (1ns) → L2 (4ns) → L3 (10ns) → RAM (100ns) → SSD (100μs) → HDD (10ms)
  - Cache lines: 64 bytes, contiguous memory fills lines — arrays beat linked lists for small N
  - Branch prediction: CPU speculative execution, misprediction penalty ~15 cycles; sort data before filtering
  - Why `O(n)` array scan beats `O(log n)` tree for small n: constant factors and cache effects dominate

- [ ] **Bitwise operations in production**
  - Permission bitmasks: `CAN_VIEW = 1`, `CAN_EDIT = 2`, `CAN_DELETE = 4`, `IS_ADMIN = 8`
  - Set with OR (`|`), check with AND (`&`), toggle with XOR (`^`)
  - Bloom filters: probabilistic set membership, bit array + k hash functions, false positives but no false negatives

### 0.2 OS Fundamentals & Concurrency

- [ ] **Processes vs. Threads**
  - Process: separate memory space, crash isolation, heavy (~300MB per Rails worker), IPC needed
  - Thread: shared heap within process, light (~1-2MB stack), shared memory mutation risk
  - Green threads / fibers / coroutines: userspace scheduling, no OS context switch

- [ ] **Concurrency vs. Parallelism**
  - Concurrency: interleaving tasks on single core (Node.js event loop, async/await)
  - Parallelism: simultaneous execution on multiple cores (Puma workers, Python multiprocessing)
  - Node.js: concurrent (single thread + libuv) but not parallel on main thread

- [ ] **Synchronization primitives**
  - Mutex: mutual exclusion lock, one thread enters critical section
  - Deadlock: circular lock dependency; prevent with strict global lock ordering (alphabetical/ID-based)
  - Race condition: read-modify-write without lock → corrupt state (bank double-withdrawal)

- [ ] **Async I/O multiplexing**
  - `epoll` (Linux) / `kqueue` (macOS): kernel event notification on socket readiness
  - How Node.js handles 10,000+ connections on one thread: non-blocking I/O delegation to OS kernel
  - vs. Apache thread-per-connection model: thread overhead, context switching cost

---

## 1. Ruby on Rails — Primary Backend *(SELLING POINT)*

### 1.1 Ruby Language Fundamentals (Zero → Senior)

- [ ] **Core syntax & data types**
  - Variables: local, instance (`@`), class (`@@`), global (`$`), constants
  - Data types: Integer, Float, String, Symbol, Array, Hash, Range, NilClass, TrueClass, FalseClass
  - Symbols vs. Strings: immutability, memory, symbol table, `freeze`
  - String interpolation, heredocs, frozen string literals (`# frozen_string_literal: true`)
  - Type coercion: `to_s`, `to_i`, `to_f`, `to_sym`, `to_a`, `to_h`

- [ ] **Blocks, Procs, and Lambdas**
  - Block syntax: `do...end` vs. `{...}`, `yield`, `block_given?`
  - Proc: `Proc.new`, closures, `proc.call`, arity flexibility
  - Lambda: `lambda {}`, `->() {}`, strict arity, `return` only exits lambda (not enclosing method)
  - Proc vs. Lambda: arity strictness, return behavior
  - `&` operator: convert block ↔ Proc, `method(:name)` to Proc
  - Currying: `proc.curry`

- [ ] **Object model & metaprogramming**
  - Everything is an object: `1.class` → `Integer`, method lookup chain
  - Eigenclass / singleton class: `class << self`, per-object methods
  - Method lookup: object → class → superclass → modules (reverse include order) → BasicObject
  - `include` vs. `extend` vs. `prepend`: where module sits in ancestor chain
  - `method_missing` + `respond_to_missing?`: dynamic dispatch
  - `define_method`: runtime method creation
  - `class_eval` / `module_eval` vs. `instance_eval`: context switching
  - `send` vs. `public_send`: invoking methods dynamically
  - `const_missing`, `inherited`, `included`, `extended` hooks
  - Open classes: monkey-patching, refinements (`using`)

- [ ] **Modules & mixins**
  - Modules as namespaces vs. mixins
  - `Comparable`, `Enumerable`: implementing `<=>` and `each` to get full API
  - Composition over inheritance: multiple module includes

- [ ] **Enumerable & collections mastery**
  - `map`, `select`, `reject`, `reduce`, `flat_map`, `each_with_object`
  - `group_by`, `partition`, `chunk`, `zip`, `tally`, `filter_map`
  - Lazy enumerators: `lazy.select.map.first(10)` — infinite sequences
  - `Enumerator`, `Enumerator::Yielder`, custom enumerators

- [ ] **Error handling**
  - `begin/rescue/ensure/retry/raise`
  - Exception hierarchy: `Exception` → `StandardError` (never rescue `Exception`)
  - Custom exception classes: inherit from `StandardError`
  - `retry` with counter for transient failures
  - `ensure` vs. `rescue`: cleanup guarantee

- [ ] **Concurrency in Ruby**
  - GIL (GVL): Global VM Lock — one thread executes Ruby at a time
  - Threads: `Thread.new`, good for I/O-bound (GIL released during I/O)
  - Fibers: cooperative concurrency, `Fiber.new`, `Fiber.yield`, `resume`
  - Ractors (Ruby 3+): true parallelism, separate GVL per Ractor, message passing
  - Process forking: `fork`, copy-on-write, used by Puma/Unicorn

- [ ] **Memory & garbage collection**
  - Mark-and-sweep GC, generational GC (young/old objects)
  - Object allocation awareness: avoid unnecessary object creation in hot paths
  - `ObjectSpace`, memory profiling with `memory_profiler` gem
  - Frozen strings for reduced allocations

### 1.2 Rails Framework (Zero → Senior)

- [ ] **MVC architecture**
  - Request lifecycle: Router → Controller → Model → View → Response
  - Convention over Configuration: naming conventions (plural tables, singular models)
  - `config/routes.rb`: resources, nested resources, concerns, constraints, namespaces, scopes
  - RESTful routing: 7 default actions, `only`, `except`, member vs. collection routes

- [ ] **ActiveRecord deep dive**
  - ORM pattern: each model = DB table, each instance = row
  - Migrations: `create_table`, `add_column`, `add_index`, `change_column`, reversible migrations
  - Associations: `belongs_to`, `has_many`, `has_one`, `has_many :through`, `has_and_belongs_to_many`
  - Polymorphic associations: `commentable_type` + `commentable_id`
  - STI (Single Table Inheritance): when to use, downsides (sparse columns)
  - Validations: presence, uniqueness (race condition!), format, custom validators, conditional validations
  - Callbacks: `before_save`, `after_create`, `around_destroy` — lifecycle hooks, when to avoid (side effects, testing pain)
  - Scopes: named scopes, default scopes (avoid!), chaining, merging
  - Query interface: `where`, `joins`, `includes`, `eager_load`, `preload`, `left_joins`, `group`, `having`, `order`
  - **Pluck vs. Select**: `select` instantiates heavy ActiveRecord model objects in heap (~1-2KB per row) vs. `pluck` returns raw arrays directly from DB driver with near-zero allocation
  - **Memory bloat prevention**: avoid `.all` or `.each` on large datasets; use `find_each(batch_size: 1000)` so GC can sweep batch memory during iteration
  - **N+1 queries**: detection (Bullet gem), fix with `includes`/`eager_load`/`preload`
  - **Raw SQL**: `find_by_sql`, `connection.execute`, Arel for complex queries
  - Transactions: `ActiveRecord::Base.transaction`, nested transactions (savepoints), rollback on exception
  - Locking: optimistic (`lock_version` column) vs. pessimistic (`.lock`, `FOR UPDATE`)
  - Database-level constraints vs. model-level validations: why both matter
  - **1 Billion row migration pattern**: Never `ALTER TABLE ADD COLUMN` with default or index in single step; use 5-step dual-write/backfill (1. nullable column -> 2. dual-write in app -> 3. async batch backfill with 100ms sleeps via Sidekiq -> 4. read path transition -> 5. drop old column/add constraints)

- [ ] **ActiveJob & Sidekiq** *(JD-CRITICAL: background processing)*
  - ActiveJob: adapter-agnostic interface, `perform_later`, `perform_now`
  - Sidekiq: Redis-backed, threaded workers, queues, weights, concurrency
  - Job options: `queue`, `retry`, `dead`, `backtrace`
  - Retry strategies: exponential backoff (Sidekiq default), `sidekiq_retry_in`
  - Dead-letter queue: Sidekiq DeadSet, manual retry, monitoring
  - Unique jobs: `sidekiq-unique-jobs` gem, idempotency patterns
  - Scheduled jobs: `perform_in`, `perform_at`, Sidekiq-Cron/Sidekiq-Scheduler
  - Monitoring: Sidekiq Web UI, queue latency, memory usage
  - Rate limiting workers: `Sidekiq::Limiter`
  - Batch jobs: `Sidekiq::Batch` for coordinated workflows with callbacks

- [ ] **ActionCable (WebSockets)**
  - Channels, subscriptions, broadcasting
  - Connection authentication: `identified_by`
  - Scaling: Redis adapter for multi-server pub/sub
  - vs. Hotwire/Turbo Streams: when to use each

- [ ] **Hotwire (Turbo + Stimulus)**
  - Turbo Drive: SPA-like navigation without JavaScript (body replacement over AJAX)
  - Turbo Frames: partial page updates, lazy loading frames, independent DOM fragment replacement
  - Turbo Streams: real-time updates over WebSocket/HTTP, CRUD actions (append, prepend, replace, remove)
  - Stimulus: minimal JavaScript controllers, data attributes, targets, actions
  - When to use Hotwire vs. React/Next.js frontend

- [ ] **API mode & serialization**
  - `rails new --api`: skip views, cookies, sessions
  - Serializers: `ActiveModelSerializers`, `Blueprinter`, `Alba`, `jsonapi-serializer`
  - JSON:API spec vs. custom JSON structure
  - Pagination: `kaminari`, `pagy`, `will_paginate` — cursor vs. offset
  - API versioning: URL path (`/api/v1/`), header-based, namespace modules
  - Rate limiting: `rack-throttle`, `rack-attack`

- [ ] **Authentication & authorization in Rails**
  - Devise: modules (database_authenticatable, registerable, recoverable, rememberable, trackable, validatable, lockable, confirmable, omniauthable)
  - JWT auth: `jwt` gem, token generation, refresh token rotation
  - Pundit: policy objects, `authorize`, scoping
  - CanCanCan: ability definitions, `can`, `cannot`, `accessible_by`
  - API token auth: `has_secure_token`, API key management

- [ ] **Testing in Rails** *(JD-CRITICAL: code quality)*
  - RSpec: `describe`, `context`, `it`, `let`, `let!`, `before`, `subject`, `shared_examples`
  - Factory patterns: FactoryBot — `build`, `create`, `build_stubbed`, traits, sequences, associations
  - Model specs: validations, associations, scopes, callbacks
  - Request specs: full integration, HTTP verbs, response assertions, JSON parsing
  - Controller specs (legacy) vs. request specs (modern)
  - Mocking/stubbing: `allow`, `expect`, `receive`, `double`, `instance_double`, `class_double`
  - System specs: Capybara, headless Chrome, end-to-end
  - Test coverage: SimpleCov, target >90%
  - VCR / WebMock: recording and stubbing external HTTP calls
  - Database cleaner strategies: transaction, truncation, deletion
  - CI integration: parallel tests (`parallel_tests` gem), RSpec profiling

- [ ] **Performance & caching**
  - Fragment caching: `cache` helper, Russian doll caching (nested `cache`)
  - Low-level caching: `Rails.cache.fetch`, `Rails.cache.write/read`
  - Cache stores: Redis, Memcached, file store, memory store
  - HTTP caching: `expires_in`, `stale?`, ETags, conditional GET
  - Query caching: automatic per-request, `ActiveRecord::Base.cache`
  - Counter caches: `belongs_to :post, counter_cache: true`
  - Bullet gem: detect N+1, unused eager loading
  - `rack-mini-profiler`: per-request timing in development
  - Benchmark: `Benchmark.measure`, `Benchmark.ips`

- [ ] **Rails security**
  - CSRF: authenticity token, `protect_from_forgery`
  - SQL injection: parameterized queries (AR does this), dangers of `where("name = '#{input}'")`
  - XSS: auto-escaping in ERB (`<%= %>`), `raw`, `html_safe` dangers
  - Mass assignment: strong parameters (`params.require().permit()`)
  - Content Security Policy: `config/initializers/content_security_policy.rb`
  - Credential management: `rails credentials:edit`, encrypted credentials, per-environment

- [ ] **Deployment & production architecture**
  - Puma: socket listening, thread worker pools (e.g. 5 workers * 5 threads = 25 concurrent requests)
  - Socket backlog queue: overflow triggers NGINX `502 Bad Gateway`
  - **Stack timeout cascade rule**: Browser (30s) -> ALB (60s) -> NGINX `proxy_read_timeout` (60s) -> Puma (15s) -> DB statement timeout (10s). Timeouts must decrease deeper into the stack so database times out first, not load balancer
  - Unicorn vs. Puma: forked worker model vs. threaded worker model
  - Asset pipeline: Sprockets vs. Propshaft vs. esbuild/Vite
  - Database migrations in production: safe migration patterns, `strong_migrations` gem
  - Zero-downtime deploys: rolling restart, connection draining
  - Rails console in production: `rails c --sandbox`

- [ ] **Rails debugging & profiling**
  - `binding.pry` / `binding.irb` / `debugger` (Ruby 3.1+)
  - `rails console`: testing queries, inspecting objects
  - Log levels: debug, info, warn, error, fatal
  - `ActiveRecord::Base.logger = Logger.new(STDOUT)` for query debugging
  - Profiling CPU: `rbspy` (non-invasive call stack sampling, Flamegraphs)
  - `rails routes`: inspect all routes
  - Stack traces: reading Rails backtraces, filtering framework noise
  - `better_errors` + `binding_of_caller` gems in development

### 1.3 Rails Error Patterns & Production Debugging

- [ ] **Common errors & fixes**
  - `ActiveRecord::RecordNotFound`: `find` vs. `find_by` (nil vs. exception)
  - `ActiveRecord::RecordInvalid`: validation failures in `save!`/`create!`
  - `PG::UniqueViolation`: race condition on unique constraints, handle with `rescue`
  - `ActionController::ParameterMissing`: strong params misconfiguration
  - `NoMethodError: undefined method for nil:NilClass`: safe navigation (`&.`)
  - `ArgumentError`: wrong number of arguments, missing keyword args
  - Memory bloat: large CSV imports → use `find_each` (batched), streaming responses
  - Connection pool exhaustion: `ActiveRecord::ConnectionTimeoutError`, tune `pool` in `database.yml`
  - Deadlocks: concurrent transactions updating same rows in different order

---

## 2. Node.js & Express — Primary Backend *(JD-CRITICAL)*

### 2.1 Node.js Fundamentals (Zero → Senior)

- [ ] **Runtime architecture**
  - V8 engine: JIT compilation, hidden classes, inline caching
  - Single-threaded event loop: why it works for I/O-bound workloads
  - Event loop phases: timers → pending callbacks → idle/prepare → poll (I/O wait) → check (`setImmediate`) → close
  - Microtask queue: `process.nextTick` and Promises run between every phase; `process.nextTick` starvation risk
  - libuv: thread pool (default 4 threads) for file I/O (`fs`), DNS resolution (`dns.lookup`), and crypto (`crypto.pbkdf2`)
  - `UV_THREADPOOL_SIZE`: increase to 8-16 when doing heavy concurrent crypto/file operations to prevent thread starvation
  - CPU-bound loop lockup: long regex, massive JSON parsing (50MB+), or crypto blocks the single thread for all concurrent requests

- [ ] **Module system**
  - CommonJS: `require`, `module.exports`, synchronous, caching
  - ES Modules: `import`/`export`, async, top-level await, `.mjs` or `"type": "module"`
  - Module resolution: file → directory (index.js) → node_modules → parent directories
  - Circular dependencies: how CommonJS and ESM handle differently

- [ ] **Async patterns**
  - Callbacks: error-first convention `(err, data)`, callback hell
  - Promises: `new Promise`, `.then/.catch/.finally`, `Promise.all/allSettled/race/any`
  - async/await: syntactic sugar over Promises, error handling with try/catch
  - Event emitters: `EventEmitter`, `on`, `emit`, `once`, `removeListener`, memory leaks (max listeners warning)
  - Streams: Readable, Writable, Duplex, Transform — backpressure handling, piping
  - `AbortController`/`AbortSignal`: cancelling async operations

- [ ] **Error handling**
  - Sync errors: try/catch
  - Async errors: `.catch()`, `try/catch` with await, `process.on('unhandledRejection')`
  - `process.on('uncaughtException')`: last resort, log and exit
  - Operational vs. programmer errors: handle vs. crash
  - Error classes: custom `AppError extends Error`, error codes, HTTP status mapping
  - Graceful shutdown: handle SIGTERM/SIGINT, drain connections, close DB pool

- [ ] **Memory management & leak hunting**
  - V8 heap: new space (Scavenger / young generation) + old space (Mark-Sweep-Compact / major GC)
  - Common memory leaks: closure scope retaining outer large variables, unbounded global caches (`const cache = {}`), forgotten setIntervals, unremoved event listeners
  - Heap snapshot profiling: `v8.writeHeapSnapshot()`, load into Chrome DevTools, inspect Retainers tree to find what references dead objects
  - `--max-old-space-size`: configure heap limit; monitor memory with `process.memoryUsage().heapUsed`

- [ ] **Worker threads & clustering**
  - `cluster` module: fork worker processes, load balance across CPU cores
  - `worker_threads`: true parallelism for CPU-bound tasks, `SharedArrayBuffer`, `MessageChannel`
  - When to use cluster (multi-process) vs. worker_threads (multi-thread)
  - PM2: process manager, cluster mode, auto-restart, log management

### 2.2 Express.js Deep Dive

- [ ] **Core concepts**
  - `app.get/post/put/patch/delete`: route handlers
  - Route parameters: `:id`, query strings, `req.params`, `req.query`, `req.body`
  - Middleware stack: `app.use()`, execution order, `next()`, error middleware `(err, req, res, next)`
  - Request lifecycle: middleware chain → route handler → response

- [ ] **Middleware patterns**
  - Built-in: `express.json()`, `express.urlencoded()`, `express.static()`
  - Authentication middleware: JWT verification, session checking
  - Error handling middleware: centralized error handler, async error wrapper
  - Logging: `morgan`, custom request logging
  - CORS: `cors` package, configuration
  - Rate limiting: `express-rate-limit`, Redis store for distributed
  - Helmet: security headers (CSP, HSTS, X-Frame-Options)
  - Compression: `compression` middleware, gzip/brotli

- [ ] **Project structure**
  - Routes → Controllers → Services → Repositories/Models
  - Separation of concerns: business logic in services, not controllers
  - Dependency injection patterns in Node.js
  - Configuration: `dotenv`, `config`, environment-based configs

- [ ] **Validation**
  - `joi`: schema-based validation, custom messages
  - `express-validator` / `zod`: request validation
  - Input sanitization: preventing injection attacks

- [ ] **ORM / Query builders**
  - Sequelize: models, migrations, associations, transactions, raw queries
  - Prisma: schema-first, type-safe, migrations, relations
  - Knex.js: query builder, migrations, seeds
  - TypeORM: decorator-based, Active Record + Data Mapper patterns
  - Mongoose (MongoDB): schemas, virtuals, middleware, population

### 2.3 TypeScript for Node.js

- [ ] **Type system**
  - Basic types: `string`, `number`, `boolean`, `null`, `undefined`, `void`, `never`, `unknown`, `any`
  - Interfaces vs. types: when to use each, extending, intersecting
  - Generics: functions, classes, constraints (`extends`), utility types
  - Utility types: `Partial`, `Required`, `Pick`, `Omit`, `Record`, `Readonly`, `ReturnType`
  - Union types, intersection types, discriminated unions
  - Type guards: `typeof`, `instanceof`, `in`, custom type predicates (`is`)
  - Type assertion: `as`, non-null assertion `!`, when safe
  - `enum` vs. `const` assertions vs. union of literals
  - Declaration files: `.d.ts`, `@types/` packages, `declare`

- [ ] **TypeScript with Express**
  - Typing request/response: `Request<Params, ResBody, ReqBody, Query>`
  - Middleware typing, custom request properties via declaration merging
  - Strict mode: `strict: true`, `noImplicitAny`, `strictNullChecks`

### 2.4 Node.js Debugging & Error Patterns

- [ ] **Common errors**
  - `ECONNREFUSED`: database/service not running
  - `EADDRINUSE`: port already in use
  - `ERR_HTTP_HEADERS_SENT`: calling `res.send()` twice
  - `ENOMEM`: out of memory, heap limit
  - `ETIMEDOUT`: connection timeout to external service
  - Unhandled promise rejections: process crash in Node 15+
  - Event loop blocking: CPU-heavy sync operation freezing all requests
  - Memory leaks: event listeners not removed, global cache growing unbounded

---

## 3. React & Next.js — Primary Frontend *(SELLING POINT)*

### 3.1 React Fundamentals (Zero → Senior)

- [ ] **Core concepts**
  - JSX: syntactic sugar for `React.createElement`, rules (single root, `className`, `htmlFor`)
  - Components: function components (modern) vs. class components (legacy, understand lifecycle)
  - Props: read-only, destructuring, `children`, default props, prop types
  - State: `useState`, immutable updates, batching, state as snapshot
  - Rendering: virtual DOM, reconciliation algorithm (diffing), keys for list rendering
  - Component lifecycle (class): mount → update → unmount, `componentDidMount`, `shouldComponentUpdate`

- [ ] **Hooks deep dive**
  - `useState`: lazy initialization, functional updates
  - `useEffect`: dependency array, cleanup function, stale closures pitfall
  - `useContext`: context API, provider pattern, avoiding unnecessary re-renders
  - `useReducer`: complex state logic, dispatch actions, vs. useState
  - `useRef`: mutable container, DOM refs, persist values across renders without re-render
  - `useMemo`: memoize expensive computations, dependency array
  - `useCallback`: memoize functions for child component optimization
  - `useLayoutEffect`: synchronous after DOM update, before paint (measure DOM)
  - Custom hooks: extract reusable logic, `use` prefix convention, composability
  - Rules of hooks: only top level, only in React functions

- [ ] **State management**
  - Component state → lifting state → Context → external libraries
  - Context API: when to use (theme, auth, locale), when NOT (frequent updates)
  - Redux: store, actions, reducers, middleware (thunk, saga), Redux Toolkit (RTK)
  - Zustand: minimal, no boilerplate, selectors for performance
  - React Query / TanStack Query: server state management, caching, background refetch, stale-while-revalidate
  - When to use which: server state (React Query) vs. client state (Zustand/Redux)

- [ ] **Performance optimization**
  - `React.memo`: skip re-render if props unchanged (shallow comparison)
  - `useMemo` / `useCallback`: avoid unnecessary recalculations/re-renders
  - Virtualization: `react-window`, `react-virtualized` for large lists
  - Code splitting: `React.lazy` + `Suspense`, route-based splitting
  - Profiler: React DevTools Profiler, identifying wasted renders
  - Avoiding common pitfalls: inline functions in JSX, object literals as props, missing keys

- [ ] **Patterns**
  - Compound components: `<Select>` + `<Select.Option>`
  - Render props: `<DataFetcher render={(data) => ...}>`
  - HOC (Higher-Order Components): `withAuth`, `withTheme` (mostly replaced by hooks)
  - Controlled vs. uncontrolled components: forms, refs
  - Error boundaries: `componentDidCatch`, `getDerivedStateFromError`, `react-error-boundary`
  - Portals: `ReactDOM.createPortal` for modals, tooltips

- [ ] **React debugging**
  - React DevTools: component tree, props/state inspection, Profiler
  - `useDebugValue`: custom hook debugging
  - `StrictMode`: detect side effects, deprecated APIs, double-rendering in dev
  - Common bugs: stale closures in useEffect, infinite re-render loops, missing dependency array

### 3.2 Next.js Deep Dive (Zero → Senior)

- [ ] **Rendering strategies & RSC architecture**
  - SSR (Server-Side Rendering): `getServerSideProps` (Pages Router) vs. Server Components (App Router)
  - SSG (Static Site Generation): `getStaticProps` + `getStaticPaths`, ISR (Incremental Static Regeneration)
  - CSR (Client-Side Rendering): `'use client'` directive, SPA behavior
  - **React Server Components (RSC)**: execute strictly on server, zero bundle JS sent to client, direct DB/ORM access
  - **RSC Payload**: serialized JSON-like stream of virtual DOM nodes + component references transmitted to browser
  - **Client Components & Hydration**: `'use client'` boundary; pre-rendered to HTML on server, interactive after client-side hydration
  - **Hydration mismatch errors**: caused by non-deterministic initial renders (browser-only APIs like `window`, differing server/client timezones, date formatting, browser extensions modifying DOM)
  - **Streaming SSR & Suspense**: HTTP/1.1 `Transfer-Encoding: chunked` or HTTP/2 streams; instant HTML shell rendering while slow async data chunks stream into `<Suspense fallback={<Skeleton />}>` placeholders

- [ ] **App Router (Next.js 13+)**
  - File-based routing: `app/` directory, `page.tsx`, `layout.tsx`, `loading.tsx`, `error.tsx`, `not-found.tsx`
  - Server Actions: `'use server'`, form mutations without custom API routes, automatic revalidation
  - Route groups: `(group)` folders for layout organization without affecting URL paths
  - Parallel routes: `@slot` simultaneous rendering; intercepting routes: modal view patterns
  - Middleware: `middleware.ts`, edge execution, request rewriting, redirects, JWT verification

- [ ] **Data fetching & caching matrix**
  - Server Components: `fetch()` with granular caching (`force-cache`, `no-store`, `next: { revalidate: 60 }`)
  - Request memoization: deduplicates identical `fetch` calls across component tree within single render pass
  - Data Cache: persistent cross-request cache; invalidation via `revalidatePath` and `revalidateTag`
  - Full Route Cache: statically optimized HTML + RSC payload cached on server
  - Router Cache: in-memory client-side cache of visited routes for instant back/forward navigation
  - `unstable_cache` / React `cache()`: memoizing and caching non-fetch database or service calls

- [ ] **Core Web Vitals & performance optimization**
  - **LCP (Largest Contentful Paint)**: target < 2.5s; optimize via `next/image` (`priority` on above-the-fold hero images), CDN edge caching
  - **INP (Interaction to Next Paint)**: target < 200ms; minimize main-thread JS blocking, break long tasks, use `useTransition`
  - **CLS (Cumulative Layout Shift)**: target < 0.1; reserve dimensions for dynamic elements, use `next/font` with zero layout shift
  - Image & font optimization: responsive srcsets, blur placeholders, self-hosted Google Fonts via `@next/font`

- [ ] **Deployment & production**
  - Vercel: edge functions, serverless API execution
  - Self-hosted Docker: `output: 'standalone'` in `next.config.js` for minimal production Docker container (~100MB)
  - Edge Runtime vs. Node.js Runtime: Edge lacks Node native APIs (`fs`, `net`, crypto primitives) but boots in <10ms

---

## 4. Working Knowledge Stack — Python, Laravel, Flutter

### 4.1 Python (Backend) *(JD-CRITICAL: listed as required language)*

- [ ] **Core language**
  - Data types: int, float, str, bool, list, tuple, dict, set, frozenset
  - List comprehensions, dict comprehensions, generator expressions
  - `*args`, `**kwargs`, unpacking operators
  - Decorators: `@decorator` syntax, `functools.wraps`, parameterized decorators
  - Context managers: `with` statement, `__enter__`/`__exit__`, `contextlib`
  - Type hints: `typing` module, `Optional`, `Union`, `List`, `Dict`, `TypeVar`, `Protocol`
  - GIL: Global Interpreter Lock, threading vs. multiprocessing vs. asyncio

- [ ] **Web frameworks**
  - FastAPI: async, Pydantic models, automatic OpenAPI docs, dependency injection
  - Django: ORM, admin, MVT pattern, middleware, signals
  - Flask: minimal, blueprints, extensions

- [ ] **Python for scripting & automation**
  - File I/O, CSV, JSON parsing
  - `requests` library: HTTP calls
  - `subprocess`: running shell commands
  - Virtual environments: `venv`, `pip`, `poetry`, `pyproject.toml`

### 4.2 Laravel (PHP)

- [ ] **Core concepts**
  - Eloquent ORM: models, relationships, eager loading, scopes, accessors/mutators
  - Blade templates: directives, components, slots
  - Artisan CLI: make commands, custom commands
  - Middleware, routing, controllers, form requests (validation)
  - Service container & dependency injection
  - Queues: Laravel Queue, drivers (Redis, database, SQS), jobs, failed jobs
  - Events & listeners, broadcasting
  - Laravel Sanctum (API tokens), Passport (OAuth2)

### 4.3 Flutter (Mobile)

- [ ] **Core concepts**
  - Dart language: null safety, `late`, `required`, async/await, `Future`, `Stream`
  - Widget tree: StatelessWidget vs. StatefulWidget, `build()`, `setState()`
  - Layout: Row, Column, Stack, Container, Expanded, Flexible, `MediaQuery`
  - Navigation: `Navigator`, named routes, `GoRouter`
  - State management: Provider, Riverpod, Bloc/Cubit, GetX
  - HTTP: `dio` / `http` package, API integration, JSON serialization (`json_serializable`)
  - Platform channels: calling native code (iOS/Android)
  - App lifecycle: `WidgetsBindingObserver`, `didChangeAppLifecycleState`

---

## 5. Databases — SQL & NoSQL *(JD-CRITICAL)*

### 5.1 PostgreSQL Deep Dive *(SELLING POINT: your optimization experience)*

- [ ] **Architecture & query lifecycle**
  - Query lifecycle: TCP fork (OS process ~5-10MB RAM per conn) → Parser (parse tree) → Analyzer (system catalogs) → Rewriter (view expansion) → Planner/Optimizer (cost estimation from table statistics) → Executor (disk block / memory page retrieval)
  - Shared buffers: in-memory page cache, `shared_buffers` tuning (typically 25% of system RAM)
  - WAL (Write-Ahead Log): crash recovery, durability guarantee, replication stream (`wal_level`)
  - VACUUM & Autovacuum: dead tuple cleanup, Free Space Map (FSM) updates; tuning `autovacuum_vacuum_scale_factor` down to 1% on tables with millions of rows to prevent bloat
  - Transaction ID Wraparound: 32-bit transaction IDs (2-billion limit), freeze vacuuming, monitoring `age(datfrozenxid)`
  - TOAST: large value storage, compression for oversized columns

- [ ] **Indexing deep dive & storage internals**
  - The B-tree: Root → Branch → Leaf (keys + `CTID` disk block pointers); 8KB block page splits on random inserts
  - `FILLFACTOR` tuning: reduce from 100 to 80-90 on update-heavy tables to allow HOT (Heap-Only Tuple) updates within the same 8KB page without updating index pointers
  - Hash index: equality only, fast point lookups
  - GIN (Generalized Inverted Index): full-text search, JSONB (`@>`), arrays
  - GiST (Generalized Search Tree): geometric data (PostGIS), range types, full-text
  - BRIN (Block Range Index): append-only time-series/log data (stores min/max per block range); microscopic memory & disk footprint compared to B-tree
  - Partial indexes: `WHERE` clause, index only relevant active rows
  - Expression indexes: index on function result, e.g., `lower(email)`
  - Composite indexes: column order matters (leftmost prefix rule)
  - Covering indexes: `INCLUDE` clause, index-only scans without heap table lookup
  - Index bloat: detection (`pgstattuple`), zero-downtime maintenance with `REINDEX CONCURRENTLY` and `pg_repack`

- [ ] **Query optimization** *(your 10x scaling story)*
  - `EXPLAIN (ANALYZE, BUFFERS)`: reading query plans; identifying `cost`, `actual time`, and `Buffers`
  - `shared hit` (RAM `shared_buffers`) vs `read` (physical disk seek I/O bottleneck)
  - Seq Scan vs. Index Scan vs. Index Only Scan vs. Bitmap Index / Heap Scan (bitmap in RAM, sorted block reads)
  - Join algorithms: Nested Loop (small sets), Hash Join (large unsorted sets), Merge Join (sorted inputs)
  - CTE vs. subquery: materialized CTEs (before PG 12), inline CTEs (PG 12+)
  - `pg_stat_statements`: identify slowest queries by total execution time, call counts, buffer reads
  - Connection pooling: PgBouncer (transaction pooling mode multiplexing 1,000 app connections to 50 DB connections)
  - Prepared statements: plan caching, parameterization

- [ ] **ACID & concurrency control**
  - MVCC: each transaction sees a consistent snapshot via `xmin`/`xmax` tuple headers
  - Isolation levels: Read Committed (default), Repeatable Read, Serializable
  - Serialization anomalies: dirty reads, non-repeatable reads, phantom reads, write skew
  - Row-level locks: `SELECT FOR UPDATE` for critical balance / inventory checks
  - Advisory locks: application-level distributed locking using Postgres engine (`pg_advisory_lock`)
  - Deadlock detection: `deadlock_timeout` (1s default), victim abort and application-level retry

- [ ] **Advanced features & multi-tenancy**
  - **Row-Level Security (RLS)**: strict database-level multi-tenant isolation via `SET LOCAL app.current_tenant = X`, preventing cross-organization data leaks even on faulty ORM queries
  - JSONB: operators (`->`, `->>`, `@>`, `?`), GIN indexing, when to use vs. normalized tables
  - Array types: `text[]`, `integer[]`, operators, unnesting
  - Window functions: `ROW_NUMBER`, `RANK`, `DENSE_RANK`, `LAG`, `LEAD`, `PARTITION BY`
  - Lateral joins: correlated subqueries as joins
  - Recursive CTEs: tree/graph traversal, organizational hierarchies
  - Partitioning: range (date ranges), list, hash — partition pruning in query execution
  - Foreign Data Wrappers (FDW): query external databases
  - `LISTEN`/`NOTIFY`: lightweight pub/sub

### 5.2 MySQL

- [ ] **Key differences from PostgreSQL**
  - InnoDB: default engine, ACID, row-level locking, clustered primary index (B+ tree where leaves contain full row data)
  - MyISAM: legacy, table-level locking, no transactions
  - `utf8mb4` vs. `utf8`: MySQL's `utf8` is broken (max 3 bytes), always use `utf8mb4` for full UTF-8 / emojis
  - `GROUP BY` behavior differences, `ONLY_FULL_GROUP_BY` mode
  - Auto-increment vs sequences, `LAST_INSERT_ID()`

### 5.3 MongoDB *(JD-CRITICAL: listed in requirements)*

- [ ] **Core concepts**
  - Documents, collections, databases
  - BSON format, `_id` field (ObjectId with timestamp, machine ID, process ID, counter)
  - Schema design: embed vs. reference decision framework
  - Embed: 1:few relationships, data read together, atomic updates; watch out for 16MB document size limit
  - Reference: 1:many/many:many, large unbounded subdocuments, independent access patterns

- [ ] **Querying**
  - Query operators: `$eq`, `$gt`, `$in`, `$regex`, `$elemMatch`, `$exists`
  - Update operators: `$set`, `$inc`, `$push`, `$pull`, `$addToSet`
  - Aggregation pipeline: `$match`, `$group`, `$project`, `$lookup` (joins), `$unwind`, `$sort`
  - Indexes: single field, compound, multikey (arrays), text, 2dsphere (geo)
  - `explain("executionStats")`: query plan analysis

- [ ] **Operations**
  - Replica sets: primary + secondaries, automatic failover, read preferences
  - Sharding: shard key selection, chunks, balancer, targeted vs. scatter-gather queries
  - Transactions: multi-document ACID (since 4.0), distributed transactions (since 4.2)
  - Change Streams: real-time event-driven applications

### 5.4 Redis *(JD-CRITICAL: cache management)*

- [ ] **Data structures & production use cases**
  - Strings: caching, counters (`INCR`), distributed locks (`SET key val NX EX 10`)
  - Hashes: object storage, partial field updates
  - Lists: FIFO queues (`LPUSH`/`RPOP`), recent activity feeds
  - Sets: unique collections, intersections, unions (`SINTER`, `SUNION`)
  - Sorted Sets (`ZSET`): leaderboards, priority queues, timestamped time-series
  - **Rate limiting via Redis Sorted Set (Sliding Window Log)**: scores as Unix epoch timestamps, `ZREMRANGEBYSCORE` to purge older window entries, `ZCARD` to count requests, atomic execution via Lua script (`EVAL`), return HTTP 429 if threshold exceeded
  - Streams: append-only log, consumer groups, message acknowledgment (`XADD`, `XREADGROUP`, `XACK`)
  - HyperLogLog: probabilistic cardinality estimation (unique visitors) with 12KB fixed memory
  - Bitmaps: daily user active streaks, feature flags

- [ ] **Caching patterns & cluster resilience**
  - Cache-aside: app checks Redis → miss → read DB → populate Redis with TTL
  - TTL strategies: absolute vs. sliding expiration, adding jitter to prevent simultaneous mass expiration
  - Cache invalidation: event-driven invalidation via database CDC or application hooks
  - Thundering herd / cache stampede: mutex locking or probabilistic early expiration (`XFetch`)
  - Persistence: RDB snapshots vs. AOF (Append-Only File with `everysec` fsync) vs. hybrid
  - Eviction policies: `allkeys-lru`, `volatile-lru`, `allkeys-lfu`, `noeviction`
  - Redis Cluster: hash slots (16384), automatic sharding, master-replica failover
  - Redlock algorithm for distributed locks: multiple Redis master consensus, clock drift caveats

---

## 6. Competitive Programming & DSA *(JD-CRITICAL: MUST HAVE)* *(SELLING POINT: 3000+ solved, ICPC)*

### 6.1 Data Structures — Internal Knowledge

- [ ] **Arrays & strings**
  - Contiguous memory, cache-friendly
  - Dynamic arrays: growth factor (1.5x-2x), amortized O(1) append
  - String immutability (in many languages), string builder patterns
  - Two-pointer technique, sliding window

- [ ] **Hash tables**
  - Hash function properties: deterministic, uniform distribution, avalanche effect
  - Collision resolution: chaining vs. open addressing (linear/quadratic probing, double hashing)
  - Load factor, resize at 0.75, rehash all entries — amortized O(1)
  - C++ `unordered_map`: hash table, O(1) avg, O(n) worst
  - Java `HashMap`: chaining with linked list → red-black tree at 8 elements
  - Worst case attacks: HashDoS, how to prevent (randomized hash seed)

- [ ] **Linked lists**
  - Singly, doubly, circular
  - Fast/slow pointer: cycle detection (Floyd's), find middle, find kth from end
  - Reversal: iterative (3 pointers), recursive
  - Merge two sorted lists, intersection of two lists
  - LRU Cache: doubly linked list + hash map

- [ ] **Stacks & queues**
  - Stack: LIFO, array or linked list implementation
  - Monotonic stack: next greater element, largest rectangle in histogram, daily temperatures
  - Queue: FIFO, circular array, linked list
  - Deque: double-ended queue
  - **Monotonic Queue in production**: strictly increasing/decreasing queue for streaming sliding window analytics (e.g., maximum concurrent requests in rolling 5-minute window) in $O(N)$ amortized time
  - Priority queue / binary heap: min/max heap, binary tree backed by array (`parent = (i-1)/2`, `left = 2i+1`)

- [ ] **Trees & production system applications**
  - BST: insert, delete, search, in-order successor/predecessor
  - Balanced BSTs: AVL (rotations), Red-Black tree (properties, recoloring)
  - Segment tree: range query + point update, lazy propagation
  - Fenwick / BIT: prefix sums, range updates
  - **Trie & Radix Trie in production**: prefix matching, autocomplete; Radix Trie (compressed Trie) used in modern HTTP routers (Next.js App Router, Rails Journey, Express) to route URLs in $O(L)$ (path length) instead of $O(N)$ regex linear scans
  - **Interval Trees**: overlapping interval search in scheduling systems (Calendly, Airbnb, Uber driver shifts) in $O(\log N + K)$ time
  - B-tree / B+ tree: disk I/O optimization, database indexes
  - Tree traversals: in-order, pre-order, post-order, level-order (BFS)
  - LCA (Lowest Common Ancestor): binary lifting, Euler tour + sparse table

- [ ] **Graphs**
  - Representations: adjacency list vs. adjacency matrix
  - BFS: shortest path (unweighted), level-order, multi-source BFS
  - DFS: cycle detection, topological sort, articulation points, bridges
  - Dijkstra: non-negative weights, O((V+E) log V) with priority queue
  - Bellman-Ford: negative weights, negative cycle detection
  - Floyd-Warshall: all-pairs shortest path, O(V³)
  - MST: Kruskal's (Union-Find), Prim's (priority queue)
  - Topological sort: Kahn's (BFS) vs. DFS-based
  - Strongly connected components: Tarjan's, Kosaraju's
  - Network flow: Ford-Fulkerson, max-flow min-cut theorem
  - Bipartite checking, matching (Hungarian algorithm basics)

- [ ] **Advanced structures & distributed algorithms**
  - Union-Find / DSU: path compression + union by rank/size → α(n) amortized
  - Sparse table: static RMQ in O(1) query, O(n log n) build
  - Suffix array, suffix automaton (string matching)
  - **Consistent Hashing ring**: why modulo hashing (`hash(key) % N`) fails on node addition/removal (invalidates almost 100% of cache); virtual nodes on $0-360^\circ$ ring, clockwise lookup; node failure re-routes only $1/N$ keys (Cassandra, DynamoDB, Redis Cluster)
  - Persistent data structures: concepts

### 6.2 Algorithmic Paradigms

- [ ] **Sorting**
  - Comparison sorts: merge sort (stable, O(n log n)), quick sort (in-place, avg O(n log n), worst O(n²))
  - Non-comparison: counting sort, radix sort, bucket sort — O(n) when applicable
  - Quick select: O(n) average kth element
  - Stability matters: when order of equal elements must be preserved

- [ ] **Binary search**
  - Classic: sorted array, O(log n)
  - Lower bound / upper bound variants
  - Search on answer: "minimum X such that f(X) is true" — binary search on monotonic function
  - Rotated sorted array, peak finding
  - Floating-point binary search for optimization

- [ ] **Sliding window & two pointers**
  - Fixed window: max sum of size k
  - Variable window: minimum window substring, longest substring without repeating
  - Two pointers: sorted array problems, container with most water, 3sum

- [ ] **Dynamic programming**
  - Top-down (memoization) vs. bottom-up (tabulation)
  - State design: what do you need to know? what changes between subproblems?
  - Classic patterns:
    - 1D: Fibonacci, climbing stairs, house robber, coin change
    - 2D: grid paths, LCS, edit distance, knapsack
    - Interval DP: matrix chain multiplication, burst balloons
    - Bitmask DP: TSP, subset problems
    - DP on trees: tree DP, rerooting
    - Digit DP: counting numbers with constraints
  - Space optimization: rolling array, O(n) → O(1) for Fibonacci-like

- [ ] **Greedy**
  - Activity selection, interval scheduling
  - Huffman coding, fractional knapsack
  - Exchange argument: prove greedy works
  - Greedy fails → DP needed: 0/1 knapsack, coin change (arbitrary denominations)

- [ ] **Backtracking**
  - State space tree, pruning
  - N-Queens, Sudoku, word search, permutations, combinations, subsets
  - Constraint propagation for efficiency

- [ ] **Graph algorithms** (advanced CP topics)
  - Shortest path on grid with BFS/DFS
  - 0-1 BFS: deque-based, edge weights 0 or 1
  - A*: heuristic search, admissible heuristic
  - Euler path/circuit: Hierholzer's algorithm
  - Maximum bipartite matching: Hopcroft-Karp

- [ ] **String algorithms**
  - KMP: prefix function, pattern matching O(n+m)
  - Z-algorithm: Z-array, pattern matching
  - Rabin-Karp: rolling hash, multiple pattern matching
  - Aho-Corasick: multi-pattern matching, trie + failure links
  - Manacher's: longest palindromic substring O(n)

- [ ] **Number theory & math**
  - GCD (Euclidean), Extended Euclidean, modular inverse
  - Sieve of Eratosthenes, prime factorization
  - Modular exponentiation: fast power O(log n)
  - Combinatorics: nCr with modular inverse, Pascal's triangle, Catalan numbers
  - Matrix exponentiation: linear recurrence acceleration

### 6.3 Complexity Analysis

- [ ] **Big-O mastery**
  - O(1), O(log n), O(√n), O(n), O(n log n), O(n²), O(2ⁿ), O(n!) — know examples of each
  - Amortized analysis: dynamic array, union-find
  - Best / average / worst case: quicksort pivot selection
  - Space complexity: auxiliary vs. total, recursion stack depth

- [ ] **Problem-solving meta-strategy**
  - Read → understand constraints (n ≤ 10⁵ → O(n log n) needed)
  - Brute force first → optimize
  - Pattern recognition: "is this a known problem variant?"
  - Edge cases: empty input, single element, max values, negative numbers, overflow
  - Time management: 45 min interview → 5 min understand, 5 min approach, 25 min code, 10 min test/debug

---

## 7. System Design *(JD-CRITICAL: MUST HAVE at Alex Xu level)*

### 7.1 System Design Framework (Alex Xu approach)

- [ ] **Step 1: Understand the problem & establish design scope (3-5 min)**
  - Ask clarifying questions: users, scale, features, constraints
  - Functional requirements: what the system should do
  - Non-functional requirements: availability, consistency, latency, throughput
  - Back-of-envelope estimation: DAU, QPS, storage, bandwidth

- [ ] **Step 2: Propose high-level design (10-15 min)**
  - API design: endpoints, request/response
  - Data model: schema, relationships
  - High-level architecture: client, API gateway, services, database, cache, message queue

- [ ] **Step 3: Design deep dive (10-15 min)**
  - Focus on interviewer's areas of interest
  - Discuss trade-offs, alternatives considered
  - Address bottlenecks, single points of failure

- [ ] **Step 4: Wrap up (3-5 min)**
  - Error handling, monitoring, metrics
  - Scaling considerations, future improvements
  - Summarize trade-offs made

### 7.2 Staff Architectural Mindset & Capacity Planning *(from Handbook Ch 3)*

- [ ] **Challenging the PRD (Why over How)**
  - Network constraints: avoid large POST uploads to app servers; use client-direct multipart S3 pre-signed uploads
  - Compute constraints: CPU-bound transcoding offloaded via S3 Events → SQS → Auto-scaling worker fleet or AWS MediaConvert
  - UX constraints: replace long-polling with WebSockets or SSE for async processing updates
  - Cost & scale calculation: e.g., 100,000 drivers * 5s ping = 20,000 writes/sec (60,000 peak) = 30 MB/s bandwidth = 1.7B writes/day. DynamoDB write pricing ($63,000/month) → push back to 15s ping to save $40,000/month

- [ ] **Build vs. Buy vs. Open Source Framework**
  - Code liability principle: build only core competitive business advantages
  - Commodity services to buy: Authentication (Clerk/Auth0/Devise/NextAuth), Search (Algolia/Elastic Cloud), Payments (Stripe), Transactional Email (SendGrid), Hosting (Vercel/ECS)

- [ ] **Back-of-Envelope Estimation**
  - Key numbers: 1 web server ≈ 1,000-10,000 QPS; Read:Write ratio typically 10:1 to 100:1
  - Storage: 1 tweet ≈ 300B, 1 photo ≈ 200KB-1MB, 1 video ≈ 50MB
  - Hardware latency: L1 cache (1ns) → RAM (100ns) → SSD (100μs) → Network datacenter (500μs) → Cross-continent (150ms)
  - Capacity estimation: Twitter (300M MAU, 600M tweets/day → 7,000 writes/sec, 700K reads/sec)

### 7.3 Core System Design Topics (Alex Xu Vol 1 & 2)

- [ ] **Rate limiter** — token bucket, sliding window log (Redis ZSET), distributed rate limiting
- [ ] **Consistent hashing** — virtual nodes, server addition/removal, minimal redistribution
- [ ] **Key-value store** — partitioning, replication, consistency, conflict resolution (vector clocks)
- [ ] **Unique ID generator** — Snowflake (timestamp + worker ID + sequence), UUID v4, database sequences
- [ ] **URL shortener** — base62, hash collision handling, 301 vs. 302 redirect analytics
- [ ] **Web crawler** — politeness, URL frontier, deduplication, robots.txt, trap detection
- [ ] **Notification system** — push (APNs/FCM), SMS, email, fan-out, rate limiting
- [ ] **News feed system** — fan-out on write (push) vs. fan-out on read (pull), hybrid model for celebrities
- [ ] **Chat system** — WebSocket, online presence heartbeat, message ordering, group chat, media storage
- [ ] **Search autocomplete** — trie, top-K cached at nodes, data collection pipeline
- [ ] **YouTube / video streaming** — client-direct S3 upload, transcoding worker queue, CDN adaptive bitrate (HLS/DASH)
- [ ] **Google Drive** — block-level chunking, deduplication, conflict resolution, metadata DB vs blob storage
- [ ] **Proximity service** — geohash, quadtree, Google S2, spatial indexing
- [ ] **Nearby friends** — WebSocket gateway, location update fan-out via Redis pub/sub
- [ ] **Google Maps** — graph routing (Dijkstra/A*), map tile rendering, ETA estimation
- [ ] **Payment system** — PSP integration, double-entry ledger, idempotency keys, async reconciliation
- [ ] **Hotel reservation** — inventory management, overbooking mitigation, optimistic vs pessimistic locking
- [ ] **Email service** — sending pipeline, receiving (MX records), spam filtering
- [ ] **S3-like object storage** — data/metadata separation, erasure coding, multipart upload
- [ ] **Real-time gaming leaderboard** — Redis sorted sets (`ZADD`, `ZREVRANK`), eventual consistency
- [ ] **Stock exchange** — order matching engine (price-time priority FIFO), event sourcing, ultra-low latency

### 7.4 Distributed Systems Concepts & Advanced Patterns

- [ ] **CAP theorem & PACELC** — CP (Postgres, MongoDB) vs. AP (Cassandra, DynamoDB); PACELC: partition -> A or C, else Latency or Consistency
- [ ] **Consistency models** — linearizability, sequential, causal, eventual, read-your-own-writes
- [ ] **Replication & Consensus** — leader-follower, replication lag mitigation, Raft consensus algorithm
- [ ] **CQRS (Command Query Responsibility Segregation)**: splitting write operations (normalized Postgres primary) from read operations (denormalized read replica or Elasticsearch)
- [ ] **The Saga Pattern**: distributed transactions across microservices; Event Choreography (Kafka events) vs. Event Orchestration (centralized coordinator like AWS Step Functions / Temporal) with compensating rollback transactions
- [ ] **The Outbox Pattern**: atomic local DB write + message dispatch; write entity + event to `outbox_events` table in single ACID transaction; Debezium CDC / polling worker reads WAL and publishes to Kafka with at-least-once delivery

### 7.5 Microservices Architecture *(JD-CRITICAL)*

- [ ] **Monolith → microservices**
  - When to migrate, strangler fig pattern
  - Service boundaries: domain-driven design, bounded contexts
  - **Strict rule**: database-per-service isolation; shared DB = distributed monolith anti-pattern

- [ ] **Communication patterns**
  - Synchronous: REST, gRPC (HTTP/2 binary protobuf, strictly typed, cascading failure risk)
  - Asynchronous: RabbitMQ ("Smart Broker, Dumb Consumer", task queues) vs. Kafka ("Dumb Broker, Smart Consumer", append-only immutable log, consumer offsets, replayability)
  - API Gateway: routing, auth, rate limiting, request aggregation

- [ ] **Operational challenges**
  - Service discovery: Consul, Kubernetes DNS
  - Circuit breaker: closed → open → half-open, failure thresholds
  - Distributed tracing: OpenTelemetry, trace/span ID propagation across HTTP headers
  - Config management: centralized config, feature flags

### 7.6 Real-World Architectural Case Studies *(from Handbook Ch 17)*

- [ ] **Case Study 1: Real-Time Ride-Sharing App (Uber-Lite)**
  - *Scale*: 100k active drivers, 5s GPS updates = 20,000 writes/sec
  - *Data layer*: Redis Geospatial (`GEOADD`, `GEORADIUS`) for ephemeral driver locations; Cassandra/DynamoDB for completed ride histories
  - *Pipeline*: Client WebSocket → WebSocket API Gateway → Kafka topic (`driver-locations`) → Node.js consumer workers → Redis; Matchmaking service runs `GEORADIUS` in $O(N+M)$ and pushes ride offers via WebSocket
  - *Infra*: EKS / Kubernetes with tuned ingress controllers for persistent TCP connections

- [ ] **Case Study 2: Flash Sale E-Commerce (The Ticketmaster Problem)**
  - *Scale*: 1,000 inventory units, 500,000 concurrent users at sale launch, zero overselling guarantee
  - *Architecture*: AWS CloudFront CDN (static assets) → API Gateway rate limiting → SQS buffer queue (bulkhead pattern to absorb the 500k RPS spike)
  - *Concurrency & Locking*: Worker fleet pulls from SQS; Redis Distributed Lock (Redlock) or atomic `DECR` for fast inventory reservation (10-minute hold); Saga pattern for payment processing via Stripe; Postgres `SELECT FOR UPDATE` for final ACID commit on checkout completion

- [ ] **Case Study 3: B2B Multi-Tenant SaaS (Project Management)**
  - *Scale*: 10,000 organizational tenants, strict legal data isolation
  - *Database isolation*: Shared database with PostgreSQL Row-Level Security (RLS) enforcing `SET LOCAL app.current_tenant = tenant_id`, preventing cross-tenant data leaks at the DB driver layer
  - *Noisy neighbor protection*: Redis sliding window rate limiter (1,000 RPM per tenant); heavy analytical reports routed to PostgreSQL Read Replicas
  - *Infra*: AWS ECS Fargate serverless containers with multi-stage Docker builds dropping compile-time secrets

---

## 8. Docker & DevOps *(JD-CRITICAL)* *(SELLING POINT)*

### 8.1 Docker Internals (Zero → Senior)

- [ ] **Linux primitives**
  - Namespaces: PID (process isolation), NET (network stack), MNT (filesystem), UTS (hostname), IPC (inter-process), USER (UID mapping), CGROUP
  - Cgroups v1/v2: CPU (shares, quota, period), memory (limit, reservation, OOM killer), I/O limits
  - OverlayFS: union filesystem, layers, copy-on-write
  - Seccomp: syscall filtering profiles
  - Capabilities: fine-grained root privilege splitting

- [ ] **Images**
  - Dockerfile instructions: `FROM`, `RUN`, `COPY`, `ADD`, `WORKDIR`, `ENV`, `ARG`, `EXPOSE`, `CMD`, `ENTRYPOINT`, `USER`, `VOLUME`, `HEALTHCHECK`
  - `CMD` vs. `ENTRYPOINT`: default command vs. fixed executable, shell form vs. exec form
  - Layer caching: order matters (least-changing first), `.dockerignore`
  - Multi-stage builds: separate build + runtime stages, minimize image size
  - Base images: `alpine` (small, musl libc), `slim` (Debian minimal), `distroless` (Google, no shell)
  - Image security: non-root user, read-only filesystem, minimal packages
  - OCI Image Spec: manifest, config, layers, content-addressable SHA256

- [ ] **Container runtime**
  - Docker daemon → containerd → runc
  - Container lifecycle: create → start → run → pause → stop → kill → remove
  - `docker exec`: attach to running container
  - Logs: `docker logs`, log drivers (json-file, syslog, fluentd)
  - Resource limits: `--memory`, `--cpus`, `--cpu-shares`

- [ ] **Networking**
  - Bridge network: default, container-to-container via IP, DNS resolution by container name
  - Host network: share host network stack, no isolation
  - Overlay network: multi-host networking (Swarm/K8s)
  - Port mapping: `-p host:container`, `EXPOSE` is documentation only
  - DNS: embedded DNS server resolves container names

- [ ] **Volumes & storage**
  - Bind mounts: host path → container, development use
  - Named volumes: Docker-managed, persistent across container lifecycle
  - tmpfs: in-memory, non-persistent, secrets
  - Volume drivers: NFS, cloud storage plugins

- [ ] **Docker Compose**
  - Service definitions, `depends_on` with `condition: service_healthy`
  - Health checks: `test`, `interval`, `timeout`, `retries`
  - Networks: custom bridge networks, service discovery
  - Environment: `.env` files, `environment` vs. `env_file`
  - Resource limits: `deploy.resources.limits.cpus/memory`
  - Profiles: selective service startup
  - `docker compose up --build`, `docker compose down -v`

- [ ] **Docker for Rails/Node.js**
  - Rails Dockerfile: multi-stage (build assets → runtime), `bundle install` layer caching
  - Node.js Dockerfile: `COPY package*.json` → `npm ci` → `COPY .` (layer caching)
  - Development vs. production Dockerfiles
  - `docker-compose.yml` for local dev: app + PostgreSQL + Redis + Sidekiq

### 8.2 Kubernetes (Awareness Level)

- [ ] **Core objects**: Pod, Deployment, Service, Ingress, ConfigMap, Secret, PV/PVC
- [ ] **Scaling**: HPA (CPU/memory metrics), `kubectl scale`
- [ ] **Deployments**: rolling update, rollback, readiness/liveness probes
- [ ] **Networking**: ClusterIP, NodePort, LoadBalancer, Ingress controllers
- [ ] **Helm**: package manager, charts, values, templating

### 8.3 CI/CD Pipelines

- [ ] **Pipeline stages**: lint → test → build image → security scan → deploy → smoke test
- [ ] **Tools**: GitHub Actions, GitLab CI, Bitbucket Pipelines (your experience)
- [ ] **Deployment strategies**: rolling, blue-green, canary, feature flags
- [ ] **GitOps**: ArgoCD, Flux — Git as source of truth
- [ ] **Docker image tagging**: semantic versioning, git SHA, `latest` pitfalls

### 8.4 AWS Cloud Architecture *(JD-CRITICAL)* *(your experience: EC2, ECS, S3)*

- [ ] **Compute & Orchestration**
  - EC2: instance sizing, EBS volume types (gp3 vs io2 IOPS), launch templates, auto-scaling groups
  - AWS ECS (Fargate vs. EC2): ECS Fargate for zero server maintenance, serverless task definitions, IAM Task Roles (no hardcoded keys)
  - AWS Lambda: serverless event-driven compute, cold starts, concurrency limits

- [ ] **Networking & VPC Design**
  - **VPC Architecture**: Public subnets (Internet Gateway, Application Load Balancers only) vs. Private subnets (NAT Gateway, EC2 instances, ECS tasks, RDS databases isolated from the internet)
  - **Security Groups & IAM Least Privilege**: Security groups referencing other SG IDs (e.g., PostgreSQL RDS SG allows inbound traffic on port 5432 *only* from the ECS Application SG ID, never 0.0.0.0/0)
  - Route 53 (DNS routing, health checks, failover) & ALB/NLB (Layer 7 path routing vs Layer 4 ultra-fast TCP)

- [ ] **Storage, Database & Messaging**
  - S3: bucket policies, CORS configuration, lifecycle management (Infrequent Access, Glacier archival), pre-signed PUT/GET URLs
  - RDS (PostgreSQL/MySQL): multi-AZ failover, read replicas, automated snapshots, storage auto-scaling
  - ElastiCache (Redis): cluster mode, replication groups, VPC peering
  - SQS (Standard vs. FIFO, dead-letter queues) & SNS (pub/sub fanout)
  - CloudWatch: log groups, metric alarms, dashboarding; AWS X-Ray for distributed tracing

### 8.5 Production OS Introspection & Advanced Git *(from Handbook Ch 3 & 12)*

- [ ] **OS-Level Production Troubleshooting**
  - **`strace` (System Call Tracing)**: trace kernel interactions of a hanging or frozen process (`strace -p <PID> -c`), spot socket deadlocks, blocked `epoll_wait`, or slow disk I/O
  - **`tcpdump` & Wireshark**: capture raw packets on server interfaces (`tcpdump -i eth0 port 443 -w trace.pcap`), isolate network packet loss, RST drops, and investigate **AWS SNAT port exhaustion** (when too many concurrent outbound requests exhaust NAT Gateway ports)
  - Profiling live processes: `rbspy` for non-invasive Ruby call stack sampling, Node.js `--inspect` and Chrome DevTools flamegraphs

- [ ] **Advanced Git Expertise (Staff Level)**
  - **Interactive Rebase (`git rebase -i`)**: curate commit history before merging; squash, fixup, and reword intermediate "WIP" commits into clean, atomic, descriptive units
  - **`git bisect`**: binary search $O(\log N)$ regression hunting between a known `good` and `bad` commit to instantly locate the exact commit that introduced a bug
  - **`git reflog`**: safety net recording every chronological movement of the `HEAD` pointer; recover lost commits or accidentally reset branches (`git reset --hard HEAD@{2}`)

---

## 9. Networking & Web Internals

### 9.1 What happens when you type a URL *(classic interview question)*

- [ ] Browser cache → OS cache (`/etc/hosts`) → DNS resolution (recursive resolver → root → TLD → authoritative name server)
- [ ] TCP 3-way handshake (SYN → SYN-ACK → ACK), TCP Fast Open (TFO)
- [ ] TLS handshake (1.2 vs. 1.3, certificate chain validation, cipher negotiation, forward secrecy)
- [ ] **Load balancer & Reverse Proxy (NGINX)**:
  - ALB terminates TLS and routes HTTP traffic to target groups based on URL path or headers
  - **NGINX Slow Client Buffering**: why application servers (Puma/Node) must sit behind NGINX; slow 3G/mobile clients take seconds to upload a request payload, which would starve backend worker threads if connected directly; NGINX asynchronously buffers the entire request in RAM using `epoll`, then blasts it to Puma/Node over localhost at gigabit speeds in <1ms, immediately freeing app workers
- [ ] Application server processing (Puma thread / Node event loop) → database query → HTTP response
- [ ] Browser rendering pipeline: HTML parsing → DOM tree, CSS parsing → CSSOM tree, Render tree, Layout (reflow), Paint, Composite

### 9.2 HTTP Deep Dive & Protocols

- [ ] **HTTP/1.1**: keep-alive persistent connections, head-of-line blocking, pipelining
- [ ] **HTTP/2**: binary framing, single-connection multiplexing, HPACK header compression, server push, stream prioritization
- [ ] **HTTP/3 / QUIC**: UDP-based transport, zero head-of-line blocking at packet level, integrated TLS 1.3, connection migration across IP address changes (Wi-Fi to LTE)
- [ ] **TCP vs. UDP in production**:
  - TCP: reliable, ordered byte stream, congestion control, retransmissions, connection handshake (used for HTTP, APIs, databases, SSH)
  - UDP: connectionless datagrams, zero handshake latency, no retransmission overhead, head-of-line blocking immune (used for DNS queries, VoIP, live video streaming, multiplayer gaming)
- [ ] **Status codes**: 200/201/204, 301 (permanent redirect) vs. 302/307 (temporary redirect) vs. 308, 400/401 (unauthenticated) vs. 403 (unauthorized)/404/409 (conflict)/422 (unprocessable)/429 (rate limited), 500/502 (Bad Gateway from crashed upstream)/503 (Service Unavailable)/504 (Gateway Timeout)
- [ ] **Headers**: Cache-Control directives, ETag validation (`If-None-Match`), Authorization, Content-Type, CORS security headers

### 9.3 HTTPS & TLS

- [ ] Symmetric (AES-256-GCM) vs. asymmetric encryption (RSA, ECDHE), hybrid session key negotiation
- [ ] Certificate chain: leaf certificate → intermediate CA → trusted root CA
- [ ] TLS 1.3: 1-RTT handshake, 0-RTT resumption (replay attack risk), ephemeral Diffie-Hellman (forward secrecy)
- [ ] mTLS (Mutual TLS): client & server mutually verify certificates; critical for zero-trust microservice communication
- [ ] OCSP stapling & Certificate Transparency (CT) logs

### 9.4 CDNs & Edge Infrastructure

- [ ] Edge PoPs, Anycast BGP routing, origin shield
- [ ] Cache-Control directives: `public`, `max-age`, `s-maxage`, `stale-while-revalidate`, `no-cache`, `no-store`
- [ ] Cache invalidation: instant purge, surrogate keys / tag-based invalidation, cache busting via content hashing

### 9.5 Real-Time Communication Protocols

- [ ] WebSocket: HTTP 101 Upgrade → persistent full-duplex bidirectional TCP connection; requires sticky sessions or Redis Pub/Sub backplane across multiple instances
- [ ] Server-Sent Events (SSE): unidirectional server → client text stream over standard HTTP; auto-reconnect, simpler than WebSockets, ideal for LLM token streaming and live financial tickers
- [ ] gRPC: HTTP/2 + Protocol Buffers; unary and bidirectional streaming; compile-time typed contracts, ideal for internal microservice communication

### 9.6 RESTful API Design & Pagination Strategies *(JD-CRITICAL)*

- [ ] Resource naming, HTTP verbs semantics, natural idempotency (GET, PUT, DELETE) vs. synthetic idempotency (POST with idempotency key)
- [ ] **Pagination Strategies at Scale**:
  - **Offset Pagination (`OFFSET X LIMIT Y`)**: the classic anti-pattern for large tables; DB must scan and discard $X$ rows from disk ($O(N)$), causing high latency and skipping/duplicate bugs when rows are added during traversal
  - **Keyset / Cursor Pagination (`WHERE id > cursor LIMIT Y`)**: the production standard; queries jump directly via the B-tree index in $O(\log N)$ time regardless of whether fetching page 1 or page 10,000
- [ ] Versioning strategies: URL path (`/v1/`) vs. custom header (`Accept-Version`) vs. content negotiation; zero-downtime deprecation workflows
- [ ] Error response contracts: standardized RFC 7807 Problem Details or consistent JSON schema
- [ ] GraphQL: query/mutation/subscription, schema-first design, DataLoader batching to solve N+1, query depth and complexity cost limiters

---

## 10. AI in Development Workflow *(SELLING POINT: resume highlights this)*

### 10.1 AI-Powered Development — Tools & Workflow

- [ ] **Agentic AI coding assistants**
  - Antigravity (Google): full agentic workflow, multi-file edits, subagents, skills, MCP
  - Claude Code (Anthropic): terminal-based, agentic, understands full codebase
  - Cursor: AI-first IDE, inline edits, chat, composer mode, codebase indexing
  - GitHub Copilot: inline completions, chat, Copilot Workspace
  - Windsurf/Codeium: Cascade flow, multi-step reasoning

- [ ] **Effective AI usage workflow**
  - **Plan → Prompt → Review → Test → Commit**
  - Start with clear spec: write requirements before prompting
  - Context management: provide relevant files, constraints, coding standards
  - Iterative refinement: refine prompts based on output quality
  - Never blindly accept: always review generated code for correctness, security, performance
  - Use AI for: boilerplate, test generation, refactoring, documentation, code review
  - Don't use AI for: critical security logic, novel algorithms without understanding

- [ ] **Pro-level agentic workflow**
  1. Define task clearly (WHAT, not HOW)
  2. Provide context: existing code, constraints, conventions
  3. Use `/plan` for complex tasks: get structured plan before execution
  4. Review diff carefully: understand every change
  5. Run tests: verify correctness, add edge case tests
  6. Use `/learn` to persist patterns and corrections
  7. Create skills/rules for repeated workflows
  8. Use subagents for parallel research tasks
  9. Customize with MCP servers for project-specific tools

- [ ] **AI for code review**
  - Automated PR review: identify bugs, style issues, security concerns
  - Suggest improvements: performance, readability, maintainability
  - Check test coverage gaps
  - Limitation: AI can miss context-dependent business logic errors

- [ ] **AI for testing**
  - Generate unit tests from function signatures
  - Generate edge case inputs
  - Property-based test generation
  - Test data generation for integration tests
  - Limitation: AI may generate tests that pass but don't test meaningful behavior

- [ ] **AI for debugging**
  - Paste error + stack trace → get diagnosis and fix suggestions
  - Explain complex error messages
  - Generate debugging scripts
  - Production log analysis

### 10.2 AI API Integration for Projects

- [ ] **OpenAI API**
  - Chat completions: `gpt-4o`, `gpt-4o-mini`, messages array (system/user/assistant)
  - Function calling / tool use: define tools, model returns structured calls
  - Structured outputs: JSON mode, response format schema
  - Streaming: SSE for real-time token delivery
  - Embeddings API: `text-embedding-3-small/large`, vector generation
  - Vision: image inputs in chat completions
  - Rate limits, token counting (`tiktoken`), cost optimization
  - Best practices: system prompt engineering, temperature tuning, max_tokens

- [ ] **Anthropic Claude API**
  - Messages API: system prompt, messages array, tool use
  - Extended thinking: chain-of-thought reasoning
  - Long context: 200K token window
  - Tool use / function calling

- [ ] **Google Gemini API**
  - `generateContent`, multimodal (text, image, video, audio)
  - Function calling, structured output
  - Grounding with Google Search

- [ ] **Integration patterns**
  - API wrapper service: centralize LLM calls, swap providers
  - Retry with exponential backoff for rate limits
  - Streaming responses to frontend: SSE/WebSocket relay
  - Token budget management: estimate before sending
  - Prompt templates: parameterized, version-controlled
  - Guardrails: input validation, output filtering, content moderation
  - Caching LLM responses: semantic cache with embeddings, exact match cache

- [ ] **RAG (Retrieval-Augmented Generation) implementation**
  - Document ingestion: load → chunk → embed → store in vector DB
  - Chunking: fixed-size with overlap, semantic boundaries
  - Vector databases: Pinecone, Weaviate, pgvector, Chroma
  - Query: embed user query → similarity search → inject context → LLM generates answer
  - Advanced: hybrid search (BM25 + vector), reranking, HyDE
  - Evaluation: retrieval metrics (recall@k), generation metrics (faithfulness)

- [ ] **AI agents in production**
  - ReAct pattern: Reason → Act → Observe → repeat
  - Tool use: web search, DB queries, API calls, code execution
  - Memory: conversation history, long-term (vector store)
  - Multi-agent orchestration: specialist agents, routing
  - Frameworks: LangChain, LlamaIndex, Semantic Kernel
  - Production concerns: cost monitoring, latency tracking, error handling, fallbacks

### 10.3 LLM Fundamentals (Conceptual)

- [ ] **Transformer architecture**: self-attention, multi-head attention, positional encoding
- [ ] **Inference**: KV cache, quantization (FP16→INT8→INT4), batching
- [ ] **Fine-tuning vs. RAG**: when to use each, LoRA for efficient fine-tuning
- [ ] **Prompt engineering**: zero-shot, few-shot, chain-of-thought, self-consistency

---

## 11. Security, Reliability & Observability

### 11.1 Security *(JD: "high quality code", "best practices")*

- [ ] **OWASP Top 10**: injection, broken auth, XSS, CSRF, SSRF, insecure deserialization
- [ ] **Auth**: JWT (claims, signing, revocation), OAuth 2.0 flows, API key management
- [ ] **Encryption**: at rest (KMS, field-level), in transit (TLS everywhere)
- [ ] **Secret management**: Rails credentials, environment variables, Vault, AWS Secrets Manager
- [ ] **CORS**: preflight, allowed origins, credentials
- [ ] **Container security**: non-root, image scanning (Trivy), read-only rootfs

### 11.2 Reliability & Resilience

- [ ] **Circuit breaker**: closed → open → half-open, failure threshold
- [ ] **Retry**: exponential backoff + jitter
- [ ] **Timeout**: connect + read timeout, deadline propagation
- [ ] **Bulkhead**: isolate resources per dependency
- [ ] **Graceful degradation**: fallback responses, cached data
- [ ] **Load shedding**: reject excess requests (503 + Retry-After)
- [ ] **Chaos engineering**: inject failures, build confidence

### 11.3 Observability

- [ ] **Three pillars**: logs (structured JSON), metrics (Prometheus), traces (OpenTelemetry)
- [ ] **RED method**: Rate, Errors, Duration (for services)
- [ ] **USE method**: Utilization, Saturation, Errors (for resources)
- [ ] **SLI/SLO/SLA**: measurable indicators, targets, contracts, error budgets
- [ ] **Alerting**: actionable alerts only, runbooks, PagerDuty/OpsGenie
- [ ] **Incident response**: detect → triage → mitigate → resolve → blameless postmortem

---

## 12. Behavioral & Past Projects *(JD: client communication, mentoring)*

### 12.1 Your Projects — Key Stories

- [ ] **Technonext 10x scaling** *(SELLING POINT)*
  - Situation: system crashing at 1,000 users
  - Action: compound indexing, query optimization, Redis caching, PG tuning, rate limiting
  - Result: 10,000+ users on same hardware, $0 infra cost
  - Depth: explain specific indexes added, query patterns fixed, Redis caching strategy, PgBouncer config

- [ ] **Multi-tenant auth engine** *(SELLING POINT)*
  - 4-layer authorization: gateway → user → role → module
  - Cross-organization data isolation
  - Technical decisions: why 4 layers, how middleware chain works

- [ ] **Gauntlet FinTech** *(SELLING POINT: payments, webhooks)*
  - Plaid + Stripe + Dwolla integration
  - ACH transfer flow, KYC verification pipeline
  - Webhook handling: idempotency keys, signature verification, retry handling, reconciliation
  - Multi-party ledger: double-entry bookkeeping
  - State machine: initiated → pending → processing → completed/failed

- [ ] **Auro24 Apple MDM**
  - Monolith → Rails API + Next.js migration
  - 2,000+ device onboarding, zero-downtime cutover
  - API design for device enrollment, policy enforcement

- [ ] **AI workflow integration in team**
  - How you introduced agentic AI workflows
  - Impact on feature delivery speed, testing, code reviews
  - Specific tools and process changes

### 12.2 Behavioral & Staff Leadership *(from Handbook Ch 4 & 15)*

- [ ] **STAR method execution**: Situation → Task → Action ("I", not "we") → Result (quantifiable business impact)
- [ ] **"Tell me about a time..." stories**
  - ...you scaled a system under pressure (Technonext 10x with zero infra cost)
  - ...you led an engineering team / mentored juniors (Technonext team lead)
  - ...you worked directly with non-technical founders/clients (Gauntlet US FinTech, Auro24 enterprise cutover)
  - ...you disagreed with a technical decision (disagree and commit vs. data-driven pushback)
  - ...you debugged a critical production outage (the hardest bug: e.g., ALB 502s from SNAT port exhaustion / connection pool timeout)
  - ...you had to ramp up on an unfamiliar stack rapidly

- [ ] **Staff Engineering Mindset & Force Multiplication**
  - The true definition of a "10x Engineer": not typing 10x more lines of code, but multiplying team velocity (making 10 engineers 2x more effective via tooling, shared libraries, unblocking, and architectural guidance)
  - Code liability principle: the best code is the code you convinced the team *not* to write

- [ ] **Client Communication & Business Alignment *(JD requirement)***
  - Requirement discovery: asking probing questions to uncover edge cases before development starts
  - Translating technical trade-offs into financial and business impact (e.g., explaining why reducing driver GPS polling interval from 5s to 15s saves $40,000/month in cloud infrastructure)
  - Managing scope creep and pushing back gracefully with data, alternative MVP phases, and delivery estimates

- [ ] **Mentoring & Junior Development *(JD requirement)***
  - **Socratic Mentorship**: teaching juniors "how to fish" by guiding their troubleshooting lifecycle (inspecting server logs, reproducing in isolation, reading network requests, tracing stack frames) rather than giving direct answers
  - **Pull Request (PR) Excellence**: eliminating rubber-stamp "LGTM" reviews; reviewing for atomic scope (single purpose, splitting mega-PRs), transaction rollback handling, error fallbacks, structured logging, and meaningful edge-case tests over vanity 100% test coverage

- [ ] **Production Incident Management (SEV-1 Protocol)**
  - Dedicated **Incident Commander (IC)** role: isolates external stakeholder communication from the engineering responders
  - **Mitigation first, debugging second**: priority is always restoring service (immediate rollback, traffic diversion, load shedding) before conducting root-cause analysis
  - **Blameless Post-Mortem**: eliminating "human error" as root cause; identifying systemic, architectural, and CI/CD guardrail failures that allowed the mistake to reach production

---

## 13. Resume Confidence Checklist

> **Goal**: Be able to explain every line of your resume in depth. No claim without substance.

- [ ] **"5+ years experience"** — timeline: RightCodes (Nov 2020 - Oct 2023, ~3 yrs) + Itransition (Nov 2023 - Dec 2025, ~2 yrs) + Technonext (Jan 2026 - present) = 5+ years ✅
- [ ] **"10x scaling, $0 infra"** — can you explain exact before/after metrics, specific optimizations?
- [ ] **"3,000+ problems solved"** — Codeforces profile, contest history, OJ profiles to show
- [ ] **"ICPC Regionalist"** — years (2018-2020), team name, regional standing
- [ ] **"Multi-tenant SaaS"** — explain tenant isolation, data partitioning, auth model
- [ ] **"Apple MDM"** — explain MDM protocol basics, device enrollment, profile management
- [ ] **"FinTech"** — explain ACH flow, KYC process, ledger reconciliation, compliance considerations
- [ ] **"PostgreSQL optimization"** — specific examples of queries optimized, indexes added
- [ ] **"Agentic AI workflows"** — specific tools used, how integrated into team process, measurable impact
- [ ] **Every technology listed** — can you answer 3 questions about each? If not, remove or study
- [ ] **Kubernetes** — listed on resume, be prepared to explain: pods, deployments, services, HPA (even if basic)
- [ ] **GraphQL** — listed on resume, be prepared for: query/mutation, resolvers, N+1 problem, DataLoader
- [ ] **React Native** — listed on resume, be prepared to differentiate from Flutter, when to choose each

---

> **Study Priority**:
> 1. **CP/DSA** (MUST HAVE) — you're strong here, but practice explaining solutions verbally
> 2. **System Design** (MUST HAVE) — study Alex Xu Vol 1 & 2 cover to cover
> 3. **Node.js + Python** (JD primary languages) — you know Rails, bridge to Node patterns
> 4. **Docker + AWS** (JD infrastructure) — deepen beyond usage to internals
> 5. **Your project stories** — rehearse with STAR, quantify everything
> 6. **AI workflow** — prepare demo-ready explanation of your agentic AI process
