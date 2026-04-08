---
name: backend-patterns
description: Core NestJS patterns for reward-system — Drizzle ORM (MySQL), Zod validation, service layer error codes with BusinessExceptionFilter, TransactionRunner for atomic operations, Pino logging, and JWT auth guards.
origin: EWE
---

# Backend Development Patterns

Core NestJS patterns for reward-system — REST APIs, Drizzle ORM (MySQL), Zod validation, transactional operations, structured error handling, and Pino logging.

## When to Activate

- Implementing a NestJS controller, service, or repository
- Defining a Drizzle schema (MySQL tables, indexes, relations)
- Structuring error codes and exception handling
- Implementing multi-step atomic operations with `TransactionRunner`
- Writing Zod validation for request body or query params
- Setting up offset pagination
- Adding JWT auth guard or extracting user context

## Related Skills

- **caching-patterns** — Redis cache-aside, in-memory cache, HTTP cache headers
- **rate-limiting** — NestJS Throttler, in-memory sliding window, Redis-backed distributed limiting
- **grpc-patterns** — Proto file design, NestJS gRPC controllers, Zod/exception mapping
- **message-queue-patterns** — Kafka / RabbitMQ / ActiveMQ producer & consumer setup
- **monitoring-patterns** — Prometheus metrics, Grafana dashboards, PromQL queries
- **cloud-deployment** — GCP Cloud Run, AWS ECS/Fargate, multi-stage Dockerfile

---

## API Design Patterns

### RESTful API Structure

```typescript
// ✅ Resource-based URLs
GET    /api/markets                 # List resources
GET    /api/markets/:id             # Get single resource
POST   /api/markets                 # Create resource
PUT    /api/markets/:id             # Replace resource
PATCH  /api/markets/:id             # Update resource
DELETE /api/markets/:id             # Delete resource

// ✅ Query parameters for filtering, sorting, pagination
GET /api/markets?status=active&sort=volume&limit=20&offset=0
```

### Validation & Response Contract

- Any request with parameters must return `400 Bad Request` when validation fails.
- Successful responses default to `200 OK` (unless protocol semantics explicitly require another status, e.g. `201 Created`).
- Keep response format consistent:
  - Success: `{ data }`
  - Failure: `{ error: { code, message } }`

### Repository Pattern

One repository class per table, injected via `DRIZZLE` token. The contract is defined as an interface; all write methods accept `tx?: TDrizzleTransaction` to participate in transactions.

```typescript
interface IActivityRepository {
  findByUserId(userId: string, limit: number, offset: number): Promise<TActivity[]>
  countByUserId(userId: string): Promise<number>
  create(data: TNewActivity, tx?: TDrizzleTransaction): Promise<void>
}
```

See **Database Patterns** for the full Drizzle implementation, `TransactionRunner`, N+1 prevention, and pagination.

### Service Layer & Error Codes

Each module defines a string enum and an error map. Services throw `new Error(ErrorKey)`. The global `BusinessExceptionFilter` maps the key to the correct HTTP response — no try/catch needed in controllers.

```typescript
// modules/activity/errors.ts
export enum ActivityErrorCode {
  ALREADY_QUEUED = 'ALREADY_QUEUED',
  ALREADY_VERIFIED = 'ALREADY_VERIFIED'
}

export const ACTIVITY_ERRORS: Record<string, ErrorEntry> = {
  [ActivityErrorCode.ALREADY_QUEUED]: {
    code: 50001, // 50xxx = activity module range
    status: HttpStatus.CONFLICT,
    message: 'This transaction is already in the verification queue.'
  },
  [ActivityErrorCode.ALREADY_VERIFIED]: {
    code: 50002,
    status: HttpStatus.CONFLICT,
    message: 'This transaction has already been verified.'
  }
}
```

```typescript
// common/exceptions/errorMapping.ts — merge all module maps here
export const ERROR_MAP: Record<string, ErrorEntry> = {
  ...COMMON_ERRORS, // 10xxx
  ...PROFILE_ERRORS, // 20xxx
  ...SPIN_ERRORS, // 30xxx
  ...CHECKIN_ERRORS, // 40xxx
  ...ACTIVITY_ERRORS // 50xxx
  // 90xxx — system errors
}
```

