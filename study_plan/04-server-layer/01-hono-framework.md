# Lesson 01: Hono Framework

## Learning Objectives

By the end of this lesson, you will be able to:
- Explain what Hono is and why opencode uses it
- Understand basic Hono concepts (app, routes, middleware, context)
- Trace how the server is initialized and configured
- Understand CORS configuration and error handling middleware
- Read and understand route definitions in the codebase

---

## What is Hono?

Hono (炎 - Japanese for "flame") is a small, fast, and ultralight web framework for JavaScript/TypeScript. It's designed to work across multiple runtimes including:

- **Bun** (what opencode uses)
- Deno
- Cloudflare Workers
- Node.js
- AWS Lambda

### Key Characteristics

```
┌─────────────────────────────────────────────────────────────────┐
│                        Hono Framework                           │
├─────────────────────────────────────────────────────────────────┤
│  ✓ Ultrafast - Built for performance                           │
│  ✓ TypeScript-first - Full type safety                         │
│  ✓ Zero dependencies - Minimal footprint                       │
│  ✓ Multi-runtime - Works with Bun, Deno, Node, etc.           │
│  ✓ Web Standards - Uses Fetch API (Request/Response)          │
│  ✓ Middleware support - Composable request handling            │
└─────────────────────────────────────────────────────────────────┘
```

---

## Why opencode Uses Hono

opencode chose Hono for several reasons:

1. **Bun Compatibility**: Hono is optimized for Bun runtime, which opencode uses
2. **TypeScript-First**: Full type inference for routes, parameters, and responses
3. **Performance**: Minimal overhead for the HTTP layer
4. **OpenAPI Support**: Via `hono-openapi` for automatic API documentation
5. **Built-in Middleware**: CORS, authentication, streaming support

---

## Basic Hono Concepts

### 1. The App Instance

A Hono app is the central object that handles all HTTP requests:

```typescript
import { Hono } from "hono"

const app = new Hono()
```

### 2. Routes

Routes define how the server responds to different HTTP methods and paths:

```typescript
app.get("/path", handler)      // GET request
app.post("/path", handler)     // POST request
app.put("/path", handler)      // PUT request
app.delete("/path", handler)   // DELETE request
app.patch("/path", handler)    // PATCH request
app.all("/path", handler)      // All methods
```

### 3. Context (c)

Every handler receives a Context object that provides:

```typescript
app.get("/example", async (c) => {
  // Request information
  c.req.method          // HTTP method
  c.req.path            // Request path
  c.req.query("param")  // Query parameter
  c.req.header("name")  // Request header
  c.req.json()          // Parse JSON body
  
  // Response helpers
  return c.json({ data })      // JSON response
  return c.text("hello")       // Text response
  return c.html("<h1>Hi</h1>") // HTML response
})
```

### 4. Middleware

Middleware functions run before/after route handlers:

```typescript
app.use(async (c, next) => {
  // Before handler
  console.log("Request:", c.req.path)
  
  await next()  // Call the route handler
  
  // After handler
  console.log("Response sent")
})
```

### 5. Route Chaining

Routes can be chained and composed:

```typescript
app
  .get("/", handler1)
  .post("/", handler2)
  .route("/api", apiRoutes)  // Mount sub-routes
```

---

## Server Setup in opencode

The main server is defined in `packages/opencode/src/server/server.ts`. Let's examine its structure:

### Server Namespace and Lazy Initialization

```1:61:packages/opencode/src/server/server.ts
import { createHash } from "node:crypto"
import { Log } from "../util/log"
import { describeRoute, generateSpecs, validator, resolver, openAPIRouteHandler } from "hono-openapi"
import { Hono } from "hono"
import { cors } from "hono/cors"
import { proxy } from "hono/proxy"
import { basicAuth } from "hono/basic-auth"
import z from "zod"
// ... more imports

export namespace Server {
  const log = Log.create({ service: "server" })

  export const Default = lazy(() => createApp({}))

  export const createApp = (opts: { cors?: string[] }): Hono => {
    const app = new Hono()
    // ... configuration
  }
}
```

