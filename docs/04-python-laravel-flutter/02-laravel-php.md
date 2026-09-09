# 02 — Laravel & PHP: Framework Architecture, ORM, Queues & Distributed Systems

> **Context**: High-Throughput Web Platforms & Modern PHP (JD-CRITICAL). Comprehensive coverage of Laravel internals, Eloquent ORM mechanics, the Service Container & Reflection engine, asynchronous queues, real-time event broadcasting, and enterprise API security (Sanctum vs. Passport).

---

## 1. Eloquent ORM: Mechanics, Relationships & Performance

### 1.1 Active Record Pattern & Query Compilation

#### 1. Definition & Core Concept
Eloquent is an implementation of Martin Fowler’s **Active Record Pattern**. In Eloquent, a model class represents a database table, and an instance of that class represents a single row. The model encapsulates both database access (CRUD) and business domain logic.

#### 2. Internal Mechanics

```
Model::where('status', 'ACTIVE')->get()
               │
               ▼
1. Forwarded via __callStatic() to newQuery()
               │
               ▼
2. Instantiates Illuminate\Database\Eloquent\Builder
   └── Wraps underlying Illuminate\Database\Query\Builder (Query Grammar)
               │
               ▼
3. Scopes & Clauses added to Query\Builder ($bindings, $wheres, $columns)
               │
               ▼
4. Terminal Method get() called:
   ├── Query\Grammar compiles AST to SQL: "select * from `users` where `status` = ?"
   ├── Connection runs PDO statement with parameter bindings
   ├── Raw array of stdClass rows returned from PDO
   └── Eloquent\Builder::hydrate() instantiates Model instances,
       populating $attributes and original state ($original)
```

- **Attribute Hydration & Tracking**: When PDO returns raw database rows, Eloquent populates two distinct internal arrays on each model instance:
  - `$attributes`: Current state of the model (mutated via setters/accessors).
  - `$original`: Exact snapshot returned by the database.
- **Dirty Checking**: When calling `$model->save()`, Eloquent executes `getDirty()`, comparing `$attributes` against `$original`. If no values have changed, Eloquent cancels the `UPDATE` query entirely, preventing unnecessary disk I/O and lock contention in the database.

#### 3. Production Code & Real-World Usage

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Casts\Attribute;
use Illuminate\Database\Eloquent\Relations\HasMany;
use Illuminate\Database\Eloquent\Relations\BelongsTo;
use Illuminate\Database\Eloquent\Relations\MorphMany;
use Illuminate\Database\Eloquent\Builder;
use App\Enums\OrderStatus;

class Order extends Model
{
    protected $table = 'orders';

    protected $fillable = [
        'user_id',
        'order_number',
        'status',
        'metadata',
        'total_cents',
    ];

    // 1. Modern Attribute Casting (Laravel 9/10/11)
    protected function casts(): array
    {
        return [
            'status' => OrderStatus::class,       // Native PHP 8.1+ Enum casting
            'metadata' => 'encrypted:array',     // At-rest AES-256-CBC encryption
            'total_cents' => 'integer',
            'created_at' => 'immutable_datetime',
        ];
    }

    // 2. Modern Accessors & Mutators via Attribute::make
    protected function totalDollars(): Attribute
    {
        return Attribute::make(
            get: fn (mixed $value, array $attributes) => $attributes['total_cents'] / 100,
            set: fn (float $value) => ['total_cents' => (int) round($value * 100)]
        );
    }

    // 3. Relationships
    public function user(): BelongsTo
    {
        return $this->belongsTo(User::class);
    }

    public function items(): HasMany
    {
        return $this->hasMany(OrderItem::class);
    }

    public function auditLogs(): MorphMany
    {
        return $this->morphMany(AuditLog::class, 'auditable');
    }

    // 4. Local Query Scope
    public function scopeCompleted(Builder $query): Builder
    {
        return $query->where('status', OrderStatus::COMPLETED->value);
    }

    public function scopeForTenant(Builder $query, string $tenantId): Builder
    {
        return $query->where('tenant_id', $tenantId);
    }
}
```

#### 4. Production Pitfalls & Debugging

##### The N+1 Query Disaster & Prevention
If a developer accesses a relationship within a loop without eager loading:
```php
// BAD: Generates 1 query for orders + 100 queries for users (101 queries)
$orders = Order::all();
foreach ($orders as $order) {
    echo $order->user->name; // Lazy loading triggered on every iteration
}

