---
name: message-queue-patterns
description: Message queue patterns for Kafka (primary), RabbitMQ, and ActiveMQ — producer/consumer setup in NestJS, Zod event schema validation, DLQ configuration, and queue selection guidance.
origin: EWE
---

# Message Queue Patterns

Event-driven architecture patterns for Kafka, RabbitMQ, and ActiveMQ in NestJS.

## When to Activate

- Setting up a Kafka producer or consumer in NestJS
- Implementing RabbitMQ task queues or RPC-style messaging
- Integrating with an existing ActiveMQ / JMS ecosystem
- Designing event schemas or choosing between message brokers
- Configuring dead-letter queues (DLQ) or backpressure control

---

## Queue Selection

| Scenario                             | Use         |
|--------------------------------------|-------------|
| High-throughput event streaming      | Kafka       |
| Task queues / RPC-style messaging    | RabbitMQ    |
| Legacy integration / JMS ecosystem   | ActiveMQ    |
| All new greenfield services          | Kafka       |

---

## Kafka (Primary)

### Module Setup

```typescript
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

### Producer

```typescript
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
    this.kafka.emit('market.events', {
      key:     validated.marketId,        // partition by market ID
      value:   JSON.stringify(validated),
      headers: { 'content-type': 'application/json' },
    }).subscribe({
      error: (err) => this.logger.error('Kafka emit failed', err),
    })
  }
}
```

### Consumer

```typescript
@Controller()
export class MarketEventsConsumer {
  private readonly logger = new Logger(MarketEventsConsumer.name)

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
      this.logger.error('Invalid event schema', { errors: event.error.flatten() })
      return
    }

    await heartbeat()
    // process event...
  }
}
```

---

## RabbitMQ (Fallback)

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
    noAck:         false,   // manual ack — at-least-once delivery
    prefetchCount: 10,      // backpressure control
  },
}])
```

---

## ActiveMQ (STOMP)

```typescript
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
