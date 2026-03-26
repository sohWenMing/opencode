# Lesson 01: Yargs Command System

## Learning Objectives

By the end of this lesson, you will be able to:
- Understand what yargs is and why it's used for CLI applications
- Explain how opencode registers and organizes commands
- Identify the command structure pattern (command, describe, builder, handler)
- Navigate the CLI command organization in the codebase
- Create and modify CLI commands

## What is Yargs?

Yargs is a popular Node.js library for building command-line interfaces. It provides:

- **Argument parsing**: Converts command-line arguments into structured data
- **Command routing**: Directs execution to the appropriate handler
- **Help generation**: Automatically creates `--help` output
- **Validation**: Ensures required arguments are provided
- **Type coercion**: Converts string arguments to appropriate types

```
┌─────────────────────────────────────────────────────────────┐
│                    User Input                               │
│         opencode run "Fix the bug" --model gpt-4            │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    Yargs Parser                             │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐  │
│  │   Command   │  │ Positional  │  │      Options        │  │
│  │    "run"    │  │  "Fix..."   │  │  model: "gpt-4"     │  │
│  └─────────────┘  └─────────────┘  └─────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                  Command Handler                            │
│              RunCommand.handler(args)                       │
└─────────────────────────────────────────────────────────────┘
```

## CLI Entry Point

The CLI entry point is at `packages/opencode/src/index.ts`. This file:

1. Imports all command modules
2. Configures yargs with global options
3. Registers all commands
4. Handles errors and cleanup

### Yargs Configuration

```typescript
// packages/opencode/src/index.ts (lines 50-67)
let cli = yargs(hideBin(process.argv))
  .parserConfiguration({ "populate--": true })
  .scriptName("opencode")
  .wrap(100)
  .help("help", "show help")
  .alias("help", "h")
  .version("version", "show version number", Installation.VERSION)
  .alias("version", "v")
  .option("print-logs", {
    describe: "print logs to stderr",
    type: "boolean",
  })
  .option("log-level", {
    describe: "log level",
    type: "string",
    choices: ["DEBUG", "INFO", "WARN", "ERROR"],
  })
```

Key configuration options:
- `hideBin(process.argv)`: Removes the node executable and script path from arguments
- `parserConfiguration({ "populate--": true })`: Captures arguments after `--` in a special array
- `scriptName("opencode")`: Sets the name shown in help text
- `wrap(100)`: Sets the terminal width for help text wrapping

### Middleware

Yargs middleware runs before any command handler:

```typescript
// packages/opencode/src/index.ts (lines 67-123)
.middleware(async (opts) => {
  await Log.init({
    print: process.argv.includes("--print-logs"),
    dev: Installation.isLocal(),
    level: (() => {
      if (opts.logLevel) return opts.logLevel as Log.Level
      if (Installation.isLocal()) return "DEBUG"
      return "INFO"
    })(),
  })

  process.env.AGENT = "1"
  process.env.OPENCODE_PID = String(process.pid)

  // Database migration on first run...
})
```

The middleware:
1. Initializes the logging system
2. Sets environment variables
3. Performs one-time database migration if needed

## Command Registration

Commands are registered using the `.command()` method:

```typescript
// packages/opencode/src/index.ts (lines 126-148)
cli = cli
  .command(AcpCommand)
  .command(McpCommand)
  .command(TuiThreadCommand)
  .command(AttachCommand)
  .command(RunCommand)
  .command(GenerateCommand)
  .command(DebugCommand)
  .command(ConsoleCommand)
  .command(ProvidersCommand)
  .command(AgentCommand)
  .command(UpgradeCommand)
  .command(UninstallCommand)
  .command(ServeCommand)
  .command(WebCommand)
  .command(ModelsCommand)
  .command(StatsCommand)
  .command(ExportCommand)
  .command(ImportCommand)
  .command(GithubCommand)
  .command(PrCommand)
  .command(SessionCommand)
  .command(DbCommand)
```

## Command Structure

Each command follows a consistent structure defined by the `cmd()` helper function:

