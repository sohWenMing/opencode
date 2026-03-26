# Module 06: Session System - Lesson 04: Processor & Streaming

## Learning Objectives

By the end of this lesson, you will be able to:

1. Understand the SessionProcessor's role in handling LLM streams
2. Trace how stream events are mapped to message parts
3. Explain tool call handling during streaming
4. Identify doom loop detection and prevention
5. Understand processor return values and their meanings

---

## The SessionProcessor

The `SessionProcessor` in `packages/opencode/src/session/processor.ts` is responsible for:

1. Calling `LLM.stream()` to get the AI response
2. Iterating over stream events
3. Creating and updating message parts
4. Handling tool calls and results
5. Detecting doom loops
6. Managing retries

```
┌─────────────────────────────────────────────────────────────────┐
│                    SessionProcessor                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  LLM.stream() ──▶ fullStream ──▶ Event Handler ──▶ Message Parts│
│                                                                  │
│  Events:                                                         │
│  ├── start              → Set status to busy                    │
│  ├── reasoning-start    → Create ReasoningPart                  │
│  ├── reasoning-delta    → Append to reasoning text              │
│  ├── reasoning-end      → Finalize ReasoningPart                │
│  ├── text-start         → Create TextPart                       │
│  ├── text-delta         → Append to text                        │
│  ├── text-end           → Finalize TextPart                     │
│  ├── tool-input-start   → Create ToolPart (pending)             │
│  ├── tool-call          → Update ToolPart (running)             │
│  ├── tool-result        → Update ToolPart (completed)           │
│  ├── tool-error         → Update ToolPart (error)               │
│  ├── start-step         → Create StepStartPart + snapshot       │
│  ├── finish-step        → Create StepFinishPart + usage         │
│  ├── error              → Throw error                           │
│  └── finish             → (no-op)                               │
│                                                                  │
│  Returns: "continue" | "stop" | "compact"                       │
└─────────────────────────────────────────────────────────────────┘
```

---

## Creating a Processor

```typescript
// From packages/opencode/src/session/processor.ts

export function create(input: {
  assistantMessage: MessageV2.Assistant
  sessionID: SessionID
  model: Provider.Model
  abort: AbortSignal
}) {
  const toolcalls: Record<string, MessageV2.ToolPart> = {}
  let snapshot: string | undefined
  let blocked = false
  let attempt = 0
  let needsCompaction = false

  const result = {
    get message() {
      return input.assistantMessage
    },
    
    partFromToolCall(toolCallID: string) {
      return toolcalls[toolCallID]
    },
    
    async process(streamInput: LLM.StreamInput) {
      // ... main processing logic
    },
  }
  return result
}
```

### State Variables

| Variable | Purpose |
|----------|---------|
| `toolcalls` | Maps tool call IDs to their parts |
| `snapshot` | Git snapshot taken at step start |
| `blocked` | Set when permission denied |
| `attempt` | Current retry attempt number |
| `needsCompaction` | Set when context overflow detected |

---

## The Process Method

```typescript
async process(streamInput: LLM.StreamInput) {
  log.info("process")
  needsCompaction = false
  const shouldBreak = (await Config.get()).experimental?.continue_loop_on_deny !== true
  
  while (true) {
    try {
      let currentText: MessageV2.TextPart | undefined
      let reasoningMap: Record<string, MessageV2.ReasoningPart> = {}
      
      // ═══════════════════════════════════════════════════════════
      // Call LLM and iterate over stream
      // ═══════════════════════════════════════════════════════════
      const stream = await LLM.stream(streamInput)

      for await (const value of stream.fullStream) {
        input.abort.throwIfAborted()
        
        switch (value.type) {
          // ... event handlers (see below)
        }
        
        if (needsCompaction) break
      }
    } catch (e: any) {
      // ... error handling (see below)
    }
    
    // ... cleanup and return
  }
}
```

---

## Stream Event Handlers

### Session Status

```typescript
case "start":
  await SessionStatus.set(input.sessionID, { type: "busy" })
  break
```

