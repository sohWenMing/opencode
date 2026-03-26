# Lesson 02: Routes and Handlers

## Learning Objectives

By the end of this lesson, you will be able to:
- Navigate the routes folder structure
- Understand what each route file handles
- Read and understand route handler patterns
- Trace how routes connect to core business logic
- Implement request validation with Zod schemas

---

## Routes Folder Structure

All HTTP routes are organized in `packages/opencode/src/server/routes/`:

```
packages/opencode/src/server/routes/
├── config.ts        # Configuration management
├── event.ts         # Server-Sent Events (SSE)
├── experimental.ts  # Experimental/beta features
├── file.ts          # File operations and search
├── global.ts        # Global operations (health, config)
├── mcp.ts           # Model Context Protocol integration
├── permission.ts    # Permission request handling
├── project.ts       # Project management
├── provider.ts      # LLM provider operations
├── pty.ts           # Pseudo-terminal sessions
├── question.ts      # User question handling
├── session.ts       # Chat session management
├── tui.ts           # Terminal UI control
└── workspace.ts     # Workspace management
```

---

## Route Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                        HTTP Request                                 │
└─────────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│                     Route Handler Layer                             │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                 │
│  │ describeRoute│  │ validator   │  │ async (c)   │                 │
│  │ (OpenAPI)   │  │ (Zod)       │  │ => handler  │                 │
│  └─────────────┘  └─────────────┘  └─────────────┘                 │
└─────────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│                     Core Business Logic                             │
│  Session, Provider, Config, MCP, Permission, etc.                  │
└─────────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│                     Storage / External Services                     │
│  SQLite Database, File System, LLM APIs, MCP Servers               │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Common Route Patterns

### Pattern 1: Basic Route with OpenAPI Documentation

Every route uses `describeRoute` for OpenAPI documentation and `resolver` for response schemas:

```typescript
import { Hono } from "hono"
import { describeRoute, resolver } from "hono-openapi"
import z from "zod"

new Hono().get(
  "/",
  describeRoute({
    summary: "Short description",
    description: "Detailed description",
    operationId: "unique.operation.id",
    responses: {
      200: {
        description: "Success response",
        content: {
          "application/json": {
            schema: resolver(z.object({ data: z.string() })),
          },
        },
      },
    },
  }),
  async (c) => {
    return c.json({ data: "value" })
  },
)
```

### Pattern 2: Request Validation

Use `validator` to validate params, query, and body:

```typescript
import { validator } from "hono-openapi"

new Hono().post(
  "/:id",
  validator("param", z.object({ id: z.string() })),
  validator("query", z.object({ filter: z.string().optional() })),
  validator("json", z.object({ name: z.string() })),
  async (c) => {
    const { id } = c.req.valid("param")
    const { filter } = c.req.valid("query")
    const { name } = c.req.valid("json")
    // ...
  },
)
```

### Pattern 3: Lazy Route Initialization

Routes use `lazy()` to defer initialization:

```typescript
import { lazy } from "../../util/lazy"

export const MyRoutes = lazy(() =>
  new Hono()
    .get("/", handler1)
    .post("/", handler2)
)
```

---

## Session Routes (`session.ts`)

The largest route file, handling all chat session operations.

### Key Endpoints

| Method | Path | Description |
|--------|------|-------------|
| GET | `/` | List all sessions |
| GET | `/status` | Get status of all sessions |
| GET | `/:sessionID` | Get a specific session |
| POST | `/` | Create a new session |
| DELETE | `/:sessionID` | Delete a session |
| PATCH | `/:sessionID` | Update session (title, archive) |
| POST | `/:sessionID/message` | Send a message (prompt) |
| GET | `/:sessionID/message` | Get session messages |
| POST | `/:sessionID/abort` | Abort active processing |
| POST | `/:sessionID/fork` | Fork session at a message |
| POST | `/:sessionID/revert` | Revert to previous state |
| POST | `/:sessionID/share` | Create shareable link |

### Example: List Sessions