```typescript
// packages/opencode/src/cli/cmd/cmd.ts
import type { CommandModule } from "yargs"

type WithDoubleDash<T> = T & { "--"?: string[] }

export function cmd<T, U>(input: CommandModule<T, WithDoubleDash<U>>) {
  return input
}
```

### The Four Parts of a Command

Every command has four parts:

```
┌─────────────────────────────────────────────────────────────┐
│                    Command Module                           │
├─────────────────────────────────────────────────────────────┤
│  command:   "run [message..]"                               │
│             The command syntax with positional args         │
├─────────────────────────────────────────────────────────────┤
│  describe:  "run opencode with a message"                   │
│             Description shown in help text                  │
├─────────────────────────────────────────────────────────────┤
│  builder:   (yargs) => yargs.option(...)                    │
│             Configures options and arguments                │
├─────────────────────────────────────────────────────────────┤
│  handler:   async (args) => { ... }                         │
│             The function that executes the command          │
└─────────────────────────────────────────────────────────────┘
```

### Example: The Run Command

```typescript
// packages/opencode/src/cli/cmd/run.ts (lines 221-305)
export const RunCommand = cmd({
  command: "run [message..]",
  describe: "run opencode with a message",
  builder: (yargs: Argv) => {
    return yargs
      .positional("message", {
        describe: "message to send",
        type: "string",
        array: true,
        default: [],
      })
      .option("command", {
        describe: "the command to run, use message for args",
        type: "string",
      })
      .option("continue", {
        alias: ["c"],
        describe: "continue the last session",
        type: "boolean",
      })
      .option("session", {
        alias: ["s"],
        describe: "session id to continue",
        type: "string",
      })
      .option("model", {
        type: "string",
        alias: ["m"],
        describe: "model to use in the format of provider/model",
      })
      // ... more options
  },
  handler: async (args) => {
    // Command implementation
  },
})
```

### Example: The Serve Command (Simple)

```typescript
// packages/opencode/src/cli/cmd/serve.ts
export const ServeCommand = cmd({
  command: "serve",
  builder: (yargs) => withNetworkOptions(yargs),
  describe: "starts a headless opencode server",
  handler: async (args) => {
    if (!Flag.OPENCODE_SERVER_PASSWORD) {
      console.log("Warning: OPENCODE_SERVER_PASSWORD is not set")
    }
    const opts = await resolveNetworkOptions(args)
    const server = Server.listen(opts)
    console.log(`opencode server listening on http://${server.hostname}:${server.port}`)
    await new Promise(() => {})
    await server.stop()
  },
})
```

## Available Commands

| Command | Description | Key Options |
|---------|-------------|-------------|
| `$0` (default) | Start the TUI | `--model`, `--continue`, `--session` |
| `run` | Run with a message | `--model`, `--continue`, `--format` |
| `serve` | Headless server | `--port`, `--hostname` |
| `web` | Web interface | `--port`, `--hostname` |
| `mcp` | Manage MCP servers | `add`, `list`, `auth` subcommands |
| `models` | List models | `--provider`, `--verbose` |
| `providers` | List providers | - |
| `agent` | Manage agents | `create`, `list` subcommands |
| `upgrade` | Upgrade opencode | - |
| `uninstall` | Uninstall opencode | - |
| `export` | Export sessions | - |
| `import` | Import sessions | - |

## Command Organization

Commands are organized in the `packages/opencode/src/cli/cmd/` directory:

```
packages/opencode/src/cli/cmd/
├── cmd.ts              # Helper function for creating commands
├── run.ts              # Run command (non-interactive)
├── serve.ts            # Headless server
├── web.ts              # Web interface
├── mcp.ts              # MCP server management
├── models.ts           # List models
├── providers.ts        # List providers
├── agent.ts            # Agent management
├── upgrade.ts          # Upgrade command
├── uninstall.ts        # Uninstall command
├── export.ts           # Export sessions
├── import.ts           # Import sessions
├── debug/              # Debug subcommands
│   ├── index.ts
│   ├── agent.ts
│   ├── config.ts
│   └── ...
└── tui/                # Terminal UI
    ├── thread.ts       # TUI entry point
    ├── app.tsx         # Main TUI application
    └── ...
