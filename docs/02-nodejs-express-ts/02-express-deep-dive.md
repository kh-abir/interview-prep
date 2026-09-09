# 02 — Express.js Deep Dive & Production Architecture

> **Context**: Primary Node.js web framework. Covers middleware onion mechanics, async error propagation, layered enterprise architecture, validation, and database abstraction layers (Prisma/Knex/Mongoose).

---

## 1. Express Middleware Pipeline Mechanics

### 1.1 Middleware Onion Execution Order
Express is fundamentally a routing and middleware web framework. An incoming HTTP request passes through a linked list of middleware functions.

```
Incoming HTTP Request
       │
       ▼
[ Middleware 1: Request Correlation ID ] (sets req.id)
       │ next()
       ▼
[ Middleware 2: Rate Limiter (Redis)   ] (checks IP/Tenant quota)
       │ next()
       ▼
[ Middleware 3: Security Headers       ] (Helmet)
       │ next()
       ▼
[ Middleware 4: Body Parser            ] (express.json)
       │ next()
       ▼
[ Middleware 5: Authentication (JWT)   ] (verifies token, populates req.user)
       │ next()
       ▼
[ Route Handler / Controller           ] (invokes Service layer, returns res.json)
       │ (Error thrown or passed via next(err))
       ▼
[ Centralized Error Handler (4 params) ] (formats RFC 7807 problem details)
```

### 1.2 Async Error Handling: Express 4 vs Express 5
- **Express 4**: If an unhandled rejection occurs inside an `async` route handler without `try/catch`, it bypasses Express and triggers `unhandledRejection` (or crashes the Node.js process in Node 15+). Requires wrapping every async controller with an async error boundary.
- **Express 5**: Native support for returned Promises; automatically passes rejected promises to `next(err)`.

```javascript
// Express 4 Async Wrapper Utility
const asyncHandler = (fn) => (req, res, next) => {
  Promise.resolve(fn(req, res, next)).catch(next);
};

// Usage in Route
router.get(
  '/users/:id',
  asyncHandler(async (req, res) => {
    const user = await userService.getUserById(req.params.id);
    if (!user) throw new NotFoundError(`User ${req.params.id} not found`);
    res.json({ success: true, data: user });
  })
);
```

### 1.3 The 4-Argument Error Middleware
Express distinguishes error-handling middleware strictly by **function arity** (`fn.length === 4`). If you omit `next`, Express treats it as a standard request middleware.

```typescript
// src/middleware/errorHandler.ts
import { Request, Response, NextFunction } from 'express';
import { AppError } from '../errors/AppError';
import { logger } from '../utils/logger';

export function errorHandler(
  err: Error,
  req: Request,
  res: Response,
  next: NextFunction
): void {
  // If headers already sent, delegate to default Express error handler to close stream
  if (res.headersSent) {
    return next(err);
  }

  const requestId = (req.headers['x-request-id'] as string) || 'unknown';

  if (err instanceof AppError) {
    logger.warn({ err, requestId }, 'Operational business error handled');
    res.status(err.statusCode).json({
      type: err.type,
      title: err.message,
      status: err.statusCode,
      details: err.details,
      instance: req.originalUrl,
      requestId
    });
    return;
  }

  // Programmer or unhandled infrastructure error
  logger.error({ err, requestId }, 'Unhandled SEV-2 system error encountered');
  res.status(500).json({
    type: 'https://api.internal/errors/internal-server-error',
    title: 'Internal Server Error',
    status: 500,
    requestId
  });
}
```

---

## 2. Layered Enterprise Architecture

```
src/
├── routes/          # Defines HTTP verbs, paths, and middleware attachments
├── controllers/     # Unpacks HTTP requests (req.params, req.body), calls services
├── services/        # Pure business logic; zero awareness of Express `req` or `res`
├── repositories/    # Database queries (Prisma, Knex, or Mongoose)
├── models/          # Domain entities and schemas
├── middleware/      # Auth, rate limiting, validation, request tracing
└── utils/           # Shared loggers, cryptography, helpers
```

### 2.1 Separation of Concerns Implementation
```typescript
// 1. Controller: Purely handles HTTP transport
export class OrderController {
  constructor(private orderService: OrderService) {}

  createOrder = async (req: Request, res: Response) => {
    const { items, currency } = req.body;
    const userId = req.user.id; // populated by auth middleware

    const order = await this.orderService.placeOrder({ userId, items, currency });
    res.status(201).json({ data: order });
  };
}

// 2. Service: Encapsulates domain logic & transactions
export class OrderService {
  constructor(
    private orderRepo: OrderRepository,
    private inventoryService: InventoryService,
    private paymentGateway: PaymentGateway
  ) {}

  async placeOrder(dto: CreateOrderDTO): Promise<Order> {
    // 1. Reserve inventory
    await this.inventoryService.reserveItems(dto.items);

    // 2. Charge customer via external payment processor
    const charge = await this.paymentGateway.authorize(dto.userId, dto.items);

    // 3. Persist order entity
    return this.orderRepo.create({
      userId: dto.userId,
      items: dto.items,
      paymentReference: charge.id,
      status: 'PAID'
    });
  }
}
```

---

## 3. Schema Validation with Zod

Never trust client input. Validate HTTP params, query parameters, and body payloads at the route perimeter.

