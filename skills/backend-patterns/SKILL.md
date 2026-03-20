---
name: backend-patterns
description: Backend architecture patterns, gRPC service design, API design, database optimization, message queues, cloud deployment, monitoring, and server-side best practices for Node.js, Nest.js, Express, and Next.js API routes.
origin: ECC
---

# Backend Development Patterns

Backend architecture patterns and best practices for scalable server-side applications.

## Tech Stack Preference

| Concern        | Primary              | Fallback            |
|----------------|----------------------|---------------------|
| Framework      | NestJS               | Express → Koa       |
| Validation     | Zod                  | —                   |
| ORM            | Drizzle              | —                   |
| Message Queue  | Kafka                | RabbitMQ → ActiveMQ |
| Cloud          | GCP                  | AWS                 |
| Monitoring     | Prometheus + Grafana | —                   |

## When to Activate

- Designing gRPC services or proto files
- Designing REST or GraphQL API endpoints
- Implementing repository, service, or controller layers
- Optimizing database queries (N+1, indexing, connection pooling) with Drizzle ORM
- Adding caching (Redis, in-memory, HTTP cache headers)
- Setting up background jobs or async processing with Kafka / RabbitMQ / ActiveMQ
- Structuring error handling and validation for APIs (Zod)
- Building interceptors for auth, logging, rate limiting
- Deploying services to GCP (Cloud Run / GKE) or AWS (ECS / EKS)
- Instrumenting services with Prometheus metrics and Grafana dashboards
- Building middleware (auth, logging, rate limiting)

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

```typescript
interface MarketRepository {
  findAll(filters?: MarketFilters): Promise<Market[]>
  findById(id: string): Promise<Market | null>
  create(data: CreateMarketDto): Promise<Market>
  update(id: string, data: UpdateMarketDto): Promise<Market>
  delete(id: string): Promise<void>
}

class SupabaseMarketRepository implements MarketRepository {
  async findAll(filters?: MarketFilters): Promise<Market[]> {
    let query = supabase.from('markets').select('*')

    if (filters?.status) {
      query = query.eq('status', filters.status)
    }

    if (filters?.limit) {
      query = query.limit(filters.limit)
    }

    const { data, error } = await query

    if (error) throw new Error(error.message)
    return data
  }

  // Other methods...
}
```

### Service Layer Pattern

```typescript
// Error codes should be numeric, but named via const/enum
export enum MarketServiceErrorCode {
  INVALID_TASK_ID = 30001,
  MARKET_INVALID_QUERY = 30002,
  MARKET_NOT_FOUND = 30003,
}

// Controller maps errorCode -> protocol-specific HTTP errors
// (and keeps response format as { data } / { error: { code, message } }).
class MarketService {
  constructor(private marketRepo: MarketRepository) {}

  async searchMarkets(query: string, limit = 10): Promise<Market[]> {
    if (!query.trim()) {
      // Service throws only an Error whose message carries the errorCode
      throw new Error(String(MarketServiceErrorCode.MARKET_INVALID_QUERY))
    }

    const embedding = await generateEmbedding(query)
    const results = await this.vectorSearch(embedding, limit)

    if (!results.length) {
      throw new Error(String(MarketServiceErrorCode.MARKET_NOT_FOUND))
    }

    const markets = await this.marketRepo.findByIds(results.map((r) => r.id))
    return markets.sort((a, b) => {
      const scoreA = results.find((r) => r.id === a.id)?.score ?? 0
      const scoreB = results.find((r) => r.id === b.id)?.score ?? 0
      return scoreA - scoreB
    })
  }
}

// Example (NestJS):
// @Get('/markets/search')
async function controllerSearch(query: SearchMarketsDto, marketService: MarketService) {
  try {
    const markets = await marketService.searchMarkets(query.q, query.limit)
    return { data: markets }
  } catch (e) {
    const errorCode = Number((e as Error)?.message)

    switch (errorCode) {
      case MarketServiceErrorCode.MARKET_INVALID_QUERY:
        throw new BadRequestException({
          error: { code: errorCode, message: 'Query must not be empty' },
        })

      case MarketServiceErrorCode.MARKET_NOT_FOUND:
        throw new NotFoundException({
          error: { code: errorCode, message: 'No markets found' },
        })

      default:
        throw new InternalServerErrorException({
          error: { code: 50000, message: 'Unexpected error' },
        })
    }
  }
}
```

