# Module 09, Lesson 01: Plugin System

## Learning Objectives

By the end of this lesson, you will be able to:

1. Understand what plugins are in opencode and their purpose
2. Navigate the plugin folder structure
3. Identify plugin hooks and their lifecycle
4. Create custom plugins that extend opencode functionality
5. Configure plugins in opencode

## What Are Plugins in opencode?

Plugins are extensions that allow you to customize and extend opencode's behavior without modifying the core codebase. They provide a powerful mechanism for:

- **Authentication**: Adding custom OAuth flows for providers (Codex, Copilot, GitLab, Poe)
- **Custom Tools**: Defining new tools the AI can use
- **Request Modification**: Intercepting and modifying LLM requests/responses
- **Event Handling**: Reacting to system events
- **Permission Control**: Customizing permission behavior

```
┌─────────────────────────────────────────────────────────────────┐
│                        opencode Core                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────┐   ┌─────────────┐   ┌─────────────┐           │
│  │   Session   │   │   Provider  │   │    Tool     │           │
│  │   System    │   │   System    │   │   System    │           │
│  └──────┬──────┘   └──────┬──────┘   └──────┬──────┘           │
│         │                 │                 │                   │
│         └─────────────────┼─────────────────┘                   │
│                           │                                     │
│                    ┌──────▼──────┐                              │
│                    │   Plugin    │                              │
│                    │   System    │                              │
│                    └──────┬──────┘                              │
│                           │                                     │
├───────────────────────────┼─────────────────────────────────────┤
│                           │                                     │
│  ┌────────────┐  ┌────────▼───────┐  ┌────────────┐            │
│  │   Codex    │  │    Copilot     │  │   Custom   │            │
│  │   Plugin   │  │    Plugin      │  │   Plugins  │            │
│  └────────────┘  └────────────────┘  └────────────┘            │
│                                                                 │
│                    Plugin Layer                                 │
└─────────────────────────────────────────────────────────────────┘
```

## Plugin Folder Structure

The plugin system is organized across two locations:

### 1. Plugin SDK (`packages/plugin/`)

This package provides the types and utilities for creating plugins:

```
packages/plugin/
├── src/
│   ├── index.ts      # Main exports and type definitions
│   ├── tool.ts       # Tool creation utilities
│   ├── shell.ts      # Shell type definitions
│   └── example.ts    # Example plugin implementation
├── package.json
└── tsconfig.json
```

### 2. Built-in Plugins (`packages/opencode/src/plugin/`)

These are plugins bundled directly with opencode:

```
packages/opencode/src/plugin/
├── index.ts          # Plugin system core (loading, triggering)
├── codex.ts          # OpenAI Codex OAuth authentication
└── copilot.ts        # GitHub Copilot authentication
```

## Plugin Types and Interfaces

### The Plugin Function

Every plugin is a function that receives context and returns hooks:

```typescript
// From packages/plugin/src/index.ts

export type PluginInput = {
  client: ReturnType<typeof createOpencodeClient>  // SDK client
  project: Project                                   // Project info
  directory: string                                  // Working directory
  worktree: string                                   // Git worktree root
  serverUrl: URL                                     // Server URL
  $: BunShell                                        // Bun shell for commands
}

export type Plugin = (input: PluginInput) => Promise<Hooks>
```

### The Hooks Interface

Hooks define what your plugin can intercept and modify:

```typescript
// From packages/plugin/src/index.ts

export interface Hooks {
  // React to system events
  event?: (input: { event: Event }) => Promise<void>
  
  // Called when config changes
  config?: (input: Config) => Promise<void>
  
  // Define custom tools
  tool?: {
    [key: string]: ToolDefinition
  }
  
  // Custom authentication flows
  auth?: AuthHook
  
  // Intercept new messages
  "chat.message"?: (input, output) => Promise<void>
  
  // Modify LLM parameters
  "chat.params"?: (input, output) => Promise<void>
  
  // Add custom headers to LLM requests
  "chat.headers"?: (input, output) => Promise<void>
  
  // Customize permission behavior
  "permission.ask"?: (input, output) => Promise<void>
  
  // Before/after tool execution
  "tool.execute.before"?: (input, output) => Promise<void>
  "tool.execute.after"?: (input, output) => Promise<void>
  
  // Modify shell environment
  "shell.env"?: (input, output) => Promise<void>
  
  // Experimental hooks for advanced customization
  "experimental.chat.messages.transform"?: (input, output) => Promise<void>
  "experimental.chat.system.transform"?: (input, output) => Promise<void>
  "experimental.session.compacting"?: (input, output) => Promise<void>
}
```

## Plugin Lifecycle

