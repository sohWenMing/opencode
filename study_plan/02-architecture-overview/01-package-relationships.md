# Module 02 - Lesson 01: Package Relationships

## Learning Objectives

By the end of this lesson, you will be able to:
- Understand the monorepo structure and how packages are organized
- Identify the role of each package in the opencode ecosystem
- Explain how packages depend on each other using workspace references
- Recognize key external dependencies and their purposes
- Navigate the exports field and understand how modules are exposed

---

## The Monorepo Structure

opencode uses a **Bun workspaces** monorepo pattern. The root `package.json` defines which directories contain packages:

```json
"workspaces": {
  "packages": [
    "packages/*",
    "packages/console/*",
    "packages/sdk/js",
    "packages/slack"
  ]
}
```

This means any folder matching these patterns is treated as a separate npm package that can depend on other packages in the monorepo.

---

## Package Overview

```
packages/
├── opencode/          # Core CLI and server (the heart of the project)
├── app/               # Web/desktop UI application (SolidJS)
├── ui/                # Shared UI components library
├── plugin/            # Plugin SDK for extending opencode
├── sdk/               # JavaScript SDK for API clients
├── util/              # Shared utilities
├── script/            # Build and release scripts
├── desktop/           # Tauri desktop wrapper
├── desktop-electron/  # Electron desktop wrapper
├── web/               # Marketing website
├── storybook/         # UI component documentation
├── enterprise/        # Enterprise features
├── slack/             # Slack integration
├── function/          # Serverless functions
└── console/           # Cloud console (app + core)
```

---

## The Core Package: `packages/opencode`

This is the main package containing:
- The CLI entry point (`src/index.ts`)
- The HTTP server (`src/server/`)
- Session management (`src/session/`)
- LLM provider integrations (`src/provider/`)
- Tool implementations (`src/tool/`)
- Storage layer (`src/storage/`)

### Package Identity

```json
{
  "name": "opencode",
  "version": "1.3.2",
  "bin": {
    "opencode": "./bin/opencode"
  }
}
```

The `bin` field makes `opencode` available as a CLI command when installed globally.

---

## Workspace Dependencies

Packages reference each other using `workspace:*` syntax:

```json
// Root package.json
"dependencies": {
  "@opencode-ai/plugin": "workspace:*",
  "@opencode-ai/script": "workspace:*",
  "@opencode-ai/sdk": "workspace:*"
}
```

```json
// packages/opencode/package.json
"dependencies": {
  "@opencode-ai/plugin": "workspace:*",
  "@opencode-ai/script": "workspace:*",
  "@opencode-ai/sdk": "workspace:*",
  "@opencode-ai/util": "workspace:*"
}
```

The `workspace:*` protocol tells Bun to resolve these from the local monorepo rather than npm.

---

## Package Relationship Diagram

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           opencode Monorepo                              │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │                    packages/opencode (CORE)                       │   │
│  │  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐    │   │
│  │  │   cli   │ │ server  │ │ session │ │provider │ │  tool   │    │   │
│  │  └────┬────┘ └────┬────┘ └────┬────┘ └────┬────┘ └────┬────┘    │   │
│  │       │           │           │           │           │          │   │
│  │       └───────────┴───────────┴───────────┴───────────┘          │   │
│  │                              │                                    │   │
│  │                    ┌─────────┴─────────┐                         │   │
│  │                    │     storage       │                         │   │
│  │                    └───────────────────┘                         │   │
│  └──────────────────────────────┬───────────────────────────────────┘   │
│                                 │                                        │
│         ┌───────────────────────┼───────────────────────┐               │
│         │                       │                       │               │
│         ▼                       ▼                       ▼               │
│  ┌─────────────┐        ┌─────────────┐         ┌─────────────┐        │
│  │   @opencode │        │   @opencode │         │   @opencode │        │
│  │   -ai/util  │        │   -ai/plugin│         │   -ai/sdk   │        │
│  └─────────────┘        └─────────────┘         └─────────────┘        │
│         │                       │                       │               │
│         └───────────────────────┼───────────────────────┘               │
│                                 │                                        │
│                                 ▼                                        │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                        UI / Apps Layer                           │   │
│  │  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐   │   │
│  │  │   app   │ │   ui    │ │ desktop │ │   web   │ │ console │   │   │
│  │  └─────────┘ └─────────┘ └─────────┘ └─────────┘ └─────────┘   │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Key External Dependencies

The core package (`packages/opencode`) relies on several important external libraries:

### AI/LLM Integration

| Package | Purpose |
|---------|---------|
| `ai` (Vercel AI SDK) | Unified interface for streaming LLM responses |
| `@ai-sdk/anthropic` | Claude models |
| `@ai-sdk/openai` | OpenAI/GPT models |
| `@ai-sdk/google` | Gemini models |
| `@ai-sdk/amazon-bedrock` | AWS Bedrock models |
| `@openrouter/ai-sdk-provider` | OpenRouter aggregator |

