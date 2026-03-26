# Lesson 01: Tool Abstraction

## Learning Objectives

By the end of this lesson, you will be able to:
- Understand what tools are in the opencode architecture
- Explain the `Tool.define` function and its purpose
- Describe the structure of a tool: name, description, parameters, and execute function
- Understand the `Tool.Context` and what it provides
- Explain tool result types and how truncation works

## What Are Tools?

Tools are **capabilities that the AI can use to interact with the outside world**. When you ask opencode to read a file, run a command, or search for code, the AI invokes a tool to perform that action.

Think of tools as the "hands" of the AI:
- The AI's "brain" (the LLM) decides what to do
- Tools are how it actually does things
- Each tool has a specific purpose and defined interface

```
┌─────────────────────────────────────────────────────────────┐
│                        AI Agent                              │
│  ┌─────────────────────────────────────────────────────┐    │
│  │                    LLM (Brain)                       │    │
│  │  "I need to read the file at src/index.ts"          │    │
│  └─────────────────────────────────────────────────────┘    │
│                           │                                  │
│                           ▼                                  │
│  ┌─────────────────────────────────────────────────────┐    │
│  │                   Tool System                        │    │
│  │  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐   │    │
│  │  │  read   │ │  bash   │ │  edit   │ │  grep   │   │    │
│  │  └────┬────┘ └─────────┘ └─────────┘ └─────────┘   │    │
│  └───────┼──────────────────────────────────────────────┘    │
│          │                                                   │
└──────────┼───────────────────────────────────────────────────┘
           │
           ▼
    ┌──────────────┐
    │  File System │
    │  src/index.ts│
    └──────────────┘
```

## The Tool.define Function

All tools in opencode are created using the `Tool.define` function. This function provides a consistent way to define tools with validation, error handling, and output truncation built in.

Here's the core implementation from `packages/opencode/src/tool/tool.ts`:

```typescript
export namespace Tool {
  export function define<Parameters extends z.ZodType, Result extends Metadata>(
    id: string,
    init: Info<Parameters, Result>["init"] | Awaited<ReturnType<Info<Parameters, Result>["init"]>>,
  ): Info<Parameters, Result> {
    return {
      id,
      init: async (initCtx) => {
        const toolInfo = init instanceof Function ? await init(initCtx) : init
        const execute = toolInfo.execute
        toolInfo.execute = async (args, ctx) => {
          try {
            toolInfo.parameters.parse(args)
          } catch (error) {
            if (error instanceof z.ZodError && toolInfo.formatValidationError) {
              throw new Error(toolInfo.formatValidationError(error), { cause: error })
            }
            throw new Error(
              `The ${id} tool was called with invalid arguments: ${error}.\nPlease rewrite the input so it satisfies the expected schema.`,
              { cause: error },
            )
          }
          const result = await execute(args, ctx)
          // skip truncation for tools that handle it themselves
          if (result.metadata.truncated !== undefined) {
            return result
          }
          const truncated = await Truncate.output(result.output, {}, initCtx?.agent)
          return {
            ...result,
            output: truncated.content,
            metadata: {
              ...result.metadata,
              truncated: truncated.truncated,
              ...(truncated.truncated && { outputPath: truncated.outputPath }),
            },
          }
        }
        return toolInfo
      },
    }
  }
}
```

### Key Features of Tool.define

1. **Lazy Initialization**: Tools can be initialized lazily with context about the current agent
2. **Parameter Validation**: Uses Zod schemas to validate inputs before execution
3. **Automatic Truncation**: Large outputs are automatically truncated to prevent context overflow
4. **Consistent Error Messages**: Validation errors are formatted consistently

## Tool Structure

Every tool has four main components:

### 1. ID (Name)

A unique string identifier for the tool:

```typescript
Tool.define("read", { ... })  // ID is "read"
Tool.define("bash", { ... })  // ID is "bash"
```

### 2. Description

A human-readable description that tells the AI when and how to use the tool:

```typescript
{
  description: "Read the contents of a file at the specified path...",
  // ...
}
```

### 3. Parameters (Zod Schema)

A Zod schema defining what arguments the tool accepts:

```typescript
parameters: z.object({
  filePath: z.string().describe("The absolute path to the file to read"),
  offset: z.coerce.number().describe("The line number to start reading from").optional(),
  limit: z.coerce.number().describe("The maximum number of lines to read").optional(),
})
```

The `.describe()` method is crucial - it tells the AI what each parameter is for.

### 4. Execute Function

The actual implementation that runs when the tool is called:

```typescript
async execute(params, ctx) {
  // Perform the action
  // Return the result
  return {
    title: "Short title for UI",
    metadata: { /* structured data */ },
    output: "Text output for the AI",
  }
}
```

## Tool.Context

The `execute` function receives a context object with essential information:

```typescript
export type Context<M extends Metadata = Metadata> = {
  sessionID: SessionID        // Current session identifier
  messageID: MessageID        // Current message identifier
  agent: string               // Name of the agent using the tool
  abort: AbortSignal          // Signal to cancel long-running operations
  callID?: string             // Unique ID for this tool call
  extra?: { [key: string]: any }  // Additional context data
  messages: MessageV2.WithParts[] // Conversation history
  metadata(input: { title?: string; metadata?: M }): void  // Update UI metadata
  ask(input: Omit<Permission.Request, "id" | "sessionID" | "tool">): Promise<void>  // Request permission
}
```

### Key Context Properties

