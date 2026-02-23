# Architecture Patterns for Bun.js

## Pattern 1: Clean Architecture with ElysiaJS (Recommended)

Domain-driven modules with Elysia plugin composition.

### Full Project Structure

```
project-root/
├─ src/
│   ├─ modules/
│   │   ├─ auth/
│   │   │   ├─ application/
│   │   │   │   ├─ login.usecase.ts
│   │   │   │   └─ register.usecase.ts
│   │   │   ├─ domain/
│   │   │   │   ├─ user.entity.ts
│   │   │   │   ├─ auth.repository.ts      (interface)
│   │   │   │   └─ auth.errors.ts
│   │   │   └─ infrastructure/
│   │   │       ├─ routes.ts               (Elysia plugin)
│   │   │       ├─ pg-auth.repository.ts   (implements interface)
│   │   │       └─ schemas.ts              (TypeBox validation)
│   │   ├─ users/
│   │   └─ payments/
│   ├─ shared/
│   │   └─ infrastructure/
│   │       ├─ database.ts                 (connection pool)
│   │       ├─ error-handler.ts            (global Elysia error handler)
│   │       └─ logger.ts
│   └─ app.ts                              (composition root)
├─ tests/
│   ├─ unit/
│   └─ integration/
├─ bunfig.toml
├─ package.json
└─ tsconfig.json
```

### Module Route Plugin Pattern

```typescript
// src/modules/auth/infrastructure/routes.ts
import { Elysia, t } from "elysia";
import { jwt } from "@elysiajs/jwt";
import { LoginUseCase } from "../application/login.usecase";

export const authPlugin = new Elysia({ prefix: "/auth" })
  .use(jwt({ name: "jwt", secret: process.env.JWT_SECRET! }))
  .post("/login", async ({ body, jwt }) => {
    const useCase = new LoginUseCase(/* inject repo */);
    const user = await useCase.execute(body.email, body.password);
    const token = await jwt.sign({ sub: user.id });
    return { token };
  }, {
    body: t.Object({
      email: t.String({ format: "email" }),
      password: t.String({ minLength: 8 }),
    }),
  });
```

### Composition Root

```typescript
// src/app.ts
import { Elysia } from "elysia";
import { cors } from "@elysiajs/cors";
import { swagger } from "@elysiajs/swagger";
import { authPlugin } from "./modules/auth/infrastructure/routes";
import { usersPlugin } from "./modules/users/infrastructure/routes";
import { errorHandler } from "./shared/infrastructure/error-handler";

const app = new Elysia()
  .use(cors({ origin: [process.env.FRONTEND_URL!] }))
  .use(swagger({ path: "/docs" }))
  .use(errorHandler)
  .use(authPlugin)
  .use(usersPlugin)
  .get("/health", () => ({ ok: true, timestamp: Date.now() }))
  .listen(3000);

console.log(`Server running at ${app.server?.url}`);
export type App = typeof app;  // Export for Eden Treaty client
```

### Eden Treaty Client (Auto-typed)

```typescript
// client-app/api.ts
import { treaty } from "@elysiajs/eden";
import type { App } from "../server/src/app";

const api = treaty<App>("http://localhost:3000");

// Fully typed — IntelliSense for routes, params, responses
const { data, error } = await api.auth.login.post({
  email: "user@example.com",
  password: "secure123",
});
// data typed as { token: string }
```

### Repository Pattern

```typescript
// domain/auth.repository.ts (interface)
export interface AuthRepository {
  findByEmail(email: string): Promise<User | null>;
  create(user: CreateUser): Promise<User>;
}

// infrastructure/pg-auth.repository.ts (implementation)
import { sql } from "../../shared/infrastructure/database";
import type { AuthRepository } from "../domain/auth.repository";

export class PgAuthRepository implements AuthRepository {
  async findByEmail(email: string) {
    const [user] = await sql`SELECT * FROM users WHERE email = ${email}`;
    return user ?? null;
  }
  async create(data: CreateUser) {
    // IMPORTANT: never pass undefined values to sql — use ?? null (throws in Bun v1.3+)
    const [user] = await sql`INSERT INTO users ${sql(data)} RETURNING *`;
    return user;
  }
}
```

### Database Setup (Native Bun.SQL)

