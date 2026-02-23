---
name: bun-project-init
description: "Initialize a complete Bun.js project with Docker, Makefile, and environment files. Use when: (1) user wants to scaffold a new Bun project from scratch, (2) user wants a Dockerized Bun development environment, (3) user mentions 'init bun project', 'new bun app', 'scaffold bun', 'bun starter', 'bun docker setup'. Generates compose.yaml, Dockerfile, Makefile, .env/.env.example, and runs bun init with optional Tailwind, React, and shadcn/ui setup inside Docker."
---

# Bun Project Initializer Skill

Scaffolds a production-ready Bun.js project with Docker-first development workflow.

## What It Generates

```
./                        # Files created in current directory
├── compose.yaml          # Docker Compose with dev service
├── Dockerfile            # Multi-stage Bun Dockerfile with bash
├── Makefile              # Common dev commands
├── Claude.md             # Rules and conventions for Claude
├── .env                  # Local environment variables (gitignored)
├── .env.example          # Template for env vars
├── .gitignore            # Standard ignores
├── docs/                 # Project documentation
│   ├── INDEX.md          # Documentation index — single source of truth
│   └── architecture.md   # System architecture overview
└── src/                  # (created by bun init inside container)
```

## Step-by-Step Process

### 1. Ask the User

Generate all files **in the current working directory** — do NOT create a subdirectory. Derive `{{PROJECT_NAME}}` from the current folder name (lowercase, hyphenated). Never ask for a project name.

Before generating, confirm:
- **Port** — host port to expose (default: `3000`)
- **Database** — optionally add a Postgres or Redis service to compose (default: none)

The default `make init` runs `bun init --react=shadcn` inside the container — sets up React, Tailwind, and shadcn/ui automatically.

### 2. Generate Files

Use the templates below. Replace placeholders:
- `{{PROJECT_NAME}}` — derived from **current folder name** (`basename $(pwd)`, lowercased, spaces→hyphens)
- `{{PORT}}` — exposed port
- `{{BUN_VERSION}}` — use `1` (latest stable)

All files are written directly into the current directory.

### 3. File Templates

---

#### compose.yaml

```yaml
services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
      target: dev
    container_name: {{PROJECT_NAME}}
    volumes:
      - .:/app
      - node_modules:/app/node_modules
    ports:
      - "${PORT:-{{PORT}}}:{{PORT}}"
    env_file:
      - .env
    command: bun --hot src/index.ts
    restart: unless-stopped

volumes:
  node_modules:
```

If the user requested **Postgres**, add:

```yaml
  db:
    image: postgres:17-alpine
    container_name: {{PROJECT_NAME}}-db
    environment:
      POSTGRES_USER: ${DB_USER:-postgres}
      POSTGRES_PASSWORD: ${DB_PASSWORD:-postgres}
      POSTGRES_DB: ${DB_NAME:-{{PROJECT_NAME}}}
    ports:
      - "${DB_PORT:-5432}:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
    restart: unless-stopped

# add to volumes:
  pgdata:
```

If the user requested **Redis**, add:

```yaml
  redis:
    image: redis:7-alpine
    container_name: {{PROJECT_NAME}}-redis
    ports:
      - "${REDIS_PORT:-6379}:6379"
    restart: unless-stopped
```

---

#### Dockerfile

```dockerfile
# ---- Base ----
FROM oven/bun:{{BUN_VERSION}} AS base
WORKDIR /app

# Install bash and common utilities
RUN apt-get update && apt-get install -y --no-install-recommends \
    bash \
    curl \
    git \
    ca-certificates \
  && rm -rf /var/lib/apt/lists/*

SHELL ["/bin/bash", "-c"]

# ---- Dev (used by docker compose — volumes mount everything) ----
FROM base AS dev
EXPOSE {{PORT}}
CMD ["bun", "--hot", "src/index.ts"]

# ---- Dependencies (production only) ----
FROM base AS deps
COPY package.json bun.lockb* ./
RUN bun install --frozen-lockfile 2>/dev/null || bun install

# ---- Production build ----
FROM base AS build
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN bun build src/index.ts --compile --outfile=server

# ---- Production ----
FROM gcr.io/distroless/cc-debian12 AS production
WORKDIR /app
COPY --from=build /app/server ./server
EXPOSE {{PORT}}
CMD ["./server"]
```

---

#### Makefile

