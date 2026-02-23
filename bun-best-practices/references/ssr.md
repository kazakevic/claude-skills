# Server-Side Rendering (SSR) with Bun

## Approach Comparison

| Approach | When to use |
|----------|-------------|
| `renderToReadableStream` | React 19 streaming SSR — preferred for all new apps |
| `renderToString` | Simple, blocking SSR — use only when streaming isn't needed |
| Bun HTML bundler (`import page from "./page.html"`) | SPA / CSR — NOT SSR; Bun serves HTML with client-side React |
| ElysiaJS + SSR plugin | Full-stack app with type-safe API + SSR on same server |

Bun runs `.tsx` natively — no Babel, no webpack, no additional config needed.

---

## Minimal SSR with Bun.serve()

```typescript
// src/server.ts
import { renderToReadableStream } from "react-dom/server";
import { App } from "./App";

Bun.serve({
  port: 3000,
  async fetch(req) {
    const url = new URL(req.url);

    // Serve client bundle
    if (url.pathname === "/client.js") {
      return new Response(Bun.file("./dist/client.js"));
    }

    // SSR all other routes
    const stream = await renderToReadableStream(
      <html lang="en">
        <head>
          <meta charSet="utf-8" />
          <title>My App</title>
        </head>
        <body>
          <div id="root">
            <App url={req.url} />
          </div>
          <script type="module" src="/client.js" />
        </body>
      </html>,
      {
        bootstrapScripts: ["/client.js"],
        onError(error) { console.error(error); },
      }
    );

    return new Response(stream, {
      headers: { "Content-Type": "text/html; charset=utf-8" },
    });
  },
});
```

---

## Client Hydration

```typescript
// src/client.tsx
import { hydrateRoot } from "react-dom/client";
import { App } from "./App";

// hydrateRoot — attaches React to server-rendered HTML (no re-render)
hydrateRoot(document, <App url={window.location.href} />);
```

Build the client bundle:

```bash
bun build ./src/client.tsx --outdir=./dist --target=browser --minify
```

---

## Streaming SSR with Suspense (React 19)

`renderToReadableStream` streams HTML progressively — browser receives the shell immediately, deferred content streams in as it resolves.

```tsx
// src/App.tsx
import { Suspense } from "react";
import { UserProfile } from "./UserProfile";

export function App({ url }: { url: string }) {
  return (
    <html>
      <body>
        {/* Shell renders immediately */}
        <header><h1>My App</h1></header>

        {/* Deferred — streams in when data resolves */}
        <Suspense fallback={<p>Loading profile...</p>}>
          <UserProfile />
        </Suspense>
      </body>
    </html>
  );
}
```

```tsx
// src/UserProfile.tsx — async Server Component pattern
export async function UserProfile() {
  const user = await fetchUser();  // awaited on server
  return <div>{user.name}</div>;
}
```

---

## ElysiaJS + SSR

Full-stack: API routes + SSR on the same Elysia server.

```typescript
// src/app.ts
import { Elysia } from "elysia";
import { renderToReadableStream } from "react-dom/server";
import { App } from "./App";

const app = new Elysia()
  // API routes
  .get("/api/users", () => db.users.findMany())

  // Static assets
  .get("/assets/*", ({ params }) =>
    new Response(Bun.file(`./dist/${params["*"]}`))
  )

  // SSR catch-all
  .get("/*", async ({ request }) => {
    const stream = await renderToReadableStream(
      <App url={request.url} />,
      { bootstrapScripts: ["/assets/client.js"] }
    );
    return new Response(stream, {
      headers: { "Content-Type": "text/html; charset=utf-8" },
    });
  })
  .listen(3000);
```

---

## renderToString (Blocking SSR)

Use only when you need a complete HTML string (e.g., email rendering, PDF generation).

```typescript
import { renderToString } from "react-dom/server";

const html = renderToString(<EmailTemplate data={data} />);
// Blocks until full render — use renderToReadableStream for HTTP responses
```

---

## Build Pipeline

```
src/
  server.ts       → runs with `bun --hot src/server.ts`
  client.tsx      → built with `bun build` → dist/client.js
  App.tsx         → shared between server and client
```

```bash
# Dev — build client bundle, start server with hot reload
bun build ./src/client.tsx --outdir=./dist --target=browser --watch &
bun --hot src/server.ts

# Production
bun build ./src/client.tsx --outdir=./dist --target=browser --minify --splitting
bun build --compile src/server.ts --outfile=server
```

---

## SSR Performance Tips

- Use `renderToReadableStream` over `renderToString` — browser starts painting immediately
- Wrap slow data fetches in `<Suspense>` — shell renders without blocking on data
- Keep server components lean — avoid importing large client-side-only libraries
- Cache rendered shells with `Bun.Redis` for static-ish routes
- Use `bootstrapScripts` (not inline `<script>`) — lets Bun/browser handle module loading correctly

---

## Gotchas

- **`window` / `document` on server**: Guard browser-only APIs with `typeof window !== "undefined"` or move them to `useEffect`
- **CSS modules in SSR**: Bun handles `.module.css` natively in the bundler but not at runtime — build the client bundle first
- **React 19 requirement**: `renderToReadableStream` requires React 19+. Older versions use `renderToPipeableStream` (Node streams) — not compatible with Bun's native `ReadableStream`
- **`bootstrapScripts` timing**: Scripts are injected at end of `<body>` — do not manually add `<script>` tags for the same bundle
