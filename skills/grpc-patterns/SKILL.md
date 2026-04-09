---
name: grpc-patterns
description: gRPC service design with proto files, NestJS gRPC controllers and clients, Zod validation at gRPC boundaries, exception mapping, and auth/logging interceptors.
origin: EWE
---

# gRPC Patterns

gRPC service design and NestJS implementation patterns for high-performance inter-service communication.

## When to Activate

- Designing a `.proto` file or gRPC service contract
- Implementing NestJS `@GrpcMethod` or `@GrpcStreamMethod` controller
- Setting up a gRPC client with `ClientsModule`
- Writing Zod schemas to validate gRPC request/response shapes
- Implementing `GrpcExceptionFilter` or mapping gRPC status codes to HTTP
- Adding auth or logging interceptors to a gRPC service

---

## Proto File Design

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

message UpdateMarketRequest {
  string                      id          = 1;
  google.protobuf.StringValue title       = 2;
  google.protobuf.Int32Value  max_traders = 3;
}

message OldMessage {
  reserved 5, 10 to 12;
  reserved "deprecated_field";
}

service MarketService {
  rpc GetMarket    (GetMarketRequest)   returns (GetMarketResponse);
  rpc ListMarkets  (ListMarketsRequest) returns (ListMarketsResponse);
  rpc WatchMarkets (ListMarketsRequest) returns (stream Market);
}
```

---

## NestJS Setup

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
@Controller()
export class MarketController {
  constructor(private readonly marketService: MarketService) {}

  @GrpcMethod('MarketService', 'GetMarket')
  async getMarket(data: unknown) {
    const req = GetMarketRequestSchema.parse(data)
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
        this.marketService.streamMarkets(req, subject).catch((err) => subject.error(err))
      },
      error: (err) => subject.error(err),
    })
    return subject.asObservable()
  }
}
```

### Client Consumer

```typescript
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

---

## Zod Validation & Exception Mapping

```typescript
// market.schema.ts
export const GetMarketRequestSchema = z.object({ id: z.string().uuid() })

export const MarketStatusSchema = z.enum([
  'MARKET_STATUS_UNSPECIFIED',
  'MARKET_STATUS_ACTIVE',
  'MARKET_STATUS_CLOSED',
  'MARKET_STATUS_RESOLVED',
])

export const ListMarketsRequestSchema = z.object({
  status: MarketStatusSchema.default('MARKET_STATUS_UNSPECIFIED'),
  page:   z.number().int().min(1).default(1),
  limit:  z.number().int().min(1).max(100).default(20),
})
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
        details: exception.errors.map((e) => `${e.path.join('.')}: ${e.message}`).join('; '),
      }))
    }
    if (exception instanceof RpcException) {
      return throwError(() => exception.getError())
    }
    return throwError(() => ({ code: GrpcStatus.INTERNAL, message: 'Internal server error' }))
  }
}

// gRPC status → HTTP status mapping
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

---

## Interceptors

```typescript
// auth.interceptor.ts
@Injectable()
export class GrpcAuthInterceptor implements NestInterceptor {
  intercept(ctx: ExecutionContext, next: CallHandler): Observable<unknown> {
    const metadata = ctx.switchToRpc().getContext()
    const token    = metadata.get('authorization')?.[0]?.toString()?.replace('Bearer ', '')
    if (!token) throw new RpcException({ code: GrpcStatus.UNAUTHENTICATED, message: 'Missing token' })
    try {
      const user = verifyToken(token)
      metadata.set('user-id', user.id)
    } catch {
      throw new RpcException({ code: GrpcStatus.UNAUTHENTICATED, message: 'Invalid token' })
    }
    return next.handle()
  }
}

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

## New gRPC Service Checklist

- [ ] Create versioned `.proto` file (`package foo.v1`)
- [ ] Use `google.protobuf.Timestamp` for dates — never plain strings
- [ ] Define Zod schemas mirroring proto contracts; validate at NestJS handler boundary
- [ ] Apply `GrpcAuthInterceptor`, `GrpcLoggingInterceptor` globally
- [ ] Apply `GrpcExceptionFilter` globally
- [ ] Register `grpc.health.v1` for liveness/readiness probes
- [ ] Add logger: `private readonly logger = new Logger(ClassName.name)`
- [ ] Register module in `AppModule` imports
- [ ] Dockerfile + cloud deployment — see **cloud-deployment** skill

**Remember**: gRPC is contract-first. Define `.proto` before any code, version your packages, and validate all inputs with Zod at the boundary.
