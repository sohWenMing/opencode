# Exercise 01: CLI and TUI Exploration

## Overview

This exercise will help you understand the CLI and TUI architecture through hands-on exploration. You'll run commands, trace code paths, and examine components.

## Prerequisites

- opencode installed and working
- Access to the source code
- A terminal that supports the TUI

## Part 1: CLI Command Exploration

### Exercise 1.1: Explore Help Output

Run the following commands and answer the questions:

```bash
# Main help
opencode --help

# Command-specific help
opencode run --help
opencode mcp --help
opencode agent --help
opencode models --help
```

**Questions:**
1. What is the default command (when you just run `opencode`)?
2. How many top-level commands are available?
3. What options are available for the `run` command?
4. What subcommands does `mcp` have?

### Exercise 1.2: Run Non-Interactive Commands

Try these commands and observe the output:

```bash
# List available models
opencode models

# List with provider filter
opencode models anthropic

# List providers
opencode providers

# List agents
opencode agent list
```

**Questions:**
1. What format is the model output in?
2. How are models identified (provider/model)?
3. What agents are available by default?

### Exercise 1.3: Examine Command Options

Run `opencode run --help` and identify:

1. Positional arguments
2. Boolean flags (--continue, --share)
3. String options (--model, --session)
4. Array options (--file)

**Task:** Write a command that would:
- Continue the last session
- Use a specific model
- Attach a file

```bash
# Your command here:
opencode run --continue --model anthropic/claude-3-5-sonnet-20241022 --file ./README.md "Summarize this file"
```

## Part 2: Code Tracing

### Exercise 2.1: Trace Command Registration

**Goal:** Understand how commands are registered and routed.

1. Open `packages/opencode/src/index.ts`
2. Find the import for `RunCommand`
3. Find where it's registered with `.command()`
4. Open `packages/opencode/src/cli/cmd/run.ts`

**Document the flow:**

```
User runs: opencode run "Hello"
     │
     ▼
┌─────────────────────────┐
│ packages/opencode/src/  │
│ index.ts                │
│ - yargs parses args     │
│ - finds "run" command   │
└─────────────────────────┘
     │
     ▼
┌─────────────────────────┐
│ packages/opencode/src/  │
│ cli/cmd/run.ts          │
│ - builder() configures  │
│ - handler() executes    │
└─────────────────────────┘
```

### Exercise 2.2: Trace the Default Command

**Goal:** Understand how the TUI starts.

1. Find `TuiThreadCommand` in `packages/opencode/src/index.ts`
2. Open `packages/opencode/src/cli/cmd/tui/thread.ts`
3. Find where `tui()` is called
4. Open `packages/opencode/src/cli/cmd/tui/app.tsx`
5. Find the `render()` call

**Questions:**
1. What does `$0` mean in the command definition?
2. What is the purpose of the worker?
3. What providers wrap the main App component?

### Exercise 2.3: Trace a Subcommand

**Goal:** Understand nested command structure.

1. Open `packages/opencode/src/cli/cmd/mcp.ts`
2. Find `McpCommand` and its subcommands
3. Find `McpListCommand`
4. Trace what happens when you run `opencode mcp list`

**Document the structure:**

```
McpCommand
├── command: "mcp"
├── describe: "manage MCP servers"
├── builder: registers subcommands
│   ├── McpAddCommand
│   ├── McpListCommand
│   ├── McpAuthCommand
│   ├── McpLogoutCommand
│   └── McpDebugCommand
└── handler: empty (requires subcommand)
```

## Part 3: TUI Component Exploration

### Exercise 3.1: Explore the Provider Hierarchy

Open `packages/opencode/src/cli/cmd/tui/app.tsx` and list all the providers in order:

```
1. ErrorBoundary
2. ArgsProvider
3. ExitProvider
4. KVProvider
5. ToastProvider
6. RouteProvider
7. TuiConfigProvider
8. SDKProvider
9. SyncProvider
10. ThemeProvider
11. LocalProvider
12. KeybindProvider
13. PromptStashProvider
14. DialogProvider
15. CommandProvider
16. FrecencyProvider
17. PromptHistoryProvider
18. PromptRefProvider
```

**Questions:**
1. Why is `ErrorBoundary` at the top?
2. Why is `SDKProvider` before `SyncProvider`?
3. What does each provider provide?

### Exercise 3.2: Examine the Route System

1. Open `packages/opencode/src/cli/cmd/tui/context/route.tsx`
2. Identify the route types (HomeRoute, SessionRoute)
3. Find where routes are used in `app.tsx`

