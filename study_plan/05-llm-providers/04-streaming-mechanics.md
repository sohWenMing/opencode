# Lesson 04: Streaming Mechanics

## Learning Objectives

By the end of this lesson, you will be able to:
- Understand how streaming works in the Vercel AI SDK
- Trace the `LLM.stream` function in opencode
- Identify and handle different stream event types
- Understand how the processor consumes the stream
- Handle errors during streaming

---

## Why Streaming?

Streaming allows the UI to show responses as they're generated, rather than waiting for the complete response:

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Non-Streaming vs Streaming                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Non-Streaming:                                                     │
│  ┌─────────┐    ┌─────────────────────────────┐    ┌─────────┐     │
│  │ Request │───▶│ Wait for complete response  │───▶│ Display │     │
│  └─────────┘    │ (could be 30+ seconds)      │    │ all     │     │
│                 └─────────────────────────────┘    └─────────┘     │
│                                                                     │
│  Streaming:                                                         │
│  ┌─────────┐    ┌───┐ ┌───┐ ┌───┐ ┌───┐ ┌───┐    ┌─────────┐     │
│  │ Request │───▶│ H │▶│ e │▶│ l │▶│ l │▶│ o │───▶│ Display │     │
│  └─────────┘    └───┘ └───┘ └───┘ └───┘ └───┘    │ as it   │     │
│                 (tokens arrive ~50-100ms apart)   │ arrives │     │
│                                                   └─────────┘     │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

Benefits:
- **Better UX**: Users see progress immediately
- **Faster perceived latency**: First token appears quickly
- **Cancellation**: Users can stop generation early
- **Tool calls**: Can execute tools as they're called

---

## The LLM.stream Function

The core streaming function is in `packages/opencode/src/session/llm.ts`:

```typescript
export namespace LLM {
  const log = Log.create({ service: "llm" })
  export const OUTPUT_TOKEN_MAX = ProviderTransform.OUTPUT_TOKEN_MAX

  export type StreamInput = {
    user: MessageV2.User
    sessionID: string
    model: Provider.Model
    agent: Agent.Info
    permission?: Permission.Ruleset
    system: string[]
    abort: AbortSignal
    messages: ModelMessage[]
    small?: boolean
    tools: Record<string, Tool>
    retries?: number
    toolChoice?: "auto" | "required" | "none"
  }

  export type StreamOutput = StreamTextResult<ToolSet, unknown>

  export async function stream(input: StreamInput) {
    // ... implementation
  }
}
```

### Input Parameters

```
┌─────────────────────────────────────────────────────────────────────┐
│                    StreamInput Parameters                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Required:                                                          │
│  ├─ user          The user message that triggered this stream       │
│  ├─ sessionID     Session identifier for caching/tracking           │
│  ├─ model         Provider.Model with capabilities and config       │
│  ├─ agent         Agent configuration (prompt, options)             │
│  ├─ system        Array of system prompt parts                      │
│  ├─ abort         AbortSignal for cancellation                      │
│  ├─ messages      Full conversation history                         │
│  └─ tools         Available tools for the model to call             │
│                                                                     │
│  Optional:                                                          │
│  ├─ permission    Permission rules for tool access                  │
│  ├─ small         Use "small model" options (reduced reasoning)     │
│  ├─ retries       Number of retries on failure (default: 0)         │
│  └─ toolChoice    "auto" | "required" | "none"                      │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Stream Initialization

The function sets up logging, loads configuration, and prepares the request:

```typescript
export async function stream(input: StreamInput) {
  const l = log
    .clone()
    .tag("providerID", input.model.providerID)
    .tag("modelID", input.model.id)
    .tag("sessionID", input.sessionID)
    .tag("small", (input.small ?? false).toString())
    .tag("agent", input.agent.name)
    .tag("mode", input.agent.mode)
  l.info("stream", {
    modelID: input.model.id,
    providerID: input.model.providerID,
  })
  
  const [language, cfg, provider, auth] = await Promise.all([
    Provider.getLanguage(input.model),
    Config.get(),
    Provider.getProvider(input.model.providerID),
    Auth.get(input.model.providerID),
  ])
  
  // ... rest of initialization
}
```

### System Prompt Assembly

```typescript
const system: string[] = []
system.push(
  [
    // use agent prompt otherwise provider prompt
    ...(input.agent.prompt ? [input.agent.prompt] : SystemPrompt.provider(input.model)),
    // any custom prompt passed into this call
    ...input.system,
    // any custom prompt from last user message
    ...(input.user.system ? [input.user.system] : []),
  ]
    .filter((x) => x)
    .join("\n"),
)

