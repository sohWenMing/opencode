# Module 05: LLM Providers

## Overview

This module covers how opencode integrates with various AI/LLM providers like Anthropic, OpenAI, Google, and many others. You'll learn about the Vercel AI SDK abstraction layer, the provider registry system, message transforms, and streaming mechanics.

**Estimated Time:** 6-8 hours

**Prerequisites:**
- Module 04 (Server Layer) completed
- Basic understanding of REST APIs
- Familiarity with async/await and iterators

---

## Module Structure

```
┌─────────────────────────────────────────────────────────────────────┐
│                    LLM Provider Architecture                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                    Application Layer                         │   │
│  │                   (Session, Processor)                       │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                               │                                     │
│                               ▼                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                    LLM.stream()                              │   │
│  │              packages/opencode/src/session/llm.ts            │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                               │                                     │
│                               ▼                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                    Vercel AI SDK                             │   │
│  │                    (streamText, tools)                       │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                               │                                     │
│         ┌─────────────────────┼─────────────────────┐               │
│         │                     │                     │               │
│         ▼                     ▼                     ▼               │
│  ┌─────────────┐       ┌─────────────┐       ┌─────────────┐       │
│  │ @ai-sdk/   │       │ @ai-sdk/   │       │ @ai-sdk/   │       │
│  │ anthropic  │       │ openai     │       │ google     │       │
│  └─────────────┘       └─────────────┘       └─────────────┘       │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Lessons

| # | Lesson | Description | Key Files |
|---|--------|-------------|-----------|
| 01 | [Vercel AI SDK](./01-vercel-ai-sdk.md) | Unified interface for LLMs | `ai` package |
| 02 | [Provider Registry](./02-provider-registry.md) | How providers are registered | `provider/provider.ts` |
| 03 | [Model Transforms](./03-model-transforms.md) | Provider-specific normalization | `provider/transform.ts` |
| 04 | [Streaming Mechanics](./04-streaming-mechanics.md) | How streaming works | `session/llm.ts` |

---

## Key Concepts

### 1. Provider Abstraction

opencode uses the Vercel AI SDK to abstract away differences between LLM providers:

```typescript
// Same code works for any provider
const result = await streamText({
  model: languageModel,  // Could be Anthropic, OpenAI, Google, etc.
  messages,
  tools,
})
```

### 2. Provider Registry

Providers are registered in `BUNDLED_PROVIDERS` and can have custom initialization via `CUSTOM_LOADERS`:

```typescript
const BUNDLED_PROVIDERS = {
  "@ai-sdk/anthropic": createAnthropic,
  "@ai-sdk/openai": createOpenAI,
  // ... 20+ providers
}

