---
name: db-transaction-patterns
description: General approach for atomic database operations and TransactionRunner patterns in NestJS with Drizzle ORM — not tied to any specific project.
origin: EWE
---

# Database Transaction Patterns

Patterns for atomic multi-step database operations using a `TransactionRunner` wrapper with Drizzle ORM. Ensures all-or-nothing semantics when a service method touches multiple tables.

## When to Activate

- Implementing a service method that writes to multiple tables
- Adding `SELECT FOR UPDATE` for concurrent write prevention
- Reviewing repository methods for transaction support (`tx?` parameter)
- Setting up `TransactionRunner` in a new NestJS module

## Related Skills

- **backend-patterns** — Core NestJS patterns, Drizzle schema, repository pattern
- **exception-handling** — BusinessExceptionFilter, error code conventions

---

## TransactionRunner

A thin injectable wrapper around Drizzle's `.transaction()`. Keeps transaction orchestration out of repositories and in the service layer where business logic lives.

```typescript
// database/transactionRunner.service.ts
@Injectable()
export class TransactionRunner {
  constructor(@Inject(DRIZZLE) private readonly db: TDrizzleDb) {}

  run<T>(fn: (tx: TDrizzleTransaction) => Promise<T>): Promise<T> {
    return this.db.transaction(fn)
  }
}
```

Register `TransactionRunner` as a provider and export in `DrizzleModule` — see **backend-patterns** skill for the full module setup.

### Transaction Type

```typescript
// database/types.ts
import type { MySql2Database } from 'drizzle-orm/mysql2'

export type TDrizzleDb = MySql2Database<typeof schema>
export type TDrizzleTransaction = Parameters<Parameters<TDrizzleDb['transaction']>[0]>[0]
```

---

## Repository Convention: `tx?` Parameter

Every repository write method accepts an optional `tx` parameter. When provided, the operation participates in the caller's transaction. When omitted, it runs standalone.

```typescript
@Injectable()
export class OrderRepository {
  constructor(@Inject(DRIZZLE) private readonly db: MySql2Database<typeof schema>) {}

  async create(data: TNewOrder, tx?: TDrizzleTransaction) {
    const executor = tx ?? this.db
    await executor.insert(schema.orders).values(data)
  }

  async updateStatus(id: string, status: string, tx?: TDrizzleTransaction) {
    const executor = tx ?? this.db
    await executor
      .update(schema.orders)
      .set({ status, updatedAt: new Date() })
      .where(eq(schema.orders.id, id))
  }
}
```

Rules:

- `const executor = tx ?? this.db` — always at the top of write methods
- Read-only methods can also accept `tx?` when they need to see uncommitted writes within the same transaction
- Never start a new transaction inside an existing one — Drizzle does not support nested transactions

---

## SELECT FOR UPDATE

Prevents concurrent modification of the same row within a transaction. Use when a read-then-write sequence must be atomic.

```typescript
// Repository method
async findForUpdate(id: string, tx: TDrizzleTransaction) {
  const [row] = await tx
    .select()
    .from(schema.orders)
    .where(eq(schema.orders.id, id))
    .limit(1)
    .for('update')
  return row ?? null
}
```

Note: `tx` is required (not optional) — `FOR UPDATE` only makes sense inside a transaction.

---

## Service Usage — Atomic Multi-Step Operations

Inject `TransactionRunner` in the service. All repository calls within `run()` share the same transaction — if any step fails, everything rolls back.

```typescript
@Injectable()
export class OrderService {
  constructor(
    private readonly orderRepo: OrderRepository,
    private readonly inventoryRepo: InventoryRepository,
    private readonly ledgerRepo: LedgerRepository,
    private readonly transactionRunner: TransactionRunner,
  ) {}

  async placeOrder(userId: string, items: TOrderItem[]) {
    return this.transactionRunner.run(async tx => {
      // Lock inventory rows to prevent overselling
      for (const item of items) {
        const stock = await this.inventoryRepo.findForUpdate(item.productId, tx)
        if (!stock || stock.quantity < item.quantity) {
          throw new Error(OrderErrorCode.INSUFFICIENT_STOCK)
        }
      }

      // Deduct inventory
      for (const item of items) {
        await this.inventoryRepo.deduct(item.productId, item.quantity, tx)
      }

      // Create order
      const order = await this.orderRepo.create({ userId, items, status: 'confirmed' }, tx)

      // Record ledger entry
      await this.ledgerRepo.create({ orderId: order.id, amount: order.total, type: 'debit' }, tx)

      return order
    })
  }
}
```

---

## When to Use Transactions

| Scenario | Use Transaction? |
|----------|-----------------|
| Single INSERT/UPDATE | No — already atomic |
| Read-then-write (check + update) | Yes — prevents race conditions |
| Multi-table writes that must all succeed | Yes — ensures consistency |
| Read-only queries | No — unless reading uncommitted writes |
| Idempotent retryable operations | Maybe — consider if partial writes are acceptable |

---

## Checklist

- [ ] `TransactionRunner` registered in `DrizzleModule` and exported
- [ ] Every repository write method accepts `tx?: TDrizzleTransaction`
- [ ] Write methods use `const executor = tx ?? this.db`
- [ ] `SELECT FOR UPDATE` methods require `tx` (not optional)
- [ ] Service methods that touch multiple tables use `transactionRunner.run()`
- [ ] Error thrown inside `run()` triggers automatic rollback
- [ ] No nested transactions — never call `transactionRunner.run()` inside another `run()`