// FIX: Eager loading generates exactly 2 queries
$orders = Order::with('user')->get();
// Query 1: select * from `orders`
// Query 2: select * from `users` where `users`.`id` in (1, 2, 3, ...)
```

To permanently prevent N+1 queries in production development, enforce strict loading in `AppServiceProvider`:

```php
// app/Providers/AppServiceProvider.php
public function boot(): void
{
    // Throws LazyLoadingViolationException in non-production environments
    Model::preventLazyLoading(! $this->app->isProduction());
    
    // Throws exception if saving an unfillable attribute or accessing missing attributes
    Model::preventSilentlyDiscardingAttributes(! $this->app->isProduction());
    Model::preventAccessingMissingAttributes(! $this->app->isProduction());
}
```

#### 5. Trade-offs & Decision Matrix

| Dimension | Active Record (Eloquent) | Data Mapper (Doctrine / Hibernate) | Raw PDO / Query Builder |
| :--- | :--- | :--- | :--- |
| **Development Velocity** | Extreme (intuitive method chaining) | Moderate (requires mapping config) | Moderate (manual query writing) |
| **Domain Purity** | Low (Models coupled to database schema) | High (Entities are plain PHP POPOs) | N/A |
| **Memory Consumption** | High (Hydrating 10k models consumes ~50MB) | Moderate | Minimal (PDO returns raw arrays) |
| **Complex Batch Ingestion** | Poor (Model events & dirty checks slow down inserts) | Moderate | Maximum (`insertOrIgnore`, multi-row raw SQL) |

#### 6. Senior Interview Q&A
- **Q**: *What is the difference between `lazy eager loading` (`load()`, `loadMissing()`) and standard `eager loading` (`with()`)?*
- **A**: `with()` modifies the initial SQL builder query *before* execution so that relationships are fetched immediately following the parent query. `load()` and `loadMissing()` are executed *after* model instances are already hydrated into memory. `loadMissing()` inspects the model's internal `$relations` array and only executes queries for relations that have not already been populated.

---

### 1.2 Scopes, Global Scopes & Multitenancy

#### 1. Definition & Core Concept
Scopes encapsulate common query constraints directly inside model definitions. **Global Scopes** apply constraints automatically to every single query executed against a model, making them the industry standard for implementing **soft deletes** and **multi-tenant data isolation**.

#### 2. Internal Mechanics
When `addGlobalScope()` is registered on a model (usually inside `booted()`), Eloquent stores the scope instance in its static `$globalScopes` array. Every time `newEloquentBuilder()` is invoked, Eloquent loops over registered global scopes and calls their `apply(Builder $builder, Model $model)` method, appending `WHERE` clauses to the SQL AST.

#### 3. Production Code & Real-World Usage

```php
<?php

namespace App\Models\Scopes;

use Illuminate\Database\Eloquent\Scope;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Builder;

class TenantScope implements Scope
{
    public function apply(Builder $builder, Model $model): void
    {
        if (app()->has('current_tenant_id')) {
            $builder->where($model->getTable() . '.tenant_id', '=', app('current_tenant_id'));
        }
    }
}

// In Model:
namespace App\Models;

use App\Models\Scopes\TenantScope;
use Illuminate\Database\Eloquent\Model;

class Invoice extends Model
{
    protected static function booted(): void
    {
        static::addGlobalScope(new TenantScope());
    }
}

// Bypassing scope for Super-Admin reporting:
$allInvoices = Invoice::withoutGlobalScope(TenantScope::class)->get();
```

#### 4. Production Pitfalls & Debugging
- **Global Scope Leaks in Background Workers / Artisan**: In a long-running queue worker or CLI command, if `current_tenant_id` is set once and never cleared, subsequent jobs running in the same PHP worker process will silently query against the previous job's tenant, creating a catastrophic multi-tenant data leak. Always clear container state after each job lifecycle.

---

## 2. Laravel Core Architecture & Lifecycle

### 2.1 The HTTP Request Lifecycle & Middleware Pipeline

#### 1. Definition & Core Concept
Every HTTP request entering a Laravel application passes through a unified front-controller (`public/index.php`), boots the Service Container, constructs a Request object, and flows through an "onion-skin" **Pipeline** of Middleware before reaching the Controller.

#### 2. Internal Mechanics

```
HTTP Request (via Nginx / Caddy)
       │
       ▼
