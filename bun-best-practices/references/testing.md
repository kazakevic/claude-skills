# Testing with bun:test

Jest-compatible test runner built into Bun. 10-30x faster than Jest. Zero config.

---

## Basic Usage

```typescript
import { describe, it, expect, beforeEach, afterEach } from "bun:test";

describe("UserService", () => {
  it("creates a user", async () => {
    const user = await userService.create({ email: "test@example.com" });
    expect(user.id).toBeDefined();
    expect(user.email).toBe("test@example.com");
  });

  it("throws on duplicate email", async () => {
    await expect(
      userService.create({ email: "existing@example.com" })
    ).rejects.toThrow("Email already in use");
  });
});
```

---

## Running Tests

```bash
bun test                            # All tests
bun test --grep "UserService"       # Filter by name (alias: -t)
bun test --grep "creates"           # Substring match
bun test src/modules/auth/          # Specific directory
bun test --watch                    # Watch mode
bun test --coverage                 # Coverage report
bun test --bail                     # Stop on first failure
bun test --timeout 10000            # Timeout per test (ms)
```

---

## Parallel & Retry

```typescript
import { test } from "bun:test";

// Run tests concurrently within a describe block
test.concurrent("fetches data in parallel", async () => {
  const [a, b] = await Promise.all([fetchA(), fetchB()]);
  expect(a).toBeDefined();
});

// Retry flaky tests
test.retry(3)("connects to external service", async () => {
  const conn = await connectToService();
  expect(conn.ok).toBe(true);
});
```

---

## Mocking

```typescript
import { mock, spyOn } from "bun:test";

// Mock a module
const mockFetch = mock(async (url: string) => ({
  json: async () => ({ data: "mocked" }),
}));

// Spy on object method
const spy = spyOn(mailer, "send");
await userService.register(userData);
expect(spy).toHaveBeenCalledWith(
  expect.objectContaining({ to: userData.email })
);

// Restore original
spy.mockRestore();
```

---

## Fake Timers

Works with `@testing-library/react` (fixed in Bun v1.3.6).

```typescript
import { describe, it, expect, beforeEach, afterEach } from "bun:test";
import { setSystemTime, useFakeTimers } from "bun:test";

describe("cache expiry", () => {
  beforeEach(() => {
    useFakeTimers();
  });

  afterEach(() => {
    // restore real timers
    setSystemTime(); // reset to real time
  });

  it("expires after TTL", async () => {
    cache.set("key", "value", { ttl: 5000 });
    setSystemTime(Date.now() + 6000);  // advance 6 seconds
    expect(cache.get("key")).toBeNull();
  });
});
```

---

## Database Testing Strategy

```typescript
// Unit tests — SQLite in-memory (fast, no external dependency)
import { SQL } from "bun";

const testDb = new SQL(":memory:");
await testDb`CREATE TABLE users (id INTEGER PRIMARY KEY, email TEXT)`;

// Integration tests — real PostgreSQL
// Use a test database, run migrations before suite
const testDb = new SQL("postgres://localhost:5432/test_db");
```

### Test Isolation Pattern

```typescript
import { beforeEach, afterEach } from "bun:test";
import { sql } from "../src/shared/infrastructure/database";

// Wrap each test in a transaction and rollback
let rollback: () => Promise<void>;

beforeEach(async () => {
  await sql`BEGIN`;
  rollback = async () => sql`ROLLBACK`;
});

afterEach(async () => {
  await rollback();
});
```

---

## Snapshot Testing

```typescript
import { expect, it } from "bun:test";

it("matches snapshot", () => {
  const output = renderComponent(<UserCard name="Alice" />);
  expect(output).toMatchSnapshot();
});
```

Snapshots stored in `__snapshots__/` directory.

---

## Coverage Configuration

```toml
# bunfig.toml
[test]
coverage = true
coverageReporter = ["text", "lcov", "html"]
coverageThreshold = { line = 80, function = 80, statement = 80 }
coverageDir = "coverage"
```

```bash
bun test --coverage
# Opens HTML report: open coverage/index.html
```

---

## VS Code Integration

Install the **Bun for Visual Studio Code** extension:
- Run/debug individual tests from gutter icons
- Breakpoints work with native Bun source maps
- Test explorer panel

---

## Common Matchers

```typescript
expect(value).toBe(exact);           // strict equality
expect(value).toEqual(deep);         // deep equality
expect(value).toBeNull();
expect(value).toBeDefined();
expect(value).toBeTruthy();
expect(value).toContain(item);       // array or string
expect(value).toHaveLength(n);
expect(obj).toHaveProperty("key");
expect(obj).toMatchObject({ partial: true });
expect(fn).toThrow("message");
expect(fn).toThrow(ErrorClass);
expect(promise).rejects.toThrow();
expect(mock).toHaveBeenCalled();
expect(mock).toHaveBeenCalledTimes(2);
expect(mock).toHaveBeenCalledWith(args);
```
