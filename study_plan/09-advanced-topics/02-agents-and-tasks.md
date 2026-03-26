# Module 09, Lesson 02: Agents and Tasks

## Learning Objectives

By the end of this lesson, you will be able to:

1. Understand what agents are in opencode (different modes/personas)
2. Navigate the agent folder and understand agent definitions
3. Explain agent prompts and permissions
4. Understand the task tool for spawning subagents
5. Differentiate between agent types (build, plan, explore, etc.)

## What Are Agents?

Agents in opencode are **specialized personas** that define how the AI behaves in different contexts. Each agent has:

- **Name**: Unique identifier (e.g., "build", "plan", "explore")
- **Description**: What the agent does
- **Mode**: Whether it's a primary agent or subagent
- **Permissions**: What tools and actions it can perform
- **Prompt**: Optional system prompt customization
- **Model**: Optional specific model override

```
┌─────────────────────────────────────────────────────────────────┐
│                       Agent Architecture                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                    PRIMARY AGENTS                        │   │
│  │  (User-facing, can be selected as default)               │   │
│  │                                                          │   │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐              │   │
│  │  │  build   │  │   plan   │  │  custom  │              │   │
│  │  │ (default)│  │ (read-   │  │  agents  │              │   │
│  │  │          │  │  only)   │  │          │              │   │
│  │  └──────────┘  └──────────┘  └──────────┘              │   │
│  └─────────────────────────────────────────────────────────┘   │
│                           │                                     │
│                           │ spawn via Task tool                 │
│                           ▼                                     │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                      SUBAGENTS                           │   │
│  │  (Spawned by primary agents for specific tasks)          │   │
│  │                                                          │   │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐              │   │
│  │  │ general  │  │ explore  │  │  custom  │              │   │
│  │  │ (multi-  │  │ (code-   │  │ subagents│              │   │
│  │  │  step)   │  │  base)   │  │          │              │   │
│  │  └──────────┘  └──────────┘  └──────────┘              │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                    HIDDEN AGENTS                         │   │
│  │  (Internal use only, not user-selectable)                │   │
│  │                                                          │   │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐              │   │
│  │  │compaction│  │  title   │  │ summary  │              │   │
│  │  │(context) │  │(generate)│  │(session) │              │   │
│  │  └──────────┘  └──────────┘  └──────────┘              │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## Agent Folder Structure

```
packages/opencode/src/agent/
├── agent.ts                    # Main agent definitions and service
└── prompt/
    ├── compaction.txt          # Context compaction prompt
    ├── explore.txt             # Explore agent system prompt
    ├── summary.txt             # Session summary prompt
    └── title.txt               # Title generation prompt
```

## Agent Schema Definition

```typescript
// From packages/opencode/src/agent/agent.ts

export const Info = z.object({
  name: z.string(),
  description: z.string().optional(),
  mode: z.enum(["subagent", "primary", "all"]),
  native: z.boolean().optional(),        // Built-in vs custom
  hidden: z.boolean().optional(),        // Hide from user selection
  topP: z.number().optional(),
  temperature: z.number().optional(),
  color: z.string().optional(),
  permission: Permission.Ruleset,        // What the agent can do
  model: z.object({                      // Optional model override
    modelID: ModelID.zod,
    providerID: ProviderID.zod,
  }).optional(),
  variant: z.string().optional(),
  prompt: z.string().optional(),         // Custom system prompt
  options: z.record(z.string(), z.any()),
  steps: z.number().int().positive().optional(),
})
```

## Built-in Agent Definitions

### 1. Build Agent (Default Primary)

The default agent for executing tasks with full tool access:

```typescript
// From packages/opencode/src/agent/agent.ts

