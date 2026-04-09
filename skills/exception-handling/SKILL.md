---
name: exception-handling
description: Standardized NestJS exception filter structure, error code conventions, and BusinessExceptionFilter as a general pattern applicable across all backend projects.
origin: EWE
---

# Exception Handling Patterns

Standardized error code conventions and a global `BusinessExceptionFilter` for NestJS projects. Services throw domain error keys — no HTTP knowledge leaks into business logic.

## When to Activate

- Defining error codes for a new NestJS module
- Implementing or extending the global exception filter
- Structuring service-layer error handling (throw key, not HTTP status)
- Reviewing error response consistency across endpoints

## Related Skills

- **backend-patterns** — Core NestJS patterns, API design, repository, validation
- **db-transaction-patterns** — TransactionRunner, atomic operations

---

## Error Code Conventions

Each module defines a string enum for error keys and a record mapping each key to its HTTP status, numeric code, and user-facing message. Numeric codes are namespaced by module to avoid collisions.

```typescript
// modules/<name>/errors.ts
export enum OrderErrorCode {
  NOT_FOUND = 'ORDER_NOT_FOUND',
  ALREADY_CANCELLED = 'ORDER_ALREADY_CANCELLED',
  INSUFFICIENT_STOCK = 'INSUFFICIENT_STOCK',
}

export const ORDER_ERRORS: Record<string, IErrorEntry> = {
  [OrderErrorCode.NOT_FOUND]: {
    code: 30001,           // 30xxx = order module range
    status: HttpStatus.NOT_FOUND,
    message: 'Order not found.',
  },
  [OrderErrorCode.ALREADY_CANCELLED]: {
    code: 30002,
    status: HttpStatus.CONFLICT,
    message: 'This order has already been cancelled.',
  },
  [OrderErrorCode.INSUFFICIENT_STOCK]: {
    code: 30003,
    status: HttpStatus.UNPROCESSABLE_ENTITY,
    message: 'Insufficient stock for this order.',
  },
}
```

### IErrorEntry Type

```typescript
interface IErrorEntry {
  code: number       // Numeric error code (module-namespaced)
  status: HttpStatus // HTTP status to return
  message: string    // User-facing error message
}
```

### Merge All Module Maps

```typescript
// common/exceptions/errorMapping.ts
export const ERROR_MAP: Record<string, IErrorEntry> = {
  ...COMMON_ERRORS,   // 10xxx — shared (validation, auth, etc.)
  ...USER_ERRORS,     // 20xxx
  ...ORDER_ERRORS,    // 30xxx
  ...PAYMENT_ERRORS,  // 40xxx
  // 90xxx — system errors (reserved)
}
```

Suggested numeric ranges:

| Range | Module |
|-------|--------|
| 10xxx | Common / shared |
| 20xxx–80xxx | Domain modules (assign per project) |
| 90xxx | System / infrastructure |

---

## BusinessExceptionFilter (Global)

A single `@Catch()` filter intercepts all thrown errors, maps known keys to structured HTTP responses, and logs at the appropriate level.

```typescript
// common/filters/businessException.filter.ts
@Catch()
export class BusinessExceptionFilter implements ExceptionFilter {
  private readonly logger = new Logger(BusinessExceptionFilter.name)

  catch(exception: unknown, host: ArgumentsHost) {
    const ctx = host.switchToHttp()
    const res = ctx.getResponse<Response>()
    const req = ctx.getRequest<Request>()
    const key = (exception as Error)?.message
    const entry = ERROR_MAP[key]

    if (entry) {
      const { code, status, message } = entry
      if (status >= 500) this.logger.error(`${req.method} ${req.url} -> ${status}`, (exception as Error).stack)
      else this.logger.warn(`${req.method} ${req.url} -> ${status} [${key}]`)
      return res.status(status).json({ error: { code, message } })
    }

    // NestJS built-in exceptions (BadRequestException, etc.) — pass through
    if (exception instanceof HttpException) {
      const status = exception.getStatus()
      return res.status(status).json(exception.getResponse())
    }

    // Unmapped error -> 500
    this.logger.error('Unhandled exception', (exception as Error)?.stack)
    return res.status(500).json({ error: { code: 90000, message: 'Internal server error' } })
  }
}
```

### Register Globally

```typescript
// app.module.ts
providers: [{ provide: APP_FILTER, useClass: BusinessExceptionFilter }]
```

---

## Service Layer Usage

Services throw the enum key as an `Error` message. No try/catch needed in controllers — the global filter handles everything.

```typescript
// Service — throws enum key only, no HTTP knowledge
async cancelOrder(orderId: string) {
  const order = await this.orderRepo.findById(orderId)
  if (!order) throw new Error(OrderErrorCode.NOT_FOUND)
  if (order.status === 'cancelled') throw new Error(OrderErrorCode.ALREADY_CANCELLED)
  // ...
}
```

Controller stays clean — no try/catch:

```typescript
@Auth()
@Post(':id/cancel')
async cancelOrder(@Param('id') id: string): Promise<{ data: TOrder }> {
  return { data: await this.orderService.cancelOrder(id) }
}
```

---

## Checklist

- [ ] Define string enum + `IErrorEntry` records in `modules/<name>/errors.ts`
- [ ] Assign a numeric range (e.g. 30xxx) — check `errorMapping.ts` for conflicts
- [ ] Merge into `common/exceptions/errorMapping.ts`
- [ ] Services throw `new Error(EnumKey)` — never throw HTTP exceptions directly
- [ ] Controllers have no try/catch — `BusinessExceptionFilter` handles all errors
- [ ] `BusinessExceptionFilter` registered globally via `APP_FILTER`
- [ ] Verify error responses match the `{ error: { code, message } }` contract