### Server Framework

| Package | Purpose |
|---------|---------|
| `hono` | Fast, lightweight HTTP framework |
| `hono-openapi` | OpenAPI spec generation |
| `@hono/zod-validator` | Request validation |

### Database/Storage

| Package | Purpose |
|---------|---------|
| `drizzle-orm` | Type-safe SQL ORM |
| `drizzle-kit` | Database migrations |

### Effect System

| Package | Purpose |
|---------|---------|
| `effect` | Functional effect system for services |
| `@effect/platform-node` | Node.js platform bindings |

### UI Framework

| Package | Purpose |
|---------|---------|
| `solid-js` | Reactive UI framework (TUI) |
| `@opentui/solid` | Terminal UI components |

### Utilities

| Package | Purpose |
|---------|---------|
| `zod` | Runtime schema validation |
| `remeda` | Functional utilities |
| `yargs` | CLI argument parsing |

---

## The Exports Field

The `exports` field in `packages/opencode/package.json` controls how modules are exposed:

```json
"exports": {
  "./*": "./src/*.ts"
}
```

This wildcard pattern means any import like:
```typescript
import { Session } from "opencode/session"
```

Resolves to:
```
packages/opencode/src/session.ts
```

### Conditional Imports

The `imports` field handles platform-specific code:

```json
"imports": {
  "#db": {
    "bun": "./src/storage/db.bun.ts",
    "node": "./src/storage/db.node.ts",
    "default": "./src/storage/db.bun.ts"
  }
}
```

This allows code to import `#db` and get the correct implementation based on the runtime.

---

## The Dependency Catalog

The root `package.json` uses a **catalog** pattern to centralize version management:

```json
"workspaces": {
  "catalog": {
    "effect": "4.0.0-beta.35",
    "ai": "5.0.124",
    "hono": "4.10.7",
    "drizzle-orm": "1.0.0-beta.19-d95b7a4",
    "zod": "4.1.8",
    "solid-js": "1.9.10"
  }
}
```

Packages reference catalog versions with `catalog:`:

```json
"dependencies": {
  "effect": "catalog:",
  "ai": "catalog:",
  "hono": "catalog:"
}
```

This ensures all packages use the same version of shared dependencies.

---

## Internal Package Details

### `@opencode-ai/util`

Shared utilities used across packages:

```json
{
  "name": "@opencode-ai/util",
  "exports": {
    "./*": "./src/*.ts"
  },
  "dependencies": {
    "zod": "catalog:"
  }
}
```

Provides: error handling (`error.ts`), slugs (`slug.ts`), identifiers (`identifier.ts`)

### `@opencode-ai/plugin`

SDK for building opencode plugins:

```json
{
  "name": "@opencode-ai/plugin",
  "exports": {
    ".": "./src/index.ts",
    "./tool": "./src/tool.ts"
  },
  "dependencies": {
    "@opencode-ai/sdk": "workspace:*",
    "zod": "catalog:"
  }
}
```

### `@opencode-ai/script`

Build and release automation:

```json
{
  "name": "@opencode-ai/script",
  "dependencies": {
    "semver": "^7.6.3"
  }
}
```

---

## Self-Check Questions

1. **What does `workspace:*` mean in a dependency declaration?**
   <details>
   <summary>Answer</summary>
   It tells the package manager to resolve the dependency from the local monorepo workspace rather than from npm.
   </details>

2. **Which package contains the main CLI entry point?**
   <details>
   <summary>Answer</summary>
   `packages/opencode` - specifically `src/index.ts`
   </details>

3. **What is the purpose of the catalog in the root package.json?**
   <details>
   <summary>Answer</summary>
   It centralizes version management so all packages use the same versions of shared dependencies, preventing version conflicts.
   </details>

4. **How does the exports field `./*` pattern work?**
   <details>
   <summary>Answer</summary>
   It maps any import path to the corresponding file in `src/`. For example, `opencode/session` maps to `src/session.ts`.
   </details>

---

## Exercises

1. **Explore the workspace**: Run `ls packages/` and identify which packages are UI-related vs backend-related.

2. **Trace a dependency**: Find where `@opencode-ai/util` is used in `packages/opencode`. What utilities does it provide?

3. **Catalog lookup**: Find three dependencies that use `catalog:` and look up their actual versions in the root package.json.

---

## Further Reading

- [Bun Workspaces Documentation](https://bun.sh/docs/install/workspaces)
- [Package.json exports field](https://nodejs.org/api/packages.html#exports)
- [Vercel AI SDK](https://sdk.vercel.ai/docs)
- [Hono Framework](https://hono.dev/)