Key observations:
- Uses a namespace pattern for organization
- `lazy()` ensures single initialization
- `createApp` is a factory function that accepts options

### Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                         Server.createApp()                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                    Middleware Stack                          │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐  │   │
│  │  │ Error       │→ │ Auth        │→ │ Logging             │  │   │
│  │  │ Handler     │  │ (optional)  │  │ (timing)            │  │   │
│  │  └─────────────┘  └─────────────┘  └─────────────────────┘  │   │
│  │         ↓                                                    │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐  │   │
│  │  │ CORS        │→ │ Workspace   │→ │ Instance            │  │   │
│  │  │ Handler     │  │ Context     │  │ Provider            │  │   │
│  │  └─────────────┘  └─────────────┘  └─────────────────────┘  │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                      Route Handlers                          │   │
│  │  /global    /session    /provider    /config    /mcp        │   │
│  │  /project   /pty        /event       /file      /permission │   │
│  │  /question  /tui        /experimental                       │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                    Fallback Proxy                            │   │
│  │              → app.opencode.ai (web UI)                     │   │
│  └─────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Error Handling Middleware

The server has comprehensive error handling:

```66:84:packages/opencode/src/server/server.ts
    return app
      .onError((err, c) => {
        log.error("failed", {
          error: err,
        })
        if (err instanceof NamedError) {
          let status: ContentfulStatusCode
          if (err instanceof NotFoundError) status = 404
          else if (err instanceof Provider.ModelNotFoundError) status = 400
          else if (err.name === "ProviderAuthValidationFailed") status = 400
          else if (err.name.startsWith("Worktree")) status = 400
          else status = 500
          return c.json(err.toObject(), { status })
        }
        if (err instanceof HTTPException) return err.getResponse()
        const message = err instanceof Error && err.stack ? err.stack : err.toString()
        return c.json(new NamedError.Unknown({ message }).toObject(), {
          status: 500,
        })
      })
```

### Error Handling Flow

```
┌──────────────┐     ┌──────────────────────────────────────────────┐
│   Request    │────▶│              Route Handler                    │
└──────────────┘     └──────────────────────────────────────────────┘
                                        │
                                        │ throws error
                                        ▼
                     ┌──────────────────────────────────────────────┐
                     │              .onError()                       │
                     ├──────────────────────────────────────────────┤
                     │  Is NamedError?                              │
                     │    ├─ NotFoundError → 404                    │
                     │    ├─ ModelNotFoundError → 400               │
                     │    ├─ ProviderAuthValidationFailed → 400     │
                     │    ├─ Worktree* → 400                        │
                     │    └─ Other → 500                            │
                     │                                              │
                     │  Is HTTPException?                           │
                     │    └─ Return exception's response            │
                     │                                              │
                     │  Unknown error?                              │
                     │    └─ Wrap in NamedError.Unknown → 500       │
                     └──────────────────────────────────────────────┘
```

---

## Authentication Middleware

Optional basic authentication when `OPENCODE_SERVER_PASSWORD` is set:

```85:93:packages/opencode/src/server/server.ts
      .use((c, next) => {
        // Allow CORS preflight requests to succeed without auth.
        // Browser clients sending Authorization headers will preflight with OPTIONS.
        if (c.req.method === "OPTIONS") return next()
        const password = Flag.OPENCODE_SERVER_PASSWORD
        if (!password) return next()
        const username = Flag.OPENCODE_SERVER_USERNAME ?? "opencode"
        return basicAuth({ username, password })(c, next)
      })
```

Key points:
- OPTIONS requests bypass auth (for CORS preflight)
- Auth is only enabled if `OPENCODE_SERVER_PASSWORD` is set
- Default username is "opencode"

---

## Logging Middleware

Request logging with timing:

```94:110:packages/opencode/src/server/server.ts
      .use(async (c, next) => {
        const skipLogging = c.req.path === "/log"
        if (!skipLogging) {
          log.info("request", {
            method: c.req.method,
            path: c.req.path,
          })
        }
        const timer = log.time("request", {
          method: c.req.method,
          path: c.req.path,
        })
        await next()
        if (!skipLogging) {
          timer.stop()
        }
      })
```