const header = system[0]
await Plugin.trigger(
  "experimental.chat.system.transform",
  { sessionID: input.sessionID, model: input.model },
  { system },
)
// rejoin to maintain 2-part structure for caching if header unchanged
if (system.length > 2 && system[0] === header) {
  const rest = system.slice(1)
  system.length = 0
  system.push(header, rest.join("\n"))
}
```

---

## The streamText Call

The actual streaming is done via the AI SDK's `streamText`:

```typescript
return streamText({
  onError(error) {
    l.error("stream error", { error })
  },
  async experimental_repairToolCall(failed) {
    const lower = failed.toolCall.toolName.toLowerCase()
    if (lower !== failed.toolCall.toolName && tools[lower]) {
      l.info("repairing tool call", {
        tool: failed.toolCall.toolName,
        repaired: lower,
      })
      return {
        ...failed.toolCall,
        toolName: lower,
      }
    }
    return {
      ...failed.toolCall,
      input: JSON.stringify({
        tool: failed.toolCall.toolName,
        error: failed.error.message,
      }),
      toolName: "invalid",
    }
  },
  temperature: params.temperature,
  topP: params.topP,
  topK: params.topK,
  providerOptions: ProviderTransform.providerOptions(input.model, params.options),
  activeTools: Object.keys(tools).filter((x) => x !== "invalid"),
  tools,
  toolChoice: input.toolChoice,
  maxOutputTokens,
  abortSignal: input.abort,
  headers: {
    ...(input.model.providerID.startsWith("opencode")
      ? {
          "x-opencode-project": Instance.project.id,
          "x-opencode-session": input.sessionID,
          "x-opencode-request": input.user.id,
          "x-opencode-client": Flag.OPENCODE_CLIENT,
        }
      : {
          "User-Agent": `opencode/${Installation.VERSION}`,
        }),
    ...input.model.headers,
    ...headers,
  },
  maxRetries: input.retries ?? 0,
  messages,
  model: wrapLanguageModel({
    model: language,
    middleware: [
      {
        async transformParams(args) {
          if (args.type === "stream") {
            // @ts-expect-error
            args.params.prompt = ProviderTransform.message(args.params.prompt, input.model, options)
          }
          return args.params
        },
      },
    ],
  }),
  experimental_telemetry: {
    isEnabled: cfg.experimental?.openTelemetry,
    metadata: {
      userId: cfg.username ?? "unknown",
      sessionId: input.sessionID,
    },
  },
})
```

---

## Tool Call Repair

The `experimental_repairToolCall` callback handles malformed tool calls:

```typescript
async experimental_repairToolCall(failed) {
  // Try case-insensitive match first
  const lower = failed.toolCall.toolName.toLowerCase()
  if (lower !== failed.toolCall.toolName && tools[lower]) {
    l.info("repairing tool call", {
      tool: failed.toolCall.toolName,
      repaired: lower,
    })
    return {
      ...failed.toolCall,
      toolName: lower,
    }
  }
  
  // Otherwise, redirect to "invalid" tool
  return {
    ...failed.toolCall,
    input: JSON.stringify({
      tool: failed.toolCall.toolName,
      error: failed.error.message,
    }),
    toolName: "invalid",
  }
}
```

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Tool Call Repair Flow                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Model calls: "Read" (wrong case)                                   │
│         │                                                           │
│         ▼                                                           │
│  ┌─────────────────────────────────────┐                            │
│  │ Tool "Read" not found               │                            │
│  │ Check lowercase: "read"             │                            │
│  └─────────────────────────────────────┘                            │
│         │                                                           │
│         ▼                                                           │
│  ┌─────────────────────────────────────┐                            │
│  │ "read" exists → repair to "read"    │                            │
│  └─────────────────────────────────────┘                            │
│         │                                                           │
│         ▼                                                           │
│  Tool executes successfully                                         │
│                                                                     │
│  ─────────────────────────────────────────────────────────────────  │
│                                                                     │
│  Model calls: "nonexistent_tool"                                    │
│         │                                                           │
│         ▼                                                           │
│  ┌─────────────────────────────────────┐                            │
│  │ Tool not found, no case match       │                            │
│  │ Redirect to "invalid" tool          │                            │
│  └─────────────────────────────────────┘                            │
│         │                                                           │
│         ▼                                                           │
│  "invalid" tool returns error message                               │
│  Model learns the tool doesn't exist                                │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Stream Events

The `fullStream` iterator yields different event types:

```typescript
for await (const event of result.fullStream) {
  switch (event.type) {
    case "text-delta":
      // Partial text content
      event.textDelta  // The new text chunk
      break
      
    case "reasoning":
      // Model's reasoning/thinking (for reasoning models)
      event.textDelta  // Reasoning text chunk
      break
      
    case "tool-call":
      // Model wants to call a tool
      event.toolCallId  // Unique ID for this call
      event.toolName    // Name of the tool
      event.args        // Parsed arguments
      break
      
    case "tool-result":
      // Result of a tool execution
      event.toolCallId  // Matches the tool-call ID
      event.result      // The tool's return value
      break
      
    case "tool-call-streaming-start":
      // Tool call is starting to stream
      event.toolCallId
      event.toolName
      break
      
    case "tool-call-delta":
      // Partial tool call arguments
      event.toolCallId
      event.argsTextDelta  // Partial JSON
      break
      
    case "finish":
      // Stream completed
      event.finishReason  // "stop", "tool-calls", "length", etc.
      event.usage         // Token usage statistics
      break
      
    case "error":
      // An error occurred
      event.error
      break
  }
}
```

### Event Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Stream Event Timeline                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Time ──────────────────────────────────────────────────────────▶   │
│                                                                     │
│  Text Response:                                                     │
│  ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌────────┐           │
│  │text- │ │text- │ │text- │ │text- │ │text- │ │finish  │           │
│  │delta │ │delta │ │delta │ │delta │ │delta │ │        │           │
│  │"Hel" │ │"lo, "│ │"I "  │ │"can "│ │"help"│ │stop    │           │
│  └──────┘ └──────┘ └──────┘ └──────┘ └──────┘ └────────┘           │
│                                                                     │
│  Tool Call Response:                                                │
│  ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌────────┐           │
│  │text- │ │tool- │ │tool- │ │tool- │ │tool- │ │finish  │           │
│  │delta │ │call- │ │call- │ │call  │ │result│ │        │           │
│  │"Let" │ │start │ │delta │ │      │ │      │ │tool-   │           │
│  │      │ │      │ │      │ │      │ │      │ │calls   │           │
│  └──────┘ └──────┘ └──────┘ └──────┘ └──────┘ └────────┘           │
│                                                                     │
│  Reasoning Model:                                                   │
│  ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌────────┐           │
│  │reason│ │reason│ │text- │ │text- │ │text- │ │finish  │           │
│  │ing   │ │ing   │ │delta │ │delta │ │delta │ │        │           │
│  │"Let" │ │"me"  │ │"The" │ │"ans" │ │"wer" │ │stop    │           │
│  └──────┘ └──────┘ └──────┘ └──────┘ └──────┘ └────────┘           │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## GitLab Workflow Integration

For GitLab's workflow models, opencode provides a tool executor:

```typescript
if (language instanceof GitLabWorkflowLanguageModel) {
  const workflowModel = language
  workflowModel.systemPrompt = system.join("\n")
  workflowModel.toolExecutor = async (toolName, argsJson, _requestID) => {
    const t = tools[toolName]
    if (!t || !t.execute) {
      return { result: "", error: `Unknown tool: ${toolName}` }
    }
    try {
      const result = await t.execute!(JSON.parse(argsJson), {
        toolCallId: _requestID,
        messages: input.messages,
        abortSignal: input.abort,
      })
      const output = typeof result === "string" ? result : (result?.output ?? JSON.stringify(result))
      return {
        result: output,
        metadata: typeof result === "object" ? result?.metadata : undefined,
        title: typeof result === "object" ? result?.title : undefined,
      }
    } catch (e: any) {
      return { result: "", error: e.message ?? String(e) }
    }
  }
}
```

---

## LiteLLM Proxy Compatibility

Some proxies require tools even when not actively using them:

```typescript
// LiteLLM and some Anthropic proxies require the tools parameter to be present
// when message history contains tool calls, even if no tools are being used.
const isLiteLLMProxy =
  provider.options?.["litellmProxy"] === true ||
  input.model.providerID.toLowerCase().includes("litellm") ||
  input.model.api.id.toLowerCase().includes("litellm")

