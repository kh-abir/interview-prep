# 03 — TypeScript for Backend Architecture

> **Context**: Primary Backend Engineering Language. Focuses on advanced type-system constructs, type safety at API boundaries, compiler optimization, and Express request declaration merging.

---

## 1. Advanced Type System Constructs

### 1.1 `unknown` vs. `any` vs. `never`
- **`any`**: Disables the type checker completely. Propagates like a contagion across your codebase, destroying compile-time safety. Avoid in production backend code.
- **`unknown`**: Type-safe counterpart of `any`. Represents a value whose structure is completely unknown (e.g. raw JSON from external webhook or message broker). Forces the developer to perform runtime type narrowing before performing operations on it.
- **`never`**: Represents a value that can never occur. Used for unreachable code, exhausted pattern matches, and functions that always throw.

```typescript
// Exhaustive Type Checking with `never`
type PaymentStatus = 'PENDING' | 'AUTHORIZED' | 'SETTLED' | 'FAILED';

function handlePayment(status: PaymentStatus): void {
  switch (status) {
    case 'PENDING':
      return notifyCustomerWait();
    case 'AUTHORIZED':
      return captureAuthorizedCharge();
    case 'SETTLED':
      return creditMerchantBalance();
    case 'FAILED':
      return triggerFailureAlert();
    default: {
      // If a new status (e.g. 'REFUNDED') is added to PaymentStatus union,
      // TypeScript fails to compile HERE because status is not reducible to `never`!
      const _exhaustiveCheck: never = status;
      throw new Error(`Unhandled payment status: ${_exhaustiveCheck}`);
    }
  }
}
```

### 1.2 `type` vs. `interface` Decision Matrix

| Feature | `interface` | `type` Alias |
|---|---|---|
| **Declaration Merging** | Yes (merges multiple declarations across files) | No (duplicate names trigger compile error) |
| **Union / Intersection** | Cannot express arbitrary unions (`A \| B`) | **Yes** (unions, primitives, tuples) |
| **Mapped Types** | No | **Yes** (`[K in Keys]: Value`) |
| **Object Extension** | `interface B extends A` | `type B = A & { ... }` |
| **Best Practice Rule** | Use for **public API contracts** and **library declaration merging** | Use for **domain models**, **unions**, **utility transforms**, and **DTOs** |

---

## 2. Generics & Utility Types in Enterprise Services

### 2.1 Generic Repository Pattern
```typescript
export interface BaseEntity {
  id: string;
  createdAt: Date;
  updatedAt: Date;
}

export interface IRepository<T extends BaseEntity> {
  findById(id: string): Promise<T | null>;
  findMany(filter: Partial<T>): Promise<T[]>;
  create(data: Omit<T, 'id' | 'createdAt' | 'updatedAt'>): Promise<T>;
  update(id: string, patch: Partial<Omit<T, 'id'>>): Promise<T>;
  delete(id: string): Promise<boolean>;
}

export class PostgresRepository<T extends BaseEntity> implements IRepository<T> {
  constructor(
    private readonly dbPool: any,
    private readonly tableName: string
  ) {}

  async findById(id: string): Promise<T | null> {
    const result = await this.dbPool.query(
      `SELECT * FROM ${this.tableName} WHERE id = $1 LIMIT 1`,
      [id]
    );
    return result.rows[0] ?? null;
  }

  // Implementation of other methods...
}
```

### 2.2 Built-in Utility Types Matrix
- `Partial<T>`: Makes all properties optional. Ideal for update patch payloads.
- `Required<T>`: Makes all properties required.
- `Pick<T, K>`: Extracts a subset of properties from `T`.
- `Omit<T, K>`: Excludes a subset of properties from `T`.
- `Record<K, T>`: Constructs an object type whose property keys are `K` and values are `T`.
- `ReturnType<T>`: Obtains the return type of a function type.
- `Awaited<T>`: Unwraps a Promise type recursively (`Awaited<Promise<string>>` -> `string`).

---

## 3. Discriminated Unions & Custom Type Predicates

### 3.1 Tagged Discriminated Unions for Event Sourcing
```typescript
interface UserRegisteredEvent {
  type: 'USER_REGISTERED';
  payload: { userId: string; email: string; timestamp: number };
}

interface OrderPlacedEvent {
  type: 'ORDER_PLACED';
  payload: { orderId: string; totalAmountCents: number; currency: string };
}

interface PaymentFailedEvent {
  type: 'PAYMENT_FAILED';
  payload: { orderId: string; reason: string; code: number };
}

type DomainEvent = UserRegisteredEvent | OrderPlacedEvent | PaymentFailedEvent;

// Event Consumer with compile-time type narrowing based on the `type` tag
function processDomainEvent(event: DomainEvent): void {
  if (event.type === 'USER_REGISTERED') {
    // TypeScript automatically narrows event.payload to UserRegisteredEvent's payload!
    sendWelcomeEmail(event.payload.email);
  } else if (event.type === 'ORDER_PLACED') {
    reserveInventory(event.payload.orderId);
  } else {
    notifyFraudSystem(event.payload.orderId, event.payload.reason);
  }
}
```