### Reasoning (Chain of Thought)

```typescript
case "reasoning-start":
  if (value.id in reasoningMap) continue  // Dedupe
  
  const reasoningPart = {
    id: PartID.ascending(),
    messageID: input.assistantMessage.id,
    sessionID: input.assistantMessage.sessionID,
    type: "reasoning" as const,
    text: "",
    time: { start: Date.now() },
    metadata: value.providerMetadata,
  }
  reasoningMap[value.id] = reasoningPart
  await Session.updatePart(reasoningPart)
  break

case "reasoning-delta":
  if (value.id in reasoningMap) {
    const part = reasoningMap[value.id]
    part.text += value.text
    if (value.providerMetadata) part.metadata = value.providerMetadata
    
    // Send delta for real-time UI updates
    await Session.updatePartDelta({
      sessionID: part.sessionID,
      messageID: part.messageID,
      partID: part.id,
      field: "text",
      delta: value.text,
    })
  }
  break

case "reasoning-end":
  if (value.id in reasoningMap) {
    const part = reasoningMap[value.id]
    part.text = part.text.trimEnd()
    part.time = { ...part.time, end: Date.now() }
    if (value.providerMetadata) part.metadata = value.providerMetadata
    await Session.updatePart(part)
    delete reasoningMap[value.id]
  }
  break
```

### Text Content

```typescript
case "text-start":
  currentText = {
    id: PartID.ascending(),
    messageID: input.assistantMessage.id,
    sessionID: input.assistantMessage.sessionID,
    type: "text",
    text: "",
    time: { start: Date.now() },
    metadata: value.providerMetadata,
  }
  await Session.updatePart(currentText)
  break

case "text-delta":
  if (currentText) {
    currentText.text += value.text
    if (value.providerMetadata) currentText.metadata = value.providerMetadata
    
    await Session.updatePartDelta({
      sessionID: currentText.sessionID,
      messageID: currentText.messageID,
      partID: currentText.id,
      field: "text",
      delta: value.text,
    })
  }
  break

case "text-end":
  if (currentText) {
    currentText.text = currentText.text.trimEnd()
    
    // Allow plugins to transform text
    const textOutput = await Plugin.trigger(
      "experimental.text.complete",
      {
        sessionID: input.sessionID,
        messageID: input.assistantMessage.id,
        partID: currentText.id,
      },
      { text: currentText.text },
    )
    currentText.text = textOutput.text
    currentText.time = { start: Date.now(), end: Date.now() }
    if (value.providerMetadata) currentText.metadata = value.providerMetadata
    await Session.updatePart(currentText)
  }
  currentText = undefined
  break
```

### Tool Calls