```
┌─────────────────────────────────────────────────────────────────┐
│                     Plugin Lifecycle                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. INITIALIZATION                                              │
│     ┌──────────────────────────────────────────────────────┐   │
│     │  • Load internal plugins (Codex, Copilot, etc.)      │   │
│     │  • Install npm plugins from config                    │   │
│     │  • Call plugin functions with PluginInput             │   │
│     │  • Collect returned Hooks                             │   │
│     └──────────────────────────────────────────────────────┘   │
│                           │                                     │
│                           ▼                                     │
│  2. CONFIG NOTIFICATION                                         │
│     ┌──────────────────────────────────────────────────────┐   │
│     │  • Call hook.config(cfg) for each plugin             │   │
│     └──────────────────────────────────────────────────────┘   │
│                           │                                     │
│                           ▼                                     │
│  3. EVENT SUBSCRIPTION                                          │
│     ┌──────────────────────────────────────────────────────┐   │
│     │  • Subscribe to Bus events                           │   │
│     │  • Forward events to hook.event() handlers           │   │
│     └──────────────────────────────────────────────────────┘   │
│                           │                                     │
│                           ▼                                     │
│  4. RUNTIME TRIGGERS                                            │
│     ┌──────────────────────────────────────────────────────┐   │
│     │  • Plugin.trigger("hook.name", input, output)        │   │
│     │  • Each plugin's hook is called in sequence          │   │
│     │  • Hooks can modify the output object                │   │
│     └──────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## Plugin System Core Implementation

The plugin system is implemented as an Effect service:

```typescript
// From packages/opencode/src/plugin/index.ts

export namespace Plugin {
  // Built-in plugins that are directly imported
  const INTERNAL_PLUGINS: PluginInstance[] = [
    CodexAuthPlugin, 
    CopilotAuthPlugin, 
    GitlabAuthPlugin, 
    PoeAuthPlugin
  ]

  // Old npm packages now built-in — skip if users still have them
  const DEPRECATED_PLUGIN_PACKAGES = [
    "opencode-openai-codex-auth", 
    "opencode-copilot-auth"
  ]

  export const layer = Layer.effect(
    Service,
    Effect.gen(function* () {
      const bus = yield* Bus.Service

      const cache = yield* InstanceState.make<State>(
        Effect.fn("Plugin.state")(function* (ctx) {
          const hooks: Hooks[] = []

          yield* Effect.promise(async () => {
            // Create SDK client for plugins
            const client = createOpencodeClient({
              baseUrl: "http://localhost:4096",
              directory: ctx.directory,
              // ...
            })
            
            const input: PluginInput = {
              client,
              project: ctx.project,
              worktree: ctx.worktree,
              directory: ctx.directory,
              serverUrl: Server.url ?? new URL("http://localhost:4096"),
              $: Bun.$,
            }

            // Load internal plugins
            for (const plugin of INTERNAL_PLUGINS) {
              log.info("loading internal plugin", { name: plugin.name })
              const init = await plugin(input).catch((err) => {
                log.error("failed to load internal plugin", { name: plugin.name })
              })
              if (init) hooks.push(init)
            }

            // Load npm plugins from config
            let plugins = cfg.plugin ?? []
            for (let plugin of plugins) {
              // Skip deprecated packages
              if (DEPRECATED_PLUGIN_PACKAGES.some((pkg) => plugin.includes(pkg))) 
                continue
              
              // Install and import plugin
              plugin = await BunProc.install(pkg, version)
              const mod = await import(plugin)
              for (const [_name, fn] of Object.entries<PluginInstance>(mod)) {
                hooks.push(await fn(input))
              }
            }

            // Notify plugins of current config
            for (const hook of hooks) {
              await hook.config?.(cfg)
            }
          })

          // Subscribe to bus events
          yield* bus.subscribeAll().pipe(
            Stream.runForEach((input) =>
              Effect.sync(() => {
                for (const hook of hooks) {
                  hook["event"]?.({ event: input as any })
                }
              }),
            ),
            Effect.forkScoped,
          )

          return { hooks }
        }),
      )

      // Trigger mechanism for hooks
      const trigger = Effect.fn("Plugin.trigger")(function* <Name, Input, Output>(
        name: Name, 
        input: Input, 
        output: Output
      ) {
        const state = yield* InstanceState.get(cache)
        for (const hook of state.hooks) {
          const fn = hook[name] as any
          if (!fn) continue
          await fn(input, output)
        }
        return output
      })

      return Service.of({ trigger, list, init })
    }),
  )
}
```

## Creating Custom Tools

Plugins can define custom tools using the `tool` helper:

```typescript
// From packages/plugin/src/tool.ts

export type ToolContext = {
  sessionID: string
  messageID: string
  agent: string
  directory: string      // Current project directory
  worktree: string       // Git worktree root
  abort: AbortSignal     // For cancellation
  metadata(input: { title?: string; metadata?: any }): void
  ask(input: AskInput): Promise<void>  // Request permissions
}

export function tool<Args extends z.ZodRawShape>(input: {
  description: string
  args: Args
  execute(args: z.infer<z.ZodObject<Args>>, context: ToolContext): Promise<string>
}) {
  return input
}
tool.schema = z  // Zod for schema definition
```

### Example Plugin with Custom Tool

```typescript
// From packages/plugin/src/example.ts