public/index.php
       │
       ▼
bootstrap/app.php (Creates Application Instance)
       │
       ▼
Illuminate\Foundation\Http\Kernel::handle(Request $request)
       │
       ├── 1. Bootstrappers Run:
       │      - LoadEnvironmentVariables (.env)
       │      - LoadConfiguration (config/*.php cached via config:cache)
       │      - HandleExceptions
       │      - RegisterFacades
       │      - RegisterProviders (Service Providers register() phase)
       │      - BootProviders (Service Providers boot() phase)
       │
       ├── 2. Pipeline Execution (Illuminate\Pipeline\Pipeline)
       │      $pipeline->send($request)
       │               ->through($middleware)
       │               ->then($destination)
       │
       │      Request In:   [ Global MW ] ──► [ Group MW (api) ] ──► [ Route MW ]
       │                                                                      │
       │      Controller:                                               Controller@action
       │                                                                      │
       │      Response Out: [ Global MW ] ◄── [ Group MW (api) ] ◄── [ Route MW ]
       │
       └── 3. Terminable Middleware (terminate($request, $response)) runs AFTER socket flush
```

The Pipeline is implemented using functional composition via `array_reduce`. Each middleware wraps the next layer as a closure:
$$\text{Pipeline} = \text{array\_reduce}(\text{middleware}, \text{getInitialSlice}(), \text{destination})$$

#### 3. Production Code & Real-World Usage

```php
<?php

namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Symfony\Component\HttpFoundation\Response;
use Illuminate\Support\Facades\Log;

class PerformanceTelemetryMiddleware
{
    public function handle(Request $request, Closure $next): Response
    {
        $start = hrtime(true);

        // Forward request down the onion middleware chain
        $response = $next($request);

        // Add telemetry headers to response
        $durationMs = (hrtime(true) - $start) / 1e6;
        $response->headers->set('X-Response-Time-Ms', number_format($durationMs, 2));

        return $response;
    }

    /**
     * Terminable Middleware: Runs AFTER HTTP response has been sent to client.
     * Perfect for heavy audit logging or stats without penalizing user latency.
     */
    public function terminate(Request $request, Response $response): void
    {
        if ($response->getStatusCode() >= 400) {
            Log::warning('HTTP Error Handled', [
                'uri' => $request->getRequestUri(),
                'status' => $response->getStatusCode(),
                'ip' => $request->ip(),
                'user_id' => $request->user()?->id,
            ]);
        }
    }
}
```

---

### 2.2 The Service Container & Inversion of Control (IoC)

#### 1. Definition & Core Concept
The **Service Container** (`Illuminate\Container\Container`) is the central engine of Laravel. It manages class dependencies and performs **Dependency Injection (DI)** using PHP's `ReflectionClass` to automatically inspect and instantiate constructor dependencies without manual configuration.

#### 2. Internal Mechanics
1. **Binding Types**:
   - `bind($abstract, $concrete)`: Registers a transient binding; a fresh instance is constructed on every resolution.
   - `singleton($abstract, $concrete)`: Registers a shared binding; instantiated once upon first resolution and cached in the `$instances` array.
   - `scoped($abstract, $concrete)`: Instantiated once per request/job lifecycle; automatically cleared when the request ends or worker resets.
   - `instance($abstract, $instance)`: Binds an already-existing object instance directly.
2. **Contextual Binding**:
   Provides specific implementations depending on which class is asking for the dependency:
   `$this->app->when(ReportController::class)->needs(Filesystem::class)->give(S3Filesystem::class)`.
3. **Zero-Configuration Resolution (Auto-Wiring)**:
   When resolving an un-bound concrete class, the container uses reflection:
   - Inspects `ReflectionClass::getConstructor()`.
   - Iterates over `ReflectionParameter::getType()`.
   - Recursively resolves each parameter type from the container.

#### 3. Production Code & Real-World Usage

```php
<?php