if (isLiteLLMProxy && Object.keys(tools).length === 0 && hasToolCalls(input.messages)) {
  tools["_noop"] = tool({
    description:
      "Placeholder for LiteLLM/Anthropic proxy compatibility - required when message history contains tool calls but no active tools are needed",
    inputSchema: jsonSchema({ type: "object", properties: {} }),
    execute: async () => ({ output: "", title: "", metadata: {} }),
  })
}
```

### Helper Function

```typescript
export function hasToolCalls(messages: ModelMessage[]): boolean {
  for (const msg of messages) {
    if (!Array.isArray(msg.content)) continue
    for (const part of msg.content) {
      if (part.type === "tool-call" || part.type === "tool-result") return true
    }
  }
  return false
}
```

---

## Error Handling

Errors during streaming are handled via the `onError` callback:

```typescript
return streamText({
  onError(error) {
    l.error("stream error", { error })
  },
  // ...
})
```

Common error types:
- **Rate limiting**: Provider returns 429
- **Token limit exceeded**: Response too long
- **Invalid request**: Malformed messages or parameters
- **Network errors**: Connection issues
- **Abort**: User cancelled via AbortSignal

---

## Model Middleware

The middleware transforms messages before sending:

```typescript
model: wrapLanguageModel({
  model: language,
  middleware: [
    {
      async transformParams(args) {
        if (args.type === "stream") {
          args.params.prompt = ProviderTransform.message(
            args.params.prompt, 
            input.model, 
            options
          )
        }
        return args.params
      },
    },
  ],
}),
```

This ensures provider-specific transforms (empty content filtering, tool ID normalization, etc.) are applied.

---

## Usage Tracking

The finish event includes token usage:

```typescript
case "finish":
  event.usage.promptTokens      // Input tokens
  event.usage.completionTokens  // Output tokens
  event.usage.totalTokens       // Total
  
  // Some providers include cache info
  event.usage.cachedPromptTokens
