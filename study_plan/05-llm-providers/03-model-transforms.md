# Lesson 03: Model Transforms

## Learning Objectives

By the end of this lesson, you will be able to:
- Explain why message transforms are needed for different providers
- Understand the ProviderTransform module structure
- Trace how messages are normalized for specific providers
- Understand schema transforms for tool definitions
- Configure provider-specific options and variants

---

## Why Transforms Are Needed

Different LLM providers have different requirements and quirks:

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Provider Differences                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Anthropic:                                                         │
│  ├─ Rejects messages with empty content                             │
│  ├─ Tool call IDs must match specific format                        │
│  └─ Supports cache control on messages                              │
│                                                                     │
│  OpenAI:                                                            │
│  ├─ Uses "responses" API for newer models                           │
│  ├─ Supports prompt caching with cache keys                         │
│  └─ Has reasoning effort levels                                     │
│                                                                     │
│  Google/Gemini:                                                     │
│  ├─ Enum values must be strings (not integers)                      │
│  ├─ Array items need explicit type                                  │
│  └─ Uses thinkingConfig for reasoning                               │
│                                                                     │
│  Mistral:                                                           │
│  ├─ Tool call IDs must be exactly 9 alphanumeric characters         │
│  └─ Tool messages can't be followed by user messages                │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

The `ProviderTransform` module normalizes these differences.

---

## The ProviderTransform Module

Located in `packages/opencode/src/provider/transform.ts`, this module provides functions to transform messages, options, and schemas for different providers.

### Module Structure

