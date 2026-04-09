---
name: monitoring-patterns
description: Prometheus metrics setup, Grafana dashboard configuration, PromQL reference queries, and NestJS instrumentation interceptors for gRPC and REST services.
origin: EWE
---

# Monitoring Patterns

Prometheus + Grafana instrumentation for NestJS services.

## When to Activate

- Setting up Prometheus metrics in a NestJS service
- Writing PromQL queries for a Grafana dashboard
- Adding metrics interceptors to gRPC services or HTTP middleware to REST services
- Configuring alert rules for latency or error rate thresholds

---

## Prometheus Setup

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
// metrics.service.ts
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

---

## Key PromQL Queries

```promql
# Request rate per method
rate(grpc_requests_total[1m])

# P99 latency per method
histogram_quantile(0.99, rate(grpc_request_duration_seconds_bucket[5m]))

# Error rate per method
sum(rate(grpc_requests_total{status!="OK"}[1m])) by (method)
  / sum(rate(grpc_requests_total[1m])) by (method)

# Kafka consumer lag (requires kafka-exporter)
kafka_consumer_group_lag{topic="market.events"}

# DB active connections (MySQL — requires mysqld_exporter)
mysql_global_status_threads_connected
```