```typescript
case "tool-input-start":
  // Create tool part in pending state
  const part = await Session.updatePart({
    id: toolcalls[value.id]?.id ?? PartID.ascending(),
    messageID: input.assistantMessage.id,
    sessionID: input.assistantMessage.sessionID,
    type: "tool",
    tool: value.toolName,
    callID: value.id,
    state: {
      status: "pending",
      input: {},
      raw: "",
    },
  })
  toolcalls[value.id] = part as MessageV2.ToolPart
  break

case "tool-call": {
  const match = toolcalls[value.toolCallId]
  if (match) {
    // Update to running state
    const part = await Session.updatePart({
      ...match,
      tool: value.toolName,
      state: {
        status: "running",
        input: value.input,
        time: { start: Date.now() },
      },
      metadata: value.providerMetadata,
    })
    toolcalls[value.toolCallId] = part as MessageV2.ToolPart

    // ─────────────────────────────────────────────────────────
    // DOOM LOOP DETECTION
    // ─────────────────────────────────────────────────────────
    const parts = await MessageV2.parts(input.assistantMessage.id)
    const lastThree = parts.slice(-DOOM_LOOP_THRESHOLD)

    if (
      lastThree.length === DOOM_LOOP_THRESHOLD &&
      lastThree.every(
        (p) =>
          p.type === "tool" &&
          p.tool === value.toolName &&
          p.state.status !== "pending" &&
          JSON.stringify(p.state.input) === JSON.stringify(value.input),
      )
    ) {
      const agent = await Agent.get(input.assistantMessage.agent)
      await Permission.ask({
        permission: "doom_loop",
        patterns: [value.toolName],
        sessionID: input.assistantMessage.sessionID,
        metadata: {
          tool: value.toolName,
          input: value.input,
        },
        always: [value.toolName],
        ruleset: agent.permission,
      })
    }
  }
  break
}

case "tool-result": {
  const match = toolcalls[value.toolCallId]
  if (match && match.state.status === "running") {
    await Session.updatePart({
      ...match,
      state: {
        status: "completed",
        input: value.input ?? match.state.input,
        output: value.output.output,
        metadata: value.output.metadata,
        title: value.output.title,
        time: {
          start: match.state.time.start,
          end: Date.now(),
        },
        attachments: value.output.attachments,
      },
    })
    delete toolcalls[value.toolCallId]
  }
  break
}

case "tool-error": {
  const match = toolcalls[value.toolCallId]
  if (match && match.state.status === "running") {
    await Session.updatePart({
      ...match,
      state: {
        status: "error",
        input: value.input ?? match.state.input,
        error: value.error instanceof Error ? value.error.message : String(value.error),
        time: {
          start: match.state.time.start,
          end: Date.now(),
        },
      },
    })

    // Check if permission was denied
    if (
      value.error instanceof Permission.RejectedError ||
      value.error instanceof Question.RejectedError
    ) {
      blocked = shouldBreak
    }
    delete toolcalls[value.toolCallId]
  }
  break
}
```

### Step Tracking

```typescript
case "start-step":
  // Take git snapshot before step
  snapshot = await Snapshot.track()
  await Session.updatePart({
    id: PartID.ascending(),
    messageID: input.assistantMessage.id,
    sessionID: input.sessionID,
    snapshot,
    type: "step-start",
  })
  break

case "finish-step":
  // Calculate usage and cost
  const usage = Session.getUsage({
    model: input.model,
    usage: value.usage,
    metadata: value.providerMetadata,
  })
  
  input.assistantMessage.finish = value.finishReason
  input.assistantMessage.cost += usage.cost
  input.assistantMessage.tokens = usage.tokens
  
  await Session.updatePart({
    id: PartID.ascending(),
    reason: value.finishReason,
    snapshot: await Snapshot.track(),
    messageID: input.assistantMessage.id,
    sessionID: input.assistantMessage.sessionID,
    type: "step-finish",
    tokens: usage.tokens,
    cost: usage.cost,
  })
  await Session.updateMessage(input.assistantMessage)
  
  // Create patch if files changed
  if (snapshot) {
    const patch = await Snapshot.patch(snapshot)
    if (patch.files.length) {
      await Session.updatePart({
        id: PartID.ascending(),
        messageID: input.assistantMessage.id,
        sessionID: input.sessionID,
        type: "patch",
        hash: patch.hash,
        files: patch.files,
      })
    }
    snapshot = undefined
  }
  
  // Trigger summary update
  SessionSummary.summarize({
    sessionID: input.sessionID,
    messageID: input.assistantMessage.parentID,
  })
  
  // Check for context overflow
  if (
    !input.assistantMessage.summary &&
    (await SessionCompaction.isOverflow({ tokens: usage.tokens, model: input.model }))
  ) {
    needsCompaction = true
  }
  break
```

### Error Handling

```typescript
case "error":
  throw value.error
```

---

## Error Handling and Retries

