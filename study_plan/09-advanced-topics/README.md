# Module 09: Advanced Topics

## Overview

This module covers advanced areas of the opencode codebase that extend beyond the core functionality. You'll learn about the plugin system, agent architecture, the Tauri desktop app, and the VS Code extension.

## Prerequisites

Before starting this module, you should have completed:
- Module 00: Foundations
- Module 06: Session System
- Module 07: Tool System

## Learning Path

```
┌─────────────────────────────────────────────────────────────────┐
│                    Module 09: Advanced Topics                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  Lesson 01: Plugin System                                │   │
│  │  • What plugins are and their purpose                    │   │
│  │  • Plugin hooks and lifecycle                            │   │
│  │  • Creating custom plugins                               │   │
│  │  • Built-in plugins (Codex, Copilot)                     │   │
│  └─────────────────────────────────────────────────────────┘   │
│                           │                                     │
│                           ▼                                     │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  Lesson 02: Agents and Tasks                             │   │
│  │  • Agent definitions and modes                           │   │
│  │  • Agent prompts and permissions                         │   │
│  │  • Task tool for subagent spawning                       │   │
│  │  • Custom agent configuration                            │   │
│  └─────────────────────────────────────────────────────────┘   │
│                           │                                     │
│                           ▼                                     │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  Lesson 03: Desktop App with Tauri                       │   │
│  │  • Tauri framework overview                              │   │
│  │  • Rust backend code                                     │   │
│  │  • Sidecar management                                    │   │
│  │  • Building the desktop app                              │   │
│  └─────────────────────────────────────────────────────────┘   │
│                           │                                     │
│                           ▼                                     │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  Lesson 04: VS Code Extension                            │   │
│  │  • Extension structure and manifest                      │   │
│  │  • Commands, keybindings, menus                          │   │
│  │  • Terminal integration                                  │   │
│  │  • Communication with opencode                           │   │
│  └─────────────────────────────────────────────────────────┘   │
│                           │                                     │
│                           ▼                                     │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  Exercises: Advanced Exploration                         │   │
│  │  • Plugin system exploration                             │   │
│  │  • Agent comparison and design                           │   │
│  │  • Desktop app analysis                                  │   │
│  │  • VS Code extension examination                         │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## Lessons

| Lesson | Topic | Key Concepts |
|--------|-------|--------------|
| [01](./01-plugin-system.md) | Plugin System | Hooks, lifecycle, custom tools, authentication |
| [02](./02-agents-and-tasks.md) | Agents and Tasks | Agent modes, permissions, prompts, subagents |
| [03](./03-desktop-tauri.md) | Desktop App (Tauri) | Rust, sidecar, commands, events |
| [04](./04-vscode-extension.md) | VS Code Extension | Commands, terminals, file references |

## Exercises

| Exercise | Focus Area |
|----------|------------|
| [01](./exercises/01-advanced-exploration.md) | Comprehensive exploration of all advanced topics |

## Key Files to Study

### Plugin System
- `packages/plugin/src/index.ts` - Plugin types and hooks
- `packages/plugin/src/tool.ts` - Tool creation utilities
- `packages/opencode/src/plugin/index.ts` - Plugin loading and triggering
- `packages/opencode/src/plugin/codex.ts` - Codex OAuth plugin
- `packages/opencode/src/plugin/copilot.ts` - Copilot authentication

### Agent System
- `packages/opencode/src/agent/agent.ts` - Agent definitions
- `packages/opencode/src/agent/prompt/*.txt` - Agent prompts

### Desktop App
- `packages/desktop/src-tauri/src/lib.rs` - Main Tauri library
- `packages/desktop/src-tauri/src/server.rs` - Sidecar management
- `packages/desktop/src-tauri/Cargo.toml` - Rust dependencies
- `packages/desktop/src-tauri/tauri.conf.json` - Tauri configuration
- `packages/desktop/src/index.tsx` - Frontend entry point

### VS Code Extension
- `sdks/vscode/src/extension.ts` - Extension code
- `sdks/vscode/package.json` - Extension manifest

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                    opencode Ecosystem                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                     Interfaces                            │  │
│  │                                                           │  │
│  │  ┌────────────┐  ┌────────────┐  ┌────────────┐         │  │
│  │  │  Desktop   │  │    Web     │  │    CLI     │         │  │
│  │  │  (Tauri)   │  │   (SPA)    │  │   (TUI)    │         │  │
│  │  └─────┬──────┘  └─────┬──────┘  └─────┬──────┘         │  │
│  │        │               │               │                 │  │
│  │        └───────────────┼───────────────┘                 │  │
│  │                        │                                 │  │
│  │                        ▼                                 │  │
│  │  ┌────────────────────────────────────────────────────┐ │  │
│  │  │              VS Code Extension                      │ │  │
│  │  │         (Terminal + HTTP API)                       │ │  │
│  │  └────────────────────────────────────────────────────┘ │  │
│  └──────────────────────────────────────────────────────────┘  │
│                           │                                     │
│                           ▼                                     │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                    opencode Core                          │  │
│  │                                                           │  │
│  │  ┌────────────┐  ┌────────────┐  ┌────────────┐         │  │
│  │  │   Agent    │  │  Session   │  │    Tool    │         │  │
│  │  │   System   │  │   System   │  │   System   │         │  │
│  │  └────────────┘  └────────────┘  └────────────┘         │  │
│  │                        │                                 │  │
│  │                        ▼                                 │  │
│  │  ┌────────────────────────────────────────────────────┐ │  │
│  │  │                 Plugin System                       │ │  │
│  │  │                                                     │ │  │
│  │  │  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐          │ │  │
│  │  │  │Codex │  │Copilot│  │GitLab│  │Custom│          │ │  │
│  │  │  └──────┘  └──────┘  └──────┘  └──────┘          │ │  │
│  │  └────────────────────────────────────────────────────┘ │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## Learning Outcomes

After completing this module, you will be able to:

1. **Plugin System**
   - Create custom plugins with tools and hooks
   - Understand the plugin lifecycle
   - Implement authentication plugins
   - Modify LLM requests and responses

2. **Agent System**
   - Configure custom agents
   - Understand agent permissions
   - Design agent prompts
   - Use the Task tool for parallel work

3. **Desktop App**
   - Understand Tauri architecture
   - Read and modify Rust code
   - Build the desktop app
   - Debug sidecar issues

4. **VS Code Extension**
   - Understand extension activation
   - Add custom commands
   - Integrate with terminals
   - Communicate with opencode

## Estimated Time

- Lesson 01: 2-3 hours
- Lesson 02: 2-3 hours
- Lesson 03: 3-4 hours
- Lesson 04: 1-2 hours
- Exercises: 4-6 hours

**Total: 12-18 hours**

## Next Steps

After completing this module, consider:

1. **Building a Plugin**: Create a plugin for your specific workflow
2. **Customizing Agents**: Design agents tailored to your projects
3. **Contributing**: Submit improvements to the desktop app or extension
4. **Exploring Further**: Dive into other modules for deeper understanding