### Middleware Pattern

```typescript
export function withAuth(handler: NextApiHandler): NextApiHandler {
  return async (req, res) => {
    const token = req.headers.authorization?.replace('Bearer ', '')
    if (!token) return res.status(401).json({ error: 'Unauthorized' })

    try {
      req.user = await verifyToken(token)
      return handler(req, res)
    } catch {
      return res.status(401).json({ error: 'Invalid token' })
    }
  }
}
```

---

## gRPC — Proto File Design

```proto
// market.proto
syntax = "proto3";
package market.v1;

import "google/protobuf/timestamp.proto";
import "google/protobuf/wrappers.proto";

// ✅ PascalCase messages, SCREAMING_SNAKE enums
// ✅ Version the package — never break a published package
// ✅ Use google.protobuf.Timestamp for dates, never plain strings
// ✅ Use oneof for sum types instead of nullable fields
// ✅ Reserve deprecated field numbers to prevent reuse

enum MarketStatus {
  MARKET_STATUS_UNSPECIFIED = 0;
  MARKET_STATUS_ACTIVE      = 1;
  MARKET_STATUS_CLOSED      = 2;
  MARKET_STATUS_RESOLVED    = 3;
}

message Market {
  string                    id         = 1;
  string                    title      = 2;
  MarketStatus              status     = 3;
  google.protobuf.Timestamp created_at = 4;
  google.protobuf.Timestamp updated_at = 5;
}

message GetMarketRequest  { string id = 1; }
message GetMarketResponse { Market market = 1; }

message ListMarketsRequest {
  MarketStatus status = 1;
  int32        page   = 2;
  int32        limit  = 3;
}
message ListMarketsResponse {
  repeated Market markets   = 1;
  int32           total     = 2;
  int32           next_page = 3;
}

// Optional field via wrapper type (proto3 has no null)
message UpdateMarketRequest {
  string                      id          = 1;
  google.protobuf.StringValue title       = 2;
  google.protobuf.Int32Value  max_traders = 3;
}

// Deprecated field reservation
message OldMessage {
  reserved 5, 10 to 12;
  reserved "deprecated_field";
}

service MarketService {
  rpc GetMarket    (GetMarketRequest)   returns (GetMarketResponse);
  rpc ListMarkets  (ListMarketsRequest) returns (ListMarketsResponse);
  rpc WatchMarkets (ListMarketsRequest) returns (stream Market);  // server-stream
}
```

## gRPC — NestJS Setup

### Module Configuration

```typescript
// app.module.ts
ClientsModule.register([{
  name: 'MARKET_SERVICE',
  transport: Transport.GRPC,
  options: {
    url:       process.env.MARKET_SERVICE_URL ?? 'localhost:50051',
    package:   'market.v1',
    protoPath: join(__dirname, '../proto/market.proto'),
    channelOptions: {
      'grpc.keepalive_time_ms':          30_000,
      'grpc.keepalive_timeout_ms':       10_000,
      'grpc.max_receive_message_length': 10 * 1024 * 1024,
    },
  },
}])
```

### Server Controller

```typescript
// market.controller.ts
@Controller()
export class MarketController {
  constructor(private readonly marketService: MarketService) {}

  @GrpcMethod('MarketService', 'GetMarket')
  async getMarket(data: unknown) {
    const req = GetMarketRequestSchema.parse(data)        // Zod at boundary
    return this.marketService.getMarket(req.id)
  }

  @GrpcMethod('MarketService', 'ListMarkets')
  async listMarkets(data: unknown) {
    const req = ListMarketsRequestSchema.parse(data)
    return this.marketService.listMarkets(req)
  }

  @GrpcStreamMethod('MarketService', 'WatchMarkets')
  watchMarkets(data$: Observable<unknown>): Observable<Market> {
    const subject = new Subject<Market>()
    data$.subscribe({
      next:  (data) => {
        const req = ListMarketsRequestSchema.parse(data)
        this.marketService.streamMarkets(req, subject)
          .catch((err) => subject.error(err))
      },
      error: (err) => subject.error(err),
    })
    return subject.asObservable()
  }
}
```

### Client Consumer

