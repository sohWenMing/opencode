# Lesson 02: Solid.js for Terminal UI

## Learning Objectives

By the end of this lesson, you will be able to:
- Understand what Solid.js is and its core principles
- Explain why Solid.js is used for a terminal UI
- Navigate the TUI folder structure
- Understand Solid's reactive primitives (signals, effects, memos)
- Identify how components, routes, and context work together

## What is Solid.js?

Solid.js is a reactive JavaScript framework for building user interfaces. Unlike React, Solid:

- **Has no Virtual DOM**: Updates the actual DOM directly
- **Uses fine-grained reactivity**: Only the specific parts that change are updated
- **Compiles away**: The framework disappears at build time
- **Is extremely fast**: Minimal runtime overhead

```
┌─────────────────────────────────────────────────────────────┐
│                    React (Virtual DOM)                      │
│  State Change → Re-render Component → Diff VDOM → Update    │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                 Solid.js (Fine-grained)                     │
│  Signal Change → Update Only Subscribed Elements            │
└─────────────────────────────────────────────────────────────┘
```

## Why Solid for a Terminal UI?

The terminal UI needs to be:

1. **Responsive**: Updates must be immediate for good UX
2. **Efficient**: Terminal rendering is expensive
3. **Predictable**: State changes should have clear effects

Solid's fine-grained reactivity is perfect because:
- Only changed parts of the UI re-render
- No diffing overhead
- Direct updates to terminal cells
- Minimal memory allocation

## TUI Folder Structure

The TUI code lives in `packages/opencode/src/cli/cmd/tui/`:

```
packages/opencode/src/cli/cmd/tui/
├── app.tsx                 # Main application component
├── thread.ts               # Entry point (starts the TUI)
├── worker.ts               # Background worker for server
├── event.ts                # Custom TUI events
├── win32.ts                # Windows-specific handling
│
├── context/                # Solid context providers
│   ├── args.tsx            # CLI arguments context
│   ├── directory.tsx       # Current directory context
│   ├── exit.tsx            # Exit handling context
│   ├── helper.tsx          # Context creation helper
│   ├── keybind.tsx         # Keyboard shortcuts context
│   ├── kv.tsx              # Key-value storage context
│   ├── local.tsx           # Local state (model, agent)
│   ├── prompt.tsx          # Prompt reference context
│   ├── route.tsx           # Navigation/routing context
│   ├── sdk.tsx             # SDK client context
│   ├── sync.tsx            # Server sync context
│   ├── theme.tsx           # Theme/colors context
│   └── tui-config.tsx      # TUI configuration context
│
├── routes/                 # Page components
│   ├── home.tsx            # Home screen
│   └── session/            # Session view
│       ├── index.tsx       # Main session component
│       ├── header.tsx      # Session header
│       ├── footer.tsx      # Session footer
│       ├── sidebar.tsx     # Session sidebar
│       └── ...
│
├── component/              # Reusable components
│   ├── border.tsx          # Border styling
│   ├── logo.tsx            # OpenCode logo
│   ├── spinner.tsx         # Loading spinner
│   ├── tips.tsx            # Tips display
│   ├── todo-item.tsx       # Todo list item
│   ├── prompt/             # Prompt input components
│   │   ├── index.tsx       # Main prompt component
│   │   ├── autocomplete.tsx
│   │   ├── history.tsx
│   │   └── ...
│   └── dialog-*.tsx        # Various dialog components
│
├── ui/                     # Low-level UI components
│   ├── dialog.tsx          # Dialog system
│   ├── toast.tsx           # Toast notifications
│   ├── link.tsx            # Clickable links
│   └── dialog-*.tsx        # Dialog variants
│
└── util/                   # Utility functions
    ├── clipboard.ts        # Clipboard operations
    ├── editor.ts           # External editor integration
    ├── selection.ts        # Text selection
    ├── signal.ts           # Signal utilities
    ├── terminal.ts         # Terminal utilities
    └── transcript.ts       # Session transcript
```

## Solid.js Reactive Primitives