### 3.2 Custom Type Guards (`arg is Type`)
```typescript
interface ApiSuccessResponse<T> {
  success: true;
  data: T;
}

interface ApiErrorResponse {
  success: false;
  error: { code: string; message: string };
}

type ApiResponse<T> = ApiSuccessResponse<T> | ApiErrorResponse;

// Custom Type Predicate function
function isSuccessResponse<T>(res: ApiResponse<T>): res is ApiSuccessResponse<T> {
  return res.success === true;
}

async function fetchUserData(userId: string) {
  const response: ApiResponse<{ id: string; name: string }> = await callRemoteApi(userId);

  if (isSuccessResponse(response)) {
    // Inside this block, TypeScript guarantees `response.data` exists!
    console.log(`User name: ${response.data.name}`);
  } else {
    console.error(`API Failure: ${response.error.message}`);
  }
}
```

---

## 4. TypeScript with Express: Declaration Merging & Strict Typing

### 4.1 Augmenting the Express Request Object
To inject authenticated user models, session tokens, or tenant IDs into Express without resorting to `(req as any).user`, use TypeScript **Declaration Merging**.

```typescript
// src/types/express/index.d.ts
import { AuthenticatedUser } from '../../models/User';

declare global {
  namespace Express {
    interface Request {
      user?: AuthenticatedUser;
      tenantId?: string;
      correlationId: string;
    }
  }
}
```

### 4.2 Strictly Typed Request Handlers
```typescript
import { Request, Response } from 'express';

// Typed route params, response body, request body, query params
interface UpdateUserParams {
  userId: string;
}

interface UpdateUserBody {
  displayName?: string;
  avatarUrl?: string;
}

interface UpdateUserQuery {
  notify?: 'true' | 'false';
}

interface UserResponse {
  success: boolean;
  user: { id: string; displayName: string };
}

export const updateUserHandler = async (
  req: Request<UpdateUserParams, UserResponse, UpdateUserBody, UpdateUserQuery>,
  res: Response<UserResponse>
) => {
  const { userId } = req.params;     // Typed as string
  const { displayName } = req.body;  // Typed as string | undefined
  const { notify } = req.query;      // Typed as 'true' | 'false' | undefined

  const updated = await userService.update(userId, { displayName });

  res.status(200).json({
    success: true,
    user: updated,
  });
};
```

---

## 5. Senior `tsconfig.json` Production Standard

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "lib": ["ES2022"],
    "outDir": "./dist",
    "rootDir": "./src",

    /* Strict Type-Checking Options */
    "strict": true,
    "noImplicitAny": true,
    "strictNullChecks": true,
    "strictFunctionTypes": true,
    "strictBindCallApply": true,
    "strictPropertyInitialization": true,
    "noImplicitThis": true,
    "alwaysStrict": true,

    /* Additional Production Checks */
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "exactOptionalPropertyTypes": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true,
    "noUncheckedIndexedAccess": true, // Treats array lookups arr[i] as T | undefined!

    /* Module Resolution & Interop */
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist", "**/*.spec.ts"]
}
```

---

## 6. Senior Interview Q&A Cheatsheet

### Q1: "Why should you use `as const` instead of TypeScript `enum`?"
> **Answer**:
> 1. Standard TypeScript numeric and string `enum` generates runtime boilerplate JavaScript IIFE code in the output bundle.
> 2. Numeric enums are not type-safe (they permit assignment of arbitrary numbers).
> 3. `const assertions` (`as const`) generate **zero runtime JavaScript overhead** while providing pure compile-time type safety:
>    ```typescript
>    export const UserRole = {
>      ADMIN: 'ADMIN',
>      MEMBER: 'MEMBER',
>    } as const;
>    export type UserRole = (typeof UserRole)[keyof typeof UserRole]; // 'ADMIN' | 'MEMBER'
>    ```

### Q2: "What does `noUncheckedIndexedAccess` in tsconfig do, and why is it critical for backend reliability?"
> **Answer**: By default, if you define `const items: string[] = ['a']`, accessing `items[10]` is typed by TypeScript as `string`, even though at runtime it evaluates to `undefined`. This causes catastrophic `TypeError: Cannot read properties of undefined` in production.
> When `noUncheckedIndexedAccess: true` is enabled, any indexed array or record access is typed as `T | undefined` (`string | undefined`), forcing the backend engineer to guard or check for existence before using the value.