```typescript
// market.client.ts
@Injectable()
export class MarketClient implements OnModuleInit {
  private grpc!: MarketGrpcService

  constructor(@Inject('MARKET_SERVICE') private client: ClientGrpc) {}

  onModuleInit() {
    this.grpc = this.client.getService<MarketGrpcService>('MarketService')
  }

  getMarket(id: string) {
    return firstValueFrom(this.grpc.getMarket({ id }))
  }

  listMarkets(req: ListMarketsRequest) {
    return firstValueFrom(this.grpc.listMarkets(req))
  }
}
```

## gRPC — Zod Validation & Exception Mapping

```typescript
// market.schema.ts
export const MarketStatusSchema = z.enum([
  'MARKET_STATUS_UNSPECIFIED',
  'MARKET_STATUS_ACTIVE',
  'MARKET_STATUS_CLOSED',
  'MARKET_STATUS_RESOLVED',
])

export const GetMarketRequestSchema = z.object({
  id: z.string().uuid(),
})

export const ListMarketsRequestSchema = z.object({
  status: MarketStatusSchema.default('MARKET_STATUS_UNSPECIFIED'),
  page:   z.number().int().min(1).default(1),
  limit:  z.number().int().min(1).max(100).default(20),
})

export const CreateMarketRequestSchema = z.object({
  title:      z.string().min(3).max(255),
  maxTraders: z.number().int().positive().optional(),
  closesAt:   z.string().datetime().optional(),
})

export type TGetMarketRequest    = z.infer<typeof GetMarketRequestSchema>
export type TListMarketsRequest  = z.infer<typeof ListMarketsRequestSchema>
export type TCreateMarketRequest = z.infer<typeof CreateMarketRequestSchema>
```

```typescript
// grpc-exception.filter.ts
@Catch()
export class GrpcExceptionFilter implements RpcExceptionFilter {
  catch(exception: unknown): Observable<never> {
    if (exception instanceof ZodError) {
      return throwError(() => ({
        code:    GrpcStatus.INVALID_ARGUMENT,
        message: 'Validation failed',
        details: exception.errors
          .map((e) => `${e.path.join('.')}: ${e.message}`)
          .join('; '),
      }))
    }

    if (exception instanceof RpcException) {
      return throwError(() => exception.getError())
    }

    return throwError(() => ({
      code:    GrpcStatus.INTERNAL,
      message: 'Internal server error',
    }))
  }
}

// gRPC status → HTTP status (for REST gateway)
export const grpcToHttpStatus: Record<number, number> = {
  [GrpcStatus.OK]:                 200,
  [GrpcStatus.INVALID_ARGUMENT]:   400,
  [GrpcStatus.UNAUTHENTICATED]:    401,
  [GrpcStatus.PERMISSION_DENIED]:  403,
  [GrpcStatus.NOT_FOUND]:          404,
  [GrpcStatus.ALREADY_EXISTS]:     409,
  [GrpcStatus.RESOURCE_EXHAUSTED]: 429,
  [GrpcStatus.UNAVAILABLE]:        503,
  [GrpcStatus.INTERNAL]:           500,
}
```

## gRPC — Interceptors

```typescript
// auth.interceptor.ts
@Injectable()
export class GrpcAuthInterceptor implements NestInterceptor {
  intercept(ctx: ExecutionContext, next: CallHandler): Observable<unknown> {
    const metadata = ctx.switchToRpc().getContext()
    const token    = metadata.get('authorization')?.[0]
      ?.toString()
      ?.replace('Bearer ', '')

    if (!token) {
      throw new RpcException({ code: GrpcStatus.UNAUTHENTICATED, message: 'Missing token' })
    }

    try {
      const user = verifyToken(token)
      metadata.set('user-id', user.id)
    } catch {
      throw new RpcException({ code: GrpcStatus.UNAUTHENTICATED, message: 'Invalid token' })
    }

    return next.handle()
  }
}
```

```typescript
// logging.interceptor.ts
@Injectable()
export class GrpcLoggingInterceptor implements NestInterceptor {
  private readonly logger = new Logger('gRPC')

  intercept(ctx: ExecutionContext, next: CallHandler): Observable<unknown> {
    const method = `${ctx.getClass().name}.${ctx.getHandler().name}`
    const start  = Date.now()

    return next.handle().pipe(
      tap({
        next:  ()  => this.logger.log({ method, duration: Date.now() - start, status: 'OK' }),
        error: (e) => this.logger.error({ method, duration: Date.now() - start, code: e?.code }),
      }),
    )
  }
}
```