```typescript
} catch (e: any) {
  log.error("process", {
    error: e,
    stack: JSON.stringify(e.stack),
  })
  
  const error = MessageV2.fromError(e, { 
    providerID: input.model.providerID, 
    aborted: input.abort.aborted 
  })
  
  // Context overflow → trigger compaction
  if (MessageV2.ContextOverflowError.isInstance(error)) {
    needsCompaction = true
    Bus.publish(Session.Event.Error, {
      sessionID: input.sessionID,
      error,
    })
  } else {
    // Check if retryable
    const retry = SessionRetry.retryable(error)
    if (retry !== undefined) {
      attempt++
      const delay = SessionRetry.delay(attempt, error.name === "APIError" ? error : undefined)
      
      await SessionStatus.set(input.sessionID, {
        type: "retry",
        attempt,
        message: retry,
        next: Date.now() + delay,
      })
      
      await SessionRetry.sleep(delay, input.abort).catch(() => {})
      continue  // Retry the loop
    }
    
    // Not retryable → set error and stop
    input.assistantMessage.error = error
    Bus.publish(Session.Event.Error, {
      sessionID: input.assistantMessage.sessionID,
      error: input.assistantMessage.error,
    })
    await SessionStatus.set(input.sessionID, { type: "idle" })
  }
}
```

---

## Doom Loop Detection

The doom loop detector prevents the model from calling the same tool with the same arguments repeatedly:

```typescript
const DOOM_LOOP_THRESHOLD = 3

// In tool-call handler:
const parts = await MessageV2.parts(input.assistantMessage.id)
const lastThree = parts.slice(-DOOM_LOOP_THRESHOLD)

if (
  lastThree.length === DOOM_LOOP_THRESHOLD &&
  lastThree.every(
    (p) =>
      p.type === "tool" &&
      p.tool === value.toolName &&
      p.state.status !== "pending" &&
      JSON.stringify(p.state.input) === JSON.stringify(value.input),
  )
) {
  // Ask for permission to continue
  await Permission.ask({
    permission: "doom_loop",
    patterns: [value.toolName],
    sessionID: input.assistantMessage.sessionID,
    metadata: {
      tool: value.toolName,
      input: value.input,
    },
    always: [value.toolName],
    ruleset: agent.permission,
  })
}
```

This triggers a permission prompt asking the user if they want to continue.

---

## Cleanup and Return Values

```typescript
// After the try-catch block:

// Create patch for any remaining changes
if (snapshot) {
  const patch = await Snapshot.patch(snapshot)
  if (patch.files.length) {
    await Session.updatePart({
      id: PartID.ascending(),
      messageID: input.assistantMessage.id,
      sessionID: input.sessionID,
      type: "patch",
      hash: patch.hash,
      files: patch.files,
    })
  }
  snapshot = undefined
}

// Mark any incomplete tools as aborted
const p = await MessageV2.parts(input.assistantMessage.id)
for (const part of p) {
  if (part.type === "tool" && part.state.status !== "completed" && part.state.status !== "error") {
    await Session.updatePart({
      ...part,
      state: {
        ...part.state,
        status: "error",
        error: "Tool execution aborted",
        time: { start: Date.now(), end: Date.now() },
      },
    })
  }
}

// Update message completion time
input.assistantMessage.time.completed = Date.now()
await Session.updateMessage(input.assistantMessage)

// Return appropriate result
if (needsCompaction) return "compact"
if (blocked) return "stop"
if (input.assistantMessage.error) return "stop"
return "continue"
```

### Return Values

| Value | Meaning |
|-------|---------|
| `"continue"` | Loop should continue (tool calls made, model wants to continue) |
| `"stop"` | Loop should stop (error, permission denied, model finished) |
| `"compact"` | Context overflow detected, trigger compaction |

---

## The Retry System

