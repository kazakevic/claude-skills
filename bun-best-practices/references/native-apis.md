# Bun Native APIs

Built-in APIs that replace npm packages. Use these before reaching for third-party alternatives.

---

## Bun.SQL — Database Client

Unified driver for PostgreSQL, MySQL/MariaDB, and SQLite. No npm package needed.

```typescript
import { SQL } from "bun";

// Auto-selects driver from DATABASE_URL protocol (postgres://, mysql://, file:)
const sql = new SQL({ max: 10, idleTimeout: 30 });
```

### PostgreSQL Usage

```typescript
// Tagged template — safe parameterization
const [user] = await sql`SELECT * FROM users WHERE email = ${email}`;

// Bulk insert
const users = await sql`INSERT INTO users ${sql(dataArray)} RETURNING *`;

// TEXT[] column — use sql.array() for single value
await sql`UPDATE posts SET tags = ${sql.array(["news", "tech"])} WHERE id = ${id}`;

// IN clause expansion
await sql`SELECT * FROM users WHERE id IN ${sql([1, 2, 3])}`;
```

### CRITICAL: undefined breaks queries (v1.3+)

```typescript
// BAD — throws in Bun v1.3+
await sql`INSERT INTO items (name, desc) VALUES (${name}, ${description})`;
// If description is undefined → throws instead of silent NULL

// GOOD — always coerce to null
await sql`INSERT INTO items (name, desc) VALUES (${name}, ${description ?? null})`;
```

### MySQL / SQLite

```typescript
// MySQL — same API, 9x faster than mysql2
const mysql = new SQL("mysql://user:pass@localhost:3306/mydb");

// SQLite — ideal for testing, embedded use
const db = new SQL("file:./local.db");
const rows = await db`SELECT * FROM items`;
```

### Performance
- PostgreSQL: up to 50% faster than popular Node.js pg clients
- MySQL: 9x faster than mysql2, 4x faster than MariaDB client
- SQLite: native, zero-copy reads

---

## Bun.Redis — Built-in Redis Client

Native RESP3 implementation in Zig. 7.9x faster than ioredis. Added in Bun v1.3.

```typescript
import { Redis } from "bun";

const redis = new Redis("redis://localhost:6379");
// Or from env: new Redis(process.env.REDIS_URL)

await redis.set("key", "value");
await redis.set("key", "value", { ex: 60 });  // TTL 60s
const val = await redis.get("key");            // string | null

// Increment / counters
await redis.incr("visits");
await redis.incrby("score", 10);

// Hashes
await redis.hset("user:1", { name: "Alice", age: "30" });
const name = await redis.hget("user:1", "name");

// Lists
await redis.lpush("queue", "job1");
const job = await redis.rpop("queue");

// Sets
await redis.sadd("tags", "news", "tech");
const tags = await redis.smembers("tags");

// TTL / expiry
await redis.expire("session:abc", 3600);
const ttl = await redis.ttl("session:abc");

// Pipelining (batch commands)
const [r1, r2] = await redis.pipeline()
  .set("a", "1")
  .get("a")
  .exec();

// Delete
await redis.del("key");
await redis.del("key1", "key2");
```

**Note**: MULTI/EXEC transactions use raw commands for now.

---

## Bun.file() — File I/O

```typescript
// Read
const file = Bun.file("./data.json");
const json = await file.json();
const text = await file.text();
const buffer = await file.arrayBuffer();

// Write
await Bun.write("./output.txt", "hello");
await Bun.write("./output.json", JSON.stringify(data));

// Stream large files
const stream = Bun.file("./large.csv").stream();

// Check existence
const exists = await Bun.file("./config.json").exists();
const size = Bun.file("./data.bin").size;
```

---

## Bun.password — Secure Hashing

```typescript
// Hash (bcrypt or argon2)
const hash = await Bun.password.hash("mypassword");
const hash = await Bun.password.hash("mypassword", { algorithm: "argon2id", memoryCost: 65536 });

// Verify
const match = await Bun.password.verify("mypassword", hash);
```

---

## Bun.secrets — OS Keychain Credentials

For local dev only (not Docker/production). Uses macOS Keychain, Linux Secret Service, Windows Credential Manager.

```typescript
await Bun.secrets.set("DB_PASSWORD", "hunter2");
const pw = await Bun.secrets.get("DB_PASSWORD");
await Bun.secrets.delete("DB_PASSWORD");
```

---

## Bun.YAML — Native YAML

```typescript
import config from "./config.yaml";  // auto-parsed as default import

const parsed = Bun.YAML.parse(yamlString);
const serialized = Bun.YAML.stringify(obj);
```

---

## Bun.JSONC — JSON with Comments

```typescript
const data = Bun.JSONC.parse(`{
  "port": 3000,  // server port
  "debug": true  /* enable verbose logging */
}`);
```

---

## Bun.Archive — Tarball API

```typescript
const archive = new Bun.Archive();
archive.add("file.txt", "content");
archive.add("nested/path.json", JSON.stringify(data));
const bytes = archive.toBuffer({ compression: "gzip" });

// Extract
const extracted = Bun.Archive.extract(bytes);
```

---

## Bun.stripANSI — Remove Color Codes

SIMD-accelerated. Useful for log processing.

```typescript
const clean = Bun.stripANSI("\x1b[31mRed text\x1b[0m");
// → "Red text"
```

---

## Bun.serve() — HTTP Server

```typescript
Bun.serve({
  port: 3000,
  routes: {
    "/": new Response("Hello"),
    "/api/data": async (req) => Response.json({ ok: true }),
  },
  // Static files (zero-copy)
  static: {
    "/favicon.ico": Bun.file("./public/favicon.ico"),
  },
  // TLS
  tls: {
    cert: Bun.file("./certs/cert.pem"),
    key: Bun.file("./certs/key.pem"),
  },
  // Error handling
  error(err) {
    return new Response(`Error: ${err.message}`, { status: 500 });
  },
});
```

---

## Bun.build() — Programmatic Bundler

```typescript
const result = await Bun.build({
  entrypoints: ["./src/index.ts"],
  outdir: "./dist",
  target: "bun",
  minify: true,
  splitting: true,     // code splitting
  sourcemap: "linked",
  metafile: true,      // bundle analysis
});

// Cross-platform compile
await Bun.build({
  entrypoints: ["./src/app.ts"],
  compile: true,
  target: "bun-linux-x64",    // or bun-darwin-arm64, bun-windows-x64
  outfile: "./dist/server",
});
```

---

## Bun.sleep() — Async Sleep

```typescript
await Bun.sleep(1000);        // 1 second
await Bun.sleep(new Date()); // sleep until date
Bun.sleepSync(100);           // synchronous (blocks)
```

---

## Bun.hash() — Hashing Utilities

```typescript
Bun.hash("hello");           // Wyhash (fast, non-crypto)
Bun.hash.md5("hello");
Bun.hash.sha256("hello");
Bun.hash.sha512("hello");
Bun.CryptoHasher.hash("SHA-256", "hello", "hex");
```

---

## Bun.env — Environment Variables

```typescript
// Type-safe access
const port = Number(Bun.env.PORT ?? 3000);
const dbUrl = Bun.env.DATABASE_URL!;

// Bun auto-loads .env in dev — no dotenv package needed
```
