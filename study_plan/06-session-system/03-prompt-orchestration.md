# Module 06: Session System - Lesson 03: Prompt Orchestration

## Learning Objectives

By the end of this lesson, you will be able to:

1. Understand the SessionPrompt module as the "harness" that drives the agent
2. Trace the main `while(true)` loop that orchestrates conversations
3. Explain how tasks/subagents are queued and processed
4. Understand model resolution and tool assembly
5. Follow error handling and retry logic

---

## The Heart of opencode: SessionPrompt

The `SessionPrompt` module in `packages/opencode/src/session/prompt.ts` is the **orchestration layer** that drives conversations. It's the "harness" that:

1. Manages the agent loop
2. Processes queued tasks (subtasks, compactions)
3. Assembles tools from multiple sources
4. Constructs system prompts
5. Invokes the processor for LLM calls
6. Handles errors and retries

```
┌─────────────────────────────────────────────────────────────────┐
│                     SessionPrompt.loop()                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                    while (true)                           │   │
│  │  ┌─────────────────────────────────────────────────────┐ │   │
│  │  │ 1. Load messages (filterCompacted)                  │ │   │
│  │  │ 2. Find last user/assistant messages                │ │   │
│  │  │ 3. Check for pending tasks (subtask, compaction)    │ │   │
│  │  │ 4. Process task OR normal LLM call                  │ │   │
│  │  │ 5. Check finish condition                           │ │   │
│  │  │ 6. Loop or break                                    │ │   │
│  │  └─────────────────────────────────────────────────────┘ │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Session State Management

The module maintains state for active sessions:

```typescript
// From packages/opencode/src/session/prompt.ts