---

## Database Patterns — Drizzle ORM

### Schema Definition

```typescript
// db/schema.ts
import { pgTable, uuid, varchar, pgEnum, timestamp, integer } from 'drizzle-orm/pg-core'

export const marketStatusEnum = pgEnum('market_status', ['active', 'closed', 'resolved'])

export const markets = pgTable('markets', {
  id:         uuid('id').defaultRandom().primaryKey(),
  title:      varchar('title', { length: 255 }).notNull(),
  status:     marketStatusEnum('status').notNull().default('active'),
  maxTraders: integer('max_traders'),
  createdAt:  timestamp('created_at', { withTimezone: true }).defaultNow().notNull(),
  updatedAt:  timestamp('updated_at', { withTimezone: true }).defaultNow().notNull(),
})

export type TMarket    = typeof markets.$inferSelect
export type TNewMarket = typeof markets.$inferInsert
```

### Repository Pattern

```typescript
// market.repository.ts
@Injectable()
export class MarketRepository {
  constructor(@InjectDatabase() private db: NodePgDatabase) {}

  async findById(id: string) {
    const [market] = await this.db
      .select()
      .from(markets)
      .where(eq(markets.id, id))
      .limit(1)
    return market ?? null
  }

  async findMany(req: TListMarketsRequest) {
    const offset     = (req.page - 1) * req.limit
    const conditions = []

    if (req.status !== 'MARKET_STATUS_UNSPECIFIED') {
      const dbStatus = req.status.replace('MARKET_STATUS_', '').toLowerCase()
      conditions.push(eq(markets.status, dbStatus as 'active' | 'closed' | 'resolved'))
    }

    const where = conditions.length ? and(...conditions) : undefined

    const [rows, [{ count }]] = await Promise.all([
      this.db.select().from(markets).where(where).limit(req.limit).offset(offset),
      this.db.select({ count: sql<number>`count(*)::int` }).from(markets).where(where),
    ])

    return { rows, total: count }
  }

  async create(data: TNewMarket) {
    const [market] = await this.db.insert(markets).values(data).returning()
    return market
  }

  async update(id: string, patch: Partial<TNewMarket>) {
    const [market] = await this.db
      .update(markets)
      .set({ ...patch, updatedAt: new Date() })
      .where(eq(markets.id, id))
      .returning()
    return market ?? null
  }
}
```

### DB Module (Connection Pool)

```typescript
// db.module.ts
@Global()
@Module({
  providers: [{
    provide: 'DATABASE',
    useFactory: () =>
      drizzle(new Pool({
        connectionString:        process.env.DATABASE_URL,
        max:                     20,
        idleTimeoutMillis:       30_000,
        connectionTimeoutMillis: 5_000,
      })),
  }],
  exports: ['DATABASE'],
})
export class DbModule {}
```

### N+1 Query Prevention

```typescript
// ❌ BAD: N+1
const markets = await getMarkets()
for (const market of markets) {
  market.creator = await getUser(market.creator_id)
}

// ✅ GOOD: batch fetch
const markets    = await getMarkets()
const creatorIds = markets.map((m) => m.creator_id)
const creators   = await this.db
  .select()
  .from(users)
  .where(inArray(users.id, creatorIds))

const creatorMap = new Map(creators.map((c) => [c.id, c]))
markets.forEach((m) => { m.creator = creatorMap.get(m.creator_id) })
```

---

## Message Queue Patterns

### Kafka (Primary)

```typescript
// kafka.module.ts
ClientsModule.register([{
  name: 'KAFKA_CLIENT',
  transport: Transport.KAFKA,
  options: {
    client: {
      clientId: 'market-service',
      brokers:  (process.env.KAFKA_BROKERS ?? 'localhost:9092').split(','),
      ssl:      process.env.NODE_ENV === 'production',
      sasl: process.env.KAFKA_SASL_USERNAME
        ? {
            mechanism: 'scram-sha-512',
            username:  process.env.KAFKA_SASL_USERNAME,
            password:  process.env.KAFKA_SASL_PASSWORD!,
          }
        : undefined,
    },
    consumer: { groupId: 'market-service-group' },
    producer: { createPartitioner: Partitioners.LegacyPartitioner },
  },
}])
```