### Signals: Reactive State

Signals are the foundation of Solid's reactivity:

```typescript
import { createSignal } from "solid-js"

// Create a signal with initial value
const [count, setCount] = createSignal(0)

// Read the value (call as function)
console.log(count())  // 0

// Update the value
setCount(1)
setCount(prev => prev + 1)
```

### Effects: Side Effects on Change

Effects run when their dependencies change:

```typescript
import { createEffect } from "solid-js"

const [name, setName] = createSignal("World")

// This runs whenever `name` changes
createEffect(() => {
  console.log(`Hello, ${name()}!`)
})

setName("OpenCode")  // Logs: "Hello, OpenCode!"
```

### Memos: Derived State

Memos cache computed values:

```typescript
import { createMemo } from "solid-js"

const [items, setItems] = createSignal([1, 2, 3, 4, 5])

// Only recalculates when `items` changes
const total = createMemo(() => {
  return items().reduce((a, b) => a + b, 0)
})

console.log(total())  // 15
```

## Real Examples from OpenCode

### Route Context

The route context manages navigation:

```typescript
// packages/opencode/src/cli/cmd/tui/context/route.tsx
import { createStore } from "solid-js/store"
import { createSimpleContext } from "./helper"

export type HomeRoute = {
  type: "home"
  initialPrompt?: PromptInfo
  workspaceID?: string
}

export type SessionRoute = {
  type: "session"
  sessionID: string
  initialPrompt?: PromptInfo
}

export type Route = HomeRoute | SessionRoute

export const { use: useRoute, provider: RouteProvider } = createSimpleContext({
  name: "Route",
  init: () => {
    const [store, setStore] = createStore<Route>({
      type: "home",
    })

    return {
      get data() {
        return store
      },
      navigate(route: Route) {
        console.log("navigate", route)
        setStore(route)
      },
    }
  },
})
```

### Using Routes in Components

```typescript
// packages/opencode/src/cli/cmd/tui/app.tsx (lines 806-815)
return (
  <box>
    <Switch>
      <Match when={route.data.type === "home"}>
        <Home />
      </Match>
      <Match when={route.data.type === "session"}>
        <Session />
      </Match>
    </Switch>
  </box>
)
```

### SDK Context

The SDK context provides the API client:

```typescript
// packages/opencode/src/cli/cmd/tui/context/sdk.tsx (lines 11-125)
export const { use: useSDK, provider: SDKProvider } = createSimpleContext({
  name: "SDK",
  init: (props: {
    url: string
    directory?: string
    fetch?: typeof fetch
    headers?: RequestInit["headers"]
    events?: EventSource
  }) => {
    const abort = new AbortController()
    
    function createSDK() {
      return createOpencodeClient({
        baseUrl: props.url,
        signal: abort.signal,
        directory: props.directory,
        fetch: props.fetch,
        headers: props.headers,
      })
    }

    let sdk = createSDK()
    const emitter = createGlobalEmitter<{
      [key in Event["type"]]: Extract<Event, { type: key }>
    }>()

    // Event handling with batching...
    
    return {
      get client() {
        return sdk
      },
      directory: props.directory,
      event: emitter,
      fetch: props.fetch ?? fetch,
      url: props.url,
    }
  },
})
```

### Context Helper

OpenCode uses a helper to create contexts consistently:

```typescript
// packages/opencode/src/cli/cmd/tui/context/helper.tsx
import { createContext, Show, useContext, type ParentProps } from "solid-js"

export function createSimpleContext<T, Props extends Record<string, any>>(input: {
  name: string
  init: ((input: Props) => T) | (() => T)
}) {
  const ctx = createContext<T>()

  return {
    provider: (props: ParentProps<Props>) => {
      const init = input.init(props)
      return (
        <Show when={init.ready === undefined || init.ready === true}>
          <ctx.Provider value={init}>{props.children}</ctx.Provider>
        </Show>
      )
    },
    use() {
      const value = useContext(ctx)
      if (!value) throw new Error(`${input.name} context must be used within a provider`)
      return value
    },
  }
}
```