```typescript
// From packages/opencode/src/session/retry.ts

export namespace SessionRetry {
  export const RETRY_INITIAL_DELAY = 2000
  export const RETRY_BACKOFF_FACTOR = 2
  export const RETRY_MAX_DELAY_NO_HEADERS = 30_000

  export function delay(attempt: number, error?: MessageV2.APIError) {
    if (error) {
      const headers = error.data.responseHeaders
      if (headers) {
        // Check for retry-after-ms header
        const retryAfterMs = headers["retry-after-ms"]
        if (retryAfterMs) {
          const parsedMs = Number.parseFloat(retryAfterMs)
          if (!Number.isNaN(parsedMs)) return parsedMs
        }

        // Check for retry-after header (seconds)
        const retryAfter = headers["retry-after"]
        if (retryAfter) {
          const parsedSeconds = Number.parseFloat(retryAfter)
          if (!Number.isNaN(parsedSeconds)) return Math.ceil(parsedSeconds * 1000)
        }

        // Exponential backoff
        return RETRY_INITIAL_DELAY * Math.pow(RETRY_BACKOFF_FACTOR, attempt - 1)
      }
    }

    return Math.min(
      RETRY_INITIAL_DELAY * Math.pow(RETRY_BACKOFF_FACTOR, attempt - 1), 
      RETRY_MAX_DELAY_NO_HEADERS
    )
  }

  export function retryable(error: ReturnType<NamedError["toObject"]>) {
    // Context overflow should not be retried
    if (MessageV2.ContextOverflowError.isInstance(error)) return undefined
    
    if (MessageV2.APIError.isInstance(error)) {
      if (!error.data.isRetryable) return undefined
      if (error.data.responseBody?.includes("FreeUsageLimitError"))
        return `Free usage exceeded, add credits https://opencode.ai/zen`
      return error.data.message.includes("Overloaded") 
        ? "Provider is overloaded" 
        : error.data.message
    }

    // Parse JSON error messages
    const json = /* parse error.data.message as JSON */
    if (json?.type === "error" && json.error?.type === "too_many_requests") {
      return "Too Many Requests"
    }
    // ... more patterns
    
    return undefined  // Not retryable
  }
}
```

---

## Session Status Updates

```typescript
// From packages/opencode/src/session/status.ts

export namespace SessionStatus {
  export const Info = z.union([
    z.object({ type: z.literal("idle") }),
    z.object({
      type: z.literal("retry"),
      attempt: z.number(),
      message: z.string(),
      next: z.number(),  // When retry will happen
    }),
    z.object({ type: z.literal("busy") }),
  ])

  export const Event = {
    Status: BusEvent.define(
      "session.status",
      z.object({
        sessionID: SessionID.zod,
        status: Info,
      }),
    ),
  }

  export async function set(sessionID: SessionID, status: Info) {
    return runPromise((svc) => svc.set(sessionID, status))
  }
}
```

---

## Self-Check Questions

1. **What's the difference between `tool-input-start` and `tool-call` events?**
   <details>
   <summary>Answer</summary>
   `tool-input-start` fires when the model starts streaming tool arguments (pending state). `tool-call` fires when the arguments are complete and execution begins (running state).
   </details>

2. **How does doom loop detection work?**
   <details>
   <summary>Answer</summary>
   It checks if the last 3 tool calls have the same tool name and identical input. If so, it prompts the user for permission to continue.
   </details>

3. **What triggers a "compact" return value?**
   <details>
   <summary>Answer</summary>
   Either a `ContextOverflowError` from the API, or `SessionCompaction.isOverflow()` returning true after a step finishes.
   </details>

4. **How are retry delays calculated?**
   <details>
   <summary>Answer</summary>
   First checks for `retry-after-ms` or `retry-after` headers. If not present, uses exponential backoff: `2000 * 2^(attempt-1)`, capped at 30 seconds.
   </details>

---

## Exercises

1. **Trace a tool call**: Add logging to follow a tool from `tool-input-start` through `tool-result`.

2. **Test doom loop**: Create a scenario that triggers doom loop detection and observe the permission prompt.

3. **Examine retry behavior**: Simulate a rate limit error and trace the retry logic.

---

## Further Reading

- `packages/opencode/src/session/llm.ts` - LLM streaming implementation
- `packages/opencode/src/snapshot/index.ts` - Git snapshot system
- `packages/opencode/src/permission/index.ts` - Permission system
- AI SDK documentation on streaming