| Property | Purpose |
|----------|---------|
| `sessionID` | Identifies the current conversation session |
| `messageID` | Identifies the specific message being processed |
| `abort` | Allows cancellation of long-running operations |
| `messages` | Access to conversation history for context |
| `metadata()` | Update the UI with progress information |
| `ask()` | Request user permission for sensitive operations |

### Using ctx.metadata()

Tools can update the UI in real-time:

```typescript
async execute(params, ctx) {
  ctx.metadata({
    metadata: {
      output: "",
      description: params.description,
    },
  })
  
  // As work progresses...
  ctx.metadata({
    metadata: {
      output: partialOutput,
      description: params.description,
    },
  })
}
```

### Using ctx.ask()

Tools request permission for sensitive operations:

```typescript
async execute(params, ctx) {
  await ctx.ask({
    permission: "read",
    patterns: [filepath],
    always: ["*"],
    metadata: {},
  })
  // Permission granted, proceed with operation
}
```

## Tool Result Types

Every tool must return an object with this structure:

```typescript
{
  title: string,           // Short description for UI
  metadata: M,             // Structured data (type varies by tool)
  output: string,          // Text output sent to the AI
  attachments?: Omit<MessageV2.FilePart, "id" | "sessionID" | "messageID">[]  // Optional files/images
}
```

### Example Results

**Simple text result:**
```typescript
return {
  title: "src/index.ts",
  metadata: { preview: "...", truncated: false },
  output: "<path>src/index.ts</path>\n<content>...</content>",
}
```

**Result with image attachment:**
```typescript
return {
  title: "screenshot.png",
  output: "Image read successfully",
  metadata: { truncated: false },
  attachments: [{
    type: "file",
    mime: "image/png",
    url: `data:image/png;base64,${base64Content}`,
  }],
}
```

## Output Truncation

Large outputs can overwhelm the AI's context window. The `Truncate` namespace handles this automatically.

From `packages/opencode/src/tool/truncate.ts`:

```typescript
export namespace Truncate {
  export const MAX_LINES = 2000
  export const MAX_BYTES = 50 * 1024  // 50KB

  export type Result = 
    | { content: string; truncated: false } 
    | { content: string; truncated: true; outputPath: string }
}
```

### How Truncation Works

```
┌─────────────────────────────────────────────────────────────┐
│                    Tool Output                               │
│  (potentially 100,000+ lines)                               │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                  Truncate.output()                          │
│  Check: lines <= 2000 AND bytes <= 50KB?                   │
└─────────────────────────────────────────────────────────────┘
           │                              │
           │ YES                          │ NO
           ▼                              ▼
┌──────────────────┐         ┌──────────────────────────────┐
│ Return as-is     │         │ 1. Save full output to file  │
│ truncated: false │         │ 2. Return preview + hint     │
└──────────────────┘         │ 3. truncated: true           │
                             │ 4. outputPath: /path/to/file │
                             └──────────────────────────────┘
```

### Truncation Hint

When output is truncated, the AI receives a hint:

```
...1500 lines truncated...

The tool call succeeded but the output was truncated. Full output saved to: /path/to/file
Use the Task tool to have explore agent process this file with Grep and Read (with offset/limit). 
Do NOT read the full file yourself - delegate to save context.
```

## Complete Tool Example

Here's a simplified version of the read tool showing all concepts:

```typescript
export const ReadTool = Tool.define("read", {
  description: "Read the contents of a file...",
  
  parameters: z.object({
    filePath: z.string().describe("The absolute path to the file to read"),
    offset: z.coerce.number().describe("Line number to start from").optional(),
    limit: z.coerce.number().describe("Max lines to read").optional(),
  }),
  
  async execute(params, ctx) {
    // 1. Resolve the path
    let filepath = params.filePath
    if (!path.isAbsolute(filepath)) {
      filepath = path.resolve(Instance.directory, filepath)
    }
    
    // 2. Request permission
    await ctx.ask({
      permission: "read",
      patterns: [filepath],
      always: ["*"],
      metadata: {},
    })
    
    // 3. Check if file exists
    const stat = Filesystem.stat(filepath)
    if (!stat) {
      throw new Error(`File not found: ${filepath}`)
    }
    
    // 4. Read the file content
    const content = await readFileContent(filepath, params)
    
    // 5. Return the result
    return {
      title: path.relative(Instance.worktree, filepath),
      output: formatOutput(filepath, content),
      metadata: {
        preview: content.slice(0, 500),
        truncated: false,
      },
    }
  },
})
```

## Self-Check Questions

1. What are the four main components of a tool definition?
2. Why does `Tool.define` wrap the execute function?
3. What is the purpose of `ctx.ask()` in a tool?
4. When does automatic truncation occur?
5. What information does the truncation hint provide to the AI?

## Exercises

### Exercise 1: Analyze Tool Structure
Look at `packages/opencode/src/tool/glob.ts` and identify:
- The tool ID
- What parameters it accepts
- What permission it requests
- What metadata it returns

### Exercise 2: Trace the Validation Flow
Given this tool call with invalid parameters:
```json
{
  "tool": "read",
  "arguments": {
    "filePath": 123
  }
}
```
Trace through `Tool.define` and explain what error message would be generated.

### Exercise 3: Design a Tool Interface
Design the parameter schema and return type for a hypothetical "compress" tool that:
- Takes a file path and compression level (1-9)
- Returns the compressed file path and compression ratio

## Further Reading

- `packages/opencode/src/tool/tool.ts` - Core tool abstraction
- `packages/opencode/src/tool/truncate.ts` - Output truncation logic
- `packages/opencode/src/tool/schema.ts` - Tool-related schemas
- Zod documentation: https://zod.dev/
