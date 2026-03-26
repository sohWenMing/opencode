# Module 06: Session System - Lesson 05: Compaction & Context Management

## Learning Objectives

By the end of this lesson, you will be able to:

1. Understand why compaction is needed (context window limits)
2. Explain how the compaction process works
3. Trace when compaction is triggered
4. Understand the pruning mechanism for old tool outputs
5. Follow the summary generation process

---

## Why Compaction?

LLMs have finite context windows. When a conversation grows too long:

1. **Context overflow**: The API rejects the request
2. **Performance degradation**: Longer contexts are slower and more expensive
3. **Relevance decay**: Old information becomes less useful

Compaction solves this by **summarizing** the conversation history, allowing the agent to continue with a condensed context.

```
┌─────────────────────────────────────────────────────────────────┐
│                    BEFORE COMPACTION                             │
├─────────────────────────────────────────────────────────────────┤
│  [User 1] [Asst 1] [User 2] [Asst 2] ... [User N] [Asst N]     │
│                                                                  │
│  Total tokens: 180,000 (exceeds 128K limit)                     │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    AFTER COMPACTION                              │
├─────────────────────────────────────────────────────────────────┤
│  [CompactionPart] [Summary Message] [User N+1] ...              │
│                                                                  │
│  Total tokens: 15,000 (within limit)                            │
└─────────────────────────────────────────────────────────────────┘
```

---

## Overflow Detection

The system detects when compaction is needed:

```typescript
// From packages/opencode/src/session/compaction.ts

const COMPACTION_BUFFER = 20_000

export async function isOverflow(input: { 
  tokens: MessageV2.Assistant["tokens"]
  model: Provider.Model 
}) {
  const config = await Config.get()
  
  // Compaction can be disabled
  if (config.compaction?.auto === false) return false
  
  const context = input.model.limit.context
  if (context === 0) return false  // No limit

  // Calculate total token usage
  const count =
    input.tokens.total ||
    input.tokens.input + input.tokens.output + input.tokens.cache.read + input.tokens.cache.write

  // Calculate usable context
  const reserved =
    config.compaction?.reserved ?? 
    Math.min(COMPACTION_BUFFER, ProviderTransform.maxOutputTokens(input.model))
  const usable = input.model.limit.input
    ? input.model.limit.input - reserved
    : context - ProviderTransform.maxOutputTokens(input.model)
    
  return count >= usable
}
```

### Overflow Triggers

| Trigger | Location | Condition |
|---------|----------|-----------|
| After step finish | `processor.ts` | `isOverflow()` returns true |
| API error | `processor.ts` | `ContextOverflowError` caught |
| Loop check | `prompt.ts` | `lastFinished.tokens` exceeds limit |

---

## Creating a Compaction Request

When compaction is needed, a marker is added to the conversation:

```typescript
// From packages/opencode/src/session/compaction.ts

export const create = fn(
  z.object({
    sessionID: SessionID.zod,
    agent: z.string(),
    model: z.object({
      providerID: ProviderID.zod,
      modelID: ModelID.zod,
    }),
    auto: z.boolean(),
    overflow: z.boolean().optional(),
  }),
  async (input) => {
    // Create a user message with compaction marker
    const msg = await Session.updateMessage({
      id: MessageID.ascending(),
      role: "user",
      model: input.model,
      sessionID: input.sessionID,
      agent: input.agent,
      time: { created: Date.now() },
    })
    
    // Add CompactionPart
    await Session.updatePart({
      id: PartID.ascending(),
      messageID: msg.id,
      sessionID: msg.sessionID,
      type: "compaction",
      auto: input.auto,
      overflow: input.overflow,
    })
  },
)
```

The `CompactionPart` signals to the loop that compaction should be processed on the next iteration.

---

## Processing Compaction