```typescript
// market-events.producer.ts
export const MarketCreatedEventSchema = z.object({
  eventType: z.literal('market.created'),
  marketId:  z.string().uuid(),
  title:     z.string(),
  createdAt: z.string().datetime(),
})

export type TMarketCreatedEvent = z.infer<typeof MarketCreatedEventSchema>

@Injectable()
export class MarketEventsProducer {
  constructor(@Inject('KAFKA_CLIENT') private kafka: ClientKafka) {}

  async emitMarketCreated(event: TMarketCreatedEvent) {
    const validated = MarketCreatedEventSchema.parse(event)
    await this.kafka.emit('market.events', {
      key:   validated.marketId,        // partition by market ID
      value: JSON.stringify(validated),
      headers: { 'content-type': 'application/json' },
    }).toPromise()
  }
}
```

```typescript
// market-events.consumer.ts
@Controller()
export class MarketEventsConsumer {
  @MessagePattern('market.events')
  async handleMarketEvent(
    @Payload() message: unknown,
    @Ctx()     ctx: KafkaContext,
  ) {
    const heartbeat = ctx.getHeartbeat()
    const event     = MarketCreatedEventSchema.safeParse(
      typeof message === 'string' ? JSON.parse(message) : message,
    )

    if (!event.success) {
      // Log and skip — never throw (Kafka will retry infinitely)
      console.error('Invalid event', event.error.flatten())
      return
    }

    await heartbeat()
    // Process event...
  }
}
```

### RabbitMQ (Fallback)

```typescript
ClientsModule.register([{
  name: 'RABBITMQ_CLIENT',
  transport: Transport.RMQ,
  options: {
    urls:         [process.env.RABBITMQ_URL ?? 'amqp://localhost:5672'],
    queue:        'market_events',
    queueOptions: {
      durable: true,
      arguments: {
        'x-dead-letter-exchange': 'market_events.dlx',  // DLQ
        'x-message-ttl':          3_600_000,             // 1-hour TTL
      },
    },
    noAck:        false,    // manual ack — at-least-once delivery
    prefetchCount: 10,      // backpressure control
  },
}])
```

### ActiveMQ (STOMP)

```typescript
// activemq.service.ts
@Injectable()
export class ActiveMQService implements OnModuleInit, OnModuleDestroy {
  private channel!: Stompit.Channel

  onModuleInit() {
    const servers = [{ host: process.env.ACTIVEMQ_HOST ?? 'localhost', port: 61613, ssl: false }]
    this.channel  = new Stompit.Channel(new Stompit.ConnectFailover(servers))
  }

  async publish(destination: string, body: object) {
    return new Promise<void>((resolve, reject) => {
      this.channel.send(
        { destination, 'content-type': 'application/json' },
        JSON.stringify(body),
        (err) => (err ? reject(err) : resolve()),
      )
    })
  }

  onModuleDestroy() { this.channel.close() }
}
```

### Queue Selection

| Scenario                             | Use         |
|--------------------------------------|-------------|
| High-throughput event streaming      | Kafka       |
| Task queues / RPC-style messaging    | RabbitMQ    |
| Legacy integration / JMS ecosystem   | ActiveMQ    |
| All new greenfield services          | Kafka       |

---

## Caching Strategies

### Redis Cache-Aside

```typescript
class CachedMarketRepository implements MarketRepository {
  constructor(
    private baseRepo: MarketRepository,
    private redis: RedisClient,
  ) {}

  async findById(id: string): Promise<TMarket | null> {
    const cached = await this.redis.get(`market:${id}`)
    if (cached) return JSON.parse(cached)

    const market = await this.baseRepo.findById(id)
    if (market) await this.redis.setex(`market:${id}`, 300, JSON.stringify(market))

    return market
  }

  async invalidate(id: string) {
    await this.redis.del(`market:${id}`)
  }
}
```

---

## Error Handling Patterns

### Centralized Error Handler

