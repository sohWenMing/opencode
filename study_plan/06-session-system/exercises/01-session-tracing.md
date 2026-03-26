# Module 06: Session System - Exercises: Session Tracing

## Overview

These exercises will help you understand the session system by tracing actual code paths and examining real data structures.

---

## Exercise 1: Trace a Message Through the Full Loop

### Objective
Follow a user message from input to final assistant response.

### Setup
1. Open the opencode codebase in your editor
2. Set breakpoints in key locations (or add console.log statements)

### Steps

#### Step 1: Entry Point
Add a breakpoint/log in `packages/opencode/src/session/prompt.ts`:

```typescript
// In SessionPrompt.prompt()
export const prompt = fn(PromptInput, async (input) => {
  console.log("=== PROMPT ENTRY ===", {
    sessionID: input.sessionID,
    agent: input.agent,
    partsCount: input.parts.length,
  })
  // ...
})
```

#### Step 2: User Message Creation
Add a breakpoint/log in `createUserMessage()`:

```typescript
async function createUserMessage(input: PromptInput) {
  console.log("=== CREATE USER MESSAGE ===", {
    agent: input.agent,
    model: input.model,
  })
  // ...
  
  // After message is created:
  console.log("=== USER MESSAGE CREATED ===", {
    messageID: info.id,
    partsCount: parts.length,
  })
}
```

#### Step 3: Loop Entry
Add a breakpoint/log in `loop()`:

```typescript
while (true) {
  console.log("=== LOOP ITERATION ===", {
    step,
    sessionID,
    aborted: abort.aborted,
  })
  // ...
}
```

#### Step 4: Processor Creation
Add a breakpoint/log before processor creation:

```typescript
const processor = SessionProcessor.create({
  // ...
})
console.log("=== PROCESSOR CREATED ===", {
  messageID: processor.message.id,
  model: model.id,
})
```

#### Step 5: Stream Processing
Add a breakpoint/log in `packages/opencode/src/session/processor.ts`:

```typescript
for await (const value of stream.fullStream) {
  console.log("=== STREAM EVENT ===", {
    type: value.type,
  })
  // ...
}
```

### Questions to Answer

1. How many loop iterations occur for a simple "Hello" message?
2. What stream events are emitted for a text-only response?
3. What stream events are emitted when a tool is called?
4. How does the `finish` reason change the loop behavior?

### Expected Output Pattern

```
=== PROMPT ENTRY === { sessionID: "session_...", agent: "build", partsCount: 1 }
=== CREATE USER MESSAGE === { agent: "build", model: { providerID: "anthropic", modelID: "claude-sonnet-4-20250514" } }
=== USER MESSAGE CREATED === { messageID: "message_...", partsCount: 1 }
=== LOOP ITERATION === { step: 0, sessionID: "session_...", aborted: false }
=== PROCESSOR CREATED === { messageID: "message_...", model: "claude-sonnet-4-20250514" }
=== STREAM EVENT === { type: "start" }
=== STREAM EVENT === { type: "start-step" }
=== STREAM EVENT === { type: "text-start" }
=== STREAM EVENT === { type: "text-delta" }
... (many text-delta events)
=== STREAM EVENT === { type: "text-end" }
=== STREAM EVENT === { type: "finish-step" }
=== STREAM EVENT === { type: "finish" }
=== LOOP ITERATION === { step: 1, sessionID: "session_...", aborted: false }
(exits because finish reason is "stop")
```

---

## Exercise 2: Understand the Processor State Machine

### Objective
Map out all possible states and transitions in the processor.

### Task
Create a state diagram showing:

1. All possible stream event types
2. What state changes each event causes
3. What parts are created/updated for each event

### Template

```
┌─────────────────────────────────────────────────────────────────┐
│                    PROCESSOR STATE MACHINE                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Event: start                                                    │
│  ├── Action: SessionStatus.set("busy")                          │
│  └── Parts: (none)                                              │
│                                                                  │
│  Event: reasoning-start                                          │
│  ├── Action: Create ReasoningPart                               │
│  └── Parts: ReasoningPart (text: "")                            │
│                                                                  │
│  Event: reasoning-delta                                          │
│  ├── Action: Append to reasoning text                           │
│  └── Parts: ReasoningPart (text updated)                        │
│                                                                  │
│  ... (continue for all events)                                  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Questions to Answer

1. What events can occur in parallel?
2. What events must occur in sequence?
3. What happens if an event is received out of order?
4. How are multiple tool calls handled simultaneously?

---

## Exercise 3: Examine Compaction Triggers

### Objective
Understand when and why compaction is triggered.

### Setup
Create a test session that will trigger compaction:

```typescript
// Test script to trigger compaction
import { Session } from "./packages/opencode/src/session"
import { SessionPrompt } from "./packages/opencode/src/session/prompt"

async function testCompaction() {
  // Create a session
  const session = await Session.create()
  
  // Send many messages to fill context
  for (let i = 0; i < 50; i++) {
    await SessionPrompt.prompt({
      sessionID: session.id,
      parts: [{
        type: "text",
        text: `Message ${i}: ${generateLongText()}`,
      }],
    })
  }
}