```typescript
// From packages/opencode/src/session/compaction.ts

export async function process(input: {
  parentID: MessageID
  messages: MessageV2.WithParts[]
  sessionID: SessionID
  abort: AbortSignal
  auto: boolean
  overflow?: boolean
}) {
  const userMessage = input.messages.findLast((m) => m.info.id === input.parentID)!.info as MessageV2.User

  let messages = input.messages
  let replay: MessageV2.WithParts | undefined
  
  // ─────────────────────────────────────────────────────────────
  // Handle overflow: find a message to replay after compaction
  // ─────────────────────────────────────────────────────────────
  if (input.overflow) {
    const idx = input.messages.findIndex((m) => m.info.id === input.parentID)
    for (let i = idx - 1; i >= 0; i--) {
      const msg = input.messages[i]
      if (msg.info.role === "user" && !msg.parts.some((p) => p.type === "compaction")) {
        replay = msg
        messages = input.messages.slice(0, i)
        break
      }
    }
    // Ensure there's content to summarize
    const hasContent =
      replay && messages.some((m) => 
        m.info.role === "user" && !m.parts.some((p) => p.type === "compaction")
      )
    if (!hasContent) {
      replay = undefined
      messages = input.messages
    }
  }

  // ─────────────────────────────────────────────────────────────
  // Get compaction agent and model
  // ─────────────────────────────────────────────────────────────
  const agent = await Agent.get("compaction")
  const model = agent.model
    ? await Provider.getModel(agent.model.providerID, agent.model.modelID)
    : await Provider.getModel(userMessage.model.providerID, userMessage.model.modelID)
    
  // ─────────────────────────────────────────────────────────────
  // Create assistant message for summary
  // ─────────────────────────────────────────────────────────────
  const msg = await Session.updateMessage({
    id: MessageID.ascending(),
    role: "assistant",
    parentID: input.parentID,
    sessionID: input.sessionID,
    mode: "compaction",
    agent: "compaction",
    variant: userMessage.variant,
    summary: true,  // Mark as summary message
    path: {
      cwd: Instance.directory,
      root: Instance.worktree,
    },
    cost: 0,
    tokens: { output: 0, input: 0, reasoning: 0, cache: { read: 0, write: 0 } },
    modelID: model.id,
    providerID: model.providerID,
    time: { created: Date.now() },
  }) as MessageV2.Assistant
  
  // ─────────────────────────────────────────────────────────────
  // Create processor and generate summary
  // ─────────────────────────────────────────────────────────────
  const processor = SessionProcessor.create({
    assistantMessage: msg,
    sessionID: input.sessionID,
    model,
    abort: input.abort,
  })
  
  // Allow plugins to customize compaction
  const compacting = await Plugin.trigger(
    "experimental.session.compacting",
    { sessionID: input.sessionID },
    { context: [], prompt: undefined },
  )
  
  const defaultPrompt = `Provide a detailed prompt for continuing our conversation above.
Focus on information that would be helpful for continuing the conversation, including what we did, what we're doing, which files we're working on, and what we're going to do next.
The summary that you construct will be used so that another agent can read it and continue the work.

When constructing the summary, try to stick to this template:
---
## Goal

[What goal(s) is the user trying to accomplish?]

## Instructions

- [What important instructions did the user give you that are relevant]
- [If there is a plan or spec, include information about it so next agent can continue using it]

## Discoveries

[What notable things were learned during this conversation that would be useful for the next agent to know when continuing the work]

## Accomplished

[What work has been completed, what work is still in progress, and what work is left?]

## Relevant files / directories

