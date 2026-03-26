# Lesson 01: Vercel AI SDK

## Learning Objectives

By the end of this lesson, you will be able to:
- Explain what the Vercel AI SDK is and why it exists
- Understand the LanguageModel interface abstraction
- Use the `streamText` function for streaming LLM responses
- Define tools using the AI SDK's tool system
- Understand why opencode chose this SDK for provider abstraction

---

## What is the Vercel AI SDK?

The Vercel AI SDK (also known as `ai`) is a unified TypeScript library for building AI-powered applications. It provides a **single interface** to work with multiple LLM providers without changing your application code.

```
┌─────────────────────────────────────────────────────────────────────┐
│                      Vercel AI SDK Architecture                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│                    ┌─────────────────────┐                          │
│                    │   Your Application  │                          │
│                    │    (opencode)       │                          │
│                    └──────────┬──────────┘                          │
│                               │                                     │
│                    ┌──────────▼──────────┐                          │
│                    │   Vercel AI SDK     │                          │
│                    │   (Unified API)     │                          │
│                    └──────────┬──────────┘                          │
│                               │                                     │
│         ┌─────────────────────┼─────────────────────┐               │
│         │                     │                     │               │
│  ┌──────▼──────┐  ┌───────────▼───────────┐  ┌─────▼─────┐         │
│  │ @ai-sdk/   │  │ @ai-sdk/              │  │ @ai-sdk/ │         │
│  │ anthropic  │  │ openai                │  │ google   │         │
│  └──────┬──────┘  └───────────┬───────────┘  └─────┬─────┘         │
│         │                     │                     │               │
│  ┌──────▼──────┐  ┌───────────▼───────────┐  ┌─────▼─────┐         │
│  │  Anthropic  │  │       OpenAI          │  │  Google   │         │
│  │    API      │  │        API            │  │   API     │         │
│  └─────────────┘  └───────────────────────┘  └───────────┘         │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Key Benefits

| Benefit | Description |
|---------|-------------|
| **Provider Abstraction** | Switch between Claude, GPT, Gemini without code changes |
| **Unified Streaming** | Same streaming API regardless of provider |
| **Type Safety** | Full TypeScript support with generics |
| **Tool Support** | Standardized tool/function calling interface |
| **Middleware** | Transform requests/responses uniformly |

---

## Core Concepts

### The `ai` Package

The main package provides core functionality:

```typescript
import {
  streamText,           // Stream text from an LLM
  generateText,         // Generate text (non-streaming)
  tool,                 // Define a tool
  jsonSchema,           // Create JSON schemas for tools
  wrapLanguageModel,    // Add middleware to models
  type ModelMessage,    // Message type
  type Tool,            // Tool type
  type ToolSet,         // Collection of tools
} from "ai"
```

### Provider Packages

Each LLM provider has its own package:

```typescript
import { createAnthropic } from "@ai-sdk/anthropic"
import { createOpenAI } from "@ai-sdk/openai"
import { createGoogleGenerativeAI } from "@ai-sdk/google"
import { createAmazonBedrock } from "@ai-sdk/amazon-bedrock"
import { createAzure } from "@ai-sdk/azure"
```

---

## The LanguageModel Interface

The SDK defines a `LanguageModelV2` interface that all providers must implement. This is the key abstraction that enables provider switching.

```
┌─────────────────────────────────────────────────────────────────────┐
│                    LanguageModelV2 Interface                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  interface LanguageModelV2 {                                        │
│    // Provider identifier                                           │
│    readonly provider: string                                        │
│                                                                     │
│    // Model identifier                                              │
│    readonly modelId: string                                         │
│                                                                     │
│    // Generate a response (streaming)                               │
│    doStream(params: DoStreamParams): Promise<DoStreamResult>        │
│                                                                     │
│    // Generate a response (non-streaming)                           │
│    doGenerate(params: DoGenerateParams): Promise<DoGenerateResult>  │
│  }                                                                  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### How opencode Gets a Language Model

From `packages/opencode/src/provider/provider.ts`:

```typescript
export async function getLanguage(model: Model): Promise<LanguageModelV2> {
  const s = await state()
  const key = `${model.providerID}/${model.id}`
  if (s.models.has(key)) return s.models.get(key)!

  const provider = s.providers[model.providerID]
  const sdk = await getSDK(model)

  try {
    const language = s.modelLoaders[model.providerID]
      ? await s.modelLoaders[model.providerID](sdk, model.api.id, { ...provider.options, ...model.options })
      : sdk.languageModel(model.api.id)
    s.models.set(key, language)
    return language
  } catch (e) {
    if (e instanceof NoSuchModelError)
      throw new ModelNotFoundError({
        modelID: model.id,
        providerID: model.providerID,
      }, { cause: e })
    throw e
  }
}
```