```

---

## Self-Check Questions

1. **Why does opencode use streaming instead of waiting for complete responses?**
   <details>
   <summary>Answer</summary>
   Streaming provides better UX by showing responses as they're generated, allows early cancellation, and enables tool execution as soon as tool calls are received rather than waiting for the full response.
   </details>

2. **What does the `experimental_repairToolCall` callback do?**
   <details>
   <summary>Answer</summary>
   It attempts to fix malformed tool calls. First it tries case-insensitive matching (e.g., "Read" → "read"). If that fails, it redirects to an "invalid" tool that returns an error message to the model.
   </details>

3. **What's the difference between `tool-call` and `tool-call-streaming-start` events?**
   <details>
   <summary>Answer</summary>
   `tool-call-streaming-start` indicates a tool call is beginning to stream (you know the tool name but not all arguments yet). `tool-call` is emitted when the complete tool call with all arguments is ready for execution.
   </details>

4. **Why is the `_noop` tool added for LiteLLM proxies?**
   <details>
   <summary>Answer</summary>
   LiteLLM and some Anthropic proxies require the `tools` parameter to be present when the message history contains tool calls, even if no tools are currently active. The `_noop` tool satisfies this requirement.
   </details>

5. **How does the middleware transform messages?**
   <details>
   <summary>Answer</summary>
   The middleware's `transformParams` function intercepts stream requests and applies `ProviderTransform.message()` to normalize messages for the specific provider (handling empty content, tool IDs, cache control, etc.).
   </details>

---

## Exercises

### Exercise 1: Stream Event Logging

Write code to log all stream events with their types:

```typescript
const result = await LLM.stream(input)

for await (const event of result.fullStream) {
  console.log(`Event: ${event.type}`, event)
}
```

Run this and observe the event sequence for:
1. A simple text response
2. A response that calls a tool
3. A response from a reasoning model

### Exercise 2: Cancellation Handling

Implement a timeout that cancels the stream after 30 seconds:

```typescript
const controller = new AbortController()
const timeout = setTimeout(() => controller.abort(), 30000)

try {
  const result = await LLM.stream({
    ...input,
    abort: controller.signal,
  })
  // ... consume stream
} finally {
  clearTimeout(timeout)
}
```

### Exercise 3: Token Usage Tracking

Track cumulative token usage across multiple stream calls:

```typescript
let totalPromptTokens = 0
let totalCompletionTokens = 0

for await (const event of result.fullStream) {
  if (event.type === "finish") {
    totalPromptTokens += event.usage.promptTokens
    totalCompletionTokens += event.usage.completionTokens
    console.log(`Total: ${totalPromptTokens} in, ${totalCompletionTokens} out`)
  }
}
```

---

## Further Reading

- [Vercel AI SDK Streaming](https://sdk.vercel.ai/docs/ai-sdk-core/streaming)
- [Server-Sent Events (SSE)](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events)
- `packages/opencode/src/session/llm.ts` - Full streaming implementation
- `packages/opencode/src/session/processor.ts` - How the processor consumes streams
- `packages/opencode/src/session/prompt.ts` - System prompt construction