build: {
  name: "build",
  description: "The default agent. Executes tools based on configured permissions.",
  options: {},
  permission: Permission.merge(
    defaults,
    Permission.fromConfig({
      question: "allow",      // Can ask questions
      plan_enter: "allow",    // Can switch to plan mode
    }),
    user,
  ),
  mode: "primary",
  native: true,
}
```

### 2. Plan Agent (Read-Only Primary)

A read-only mode for planning without making changes:

```typescript
plan: {
  name: "plan",
  description: "Plan mode. Disallows all edit tools.",
  options: {},
  permission: Permission.merge(
    defaults,
    Permission.fromConfig({
      question: "allow",
      plan_exit: "allow",     // Can exit plan mode
      external_directory: {
        [path.join(Global.Path.data, "plans", "*")]: "allow",
      },
      edit: {
        "*": "deny",          // No editing allowed!
        [path.join(".opencode", "plans", "*.md")]: "allow",  // Except plan files
      },
    }),
    user,
  ),
  mode: "primary",
  native: true,
}
```

### 3. General Subagent

For multi-step tasks executed in parallel:

```typescript
general: {
  name: "general",
  description: `General-purpose agent for researching complex questions 
    and executing multi-step tasks. Use this agent to execute multiple 
    units of work in parallel.`,
  permission: Permission.merge(
    defaults,
    Permission.fromConfig({
      todowrite: "deny",      // Can't manage todos (parent does that)
    }),
    user,
  ),
  options: {},
  mode: "subagent",
  native: true,
}
```

### 4. Explore Subagent

Specialized for codebase exploration with limited tools:

```typescript
explore: {
  name: "explore",
  permission: Permission.merge(
    defaults,
    Permission.fromConfig({
      "*": "deny",            // Deny everything by default
      grep: "allow",          // Allow search tools
      glob: "allow",
      list: "allow",
      bash: "allow",
      webfetch: "allow",
      websearch: "allow",
      codesearch: "allow",
      read: "allow",          // Allow reading files
      external_directory: {
        "*": "ask",
        ...whitelistedDirs,
      },
    }),
    user,
  ),
  description: `Fast agent specialized for exploring codebases. Use this 
    when you need to quickly find files by patterns (eg. "src/components/**/*.tsx"), 
    search code for keywords (eg. "API endpoints"), or answer questions about 
    the codebase (eg. "how do API endpoints work?"). When calling this agent, 
    specify the desired thoroughness level: "quick" for basic searches, 
    "medium" for moderate exploration, or "very thorough" for comprehensive 
    analysis across multiple locations and naming conventions.`,
  prompt: PROMPT_EXPLORE,
  options: {},
  mode: "subagent",
  native: true,
}
```

### 5. Hidden Utility Agents

These agents are used internally and hidden from users:

```typescript
// Compaction agent - summarizes context when it gets too long
compaction: {
  name: "compaction",
  mode: "primary",
  native: true,
  hidden: true,
  prompt: PROMPT_COMPACTION,
  permission: Permission.merge(
    defaults,
    Permission.fromConfig({ "*": "deny" }),  // No tools allowed
    user,
  ),
  options: {},
}

// Title agent - generates session titles
title: {
  name: "title",
  mode: "primary",
  options: {},
  native: true,
  hidden: true,
  temperature: 0.5,
  permission: Permission.merge(
    defaults,
    Permission.fromConfig({ "*": "deny" }),
    user,
  ),
  prompt: PROMPT_TITLE,
}

// Summary agent - summarizes sessions
summary: {
  name: "summary",
  mode: "primary",
  options: {},
  native: true,
  hidden: true,
  permission: Permission.merge(
    defaults,
    Permission.fromConfig({ "*": "deny" }),
    user,
  ),
  prompt: PROMPT_SUMMARY,
}
```

## Agent Prompts

### Explore Agent Prompt

```text
// From packages/opencode/src/agent/prompt/explore.txt

You are a file search specialist. You excel at thoroughly navigating 
and exploring codebases.

Your strengths:
- Rapidly finding files using glob patterns
- Searching code and text with powerful regex patterns
- Reading and analyzing file contents

Guidelines:
- Use Glob for broad file pattern matching
- Use Grep for searching file contents with regex
- Use Read when you know the specific file path you need to read
- Use Bash for file operations like copying, moving, or listing directory contents
- Adapt your search approach based on the thoroughness level specified by the caller
- Return file paths as absolute paths in your final response
- For clear communication, avoid using emojis
- Do not create any files, or run bash commands that modify the user's system state