Key points:
- Models are cached by `providerID/modelID` key
- Custom loaders can override default model creation
- Falls back to `sdk.languageModel(modelId)` for standard providers

---

## The `streamText` Function

`streamText` is the primary function for streaming LLM responses. It returns an async iterable of stream events.

### Basic Usage

```typescript
import { streamText } from "ai"

const result = await streamText({
  model: languageModel,           // LanguageModelV2 instance
  messages: [                     // Conversation history
    { role: "user", content: "Hello!" }
  ],
  tools: { /* tool definitions */ },
  maxOutputTokens: 4096,
  temperature: 0.7,
  abortSignal: controller.signal, // For cancellation
})

// Consume the stream
for await (const event of result.fullStream) {
  switch (event.type) {
    case "text-delta":
      process.stdout.write(event.textDelta)
      break
    case "tool-call":
      console.log("Tool called:", event.toolName)
      break
    // ... more event types
  }
}
```

### How opencode Uses streamText

From `packages/opencode/src/session/llm.ts`:

```typescript
return streamText({
  onError(error) {
    l.error("stream error", { error })
  },
  async experimental_repairToolCall(failed) {
    const lower = failed.toolCall.toolName.toLowerCase()
    if (lower !== failed.toolCall.toolName && tools[lower]) {
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
  headers: { /* ... */ },
  maxRetries: input.retries ?? 0,
  messages,
  model: wrapLanguageModel({
    model: language,
    middleware: [
      {
        async transformParams(args) {
          if (args.type === "stream") {
            args.params.prompt = ProviderTransform.message(
              args.params.prompt, input.model, options
            )
          }
          return args.params
        },
      },
    ],
  }),
})
```

### streamText Parameters

```
┌─────────────────────────────────────────────────────────────────────┐
│                    streamText Configuration                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Required:                                                          │
│  ├─ model           LanguageModelV2 instance                        │
│  └─ messages        Array of ModelMessage                           │
│                                                                     │
│  Optional:                                                          │
│  ├─ tools           Record<string, Tool>                            │
│  ├─ toolChoice      "auto" | "required" | "none"                    │
│  ├─ maxOutputTokens Maximum tokens to generate                      │
│  ├─ temperature     Randomness (0-2)                                │
│  ├─ topP            Nucleus sampling                                │
│  ├─ topK            Top-k sampling                                  │
│  ├─ abortSignal     For cancellation                                │
│  ├─ maxRetries      Retry count on failure                          │
│  ├─ headers         Custom HTTP headers                             │
│  ├─ providerOptions Provider-specific settings                      │
│  └─ onError         Error callback                                  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Tool Definitions

The AI SDK provides a standardized way to define tools that LLMs can call.

### Creating a Tool

```typescript
import { tool, jsonSchema } from "ai"

const readFileTool = tool({
  description: "Read the contents of a file",
  inputSchema: jsonSchema({
    type: "object",
    properties: {
      path: {
        type: "string",
        description: "The path to the file to read"
      }
    },
    required: ["path"]
  }),
  execute: async ({ path }) => {
    const content = await Bun.file(path).text()
    return { output: content, title: `Read ${path}` }
  }
})
```

### Tool Schema Structure

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Tool Definition                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  tool({                                                             │
│    description: string,      // Shown to the LLM                    │
│    inputSchema: JSONSchema,  // Parameters the tool accepts         │
│    execute: async (input, context) => {                             │
│      // input: parsed parameters                                    │
│      // context: { toolCallId, messages, abortSignal }              │
│      return {                                                       │
│        output: string,       // Result text                         │
│        title?: string,       // Display title                       │
│        metadata?: object,    // Additional data                     │
│      }                                                              │
│    }                                                                │
│  })                                                                 │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Tool Execution Context

The `execute` function receives context about the call:

```typescript
execute: async (input, context) => {
  context.toolCallId   // Unique ID for this tool call
  context.messages     // Full conversation history
  context.abortSignal  // For cancellation
}
```

---

## Model Middleware

The SDK supports middleware for transforming requests and responses:

```typescript
import { wrapLanguageModel } from "ai"