function generateLongText() {
  return "Lorem ipsum ".repeat(500)
}
```

### Add Logging

Add logs to track compaction:

```typescript
// In packages/opencode/src/session/compaction.ts

export async function isOverflow(input: {...}) {
  // ...
  const result = count >= usable
  console.log("=== OVERFLOW CHECK ===", {
    count,
    usable,
    context: input.model.limit.context,
    overflow: result,
  })
  return result
}

export async function process(input: {...}) {
  console.log("=== COMPACTION START ===", {
    sessionID: input.sessionID,
    auto: input.auto,
    overflow: input.overflow,
    messageCount: input.messages.length,
  })
  // ...
}
```

### Questions to Answer

1. At what token count does compaction trigger?
2. How much does compaction reduce the context size?
3. What information is preserved in the summary?
4. What information is lost?

---

## Exercise 4: Debug a Session

### Objective
Practice debugging common session issues.

### Scenario 1: Session Stuck in "Busy" State

**Symptoms:**
- `Session.BusyError` thrown when trying to send a message
- Session status shows "busy" but no activity

**Debugging Steps:**

1. Check the session state:
```typescript
import { SessionStatus } from "./packages/opencode/src/session/status"

const status = await SessionStatus.get(sessionID)
console.log("Session status:", status)
```

2. Check for active abort controller:
```typescript
// In prompt.ts, add:
console.log("Active sessions:", Object.keys(state()))
```

3. Force cancel the session:
```typescript
await SessionPrompt.cancel(sessionID)
```

### Scenario 2: Tool Call Not Completing

**Symptoms:**
- Tool shows "running" state indefinitely
- No tool-result event received

**Debugging Steps:**

1. Check the tool part state:
```typescript
const parts = await MessageV2.parts(messageID)
const toolParts = parts.filter(p => p.type === "tool")
console.log("Tool parts:", toolParts.map(p => ({
  tool: p.tool,
  status: p.state.status,
  callID: p.callID,
})))
```

2. Check for errors in tool execution:
```typescript
// Add to tool execute wrapper in prompt.ts
try {
  const result = await item.execute(args, ctx)
  console.log("Tool result:", result)
  return result
} catch (error) {
  console.error("Tool error:", error)
  throw error
}
```

### Scenario 3: Context Overflow Loop

**Symptoms:**
- Repeated compaction attempts
- Session never completes

**Debugging Steps:**

1. Check token counts:
```typescript
// After each step, log tokens
console.log("Token usage:", {
  input: processor.message.tokens.input,
  output: processor.message.tokens.output,
  total: processor.message.tokens.total,
})
```

2. Check compaction results:
```typescript
// In compaction.process()
console.log("Compaction result:", result)
console.log("Message count before:", input.messages.length)
console.log("Summary length:", processor.message.parts.find(p => p.type === "text")?.text.length)
```

---

## Exercise 5: Implement a Custom Trace

### Objective
Build a comprehensive tracing system for debugging.

### Implementation

Create a trace module:

```typescript
// packages/opencode/src/session/trace.ts

export namespace SessionTrace {
  const traces: Map<string, TraceEntry[]> = new Map()
  
  interface TraceEntry {
    timestamp: number
    event: string
    data: any
  }
  
  export function start(sessionID: string) {
    traces.set(sessionID, [])
  }
  
  export function log(sessionID: string, event: string, data: any) {
    const entries = traces.get(sessionID)
    if (entries) {
      entries.push({
        timestamp: Date.now(),
        event,
        data,
      })
    }
  }
  
  export function dump(sessionID: string) {
    const entries = traces.get(sessionID) ?? []
    console.log(`\n=== SESSION TRACE: ${sessionID} ===\n`)
    for (const entry of entries) {
      const time = new Date(entry.timestamp).toISOString()
      console.log(`[${time}] ${entry.event}`)
      console.log(JSON.stringify(entry.data, null, 2))
      console.log()
    }
  }
  
  export function clear(sessionID: string) {
    traces.delete(sessionID)
  }
}
```

### Add Trace Points

```typescript
// In prompt.ts
SessionTrace.start(sessionID)
SessionTrace.log(sessionID, "loop.start", { step })
SessionTrace.log(sessionID, "processor.created", { messageID: processor.message.id })

// In processor.ts
SessionTrace.log(input.sessionID, "stream.event", { type: value.type })

// At the end
SessionTrace.dump(sessionID)
```

---

## Submission Checklist

- [ ] Completed Exercise 1 with trace output
- [ ] Created state diagram for Exercise 2
- [ ] Documented compaction thresholds from Exercise 3
- [ ] Successfully debugged at least one scenario from Exercise 4
- [ ] Implemented custom trace module from Exercise 5

---

## Further Exploration

1. **Performance Analysis**: Measure time spent in each phase of the loop
2. **Memory Profiling**: Track memory usage during long sessions
3. **Error Injection**: Test error handling by injecting failures at various points
4. **Concurrent Sessions**: Test behavior with multiple simultaneous sessions