namespace App\Providers;

use Illuminate\Support\ServiceProvider;
use App\Services\Payment\PaymentGatewayInterface;
use App\Services\Payment\StripePaymentGateway;
use App\Services\Payment\MockPaymentGateway;

class PaymentServiceProvider extends ServiceProvider
{
    public function register(): void
    {
        // Bind interface to implementation based on environment
        $this->app->singleton(PaymentGatewayInterface::class, function ($app) {
            if ($app->environment('testing')) {
                return new MockPaymentGateway();
            }

            return new StripePaymentGateway(
                apiKey: config('services.stripe.secret'),
                webhookSecret: config('services.stripe.webhook_secret')
            );
        });

        // Contextual binding example
        $this->app->when(\App\Services\HighVolumeExporter::class)
            ->needs(\Illuminate\Contracts\Filesystem\Filesystem::class)
            ->give(function () {
                return \Storage::disk('s3-archive');
            });
    }

    public function boot(): void
    {
        // Boot services, register event listeners, or publish configs
    }
}
```

#### 4. Production Pitfalls & Debugging
- **Memory Leaks via Singleton State in Octane / Queue Workers**: In standard PHP-FPM, memory is wiped clean at the end of each HTTP request. Under **Laravel Octane** (Swoole/RoadRunner) or long-running queue workers, the container persists in RAM. Mutating state on a singleton service (e.g. `$this->currentUser = $user`) bleeds data across multiple requests from completely different users.
  - *Fix*: Bind request-specific services with `scoped()`, or reset state in Octane event hooks (`OperationTerminated`).

#### 5. Trade-offs & Decision Matrix

| Container Binding | Lifecycle | Memory Impact | Best For |
| :--- | :--- | :--- | :--- |
| **`bind()`** | New instance per injection | Low (immediately GC-eligible after scope) | Lightweight stateless helpers, transient query models |
| **`singleton()`** | Single instance across app runtime | Persistent (Kept in `$instances` array) | Stateless database pools, SDK clients (Stripe, AWS) |
| **`scoped()`** | Single instance per HTTP request / Queue job | Flushed after each request cycle | Request context, Tenant identification, User-scoped audit loggers |

#### 6. Senior Interview Q&A
- **Q**: *How does Laravel’s Facade architecture resolve calls statically under the hood?*
- **A**: A Facade extends `Illuminate\Support\Facades\Facade` and implements `getFacadeAccessor()`, which returns a container binding string (e.g., `'db'` or `DB::class`). When a static method is called (`DB::table(...)`), PHP's `__callStatic()` magic method intercepts it, asks `Facade::resolveFacadeInstance('db')` from the Service Container, and executes the method dynamically on the underlying resolved object.

---

### 2.3 Form Requests & Robust Input Validation

#### 1. Definition & Core Concept
Form Requests (`Illuminate\Foundation\Http\FormRequest`) isolate authorization and validation logic away from controllers into dedicated, reusable request classes.

#### 2. Internal Mechanics
When type-hinted in a controller method, Laravel's Route Dependency Resolver resolves the Form Request through the Container. Before reaching the controller:
1. `prepareForValidation()` is invoked (used for data sanitization/normalization).
2. `authorize()` is called. If it returns `false`, an `AuthorizationException` is thrown, returning an immediate `403 Forbidden` response.
3. `rules()` are validated against the Validator Factory.
   - If validation fails on an API request (`Accept: application/json`), a `ValidationException` is thrown and converted to a `422 Unprocessable Entity` JSON response containing error messages.
   - If validation fails on a web request, it triggers an HTTP redirect back to the previous URL with errors flashed to the session.
4. `passedValidation()` is invoked for post-validation mutations.

#### 3. Production Code & Real-World Usage

```php
<?php

namespace App\Http\Requests;

use Illuminate\Foundation\Http\FormRequest;
use Illuminate\Validation\Rule;
use App\Enums\UserRole;

class UpdateUserRequest extends FormRequest
{
    public function authorize(): bool
    {
        // Enforce authorization policy: Only admins can update other accounts
        return $this->user()->can('update', $this->route('user'));
    }