```typescript
// shared/infrastructure/database.ts
import { SQL } from "bun";

// Lazy initialization — connects on first query, not at import (critical for cold starts)
export const sql = new SQL({ max: 10, idleTimeout: 30 });
// Reads DATABASE_URL from env automatically
```

---

## Pattern 2: Microservices with Bun

Each service is a standalone Bun application.

### Service Layout

```
services/
├─ api-gateway/          (Bun + Elysia — routing, auth, rate limiting)
├─ user-service/         (Bun + Elysia — user CRUD, profiles)
├─ payment-service/      (Bun + Elysia — Stripe, saga orchestration)
├─ notification-service/ (Bun + Elysia — WebSockets, email, push)
├─ shared-libs/          (shared TypeBox schemas, error types)
└─ infra/
    ├─ docker-compose.yml
    └─ k8s/
```

### Inter-Service Communication

```typescript
// Event-driven via Kafka/NATS
const app = new Elysia()
  .post("/users", async ({ body }) => {
    const user = await createUser(body);
    await kafka.publish("user.created", { userId: user.id });
    return user;
  });

kafka.subscribe("user.created", async (event) => {
  await sendWelcomeEmail(event.userId);
});
```

**Performance expectations**: 150k req/sec at p99=12ms (Graviton3). 3x fewer pods than Node.js. 35% lower Lambda costs.

---

## Pattern 3: Serverless & Edge

### AWS Lambda (Container — cross-compile)

```dockerfile
FROM oven/bun:1 AS builder
WORKDIR /app
COPY package.json bun.lock ./
RUN bun install --frozen-lockfile
COPY . .
RUN bun build --compile --target=bun-linux-x64 src/lambda.ts --outfile=lambda

FROM debian:bookworm-slim
COPY --from=builder /app/lambda /lambda
ENTRYPOINT ["/lambda"]
```

### Cloudflare Workers (via Hono)

```typescript
import { Hono } from "hono";

const app = new Hono();
app.get("/", (c) => c.json({ status: "ok" }));

export default app;
```

### Cold Start Optimization
- Lazy-init DB connections (on first request, not import time)
- Reduce dependency fan-out — fewer imports = faster resolution
- Avoid heavy top-level awaits
- Use `Bun.SQL` (native driver, no npm overhead)
- Compile to single executable when possible

---

## Pattern 4: Full-Stack Bun

```typescript
// app.ts — single process serves frontend + API
import { serve } from "bun";
import homepage from "./index.html";

serve({
  port: 3000,
  routes: {
    "/": homepage,  // Bun serves HTML with HMR in dev
    "/api/users": async (req) => {
      const { sql } = await import("./shared/infrastructure/database");
      const users = await sql`SELECT * FROM users LIMIT 10`;
      return Response.json({ users });
    },
  },
  development: { hmr: true, console: true },
});
```

```bash
bun build ./index.html --production  # Tree-shaking, minification, code splitting
```

---

## Shared Configuration

### bunfig.toml

```toml
[install]
frozen = true  # Like --frozen-lockfile

[test]
coverage = true
coverageReporter = ["text", "lcov"]

[run]
env = ".env"
```

### tsconfig.json (Bun-optimized, v1.3+)

```json
{
  "compilerOptions": {
    "target": "ESNext",
    "module": "Preserve",
    "moduleResolution": "bundler",
    "types": ["bun-types"],
    "strict": true,
    "noEmit": true,
    "skipLibCheck": true
  },
  "include": ["src/**/*.ts"]
}
```

> `"module": "Preserve"` is the new default in Bun v1.3. Use it instead of `"ESNext"` to avoid bundler output surprises.

### Error Handling Plugin

```typescript
// shared/infrastructure/error-handler.ts
import { Elysia } from "elysia";

export class AppError extends Error {
  constructor(public statusCode: number, message: string) {
    super(message);
  }
}

export const errorHandler = new Elysia()
  .onError(({ code, error, set }) => {
    if (error instanceof AppError) {
      set.status = error.statusCode;
      return { error: error.message };
    }
    if (code === "VALIDATION") {
      set.status = 400;
      return { error: "Invalid request" };
    }
    console.error(error);
    set.status = 500;
    return { error: "Internal server error" };
  });
```
