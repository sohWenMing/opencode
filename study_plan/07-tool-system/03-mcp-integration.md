# Lesson 03: MCP Integration

## Learning Objectives

By the end of this lesson, you will be able to:
- Explain what MCP (Model Context Protocol) is
- Understand how opencode integrates MCP tools
- Describe the MCP folder structure and client setup
- Explain how MCP tools are merged with built-in tools
- Understand MCP resources and prompts

## What is MCP?

**Model Context Protocol (MCP)** is an open standard for connecting AI assistants to external tools and data sources. It provides a standardized way for:

- **Tools**: Actions the AI can perform (like built-in tools)
- **Resources**: Data the AI can read (files, databases, APIs)
- **Prompts**: Pre-defined prompt templates

```
┌─────────────────────────────────────────────────────────────────────┐
│                         opencode                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                      AI Agent                                │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                              │                                      │
│              ┌───────────────┼───────────────┐                     │
│              ▼               ▼               ▼                     │
│  ┌───────────────┐  ┌───────────────┐  ┌───────────────┐          │
│  │  Built-in     │  │  MCP Server   │  │  MCP Server   │          │
│  │  Tools        │  │  (Local)      │  │  (Remote)     │          │
│  │  • read       │  │  • custom     │  │  • cloud API  │          │
│  │  • bash       │  │    tools      │  │    tools      │          │
│  │  • edit       │  │               │  │               │          │
│  └───────────────┘  └───────────────┘  └───────────────┘          │
└─────────────────────────────────────────────────────────────────────┘
```

## MCP Configuration

MCP servers are configured in `opencode.json`:

```json
{
  "mcp": {
    "my-local-server": {
      "type": "local",
      "command": ["npx", "my-mcp-server"],
      "environment": {
        "API_KEY": "..."
      }
    },
    "my-remote-server": {
      "type": "remote",
      "url": "https://mcp.example.com",
      "oauth": {
        "clientId": "...",
        "scope": "read write"
      }
    }
  }
}
```

### Configuration Types

**Local MCP Server:**
```typescript
{
  type: "local",
  command: string[],      // Command to start the server
  environment?: Record<string, string>,  // Environment variables
  timeout?: number,       // Connection timeout
  enabled?: boolean,      // Enable/disable
}
```

**Remote MCP Server:**
```typescript
{
  type: "remote",
  url: string,            // Server URL
  headers?: Record<string, string>,  // HTTP headers
  oauth?: {               // OAuth configuration
    clientId?: string,
    clientSecret?: string,
    scope?: string,
  } | false,              // false to disable OAuth
  timeout?: number,
  enabled?: boolean,
}
```

## MCP Folder Structure

The MCP integration lives in `packages/opencode/src/mcp/`:

```
packages/opencode/src/mcp/
├── index.ts           # Main MCP namespace and service
├── auth.ts            # OAuth token management
├── oauth-provider.ts  # OAuth provider implementation
└── oauth-callback.ts  # OAuth callback server
```

## MCP Client Setup

The MCP namespace in `packages/opencode/src/mcp/index.ts` manages all MCP connections.

### Status Types

```typescript
export const Status = z.discriminatedUnion("status", [
  z.object({ status: z.literal("connected") }),
  z.object({ status: z.literal("disabled") }),
  z.object({ status: z.literal("failed"), error: z.string() }),
  z.object({ status: z.literal("needs_auth") }),
  z.object({ status: z.literal("needs_client_registration"), error: z.string() }),
])
```

### Creating MCP Clients

```typescript
async function create(key: string, mcp: Config.Mcp) {
  if (mcp.enabled === false) {
    return { mcpClient: undefined, status: { status: "disabled" } }
  }

  if (mcp.type === "remote") {
    // Set up OAuth if needed
    const authProvider = !oauthDisabled 
      ? new McpOAuthProvider(key, mcp.url, oauthConfig)
      : undefined

    // Try StreamableHTTP first, then SSE
    const transports = [
      { name: "StreamableHTTP", transport: new StreamableHTTPClientTransport(...) },
      { name: "SSE", transport: new SSEClientTransport(...) },
    ]

    for (const { name, transport } of transports) {
      try {
        const client = new Client({ name: "opencode", version: Installation.VERSION })
        await withTimeout(client.connect(transport), connectTimeout)
        return { mcpClient: client, status: { status: "connected" } }
      } catch (error) {
        if (error instanceof UnauthorizedError) {
          return { mcpClient: undefined, status: { status: "needs_auth" } }
        }
      }
    }
  }

  if (mcp.type === "local") {
    const [cmd, ...args] = mcp.command
    const transport = new StdioClientTransport({
      command: cmd,
      args,
      cwd: Instance.directory,
      env: { ...process.env, ...mcp.environment },
    })

    const client = new Client({ name: "opencode", version: Installation.VERSION })
    await withTimeout(client.connect(transport), connectTimeout)
    return { mcpClient: client, status: { status: "connected" } }
  }
}
```