Complete the user's search request efficiently and report your findings clearly.
```

### Compaction Agent Prompt

```text
// From packages/opencode/src/agent/prompt/compaction.txt

You are a helpful AI assistant tasked with summarizing conversations.

When asked to summarize, provide a detailed but concise summary of the conversation.
Focus on information that would be helpful for continuing the conversation, including:
- What was done
- What is currently being worked on
- Which files are being modified
- What needs to be done next
- Key user requests, constraints, or preferences that should persist
- Important technical decisions and why they were made

Your summary should be comprehensive enough to provide context but concise 
enough to be quickly understood.

Do not respond to any questions in the conversation, only output the summary.
```

### Title Agent Prompt

```text
// From packages/opencode/src/agent/prompt/title.txt

You are a title generator. You output ONLY a thread title. Nothing else.

<task>
Generate a brief title that would help the user find this conversation later.

Follow all rules in <rules>
Your output must be:
- A single line
- ≤50 characters
- No explanations
</task>

<rules>
- you MUST use the same language as the user message you are summarizing
- Title must be grammatically correct and read naturally
- Never include tool names in the title
- Focus on the main topic or question the user needs to retrieve
- Keep exact: technical terms, numbers, filenames, HTTP codes
- Remove: the, this, my, a, an
- Never assume tech stack
- NEVER respond to questions, just generate a title
</rules>
```

## Permission System

Each agent has a permission ruleset that controls what it can do:

```typescript
// Default permissions for all agents
const defaults = Permission.fromConfig({
  "*": "allow",                    // Allow most things
  doom_loop: "ask",                // Ask before potential loops
  external_directory: {
    "*": "ask",                    // Ask before accessing external dirs
    ...whitelistedDirs,            // Except whitelisted ones
  },
  question: "deny",                // Don't ask questions by default
  plan_enter: "deny",              // Don't switch modes by default
  plan_exit: "deny",
  read: {
    "*": "allow",
    "*.env": "ask",                // Ask before reading .env files
    "*.env.*": "ask",
    "*.env.example": "allow",      // But .env.example is fine
  },
})
```

## The Task Tool: Spawning Subagents

The Task tool allows agents to spawn subagents for parallel work:

```
┌─────────────────────────────────────────────────────────────────┐
│                    Task Tool Flow                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Primary Agent (build)                                          │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  User: "Refactor the auth module and update tests"      │   │
│  │                                                          │   │
│  │  Agent thinks: "This is complex, I'll use subagents"    │   │
│  │                                                          │   │
│  │  ┌─────────────────────────────────────────────────┐    │   │
│  │  │ Task(explore, "Find all auth-related files")    │    │   │
│  │  │ Task(explore, "Find all test files for auth")   │    │   │
│  │  └─────────────────────────────────────────────────┘    │   │
│  └─────────────────────────────────────────────────────────┘   │
│                           │                                     │
│            ┌──────────────┴──────────────┐                     │
│            ▼                              ▼                     │
│  ┌──────────────────┐          ┌──────────────────┐            │
│  │ Explore Subagent │          │ Explore Subagent │            │
│  │ (auth files)     │          │ (test files)     │            │
│  │                  │          │                  │            │
│  │ Uses: Glob, Grep │          │ Uses: Glob, Grep │            │
│  │       Read       │          │       Read       │            │
│  └────────┬─────────┘          └────────┬─────────┘            │
│           │                              │                      │
│           └──────────────┬───────────────┘                     │
│                          ▼                                      │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  Primary Agent receives results, continues work         │   │
│  │                                                          │   │
│  │  ┌─────────────────────────────────────────────────┐    │   │
│  │  │ Task(general, "Refactor auth module")           │    │   │
│  │  │ Task(general, "Update auth tests")              │    │   │
│  │  └─────────────────────────────────────────────────┘    │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Task Tool Parameters

```typescript
// Task tool invocation
{
  subagent_type: "explore" | "generalPurpose" | "shell" | "browser-use" | "best-of-n-runner",
  prompt: "Detailed task description",
  description: "Short 3-5 word summary",
  model?: "fast",           // Optional model selection
  readonly?: boolean,       // Run in read-only mode
  resume?: string,          // Resume existing agent
  run_in_background?: boolean,
}
```