```29:73:packages/opencode/src/server/routes/session.ts
export const SessionRoutes = lazy(() =>
  new Hono()
    .get(
      "/",
      describeRoute({
        summary: "List sessions",
        description: "Get a list of all OpenCode sessions, sorted by most recently updated.",
        operationId: "session.list",
        responses: {
          200: {
            description: "List of sessions",
            content: {
              "application/json": {
                schema: resolver(Session.Info.array()),
              },
            },
          },
        },
      }),
      validator(
        "query",
        z.object({
          directory: z.string().optional().meta({ description: "Filter sessions by project directory" }),
          roots: z.coerce.boolean().optional().meta({ description: "Only return root sessions (no parentID)" }),
          start: z.coerce
            .number()
            .optional()
            .meta({ description: "Filter sessions updated on or after this timestamp (milliseconds since epoch)" }),
          search: z.string().optional().meta({ description: "Filter sessions by title (case-insensitive)" }),
          limit: z.coerce.number().optional().meta({ description: "Maximum number of sessions to return" }),
        }),
      ),
      async (c) => {
        const query = c.req.valid("query")
        const sessions: Session.Info[] = []
        for await (const session of Session.list({
          directory: query.directory,
          roots: query.roots,
          start: query.start,
          search: query.search,
          limit: query.limit,
        })) {
          sessions.push(session)
        }
        return c.json(sessions)
      },
    )
```

### Example: Send Message (Prompt)

```783:823:packages/opencode/src/server/routes/session.ts
    .post(
      "/:sessionID/message",
      describeRoute({
        summary: "Send message",
        description: "Create and send a new message to a session, streaming the AI response.",
        operationId: "session.prompt",
        responses: {
          200: {
            description: "Created message",
            content: {
              "application/json": {
                schema: resolver(
                  z.object({
                    info: MessageV2.Assistant,
                    parts: MessageV2.Part.array(),
                  }),
                ),
              },
            },
          },
          ...errors(400, 404),
        },
      }),
      validator(
        "param",
        z.object({
          sessionID: SessionID.zod,
        }),
      ),
      validator("json", SessionPrompt.PromptInput.omit({ sessionID: true })),
      async (c) => {
        c.status(200)
        c.header("Content-Type", "application/json")
        return stream(c, async (stream) => {
          const sessionID = c.req.valid("param").sessionID
          const body = c.req.valid("json")
          const msg = await SessionPrompt.prompt({ ...body, sessionID })
          stream.write(JSON.stringify(msg))
        })
      },
    )
```

---

## Provider Routes (`provider.ts`)

Manages LLM provider connections and authentication.

### Key Endpoints

| Method | Path | Description |
|--------|------|-------------|
| GET | `/` | List all providers |
| GET | `/auth` | Get auth methods for providers |
| POST | `/:providerID/oauth/authorize` | Start OAuth flow |
| POST | `/:providerID/oauth/callback` | Complete OAuth flow |

### Example: List Providers

```16:65:packages/opencode/src/server/routes/provider.ts
export const ProviderRoutes = lazy(() =>
  new Hono()
    .get(
      "/",
      describeRoute({
        summary: "List providers",
        description: "Get a list of all available AI providers, including both available and connected ones.",
        operationId: "provider.list",
        responses: {
          200: {
            description: "List of providers",
            content: {
              "application/json": {
                schema: resolver(
                  z.object({
                    all: ModelsDev.Provider.array(),
                    default: z.record(z.string(), z.string()),
                    connected: z.array(z.string()),
                  }),
                ),
              },
            },
          },
        },
      }),
      async (c) => {
        const config = await Config.get()
        const disabled = new Set(config.disabled_providers ?? [])
        const enabled = config.enabled_providers ? new Set(config.enabled_providers) : undefined

        const allProviders = await ModelsDev.get()
        const filteredProviders: Record<string, (typeof allProviders)[string]> = {}
        for (const [key, value] of Object.entries(allProviders)) {
          if ((enabled ? enabled.has(key) : true) && !disabled.has(key)) {
            filteredProviders[key] = value
          }
        }

        const connected = await Provider.list()
        const providers = Object.assign(
          mapValues(filteredProviders, (x) => Provider.fromModelsDevProvider(x)),
          connected,
        )
        return c.json({
          all: Object.values(providers),
          default: mapValues(providers, (item) => Provider.sort(Object.values(item.models))[0].id),
          connected: Object.keys(connected),
        })
      },
    )
```

---

## Project Routes (`project.ts`)

Manages project information and git operations.

### Key Endpoints