## Converting MCP Tools to AI SDK Tools

MCP tools use a different format than the AI SDK. The `convertMcpTool` function bridges this gap:

```typescript
function convertMcpTool(mcpTool: MCPToolDef, client: MCPClient, timeout?: number): Tool {
  const inputSchema = mcpTool.inputSchema

  // Ensure schema is always an object type
  const schema: JSONSchema7 = {
    ...inputSchema,
    type: "object",
    properties: inputSchema.properties ?? {},
    additionalProperties: false,
  }

  return dynamicTool({
    description: mcpTool.description ?? "",
    inputSchema: jsonSchema(schema),
    execute: async (args: unknown) => {
      return client.callTool(
        {
          name: mcpTool.name,
          arguments: (args || {}) as Record<string, unknown>,
        },
        CallToolResultSchema,
        {
          resetTimeoutOnProgress: true,
          timeout,
        },
      )
    },
  })
}
```

## MCP Service Interface

The MCP service provides these operations:

```typescript
export interface Interface {
  // Connection management
  readonly status: () => Effect.Effect<Record<string, Status>>
  readonly clients: () => Effect.Effect<Record<string, MCPClient>>
  readonly add: (name: string, mcp: Config.Mcp) => Effect.Effect<{ status: Record<string, Status> }>
  readonly connect: (name: string) => Effect.Effect<void>
  readonly disconnect: (name: string) => Effect.Effect<void>

  // Tools, prompts, resources
  readonly tools: () => Effect.Effect<Record<string, Tool>>
  readonly prompts: () => Effect.Effect<Record<string, PromptInfo & { client: string }>>
  readonly resources: () => Effect.Effect<Record<string, ResourceInfo & { client: string }>>

  // Resource/prompt access
  readonly getPrompt: (clientName: string, name: string, args?: Record<string, string>) => Effect.Effect<...>
  readonly readResource: (clientName: string, resourceUri: string) => Effect.Effect<...>

  // OAuth
  readonly startAuth: (mcpName: string) => Effect.Effect<{ authorizationUrl: string; oauthState: string }>
  readonly authenticate: (mcpName: string) => Effect.Effect<Status>
  readonly finishAuth: (mcpName: string, authorizationCode: string) => Effect.Effect<Status>
  readonly removeAuth: (mcpName: string) => Effect.Effect<void>
}
```

## Getting MCP Tools

The `tools()` function collects tools from all connected MCP servers:

```typescript
const tools = Effect.fn("MCP.tools")(function* () {
  const result: Record<string, Tool> = {}
  const s = yield* InstanceState.get(cache)
  const cfg = yield* Effect.promise(() => Config.get())
  const defaultTimeout = cfg.experimental?.mcp_timeout

  // Only get tools from connected clients
  const connectedClients = Object.entries(s.clients).filter(
    ([clientName]) => s.status[clientName]?.status === "connected",
  )

  yield* Effect.forEach(
    connectedClients,
    ([clientName, client]) =>
      Effect.gen(function* () {
        const mcpConfig = config[clientName]
        const timeout = mcpConfig?.timeout ?? defaultTimeout

        const listed = s.defs[clientName]
        if (!listed) return

        for (const mcpTool of listed) {
          // Sanitize names for tool IDs
          const sanitizedClientName = clientName.replace(/[^a-zA-Z0-9_-]/g, "_")
          const sanitizedToolName = mcpTool.name.replace(/[^a-zA-Z0-9_-]/g, "_")
          
          result[sanitizedClientName + "_" + sanitizedToolName] = convertMcpTool(mcpTool, client, timeout)
        }
      }),
    { concurrency: "unbounded" },
  )
  
  return result
})
```

### Tool Naming Convention

MCP tools are named: `{serverName}_{toolName}`

For example:
- Server: `my-database`
- Tool: `query`
- Result: `my_database_query`

## MCP Resources

Resources are read-only data that MCP servers expose:

```typescript
export const Resource = z.object({
  name: z.string(),
  uri: z.string(),
  description: z.string().optional(),
  mimeType: z.string().optional(),
  client: z.string(),
})
```

### Reading Resources

```typescript
const readResource = Effect.fn("MCP.readResource")(function* (clientName: string, resourceUri: string) {
  const s = yield* InstanceState.get(cache)
  const client = s.clients[clientName]
  if (!client) return undefined
  
  return yield* Effect.tryPromise({
    try: () => client.readResource({ uri: resourceUri }),
    catch: (e) => {
      log.error("failed to read resource", { clientName, resourceUri, error: e?.message })
      return e
    },
  }).pipe(Effect.orElseSucceed(() => undefined))
})
```

## MCP Prompts

Prompts are pre-defined templates that MCP servers provide:

```typescript
const getPrompt = Effect.fn("MCP.getPrompt")(function* (
  clientName: string,
  name: string,
  args?: Record<string, string>,
) {
  const s = yield* InstanceState.get(cache)
  const client = s.clients[clientName]
  if (!client) return undefined
  
  return yield* Effect.tryPromise({
    try: () => client.getPrompt({ name, arguments: args }),
    catch: (e) => {
      log.error("failed to get prompt", { clientName, promptName: name, error: e?.message })
      return e
    },
  }).pipe(Effect.orElseSucceed(() => undefined))
})
```

## OAuth Authentication

Many remote MCP servers require OAuth authentication.

### OAuth Flow

```
┌─────────────────────────────────────────────────────────────────────┐
│                      OAuth Authentication Flow                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  1. User runs: opencode mcp auth my-server                          │
│                        │                                             │
│                        ▼                                             │
│  2. startAuth() generates state and authorization URL               │
│                        │                                             │
│                        ▼                                             │
│  3. Browser opens authorization URL                                  │
│                        │                                             │
│                        ▼                                             │
│  4. User authorizes in browser                                       │
│                        │                                             │
│                        ▼                                             │
│  5. OAuth callback server receives authorization code               │
│                        │                                             │
│                        ▼                                             │
│  6. finishAuth() exchanges code for tokens                          │
│                        │                                             │
│                        ▼                                             │
│  7. Tokens stored in mcp-auth.json                                  │
│                        │                                             │
│                        ▼                                             │
│  8. MCP client reconnects with tokens                               │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### OAuth Token Storage

From `packages/opencode/src/mcp/auth.ts`:

```typescript
export namespace McpAuth {
  export const Tokens = z.object({
    accessToken: z.string(),
    refreshToken: z.string().optional(),
    expiresAt: z.number().optional(),
    scope: z.string().optional(),
  })

  export const Entry = z.object({
    tokens: Tokens.optional(),
    clientInfo: ClientInfo.optional(),
    codeVerifier: z.string().optional(),
    oauthState: z.string().optional(),
    serverUrl: z.string().optional(),
  })

  // Stored in ~/.opencode/mcp-auth.json
  const filepath = path.join(Global.Path.data, "mcp-auth.json")
}
```

## Tool Change Notifications

MCP servers can notify when their tools change:

```typescript
function watch(s: State, name: string, client: MCPClient, timeout?: number) {
  client.setNotificationHandler(ToolListChangedNotificationSchema, async () => {
    log.info("tools list changed notification received", { server: name })
    
    // Verify client is still connected
    if (s.clients[name] !== client || s.status[name]?.status !== "connected") return

    // Refresh tool list
    const listed = await defs(name, client, timeout)
    if (!listed) return
    
    s.defs[name] = listed
    
    // Notify the rest of the system
    await Bus.publish(ToolsChanged, { server: name })
  })
}
```

## Merging MCP Tools with Built-in Tools

In the session prompt, MCP tools are merged with built-in tools:

```typescript
// From session prompt code (simplified)
const builtinTools = await ToolRegistry.tools(model, agent)
const mcpTools = await MCP.tools()

const allTools = {
  ...Object.fromEntries(builtinTools.map(t => [t.id, t])),
  ...mcpTools,  // MCP tools are added with their prefixed names
}
```

## Async Facade Functions

The MCP namespace exports async functions for easy use:

```typescript
export const status = async () => runPromise((svc) => svc.status())
export const clients = async () => runPromise((svc) => svc.clients())
export const tools = async () => runPromise((svc) => svc.tools())
export const prompts = async () => runPromise((svc) => svc.prompts())
export const resources = async () => runPromise((svc) => svc.resources())
export const add = async (name: string, mcp: Config.Mcp) => runPromise((svc) => svc.add(name, mcp))
export const connect = async (name: string) => runPromise((svc) => svc.connect(name))
export const disconnect = async (name: string) => runPromise((svc) => svc.disconnect(name))
export const authenticate = async (mcpName: string) => runPromise((svc) => svc.authenticate(mcpName))
```

## Self-Check Questions

1. What are the two types of MCP server configurations?
2. How are MCP tool names constructed?
3. What happens when an MCP server requires OAuth but the user hasn't authenticated?
4. How does opencode handle MCP tool list changes?
5. Where are OAuth tokens stored?

## Exercises

### Exercise 1: Configure an MCP Server
Write the configuration for a local MCP server that:
- Runs `npx @modelcontextprotocol/server-filesystem`
- Has access to the `/tmp` directory
- Has a 60-second timeout

### Exercise 2: Trace Tool Discovery
Given an MCP server named "github" with tools "create_issue" and "list_repos", what would be the final tool names in opencode?

### Exercise 3: OAuth Flow Analysis
Draw a sequence diagram showing the OAuth flow when a user runs `opencode mcp auth my-server` for a server that requires authentication.

## Further Reading

- `packages/opencode/src/mcp/index.ts` - Main MCP integration
- `packages/opencode/src/mcp/auth.ts` - OAuth token management
- `packages/opencode/src/mcp/oauth-provider.ts` - OAuth provider
- MCP Specification: https://modelcontextprotocol.io/
- `@modelcontextprotocol/sdk` package documentation