## The Main Application

The main app component wires everything together:

```typescript
// packages/opencode/src/cli/cmd/tui/app.tsx (lines 109-202)
export function tui(input: {
  url: string
  args: Args
  config: TuiConfig.Info
  directory?: string
  fetch?: typeof fetch
  headers?: RequestInit["headers"]
  events?: EventSource
}) {
  return new Promise<void>(async (resolve) => {
    const mode = await getTerminalBackgroundColor()
    
    const onExit = async () => {
      resolve()
    }

    render(
      () => {
        return (
          <ErrorBoundary fallback={(error, reset) => <ErrorComponent />}>
            <ArgsProvider {...input.args}>
              <ExitProvider onExit={onExit}>
                <KVProvider>
                  <ToastProvider>
                    <RouteProvider>
                      <TuiConfigProvider config={input.config}>
                        <SDKProvider url={input.url} directory={input.directory}>
                          <SyncProvider>
                            <ThemeProvider mode={mode}>
                              <LocalProvider>
                                <KeybindProvider>
                                  <PromptStashProvider>
                                    <DialogProvider>
                                      <CommandProvider>
                                        <FrecencyProvider>
                                          <PromptHistoryProvider>
                                            <PromptRefProvider>
                                              <App />
                                            </PromptRefProvider>
                                          </PromptHistoryProvider>
                                        </FrecencyProvider>
                                      </CommandProvider>
                                    </DialogProvider>
                                  </PromptStashProvider>
                                </KeybindProvider>
                              </LocalProvider>
                            </ThemeProvider>
                          </SyncProvider>
                        </SDKProvider>
                      </TuiConfigProvider>
                    </RouteProvider>
                  </ToastProvider>
                </KVProvider>
              </ExitProvider>
            </ArgsProvider>
          </ErrorBoundary>
        )
      },
      {
        targetFps: 60,
        exitOnCtrlC: false,
        // ... more options
      },
    )
  })
}
```

### Provider Hierarchy

```
┌─────────────────────────────────────────────────────────────┐
│  ArgsProvider         CLI arguments                         │
│  └─ ExitProvider      Exit handling                         │
│     └─ KVProvider     Key-value storage                     │
│        └─ ToastProvider  Notifications                      │
│           └─ RouteProvider  Navigation                      │
│              └─ TuiConfigProvider  Configuration            │
│                 └─ SDKProvider  API client                  │
│                    └─ SyncProvider  Server sync             │
│                       └─ ThemeProvider  Colors              │
│                          └─ LocalProvider  Local state      │
│                             └─ KeybindProvider  Shortcuts   │
│                                └─ ... more providers        │
│                                   └─ App  Main component    │
└─────────────────────────────────────────────────────────────┘
```

## Reactive Patterns in the TUI

### Tracking Session Status

```typescript
// packages/opencode/src/cli/cmd/tui/component/prompt/index.tsx (lines 75)
const status = createMemo(() => 
  sync.data.session_status?.[props.sessionID ?? ""] ?? { type: "idle" }
)
```

### Responding to Events

```typescript
// packages/opencode/src/cli/cmd/tui/app.tsx (lines 693-711)
sdk.event.on(TuiEvent.CommandExecute.type, (evt) => {
  command.trigger(evt.properties.command)
})

sdk.event.on(TuiEvent.ToastShow.type, (evt) => {
  toast.show({
    title: evt.properties.title,
    message: evt.properties.message,
    variant: evt.properties.variant,
    duration: evt.properties.duration,
  })
})

sdk.event.on(TuiEvent.SessionSelect.type, (evt) => {
  route.navigate({
    type: "session",
    sessionID: evt.properties.sessionID,
  })
})
```

### Conditional Rendering