| Method | Path | Description |
|--------|------|-------------|
| GET | `/` | List all projects |
| GET | `/current` | Get current project |
| POST | `/git/init` | Initialize git repository |
| PATCH | `/:projectID` | Update project settings |

### Example: Get Current Project

```36:56:packages/opencode/src/server/routes/project.ts
    .get(
      "/current",
      describeRoute({
        summary: "Get current project",
        description: "Retrieve the currently active project that OpenCode is working with.",
        operationId: "project.current",
        responses: {
          200: {
            description: "Current project information",
            content: {
              "application/json": {
                schema: resolver(Project.Info),
              },
            },
          },
        },
      }),
      async (c) => {
        return c.json(Instance.project)
      },
    )
```

---

## Config Routes (`config.ts`)

Manages opencode configuration.

### Key Endpoints

| Method | Path | Description |
|--------|------|-------------|
| GET | `/` | Get current configuration |
| PATCH | `/` | Update configuration |
| GET | `/providers` | List configured providers |

### Example: Update Configuration

```36:60:packages/opencode/src/server/routes/config.ts
    .patch(
      "/",
      describeRoute({
        summary: "Update configuration",
        description: "Update OpenCode configuration settings and preferences.",
        operationId: "config.update",
        responses: {
          200: {
            description: "Successfully updated config",
            content: {
              "application/json": {
                schema: resolver(Config.Info),
              },
            },
          },
          ...errors(400),
        },
      }),
      validator("json", Config.Info),
      async (c) => {
        const config = c.req.valid("json")
        await Config.update(config)
        return c.json(config)
      },
    )
```

---

## MCP Routes (`mcp.ts`)

Manages Model Context Protocol server connections.

### Key Endpoints

| Method | Path | Description |
|--------|------|-------------|
| GET | `/` | Get MCP server status |
| POST | `/` | Add new MCP server |
| POST | `/:name/auth` | Start MCP OAuth |
| POST | `/:name/auth/callback` | Complete MCP OAuth |
| POST | `/:name/connect` | Connect to MCP server |
| POST | `/:name/disconnect` | Disconnect from MCP server |
| DELETE | `/:name/auth` | Remove MCP credentials |

### Example: Get MCP Status

```9:31:packages/opencode/src/server/routes/mcp.ts
export const McpRoutes = lazy(() =>
  new Hono()
    .get(
      "/",
      describeRoute({
        summary: "Get MCP status",
        description: "Get the status of all Model Context Protocol (MCP) servers.",
        operationId: "mcp.status",
        responses: {
          200: {
            description: "MCP server status",
            content: {
              "application/json": {
                schema: resolver(z.record(z.string(), MCP.Status)),
              },
            },
          },
        },
      }),
      async (c) => {
        return c.json(await MCP.status())
      },
    )
```

---

## Permission Routes (`permission.ts`)

Handles permission requests from the AI (e.g., file write, shell command).

### Key Endpoints

| Method | Path | Description |
|--------|------|-------------|
| GET | `/` | List pending permissions |
| POST | `/:requestID/reply` | Approve/deny permission |

### Example: Reply to Permission

```9:46:packages/opencode/src/server/routes/permission.ts
export const PermissionRoutes = lazy(() =>
  new Hono()
    .post(
      "/:requestID/reply",
      describeRoute({
        summary: "Respond to permission request",
        description: "Approve or deny a permission request from the AI assistant.",
        operationId: "permission.reply",
        responses: {
          200: {
            description: "Permission processed successfully",
            content: {
              "application/json": {
                schema: resolver(z.boolean()),
              },
            },
          },
          ...errors(400, 404),
        },
      }),
      validator(
        "param",
        z.object({
          requestID: PermissionID.zod,
        }),
      ),
      validator("json", z.object({ reply: Permission.Reply, message: z.string().optional() })),
      async (c) => {
        const params = c.req.valid("param")
        const json = c.req.valid("json")
        await Permission.reply({
          requestID: params.requestID,
          reply: json.reply,
          message: json.message,
        })
        return c.json(true)
      },
    )
```

---

## Question Routes (`question.ts`)

Handles questions from the AI that require user input.

### Key Endpoints

| Method | Path | Description |
|--------|------|-------------|
| GET | `/` | List pending questions |
| POST | `/:requestID/reply` | Answer a question |
| POST | `/:requestID/reject` | Reject a question |