const wrappedModel = wrapLanguageModel({
  model: baseModel,
  middleware: [
    {
      async transformParams(args) {
        // Modify parameters before sending to provider
        if (args.type === "stream") {
          args.params.prompt = transformMessages(args.params.prompt)
        }
        return args.params
      },
    },
  ],
})
```

opencode uses this to apply provider-specific message transformations:

```typescript
model: wrapLanguageModel({
  model: language,
  middleware: [
    {
      async transformParams(args) {
        if (args.type === "stream") {
          args.params.prompt = ProviderTransform.message(
            args.params.prompt, input.model, options
          )
        }
        return args.params
      },
    },
  ],
}),
```

---

## Why opencode Uses the Vercel AI SDK

### 1. Provider Flexibility

Users can choose their preferred LLM provider without opencode needing separate implementations:

```typescript
// Same code works for all providers
const result = await streamText({
  model: language,  // Could be Anthropic, OpenAI, Google, etc.
  messages,
  tools,
})
```

### 2. Unified Streaming

All providers return the same stream event types:

```typescript
for await (const event of result.fullStream) {
  // Same event handling regardless of provider
  if (event.type === "text-delta") { /* ... */ }
  if (event.type === "tool-call") { /* ... */ }
}
```

### 3. Built-in Tool Support

The SDK handles tool calling differences between providers:

```
┌─────────────────────────────────────────────────────────────────────┐
│              Tool Calling Abstraction                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  OpenAI:     function_call / tool_calls                             │
│  Anthropic:  tool_use blocks                                        │
│  Google:     functionCall                                           │
│                                                                     │
│                    ↓ AI SDK normalizes to ↓                         │
│                                                                     │
│  Unified:    { type: "tool-call", toolName, args }                  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 4. Active Maintenance

The SDK is actively maintained by Vercel and the community, with regular updates for new provider features.

---

## Message Types

The SDK uses a standardized message format:

```typescript
type ModelMessage = 
  | { role: "system"; content: string }
  | { role: "user"; content: string | ContentPart[] }
  | { role: "assistant"; content: string | ContentPart[] }
  | { role: "tool"; content: ToolResultPart[] }

type ContentPart = 
  | { type: "text"; text: string }
  | { type: "image"; image: string | URL }
  | { type: "file"; mediaType: string; data: string }
  | { type: "tool-call"; toolCallId: string; toolName: string; args: unknown }
  | { type: "tool-result"; toolCallId: string; result: unknown }
```

---

## Self-Check Questions

1. **What problem does the Vercel AI SDK solve?**
   <details>
   <summary>Answer</summary>
   It provides a unified interface for multiple LLM providers, allowing applications to switch between providers without code changes. It abstracts away differences in API formats, streaming protocols, and tool calling conventions.
   </details>

2. **What is the `LanguageModelV2` interface?**
   <details>
   <summary>Answer</summary>
   It's the core abstraction that all provider implementations must implement. It defines methods like `doStream` and `doGenerate` that handle the actual LLM communication.
   </details>

3. **Why does opencode use `wrapLanguageModel` with middleware?**
   <details>
   <summary>Answer</summary>
   To apply provider-specific message transformations before sending requests. Different providers have different requirements (e.g., Anthropic rejects empty content), and middleware allows transforming messages uniformly.
   </details>

4. **What's the difference between `streamText` and `generateText`?**
   <details>
   <summary>Answer</summary>
   `streamText` returns an async iterable that yields events as they arrive (real-time streaming). `generateText` waits for the complete response before returning (blocking).
   </details>

5. **How does the SDK handle tool calling differences between providers?**
   <details>
   <summary>Answer</summary>
   Each provider package translates the provider's native tool format to/from the SDK's unified format. The application code only sees standardized `tool-call` and `tool-result` events.
   </details>

---

## Exercises

### Exercise 1: Explore the AI Package

Look at the imports in `packages/opencode/src/session/llm.ts`:

```typescript
import {
  streamText,
  wrapLanguageModel,
  type ModelMessage,
  type StreamTextResult,
  type Tool,
  type ToolSet,
  tool,
  jsonSchema,
} from "ai"
```

For each import, write a one-sentence description of what it's used for in opencode.

### Exercise 2: Trace a Tool Definition

Find a tool definition in `packages/opencode/src/tool/` and identify:
1. The tool's description
2. The input schema (what parameters it accepts)
3. The execute function's return type

### Exercise 3: Stream Event Types

The `fullStream` iterator yields different event types. List all the event types you can find in the codebase and describe what each represents.

---

## Further Reading

- [Vercel AI SDK Documentation](https://sdk.vercel.ai/docs)
- [AI SDK GitHub Repository](https://github.com/vercel/ai)
- [Provider-specific documentation](https://sdk.vercel.ai/providers)
- `packages/opencode/src/session/llm.ts` - opencode's streaming implementation
- `packages/opencode/src/provider/provider.ts` - Provider loading and model resolution
