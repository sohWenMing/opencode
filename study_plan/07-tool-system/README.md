# Module 07: Tool System

## Overview

This module explains how the AI executes actions in opencode. Tools are the capabilities that allow the AI to read files, run commands, search code, and interact with external services.

## Learning Path

```
┌─────────────────────────────────────────────────────────────────────┐
│                     Module 07: Tool System                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌──────────────────┐                                               │
│  │ 01-tool-         │  Understanding the Tool.define abstraction    │
│  │ abstraction.md   │  Parameters, context, results, truncation     │
│  └────────┬─────────┘                                               │
│           │                                                          │
│           ▼                                                          │
│  ┌──────────────────┐                                               │
│  │ 02-built-in-     │  Deep dive into bash, read, write, edit,     │
│  │ tools.md         │  glob, grep, task, and other tools           │
│  └────────┬─────────┘                                               │
│           │                                                          │
│           ▼                                                          │
│  ┌──────────────────┐                                               │
│  │ 03-mcp-          │  Model Context Protocol integration          │
│  │ integration.md   │  External tools, resources, and prompts      │
│  └────────┬─────────┘                                               │
│           │                                                          │
│           ▼                                                          │
│  ┌──────────────────┐                                               │
│  │ 04-permissions.  │  Safety system for tool execution            │
│  │ md               │  Rules, evaluation, user approval            │
│  └────────┬─────────┘                                               │
│           │                                                          │
│           ▼                                                          │
│  ┌──────────────────┐                                               │
│  │ exercises/       │  Hands-on exploration of the tool system     │
│  │ 01-tool-         │  Trace calls, analyze permissions,           │
│  │ exploration.md   │  create custom tools                         │
│  └──────────────────┘                                               │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

## Prerequisites

Before starting this module, you should have completed:
- Module 00: Foundations (TypeScript, Bun, monorepo)
- Module 02: Architecture Overview (package relationships, data flow)
- Module 05: LLM Providers (understanding of AI SDK integration)

## Key Concepts

### Tools
Capabilities that allow the AI to interact with the outside world:
- **File Operations**: read, write, edit, glob
- **Code Search**: grep, codesearch
- **Execution**: bash, task
- **Web**: webfetch, websearch

### Tool.define
The abstraction for creating tools with:
- Parameter validation (Zod schemas)
- Permission integration
- Automatic output truncation
- Consistent error handling

### MCP (Model Context Protocol)
An open standard for connecting AI to external tools:
- Local MCP servers (stdio transport)
- Remote MCP servers (HTTP/SSE transport)
- OAuth authentication support

### Permissions
Safety mechanism controlling what tools can do:
- Rule-based evaluation
- User approval workflow
- Session-scoped approvals

## Key Files

| File | Purpose |
|------|---------|
| `packages/opencode/src/tool/tool.ts` | Core tool abstraction |
| `packages/opencode/src/tool/registry.ts` | Tool registration and management |
| `packages/opencode/src/tool/bash.ts` | Shell command execution |
| `packages/opencode/src/tool/read.ts` | File reading |
| `packages/opencode/src/tool/edit.ts` | File editing (str_replace) |
| `packages/opencode/src/mcp/index.ts` | MCP integration |
| `packages/opencode/src/permission/index.ts` | Permission system |
| `packages/opencode/src/permission/evaluate.ts` | Rule evaluation |

## Estimated Time

- Lesson 01: 45 minutes
- Lesson 02: 60 minutes
- Lesson 03: 45 minutes
- Lesson 04: 45 minutes
- Exercises: 90-120 minutes

**Total: ~5-6 hours**

## Learning Objectives

After completing this module, you will be able to:

1. **Understand Tool Architecture**
   - Explain how tools are defined and registered
   - Describe the tool execution lifecycle
   - Understand parameter validation and result handling

2. **Navigate Built-in Tools**
   - Identify all built-in tools and their purposes
   - Understand how key tools (bash, read, edit, grep) work
   - Trace tool calls through the codebase

3. **Work with MCP**
   - Configure MCP servers (local and remote)
   - Understand how MCP tools integrate with built-in tools
   - Handle MCP authentication

4. **Understand Permissions**
   - Explain the permission rule system
   - Trace permission evaluation
   - Configure permission rules

## Quick Reference

### Tool Definition Pattern

```typescript
export const MyTool = Tool.define("mytool", {
  description: "What this tool does...",
  
  parameters: z.object({
    param1: z.string().describe("Description for AI"),
    param2: z.number().optional(),
  }),
  
  async execute(params, ctx) {
    // Request permission
    await ctx.ask({
      permission: "mytool",
      patterns: [params.param1],
      always: ["*"],
      metadata: {},
    })
    
    // Do the work
    const result = await doSomething(params)
    
    // Return result
    return {
      title: "Short title",
      metadata: { /* structured data */ },
      output: "Text for the AI",
    }
  },
})
```

### Permission Rule Pattern

```typescript
const rules = [
  { permission: "bash", pattern: "*", action: "ask" },
  { permission: "bash", pattern: "npm *", action: "allow" },
  { permission: "bash", pattern: "rm *", action: "deny" },
]
```

### MCP Configuration Pattern

```json
{
  "mcp": {
    "my-server": {
      "type": "local",
      "command": ["npx", "my-mcp-server"],
      "timeout": 30000
    }
  }
}
```

## Next Steps

After completing this module, continue to:
- **Module 08**: Session Management (how conversations are stored and managed)
- **Module 09**: Agent System (how different agent types work)