```typescript
// Service — throws enum key only, no HTTP knowledge
async submitTransaction(userId: string, txHash: string) {
  const existing = await this.pendingTxRepo.findByHash(txHash)
  if (existing) throw new Error(ActivityErrorCode.ALREADY_QUEUED)
  // ...
}
```

Controller stays clean — no try/catch:

```typescript
@Auth()
@Post('transactions')
async saveTransaction(@Body() body: unknown): Promise<{ data: TPendingTransaction }> {
  const parsed = SaveTransactionSchema.safeParse(body)
  if (!parsed.success) throw new BadRequestException({ errors: parsed.error.flatten() })
  return { data: await this.activityService.save(parsed.data) }
}
```

### Auth Guard Pattern

Use a custom `@Auth()` decorator backed by a NestJS guard to protect routes and extract the JWT payload.

```typescript
// auth/auth.decorator.ts
export const Auth = () => UseGuards(JwtAuthGuard)

// auth/jwt.guard.ts
@Injectable()
export class JwtAuthGuard implements CanActivate {
  canActivate(ctx: ExecutionContext): boolean {
    const req = ctx.switchToHttp().getRequest()
    const token = req.headers.authorization?.replace('Bearer ', '')
    if (!token) throw new UnauthorizedException()
    req.user = verifyToken(token); // throws on invalid token
    return true
  }
}

// Controller usage
@Controller('activity')
export class ActivityController {
  @Auth()
  @Get()
  async getActivity(@Ctx() ctx: AuthPayload): Promise<{ data: TActivityResponse }> {
    return { data: await this.activityService.getActivity(ctx.userId!) }
  }
}
```

---

## Database Patterns — Drizzle ORM

### Schema Definition (MySQL)

```typescript
// database/schema/activities.schema.ts
import { mysqlTable, char, varchar, bigint, datetime, index, sql } from 'drizzle-orm/mysql-core'

export const activities = mysqlTable(
  'activities',
  {
    id: char('id', { length: 36 }).primaryKey(),
    userId: char('user_id', { length: 36 }).notNull(),
    address: varchar('address', { length: 128 }).notNull(),
    taskId: char('task_id', { length: 36 }).references(() => simpleTasks.id),
    stars: bigint('stars', { mode: 'number' }).notNull(),
    createdAt: datetime('created_at')
      .notNull()
      .default(sql`CURRENT_TIMESTAMP`)
  },
  table => [
    index('activities_user_id_idx').on(table.userId),
    index('activities_task_id_idx').on(table.taskId),
    index('activities_created_at_idx').on(table.createdAt),
  ]
)

export type TActivity = typeof activities.$inferSelect
export type TNewActivity = typeof activities.$inferInsert
```

Key points:

- Use `char` for UUID/fixed-length IDs, `varchar` for variable strings, `datetime` for timestamps
- Export `$inferSelect` / `$inferInsert` types — never define types manually
- All schema files exported from `database/schema/index.ts`

**Indexing rules — decide for every new table:**

- Index every column used in `WHERE`, `ORDER BY`, or `JOIN ON`
- Composite index: equality column first, then range/sort column (e.g. `userId + createdAt`)
- Add `unique()` for business-level uniqueness constraints (e.g. `txHash`)
- Skip indexing low-cardinality columns on write-heavy tables (e.g. a boolean `isActive`)

### Repository Implementation (Drizzle + MySQL)