### Example: Reply to Question

```34:68:packages/opencode/src/server/routes/question.ts
    .post(
      "/:requestID/reply",
      describeRoute({
        summary: "Reply to question request",
        description: "Provide answers to a question request from the AI assistant.",
        operationId: "question.reply",
        responses: {
          200: {
            description: "Question answered successfully",
            content: {
              "application/json": {
                schema: resolver(z.boolean()),
              },
            },
          },
          ...errors(400, 404),
        },
      }),
      validator(
        "param",
        z.object({
          requestID: QuestionID.zod,
        }),
      ),
      validator("json", Question.Reply),
      async (c) => {
        const params = c.req.valid("param")
        const json = c.req.valid("json")
        await Question.reply({
          requestID: params.requestID,
          answers: json.answers,
        })
        return c.json(true)
      },
    )
```

---

## Global Routes (`global.ts`)

Server-wide operations not tied to a specific project.

### Key Endpoints

| Method | Path | Description |
|--------|------|-------------|
| GET | `/health` | Health check |
| GET | `/event` | Global event stream (SSE) |
| GET | `/sync-event` | Sync event stream |
| GET | `/config` | Get global config |
| PATCH | `/config` | Update global config |
| POST | `/dispose` | Dispose all instances |
| POST | `/upgrade` | Upgrade opencode |

### Example: Health Check

```70:92:packages/opencode/src/server/routes/global.ts
export const GlobalRoutes = lazy(() =>
  new Hono()
    .get(
      "/health",
      describeRoute({
        summary: "Get health",
        description: "Get health information about the OpenCode server.",
        operationId: "global.health",
        responses: {
          200: {
            description: "Health information",
            content: {
              "application/json": {
                schema: resolver(z.object({ healthy: z.literal(true), version: z.string() })),
              },
            },
          },
        },
      }),
      async (c) => {
        return c.json({ healthy: true, version: Installation.VERSION })
      },
    )
```

---

## File Routes (`file.ts`)

File system operations and search.

### Key Endpoints

| Method | Path | Description |
|--------|------|-------------|
| GET | `/find` | Search text in files (ripgrep) |
| GET | `/find/file` | Find files by name |
| GET | `/find/symbol` | Find workspace symbols |
| GET | `/file` | List files in directory |
| GET | `/file/content` | Read file content |
| GET | `/file/status` | Get git status of files |

---

## PTY Routes (`pty.ts`)

Pseudo-terminal session management for shell access.

### Key Endpoints

| Method | Path | Description |
|--------|------|-------------|
| GET | `/` | List PTY sessions |
| POST | `/` | Create PTY session |
| GET | `/:ptyID` | Get PTY session info |
| PUT | `/:ptyID` | Update PTY session |
| DELETE | `/:ptyID` | Remove PTY session |
| GET | `/:ptyID/connect` | WebSocket connection |

### WebSocket Connection

```134:210:packages/opencode/src/server/routes/pty.ts
    .get(
      "/:ptyID/connect",
      describeRoute({
        summary: "Connect to PTY session",
        description: "Establish a WebSocket connection to interact with a pseudo-terminal (PTY) session in real-time.",
        operationId: "pty.connect",
        // ...
      }),
      validator("param", z.object({ ptyID: PtyID.zod })),
      upgradeWebSocket(async (c) => {
        const id = PtyID.zod.parse(c.req.param("ptyID"))
        // ... WebSocket setup
        return {
          async onOpen(_event, ws) {
            // Connect to PTY
          },
          onMessage(event) {
            // Forward input to PTY
          },
          onClose() {
            // Cleanup
          },
        }
      }),
    )
```

---

## TUI Routes (`tui.ts`)

Control the Terminal User Interface remotely.

### Key Endpoints

| Method | Path | Description |
|--------|------|-------------|
| POST | `/append-prompt` | Append text to prompt |
| POST | `/open-help` | Open help dialog |
| POST | `/open-sessions` | Open sessions dialog |
| POST | `/open-models` | Open models dialog |
| POST | `/submit-prompt` | Submit current prompt |
| POST | `/clear-prompt` | Clear prompt |
| POST | `/execute-command` | Execute TUI command |
| POST | `/show-toast` | Show toast notification |
| POST | `/select-session` | Navigate to session |