**Task:** Draw a state diagram:

```
┌──────────────┐                    ┌──────────────┐
│              │   navigate()       │              │
│    Home      │ ─────────────────► │   Session    │
│  type:"home" │                    │type:"session"│
│              │ ◄───────────────── │              │
└──────────────┘   navigate()       └──────────────┘
```

### Exercise 3.3: Explore the Theme System

1. Open `packages/opencode/src/cli/cmd/tui/context/theme.tsx`
2. Find the list of built-in themes
3. Open one theme file (e.g., `theme/dracula.json`)

**Task:** Create a mini theme specification:

```json
{
  "theme": {
    "primary": "#your-color",
    "secondary": "#your-color",
    "background": "#your-color",
    "text": "#your-color",
    "error": "#your-color"
  }
}
```

### Exercise 3.4: Examine the Prompt Component

1. Open `packages/opencode/src/cli/cmd/tui/component/prompt/index.tsx`
2. Find the `<textarea>` component
3. Identify the event handlers (onKeyDown, onSubmit, onPaste)

**Questions:**
1. How does the prompt handle multi-line input?
2. What happens when you paste an image?
3. How is shell mode (!) triggered?

## Part 4: Hands-On Modifications

### Exercise 4.1: Add a Console Log

**Goal:** Verify you can modify and test the code.

1. Open `packages/opencode/src/cli/cmd/run.ts`
2. Add a console.log at the start of the handler:

```typescript
handler: async (args) => {
  console.log("Run command started with args:", args.message)
  // ... rest of handler
}
```

3. Run `opencode run "test message"`
4. Verify your log appears
5. Remove the log

### Exercise 4.2: Explore Keybindings

1. Open `packages/opencode/src/cli/cmd/tui/context/keybind.tsx`
2. Find where keybindings are defined
3. Look at the config structure

**Task:** List 5 keybindings and their purposes:

| Keybind | Action |
|---------|--------|
| `ctrl+x` | Leader key |
| `ctrl+c` | Interrupt/copy |
| `ctrl+d` | Exit |
| `ctrl+p` | Command palette |
| `ctrl+n` | New session |

### Exercise 4.3: Trace an Event

**Goal:** Understand the event system.

1. Open `packages/opencode/src/cli/cmd/tui/app.tsx`
2. Find `sdk.event.on(...)` calls
3. Trace what happens when `TuiEvent.SessionSelect` is emitted

**Document the flow:**

```
Event emitted: TuiEvent.SessionSelect
     │
     ▼
sdk.event.on(TuiEvent.SessionSelect.type, (evt) => {
  route.navigate({
    type: "session",
    sessionID: evt.properties.sessionID,
  })
})
     │
     ▼
Route store updated
     │
     ▼
<Switch> re-renders
     │
     ▼
<Session /> component shown
```

## Part 5: Synthesis

### Final Exercise: Document a Feature

Choose one of these features and document how it works:

1. **Session continuation** (`--continue` flag)
2. **Model selection** (`--model` flag)
3. **MCP authentication** (`opencode mcp auth`)
4. **Theme switching** (in TUI)

Your documentation should include:
- Entry point (CLI command or TUI action)
- Code path through the codebase
- Key files involved
- State changes that occur
- UI updates (if applicable)

**Template:**

```markdown
## Feature: [Name]

### Entry Point
- Command: `opencode ...`
- File: `packages/opencode/src/cli/cmd/...`

### Code Path
1. [Step 1]
2. [Step 2]
3. [Step 3]

### Key Files
- `file1.ts` - [purpose]
- `file2.tsx` - [purpose]

### State Changes
- [What state is modified]

### UI Updates
- [What the user sees]
```

## Completion Checklist

- [ ] Explored CLI help output
- [ ] Ran non-interactive commands
- [ ] Traced command registration
- [ ] Traced the TUI startup
- [ ] Examined provider hierarchy
- [ ] Explored the route system
- [ ] Examined the theme system
- [ ] Made a test modification
- [ ] Documented a feature

## Reflection Questions

1. How does the separation of CLI and TUI benefit the codebase?
2. What patterns do you see repeated across commands?
3. How does Solid.js reactivity help in the TUI?
4. What would you change about the architecture?

## Next Steps

After completing these exercises, you should be able to:
- Navigate the CLI codebase confidently
- Add new CLI commands
- Modify TUI components
- Understand the reactive data flow
- Debug issues in the user interaction layer