const CUSTOM_LOADERS = {
  anthropic: async () => ({
    autoload: false,
    options: { headers: { "anthropic-beta": "..." } }
  }),
  // ... provider-specific setup
}
```

### 3. Message Transforms

Different providers have different requirements. Transforms normalize messages:

```typescript
// Anthropic: Remove empty content
// Mistral: Normalize tool IDs to 9 chars
// Claude: Add cache control
msgs = ProviderTransform.message(msgs, model, options)
```

### 4. Streaming

Responses are streamed for better UX:

```typescript
for await (const event of result.fullStream) {
  switch (event.type) {
    case "text-delta":
      // Show text as it arrives
      break
    case "tool-call":
      // Execute tool immediately
      break
  }
}
```

---

## Key Files

| File | Purpose |
|------|---------|
| `packages/opencode/src/provider/provider.ts` | Provider registry, model loading |
| `packages/opencode/src/provider/transform.ts` | Message and schema transforms |
| `packages/opencode/src/provider/models.ts` | models.dev catalog loading |
| `packages/opencode/src/provider/schema.ts` | ProviderID and ModelID types |
| `packages/opencode/src/session/llm.ts` | Streaming implementation |
| `packages/opencode/src/provider/sdk/copilot/` | Custom Copilot provider |

---

## Supported Providers

opencode supports 20+ LLM providers out of the box:

| Provider | Package | Notes |
|----------|---------|-------|
| Anthropic | `@ai-sdk/anthropic` | Claude models |
| OpenAI | `@ai-sdk/openai` | GPT models |
| Google | `@ai-sdk/google` | Gemini models |
| Azure | `@ai-sdk/azure` | Azure OpenAI |
| Amazon Bedrock | `@ai-sdk/amazon-bedrock` | AWS-hosted models |
| Google Vertex | `@ai-sdk/google-vertex` | GCP-hosted models |
| OpenRouter | `@openrouter/ai-sdk-provider` | Multi-provider gateway |
| GitHub Copilot | Custom | VS Code integration |
| Mistral | `@ai-sdk/mistral` | Mistral AI |
| Groq | `@ai-sdk/groq` | Fast inference |
| xAI | `@ai-sdk/xai` | Grok models |
| Cerebras | `@ai-sdk/cerebras` | Fast inference |
| DeepInfra | `@ai-sdk/deepinfra` | Open models |
| Together AI | `@ai-sdk/togetherai` | Open models |
| Perplexity | `@ai-sdk/perplexity` | Search-augmented |
| Cohere | `@ai-sdk/cohere` | Command models |
| GitLab | `gitlab-ai-provider` | GitLab Duo |
| Vercel | `@ai-sdk/vercel` | Vercel AI Gateway |

---

## Learning Path

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Recommended Learning Path                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Day 1: Foundations                                                 │
│  ├─ Lesson 01: Vercel AI SDK (1.5 hours)                           │
│  │   └─ Understand the abstraction layer                           │
│  └─ Lesson 02: Provider Registry (1.5 hours)                       │
│      └─ How providers are loaded                                   │
│                                                                     │
│  Day 2: Implementation Details                                      │
│  ├─ Lesson 03: Model Transforms (1.5 hours)                        │
│  │   └─ Provider-specific handling                                 │
│  └─ Lesson 04: Streaming Mechanics (1.5 hours)                     │
│      └─ How responses are streamed                                 │
│                                                                     │
│  Day 3: Practice                                                    │
│  └─ Exercises: Provider Exploration (2-3 hours)                    │
│      └─ Hands-on exploration of the codebase                       │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Exercises

The [exercises](./exercises/) folder contains hands-on activities:

| Exercise | Description | Time |
|----------|-------------|------|
| [01-provider-exploration](./exercises/01-provider-exploration.md) | Trace provider loading, transforms, streaming | 2-3 hours |

---

## Quick Reference

### Get a Provider

```typescript
import { Provider } from "./provider/provider"

const provider = await Provider.getProvider(ProviderID.make("anthropic"))
```

### Get a Model

```typescript
const model = await Provider.getModel(
  ProviderID.make("anthropic"),
  ModelID.make("claude-sonnet-4-20250514")
)
```

### Get a Language Model

```typescript
const language = await Provider.getLanguage(model)
```

### Stream a Response

```typescript
import { LLM } from "./session/llm"

const result = await LLM.stream({
  model,
  messages,
  tools,
  abort: controller.signal,
  // ... other options
})

for await (const event of result.fullStream) {
  // Handle events
}
```

---

## Common Patterns

### Adding a New Provider

1. Check if there's an `@ai-sdk/*` package
2. Add to `BUNDLED_PROVIDERS` if bundling
3. Add `CUSTOM_LOADER` if special initialization needed
4. Add model definitions to models.dev or config

### Handling Provider Differences

1. Add transform in `ProviderTransform.normalizeMessages()`
2. Add schema transform in `ProviderTransform.schema()` if needed
3. Add options in `ProviderTransform.options()` if needed

### Debugging Provider Issues

1. Check logs: `OPENCODE_DEBUG=provider:*`
2. Verify API key: `opencode auth <provider>`
3. Check model availability: `opencode model list`
4. Test with minimal config

---

## Next Steps

After completing this module, continue to:

- **Module 06: Session System** - How the agent loop works
- **Module 07: Tool System** - How tools are defined and executed

---

## Further Reading

- [Vercel AI SDK Documentation](https://sdk.vercel.ai/docs)
- [models.dev](https://models.dev) - Model catalog
- [Anthropic API Docs](https://docs.anthropic.com)
- [OpenAI API Docs](https://platform.openai.com/docs)
- [Google AI Docs](https://ai.google.dev/docs)