import { Plugin } from "./index.js"
import { tool } from "./tool.js"

export const ExamplePlugin: Plugin = async (ctx) => {
  return {
    tool: {
      mytool: tool({
        description: "This is a custom tool",
        args: {
          foo: tool.schema.string().describe("foo"),
        },
        async execute(args) {
          return `Hello ${args.foo}!`
        },
      }),
    },
  }
}
```

## Built-in Plugin: Codex Authentication

The Codex plugin demonstrates a complex authentication flow:

```typescript
// From packages/opencode/src/plugin/codex.ts

export async function CodexAuthPlugin(input: PluginInput): Promise<Hooks> {
  return {
    auth: {
      provider: "openai",
      async loader(getAuth, provider) {
        const auth = await getAuth()
        if (auth.type !== "oauth") return {}

        // Filter models to only allowed Codex models
        const allowedModels = new Set([
          "gpt-5.1-codex",
          "gpt-5.1-codex-max",
          // ...
        ])
        
        // Zero out costs (included with ChatGPT subscription)
        for (const model of Object.values(provider.models)) {
          model.cost = { input: 0, output: 0, cache: { read: 0, write: 0 } }
        }

        return {
          apiKey: OAUTH_DUMMY_KEY,
          async fetch(requestInput, init) {
            // Handle token refresh
            if (!currentAuth.access || currentAuth.expires < Date.now()) {
              const tokens = await refreshAccessToken(currentAuth.refresh)
              // Update stored auth...
            }
            
            // Rewrite URL to Codex endpoint
            const url = parsed.pathname.includes("/v1/responses")
              ? new URL(CODEX_API_ENDPOINT)
              : parsed

            return fetch(url, { ...init, headers })
          },
        }
      },
      methods: [
        {
          label: "ChatGPT Pro/Plus (browser)",
          type: "oauth",
          authorize: async () => {
            // Start OAuth server, generate PKCE codes
            const { redirectUri } = await startOAuthServer()
            const pkce = await generatePKCE()
            const authUrl = buildAuthorizeUrl(redirectUri, pkce, state)
            
            return {
              url: authUrl,
              instructions: "Complete authorization in your browser.",
              method: "auto",
              callback: async () => {
                const tokens = await callbackPromise
                return { type: "success", refresh: tokens.refresh_token, ... }
              },
            }
          },
        },
      ],
    },
    // Add custom headers to requests
    "chat.headers": async (input, output) => {
      if (input.model.providerID !== "openai") return
      output.headers.originator = "opencode"
      output.headers["User-Agent"] = `opencode/${VERSION}`
    },
  }
}
```

## Plugin Configuration

Plugins are configured in `opencode.json`:

```json
{
  "plugin": [
    "my-custom-plugin@1.0.0",
    "file:///path/to/local/plugin"
  ]
}
```

### Configuration Options

| Option | Description |
|--------|-------------|
| `package@version` | Install from npm with specific version |
| `package` | Install latest version from npm |
| `file:///path` | Load from local file path |

## Hook Trigger Pattern

All hooks follow a consistent `(input, output) => Promise<void>` pattern:

```typescript
// Example: Modifying chat parameters
"chat.params"?: (
  input: { 
    sessionID: string
    agent: string
    model: Model
    provider: ProviderContext
    message: UserMessage 
  },
  output: { 
    temperature: number
    topP: number
    topK: number
    options: Record<string, any> 
  },
) => Promise<void>
```

The plugin modifies the `output` object in place:

```typescript
"chat.params": async (input, output) => {
  // Lower temperature for code generation
  if (input.agent === "build") {
    output.temperature = 0.2
  }
}
```

## Self-Check Questions

1. What is the difference between internal plugins and npm plugins?
2. How does the plugin trigger mechanism work?
3. What context is provided to custom tools?
4. How do authentication plugins modify fetch requests?
5. What is the purpose of the `PluginInput` object?

## Exercises

### Exercise 1: Trace Plugin Loading

1. Add logging to `packages/opencode/src/plugin/index.ts`
2. Start opencode and observe the plugin loading sequence
3. Document which plugins load and in what order

### Exercise 2: Create a Simple Plugin

Create a plugin that:
1. Logs all events to a file
2. Adds a custom header to all requests
3. Defines a simple tool that returns the current time

### Exercise 3: Analyze the Codex Plugin

1. Read through `packages/opencode/src/plugin/codex.ts`
2. Diagram the OAuth flow
3. Identify how token refresh is handled

## Further Reading

- [Effect-TS Documentation](https://effect.website/) - Understanding the Effect patterns used
- [Zod Documentation](https://zod.dev/) - Schema validation for tool arguments
- [OAuth 2.0 PKCE](https://oauth.net/2/pkce/) - The authentication flow used by Codex