Features:
- Skips logging for `/log` endpoint (prevents recursion)
- Logs method and path for each request
- Times request duration

---

## CORS Configuration

Cross-Origin Resource Sharing (CORS) controls which domains can access the API:

```111:136:packages/opencode/src/server/server.ts
      .use(
        cors({
          origin(input) {
            if (!input) return

            if (input.startsWith("http://localhost:")) return input
            if (input.startsWith("http://127.0.0.1:")) return input
            if (
              input === "tauri://localhost" ||
              input === "http://tauri.localhost" ||
              input === "https://tauri.localhost"
            )
              return input

            // *.opencode.ai (https only, adjust if needed)
            if (/^https:\/\/([a-z0-9-]+\.)*opencode\.ai$/.test(input)) {
              return input
            }
            if (opts?.cors?.includes(input)) {
              return input
            }

            return
          },
        }),
      )
```

### Allowed Origins

```
┌─────────────────────────────────────────────────────────────────┐
│                     CORS Allowed Origins                        │
├─────────────────────────────────────────────────────────────────┤
│  ✓ http://localhost:*        (local development)               │
│  ✓ http://127.0.0.1:*        (local development)               │
│  ✓ tauri://localhost         (Tauri desktop app)               │
│  ✓ http://tauri.localhost    (Tauri desktop app)               │
│  ✓ https://tauri.localhost   (Tauri desktop app)               │
│  ✓ https://*.opencode.ai     (production domains)              │
│  ✓ Custom origins via opts   (configurable)                    │
└─────────────────────────────────────────────────────────────────┘
```

---

## Workspace Context Middleware

This middleware sets up the project context for each request:

```200:226:packages/opencode/src/server/server.ts
      .use(async (c, next) => {
        if (c.req.path === "/log") return next()
        const rawWorkspaceID = c.req.query("workspace") || c.req.header("x-opencode-workspace")
        const raw = c.req.query("directory") || c.req.header("x-opencode-directory") || process.cwd()
        const directory = Filesystem.resolve(
          (() => {
            try {
              return decodeURIComponent(raw)
            } catch {
              return raw
            }
          })(),
        )

        return WorkspaceContext.provide({
          workspaceID: rawWorkspaceID ? WorkspaceID.make(rawWorkspaceID) : undefined,
          async fn() {
            return Instance.provide({
              directory,
              init: InstanceBootstrap,
              async fn() {
                return next()
              },
            })
          },
        })
      })
```

### Context Resolution

```
┌─────────────────────────────────────────────────────────────────┐
│                    Request Context Setup                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. Extract workspace ID:                                       │
│     ?workspace=... OR x-opencode-workspace header               │
│                                                                 │
│  2. Extract directory:                                          │
│     ?directory=... OR x-opencode-directory header               │
│     OR process.cwd()                                            │
│                                                                 │
│  3. Create WorkspaceContext                                     │
│     └─ Create Instance (project context)                        │
│         └─ Run InstanceBootstrap                                │
│             └─ Execute route handler                            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Route Registration

Routes are registered using `.route()` for modular organization:

```250:261:packages/opencode/src/server/server.ts
      .route("/project", ProjectRoutes())
      .route("/pty", PtyRoutes())
      .route("/config", ConfigRoutes())
      .route("/experimental", ExperimentalRoutes())
      .route("/session", SessionRoutes())
      .route("/permission", PermissionRoutes())
      .route("/question", QuestionRoutes())
      .route("/provider", ProviderRoutes())
      .route("/", FileRoutes())
      .route("/", EventRoutes())
      .route("/mcp", McpRoutes())
      .route("/tui", TuiRoutes())
