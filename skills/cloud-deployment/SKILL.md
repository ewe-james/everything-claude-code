---
name: cloud-deployment
description: Cloud deployment patterns for NestJS services — multi-stage Dockerfile, GCP Cloud Run, and AWS ECS/Fargate for both REST (HTTP) and gRPC (HTTP/2) services, with health check probe configuration.
origin: EWE
---

# Cloud Deployment Patterns

Deployment configurations for NestJS services on GCP and AWS.

## When to Activate

- Writing a Dockerfile for a NestJS service (REST or gRPC)
- Deploying to GCP Cloud Run
- Deploying to AWS ECS/Fargate
- Configuring liveness/readiness health check probes for containers

---

## REST / HTTP Service

### Dockerfile (Multi-stage)

```dockerfile
FROM oven/bun:1-alpine AS builder
WORKDIR /app
COPY package.json bun.lock ./
RUN bun install --frozen-lockfile
COPY . .
RUN bun run build

FROM oven/bun:1-alpine AS runner
WORKDIR /app
COPY --from=builder /app/dist         ./dist
COPY --from=builder /app/node_modules ./node_modules
ENV NODE_ENV=production
EXPOSE 3000
CMD ["bun", "run", "dist/main.js"]
```

### GCP Cloud Run (REST)

```yaml
spec:
  template:
    spec:
      containers:
        - image: gcr.io/$PROJECT_ID/reward-system:latest
          ports:
            - containerPort: 3000
          livenessProbe:
            httpGet: { path: /health, port: 3000 }
          readinessProbe:
            httpGet: { path: /health, port: 3000 }
          resources:
            limits: { cpu: "1", memory: 512Mi }
```

```bash
gcloud run deploy reward-system \
  --image gcr.io/$PROJECT_ID/reward-system:latest \
  --port 3000 \
  --region us-central1
```

### AWS ECS / Fargate (REST)

```json
{
  "containerDefinitions": [{
    "name": "reward-system",
    "image": "$AWS_ACCOUNT.dkr.ecr.$REGION.amazonaws.com/reward-system:latest",
    "portMappings": [{ "containerPort": 3000, "protocol": "tcp" }],
    "healthCheck": {
      "command": ["CMD-SHELL", "curl -f http://localhost:3000/health || exit 1"],
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

---

## gRPC Service

> gRPC requires HTTP/2 end-to-end. The `gen2` execution environment is mandatory on Cloud Run.

### Dockerfile (gRPC)

```dockerfile
FROM oven/bun:1-alpine AS builder
WORKDIR /app
COPY package.json bun.lock ./
RUN bun install --frozen-lockfile
COPY . .
RUN bun run build

FROM oven/bun:1-alpine AS runner
WORKDIR /app
RUN apk add --no-cache grpc-health-probe
COPY --from=builder /app/dist         ./dist
COPY --from=builder /app/node_modules ./node_modules
COPY proto ./proto
ENV NODE_ENV=production
EXPOSE 50051
CMD ["bun", "run", "dist/main.js"]
```

### GCP Cloud Run (gRPC)

```yaml
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

### AWS ECS / Fargate (gRPC)

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