```makefile
.PHONY: init start stop restart build logs bash shell install dev clean nuke test lint format

# ---- Project Init ----
init: ## Initialize the project (first time setup)
	docker compose build
	docker compose run --rm --name {{PROJECT_NAME}}-init app bun init --react=shadcn
	docker compose run --rm --name {{PROJECT_NAME}}-install app bun install
	@echo "✅ Project initialized. Run 'make start' to begin."

# ---- Docker Lifecycle ----
start: ## Start all services
	docker compose up -d

stop: ## Stop all services
	docker compose down

restart: ## Restart all services
	docker compose restart

build: ## Rebuild Docker images
	docker compose build --no-cache

logs: ## Tail logs from all services
	docker compose logs -f

# ---- Shell Access ----
bash: ## Open a bash shell in the app container
	docker compose exec app bash

shell: bash ## Alias for bash

# ---- Development ----
dev: ## Start in dev mode with hot reload (attached)
	docker compose up

install: ## Install dependencies
	docker compose run --rm --name {{PROJECT_NAME}}-install app bun install

add: ## Add a package (usage: make add p=<package>)
	docker compose run --rm --name {{PROJECT_NAME}}-add app bun add $(p)

add-dev: ## Add a dev package (usage: make add-dev p=<package>)
	docker compose run --rm --name {{PROJECT_NAME}}-add-dev app bun add -d $(p)

# ---- Code Quality ----
test: ## Run tests
	docker compose exec app bun test

lint: ## Run linter
	docker compose exec app bun run lint

format: ## Run formatter
	docker compose exec app bun run format

# ---- Cleanup ----
clean: ## Stop containers and remove volumes
	docker compose down -v

nuke: ## Full reset: remove containers, volumes, images
	docker compose down -v --rmi all
	@echo "🧹 Everything removed. Run 'make init' to start fresh."

# ---- Help ----
help: ## Show this help
	@grep -E '^[a-zA-Z_-]+:.*?## .*$$' $(MAKEFILE_LIST) | sort | awk 'BEGIN {FS = ":.*?## "}; {printf "\033[36m%-15s\033[0m %s\n", $$1, $$2}'

.DEFAULT_GOAL := help
```

---

#### Claude.md

```markdown
# Claude Rules for {{PROJECT_NAME}}

## Documentation

- **Always read `docs/INDEX.md` first** before working on any feature or task. It is the single source of truth for all project documentation.
- When you build a new feature, create a doc file describing it inside `docs/` (can be nested in subfolders by domain, e.g. `docs/auth/login.md`, `docs/payments/checkout.md`).
- Each feature doc should include: what it does, key files involved, API endpoints (if any), and any gotchas.
- After creating a feature doc, **add its path and a short one-line headline to `docs/INDEX.md`**.
- Keep docs concise — prefer short, scannable descriptions over long prose.

## Code Conventions

- All code runs inside Docker. Use `make` commands, never run `bun` directly on the host.
- Use TypeScript for all source files.
- Follow the architecture described in `docs/architecture.md`.
- Keep components and modules small and focused — one responsibility per file.
- Use the project's existing patterns before introducing new ones. Read relevant source files first.

## Git

- Write clear, concise commit messages.
- One logical change per commit.

## Environment

- Never hardcode secrets or config values. Use `.env` and reference via `process.env`.
- When adding a new env variable, update both `.env` and `.env.example`.
```

---

#### docs/INDEX.md

```markdown
# {{PROJECT_NAME}} — Documentation Index

> Single source of truth for all project docs. Every feature doc must be listed here.

## Architecture

| Doc | Description |
|-----|-------------|
| [architecture.md](./architecture.md) | System architecture overview and key decisions |

## Features

| Doc | Description |
|-----|-------------|
| | |

<!-- Add new feature docs here as: | [path](./path) | Short description | -->
```

---

#### docs/architecture.md

```markdown
# Architecture

## Overview

<!-- Describe the high-level architecture of the project here -->

{{PROJECT_NAME}} is a React + Tailwind + shadcn/ui application running on Bun inside Docker.

## Stack

- **Runtime**: Bun
- **Frontend**: React, Tailwind CSS, shadcn/ui
- **Containerization**: Docker + Docker Compose
- **Package Manager**: Bun

## Project Structure

```
src/
├── components/    # Reusable UI components
├── pages/         # Page-level components / routes
├── lib/           # Shared utilities and helpers
├── hooks/         # Custom React hooks
└── index.ts       # Application entry point
```

## Key Decisions

<!-- Document important architectural decisions here -->

| Decision | Rationale |
|----------|-----------|
| Docker-first development | Consistent environment across all machines |
| Bun runtime | Fast startup, native TS, built-in bundler |
| shadcn/ui | Copy-paste components, full ownership, Tailwind-native |

## Data Flow

<!-- Describe how data flows through the application -->

## Deployment

- **Dev**: `make dev` — hot reload via `bun --hot`
- **Production**: Multi-stage Docker build → single compiled binary on distroless
```

---

#### .env.example

```env
# App
PORT={{PORT}}
NODE_ENV=development

# Database (if using Postgres)
# DB_USER=postgres
# DB_PASSWORD=postgres
# DB_NAME={{PROJECT_NAME}}
# DB_HOST=db
# DB_PORT=5432
# DATABASE_URL=postgresql://${DB_USER}:${DB_PASSWORD}@${DB_HOST}:${DB_PORT}/${DB_NAME}

# Redis (if using Redis)
# REDIS_URL=redis://redis:6379
```

Uncomment the relevant sections based on what the user selected.

#### .env

Copy of `.env.example` with values filled in for local development.

---

#### .gitignore

```gitignore
node_modules/
dist/
*.log
.env
.DS_Store
bun.lockb
```

---

### 4. After Generation

Once all files are created directly in the current working directory (copy to `/mnt/user-data/outputs/` as well):

1. Tell the user to run:
   ```bash
   make init
   make start
   ```
2. Mention that `make help` shows all available commands
3. If databases were added, note the connection details from `.env`

### 5. Important Notes

- The Dockerfile uses `oven/bun` official image and installs `bash` explicitly so `make bash` always works
- The `dev` target in the Dockerfile is used by compose for development (hot reload with `bun --hot`)
- The `production` target compiles to a single binary and uses distroless for minimal attack surface
- `node_modules` volume is used to avoid host/container conflicts on different OS architectures
- `.env` is gitignored; `.env.example` is committed as a template