[Construct a structured list of relevant files that have been read, edited, or created that pertain to the task at hand. If all the files in a directory are relevant, include the path to the directory.]
---`

  const promptText = compacting.prompt ?? [defaultPrompt, ...compacting.context].join("\n\n")
  
  // Clone messages and transform
  const msgs = structuredClone(messages)
  await Plugin.trigger("experimental.chat.messages.transform", {}, { messages: msgs })
  
  // ─────────────────────────────────────────────────────────────
  // Call the LLM to generate summary
  // ─────────────────────────────────────────────────────────────
  const result = await processor.process({
    user: userMessage,
    agent,
    abort: input.abort,
    sessionID: input.sessionID,
    tools: {},  // No tools during compaction
    system: [],
    messages: [
      ...MessageV2.toModelMessages(msgs, model, { stripMedia: true }),
      {
        role: "user",
        content: [{ type: "text", text: promptText }],
      },
    ],
    model,
  })

  // ─────────────────────────────────────────────────────────────
  // Handle compaction result
  // ─────────────────────────────────────────────────────────────
  if (result === "compact") {
    // Compaction itself overflowed - context too large
    processor.message.error = new MessageV2.ContextOverflowError({
      message: replay
        ? "Conversation history too large to compact - exceeds model context limit"
        : "Session too large to compact - context exceeds model limit even after stripping media",
    }).toObject()
    processor.message.finish = "error"
    await Session.updateMessage(processor.message)
    return "stop"
  }

  // ─────────────────────────────────────────────────────────────
  // Auto-continue after compaction
  // ─────────────────────────────────────────────────────────────
  if (result === "continue" && input.auto) {
    if (replay) {
      // Replay the message that triggered overflow
      const original = replay.info as MessageV2.User
      const replayMsg = await Session.updateMessage({
        id: MessageID.ascending(),
        role: "user",
        sessionID: input.sessionID,
        time: { created: Date.now() },
        agent: original.agent,
        model: original.model,
        format: original.format,
        tools: original.tools,
        system: original.system,
        variant: original.variant,
      })
      for (const part of replay.parts) {
        if (part.type === "compaction") continue
        // Strip media from replayed parts
        const replayPart =
          part.type === "file" && MessageV2.isMedia(part.mime)
            ? { type: "text" as const, text: `[Attached ${part.mime}: ${part.filename ?? "file"}]` }
            : part
        await Session.updatePart({
          ...replayPart,
          id: PartID.ascending(),
          messageID: replayMsg.id,
          sessionID: input.sessionID,
        })
      }
    } else {
      // Add continue message
      const continueMsg = await Session.updateMessage({
        id: MessageID.ascending(),
        role: "user",
        sessionID: input.sessionID,
        time: { created: Date.now() },
        agent: userMessage.agent,
        model: userMessage.model,
      })
      const text =
        (input.overflow
          ? "The previous request exceeded the provider's size limit due to large media attachments. The conversation was compacted and media files were removed from context. If the user was asking about attached images or files, explain that the attachments were too large to process and suggest they try again with smaller or fewer files.\n\n"
          : "") +
        "Continue if you have next steps, or stop and ask for clarification if you are unsure how to proceed."
      await Session.updatePart({
        id: PartID.ascending(),
        messageID: continueMsg.id,
        sessionID: input.sessionID,
        type: "text",
        synthetic: true,
        text,
        time: { start: Date.now(), end: Date.now() },
      })
    }
  }
  
  if (processor.message.error) return "stop"
  Bus.publish(Event.Compacted, { sessionID: input.sessionID })
  return "continue"
}
```

---

## Pruning Old Tool Outputs

In addition to full compaction, the system prunes old tool outputs to save space:

```typescript
// From packages/opencode/src/session/compaction.ts

export const PRUNE_MINIMUM = 20_000    // Only prune if > 20K tokens
export const PRUNE_PROTECT = 40_000    // Protect last 40K tokens

const PRUNE_PROTECTED_TOOLS = ["skill"]

export async function prune(input: { sessionID: SessionID }) {
  const config = await Config.get()
  if (config.compaction?.prune === false) return
  
  log.info("pruning")
  const msgs = await Session.messages({ sessionID: input.sessionID }).catch((err) => {
    if (NotFoundError.isInstance(err)) return undefined
    throw err
  })
  if (!msgs) return
  
  let total = 0
  let pruned = 0
  const toPrune = []
  let turns = 0

  // Scan backwards through messages
  loop: for (let msgIndex = msgs.length - 1; msgIndex >= 0; msgIndex--) {
    const msg = msgs[msgIndex]
    if (msg.info.role === "user") turns++
    if (turns < 2) continue  // Protect current turn
    if (msg.info.role === "assistant" && msg.info.summary) break loop  // Stop at summary
    
    for (let partIndex = msg.parts.length - 1; partIndex >= 0; partIndex--) {
      const part = msg.parts[partIndex]
      if (part.type === "tool" && part.state.status === "completed") {
        // Don't prune protected tools
        if (PRUNE_PROTECTED_TOOLS.includes(part.tool)) continue

        // Stop if already compacted
        if (part.state.time.compacted) break loop
        
        const estimate = Token.estimate(part.state.output)
        total += estimate
        
        // Only prune after protecting recent outputs
        if (total > PRUNE_PROTECT) {
          pruned += estimate
          toPrune.push(part)
        }
      }
    }
  }
  
  log.info("found", { pruned, total })
  
  // Only prune if significant savings
  if (pruned > PRUNE_MINIMUM) {
    for (const part of toPrune) {
      if (part.state.status === "completed") {
        part.state.time.compacted = Date.now()
        await Session.updatePart(part)
      }
    }
    log.info("pruned", { count: toPrune.length })
  }
}
```

### Pruning Rules

1. **Protect current turn**: Skip first 2 user turns
2. **Stop at summary**: Don't prune before compaction boundary
3. **Protect recent outputs**: Keep last 40K tokens of tool outputs
4. **Minimum threshold**: Only prune if > 20K tokens can be saved
5. **Protected tools**: Never prune `skill` tool outputs

When a tool output is pruned, `time.compacted` is set, and `toModelMessages` replaces the output with `"[Old tool result content cleared]"`.

---

## Session Summary

The `SessionSummary` module tracks file changes across the session:

```typescript
// From packages/opencode/src/session/summary.ts

export const summarize = fn(
  z.object({
    sessionID: SessionID.zod,
    messageID: MessageID.zod,
  }),
  async (input) => {
    await Session.messages({ sessionID: input.sessionID })
      .then((all) =>
        Promise.all([
          summarizeSession({ sessionID: input.sessionID, messages: all }),
          summarizeMessage({ messageID: input.messageID, messages: all }),
        ]),
      )
      .catch((err) => {
        if (NotFoundError.isInstance(err)) return
        throw err
      })
  },
)

async function summarizeSession(input: { 
  sessionID: SessionID
  messages: MessageV2.WithParts[] 
}) {
  const diffs = await computeDiff({ messages: input.messages })
  
  await Session.setSummary({
    sessionID: input.sessionID,
    summary: {
      additions: diffs.reduce((sum, x) => sum + x.additions, 0),
      deletions: diffs.reduce((sum, x) => sum + x.deletions, 0),
      files: diffs.length,
    },
  })
  
  await Storage.write(["session_diff", input.sessionID], diffs)
  Bus.publish(Session.Event.Diff, {
    sessionID: input.sessionID,
    diff: diffs,
  })
}

export async function computeDiff(input: { messages: MessageV2.WithParts[] }) {
  let from: string | undefined
  let to: string | undefined

  // Find earliest start snapshot and latest end snapshot
  for (const item of input.messages) {
    if (!from) {
      for (const part of item.parts) {
        if (part.type === "step-start" && part.snapshot) {
          from = part.snapshot
          break
        }
      }
    }

    for (const part of item.parts) {
      if (part.type === "step-finish" && part.snapshot) {
        to = part.snapshot
      }
    }
  }

  if (from && to) return Snapshot.diffFull(from, to)
  return []
}
```

---