const state = Instance.state(
  () => {
    const data: Record<
      string,
      {
        abort: AbortController
        callbacks: {
          resolve(input: MessageV2.WithParts): void
          reject(reason?: any): void
        }[]
      }
    > = {}
    return data
  },
  async (current) => {
    // Cleanup: abort all sessions on instance teardown
    for (const item of Object.values(current)) {
      item.abort.abort()
    }
  },
)
```

### Starting a Session

```typescript
function start(sessionID: SessionID) {
  const s = state()
  if (s[sessionID]) return  // Already running
  
  const controller = new AbortController()
  s[sessionID] = {
    abort: controller,
    callbacks: [],
  }
  return controller.signal
}
```

### Preventing Concurrent Sessions

```typescript
export function assertNotBusy(sessionID: SessionID) {
  const match = state()[sessionID]
  if (match) throw new Session.BusyError(sessionID)
}
```

---

## The Main Entry Point: prompt()

```typescript
export const prompt = fn(PromptInput, async (input) => {
  const session = await Session.get(input.sessionID)
  await SessionRevert.cleanup(session)

  // Create the user message
  const message = await createUserMessage(input)
  await Session.touch(input.sessionID)

  // Handle legacy tool permissions
  const permissions: Permission.Ruleset = []
  for (const [tool, enabled] of Object.entries(input.tools ?? {})) {
    permissions.push({
      permission: tool,
      action: enabled ? "allow" : "deny",
      pattern: "*",
    })
  }
  if (permissions.length > 0) {
    session.permission = permissions
    await Session.setPermission({ sessionID: session.id, permission: permissions })
  }

  // If noReply, just return the user message
  if (input.noReply === true) {
    return message
  }

  // Start the agent loop
  return loop({ sessionID: input.sessionID })
})
```

---

## The Agent Loop: loop()

This is the **core orchestration loop** that drives the entire agent:

```typescript
export const loop = fn(LoopInput, async (input) => {
  const { sessionID, resume_existing } = input

  // Start or resume the session
  const abort = resume_existing ? resume(sessionID) : start(sessionID)
  if (!abort) {
    // Session already running - queue callback
    return new Promise<MessageV2.WithParts>((resolve, reject) => {
      const callbacks = state()[sessionID].callbacks
      callbacks.push({ resolve, reject })
    })
  }

  // Ensure cleanup on exit
  await using _ = defer(() => cancel(sessionID))

  let structuredOutput: unknown | undefined
  let step = 0
  const session = await Session.get(sessionID)
  
  // ═══════════════════════════════════════════════════════════════
  // THE MAIN LOOP
  // ═══════════════════════════════════════════════════════════════
  while (true) {
    await SessionStatus.set(sessionID, { type: "busy" })
    log.info("loop", { step, sessionID })
    if (abort.aborted) break
    
    // ─────────────────────────────────────────────────────────────
    // STEP 1: Load messages (filtered to after last compaction)
    // ─────────────────────────────────────────────────────────────
    let msgs = await MessageV2.filterCompacted(MessageV2.stream(sessionID))

    // ─────────────────────────────────────────────────────────────
    // STEP 2: Find key messages and pending tasks
    // ─────────────────────────────────────────────────────────────
    let lastUser: MessageV2.User | undefined
    let lastAssistant: MessageV2.Assistant | undefined
    let lastFinished: MessageV2.Assistant | undefined
    let tasks: (MessageV2.CompactionPart | MessageV2.SubtaskPart)[] = []
    
    for (let i = msgs.length - 1; i >= 0; i--) {
      const msg = msgs[i]
      if (!lastUser && msg.info.role === "user") 
        lastUser = msg.info as MessageV2.User
      if (!lastAssistant && msg.info.role === "assistant") 
        lastAssistant = msg.info as MessageV2.Assistant
      if (!lastFinished && msg.info.role === "assistant" && msg.info.finish)
        lastFinished = msg.info as MessageV2.Assistant
      if (lastUser && lastFinished) break
      
      // Collect pending tasks
      const task = msg.parts.filter((part) => 
        part.type === "compaction" || part.type === "subtask"
      )
      if (task && !lastFinished) {
        tasks.push(...task)
      }
    }

    if (!lastUser) throw new Error("No user message found")
    
    // ─────────────────────────────────────────────────────────────
    // STEP 3: Check exit condition
    // ─────────────────────────────────────────────────────────────
    if (
      lastAssistant?.finish &&
      !["tool-calls", "unknown"].includes(lastAssistant.finish) &&
      lastUser.id < lastAssistant.id
    ) {
      log.info("exiting loop", { sessionID })
      break
    }

    step++
    
    // Generate title on first step
    if (step === 1)
      ensureTitle({
        session,
        modelID: lastUser.model.modelID,
        providerID: lastUser.model.providerID,
        history: msgs,
      })

    // ─────────────────────────────────────────────────────────────
    // STEP 4: Resolve model
    // ─────────────────────────────────────────────────────────────
    const model = await Provider.getModel(
      lastUser.model.providerID, 
      lastUser.model.modelID
    ).catch((e) => {
      if (Provider.ModelNotFoundError.isInstance(e)) {
        Bus.publish(Session.Event.Error, {
          sessionID,
          error: new NamedError.Unknown({
            message: `Model not found: ${e.data.providerID}/${e.data.modelID}`,
          }).toObject(),
        })
      }
      throw e
    })
    
    const task = tasks.pop()

    // ─────────────────────────────────────────────────────────────
    // STEP 5A: Process pending SUBTASK
    // ─────────────────────────────────────────────────────────────
    if (task?.type === "subtask") {
      // ... (see subtask processing section below)
      continue
    }

    // ─────────────────────────────────────────────────────────────
    // STEP 5B: Process pending COMPACTION
    // ─────────────────────────────────────────────────────────────
    if (task?.type === "compaction") {
      const result = await SessionCompaction.process({
        messages: msgs,
        parentID: lastUser.id,
        abort,
        sessionID,
        auto: task.auto,
        overflow: task.overflow,
      })
      if (result === "stop") break
      continue
    }

    // ─────────────────────────────────────────────────────────────
    // STEP 5C: Check for context overflow
    // ─────────────────────────────────────────────────────────────
    if (
      lastFinished &&
      lastFinished.summary !== true &&
      (await SessionCompaction.isOverflow({ tokens: lastFinished.tokens, model }))
    ) {
      await SessionCompaction.create({
        sessionID,
        agent: lastUser.agent,
        model: lastUser.model,
        auto: true,
      })
      continue
    }

    // ─────────────────────────────────────────────────────────────
    // STEP 6: Normal LLM processing
    // ─────────────────────────────────────────────────────────────
    const agent = await Agent.get(lastUser.agent)
    if (!agent) throw new Error(`Agent not found: "${lastUser.agent}"`)
    
    const maxSteps = agent.steps ?? Infinity
    const isLastStep = step >= maxSteps
    
    // Insert reminders (plan mode, etc.)
    msgs = await insertReminders({ messages: msgs, agent, session })

    // Create processor
    const processor = SessionProcessor.create({
      assistantMessage: await Session.updateMessage({
        id: MessageID.ascending(),
        parentID: lastUser.id,
        role: "assistant",
        // ... (message fields)
      }) as MessageV2.Assistant,
      sessionID,
      model,
      abort,
    })

    // Resolve tools
    const tools = await resolveTools({
      agent,
      session,
      model,
      tools: lastUser.tools,
      processor,
      bypassAgentCheck,
      messages: msgs,
    })

    // Add structured output tool if needed
    if (lastUser.format?.type === "json_schema") {
      tools["StructuredOutput"] = createStructuredOutputTool({
        schema: lastUser.format.schema,
        onSuccess(output) {
          structuredOutput = output
        },
      })
    }

    // Build system prompt
    const skills = await SystemPrompt.skills(agent)
    const system = [
      ...(await SystemPrompt.environment(model)),
      ...(skills ? [skills] : []),
      ...(await InstructionPrompt.system()),
    ]

    // ─────────────────────────────────────────────────────────────
    // STEP 7: Call the processor
    // ─────────────────────────────────────────────────────────────
    const result = await processor.process({
      user: lastUser,
      agent,
      permission: session.permission,
      abort,
      sessionID,
      system,
      messages: [
        ...MessageV2.toModelMessages(msgs, model),
        ...(isLastStep ? [{ role: "assistant", content: MAX_STEPS }] : []),
      ],
      tools,
      model,
      toolChoice: format.type === "json_schema" ? "required" : undefined,
    })

    // ─────────────────────────────────────────────────────────────
    // STEP 8: Handle result
    // ─────────────────────────────────────────────────────────────
    if (structuredOutput !== undefined) {
      processor.message.structured = structuredOutput
      processor.message.finish = processor.message.finish ?? "stop"
      await Session.updateMessage(processor.message)
      break
    }

    if (result === "stop") break
    if (result === "compact") {
      await SessionCompaction.create({
        sessionID,
        agent: lastUser.agent,
        model: lastUser.model,
        auto: true,
        overflow: !processor.message.finish,
      })
    }
    continue
  }
  
  // Cleanup and return
  SessionCompaction.prune({ sessionID })
  for await (const item of MessageV2.stream(sessionID)) {
    if (item.info.role === "user") continue
    const queued = state()[sessionID]?.callbacks ?? []
    for (const q of queued) {
      q.resolve(item)
    }
    return item
  }
  throw new Error("Impossible")
})
```

---

## Subtask Processing

When a subtask (Task tool invocation) is queued:

```typescript
if (task?.type === "subtask") {
  const taskTool = await TaskTool.init()
  const taskModel = task.model 
    ? await Provider.getModel(task.model.providerID, task.model.modelID) 
    : model
    
  // Create assistant message for the subtask
  const assistantMessage = await Session.updateMessage({
    id: MessageID.ascending(),
    role: "assistant",
    parentID: lastUser.id,
    sessionID,
    agent: task.agent,
    // ... other fields
  }) as MessageV2.Assistant
  
  // Create tool part in running state
  let part = await Session.updatePart({
    id: PartID.ascending(),
    messageID: assistantMessage.id,
    sessionID: assistantMessage.sessionID,
    type: "tool",
    callID: ulid(),
    tool: TaskTool.id,
    state: {
      status: "running",
      input: {
        prompt: task.prompt,
        description: task.description,
        subagent_type: task.agent,
        command: task.command,
      },
      time: { start: Date.now() },
    },
  }) as MessageV2.ToolPart
  
  // Execute the task
  const taskAgent = await Agent.get(task.agent)
  const result = await taskTool.execute(taskArgs, taskCtx).catch((error) => {
    executionError = error
    return undefined
  })
  
  // Update part with result
  if (result && part.state.status === "running") {
    await Session.updatePart({
      ...part,
      state: {
        status: "completed",
        input: part.state.input,
        title: result.title,
        metadata: result.metadata,
        output: result.output,
        attachments,
        time: { ...part.state.time, end: Date.now() },
      },
    })
  }
  
  continue  // Loop again
}
```

---

## Tool Assembly: resolveTools()

Tools are assembled from multiple sources:

```typescript
export async function resolveTools(input: {
  agent: Agent.Info
  model: Provider.Model
  session: Session.Info
  tools?: Record<string, boolean>
  processor: SessionProcessor.Info
  bypassAgentCheck: boolean
  messages: MessageV2.WithParts[]
}) {
  const tools: Record<string, AITool> = {}

  // Helper to create tool context
  const context = (args: any, options: ToolCallOptions): Tool.Context => ({
    sessionID: input.session.id,
    abort: options.abortSignal!,
    messageID: input.processor.message.id,
    callID: options.toolCallId,
    extra: { model: input.model, bypassAgentCheck: input.bypassAgentCheck },
    agent: input.agent.name,
    messages: input.messages,
    metadata: async (val) => { /* update part metadata */ },
    async ask(req) {
      await Permission.ask({
        ...req,
        sessionID: input.session.id,
        ruleset: Permission.merge(input.agent.permission, input.session.permission ?? []),
      })
    },
  })

  // ─────────────────────────────────────────────────────────────
  // SOURCE 1: Tool Registry (built-in tools)
  // ─────────────────────────────────────────────────────────────
  for (const item of await ToolRegistry.tools(
    { modelID: ModelID.make(input.model.api.id), providerID: input.model.providerID },
    input.agent,
  )) {
    const schema = ProviderTransform.schema(input.model, z.toJSONSchema(item.parameters))
    tools[item.id] = tool({
      id: item.id as any,
      description: item.description,
      inputSchema: jsonSchema(schema as any),
      async execute(args, options) {
        const ctx = context(args, options)
        await Plugin.trigger("tool.execute.before", { tool: item.id, ... }, { args })
        const result = await item.execute(args, ctx)
        await Plugin.trigger("tool.execute.after", { tool: item.id, ... }, result)
        return result
      },
    })
  }

  // ─────────────────────────────────────────────────────────────
  // SOURCE 2: MCP Tools
  // ─────────────────────────────────────────────────────────────
  for (const [key, item] of Object.entries(await MCP.tools())) {
    const execute = item.execute
    if (!execute) continue

    const transformed = ProviderTransform.schema(input.model, asSchema(item.inputSchema).jsonSchema)
    item.inputSchema = jsonSchema(transformed)
    
    item.execute = async (args, opts) => {
      const ctx = context(args, opts)
      await Plugin.trigger("tool.execute.before", { tool: key, ... }, { args })
      
      // MCP tools require permission
      await ctx.ask({
        permission: key,
        metadata: {},
        patterns: ["*"],
        always: ["*"],
      })

      const result = await execute(args, opts)
      await Plugin.trigger("tool.execute.after", { tool: key, ... }, result)
      
      // Format MCP result
      const textParts: string[] = []
      const attachments: Omit<MessageV2.FilePart, "id" | "sessionID" | "messageID">[] = []

      for (const contentItem of result.content) {
        if (contentItem.type === "text") {
          textParts.push(contentItem.text)
        } else if (contentItem.type === "image") {
          attachments.push({
            type: "file",
            mime: contentItem.mimeType,
            url: `data:${contentItem.mimeType};base64,${contentItem.data}`,
          })
        }
        // ... handle other content types
      }

      return {
        title: "",
        metadata,
        output: truncated.content,
        attachments,
      }
    }
    tools[key] = item
  }

  return tools
}
```

---

## System Prompt Construction

```typescript
// From packages/opencode/src/session/system.ts

