# Lesson 02: Built-in Tools

## Learning Objectives

By the end of this lesson, you will be able to:
- List and describe all built-in tools in opencode
- Understand how the tool registry manages tools
- Deep dive into key tools: bash, read, write, edit, glob, grep, and task
- Explain how each tool validates input and produces output

## Tool Registry Overview

The tool registry (`packages/opencode/src/tool/registry.ts`) manages all available tools. It:
- Maintains a list of built-in tools
- Supports custom tools from plugins
- Filters tools based on model capabilities
- Initializes tools with agent context

```typescript
async function all(custom: Tool.Info[]): Promise<Tool.Info[]> {
  const cfg = await Config.get()
  const question = ["app", "cli", "desktop"].includes(Flag.OPENCODE_CLIENT) || Flag.OPENCODE_ENABLE_QUESTION_TOOL

  return [
    InvalidTool,
    ...(question ? [QuestionTool] : []),
    BashTool,
    ReadTool,
    GlobTool,
    GrepTool,
    EditTool,
    WriteTool,
    TaskTool,
    WebFetchTool,
    TodoWriteTool,
    WebSearchTool,
    CodeSearchTool,
    SkillTool,
    ApplyPatchTool,
    ...(Flag.OPENCODE_EXPERIMENTAL_LSP_TOOL ? [LspTool] : []),
    ...(cfg.experimental?.batch_tool === true ? [BatchTool] : []),
    ...(Flag.OPENCODE_EXPERIMENTAL_PLAN_MODE && Flag.OPENCODE_CLIENT === "cli" ? [PlanExitTool] : []),
    ...custom,
  ]
}
```

## Built-in Tools Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Built-in Tools                                │
├─────────────────────────────────────────────────────────────────────┤
│  File Operations        │  Code Search          │  Execution        │
│  ─────────────────      │  ───────────          │  ─────────        │
│  • read                 │  • grep               │  • bash           │
│  • write                │  • glob               │  • task           │
│  • edit                 │  • codesearch         │                   │
│  • apply_patch          │                       │                   │
├─────────────────────────┼───────────────────────┼───────────────────┤
│  Web                    │  Planning             │  Other            │
│  ───                    │  ────────             │  ─────            │
│  • webfetch             │  • todowrite          │  • question       │
│  • websearch            │  • plan               │  • skill          │
│                         │                       │  • lsp            │
└─────────────────────────────────────────────────────────────────────┘
```

## Deep Dive: bash.ts - Shell Command Execution

The bash tool executes shell commands with sophisticated parsing and permission handling.

### Key Features

1. **Tree-sitter Parsing**: Parses commands to understand their structure
2. **Command Arity Detection**: Identifies the "meaningful" part of commands
3. **External Directory Detection**: Detects when commands access files outside the project
4. **Timeout Handling**: Kills long-running commands
5. **Real-time Output Streaming**: Updates UI as output arrives

### Implementation Highlights

From `packages/opencode/src/tool/bash.ts`:

```typescript
export const BashTool = Tool.define("bash", async () => {
  const shell = Shell.acceptable()
  
  return {
    description: DESCRIPTION.replaceAll("${directory}", Instance.directory)
      .replaceAll("${maxLines}", String(Truncate.MAX_LINES))
      .replaceAll("${maxBytes}", String(Truncate.MAX_BYTES)),
    
    parameters: z.object({
      command: z.string().describe("The command to execute"),
      timeout: z.number().describe("Optional timeout in milliseconds").optional(),
      workdir: z.string().describe("The working directory").optional(),
      description: z.string().describe("Clear, concise description of what this command does"),
    }),
    
    async execute(params, ctx) {
      const cwd = params.workdir || Instance.directory
      const timeout = params.timeout ?? DEFAULT_TIMEOUT  // 2 minutes default
      
      // Parse command with tree-sitter
      const tree = await parser().then((p) => p.parse(params.command))
      
      // Detect external directories
      const directories = new Set<string>()
      for (const node of tree.rootNode.descendantsOfType("command")) {
        // Check cd, rm, cp, mv, mkdir, touch, chmod, chown, cat
        if (["cd", "rm", "cp", "mv", ...].includes(command[0])) {
          for (const arg of command.slice(1)) {
            const resolved = await fs.realpath(path.resolve(cwd, arg))
            if (!Instance.containsPath(resolved)) {
              directories.add(resolved)
            }
          }
        }
      }
      
      // Request permission for external directories
      if (directories.size > 0) {
        await ctx.ask({
          permission: "external_directory",
          patterns: globs,
          always: globs,
          metadata: {},
        })
      }
      
      // Request permission for the command itself
      await ctx.ask({
        permission: "bash",
        patterns: Array.from(patterns),
        always: Array.from(always),
        metadata: {},
      })
      
      // Execute the command
      const proc = spawn(params.command, {
        shell,
        cwd,
        env: { ...process.env, ...shellEnv.env },
        stdio: ["ignore", "pipe", "pipe"],
      })
      
      // Stream output to UI
      proc.stdout?.on("data", (chunk) => {
        output += chunk.toString()
        ctx.metadata({ metadata: { output, description: params.description } })
      })
      
      // Handle timeout
      const timeoutTimer = setTimeout(() => {
        timedOut = true
        Shell.killTree(proc)
      }, timeout)
      
      await new Promise((resolve) => proc.once("exit", resolve))
      
      return {
        title: params.description,
        metadata: { output, exit: proc.exitCode, description: params.description },
        output,
      }
    },
  }
})
```

### Command Arity

The `BashArity` namespace determines how much of a command is "meaningful":

```typescript
// From packages/opencode/src/permission/arity.ts
export namespace BashArity {
  export function prefix(tokens: string[]) {
    for (let len = tokens.length; len > 0; len--) {
      const prefix = tokens.slice(0, len).join(" ")
      const arity = ARITY[prefix]
      if (arity !== undefined) return tokens.slice(0, arity)
    }
    return tokens.slice(0, 1)
  }

