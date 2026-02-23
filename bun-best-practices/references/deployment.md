# Deployment Patterns for Bun.js

## Docker

### Production Dockerfile (Multi-stage with compile)

```dockerfile
FROM oven/bun:1 AS builder
WORKDIR /app

COPY package.json bun.lock ./
RUN bun install --frozen-lockfile --production

COPY . .
RUN bun build --compile --target=bun src/app.ts --outfile=server

FROM gcr.io/distroless/cc-debian12
COPY --from=builder /app/server /server
EXPOSE 3000
CMD ["/server"]
```

### Lightweight Dockerfile (without compile)

```dockerfile
FROM oven/bun:1-slim
WORKDIR /app

COPY package.json bun.lock ./
RUN bun install --frozen-lockfile --production

COPY src/ src/
EXPOSE 3000
USER bun
CMD ["bun", "src/app.ts"]
```

### Docker Compose (Dev)

```yaml
services:
  api:
    build: .
    ports: ["3000:3000"]
    volumes: ["./src:/app/src"]  # hot reload
    environment:
      - DATABASE_URL=postgres://user:pass@db:5432/app
      - REDIS_URL=redis://cache:6379
    depends_on: [db, cache]

  db:
    image: postgres:17-alpine
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
      POSTGRES_DB: app
    ports: ["5432:5432"]
    volumes: ["pgdata:/var/lib/postgresql/data"]

  cache:
    image: redis:7-alpine
    ports: ["6379:6379"]

volumes:
  pgdata:
```

---

## Serverless Deployment

### AWS Lambda (Container Image)

```dockerfile
FROM oven/bun:1 AS builder
WORKDIR /app
COPY package.json bun.lock ./
RUN bun install --frozen-lockfile
COPY . .
RUN bun build --compile --target=bun-linux-x64 src/handler.ts --outfile=handler

FROM public.ecr.aws/lambda/provided:al2023
COPY --from=builder /app/handler ${LAMBDA_TASK_ROOT}/handler
ENTRYPOINT ["./handler"]
```

Push to ECR and create Lambda from container image.

### Railway

```json
{
  "scripts": {
    "start": "bun src/app.ts"
  }
}
```

Railway auto-detects Bun from `bun.lock`. No Dockerfile needed.

### Fly.io

```toml
# fly.toml
app = "my-bun-app"

[build]
  dockerfile = "Dockerfile"

[http_service]
  internal_port = 3000
  force_https = true

[[vm]]
  size = "shared-cpu-1x"
  memory = "256mb"
```

---

## CI/CD (GitHub Actions)

```yaml
name: CI
on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: oven-sh/setup-bun@v2
        with:
          bun-version: latest
      - run: bun install --frozen-lockfile
      - run: bun test
      - run: bun run lint

  deploy:
    needs: test
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: oven-sh/setup-bun@v2
      - run: bun install --frozen-lockfile
      - run: bun build --compile src/app.ts --outfile=server
      # Deploy step: docker push, fly deploy, etc.
```

**CI speedup**: `bun install` is 20-40x faster than npm ci. Monorepo CI drops from 30 min → under 5 min.

---

## Health Check & Graceful Shutdown

```typescript
import { Elysia } from "elysia";

let isShuttingDown = false;

const app = new Elysia()
  .get("/health", () => {
    if (isShuttingDown) return new Response("Shutting down", { status: 503 });
    return { ok: true, uptime: process.uptime() };
  })
  .listen(3000);

process.on("SIGTERM", async () => {
  isShuttingDown = true;
  console.log("SIGTERM received. Draining connections...");
  await Bun.sleep(5000);
  process.exit(0);
});
```

---

## Observability

### Structured Logging

```typescript
function log(level: "info" | "warn" | "error", message: string, meta?: Record<string, unknown>) {
  console.log(JSON.stringify({
    timestamp: new Date().toISOString(),
    level,
    message,
    ...meta,
  }));
}
```

### OpenTelemetry Setup

```bash
bun add @opentelemetry/sdk-node @opentelemetry/auto-instrumentations-node
```

```typescript
import { NodeSDK } from "@opentelemetry/sdk-node";
import { getNodeAutoInstrumentations } from "@opentelemetry/auto-instrumentations-node";

const sdk = new NodeSDK({
  instrumentations: [getNodeAutoInstrumentations()],
  serviceName: "my-bun-api",
});
sdk.start();
```

### Prometheus Metrics Endpoint

```typescript
let requestCount = 0;

const app = new Elysia()
  .onBeforeHandle(() => { requestCount++; })
  .get("/metrics", () =>
    `# HELP http_requests_total Total HTTP requests\n# TYPE http_requests_total counter\nhttp_requests_total ${requestCount}`
  );
```

---

## Production Checklist

- [ ] Multi-stage Docker build with `oven/bun:1` base image
- [ ] `bun install --frozen-lockfile` in CI and Docker
- [ ] `bun.lock` committed (text format — do NOT use old `bun.lockb`)
- [ ] Health check endpoint at `/health`
- [ ] Graceful SIGTERM shutdown with connection draining
- [ ] CORS with explicit origin whitelist (never wildcard)
- [ ] Validation errors hidden in production (Elysia default)
- [ ] Structured JSON logging
- [ ] OpenTelemetry tracing configured
- [ ] Environment variables via `.env` (not hardcoded)
- [ ] TLS configured (native Bun or reverse proxy)
- [ ] Rate limiting at gateway level
- [ ] SQL params audited for `undefined` — use `value ?? null` (throws in v1.3+)
- [ ] Dependency audit: `bun pm audit`