## Agent Generation

opencode can generate new agent configurations from descriptions:

```typescript
// From packages/opencode/src/agent/agent.ts

generate: Effect.fn("Agent.generate")(function* (input: {
  description: string
  model?: { providerID: ProviderID; modelID: ModelID }
}) {
  const system = [PROMPT_GENERATE]  // Agent generation prompt
  
  const params = {
    temperature: 0.3,
    messages: [
      ...system.map((item) => ({ role: "system", content: item })),
      {
        role: "user",
        content: `Create an agent configuration based on this request: 
          "${input.description}".
          
          IMPORTANT: The following identifiers already exist and must NOT 
          be used: ${existing.map((i) => i.name).join(", ")}
          
          Return ONLY the JSON object, no other text`,
      },
    ],
    schema: z.object({
      identifier: z.string(),
      whenToUse: z.string(),
      systemPrompt: z.string(),
    }),
  }

  return yield* Effect.promise(() => generateObject(params).then((r) => r.object))
})
```

## Custom Agent Configuration

Users can define custom agents in `opencode.json`:

```json
{
  "agent": {
    "code-reviewer": {
      "name": "Code Reviewer",
      "description": "Reviews code for best practices and bugs",
      "mode": "subagent",
      "prompt": "You are an expert code reviewer...",
      "temperature": 0.3,
      "permission": {
        "*": "deny",
        "read": "allow",
        "grep": "allow",
        "glob": "allow"
      }
    },
    "build": {
      "disable": true  // Disable built-in agent
    }
  },
  "default_agent": "code-reviewer"
}
```

### Configuration Options

| Option | Type | Description |
|--------|------|-------------|
| `name` | string | Display name |
| `description` | string | What the agent does |
| `mode` | "primary" \| "subagent" \| "all" | Agent mode |
| `prompt` | string | Custom system prompt |
| `temperature` | number | LLM temperature |
| `top_p` | number | LLM top_p |
| `model` | string | Model override (e.g., "anthropic/claude-3-opus") |
| `variant` | string | Model variant |
| `color` | string | UI color |
| `hidden` | boolean | Hide from selection |
| `steps` | number | Max steps |
| `permission` | object | Permission overrides |
| `options` | object | Additional options |
| `disable` | boolean | Disable this agent |

## Agent Service API

```typescript
// From packages/opencode/src/agent/agent.ts

export namespace Agent {
  // Get a specific agent by name
  export async function get(agent: string): Promise<Agent.Info>
  
  // List all available agents
  export async function list(): Promise<Agent.Info[]>
  
  // Get the default agent name
  export async function defaultAgent(): Promise<string>
  
  // Generate a new agent configuration
  export async function generate(input: {
    description: string
    model?: { providerID: ProviderID; modelID: ModelID }
  }): Promise<{
    identifier: string
    whenToUse: string
    systemPrompt: string
  }>
}
```

## Self-Check Questions

1. What is the difference between "primary" and "subagent" modes?
2. Why does the explore agent have restricted permissions?
3. How do hidden agents differ from regular agents?
4. What is the purpose of the compaction agent?
5. How can users customize agent behavior through configuration?

## Exercises

### Exercise 1: Compare Agent Permissions

1. List all built-in agents and their permission sets
2. Create a table comparing what each agent can and cannot do
3. Identify why certain permissions are restricted for certain agents

### Exercise 2: Create a Custom Agent

1. Design an agent for a specific use case (e.g., documentation writer)
2. Define appropriate permissions
3. Write a system prompt
4. Add it to your `opencode.json` and test it

### Exercise 3: Trace Subagent Spawning

1. Start opencode with verbose logging
2. Give it a complex task that requires subagents
3. Document the subagent spawning sequence
4. Analyze how results flow back to the parent agent

## Further Reading

- [Effect-TS InstanceState](https://effect.website/) - How agent state is managed
- [Zod Schema Validation](https://zod.dev/) - Agent schema definitions
- [Permission System](../07-tool-system/04-permissions.md) - Deep dive into permissions
