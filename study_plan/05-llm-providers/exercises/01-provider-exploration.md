# Exercise 01: Provider Exploration

## Overview

In this exercise, you'll explore how opencode's provider system works by tracing code paths, examining configurations, and understanding the streaming pipeline.

**Estimated Time:** 2-3 hours

**Prerequisites:**
- Completed lessons 01-04 of this module
- opencode repository cloned locally
- Basic understanding of TypeScript and async/await

---

## Exercise 1: List Available Providers

### Objective
Understand how providers are discovered and what information is available about each.

### Tasks

1. **Find the provider listing function**
   
   Open `packages/opencode/src/provider/provider.ts` and locate the `list()` function.
   
   Questions:
   - What does this function return?
   - How is the provider state initialized?

2. **Trace the state initialization**
   
   Find the `state` variable and trace through its initialization:
   
   ```typescript
   const state = Instance.state(async () => {
     // ... initialization
   })
   ```
   
   Document the order of operations:
   - [ ] Load configuration
   - [ ] Load models.dev catalog
   - [ ] Check environment variables
   - [ ] Load auth store
   - [ ] Run custom loaders
   - [ ] Apply config overrides
   - [ ] Filter disabled providers

3. **Write a provider summary**
   
   Create a markdown table listing 5 providers with:
   - Provider ID
   - npm package
   - Environment variable(s)
   - Whether it has a custom loader

   Example:
   | Provider | npm Package | Env Var | Custom Loader |
   |----------|-------------|---------|---------------|
   | anthropic | @ai-sdk/anthropic | ANTHROPIC_API_KEY | Yes |

---

## Exercise 2: Trace Provider Loading

### Objective
Follow the complete path from provider ID to usable LanguageModel.

### Tasks

1. **Start with getLanguage**
   
   In `packages/opencode/src/provider/provider.ts`, find `getLanguage()`:
   
   ```typescript
   export async function getLanguage(model: Model): Promise<LanguageModelV2>
   ```
   
   Trace what happens when called with:
   ```typescript
   const model = {
     id: "claude-sonnet-4-20250514",
     providerID: "anthropic",
     api: {
       id: "claude-sonnet-4-20250514",
       url: "https://api.anthropic.com",
       npm: "@ai-sdk/anthropic"
     }
   }
   ```

2. **Document the loading steps**
   
   Fill in this flowchart:
   
   ```
   getLanguage(model)
       │
       ├─ Check cache: s.models.has(key)?
       │   └─ If yes: return cached model
       │
       ├─ Get provider: s.providers[model.providerID]
       │
       ├─ Get SDK: await getSDK(model)
       │   └─ What does getSDK do?
       │       ├─ _______________
       │       ├─ _______________
       │       └─ _______________
       │
       ├─ Create language model:
       │   └─ Custom loader or sdk.languageModel()?
       │
       └─ Cache and return
   ```

3. **Find the Anthropic custom loader**
   
   Locate the Anthropic entry in `CUSTOM_LOADERS`:
   
   - What headers does it add?
   - What is `autoload` set to?
   - Does it have a custom `getModel` function?

---

## Exercise 3: Understand the Transform Pipeline

### Objective
Learn how messages are transformed before being sent to providers.

### Tasks

1. **Find where transforms are applied**
   
   In `packages/opencode/src/session/llm.ts`, locate where `ProviderTransform.message()` is called:
   
   ```typescript
   model: wrapLanguageModel({
     model: language,
     middleware: [
       {
         async transformParams(args) {
           // Find this code
         },
       },
     ],
   }),
   ```

2. **Trace the transform chain**
   
   In `packages/opencode/src/provider/transform.ts`, document what `message()` does:
   
   ```
   ProviderTransform.message(msgs, model, options)
       │
       ├─ unsupportedParts()
       │   └─ What does this do? _______________
       │
       ├─ normalizeMessages()
       │   └─ Provider-specific handling for:
       │       ├─ Anthropic: _______________
       │       ├─ Claude: _______________
       │       ├─ Mistral: _______________
       │       └─ Interleaved: _______________
       │
       ├─ applyCaching() (for Anthropic)
       │   └─ What messages get cache control? _______________
       │
       └─ Remap providerOptions keys
   ```

3. **Test case analysis**
   
   Given this message array:
   ```typescript
   const messages = [
     { role: "system", content: "You are helpful." },
     { role: "user", content: [
       { type: "text", text: "Hello" },
       { type: "image", image: "data:image/png;base64,..." }
     ]},
     { role: "assistant", content: "" },
     { role: "tool", content: [
       { type: "tool-result", toolCallId: "call_ABC123!", result: "Done" }
     ]}
   ]
   ```
   
   What would the output be for:
   - **Anthropic**: (consider empty content, tool ID format, caching)
   - **Mistral**: (consider tool ID length, message sequence)
   - **Google**: (consider image support)

---

## Exercise 4: Examine Streaming Events

### Objective
Understand the different event types in a streaming response.

### Tasks