export namespace SystemPrompt {
  // Select provider-specific base prompt
  export function provider(model: Provider.Model) {
    if (model.api.id.includes("gpt-4") || model.api.id.includes("o1")) 
      return [PROMPT_BEAST]
    if (model.api.id.includes("gemini-")) 
      return [PROMPT_GEMINI]
    if (model.api.id.includes("claude")) 
      return [PROMPT_ANTHROPIC]
    return [PROMPT_DEFAULT]
  }

  // Add environment information
  export async function environment(model: Provider.Model) {
    const project = Instance.project
    return [
      [
        `You are powered by the model named ${model.api.id}.`,
        `<env>`,
        `  Working directory: ${Instance.directory}`,
        `  Workspace root folder: ${Instance.worktree}`,
        `  Is directory a git repo: ${project.vcs === "git" ? "yes" : "no"}`,
        `  Platform: ${process.platform}`,
        `  Today's date: ${new Date().toDateString()}`,
        `</env>`,
      ].join("\n"),
    ]
  }

  // Add available skills
  export async function skills(agent: Agent.Info) {
    if (Permission.disabled(["skill"], agent.permission).has("skill")) return

    const list = await Skill.available(agent)
    return [
      "Skills provide specialized instructions for specific tasks.",
      "Use the skill tool to load a skill when a task matches its description.",
      Skill.fmt(list, { verbose: true }),
    ].join("\n")
  }
}
```

---

## Structured Output Handling

When JSON schema output is requested:

```typescript
export function createStructuredOutputTool(input: {
  schema: Record<string, any>
  onSuccess: (output: unknown) => void
}): AITool {
  const { $schema, ...toolSchema } = input.schema

  return tool({
    id: "StructuredOutput" as any,
    description: `Use this tool to return your final response in the requested structured format.

IMPORTANT:
- You MUST call this tool exactly once at the end of your response
- The input must be valid JSON matching the required schema
- Complete all necessary research and tool calls BEFORE calling this tool
- This tool provides your final answer - no further actions are taken after calling it`,
    inputSchema: jsonSchema(toolSchema as any),
    async execute(args) {
      input.onSuccess(args)
      return {
        output: "Structured output captured successfully.",
        title: "Structured Output",
        metadata: { valid: true },
      }
    },
  })
}
```

---

## Cancellation

```typescript
export async function cancel(sessionID: SessionID) {
  log.info("cancel", { sessionID })
  const s = state()
  const match = s[sessionID]
  if (!match) {
    await SessionStatus.set(sessionID, { type: "idle" })
    return
  }
  match.abort.abort()
  delete s[sessionID]
  await SessionStatus.set(sessionID, { type: "idle" })
  return
}
```

---

## Loop Exit Conditions

The loop exits when:

1. **Abort signal fired**: `if (abort.aborted) break`
2. **Model finished**: `lastAssistant.finish` is not `"tool-calls"` or `"unknown"`
3. **Structured output captured**: `structuredOutput !== undefined`
4. **Processor returns "stop"**: Error or permission denied
5. **Compaction fails**: `result === "stop"` from compaction

---

## Self-Check Questions

1. **What happens if you call `prompt()` on a session that's already running?**
   <details>
   <summary>Answer</summary>
   The call queues a callback and returns a Promise that resolves when the current loop completes. The new prompt is not processed until the current one finishes.
   </details>

2. **How are subtasks different from regular tool calls?**
   <details>
   <summary>Answer</summary>
   Subtasks are queued as `SubtaskPart` and processed at the start of the next loop iteration, creating a child session. Regular tool calls are processed inline during streaming.
   </details>

3. **What triggers automatic compaction?**
   <details>
   <summary>Answer</summary>
   Compaction is triggered when `SessionCompaction.isOverflow()` returns true, which checks if token usage exceeds the model's context limit minus a buffer.
   </details>

4. **How does the system handle model not found errors?**
   <details>
   <summary>Answer</summary>
   It catches `Provider.ModelNotFoundError`, publishes a `Session.Event.Error` with suggestions, and re-throws the error to stop the loop.
   </details>

---

## Exercises

1. **Trace a full loop**: Add logging at each step and trace a simple prompt through the entire loop.

2. **Examine tool resolution**: Log the tools assembled for different agents and compare.

3. **Test structured output**: Use the API to request JSON schema output and trace how it's handled.

---

## Further Reading

- `packages/opencode/src/agent/agent.ts` - Agent configuration
- `packages/opencode/src/tool/registry.ts` - Tool registry
- `packages/opencode/src/mcp/index.ts` - MCP integration
- `packages/opencode/src/permission/index.ts` - Permission system