```typescript
class ApiError extends Error {
  constructor(
    public statusCode: number,
    public message: string,
    public isOperational = true,
  ) {
    super(message)
    Object.setPrototypeOf(this, ApiError.prototype)
  }
}

export function errorHandler(error: unknown): Response {
  if (error instanceof ApiError) {
    return NextResponse.json({ success: false, error: error.message }, { status: error.statusCode })
  }

  if (error instanceof z.ZodError) {
    return NextResponse.json(
      { success: false, error: 'Validation failed', details: error.errors },
      { status: 400 },
    )
  }

  console.error('Unexpected error:', error)
  return NextResponse.json({ success: false, error: 'Internal server error' }, { status: 500 })
}
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
        await new Promise((r) => setTimeout(r, Math.pow(2, i) * 1000))
      }
    }
  }

  throw lastError
}
```

---

## Authentication & Authorization

### JWT Validation

```typescript
export function verifyToken(token: string): IJWTPayload {
  try {
    return jwt.verify(token, process.env.JWT_SECRET!) as IJWTPayload
  } catch {
    throw new ApiError(401, 'Invalid token')
  }
}

export async function requireAuth(request: Request) {
  const token = request.headers.get('authorization')?.replace('Bearer ', '')
  if (!token) throw new ApiError(401, 'Missing authorization token')
  return verifyToken(token)
}
```

### Role-Based Access Control

```typescript
type TPermission = 'read' | 'write' | 'delete' | 'admin'

const rolePermissions: Record<IUser['role'], TPermission[]> = {
  admin:     ['read', 'write', 'delete', 'admin'],
  moderator: ['read', 'write', 'delete'],
  user:      ['read', 'write'],
}

export function requirePermission(permission: TPermission) {
  return (handler: (req: Request, user: IUser) => Promise<Response>) =>
    async (req: Request) => {
      const user = await requireAuth(req)
      if (!rolePermissions[user.role].includes(permission)) {
        throw new ApiError(403, 'Insufficient permissions')
      }
      return handler(req, user)
    }
}
```

---

## Monitoring — Prometheus + Grafana

### Metrics Setup

```typescript
// metrics.module.ts
@Module({
  imports: [PrometheusModule.register({
    path:           '/metrics',
    defaultMetrics: { enabled: true },
  })],
})
export class MetricsModule {}
```

```typescript
// grpc-metrics.service.ts
export const grpcRequestsTotal = makeCounterProvider({
  name:       'grpc_requests_total',
  help:       'Total gRPC requests',
  labelNames: ['method', 'status'],
})

export const grpcRequestDuration = makeHistogramProvider({
  name:       'grpc_request_duration_seconds',
  help:       'gRPC request duration in seconds',
  labelNames: ['method'],
  buckets:    [0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5],
})

export const mqMessagesProduced = makeCounterProvider({
  name:       'mq_messages_produced_total',
  help:       'Messages published to broker',
  labelNames: ['topic', 'broker'],
})

export const mqMessagesConsumed = makeCounterProvider({
  name:       'mq_messages_consumed_total',
  help:       'Messages consumed from broker',
  labelNames: ['topic', 'broker', 'status'],
})
```

```typescript
// metrics.interceptor.ts
@Injectable()
export class GrpcMetricsInterceptor implements NestInterceptor {
  constructor(private metrics: GrpcMetricsService) {}

  intercept(ctx: ExecutionContext, next: CallHandler): Observable<unknown> {
    const method = `${ctx.getClass().name}.${ctx.getHandler().name}`
    const start  = Date.now()

    return next.handle().pipe(
      tap({
        next:  ()  => this.metrics.recordRequest(method, 'OK',    Date.now() - start),
        error: (e) => this.metrics.recordRequest(method, String(e?.code ?? 'ERROR'), Date.now() - start),
      }),
    )
  }
}
```

### Key PromQL Queries (Grafana)

```promql
# Request rate per method
rate(grpc_requests_total[1m])

# P99 latency per method
histogram_quantile(0.99, rate(grpc_request_duration_seconds_bucket[5m]))

# Error rate
sum(rate(grpc_requests_total{status!="OK"}[1m])) by (method)
  / sum(rate(grpc_requests_total[1m])) by (method)

# Kafka consumer lag (requires kafka-exporter)
kafka_consumer_group_lag{topic="market.events"}

# DB connection pool
pg_stat_database_numbackends{datname="market_db"}
```

---

## Cloud Deployment

### GCP Cloud Run