1. **Find the StreamTextResult type**
   
   The `LLM.stream()` function returns `StreamTextResult<ToolSet, unknown>`.
   
   Look at the AI SDK documentation or type definitions to list all possible event types in `fullStream`:
   
   | Event Type | Description | Key Properties |
   |------------|-------------|----------------|
   | text-delta | | |
   | reasoning | | |
   | tool-call-streaming-start | | |
   | tool-call-delta | | |
   | tool-call | | |
   | tool-result | | |
   | finish | | |
   | error | | |

2. **Trace a tool call flow**
   
   Document the event sequence when the model calls a tool:
   
   ```
   1. text-delta: "Let me read that file"
   2. _______________
   3. _______________
   4. _______________
   5. _______________
   6. finish (reason: "tool-calls")
   ```

3. **Find the tool repair logic**
   
   In `packages/opencode/src/session/llm.ts`, find `experimental_repairToolCall`:
   
   - What happens if the model calls "READ" instead of "read"?
   - What happens if the model calls "nonexistent_tool"?
   - Why is there an "invalid" tool?

---

## Exercise 5: Provider-Specific Options

### Objective
Understand how different providers require different options.

### Tasks

1. **Compare reasoning variants**
   
   In `packages/opencode/src/provider/transform.ts`, find the `variants()` function.
   
   Compare the "high" variant for these providers:
   
   | Provider | npm Package | "high" Variant Options |
   |----------|-------------|----------------------|
   | OpenAI | @ai-sdk/openai | |
   | Anthropic | @ai-sdk/anthropic | |
   | Google | @ai-sdk/google | |
   | Bedrock | @ai-sdk/amazon-bedrock | |

2. **Find temperature defaults**
   
   In the `temperature()` function, document the default temperatures:
   
   | Model Pattern | Default Temperature |
   |---------------|-------------------|
   | qwen | |
   | claude | |
   | gemini | |
   | kimi-k2 | |
   | kimi-k2-thinking | |

3. **Understand providerOptions wrapping**
   
   The `providerOptions()` function wraps options in a provider-specific namespace.
   
   Given these options:
   ```typescript
   const options = {
     reasoningEffort: "high",
     store: false
   }
   ```
   
   What would `providerOptions(model, options)` return for:
   - OpenAI model?
   - Anthropic model?
   - Gateway model with ID "anthropic/claude-sonnet-4"?

---

## Exercise 6: Custom Provider Deep Dive

### Objective
Understand how the GitHub Copilot custom provider works.

### Tasks

1. **Explore the copilot directory**
   
   List all files in `packages/opencode/src/provider/sdk/copilot/`:
   
   ```
   copilot/
   ├─ index.ts
   ├─ copilot-provider.ts
   ├─ chat/
   │   ├─ _______________
   │   └─ _______________
   └─ responses/
       ├─ _______________
       └─ _______________
   ```

2. **Understand the provider factory**
   
   In `copilot-provider.ts`, find `createOpenaiCompatible()`:
   
   - What three model creation methods does it provide?
   - When is `chat` used vs `responses`?
   - How are headers constructed?

3. **Find the custom loader**
   
   In `packages/opencode/src/provider/provider.ts`, find the `github-copilot` custom loader:
   
   ```typescript
   "github-copilot": async () => {
     // What does this return?
   }
   ```
   
   - What does `useLanguageModel(sdk)` check?
   - What does `shouldUseCopilotResponsesApi(modelID)` do?

---

## Bonus Challenge: Add a Mock Provider

### Objective
Create a simple mock provider for testing.

### Tasks

1. **Design the mock provider**
   
   Create a plan for a mock provider that:
   - Always returns "Hello from mock!"
   - Supports a single tool called "echo"
   - Has configurable delay

2. **Identify integration points**
   
   List the files you would need to modify:
   - [ ] `packages/opencode/src/provider/provider.ts` - Add to BUNDLED_PROVIDERS?
   - [ ] Create new files in `packages/opencode/src/provider/sdk/mock/`?
   - [ ] Update models.dev or config?

3. **Sketch the implementation**
   
   Write pseudocode for the mock provider factory:
   
   ```typescript
   export function createMockProvider(options: MockProviderOptions) {
     // Return an object with languageModel() method
     // that returns a LanguageModelV2 implementation
   }
   ```

---

## Submission Checklist

- [ ] Exercise 1: Provider summary table completed
- [ ] Exercise 2: Loading flowchart filled in
- [ ] Exercise 3: Transform analysis for all three providers
- [ ] Exercise 4: Event type table and tool call sequence
- [ ] Exercise 5: Variant comparison and temperature defaults
- [ ] Exercise 6: Copilot provider structure documented
- [ ] Bonus: Mock provider design (optional)

---

## Self-Assessment

After completing these exercises, you should be able to answer:

1. How does opencode discover which providers are available?
2. What's the difference between BUNDLED_PROVIDERS and CUSTOM_LOADERS?
3. Why do different providers need different message transforms?
4. What events does a streaming response emit?
5. How does the Copilot provider differ from standard providers?

---

## Further Exploration

If you want to go deeper:

1. **Add a new provider**: Try adding support for a new LLM provider
2. **Custom transforms**: Write a transform for a hypothetical provider with unique requirements
3. **Streaming analysis**: Build a tool that visualizes streaming events in real-time
4. **Cost tracking**: Implement token cost calculation using the model's cost data