  const ARITY: Record<string, number> = {
    git: 2,           // "git checkout" is the command
    "npm run": 3,     // "npm run dev" is the command
    docker: 2,        // "docker run" is the command
    kubectl: 2,       // "kubectl get" is the command
    // ... many more
  }
}
```

## Deep Dive: read.ts - File Reading

The read tool handles files, directories, images, and PDFs.

### Key Features

1. **Path Resolution**: Handles relative and absolute paths
2. **Directory Listing**: Lists directory contents when given a directory
3. **Binary Detection**: Refuses to read binary files
4. **Image/PDF Support**: Returns as base64 attachments
5. **Line Numbering**: Adds line numbers to output
6. **Pagination**: Supports offset and limit parameters

### Implementation Highlights

From `packages/opencode/src/tool/read.ts`:

```typescript
export const ReadTool = Tool.define("read", {
  description: DESCRIPTION,
  
  parameters: z.object({
    filePath: z.string().describe("The absolute path to the file or directory"),
    offset: z.coerce.number().describe("Line number to start from (1-indexed)").optional(),
    limit: z.coerce.number().describe("Max lines to read (default 2000)").optional(),
  }),
  
  async execute(params, ctx) {
    let filepath = params.filePath
    if (!path.isAbsolute(filepath)) {
      filepath = path.resolve(Instance.directory, filepath)
    }
    
    // Check for external directory access
    await assertExternalDirectory(ctx, filepath)
    
    // Request read permission
    await ctx.ask({
      permission: "read",
      patterns: [filepath],
      always: ["*"],
      metadata: {},
    })
    
    const stat = Filesystem.stat(filepath)
    if (!stat) {
      // Suggest similar files
      const suggestions = await findSimilarFiles(filepath)
      throw new Error(`File not found: ${filepath}\n\nDid you mean?\n${suggestions.join("\n")}`)
    }
    
    // Handle directories
    if (stat.isDirectory()) {
      const entries = await fs.readdir(filepath, { withFileTypes: true })
      return {
        title: path.relative(Instance.worktree, filepath),
        output: formatDirectoryListing(entries),
        metadata: { truncated: false },
      }
    }
    
    // Handle images and PDFs
    const mime = Filesystem.mimeType(filepath)
    if (mime.startsWith("image/") || mime === "application/pdf") {
      return {
        title: filepath,
        output: "Image read successfully",
        metadata: { truncated: false },
        attachments: [{
          type: "file",
          mime,
          url: `data:${mime};base64,${base64Content}`,
        }],
      }
    }
    
    // Handle binary files
    if (await isBinaryFile(filepath, stat.size)) {
      throw new Error(`Cannot read binary file: ${filepath}`)
    }
    
    // Read text file with pagination
    const limit = params.limit ?? 2000
    const offset = params.offset ?? 1
    const lines = await readLines(filepath, offset, limit)
    
    // Add line numbers
    const content = lines.map((line, i) => `${i + offset}: ${line}`)
    
    return {
      title: path.relative(Instance.worktree, filepath),
      output: formatFileContent(filepath, content),
      metadata: { preview: content.slice(0, 20).join("\n"), truncated },
    }
  },
})
```

## Deep Dive: write.ts - File Writing

The write tool creates or overwrites files with LSP integration.

### Key Features

1. **Diff Generation**: Shows what will change before writing
2. **Auto-formatting**: Formats files after writing
3. **LSP Diagnostics**: Reports any errors introduced
4. **Event Publishing**: Notifies the system of file changes

### Implementation Highlights

From `packages/opencode/src/tool/write.ts`:

```typescript
export const WriteTool = Tool.define("write", {
  description: DESCRIPTION,
  
  parameters: z.object({
    content: z.string().describe("The content to write"),
    filePath: z.string().describe("The absolute path to the file"),
  }),
  
  async execute(params, ctx) {
    const filepath = path.isAbsolute(params.filePath) 
      ? params.filePath 
      : path.join(Instance.directory, params.filePath)
    
    await assertExternalDirectory(ctx, filepath)
    
    const exists = await Filesystem.exists(filepath)
    const contentOld = exists ? await Filesystem.readText(filepath) : ""
    
    // Generate diff for permission request
    const diff = trimDiff(createTwoFilesPatch(filepath, filepath, contentOld, params.content))
    
    await ctx.ask({
      permission: "edit",
      patterns: [path.relative(Instance.worktree, filepath)],
      always: ["*"],
      metadata: { filepath, diff },
    })
    
    // Write and format
    await Filesystem.write(filepath, params.content)
    await Format.file(filepath)
    
    // Publish events
    Bus.publish(File.Event.Edited, { file: filepath })
    await Bus.publish(FileWatcher.Event.Updated, {
      file: filepath,
      event: exists ? "change" : "add",
    })
    
    // Check for LSP errors
    let output = "Wrote file successfully."
    await LSP.touchFile(filepath, true)
    const diagnostics = await LSP.diagnostics()
    
    const errors = diagnostics[filepath]?.filter((d) => d.severity === 1)
    if (errors?.length > 0) {
      output += `\n\nLSP errors detected:\n${errors.map(LSP.Diagnostic.pretty).join("\n")}`
    }
    
    return {
      title: path.relative(Instance.worktree, filepath),
      metadata: { diagnostics, filepath, exists },
      output,
    }
  },
})
```

## Deep Dive: edit.ts - File Editing (str_replace)

The edit tool performs surgical text replacements with multiple fallback strategies.

### Key Features

1. **Multiple Replacement Strategies**: Tries various matching approaches
2. **Line Ending Normalization**: Handles CRLF/LF differences
3. **Fuzzy Matching**: Can match despite whitespace differences
4. **Replace All Support**: Can replace all occurrences

### Replacement Strategies

The edit tool tries these strategies in order:

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Replacement Strategies                            │
├─────────────────────────────────────────────────────────────────────┤
│  1. SimpleReplacer          │ Exact string match                    │
│  2. LineTrimmedReplacer     │ Match ignoring leading/trailing space │
│  3. BlockAnchorReplacer     │ Match by first/last line anchors      │
│  4. WhitespaceNormalizedReplacer │ Normalize all whitespace         │
│  5. IndentationFlexibleReplacer  │ Ignore indentation differences   │
│  6. EscapeNormalizedReplacer     │ Handle escape sequences          │
│  7. TrimmedBoundaryReplacer      │ Trim boundaries                  │
│  8. ContextAwareReplacer         │ Use context for matching         │
│  9. MultiOccurrenceReplacer      │ Find all occurrences             │
└─────────────────────────────────────────────────────────────────────┘
```

