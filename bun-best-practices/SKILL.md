---
name: bun-best-practices
description: "Best practices and architecture patterns for building production applications with Bun.js in 2026. Use when: (1) creating new Bun/TypeScript backend projects or APIs, (2) setting up Bun project structure with ElysiaJS or Hono, (3) writing Bun.serve(), bun:test, or Bun-native code, (4) optimizing Bun app performance, cold starts, or serverless deployment, (5) migrating from Node.js/Express to Bun, (6) configuring Docker, CI/CD, or deployment for Bun apps, (7) choosing between ElysiaJS vs Hono vs Bun.serve(), (8) implementing clean architecture with Elysia plugins, (9) using native Bun APIs: Bun.SQL, Bun.Redis, Bun.secrets, Bun.YAML, Bun.file(), (10) server-side rendering (SSR) with React and Bun, renderToReadableStream, hydration, streaming SSR. Triggers on mentions of Bun, ElysiaJS, Elysia, Hono with Bun, bun:test, Bun.serve, bun build, bun install, SSR, renderToReadableStream, or any Bun-specific API."
---

# Bun.js Best Practices 2026

> Bun joined Anthropic (Dec 2025). Remains MIT-licensed open-source. Powers Claude Code.

## Runtime Overview

All-in-one JS/TS toolkit: runtime + bundler + test runner + package manager. Built on JavaScriptCore, written in Zig. Runs `.ts/.tsx/.jsx` natively — zero config.

**Performance**: ~52k req/s HTTP (vs Deno 22k, Node 13k). 4-10x faster cold starts than Node.js. 30-40% less memory.

## Framework Selection

| Need | Choose | Why |
|------|--------|-----|
| Bun-native type-safe API | **ElysiaJS** | End-to-end type safety, Eden Treaty client, AOT validation (~18x faster than Zod), built-in OpenAPI |
| Multi-runtime / edge portable | **Hono** | Runs on Bun, Deno, Cloudflare Workers, Vercel Edge |
| Maximum raw throughput | **Bun.serve()** | Zero framework overhead |
| Full-stack React app | **Bun dev server** | `bun ./index.html` with HMR, JSX, CSS imports |

**Default choice: ElysiaJS.** Use Hono only when multi-runtime portability is required.

## Project Setup

```bash
bun init
bun add elysia @elysiajs/cors @elysiajs/swagger @elysiajs/jwt

bun --hot src/app.ts                           # Dev with hot reload
bun build --compile src/app.ts                  # Production single binary
bun build --compile --target=bun-linux-x64 ...  # Cross-compile
bun test --grep "pattern"                        # Run matching tests
```

## Architecture

Clean architecture with domain-driven Elysia plugins. See [references/architecture.md](references/architecture.md).

```
src/
  ├─ modules/
  │   ├─ auth/         (application/ domain/ infrastructure/)
  │   ├─ users/
  │   └─ payments/
  ├─ shared/           (database, middleware, error handling)
  └─ app.ts            (composition root)
```

## Core Best Practices

### Development Workflow
- Run `.ts` directly — never use ts-node or build steps in dev
- `bun --hot` for dev servers (instant reload)
- `bun test --grep "pattern"` — filter tests (alias for `--test-name-pattern`)
- Commit `bun.lock` (text-based format since v1.2; replaces old binary `bun.lockb`)
- `bun update --interactive` — TUI for selective dependency updates
- `bun why <package>` — explain dependency chain
- `bun pm audit` — security audit

### Performance
- Use native Bun APIs: `Bun.file()`, `Bun.password.hash()`, `Bun.SQL`, `Bun.Redis`
- Lazy-init DB connections on first request (not top-level await) — critical for cold starts
- Reduce dependency fan-out — fewer imports = faster startup
- `Bun.serve({ static: { ... } })` for static responses (zero-copy)
- Compile to single executable: `bun build --compile` (supports cross-platform targets)

### Type Safety (Elysia)
- Schema-first: TypeBox drives validation + type inference + OpenAPI docs
- Eden Treaty for auto-generated typed clients (no codegen)
- Validate at boundary: params, query, body, headers in one route-level call
- Prefer TypeBox over Zod (AOT compiled in Bun)

### Security
- `@elysiajs/jwt` for token handling
- `@elysiajs/cors` with explicit origin whitelist (never wildcard in production)
- Native TLS: `serve: { tls: { cert: Bun.file('cert.pem'), key: Bun.file('key.pem') } }`
- `Bun.secrets` for OS keychain credentials in local dev

### SQL Safety (v1.3 Breaking Change)
- `Bun.SQL` throws on explicit `undefined` params — always use `value ?? null`
- `sql.array(arr)` for single TEXT[] column inserts
- For bulk object inserts with array columns: still use manual `{elem1,elem2}` format
- See [references/native-apis.md](references/native-apis.md) for full SQL patterns

### Testing
See [references/testing.md](references/testing.md) for full patterns.

### Deployment
See [references/deployment.md](references/deployment.md) for Docker, serverless, CI/CD, health checks.

## Reference Files

| File | When to read |
|------|--------------|
| [references/architecture.md](references/architecture.md) | Clean arch, microservices, full-stack, config patterns |
| [references/deployment.md](references/deployment.md) | Docker, CI/CD, serverless, health checks, observability |
| [references/native-apis.md](references/native-apis.md) | Bun.SQL, Bun.Redis, Bun.secrets, Bun.YAML, Bun.file, etc. |
| [references/testing.md](references/testing.md) | bun:test patterns, mocking, fake timers, coverage |
| [references/ssr.md](references/ssr.md) | React SSR, streaming, hydration, ElysiaJS + SSR |

## Migration from Node.js

1. `bun install` — drop-in for npm/yarn (zero risk, 20-40x faster)
2. `bun test` — replace Jest/Vitest (low risk, 10-30x faster)
3. `bun run` — swap `node` for `bun` in scripts
4. Remove ts-node — run `.ts` directly
5. Framework swap — Express/Fastify → Elysia/Hono (medium risk, 2-4x throughput gain)
6. Full runtime — deploy with Bun in production

>95% of Node.js APIs are supported. Most Express/Fastify apps run on Bun without code changes.

## Recommended Stack

| Layer | Tool |
|-------|------|
| Runtime | Bun |
| Framework | ElysiaJS (or Hono for multi-runtime) |
| ORM | Drizzle ORM + Bun.SQL native driver |
| Caching | Bun.Redis (built-in, 7.9x faster than ioredis) |
| Validation | TypeBox (via Elysia) |
| API Docs | @elysiajs/swagger |
| Auth | @elysiajs/jwt |
| Testing | bun:test |
| Bundling | bun build |
| Linting | Biome |
| Container | Docker (oven/bun:1) |
| Monitoring | OpenTelemetry + Grafana |