```typescript
// src/middleware/validate.ts
import { Request, Response, NextFunction } from 'express';
import { AnyZodObject, ZodError } from 'zod';

export const validateRequest = (schema: AnyZodObject) => {
  return async (req: Request, res: Response, next: NextFunction) => {
    try {
      const parsed = await schema.parseAsync({
        body: req.body,
        query: req.query,
        params: req.params,
      });
      req.body = parsed.body;
      req.query = parsed.query;
      req.params = parsed.params;
      next();
    } catch (error) {
      if (error instanceof ZodError) {
        return res.status(422).json({
          status: 'error',
          errors: error.errors.map((e) => ({
            field: e.path.join('.'),
            message: e.message,
          })),
        });
      }
      next(error);
    }
  };
};

// Route Usage:
const CreateUserSchema = z.object({
  body: z.object({
    email: z.string().email(),
    password: z.string().min(8),
    role: z.enum(['ADMIN', 'MEMBER']).default('MEMBER'),
  }),
});

router.post('/users', validateRequest(CreateUserSchema), userController.create);
```

---

## 4. ORM & Query Builders Comparison

| Tool | Approach | Type Safety | Migration Tooling | Performance / Overhead |
|---|---|---|---|---|
| **Prisma** | Schema-first (`schema.prisma`), auto-generated client | **Absolute** (end-to-end generated types) | Excellent (`prisma migrate`) | Query engine binary (Rust) adds small microsecond IPC overhead; optimized batching |
| **Knex.js** | Pure SQL Query Builder | Partial (manual type annotations) | Robust native migrations | **Near-zero overhead**; direct driver queries; full SQL flexibility |
| **Sequelize** | Classic Active Record ORM | Moderate (clunky TypeScript typings) | Sequelize-CLI | Higher heap allocation overhead per model instance |
| **Mongoose** | ODM for MongoDB (Documents) | High (with TypeScript interfaces) | None (schema-less engine) | Object wrapping overhead; use `.lean()` for read performance |

### 4.1 Prisma in Production: Clean Transactions & Batched Queries
```typescript
import { PrismaClient } from '@prisma/client';

const prisma = new PrismaClient({
  log: process.env.NODE_ENV === 'development' ? ['query', 'error', 'warn'] : ['error'],
});

export async function transferBalance(
  fromUserId: string,
  toUserId: string,
  amountCents: number
) {
  // Prisma interactive transaction provides ACID semantics with auto-rollback
  return await prisma.$transaction(async (tx) => {
    const sender = await tx.account.update({
      where: { userId: fromUserId },
      data: { balanceCents: { decrement: amountCents } },
    });

    if (sender.balanceCents < 0) {
      throw new Error(`Insufficient funds for account: ${fromUserId}`);
    }

    const recipient = await tx.account.update({
      where: { userId: toUserId },
      data: { balanceCents: { increment: amountCents } },
    });

    const ledger = await tx.transferLedger.create({
      data: {
        fromUserId,
        toUserId,
        amountCents,
      },
    });

    return { sender, recipient, ledger };
  });
}
```

### 4.2 Mongoose Performance Trap: Mongoose Document Overhead
```typescript
// ANTI-PATTERN: Loading 10,000 documents as Mongoose models
// Each model wraps getters, setters, dirty flags, and validation schemas (~3KB/doc)
const orders = await OrderModel.find({ status: 'COMPLETED' });

// PRODUCTION FIX: Always use .lean() for read-only queries!
// Returns raw plain JavaScript objects directly from BSON parser
const orders = await OrderModel.find({ status: 'COMPLETED' }).lean().exec();
// Result: 5x faster query execution, 75% lower heap allocation!
```

---

## 5. Security & Rate Limiting

### 5.1 Distributed Rate Limiting via Redis
In multi-instance Docker/ECS deployments, in-memory rate limiting fails because traffic load-balances across instances. You must use a shared Redis store.

```typescript
import rateLimit from 'express-rate-limit';
import RedisStore from 'rate-limit-redis';
import Redis from 'ioredis';

const redisClient = new Redis(process.env.REDIS_URL || 'redis://localhost:6379');

export const apiRateLimiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 100, // Limit each IP or API key to 100 requests per window
  standardHeaders: true, // Return X-RateLimit-* headers in response
  legacyHeaders: false,
  store: new RedisStore({
    sendCommand: (...args: string[]) => redisClient.call(args[0], ...args.slice(1)),
  }),
  message: {
    status: 429,
    message: 'Too many requests from this client. Please try again in 15 minutes.',
  },
});
```

---

## 6. Senior Interview Q&A Cheatsheet

### Q1: "What causes `Error: Can't set headers after they are sent to the client` in Express?"
> **Answer**: This error occurs when the server attempts to write HTTP headers (via `res.status()`, `res.setHeader()`, or `res.send()`) after the response has already been committed and the socket header buffer has flushed to the network.
> **Common Causes**:
> 1. Forgetting to `return` after sending a response in an `if` branch (e.g. `if (!user) res.status(404).send(); res.json(user);`).
> 2. An async callback or promise resolution firing and attempting to respond after an earlier timeout or error middleware already returned a 500 error.
> 3. Calling `next(err)` after already invoking `res.send()`.

### Q2: "How do you achieve graceful shutdown in an Express application?"
> **Answer**: Listen for `SIGTERM` and `SIGINT` signals sent by Kubernetes or Docker:
> 1. Stop accepting new connections by calling `server.close()`.
> 2. Allow in-flight requests a grace period (e.g. 10-15 seconds) to complete.
> 3. Drain and disconnect database connection pools (`prisma.$disconnect()`, Redis `quit()`, or PG pool `end()`).
> 4. If connections do not drain within the deadline, force exit with `process.exit(1)`. Otherwise exit cleanly with `process.exit(0)`.