    protected function prepareForValidation(): void
    {
        // Data normalization before validation
        if ($this->has('email')) {
            $this->merge([
                'email' => strtolower(trim($this->input('email'))),
            ]);
        }
    }

    public function rules(): array
    {
        $userId = $this->route('user')->id;

        return [
            'name' => ['required', 'string', 'max:255'],
            'email' => [
                'required',
                'email:rfc,dns',
                Rule::unique('users', 'email')->ignore($userId),
            ],
            'role' => ['required', Rule::enum(UserRole::class)],
            'department_id' => [
                'required',
                Rule::exists('departments', 'id')->where(fn ($q) => $q->where('is_active', true)),
            ],
        ];
    }
}
```

---

### 2.4 Artisan CLI & High-Volume Task Scheduling

#### 1. Definition & Core Concept
Artisan is Laravel’s command-line interface built on the Symfony Console component. The Task Scheduler allows declarative scheduling of cron jobs directly within PHP code without configuring multiple server crontabs.

#### 2. Internal Mechanics
Instead of managing dozens of crontab entries on production servers, a single cron entry runs every minute:
`* * * * * cd /path-to-project && php artisan schedule:run >> /dev/null 2>&1`
Artisan evaluates all scheduled events defined in `routes/console.php`. For each event, it evaluates frequency constraints, checks mutex locks in cache (`withoutOverlapping()`) to prevent concurrent executions of long jobs, and runs due tasks.

#### 3. Production Code & Real-World Usage

```php
<?php

namespace App\Console\Commands;

use Illuminate\Console\Command;
use App\Models\User;
use Illuminate\Support\Facades\DB;

class PruneStaleTokensCommand extends Command
{
    // Signature supporting arguments, flags, and options
    protected $signature = 'tokens:prune 
                            {--days=30 : Number of days of inactivity before pruning} 
                            {--dry-run : Simulate deletion without executing}';

    protected $description = 'Prune expired personal access tokens and abandoned sessions';

    public function handle(): int
    {
        $days = (int) $this->option('days');
        $isDryRun = $this->option('dry-run');
        $cutoff = now()->subDays($days);

        $this->info("Scanning for tokens inactive since: {$cutoff->toDateTimeString()}");

        $query = DB::table('personal_access_tokens')
            ->where('last_used_at', '<', $cutoff)
            ->orWhere(function ($q) use ($cutoff) {
                $q->whereNull('last_used_at')->where('created_at', '<', $cutoff);
            });

        $count = $query->count();

        if ($isDryRun) {
            $this->warn("[Dry-Run] {$count} tokens would be deleted.");
            return self::SUCCESS;
        }

        $this->output->progressStart($count);
        // Chunk deletion to prevent holding database write locks
        $query->chunkById(1000, function ($tokens) {
            DB::table('personal_access_tokens')->whereIn('id', $tokens->pluck('id'))->delete();
            $this->output->progressAdvance($tokens->count());
        });
        $this->output->progressFinish();

        $this->info("Successfully pruned {$count} expired tokens.");
        return self::SUCCESS;
    }
}
```

**Task Scheduling Configuration (`routes/console.php`):**
```php
use Illuminate\Support\Facades\Schedule;

Schedule::command('tokens:prune --days=14')
    ->dailyAt('03:00')
    ->withoutOverlapping(expiresAt: 60) // Mutex lock expires in 60 mins if worker dies
    ->onOneServer()                     // Prevents duplicate execution across multi-server clusters
    ->runInBackground();
```

---

## 3. Asynchronous Queues, Workers & Fault Tolerance

### 3.1 Architecture of Laravel Queue Drivers

#### 1. Definition & Core Concept
Laravel Queues offload time-consuming tasks (email delivery, PDF rendering, external API synchronization) to background workers, keeping web HTTP requests fast and responsive.

#### 2. Internal Mechanics

```
Web Request Process
       │
       ▼
Job Dispatched: ProcessPaymentJob::dispatch($order)
       │
       ├── Serializes Job Object (serialize() + SerializesModels trait)
       │   └── SerializesModels converts Model instances to:
       │       {"class": "App\\Models\\Order", "id": 9941, "relations": []}
       │
       ▼
