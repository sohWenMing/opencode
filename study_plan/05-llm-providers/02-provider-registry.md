# Lesson 02: Provider Registry

## Learning Objectives

By the end of this lesson, you will be able to:
- Understand how opencode registers and manages LLM providers
- Explain the BUNDLED_PROVIDERS map and its purpose
- Trace how a provider is loaded from configuration to SDK instance
- Understand CUSTOM_LOADERS for provider-specific behavior
- Use the `getProvider`, `getModel`, and `getLanguage` functions
- Configure providers via environment variables and config files

---

## Provider Architecture Overview

opencode supports 20+ LLM providers through a unified registry system. The architecture has three layers:

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Provider Registry Architecture                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                    Configuration Layer                       │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐  │   │
│  │  │ Environment │  │ Config File │  │ Auth Store          │  │   │
│  │  │ Variables   │  │ opencode.   │  │ (API keys)          │  │   │
│  │  │             │  │ json        │  │                     │  │   │
│  │  └─────────────┘  └─────────────┘  └─────────────────────┘  │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                               │                                     │
│                               ▼                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                    Registry Layer                            │   │
│  │  ┌─────────────────┐  ┌─────────────────────────────────┐   │   │
│  │  │ BUNDLED_        │  │ CUSTOM_LOADERS                  │   │   │
│  │  │ PROVIDERS       │  │ (provider-specific init)        │   │   │
│  │  │ (npm → factory) │  │                                 │   │   │
│  │  └─────────────────┘  └─────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                               │                                     │
│                               ▼                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                    SDK Layer                                 │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐  │   │
│  │  │ Anthropic   │  │ OpenAI      │  │ Google              │  │   │
│  │  │ SDK         │  │ SDK         │  │ SDK                 │  │   │
│  │  └─────────────┘  └─────────────┘  └─────────────────────┘  │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## The BUNDLED_PROVIDERS Map

The `BUNDLED_PROVIDERS` map connects npm package names to their factory functions. This is the core registry.

From `packages/opencode/src/provider/provider.ts`:

```typescript
const BUNDLED_PROVIDERS: Record<string, (options: any) => SDK> = {
  "@ai-sdk/amazon-bedrock": createAmazonBedrock,
  "@ai-sdk/anthropic": createAnthropic,
  "@ai-sdk/azure": createAzure,
  "@ai-sdk/google": createGoogleGenerativeAI,
  "@ai-sdk/google-vertex": createVertex,
  "@ai-sdk/google-vertex/anthropic": createVertexAnthropic,
  "@ai-sdk/openai": createOpenAI,
  "@ai-sdk/openai-compatible": createOpenAICompatible,
  "@openrouter/ai-sdk-provider": createOpenRouter,
  "@ai-sdk/xai": createXai,
  "@ai-sdk/mistral": createMistral,
  "@ai-sdk/groq": createGroq,
  "@ai-sdk/deepinfra": createDeepInfra,
  "@ai-sdk/cerebras": createCerebras,
  "@ai-sdk/cohere": createCohere,
  "@ai-sdk/gateway": createGateway,
  "@ai-sdk/togetherai": createTogetherAI,
  "@ai-sdk/perplexity": createPerplexity,
  "@ai-sdk/vercel": createVercel,
  "gitlab-ai-provider": createGitLab,
  "@ai-sdk/github-copilot": createGitHubCopilotOpenAICompatible,
}
```

### How It Works

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Provider Resolution Flow                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  1. Model specifies npm package:                                    │
│     model.api.npm = "@ai-sdk/anthropic"                             │
│                                                                     │
│  2. Look up in BUNDLED_PROVIDERS:                                   │
│     const bundledFn = BUNDLED_PROVIDERS["@ai-sdk/anthropic"]        │
│     // bundledFn = createAnthropic                                  │
│                                                                     │
│  3. Create SDK instance:                                            │
│     const sdk = bundledFn({                                         │
│       name: "anthropic",                                            │
│       apiKey: "sk-...",                                             │
│       baseURL: "https://api.anthropic.com",                         │
│       ...options                                                    │
│     })                                                              │
│                                                                     │
│  4. Get language model:                                             │
│     const model = sdk.languageModel("claude-sonnet-4-20250514")     │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Provider Imports