```

## Subcommands

Commands can have subcommands. The `mcp` command is a good example:

```typescript
// packages/opencode/src/cli/cmd/mcp.ts (lines 53-65)
export const McpCommand = cmd({
  command: "mcp",
  describe: "manage MCP (Model Context Protocol) servers",
  builder: (yargs) =>
    yargs
      .command(McpAddCommand)
      .command(McpListCommand)
      .command(McpAuthCommand)
      .command(McpLogoutCommand)
      .command(McpDebugCommand)
      .demandCommand(),
  async handler() {},
})
```

Usage:
```bash
opencode mcp list          # List MCP servers
opencode mcp add           # Add a new MCP server
opencode mcp auth [name]   # Authenticate with OAuth
opencode mcp logout [name] # Remove credentials
```

## The Default Command (TUI)

The default command (`$0`) launches the Terminal UI:

```typescript
// packages/opencode/src/cli/cmd/tui/thread.ts (lines 66-101)
export const TuiThreadCommand = cmd({
  command: "$0 [project]",
  describe: "start opencode tui",
  builder: (yargs) =>
    withNetworkOptions(yargs)
      .positional("project", {
        type: "string",
        describe: "path to start opencode in",
      })
      .option("model", {
        type: "string",
        alias: ["m"],
        describe: "model to use in the format of provider/model",
      })
      .option("continue", {
        alias: ["c"],
        describe: "continue the last session",
        type: "boolean",
      })
      // ... more options
})
```

The `$0` syntax means this command runs when no other command is specified.

## Error Handling

The CLI has comprehensive error handling:

```typescript
// packages/opencode/src/index.ts (lines 153-166)
cli = cli
  .fail((msg, err) => {
    if (
      msg?.startsWith("Unknown argument") ||
      msg?.startsWith("Not enough non-option arguments") ||
      msg?.startsWith("Invalid values:")
    ) {
      if (err) throw err
      cli.showHelp("log")
    }
    if (err) throw err
    process.exit(1)
  })
  .strict()
```

And a try-catch around the main execution:

```typescript
// packages/opencode/src/index.ts (lines 168-213)
try {
  await cli.parse()
} catch (e) {
  // Error logging and formatting
  const formatted = FormatError(e)
  if (formatted) UI.error(formatted)
  process.exitCode = 1
} finally {
  process.exit()
}
```

## Self-Check Questions

1. What is the purpose of `hideBin(process.argv)` in yargs configuration?
2. What are the four parts of a yargs command module?
3. How does opencode handle the default command (when no command is specified)?
4. What is the purpose of the middleware in the CLI?
5. How are subcommands organized in the `mcp` command?

## Exercises

### Exercise 1: Explore Help Output
Run the following commands and examine the help output:
```bash
opencode --help
opencode run --help
opencode mcp --help
opencode agent --help
```

### Exercise 2: Trace Command Registration
1. Open `packages/opencode/src/index.ts`
2. Find where `RunCommand` is imported
3. Find where it's registered with `.command()`
4. Open `packages/opencode/src/cli/cmd/run.ts`
5. Identify the four parts of the command

### Exercise 3: Examine a Simple Command
1. Open `packages/opencode/src/cli/cmd/models.ts`
2. Identify:
   - The command syntax
   - The options defined in `builder`
   - What the handler does

## Key Takeaways

1. **Yargs provides structure**: Commands follow a consistent pattern with `command`, `describe`, `builder`, and `handler`

2. **Middleware runs first**: Global setup like logging happens before any command

3. **Commands are modular**: Each command is in its own file, making the codebase easy to navigate

4. **Subcommands enable grouping**: Related functionality (like MCP management) is grouped under a parent command

5. **Error handling is centralized**: The main entry point catches and formats all errors

## Further Reading

- [Yargs Documentation](https://yargs.js.org/)
- [Command Module API](https://yargs.js.org/docs/#api-reference-commandmodule)
- `packages/opencode/src/cli/cmd/` - Browse all command implementations
