---
name: backend-patterns
description: Common NestJS backend patterns for EWE projects — Drizzle ORM (MySQL), Zod validation, repository pattern, Pino logging, and JWT auth guards.
origin: EWE
---

# Backend Development Patterns

Common NestJS patterns for EWE backend projects — REST APIs, Drizzle ORM (MySQL), Zod validation, repository pattern, and Pino logging.

## When to Activate

- Implementing a NestJS controller, service, or repository
- Defining a Drizzle schema (MySQL tables, indexes, relations)
- Writing Zod validation for request body or query params
- Setting up offset pagination
- Adding JWT auth guard or extracting user context

## Related Skills

- **exception-handling** — BusinessExceptionFilter, error code conventions, service-layer error patterns
- **db-transaction-patterns** — TransactionRunner, atomic multi-step operations, SELECT FOR UPDATE
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

### Service Layer & Error Handling

Services contain business logic and throw domain error keys. The global `BusinessExceptionFilter` maps keys to HTTP responses — no try/catch needed in controllers.

For error code conventions (string enum, ErrorEntry, module numeric ranges) and the BusinessExceptionFilter implementation, see **exception-handling** skill.

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
}
```

Rules:

- Always select specific columns — never `select()` with no arguments in production queries
- Every write method takes `tx?: TDrizzleTransaction`; use `const executor = tx ?? this.db`

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

### Transactions & Atomic Operations

For `TransactionRunner`, `SELECT FOR UPDATE`, and atomic multi-step operation patterns, see **db-transaction-patterns** skill.

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
const enriched = activities.map(a => ({ ...a, taskName: taskMap.get(a.taskId) }))
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

## Retry with Exponential Backoff

For transient failures (external API calls, network errors):

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
- [ ] Define error codes — see **exception-handling** skill for conventions
- [ ] Repository: every write method accepts `tx?: TDrizzleTransaction`
- [ ] For multi-step atomic operations, use `TransactionRunner` — see **db-transaction-patterns** skill
- [ ] Controller: use `@Auth()` + `safeParse` for body, `createZodDto()` for query params
- [ ] Add logger: `private readonly logger = new Logger(ClassName.name)`
- [ ] Run `bun run db:generate` then `bun run db:migrate` after schema changes
- [ ] Write unit tests with Vitest for service logic; integration tests for repository queries