```yaml
# gRPC requires HTTP/2 end-to-end; gen2 execution environment is mandatory
spec:
  template:
    metadata:
      annotations:
        run.googleapis.com/execution-environment: gen2
    spec:
      containers:
        - image: gcr.io/$PROJECT_ID/market-service:latest
          ports:
            - name: h2c            # HTTP/2 cleartext — Cloud Run terminates TLS
              containerPort: 50051
          livenessProbe:
            grpc: { port: 50051 }
          readinessProbe:
            grpc: { port: 50051 }
          resources:
            limits: { cpu: "1", memory: 512Mi }
```

```bash
gcloud run deploy market-service \
  --image gcr.io/$PROJECT_ID/market-service:latest \
  --port 50051 \
  --use-http2 \
  --region us-central1
```

### AWS ECS / Fargate

```json
{
  "containerDefinitions": [{
    "name": "market-service",
    "image": "$AWS_ACCOUNT.dkr.ecr.$REGION.amazonaws.com/market-service:latest",
    "portMappings": [{ "containerPort": 50051, "protocol": "tcp" }],
    "healthCheck": {
      "command": ["CMD", "grpc_health_probe", "-addr=:50051"],
      "interval": 30,
      "timeout": 10,
      "retries": 3
    }
  }],
  "requiresCompatibilities": ["FARGATE"],
  "networkMode": "awsvpc",
  "cpu": "512",
  "memory": "1024"
}
```

### Dockerfile (Multi-stage)

```dockerfile
FROM node:22-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --frozen-lockfile
COPY . .
RUN npm run build

FROM node:22-alpine AS runner
WORKDIR /app
RUN apk add --no-cache grpc-health-probe
COPY --from=builder /app/dist         ./dist
COPY --from=builder /app/node_modules ./node_modules
COPY proto ./proto
ENV NODE_ENV=production
EXPOSE 50051
CMD ["node", "dist/main.js"]
```

---

## Rate Limiting

```typescript
class RateLimiter {
  private requests = new Map<string, number[]>()

  check(identifier: string, maxRequests: number, windowMs: number): boolean {
    const now     = Date.now()
    const recent  = (this.requests.get(identifier) ?? []).filter((t) => now - t < windowMs)

    if (recent.length >= maxRequests) return false

    recent.push(now)
    this.requests.set(identifier, recent)
    return true
  }
}

const limiter = new RateLimiter()

export async function GET(request: Request) {
  const ip = request.headers.get('x-forwarded-for') ?? 'unknown'
  if (!limiter.check(ip, 100, 60_000)) {
    return NextResponse.json({ error: 'Rate limit exceeded' }, { status: 429 })
  }
  // ...
}
```

---

## Logging

```typescript
interface ILogContext {
  userId?:    string
  requestId?: string
  method?:    string
  path?:      string
  [key: string]: unknown
}

class Logger {
  private log(level: 'info' | 'warn' | 'error', message: string, context?: ILogContext) {
    console.log(JSON.stringify({ timestamp: new Date().toISOString(), level, message, ...context }))
  }

  info (message: string, context?: ILogContext)               { this.log('info',  message, context) }
  warn (message: string, context?: ILogContext)               { this.log('warn',  message, context) }
  error(message: string, err: Error, context?: ILogContext)   {
    this.log('error', message, { ...context, error: err.message, stack: err.stack })
  }
}
```

---

## New Service Checklist

- [ ] Create versioned `.proto` file (`package foo.v1`)
- [ ] Use `google.protobuf.Timestamp` for dates — never plain strings
- [ ] Define Zod schemas mirroring proto contracts; validate at NestJS handler boundary
- [ ] Apply `GrpcAuthInterceptor`, `GrpcLoggingInterceptor`, `GrpcMetricsInterceptor` globally
- [ ] Apply `GrpcExceptionFilter` globally
- [ ] Expose `/metrics` via `PrometheusModule`
- [ ] Register `grpc.health.v1` for liveness / readiness probes
- [ ] Dockerfile: multi-stage, install `grpc-health-probe`, expose correct port
- [ ] Cloud Run: `--use-http2` + `gen2`; AWS ALB: target-group protocol `HTTP/2`

**Remember**: gRPC is contract-first. Define `.proto` before any code, version your packages, and validate all inputs with Zod at the boundary — the proto compiler enforces types in transit, not inside NestJS handlers.