```

---

## Server Startup

The `listen` function starts the HTTP server:

```574:618:packages/opencode/src/server/server.ts
  export function listen(opts: {
    port: number
    hostname: string
    mdns?: boolean
    mdnsDomain?: string
    cors?: string[]
  }) {
    url = new URL(`http://${opts.hostname}:${opts.port}`)
    const app = createApp(opts)
    const args = {
      hostname: opts.hostname,
      idleTimeout: 0,
      fetch: app.fetch,
      websocket: websocket,
    } as const
    const tryServe = (port: number) => {
      try {
        return Bun.serve({ ...args, port })
      } catch {
        return undefined
      }
    }
    const server = opts.port === 0 ? (tryServe(4096) ?? tryServe(0)) : tryServe(opts.port)
    if (!server) throw new Error(`Failed to start server on port ${opts.port}`)

    // mDNS publishing for network discovery
    const shouldPublishMDNS =
      opts.mdns &&
      server.port &&
      opts.hostname !== "127.0.0.1" &&
      opts.hostname !== "localhost" &&
      opts.hostname !== "::1"
    if (shouldPublishMDNS) {
      MDNS.publish(server.port!, opts.mdnsDomain)
    }
    // ...
  }
```

### Port Selection Logic

```
┌─────────────────────────────────────────────────────────────────┐
│                    Port Selection                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  If port === 0:                                                 │
│    1. Try port 4096 first                                       │
│    2. If 4096 fails, let OS assign a port (0)                  │
│                                                                 │
│  If port !== 0:                                                 │
│    Use the specified port                                       │
│                                                                 │
│  mDNS is published if:                                          │
│    - mdns option is true                                        │
│    - hostname is not loopback (127.0.0.1, localhost, ::1)      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## OpenAPI Documentation

The server automatically generates OpenAPI specs:

```228:240:packages/opencode/src/server/server.ts
      .get(
        "/doc",
        openAPIRouteHandler(app, {
          documentation: {
            info: {
              title: "opencode",
              version: "0.0.3",
              description: "opencode api",
            },
            openapi: "3.1.1",
          },
        }),
      )
```

Access the API documentation at `/doc` endpoint.

---

## Self-Check Questions

1. **What is the purpose of the `lazy()` wrapper around `createApp`?**
   <details>
   <summary>Answer</summary>
   It ensures the app is only created once (singleton pattern) and defers initialization until first use.
   </details>

2. **Why does the logging middleware skip the `/log` endpoint?**
   <details>
   <summary>Answer</summary>
   To prevent infinite recursion - the `/log` endpoint is used for writing logs, so logging requests to it would create a loop.
   </details>

3. **What happens if a request comes from an origin not in the CORS whitelist?**
   <details>
   <summary>Answer</summary>
   The `origin` function returns `undefined`, which means the CORS middleware won't add the `Access-Control-Allow-Origin` header, and the browser will block the request.
   </details>

4. **How does the server determine which project directory to use?**
   <details>
   <summary>Answer</summary>
   It checks (in order): `?directory` query param, `x-opencode-directory` header, or falls back to `process.cwd()`.
   </details>

5. **What's the difference between `NamedError` and `HTTPException` in error handling?**
   <details>
   <summary>Answer</summary>
   `NamedError` is opencode's custom error class that gets serialized to JSON with appropriate status codes. `HTTPException` is Hono's built-in exception that already has a response attached.
   </details>

---

## Exercises

### Exercise 1: Trace the Middleware Stack

Using the server code, list all middleware in the order they execute for a `GET /session` request. Include what each middleware does.

### Exercise 2: Add a Custom Middleware

Write a middleware that adds a custom header `X-Request-ID` with a unique identifier to every response. Where would you add it in the middleware stack?

### Exercise 3: Understand Route Mounting

Explain what URL paths the following routes would handle:
```typescript
app
  .route("/api", apiRoutes)
  .route("/api/v2", v2Routes)
```
If `apiRoutes` has a handler for `GET /users`, what's the full path?

---

## Further Reading

- [Hono Documentation](https://hono.dev/)
- [Hono OpenAPI](https://github.com/honojs/middleware/tree/main/packages/openapi)
- [Bun HTTP Server](https://bun.sh/docs/api/http)
- `packages/opencode/src/server/server.ts` - Full server implementation
- `packages/opencode/src/server/error.ts` - Error response schemas