Pushed to Driver (e.g., Redis):
  ├── If Delayed: Pushed to Redis Sorted Set (ZADD queues:default:delayed <timestamp> <payload>)
  └── If Immediate: Pushed to Redis List (LPUSH queues:default <payload>)
       │
       ▼
Background Worker Process: php artisan queue:work redis --queue=high,default
  ├── Atomic Pop: RPOPLPUSH queues:default queues:default:reserved
  ├── Unserializes Payload (Restores models from database via findOrFail(9941))
  ├── Invokes handle() method inside try-catch block
  │     ├── Success ──► LREM queues:default:reserved (Job completed)
  │     └── Exception ► Moves to queues:default:delayed (with exponential backoff)
  │                     OR writes to `failed_jobs` table if attempts exhausted
```

#### 3. Production Code & Real-World Usage

```php
<?php

namespace App\Jobs;

use App\Models\Order;
use App\Services\PaymentGateway;
use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\SerializesModels;
use Illuminate\Support\Facades\Log;
use Throwable;

class ProcessOrderPaymentJob implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    // Retry configuration
    public int $tries = 5;
    public int $maxExceptions = 3;
    public int $timeout = 120; // 2 minutes max before worker kills the job

    /**
     * Exponential backoff in seconds: 5s, 15s, 60s, 300s
     */
    public function backoff(): array
    {
        return [5, 15, 60, 300];
    }

    public function __construct(
        public Order $order,
        public string $idempotencyKey
    ) {}

    public function handle(PaymentGateway $gateway): void
    {
        // Fail early if order is already processed
        if ($this->order->is_paid) {
            return;
        }

        Log::info("Processing payment for Order #{$this->order->id}, attempt: {$this->attempts()}");

        $receipt = $gateway->charge([
            'amount' => $this->order->total_cents,
            'idempotency_key' => $this->idempotencyKey,
        ]);

        $this->order->update([
            'is_paid' => true,
            'transaction_id' => $receipt->id,
        ]);
    }

    /**
     * Permanent failure callback when all retry attempts are exhausted.
     */
    public function failed(?Throwable $exception): void
    {
        Log::critical("Payment processing completely failed for Order #{$this->order->id}", [
            'error' => $exception?->getMessage(),
            'trace' => $exception?->getTraceAsString(),
        ]);

        $this->order->update(['status' => 'PAYMENT_FAILED']);
        // Trigger alert notification to DevOps/FinTech team
    }
}
```

#### 4. Production Pitfalls & Worker Process Management

##### The Worker Memory Leak Hazard
In PHP, scripts typically run for milliseconds and exit, freeing all memory. However, `php artisan queue:work` is a **long-running daemon process**. If your job instantiates static objects or appends to global arrays, the process will slowly exhaust RAM until the OS kernel issues an Out-Of-Memory kill (`SIGKILL`).

```ini
# Production Supervisor Configuration: /etc/supervisor/conf.d/laravel-worker.conf
[program:laravel-worker]
process_name=%(program_name)s_%(process_num)02d
command=php /var/www/html/artisan queue:work redis --sleep=3 --tries=3 --max-time=3600 --memory=128
autostart=true
autorestart=true
user=www-data
numprocs=8
redirect_stderr=true
stdout_logfile=/var/log/supervisor/worker.log
stopwaitsecs=3600
```

- `--memory=128`: Worker gracefully restarts itself if it exceeds 128MB RAM after completing a job.
- `--max-time=3600`: Restarts worker every hour to eliminate long-term fragmentation.
- When deploying code updates, run:
  ```bash
  php artisan queue:restart
  ```
  This signals all running workers to finish their current job and gracefully exit. Supervisor immediately boots fresh workers running the new code.

#### 5. Trade-offs & Decision Matrix

| Driver | Latency | Max Throughput | Persistence Guarantee | Delayed Jobs Support | Operational Overhead |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Redis** | < 1ms | > 20,000 jobs/sec | High (with AOF enabled) | Native (Sorted Sets) | Low (if Redis already present) |
| **Database** | 5-20ms | ~500 jobs/sec | Maximum (ACID transactions) | Poor (Polls index every second) | Zero (uses existing DB) |
| **Amazon SQS** | 20-50ms | Unlimited | Maximum (Managed distributed queue) | Up to 15 minutes max | Zero infrastructure |

#### 6. Senior Interview Q&A
- **Q**: *What happens if a worker processing a job on Redis crashes abruptly (SIGKILL or server power failure)?*
- **A**: Laravel uses Redis `RPOPLPUSH` (or modern Lua script atomic equivalent) to atomically move the job from `queues:default` to a `queues:default:reserved` list with an expiration timestamp. If the worker dies mid-execution, the job remains in the reserved list. When another worker runs or a health check checks stale jobs, if `retry_after` seconds have passed, Laravel moves the job back to the active queue for re-processing.

---

## 4. Events, WebSockets Broadcasting & Real-Time Architecture

### 4.1 Event Dispatcher & Queued Listeners

#### 1. Definition & Core Concept
The Event Dispatcher provides an implementation of the **Observer Pattern**, decoupling distinct domain concepts (e.g. `OrderPlaced` event -> `SendEmailReceipt`, `UpdateInventory`, `NotifySlack` listeners).

#### 2. Internal Mechanics
- Synchronous Listeners run sequentially in the exact same thread and database transaction as the event dispatch call.
- Queued Listeners implement the `ShouldQueue` interface. When dispatched, Laravel automatically packages the listener into a background job and pushes it to the Queue driver.

```
Order Controller
       │
       ▼