### Implementation Highlights

From `packages/opencode/src/tool/edit.ts`:

```typescript
export const EditTool = Tool.define("edit", {
  description: DESCRIPTION,
  
  parameters: z.object({
    filePath: z.string().describe("The absolute path to the file"),
    oldString: z.string().describe("The text to replace"),
    newString: z.string().describe("The replacement text"),
    replaceAll: z.boolean().optional().describe("Replace all occurrences"),
  }),
  
  async execute(params, ctx) {
    if (params.oldString === params.newString) {
      throw new Error("No changes to apply: oldString and newString are identical.")
    }
    
    const filePath = path.isAbsolute(params.filePath) 
      ? params.filePath 
      : path.join(Instance.directory, params.filePath)
    
    await FileTime.withLock(filePath, async () => {
      // Handle new file creation (oldString is empty)
      if (params.oldString === "") {
        await Filesystem.write(filePath, params.newString)
        return
      }
      
      const contentOld = await Filesystem.readText(filePath)
      
      // Normalize line endings
      const ending = detectLineEnding(contentOld)
      const old = convertToLineEnding(normalizeLineEndings(params.oldString), ending)
      const next = convertToLineEnding(normalizeLineEndings(params.newString), ending)
      
      // Try replacement with fallback strategies
      const contentNew = replace(contentOld, old, next, params.replaceAll)
      
      // Generate diff and request permission
      const diff = trimDiff(createTwoFilesPatch(filePath, filePath, contentOld, contentNew))
      await ctx.ask({
        permission: "edit",
        patterns: [path.relative(Instance.worktree, filePath)],
        always: ["*"],
        metadata: { filepath: filePath, diff },
      })
      
      await Filesystem.write(filePath, contentNew)
      await Format.file(filePath)
    })
    
    return {
      title: path.relative(Instance.worktree, filePath),
      metadata: { diff, diagnostics },
      output: "Edit applied successfully.",
    }
  },
})

// The replace function tries multiple strategies
export function replace(content: string, oldString: string, newString: string, replaceAll = false): string {
  for (const replacer of [
    SimpleReplacer,
    LineTrimmedReplacer,
    BlockAnchorReplacer,
    WhitespaceNormalizedReplacer,
    IndentationFlexibleReplacer,
    EscapeNormalizedReplacer,
    TrimmedBoundaryReplacer,
    ContextAwareReplacer,
    MultiOccurrenceReplacer,
  ]) {
    for (const search of replacer(content, oldString)) {
      const index = content.indexOf(search)
      if (index === -1) continue
      
      if (replaceAll) {
        return content.replaceAll(search, newString)
      }
      
      // Ensure unique match
      const lastIndex = content.lastIndexOf(search)
      if (index !== lastIndex) continue
      
      return content.substring(0, index) + newString + content.substring(index + search.length)
    }
  }
  
  throw new Error("Could not find oldString in the file.")
}
```