```typescript
// modules/activity/activity.repository.ts
@Injectable()
export class ActivityRepository {
  constructor(@Inject(DRIZZLE) private readonly db: MySql2Database<typeof schema>) {}

  async findByUserId(userId: string, limit: number, offset: number) {
    return this.db
      .select({
        id: schema.activities.id,
        taskName: schema.simpleTasks.name,
        stars: schema.activities.stars,
        createdAt: schema.activities.createdAt
      })
      .from(schema.activities)
      .leftJoin(schema.simpleTasks, eq(schema.activities.taskId, schema.simpleTasks.id))
      .where(eq(schema.activities.userId, userId))
      .orderBy(desc(schema.activities.createdAt))
      .limit(limit)
      .offset(offset)
  }

  async countByUserId(userId: string): Promise<number> {
    const [{ count }] = await this.db
      .select({ count: sql<number>`count(*)` })
      .from(schema.activities)
      .where(eq(schema.activities.userId, userId))
    return count
  }

  async create(data: TNewActivity, tx?: TDrizzleTransaction) {
    const executor = tx ?? this.db
    await executor.insert(schema.activities).values(data)
  }

  // SELECT FOR UPDATE — prevents concurrent writes to the same row
  async findForUpdate(id: string, tx: TDrizzleTransaction) {
    const [row] = await tx.select().from(schema.activities).where(eq(schema.activities.id, id)).limit(1).for('update')
    return row ?? null
  }
}
```

Rules:

- Always select specific columns — never `select()` with no arguments in production queries
- Every write method takes `tx?: TDrizzleTransaction`; use `const executor = tx ?? this.db`
- Use `.for('update')` inside transactions to prevent concurrent modification

### DB Module (MySQL Connection Pool)

```typescript
// database/drizzle.module.ts
@Global()
@Module({
  providers: [
    {
      provide: DRIZZLE,
      useFactory: () =>
        drizzle(
          mysql.createPool({
            uri: process.env.DATABASE_URL,
            connectionLimit: 20,
            waitForConnections: true,
            idleTimeout: 30_000
          }),
          { schema, mode: 'default' }
        )
    },
    TransactionRunner
  ],
  exports: [DRIZZLE, TransactionRunner]
})
export class DrizzleModule {}
```

### TransactionRunner — Atomic Multi-Step Operations

Wrap multiple repository calls in a single database transaction. All repositories accept `tx?` to participate.

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

```typescript
// Usage in a service — all steps are atomic
const result = await this.transactionRunner.run(async tx => {
  const credits = await this.tasksRepo.findCredits(userId, { tx, forUpdate: true })
  if (credits <= 0) return null

  await this.tasksRepo.consumeCredit(userId, tx)
  await this.profileRepo.incrementStars(userId, amount, tx)
  await this.activityRepo.create({ userId, stars: amount }, tx)

  return this.profileRepo.findProfile(userId, { tx })
})
```

### N+1 Query Prevention

```typescript
// ❌ BAD: N+1 — one query per activity
const activities = await this.activityRepo.findByUserId(userId)
for (const activity of activities) {
  activity.taskName = await this.tasksRepo.findName(activity.taskId)
}

// ✅ GOOD: leftJoin in the repository (preferred — see Repository Implementation above)
// Alternative when JOIN isn't feasible — batch by IDs:
const taskIds = activities.map(a => a.taskId).filter(Boolean)
const tasks = await this.db.select({ id: schema.simpleTasks.id, name: schema.simpleTasks.name }).from(schema.simpleTasks).where(inArray(schema.simpleTasks.id, taskIds))

const taskMap = new Map(tasks.map(t => [t.id, t.name]))
activities.forEach(a => {
  a.taskName = taskMap.get(a.taskId)
})
```

### Offset Pagination

Standard pattern for list endpoints — return both `items` and `pagination` metadata.

```typescript
// Repository — parallel count + data fetch
async findMany(userId: string, page: number, limit: number) {
  const offset = (page - 1) * limit
  const [items, [{ total }]] = await Promise.all([
    this.db
      .select({ id: schema.activities.id, stars: schema.activities.stars, createdAt: schema.activities.createdAt })
      .from(schema.activities)
      .where(eq(schema.activities.userId, userId))
      .orderBy(desc(schema.activities.createdAt))
      .limit(limit)
      .offset(offset),
    this.db
      .select({ total: sql<number>`count(*)` })
      .from(schema.activities)
      .where(eq(schema.activities.userId, userId)),
  ])
  return { items, total }
}

// Response shape
return {
  data: {
    items,
    pagination: { page, limit, total, totalPages: Math.ceil(total / limit) },
  },
}
```

---

## Error Handling Patterns