Event::dispatch(new OrderPlaced($order))
       │
       ├── Synchronous Listener: ValidateFraudCheck (Runs immediately in HTTP thread)
       │
       └── Asynchronous Listener: SendOrderConfirmationEmail (implements ShouldQueue)
             │
             └── Automatically pushed to Redis Queue!
```

---

### 4.2 WebSockets Broadcasting (Reverb / Soketi / Pusher)

#### 1. Definition & Core Concept
Broadcasting connects server-side Laravel Events to client-side frontend applications (React, Flutter, Vue) in real time over WebSockets.

#### 2. Internal Mechanics
1. Event implements `ShouldBroadcast`.
2. Laravel serializes event public properties into a JSON broadcast payload.
3. The event is pushed to a queue worker.
4. The queue worker transmits the payload via HTTP/WebSocket to the WebSocket Server (Laravel Reverb, Soketi, or Pusher).
5. The WebSocket Server routes the payload to connected clients subscribed to the channel.

```
Laravel Event (implements ShouldBroadcast)
           │
           ▼
Queue Worker (Pushes to Redis / HTTP)
           │
           ▼
WebSocket Server (Laravel Reverb / Soketi)
           ├── Evaluates Channel Authorization (routes/channels.php)
           └── Broadcasts JSON to authorized client WebSockets
```

#### 3. Production Code & Real-World Usage

```php
<?php

namespace App\Events;

use App\Models\ChatMessage;
use Illuminate\Broadcasting\PrivateChannel;
use Illuminate\Broadcasting\PresenceChannel;
use Illuminate\Contracts\Broadcasting\ShouldBroadcast;
use Illuminate\Foundation\Events\Dispatchable;
use Illuminate\Queue\SerializesModels;

class NewChatMessageEvent implements ShouldBroadcast
{
    use Dispatchable, SerializesModels;

    public function __construct(
        public ChatMessage $message,
        public int $roomId
    ) {}

    /**
     * Broadcast on a private channel requiring authorization.
     */
    public function broadcastOn(): array
    {
        return [
            new PrivateChannel("chat.room.{$this->roomId}"),
        ];
    }

    /**
     * Customize the broadcast event name.
     */
    public function broadcastAs(): string
    {
        return 'message.created';
    }

    /**
     * Filter payload sent over the wire (prevent leaking sensitive columns).
     */
    public function broadcastWith(): array
    {
        return [
            'id' => $this->message->id,
            'content' => $this->message->body,
            'sender_name' => $this->message->user->name,
            'created_at' => $this->message->created_at->toISOString(),
        ];
    }
}
```

**Channel Authorization (`routes/channels.php`):**
```php
use App\Models\User;
use App\Models\ChatRoom;