## Deep Dive: glob.ts - File Pattern Matching

The glob tool finds files matching patterns using ripgrep.

### Implementation

From `packages/opencode/src/tool/glob.ts`:

```typescript
export const GlobTool = Tool.define("glob", {
  description: DESCRIPTION,
  
  parameters: z.object({
    pattern: z.string().describe("The glob pattern to match"),
    path: z.string().optional().describe("Directory to search in"),
  }),
  
  async execute(params, ctx) {
    await ctx.ask({
      permission: "glob",
      patterns: [params.pattern],
      always: ["*"],
      metadata: { pattern: params.pattern, path: params.path },
    })
    
    let search = params.path ?? Instance.directory
    search = path.isAbsolute(search) ? search : path.resolve(Instance.directory, search)
    
    const limit = 100
    const files = []
    
    for await (const file of Ripgrep.files({
      cwd: search,
      glob: [params.pattern],
      signal: ctx.abort,
    })) {
      if (files.length >= limit) break
      
      const full = path.resolve(search, file)
      const stats = Filesystem.stat(full)?.mtime.getTime() ?? 0
      files.push({ path: full, mtime: stats })
    }
    
    // Sort by modification time (newest first)
    files.sort((a, b) => b.mtime - a.mtime)
    
    return {
      title: path.relative(Instance.worktree, search),
      metadata: { count: files.length, truncated: files.length >= limit },
      output: files.map((f) => f.path).join("\n"),
    }
  },
})
```

