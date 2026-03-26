# OpenCode Study Plan

A comprehensive, mastery-level study plan for understanding the opencode repository from the ground up.

## Overview

This study plan is designed for someone with basic programming knowledge who wants to deeply understand how opencode works - from installation to LLM integration to the full agent architecture.

**Estimated Timeline:** 4-8 weeks (focused study)

**Format:** Each lesson includes:
- Concept introduction
- Code walkthrough with annotations
- Diagrams where helpful
- Hands-on exercises
- Self-check questions

## Prerequisites

Before starting, ensure you have:
- Node.js 18+ (for some tooling)
- Bun 1.3+ (`curl -fsSL https://bun.sh/install | bash`)
- Git
- A code editor (VS Code recommended)
- SQLite viewer (optional, for database inspection)

## Module Overview

### Weeks 1-2: Foundations

| Module | Description | Key Concepts |
|--------|-------------|--------------|
| [00-foundations](./00-foundations/) | TypeScript, Bun, monorepo basics | Types, async/await, package management |
| [01-installation-deep-dive](./01-installation-deep-dive/) | How opencode gets installed | Binary resolution, platform detection |
| [02-architecture-overview](./02-architecture-overview/) | High-level system design | Package relationships, data flow |

### Weeks 3-4: Core Systems

| Module | Description | Key Concepts |
|--------|-------------|--------------|
| [03-cli-and-tui](./03-cli-and-tui/) | User interaction layer | Yargs, Solid, OpenTUI |
| [04-server-layer](./04-server-layer/) | HTTP API and routing | Hono, REST, WebSocket |
| [05-llm-providers](./05-llm-providers/) | Model integration | Vercel AI SDK, streaming |

### Weeks 5-6: Agent Mechanics

| Module | Description | Key Concepts |
|--------|-------------|--------------|
| [06-session-system](./06-session-system/) | The agent loop (heart of the project) | Messages, prompts, processing |
| [07-tool-system](./07-tool-system/) | AI capabilities | Tool abstraction, MCP, permissions |

### Weeks 7-8: Advanced & Capstone

| Module | Description | Key Concepts |
|--------|-------------|--------------|
| [08-storage-layer](./08-storage-layer/) | Persistence | Drizzle ORM, SQLite, migrations |
| [09-advanced-topics](./09-advanced-topics/) | Deep dives | Plugins, Tauri, VS Code extension |
| [10-capstone](./10-capstone/) | Integration project | Contributing, extending opencode |

## Repository Structure Quick Reference

```
opencode/
├── packages/
│   ├── opencode/          # Core CLI and server (main focus)
│   │   ├── src/
│   │   │   ├── cli/       # Command-line interface
│   │   │   ├── server/    # HTTP API
│   │   │   ├── session/   # Chat sessions & LLM calls
│   │   │   ├── provider/  # LLM provider integrations
│   │   │   ├── tool/      # AI tools (bash, read, write, etc.)
│   │   │   └── storage/   # Database layer
│   │   └── bin/opencode   # Installation script
│   ├── app/               # Web application
│   ├── web/               # Documentation site
│   ├── desktop/           # Tauri desktop app
│   └── console/           # Cloud console
├── sdks/
│   └── vscode/            # VS Code extension
└── infra/                 # Cloud infrastructure
```

## How to Use This Study Plan

1. **Follow the modules in order** - Each builds on previous knowledge
2. **Read the actual code** - Lessons reference specific files; open them alongside
3. **Do the exercises** - Hands-on practice solidifies understanding
4. **Take notes** - Document your own insights and questions
5. **Experiment** - Make small changes to see what happens (use git to revert)

## Key Files You'll Study

| Area | Primary Files |
|------|---------------|
| Installation | `packages/opencode/bin/opencode` |
| LLM Integration | `packages/opencode/src/provider/provider.ts`, `src/session/llm.ts` |
| Agent Loop | `packages/opencode/src/session/prompt.ts`, `src/session/processor.ts` |
| Tools | `packages/opencode/src/tool/tool.ts`, `src/tool/registry.ts` |
| Storage | `packages/opencode/src/storage/`, `src/session/session.sql.ts` |

## Getting Help

- **Google/LLMs:** For language concepts (TypeScript, async/await, etc.)
- **Official Docs:** Links provided in each lesson
- **The Code Itself:** Often the best documentation

Let's begin with [Module 00: Foundations](./00-foundations/)!