// Authorize private channel
Broadcast::channel('chat.room.{roomId}', function (User $user, int $roomId) {
    // Return true if user is a member of this chat room
    return $user->chatRooms()->where('chat_rooms.id', $roomId)->exists();
});
```

---

## 5. API Authentication: Laravel Sanctum vs. Laravel Passport

### 5.1 Architecture & Token Verification Mechanics

#### 1. Definition & Core Concept
- **Laravel Sanctum**: A lightweight authentication system designed for SPAs (cookie-based stateful sessions) and mobile APIs (database-backed Personal Access Tokens).
- **Laravel Passport**: A full OAuth2 server implementation based on the `league/oauth2-server` package, supporting full OAuth2 grant specifications (Authorization Code with PKCE, Client Credentials, Refresh Tokens).

#### 2. Internal Mechanics

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                             Laravel Sanctum                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│ • Token Format: Raw string formatted as "<token_id>|<64_char_random_hash>"  │
│ • Database Query: SELECT * FROM personal_access_tokens WHERE id = <token_id>│
│ • Verification: SHA-256 hash comparison against stored hashed value         │
│ • Cost: 1 fast indexed Database/Cache query per authenticated API request   │
│ • Token Revocation: Instantaneous (simply DELETE the database row)         │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                             Laravel Passport                                │
├─────────────────────────────────────────────────────────────────────────────┤
│ • Token Format: Self-contained JSON Web Token (JWT) signed with RSA private │
│   key (oauth-private.key)                                                   │
│ • Database Query: ZERO database queries required to verify valid token!     │
│ • Verification: Cryptographic verification using public key (oauth-public)  │
│ • Token Revocation: Requires checking token ID in Redis/DB blacklist        │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### 3. Production Code & Real-World Usage

##### Laravel Sanctum API Token Issuance & Ability Guarding
```php
<?php

namespace App\Http\Controllers\Api;

use App\Http\Controllers\Controller;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Hash;
use App\Models\User;
use Illuminate\Validation\ValidationException;

class AuthController extends Controller
{
    public function issueToken(Request $request)
    {
        $request->validate([
            'email' => 'required|email',
            'password' => 'required',
            'device_name' => 'required|string',
        ]);

        $user = User::where('email', $request->email)->first();

        if (! $user || ! Hash::check($request->password, $user->password)) {
            throw ValidationException::withMessages([
                'email' => ['Invalid credentials supplied.'],
            ]);
        }

        // Issue token with granular ability scopes and expiration
        $token = $user->createToken(
            $request->device_name,
            ['orders:read', 'orders:create'], // Abilities
            now()->addDays(30)               // Expiration
        );

        return response()->json([
            'token' => $token->plainTextToken,
            'expires_at' => now()->addDays(30)->toIso8601String(),
        ]);
    }
}
```

**Guarding Routes via Abilities:**
```php
Route::middleware(['auth:sanctum', 'ability:orders:create'])->post('/orders', [OrderController::class, 'store']);
```

#### 4. Trade-offs & Decision Matrix

| Dimension | Laravel Sanctum | Laravel Passport |
| :--- | :--- | :--- |
| **Target Architecture** | First-party SPAs, Mobile Apps, Internal APIs | Third-party developer platform, Enterprise SSO |
| **Complexity & Overhead** | Low (Single migration, single config) | High (Requires RSA key pairs, multiple DB tables) |
| **Token Verification Latency** | Indexed DB query (~1-2ms) or cached | Microsecond cryptographic CPU check (Zero DB queries) |
| **OAuth2 Grant Types** | None (Personal Access Tokens only) | Full (Auth Code + PKCE, Client Credentials, Refresh) |
| **Revocation Velocity** | Immediate (Delete DB row) | Eventual / Requires Revocation Table checks |

#### 5. Senior Interview Q&A
- **Q**: *When would you choose Laravel Passport over Laravel Sanctum in an enterprise environment?*
- **A**: Choose Passport when your platform must act as an OAuth2 Identity Provider (OpenID Connect / OAuth2 server) allowing third-party applications to request scoped access on behalf of users via Authorization Code grants with PKCE (like "Login with Google/GitHub"). Choose Sanctum for mobile apps and single-page applications where first-party API access is all that is required.