All bundled providers are directly imported at the top of the file:

```typescript
// Direct imports for bundled providers
import { createAmazonBedrock, type AmazonBedrockProviderSettings } from "@ai-sdk/amazon-bedrock"
import { createAnthropic } from "@ai-sdk/anthropic"
import { createAzure } from "@ai-sdk/azure"
import { createGoogleGenerativeAI } from "@ai-sdk/google"
import { createVertex } from "@ai-sdk/google-vertex"
import { createVertexAnthropic } from "@ai-sdk/google-vertex/anthropic"
import { createOpenAI } from "@ai-sdk/openai"
import { createOpenAICompatible } from "@ai-sdk/openai-compatible"
import { createOpenRouter, type LanguageModelV2 } from "@openrouter/ai-sdk-provider"
import { createOpenaiCompatible as createGitHubCopilotOpenAICompatible } from "./sdk/copilot"
import { createXai } from "@ai-sdk/xai"
import { createMistral } from "@ai-sdk/mistral"
import { createGroq } from "@ai-sdk/groq"
import { createDeepInfra } from "@ai-sdk/deepinfra"
import { createCerebras } from "@ai-sdk/cerebras"
import { createCohere } from "@ai-sdk/cohere"
import { createGateway } from "@ai-sdk/gateway"
import { createTogetherAI } from "@ai-sdk/togetherai"
import { createPerplexity } from "@ai-sdk/perplexity"
import { createVercel } from "@ai-sdk/vercel"
import {
  createGitLab,
  VERSION as GITLAB_PROVIDER_VERSION,
  isWorkflowModel,
  discoverWorkflowModels,
} from "gitlab-ai-provider"
```

Note the special import for GitHub Copilot - it uses a custom implementation from `./sdk/copilot`.

---

## CUSTOM_LOADERS

Some providers need special initialization logic. `CUSTOM_LOADERS` provides this:

```typescript
type CustomLoader = (provider: Info) => Promise<{
  autoload: boolean           // Should this provider auto-enable?
  getModel?: CustomModelLoader // Custom model creation
  vars?: CustomVarsLoader      // Environment variable mapping
  options?: Record<string, any> // Additional SDK options
  discoverModels?: CustomDiscoverModels // Dynamic model discovery
}>
```

### Example: Anthropic Loader

```typescript
const CUSTOM_LOADERS: Record<string, CustomLoader> = {
  async anthropic() {
    return {
      autoload: false,
      options: {
        headers: {
          "anthropic-beta": "interleaved-thinking-2025-05-14,fine-grained-tool-streaming-2025-05-14",
        },
      },
    }
  },
  // ...
}
```

This adds beta feature headers to all Anthropic requests.

### Example: OpenAI Loader

```typescript
openai: async () => {
  return {
    autoload: false,
    async getModel(sdk: any, modelID: string, _options?: Record<string, any>) {
      return sdk.responses(modelID)
    },
    options: {},
  }
},
```

OpenAI uses the `responses` API instead of `languageModel`.

### Example: Amazon Bedrock Loader

The Bedrock loader is complex - it handles AWS credentials, regions, and cross-region inference:

```typescript
"amazon-bedrock": async () => {
  const config = await Config.get()
  const providerConfig = config.provider?.["amazon-bedrock"]

  // Region precedence: 1) config file, 2) env var, 3) default
  const configRegion = providerConfig?.options?.region
  const envRegion = Env.get("AWS_REGION")
  const defaultRegion = configRegion ?? envRegion ?? "us-east-1"

  // Profile: config file takes precedence over env var
  const configProfile = providerConfig?.options?.profile
  const envProfile = Env.get("AWS_PROFILE")
  const profile = configProfile ?? envProfile

  // ... credential setup ...

  return {
    autoload: true,
    options: providerOptions,
    async getModel(sdk: any, modelID: string, options?: Record<string, any>) {
      // Handle cross-region inference prefixes
      const crossRegionPrefixes = ["global.", "us.", "eu.", "jp.", "apac.", "au."]
      if (crossRegionPrefixes.some((prefix) => modelID.startsWith(prefix))) {
        return sdk.languageModel(modelID)
      }

      // Add region prefix for certain models
      const region = options?.region ?? defaultRegion
      let regionPrefix = region.split("-")[0]

      switch (regionPrefix) {
        case "us": {
          const modelRequiresPrefix = [
            "nova-micro", "nova-lite", "nova-pro", "nova-premier",
            "nova-2", "claude", "deepseek",
          ].some((m) => modelID.includes(m))
          if (modelRequiresPrefix && !region.startsWith("us-gov")) {
            modelID = `${regionPrefix}.${modelID}`
          }
          break
        }
        // ... more region handling
      }

      return sdk.languageModel(modelID)
    },
  }
},
```

### Example: GitHub Copilot Loader

```typescript
"github-copilot": async () => {
  return {
    autoload: false,
    async getModel(sdk: any, modelID: string, _options?: Record<string, any>) {
      if (useLanguageModel(sdk)) return sdk.languageModel(modelID)
      return shouldUseCopilotResponsesApi(modelID) 
        ? sdk.responses(modelID) 
        : sdk.chat(modelID)
    },
    options: {},
  }
},
```

Copilot chooses between `responses`, `chat`, or `languageModel` based on the model.

---

## Provider State Management

The provider registry uses a lazy-initialized state object:

```typescript
const state = Instance.state(async () => {
  const config = await Config.get()
  const modelsDev = await ModelsDev.get()
  const database = mapValues(modelsDev, fromModelsDevProvider)

  const disabled = new Set(config.disabled_providers ?? [])
  const enabled = config.enabled_providers ? new Set(config.enabled_providers) : null

  const providers: Record<ProviderID, Info> = {}
  const languages = new Map<string, LanguageModelV2>()
  const modelLoaders: { [providerID: string]: CustomModelLoader } = {}
  const varsLoaders: { [providerID: string]: CustomVarsLoader } = {}
  const discoveryLoaders: { [providerID: string]: CustomDiscoverModels } = {}
  const sdk = new Map<string, SDK>()

  // ... initialization logic ...

  return {
    models: languages,
    providers,
    sdk,
    modelLoaders,
    varsLoaders,
  }
})
```

### State Initialization Flow