## Deep Dive: grep.ts - Code Searching

The grep tool searches file contents using ripgrep with regex support.

### Implementation

From `packages/opencode/src/tool/grep.ts`:

```typescript
export const GrepTool = Tool.define("grep", {
  description: DESCRIPTION,
  
  parameters: z.object({
    pattern: z.string().describe("The regex pattern to search for"),
    path: z.string().optional().describe("Directory to search in"),
    include: z.string().optional().describe("File pattern to include"),
  }),
  
  async execute(params, ctx) {
    await ctx.ask({
      permission: "grep",
      patterns: [params.pattern],
      always: ["*"],
      metadata: { pattern: params.pattern, path: params.path, include: params.include },
    })
    
    const rgPath = await Ripgrep.filepath()
    const args = ["-nH", "--hidden", "--no-messages", "--field-match-separator=|", 
                  "--regexp", params.pattern]
    
    if (params.include) {
      args.push("--glob", params.include)
    }
    args.push(searchPath)
    
    const proc = Process.spawn([rgPath, ...args], {
      stdout: "pipe",
      stderr: "pipe",
      abort: ctx.abort,
    })
    
    const output = await text(proc.stdout)
    const exitCode = await proc.exited
    
    // Parse ripgrep output
    const matches = []
    for (const line of output.trim().split(/\r?\n/)) {
      const [filePath, lineNumStr, ...lineTextParts] = line.split("|")
      matches.push({
        path: filePath,
        lineNum: parseInt(lineNumStr, 10),
        lineText: lineTextParts.join("|"),
        modTime: Filesystem.stat(filePath)?.mtime.getTime() ?? 0,
      })
    }
    
    // Sort by modification time
    matches.sort((a, b) => b.modTime - a.modTime)
    
    return {
      title: params.pattern,
      metadata: { matches: matches.length, truncated: matches.length > 100 },
      output: formatMatches(matches.slice(0, 100)),
    }
  },
})
```

## Deep Dive: task.ts - Subagent Spawning

The task tool creates subagent sessions for delegating work.

### Key Features

1. **Agent Selection**: Chooses appropriate agent type
2. **Session Management**: Creates or resumes sessions
3. **Permission Inheritance**: Passes permission rules to subagent
4. **Cancellation Support**: Can cancel running subagents

### Implementation Highlights

From `packages/opencode/src/tool/task.ts`:

```typescript
export const TaskTool = Tool.define("task", async (ctx) => {
  const agents = await Agent.list().then((x) => x.filter((a) => a.mode !== "primary"))
  
  // Filter agents by permissions
  const caller = ctx?.agent
  const accessibleAgents = caller
    ? agents.filter((a) => Permission.evaluate("task", a.name, caller.permission).action !== "deny")
    : agents
  
  return {
    description: DESCRIPTION.replace("{agents}", formatAgentList(accessibleAgents)),
    
    parameters: z.object({
      description: z.string().describe("Short (3-5 words) task description"),
      prompt: z.string().describe("The task for the agent"),
      subagent_type: z.string().describe("Type of agent to use"),
      task_id: z.string().optional().describe("ID to resume a previous task"),
    }),
    
    async execute(params, ctx) {
      // Request permission unless bypassed
      if (!ctx.extra?.bypassAgentCheck) {
        await ctx.ask({
          permission: "task",
          patterns: [params.subagent_type],
          always: ["*"],
          metadata: { description: params.description, subagent_type: params.subagent_type },
        })
      }
      
      const agent = await Agent.get(params.subagent_type)
      if (!agent) throw new Error(`Unknown agent type: ${params.subagent_type}`)
      
      // Create or resume session
      const session = params.task_id
        ? await Session.get(SessionID.make(params.task_id)).catch(() => null)
        : null
      
      const newSession = session ?? await Session.create({
        parentID: ctx.sessionID,
        title: params.description + ` (@${agent.name} subagent)`,
        permission: buildPermissionRules(agent),
      })
      
      // Set up cancellation
      ctx.abort.addEventListener("abort", () => SessionPrompt.cancel(newSession.id))
      
      // Run the prompt
      const result = await SessionPrompt.prompt({
        messageID: MessageID.ascending(),
        sessionID: newSession.id,
        model: agent.model ?? parentModel,
        agent: agent.name,
        parts: await SessionPrompt.resolvePromptParts(params.prompt),
      })
      
      const text = result.parts.findLast((x) => x.type === "text")?.text ?? ""
      
      return {
        title: params.description,
        metadata: { sessionId: newSession.id, model },
        output: `task_id: ${newSession.id}\n\n<task_result>\n${text}\n</task_result>`,
      }
    },
  }
})
```

## Other Built-in Tools

### webfetch.ts - URL Fetching

Fetches web pages and converts to markdown:

```typescript
parameters: z.object({
  url: z.string().describe("The URL to fetch"),
  format: z.enum(["text", "markdown", "html"]).default("markdown"),
  timeout: z.number().optional(),
})
```

### websearch.ts - Web Search

Searches the web using the Exa API:

```typescript
parameters: z.object({
  query: z.string().describe("Search query"),
  numResults: z.number().optional(),
  livecrawl: z.enum(["fallback", "preferred"]).optional(),
  type: z.enum(["auto", "fast", "deep"]).optional(),
})
```

### todowrite.ts - Task Management

Updates the session's todo list:

```typescript
parameters: z.object({
  todos: z.array(z.object(Todo.Info.shape)).describe("The updated todo list"),
})
```

## Self-Check Questions

1. How does the bash tool detect when a command accesses external directories?
2. What happens when the read tool encounters a binary file?
3. Why does the edit tool have multiple replacement strategies?
4. How does the grep tool sort its results?
5. What is the purpose of `task_id` in the task tool?

## Exercises

### Exercise 1: Tool Comparison
Compare the permission requests made by `read.ts`, `write.ts`, and `edit.ts`. What patterns do you notice?

### Exercise 2: Trace a Grep Search
Given this grep call:
```json
{
  "tool": "grep",
  "arguments": {
    "pattern": "function.*export",
    "include": "*.ts"
  }
}
```
Trace through the code and describe what ripgrep command gets executed.

### Exercise 3: Edit Strategy Analysis
Given a file with this content:
```typescript
function hello() {
  console.log("hello")
}
```
And this edit request:
```json
{
  "oldString": "  console.log(\"hello\")",
  "newString": "  console.log(\"world\")"
}
```
Which replacement strategy would succeed? Why might others fail?

## Further Reading

- `packages/opencode/src/tool/registry.ts` - Tool registration
- `packages/opencode/src/tool/bash.txt` - Bash tool description
- `packages/opencode/src/tool/read.txt` - Read tool description
- `packages/opencode/src/file/ripgrep.ts` - Ripgrep integration
- `packages/opencode/src/shell/shell.ts` - Shell utilities