## Context Flow Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                    CONTEXT MANAGEMENT FLOW                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────┐                                               │
│  │ User Message │                                               │
│  └──────┬───────┘                                               │
│         │                                                        │
│         ▼                                                        │
│  ┌──────────────┐    ┌───────────────────┐                      │
│  │ LLM.stream() │───▶│ Context Overflow? │                      │
│  └──────────────┘    └─────────┬─────────┘                      │
│                                │                                 │
│                    ┌───────────┴───────────┐                    │
│                    │                       │                     │
│                    ▼                       ▼                     │
│              ┌─────────┐           ┌──────────────┐             │
│              │   No    │           │     Yes      │             │
│              └────┬────┘           └──────┬───────┘             │
│                   │                       │                      │
│                   ▼                       ▼                      │
│            ┌────────────┐         ┌──────────────┐              │
│            │ Continue   │         │ Create       │              │
│            │ Processing │         │ CompactionPart│              │
│            └────────────┘         └──────┬───────┘              │
│                                          │                       │
│                                          ▼                       │
│                                   ┌──────────────┐              │
│                                   │ Process      │              │
│                                   │ Compaction   │              │
│                                   └──────┬───────┘              │
│                                          │                       │
│                                          ▼                       │
│                                   ┌──────────────┐              │
│                                   │ Generate     │              │
│                                   │ Summary      │              │
│                                   └──────┬───────┘              │
│                                          │                       │
│                                          ▼                       │
│                                   ┌──────────────┐              │
│                                   │ Prune Old    │              │
│                                   │ Tool Outputs │              │
│                                   └──────┬───────┘              │
│                                          │                       │
│                                          ▼                       │
│                                   ┌──────────────┐              │
│                                   │ Continue     │              │
│                                   │ Loop         │              │
│                                   └──────────────┘              │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Configuration Options

```typescript
// In opencode.json or config
{
  "compaction": {
    "auto": true,           // Enable automatic compaction
    "prune": true,          // Enable tool output pruning
    "reserved": 20000       // Tokens to reserve for output
  }
}
```

---

## Filtering Compacted Messages

When loading messages, the system filters to only show content after the last compaction:

```typescript
// From packages/opencode/src/session/message-v2.ts

export async function filterCompacted(stream: AsyncIterable<MessageV2.WithParts>) {
  const result = [] as MessageV2.WithParts[]
  const completed = new Set<string>()
  
  for await (const msg of stream) {
    result.push(msg)
    
    // Stop at compaction boundary (user message with compaction part 
    // that has a completed summary response)
    if (
      msg.info.role === "user" &&
      completed.has(msg.info.id) &&
      msg.parts.some((part) => part.type === "compaction")
    )
      break
      
    // Track completed summary messages
    if (msg.info.role === "assistant" && msg.info.summary && msg.info.finish && !msg.info.error)
      completed.add(msg.info.parentID)
  }
  
  result.reverse()
  return result
}
```

---

## Self-Check Questions

1. **What's the difference between compaction and pruning?**
   <details>
   <summary>Answer</summary>
   Compaction summarizes the entire conversation history into a condensed form. Pruning only clears old tool outputs while keeping the conversation structure intact.
   </details>

2. **When does the system protect tool outputs from pruning?**
   <details>
   <summary>Answer</summary>
   The last 40K tokens of tool outputs are protected, outputs from the current turn are protected, and outputs from protected tools (like "skill") are never pruned.
   </details>

3. **What happens if compaction itself causes a context overflow?**
   <details>
   <summary>Answer</summary>
   The system sets a `ContextOverflowError` on the message and returns "stop", indicating the conversation is too large to continue.
   </details>

4. **How does `filterCompacted` know where to stop?**
   <details>
   <summary>Answer</summary>
   It tracks assistant messages with `summary: true` and `finish` set, then stops when it finds a user message with a `compaction` part whose ID is in the completed set.
   </details>

---

## Exercises

1. **Trigger compaction**: Create a long conversation that exceeds the context limit and observe compaction.

2. **Examine pruned outputs**: Look at tool parts with `time.compacted` set and verify they're replaced in model messages.

3. **Customize compaction prompt**: Use the plugin hook to add custom context to the compaction prompt.

---

## Further Reading

- `packages/opencode/src/snapshot/index.ts` - Git snapshot system
- `packages/opencode/src/util/token.ts` - Token estimation
- `packages/opencode/src/plugin/index.ts` - Plugin hooks
- `packages/opencode/src/agent/agent.ts` - Compaction agent configuration