```
┌─────────────────────────────────────────────────────────────────────┐
│                    ProviderTransform Functions                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Message Transforms:                                                │
│  ├─ message()        Main entry point for message transformation    │
│  ├─ normalizeMessages()  Provider-specific message normalization    │
│  ├─ applyCaching()   Add cache control to messages                  │
│  └─ unsupportedParts()  Handle unsupported modalities               │
│                                                                     │
│  Option Transforms:                                                 │
│  ├─ options()        Generate provider-specific options             │
│  ├─ smallOptions()   Options for "small" model calls                │
│  ├─ providerOptions()  Wrap options in provider namespace           │
│  ├─ temperature()    Default temperature per model                  │
│  ├─ topP()           Default topP per model                         │
│  ├─ topK()           Default topK per model                         │
│  └─ variants()       Reasoning effort variants                      │
│                                                                     │
│  Schema Transforms:                                                 │
│  └─ schema()         Transform tool schemas for providers           │
│                                                                     │
│  Utilities:                                                         │
│  ├─ maxOutputTokens()  Calculate max output tokens                  │
│  └─ OUTPUT_TOKEN_MAX   Default max output limit                     │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Message Normalization

The `message()` function is the main entry point:

```typescript
export function message(msgs: ModelMessage[], model: Provider.Model, options: Record<string, unknown>) {
  msgs = unsupportedParts(msgs, model)
  msgs = normalizeMessages(msgs, model, options)
  if (
    (model.providerID === "anthropic" ||
      model.api.id.includes("anthropic") ||
      model.api.id.includes("claude") ||
      model.id.includes("anthropic") ||
      model.id.includes("claude") ||
      model.api.npm === "@ai-sdk/anthropic") &&
    model.api.npm !== "@ai-sdk/gateway"
  ) {
    msgs = applyCaching(msgs, model)
  }

  // Remap providerOptions keys from stored providerID to expected SDK key
  const key = sdkKey(model.api.npm)
  if (key && key !== model.providerID && model.api.npm !== "@ai-sdk/azure") {
    // ... remap logic
  }

  return msgs
}
```

### Anthropic Empty Content Handling

Anthropic rejects messages with empty content:

```typescript
function normalizeMessages(
  msgs: ModelMessage[],
  model: Provider.Model,
  options: Record<string, unknown>,
): ModelMessage[] {
  // Anthropic rejects messages with empty content - filter out empty string messages
  // and remove empty text/reasoning parts from array content
  if (model.api.npm === "@ai-sdk/anthropic" || model.api.npm === "@ai-sdk/amazon-bedrock") {
    msgs = msgs
      .map((msg) => {
        if (typeof msg.content === "string") {
          if (msg.content === "") return undefined
          return msg
        }
        if (!Array.isArray(msg.content)) return msg
        const filtered = msg.content.filter((part) => {
          if (part.type === "text" || part.type === "reasoning") {
            return part.text !== ""
          }
          return true
        })
        if (filtered.length === 0) return undefined
        return { ...msg, content: filtered }
      })
      .filter((msg): msg is ModelMessage => msg !== undefined && msg.content !== "")
  }
  // ...
}
```

### Claude Tool Call ID Normalization

Claude requires specific tool call ID formats:

```typescript
if (model.api.id.includes("claude")) {
  return msgs.map((msg) => {
    if ((msg.role === "assistant" || msg.role === "tool") && Array.isArray(msg.content)) {
      msg.content = msg.content.map((part) => {
        if ((part.type === "tool-call" || part.type === "tool-result") && "toolCallId" in part) {
          return {
            ...part,
            toolCallId: part.toolCallId.replace(/[^a-zA-Z0-9_-]/g, "_"),
          }
        }
        return part
      })
    }
    return msg
  })
}
```

### Mistral Special Handling

Mistral has strict requirements:

```typescript
if (
  model.providerID === "mistral" ||
  model.api.id.toLowerCase().includes("mistral") ||
  model.api.id.toLocaleLowerCase().includes("devstral")
) {
  const result: ModelMessage[] = []
  for (let i = 0; i < msgs.length; i++) {
    const msg = msgs[i]
    const nextMsg = msgs[i + 1]

    if ((msg.role === "assistant" || msg.role === "tool") && Array.isArray(msg.content)) {
      msg.content = msg.content.map((part) => {
        if ((part.type === "tool-call" || part.type === "tool-result") && "toolCallId" in part) {
          // Mistral requires alphanumeric tool call IDs with exactly 9 characters
          const normalizedId = part.toolCallId
            .replace(/[^a-zA-Z0-9]/g, "") // Remove non-alphanumeric characters
            .substring(0, 9) // Take first 9 characters
            .padEnd(9, "0") // Pad with zeros if less than 9 characters

          return {
            ...part,
            toolCallId: normalizedId,
          }
        }
        return part
      })
    }

    result.push(msg)

    // Fix message sequence: tool messages cannot be followed by user messages
    if (msg.role === "tool" && nextMsg?.role === "user") {
      result.push({
        role: "assistant",
        content: [{ type: "text", text: "Done." }],
      })
    }
  }
  return result
}
```

### Interleaved Reasoning Handling

Some providers use special fields for reasoning content:

```typescript
if (typeof model.capabilities.interleaved === "object" && model.capabilities.interleaved.field) {
  const field = model.capabilities.interleaved.field
  return msgs.map((msg) => {
    if (msg.role === "assistant" && Array.isArray(msg.content)) {
      const reasoningParts = msg.content.filter((part: any) => part.type === "reasoning")
      const reasoningText = reasoningParts.map((part: any) => part.text).join("")

      // Filter out reasoning parts from content
      const filteredContent = msg.content.filter((part: any) => part.type !== "reasoning")

      if (reasoningText) {
        return {
          ...msg,
          content: filteredContent,
          providerOptions: {
            ...msg.providerOptions,
            openaiCompatible: {
              ...(msg.providerOptions as any)?.openaiCompatible,
              [field]: reasoningText,
            },
          },
        }
      }

      return { ...msg, content: filteredContent }
    }
    return msg
  })
}
```

---

## Cache Control

For Anthropic and compatible providers, opencode adds cache control to optimize token usage:

```typescript
function applyCaching(msgs: ModelMessage[], model: Provider.Model): ModelMessage[] {
  const system = msgs.filter((msg) => msg.role === "system").slice(0, 2)
  const final = msgs.filter((msg) => msg.role !== "system").slice(-2)

  const providerOptions = {
    anthropic: {
      cacheControl: { type: "ephemeral" },
    },
    openrouter: {
      cacheControl: { type: "ephemeral" },
    },
    bedrock: {
      cachePoint: { type: "default" },
    },
    openaiCompatible: {
      cache_control: { type: "ephemeral" },
    },
    copilot: {
      copilot_cache_control: { type: "ephemeral" },
    },
  }

  for (const msg of unique([...system, ...final])) {
    const useMessageLevelOptions =
      model.providerID === "anthropic" ||
      model.providerID.includes("bedrock") ||
      model.api.npm === "@ai-sdk/amazon-bedrock"
    const shouldUseContentOptions = !useMessageLevelOptions && 
      Array.isArray(msg.content) && msg.content.length > 0

    if (shouldUseContentOptions) {
      const lastContent = msg.content[msg.content.length - 1]
      if (lastContent && typeof lastContent === "object") {
        lastContent.providerOptions = mergeDeep(lastContent.providerOptions ?? {}, providerOptions)
        continue
      }
    }

    msg.providerOptions = mergeDeep(msg.providerOptions ?? {}, providerOptions)
  }

  return msgs
}
```

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Cache Control Strategy                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Messages marked for caching:                                       │
│  ├─ First 2 system messages (static prompt)                         │
│  └─ Last 2 messages (recent context)                                │
│                                                                     │
│  Why these?                                                         │
│  ├─ System prompt rarely changes → high cache hit rate              │
│  └─ Recent messages likely to be repeated in follow-ups             │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Unsupported Modalities

The `unsupportedParts` function handles content types the model doesn't support:

```typescript
function unsupportedParts(msgs: ModelMessage[], model: Provider.Model): ModelMessage[] {
  return msgs.map((msg) => {
    if (msg.role !== "user" || !Array.isArray(msg.content)) return msg

    const filtered = msg.content.map((part) => {
      if (part.type !== "file" && part.type !== "image") return part

      // Check for empty base64 image data
      if (part.type === "image") {
        const imageStr = part.image.toString()
        if (imageStr.startsWith("data:")) {
          const match = imageStr.match(/^data:([^;]+);base64,(.*)$/)
          if (match && (!match[2] || match[2].length === 0)) {
            return {
              type: "text" as const,
              text: "ERROR: Image file is empty or corrupted. Please provide a valid image.",
            }
          }
        }
      }

      const mime = part.type === "image" 
        ? part.image.toString().split(";")[0].replace("data:", "") 
        : part.mediaType
      const filename = part.type === "file" ? part.filename : undefined
      const modality = mimeToModality(mime)
      if (!modality) return part
      if (model.capabilities.input[modality]) return part

      const name = filename ? `"${filename}"` : modality
      return {
        type: "text" as const,
        text: `ERROR: Cannot read ${name} (this model does not support ${modality} input). Inform the user.`,
      }
    })

    return { ...msg, content: filtered }
  })
}
```

---

## Provider Options

### Default Temperature

Different models have different optimal temperatures:

```typescript
export function temperature(model: Provider.Model) {
  const id = model.id.toLowerCase()
  if (id.includes("qwen")) return 0.55
  if (id.includes("claude")) return undefined
  if (id.includes("gemini")) return 1.0
  if (id.includes("glm-4.6")) return 1.0
  if (id.includes("glm-4.7")) return 1.0
  if (id.includes("minimax-m2")) return 1.0
  if (id.includes("kimi-k2")) {
    if (["thinking", "k2.", "k2p", "k2-5"].some((s) => id.includes(s))) {
      return 1.0
    }
    return 0.6
  }
  return undefined
}
```

### Reasoning Variants

Models with reasoning capabilities have different "effort" levels:

```typescript
export function variants(model: Provider.Model): Record<string, Record<string, any>> {
  if (!model.capabilities.reasoning) return {}

  const id = model.id.toLowerCase()
  
  // ... provider-specific variant handling ...

  switch (model.api.npm) {
    case "@ai-sdk/openai":
      // OpenAI supports none, minimal, low, medium, high, xhigh
      const openaiEfforts = iife(() => {
        if (id.includes("codex")) {
          if (id.includes("5.2") || id.includes("5.3")) return [...WIDELY_SUPPORTED_EFFORTS, "xhigh"]
          return WIDELY_SUPPORTED_EFFORTS
        }
        const arr = [...WIDELY_SUPPORTED_EFFORTS]
        if (id.includes("gpt-5-") || id === "gpt-5") {
          arr.unshift("minimal")
        }
        if (model.release_date >= "2025-11-13") {
          arr.unshift("none")
        }
        if (model.release_date >= "2025-12-04") {
          arr.push("xhigh")
        }
        return arr
      })
      return Object.fromEntries(
        openaiEfforts.map((effort) => [
          effort,
          {
            reasoningEffort: effort,
            reasoningSummary: "auto",
            include: ["reasoning.encrypted_content"],
          },
        ]),
      )

    case "@ai-sdk/anthropic":
      // Anthropic uses thinking budget tokens
      return {
        high: {
          thinking: {
            type: "enabled",
            budgetTokens: Math.min(16_000, Math.floor(model.limit.output / 2 - 1)),
          },
        },
        max: {
          thinking: {
            type: "enabled",
            budgetTokens: Math.min(31_999, model.limit.output - 1),
          },
        },
      }

    case "@ai-sdk/google":
      // Google uses thinkingConfig
      return {
        high: {
          thinkingConfig: {
            includeThoughts: true,
            thinkingBudget: 16000,
          },
        },
        max: {
          thinkingConfig: {
            includeThoughts: true,
            thinkingBudget: 24576,
          },
        },
      }
    // ... more providers
  }
}
```

---

## Schema Transforms

Tool schemas need adjustment for certain providers:

```typescript
export function schema(model: Provider.Model, schema: JSONSchema.BaseSchema | JSONSchema7): JSONSchema7 {
  // Convert integer enums to string enums for Google/Gemini
  if (model.providerID === "google" || model.api.id.includes("gemini")) {
    const sanitizeGemini = (obj: any): any => {
      if (obj === null || typeof obj !== "object") {
        return obj
      }

      if (Array.isArray(obj)) {
        return obj.map(sanitizeGemini)
      }

      const result: any = {}
      for (const [key, value] of Object.entries(obj)) {
        if (key === "enum" && Array.isArray(value)) {
          // Convert all enum values to strings
          result[key] = value.map((v) => String(v))
          // If we have integer type with enum, change type to string
          if (result.type === "integer" || result.type === "number") {
            result.type = "string"
          }
        } else if (typeof value === "object" && value !== null) {
          result[key] = sanitizeGemini(value)
        } else {
          result[key] = value
        }
      }

      // Filter required array to only include fields that exist in properties
      if (result.type === "object" && result.properties && Array.isArray(result.required)) {
        result.required = result.required.filter((field: any) => field in result.properties)
      }

      if (result.type === "array" && !hasCombiner(result)) {
        if (result.items == null) {
          result.items = {}
        }
        if (isPlainObject(result.items) && !hasSchemaIntent(result.items)) {
          result.items.type = "string"
        }
      }

      // Remove properties/required from non-object types (Gemini rejects these)
      if (result.type && result.type !== "object" && !hasCombiner(result)) {
        delete result.properties
        delete result.required
      }

      return result
    }

    schema = sanitizeGemini(schema)
  }

  return schema as JSONSchema7
}
```

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Gemini Schema Requirements                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Issues Gemini has with standard JSON Schema:                       │
│                                                                     │
│  1. Integer enums → Must be string enums                            │
│     { type: "integer", enum: [1, 2, 3] }                            │
│     → { type: "string", enum: ["1", "2", "3"] }                     │
│                                                                     │
│  2. Array items must have type                                      │
│     { type: "array" }                                               │
│     → { type: "array", items: { type: "string" } }                  │
│                                                                     │
│  3. Non-object types can't have properties/required                 │
│     { type: "string", properties: {...} }                           │
│     → { type: "string" }                                            │
│                                                                     │
│  4. Required fields must exist in properties                        │
│     { required: ["a", "b"], properties: { a: {...} } }              │
│     → { required: ["a"], properties: { a: {...} } }                 │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Provider Options Mapping

The `providerOptions` function wraps options in the correct namespace:

```typescript
export function providerOptions(model: Provider.Model, options: { [x: string]: any }) {
  if (model.api.npm === "@ai-sdk/gateway") {
    // Gateway has special handling for nested provider options
    const i = model.api.id.indexOf("/")
    const rawSlug = i > 0 ? model.api.id.slice(0, i) : undefined
    const slug = rawSlug ? (SLUG_OVERRIDES[rawSlug] ?? rawSlug) : undefined
    const gateway = options.gateway
    const rest = Object.fromEntries(Object.entries(options).filter(([k]) => k !== "gateway"))
    const has = Object.keys(rest).length > 0

    const result: Record<string, any> = {}
    if (gateway !== undefined) result.gateway = gateway

    if (has) {
      if (slug) {
        result[slug] = rest
      } else if (gateway && typeof gateway === "object" && !Array.isArray(gateway)) {
        result.gateway = { ...gateway, ...rest }
      } else {
        result.gateway = rest
      }
    }

    return result
  }

  const key = sdkKey(model.api.npm) ?? model.providerID
  return { [key]: options }
}
```

### SDK Key Mapping

```typescript
function sdkKey(npm: string): string | undefined {
  switch (npm) {
    case "@ai-sdk/github-copilot":
      return "copilot"
    case "@ai-sdk/openai":
    case "@ai-sdk/azure":
      return "openai"
    case "@ai-sdk/amazon-bedrock":
      return "bedrock"
    case "@ai-sdk/anthropic":
    case "@ai-sdk/google-vertex/anthropic":
      return "anthropic"
    case "@ai-sdk/google-vertex":
    case "@ai-sdk/google":
      return "google"
    case "@ai-sdk/gateway":
      return "gateway"
    case "@openrouter/ai-sdk-provider":
      return "openrouter"
  }
  return undefined
}
```

---

## Self-Check Questions

1. **Why does Anthropic need empty content filtering?**
   <details>
   <summary>Answer</summary>
   Anthropic's API rejects messages with empty content strings or empty content arrays. This can happen when reasoning parts are filtered out or when messages are constructed programmatically.
   </details>

2. **What's special about Mistral's tool call IDs?**
   <details>
   <summary>Answer</summary>
   Mistral requires tool call IDs to be exactly 9 alphanumeric characters. The transform normalizes IDs by removing non-alphanumeric characters, truncating to 9 characters, and padding with zeros if needed.
   </details>

3. **Why does opencode add cache control to system messages?**
   <details>
   <summary>Answer</summary>
   System messages (prompts) rarely change between requests, so marking them for caching allows providers like Anthropic to reuse the cached representation, reducing token costs and latency.
   </details>

4. **How does the schema transform handle Gemini's enum requirements?**
   <details>
   <summary>Answer</summary>
   It converts all enum values to strings and changes the type from "integer" or "number" to "string". This is because Gemini doesn't support integer enums in tool schemas.
   </details>

5. **What are reasoning variants and why do they differ by provider?**
   <details>
   <summary>Answer</summary>
   Reasoning variants control how much "thinking" the model does. Different providers have different APIs: OpenAI uses `reasoningEffort`, Anthropic uses `thinking.budgetTokens`, Google uses `thinkingConfig.thinkingBudget`.
   </details>

---

## Exercises

### Exercise 1: Trace a Message Transform

Given this message array:
```typescript
const messages = [
  { role: "system", content: "You are helpful." },
  { role: "user", content: "" },
  { role: "assistant", content: [{ type: "text", text: "" }] }
]
```

What would the output be after `ProviderTransform.message()` for:
1. Anthropic
2. OpenAI
3. Mistral

### Exercise 2: Understand Variant Configuration

Look at the variants for `claude-sonnet-4` and `gpt-5`. Compare:
1. What effort levels are available?
2. What options does each level set?
3. How do the token budgets differ?

### Exercise 3: Schema Transform Testing

Create a JSON schema with:
- An integer enum property
- An array without explicit items type
- A required field that doesn't exist in properties

Then trace what `ProviderTransform.schema()` would produce for Gemini.

---

## Further Reading

- [Anthropic API Documentation](https://docs.anthropic.com/en/api/messages)
- [OpenAI API Documentation](https://platform.openai.com/docs/api-reference)
- [Google AI Documentation](https://ai.google.dev/docs)
- `packages/opencode/src/provider/transform.ts` - Full transform implementation
- `packages/opencode/src/session/llm.ts` - Where transforms are applied