```
┌─────────────────────────────────────────────────────────────────────┐
│                    State Initialization                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  1. Load configuration                                              │
│     ├─ Config.get() → opencode.json settings                        │
│     └─ ModelsDev.get() → models.dev catalog                         │
│                                                                     │
│  2. Build provider database                                         │
│     └─ Convert ModelsDev providers to internal format               │
│                                                                     │
│  3. Load from environment variables                                 │
│     └─ Check each provider's env vars (ANTHROPIC_API_KEY, etc.)     │
│                                                                     │
│  4. Load from auth store                                            │
│     └─ Auth.all() → stored API keys                                 │
│                                                                     │
│  5. Run CUSTOM_LOADERS                                              │
│     └─ Provider-specific initialization                             │
│                                                                     │
│  6. Apply config overrides                                          │
│     └─ Merge user config into providers                             │
│                                                                     │
│  7. Filter and validate                                             │
│     ├─ Remove disabled providers                                    │
│     ├─ Remove deprecated/alpha models                               │
│     └─ Apply whitelist/blacklist                                    │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Key Functions

### `getProvider`

Returns provider information by ID:

```typescript
export async function getProvider(providerID: ProviderID) {
  return state().then((s) => s.providers[providerID])
}
```

### `getModel`

Returns model information with fuzzy matching for suggestions:

```typescript
export async function getModel(providerID: ProviderID, modelID: ModelID) {
  const s = await state()
  const provider = s.providers[providerID]
  if (!provider) {
    const availableProviders = Object.keys(s.providers)
    const matches = fuzzysort.go(providerID, availableProviders, { limit: 3, threshold: -10000 })
    const suggestions = matches.map((m) => m.target)
    throw new ModelNotFoundError({ providerID, modelID, suggestions })
  }

  const info = provider.models[modelID]
  if (!info) {
    const availableModels = Object.keys(provider.models)
    const matches = fuzzysort.go(modelID, availableModels, { limit: 3, threshold: -10000 })
    const suggestions = matches.map((m) => m.target)
    throw new ModelNotFoundError({ providerID, modelID, suggestions })
  }
  return info
}
```

### `getLanguage`

Returns the actual LanguageModelV2 instance:

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

### `list`

Returns all available providers:

```typescript
export async function list() {
  return state().then((state) => state.providers)
}
```

---

## The Copilot Provider (Custom Implementation)

GitHub Copilot uses a custom provider implementation in `packages/opencode/src/provider/sdk/copilot/`.

### Why Custom?

From `packages/opencode/src/provider/sdk/copilot/README.md`:

> This is a temporary package used primarily for GitHub Copilot compatibility.
> Avoid making changes to these files unless you only want to affect the Copilot provider.

### Structure

```
packages/opencode/src/provider/sdk/copilot/
├── index.ts                    # Exports
├── copilot-provider.ts         # Main provider factory
├── chat/                       # Chat completions API
│   ├── openai-compatible-chat-language-model.ts
│   ├── convert-to-openai-compatible-chat-messages.ts
│   └── ...
├── responses/                  # Responses API
│   ├── openai-responses-language-model.ts
│   ├── convert-to-openai-responses-input.ts
│   └── ...
└── ...
```

### Provider Factory

From `packages/opencode/src/provider/sdk/copilot/copilot-provider.ts`:

```typescript
export function createOpenaiCompatible(options: OpenaiCompatibleProviderSettings = {}): OpenaiCompatibleProvider {
  const baseURL = withoutTrailingSlash(options.baseURL ?? "https://api.openai.com/v1")

  const headers = {
    ...(options.apiKey && { Authorization: `Bearer ${options.apiKey}` }),
    ...options.headers,
  }

  const getHeaders = () => withUserAgentSuffix(headers, `ai-sdk/openai-compatible/${VERSION}`)

  const createChatModel = (modelId: OpenaiCompatibleModelId) => {
    return new OpenAICompatibleChatLanguageModel(modelId, {
      provider: `${options.name ?? "openai-compatible"}.chat`,
      headers: getHeaders,
      url: ({ path }) => `${baseURL}${path}`,
      fetch: options.fetch,
    })
  }

  const createResponsesModel = (modelId: OpenaiCompatibleModelId) => {
    return new OpenAIResponsesLanguageModel(modelId, {
      provider: `${options.name ?? "openai-compatible"}.responses`,
      headers: getHeaders,
      url: ({ path }) => `${baseURL}${path}`,
      fetch: options.fetch,
    })
  }

  const provider = function (modelId: OpenaiCompatibleModelId) {
    return createChatModel(modelId)
  }

  provider.languageModel = createChatModel
  provider.chat = createChatModel
  provider.responses = createResponsesModel

  return provider as OpenaiCompatibleProvider
}
```

---

## Provider Configuration

### Environment Variables

Each provider has associated environment variables:

| Provider | Environment Variable |
|----------|---------------------|
| Anthropic | `ANTHROPIC_API_KEY` |
| OpenAI | `OPENAI_API_KEY` |
| Google | `GOOGLE_GENERATIVE_AI_API_KEY` |
| Azure | `AZURE_OPENAI_API_KEY` |
| Bedrock | `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY` |
| OpenRouter | `OPENROUTER_API_KEY` |

### Config File (opencode.json)

```json
{
  "provider": {
    "anthropic": {
      "options": {
        "baseURL": "https://custom-endpoint.example.com"
      }
    },
    "openai": {
      "models": {
        "gpt-5-custom": {
          "id": "gpt-5-turbo",
          "name": "Custom GPT-5",
          "limit": {
            "context": 128000,
            "output": 4096
          }
        }
      }
    }
  },
  "disabled_providers": ["cohere"],
  "enabled_providers": ["anthropic", "openai"]
}
```

### Auth Store

API keys can be stored securely via the `opencode auth` command:

```bash
opencode auth anthropic
# Prompts for API key and stores it securely
```

---

## Model Information Schema

From `packages/opencode/src/provider/provider.ts`:

```typescript
export const Model = z.object({
  id: ModelID.zod,
  providerID: ProviderID.zod,
  api: z.object({
    id: z.string(),
    url: z.string(),
    npm: z.string(),
  }),
  name: z.string(),
  family: z.string().optional(),
  capabilities: z.object({
    temperature: z.boolean(),
    reasoning: z.boolean(),
    attachment: z.boolean(),
    toolcall: z.boolean(),
    input: z.object({
      text: z.boolean(),
      audio: z.boolean(),
      image: z.boolean(),
      video: z.boolean(),
      pdf: z.boolean(),
    }),
    output: z.object({
      text: z.boolean(),
      audio: z.boolean(),
      image: z.boolean(),
      video: z.boolean(),
      pdf: z.boolean(),
    }),
    interleaved: z.union([z.boolean(), z.object({
      field: z.enum(["reasoning_content", "reasoning_details"]),
    })]),
  }),
  cost: z.object({
    input: z.number(),
    output: z.number(),
    cache: z.object({
      read: z.number(),
      write: z.number(),
    }),
  }),
  limit: z.object({
    context: z.number(),
    input: z.number().optional(),
    output: z.number(),
  }),
  status: z.enum(["alpha", "beta", "deprecated", "active"]),
  options: z.record(z.string(), z.any()),
  headers: z.record(z.string(), z.string()),
  release_date: z.string(),
  variants: z.record(z.string(), z.record(z.string(), z.any())).optional(),
})
```

---

## Self-Check Questions

1. **What is the purpose of BUNDLED_PROVIDERS?**
   <details>
   <summary>Answer</summary>
   It maps npm package names to their factory functions, allowing opencode to create SDK instances for each provider without dynamic imports for bundled providers.
   </details>

2. **Why do some providers need CUSTOM_LOADERS?**
   <details>
   <summary>Answer</summary>
   Some providers need special initialization logic like custom headers (Anthropic beta features), different API methods (OpenAI responses vs languageModel), or complex credential handling (AWS Bedrock).
   </details>

3. **How does opencode determine which providers are available?**
   <details>
   <summary>Answer</summary>
   It checks environment variables, the auth store, and config file. A provider is available if it has valid credentials from any of these sources and isn't in the disabled list.
   </details>

4. **What happens when you request a model that doesn't exist?**
   <details>
   <summary>Answer</summary>
   The `getModel` function throws a `ModelNotFoundError` with fuzzy-matched suggestions for similar model names.
   </details>

5. **Why does GitHub Copilot have a custom provider implementation?**
   <details>
   <summary>Answer</summary>
   Copilot has specific requirements for authentication, API endpoints, and model selection (choosing between responses, chat, or languageModel APIs based on the model).
   </details>

---

## Exercises

### Exercise 1: List Available Providers

Write code to list all available providers and their models:

```typescript
import { Provider } from "./provider/provider"

const providers = await Provider.list()
for (const [id, info] of Object.entries(providers)) {
  console.log(`Provider: ${info.name} (${id})`)
  console.log(`  Models: ${Object.keys(info.models).join(", ")}`)
}
```

### Exercise 2: Trace Provider Loading

For the Anthropic provider, trace the complete loading path:
1. Where is the API key read from?
2. What custom loader options are applied?
3. How is the SDK instance created?

### Exercise 3: Add a Custom Provider

Sketch out what you would need to add a new provider called "my-llm":
1. What environment variable would you use?
2. Would you need a CUSTOM_LOADER?
3. What npm package would you use (or would you need a custom implementation)?

---

## Further Reading

- [models.dev](https://models.dev) - The model catalog opencode uses
- `packages/opencode/src/provider/models.ts` - Model catalog loading
- `packages/opencode/src/auth.ts` - API key storage
- `packages/opencode/src/config/config.ts` - Configuration loading
- `packages/opencode/src/provider/sdk/copilot/` - Custom provider example