```typescript
// packages/opencode/src/cli/cmd/tui/routes/home.tsx (lines 59-76)
const Hint = (
  <Show when={connectedMcpCount() > 0}>
    <box flexDirection="row" gap={1}>
      <text fg={theme.text}>
        <Switch>
          <Match when={mcpError()}>
            <span style={{ fg: theme.error }}>•</span> mcp errors
          </Match>
          <Match when={true}>
            <span style={{ fg: theme.success }}>•</span>
            {Locale.pluralize(connectedMcpCount(), "{} mcp server", "{} mcp servers")}
          </Match>
        </Switch>
      </text>
    </box>
  </Show>
)
```

## Component Lifecycle

Solid provides lifecycle hooks:

```typescript
import { onMount, onCleanup, createEffect, on } from "solid-js"

function MyComponent() {
  // Runs once when component mounts
  onMount(() => {
    console.log("Component mounted")
  })

  // Runs when component unmounts
  onCleanup(() => {
    console.log("Component unmounting")
  })

  // Effect with explicit dependencies
  createEffect(on(
    () => someSignal(),
    (value, prevValue) => {
      console.log(`Changed from ${prevValue} to ${value}`)
    },
    { defer: true }  // Don't run on initial value
  ))
}
```

### Real Example: Terminal Title

```typescript
// packages/opencode/src/cli/cmd/tui/app.tsx (lines 267-286)
createEffect(() => {
  if (!terminalTitleEnabled() || Flag.OPENCODE_DISABLE_TERMINAL_TITLE) return

  if (route.data.type === "home") {
    renderer.setTerminalTitle("OpenCode")
    return
  }

  if (route.data.type === "session") {
    const session = sync.session.get(route.data.sessionID)
    if (!session || SessionApi.isDefaultTitle(session.title)) {
      renderer.setTerminalTitle("OpenCode")
      return
    }

    const title = session.title.length > 40 
      ? session.title.slice(0, 37) + "..." 
      : session.title
    renderer.setTerminalTitle(`OC | ${title}`)
  }
})
```

## Stores for Complex State

For complex nested state, Solid provides stores:

```typescript
import { createStore, produce } from "solid-js/store"

const [store, setStore] = createStore({
  prompt: {
    input: "",
    parts: [],
  },
  mode: "normal",
})

// Direct update
setStore("mode", "shell")

// Nested update
setStore("prompt", "input", "Hello")

// Complex update with produce (immer-like)
setStore(produce((draft) => {
  draft.prompt.parts.push({ type: "text", text: "new part" })
}))
```

## Self-Check Questions

1. What is the key difference between Solid.js and React's rendering model?
2. What are the three main reactive primitives in Solid.js?
3. How does the context helper (`createSimpleContext`) work?
4. Why is the provider hierarchy important in the TUI?
5. What is the purpose of `createMemo` vs `createEffect`?

## Exercises

### Exercise 1: Trace a Signal
1. Open `packages/opencode/src/cli/cmd/tui/context/route.tsx`
2. Find where the route store is created
3. Find where `navigate()` updates the store
4. Find where the route is read in `app.tsx`

### Exercise 2: Understand Context Flow
1. Open `packages/opencode/src/cli/cmd/tui/app.tsx`
2. Trace the provider hierarchy
3. Find a component that uses `useSDK()`
4. Understand how it accesses the SDK client

### Exercise 3: Examine Reactive Updates
1. Open `packages/opencode/src/cli/cmd/tui/routes/home.tsx`
2. Find the `createMemo` calls
3. Understand what triggers their recalculation
4. Find the `Show` and `Switch` components

## Key Takeaways

1. **Fine-grained reactivity**: Solid updates only what changes, perfect for terminal rendering

2. **Signals are functions**: Always call signals as functions to read their values

3. **Context provides dependency injection**: Providers make services available throughout the component tree

4. **Effects track automatically**: Dependencies are tracked by reading signals inside effects

5. **Stores handle complex state**: Use stores for nested objects and arrays

## Further Reading

- [Solid.js Documentation](https://www.solidjs.com/docs/latest)
- [Solid.js Tutorial](https://www.solidjs.com/tutorial)
- [Solid Primitives](https://github.com/solidjs-community/solid-primitives)
- `packages/opencode/src/cli/cmd/tui/context/` - All context implementations