### BusinessExceptionFilter (Global)

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
      if (status >= 500) this.logger.error(`${req.method} ${req.url} → ${status}`, (exception as Error).stack)
      else this.logger.warn(`${req.method} ${req.url} → ${status} [${key}]`)
      return res.status(status).json({ error: { code, message } })
    }

    // Unmapped error → 500
    this.logger.error('Unhandled exception', (exception as Error)?.stack)
    return res.status(500).json({ error: { code: 90000, message: 'Internal server error' } })
  }
}

// Register globally in app.module.ts
providers: [{ provide: APP_FILTER, useClass: BusinessExceptionFilter }]
```

### Retry with Exponential Backoff

```typescript
async function withRetry<T>(fn: () => Promise<T>, maxRetries = 3): Promise<T> {
  let lastError!: Error

  for (let i = 0; i < maxRetries; i++) {
    try {
      return await fn()
    } catch (err) {
      lastError = err as Error
      if (i < maxRetries - 1) {
        await new Promise(r => setTimeout(r, Math.pow(2, i) * 1000))
      }
    }
  }

  throw lastError
}
```

---

## Zod Validation Patterns

Three usage contexts with different Zod APIs:

```typescript
// 1. Request body → safeParse (structured validation errors in response)
@Post('transactions')
async saveTransaction(@Body() body: unknown) {
  const parsed = SaveTransactionSchema.safeParse(body)
  if (!parsed.success) {
    throw new BadRequestException({ message: 'Validation failed', errors: parsed.error.flatten() })
  }
  return { data: await this.service.save(parsed.data) }
}

// 2. Query params → createZodDto (NestJS pipe integration, auto-validates)
export class QueryActivityDto extends createZodDto(
  z.object({
    page:  z.coerce.number().int().min(1).default(1),
    limit: z.coerce.number().int().min(1).max(100).default(20),
  })
) {}

@Get()
async getActivity(@Query() query: QueryActivityDto) { ... }

// 3. Response shaping → parse (trust internal data; use for type inference only)
return ActivityResponseSchema.parse({ items, pagination })
```

Schema definition:

```typescript
// modules/activity/schemas/saveTransaction.schema.ts
export const SaveTransactionSchema = z.object({
  txHash: z.string().min(1),
  chainId: z.coerce.number().int(),
  taskType: z.enum(['swap', 'bridge', 'stake', 'dca']),
  address: z.string().min(1)
})
export type TSaveTransactionDto = z.infer<typeof SaveTransactionSchema>
```

---

## Logging

Pino via `nestjs-pino` — structured JSON in production, pretty-printed in development.

```typescript
// app.module.ts
LoggerModule.forRoot({
  pinoHttp: {
    transport: process.env.NODE_ENV !== 'production' ? { target: 'pino-pretty', options: { colorize: true } } : undefined,
    level: process.env.NODE_ENV !== 'production' ? 'debug' : 'info'
  }
})

// main.ts
app.useLogger(app.get(PinoLogger))
```

```typescript
// In any service or filter
private readonly logger = new Logger(ActivityService.name)

this.logger.log('Processing activity', { userId, taskId })
this.logger.warn('Low credits', { userId, credits })
this.logger.error('Failed to process', error.stack)
```

---

## New Module Checklist

- [ ] Create `modules/<name>/` with: controller, service, repository, module.ts, errors.ts, schemas/
- [ ] Register new module in `AppModule` imports
- [ ] Define Drizzle schema in `database/schema/<name>.schema.ts`; add indexes; export from `index.ts`
- [ ] Define string enum + `ErrorEntry` records in `errors.ts` — use the correct `xxxxx` numeric range
- [ ] Merge error map into `common/exceptions/errorMapping.ts`
- [ ] Repository: every write method accepts `tx?: TDrizzleTransaction`
- [ ] Service: inject `TransactionRunner` for any multi-step atomic operation
- [ ] Controller: use `@Auth()` + `safeParse` for body, `createZodDto()` for query params
- [ ] `BusinessExceptionFilter` is registered globally — no try/catch needed in controllers
- [ ] Add logger: `private readonly logger = new Logger(ClassName.name)`
- [ ] Run `bun run db:generate` then `bun run db:migrate` after schema changes
- [ ] Write unit tests with Vitest for service logic; integration tests for repository queries