---

## Request/Response Flow

```
┌──────────────────────────────────────────────────────────────────────┐
│                         Client Request                               │
│  POST /session/abc123/message                                        │
│  Body: { "parts": [...], "providerID": "anthropic", ... }           │
└──────────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌──────────────────────────────────────────────────────────────────────┐
│                      Route Matching                                  │
│  SessionRoutes → /:sessionID/message → POST handler                 │
└──────────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌──────────────────────────────────────────────────────────────────────┐
│                      Validation Layer                                │
│  validator("param", { sessionID: SessionID.zod })                   │
│  validator("json", SessionPrompt.PromptInput)                       │
└──────────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌──────────────────────────────────────────────────────────────────────┐
│                      Handler Execution                               │
│  const sessionID = c.req.valid("param").sessionID                   │
│  const body = c.req.valid("json")                                   │
│  const msg = await SessionPrompt.prompt({ ...body, sessionID })     │
└──────────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌──────────────────────────────────────────────────────────────────────┐
│                      Core Logic                                      │
│  SessionPrompt.prompt() → AI processing → Message creation          │
└──────────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌──────────────────────────────────────────────────────────────────────┐
│                      Response                                        │
│  return c.json({ info: MessageV2.Assistant, parts: [...] })         │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Error Response Patterns

The `errors()` helper standardizes error responses:

```1:36:packages/opencode/src/server/error.ts
import { resolver } from "hono-openapi"
import z from "zod"
import { NotFoundError } from "../storage/db"

export const ERRORS = {
  400: {
    description: "Bad request",
    content: {
      "application/json": {
        schema: resolver(
          z
            .object({
              data: z.any(),
              errors: z.array(z.record(z.string(), z.any())),
              success: z.literal(false),
            })
            .meta({
              ref: "BadRequestError",
            }),
        ),
      },
    },
  },
  404: {
    description: "Not found",
    content: {
      "application/json": {
        schema: resolver(NotFoundError.Schema),
      },
    },
  },
} as const

export function errors(...codes: number[]) {
  return Object.fromEntries(codes.map((code) => [code, ERRORS[code as keyof typeof ERRORS]]))
}
```

Usage in routes:

```typescript
responses: {
  200: { /* success */ },
  ...errors(400, 404),  // Adds 400 and 404 error responses
}
```

---

## Self-Check Questions

1. **What is the purpose of `describeRoute` in route definitions?**
   <details>
   <summary>Answer</summary>
   It provides OpenAPI documentation metadata including summary, description, operation ID, and response schemas that are used to generate API documentation.
   </details>

2. **How do you access validated request data in a handler?**
   <details>
   <summary>Answer</summary>
   Use `c.req.valid("param")`, `c.req.valid("query")`, or `c.req.valid("json")` to access validated data from params, query string, or request body respectively.
   </details>

3. **Why do routes use the `lazy()` wrapper?**
   <details>
   <summary>Answer</summary>
   To defer route initialization until first use, which helps with startup performance and ensures dependencies are properly initialized.
   </details>

4. **What's the difference between `/session` and `/experimental/session` endpoints?**
   <details>
   <summary>Answer</summary>
   `/session` routes are scoped to the current project/instance, while `/experimental/session` lists sessions across all projects globally.
   </details>

5. **How does the PTY route handle real-time communication?**
   <details>
   <summary>Answer</summary>
   It uses WebSocket via `upgradeWebSocket()` to establish a bidirectional connection for real-time terminal I/O.
   </details>

---

## Exercises

### Exercise 1: Trace a Request

Trace what happens when you call `GET /session?limit=10&search=test`:
1. Which route file handles this?
2. What validation occurs?
3. What core function is called?
4. What does the response look like?

### Exercise 2: Identify Route Patterns

Find three different routes that use streaming responses. What do they have in common?

### Exercise 3: Schema Analysis

Look at the `Session.Info` schema used in session routes. What fields does it contain? How is it defined?

---

## Further Reading

- `packages/opencode/src/server/routes/` - All route implementations
- `packages/opencode/src/session/index.ts` - Session business logic
- `packages/opencode/src/provider/provider.ts` - Provider business logic
- [Hono Validation](https://hono.dev/docs/guides/validation)
- [Zod Documentation](https://zod.dev/)
