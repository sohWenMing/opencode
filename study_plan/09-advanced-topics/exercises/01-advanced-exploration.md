# Module 09 Exercises: Advanced Topics Exploration

## Overview

These exercises will help you gain hands-on experience with opencode's advanced features: the plugin system, agents, the desktop app, and the VS Code extension.

---

## Exercise 1: Explore the Plugin System

### Objective
Understand how plugins are loaded, initialized, and how they extend opencode functionality.

### Tasks

#### 1.1 Plugin Loading Analysis

1. Open `packages/opencode/src/plugin/index.ts`
2. Find the `INTERNAL_PLUGINS` array and list all built-in plugins
3. Trace the plugin loading sequence by answering:
   - What order are internal plugins loaded?
   - How are npm plugins installed?
   - What happens if a plugin fails to load?

**Your findings:**
```
Internal Plugins:
1. 
2. 
3. 
4. 

Loading sequence:
1. 
2. 
3. 

Error handling:

```

#### 1.2 Plugin Hooks Inventory

1. Open `packages/plugin/src/index.ts`
2. List all available hooks in the `Hooks` interface
3. Categorize them by purpose (auth, chat, tool, etc.)

**Hook Categories:**

| Category | Hooks | Purpose |
|----------|-------|---------|
| Authentication | | |
| Chat/LLM | | |
| Tool Execution | | |
| Permissions | | |
| Experimental | | |

#### 1.3 Codex Plugin Deep Dive

1. Open `packages/opencode/src/plugin/codex.ts`
2. Answer these questions:
   - What OAuth provider does it use?
   - How does it handle token refresh?
   - What models does it filter for?
   - How does it modify fetch requests?

**Codex Plugin Analysis:**
```
OAuth Provider: 
Token Refresh Mechanism: 
Allowed Models: 
Fetch Modifications: 
```

#### 1.4 Create a Simple Plugin (Optional)

Create a plugin that logs all tool executions to a file:

```typescript
// my-logging-plugin.ts
import { Plugin } from "@opencode-ai/plugin"
import fs from "fs"

export const LoggingPlugin: Plugin = async (ctx) => {
  const logFile = `${ctx.directory}/.opencode/tool-log.txt`
  
  return {
    "tool.execute.before": async (input, output) => {
      // TODO: Log tool name and arguments
    },
    "tool.execute.after": async (input, output) => {
      // TODO: Log tool output
    },
  }
}
```

---

## Exercise 2: Understand Agent Differences

### Objective
Compare different agents and understand their permissions and use cases.

### Tasks

#### 2.1 Agent Permission Comparison

1. Open `packages/opencode/src/agent/agent.ts`
2. Create a comparison table of agent permissions:

| Permission | build | plan | general | explore |
|------------|-------|------|---------|---------|
| edit | | | | |
| read | | | | |
| bash | | | | |
| grep | | | | |
| glob | | | | |
| question | | | | |
| todowrite | | | | |

#### 2.2 Agent Prompt Analysis

Read the agent prompts in `packages/opencode/src/agent/prompt/`:

1. **explore.txt**: What are the key guidelines?
2. **compaction.txt**: What information should be preserved?
3. **title.txt**: What rules govern title generation?
4. **summary.txt**: What should summaries focus on?

**Prompt Analysis:**
```
Explore Agent Guidelines:
1. 
2. 
3. 

Compaction Focus Areas:
1. 
2. 
3. 

Title Rules:
1. 
2. 
3. 

Summary Focus:
1. 
2. 
3. 
```

#### 2.3 Agent Mode Differences

Explain the difference between these agent modes:
- `primary`
- `subagent`
- `all`

When would you use each?

**Mode Analysis:**
```
primary: 

subagent: 

all: 
```

#### 2.4 Custom Agent Design

Design a custom agent for a specific use case. Fill in:

```json
{
  "agent": {
    "your-agent-name": {
      "name": "",
      "description": "",
      "mode": "",
      "prompt": "",
      "temperature": 0,
      "permission": {
        
      }
    }
  }
}
```

**Use case description:**


---

## Exercise 3: Desktop App Exploration

### Objective
Understand the Tauri desktop app architecture and how it wraps the web app.

### Tasks

#### 3.1 Tauri Configuration Analysis

1. Open `packages/desktop/src-tauri/tauri.conf.json`
2. Answer these questions:
   - What is the app identifier?
   - What build targets are configured?
   - What external binaries are bundled?
   - What deep link schemes are registered?

**Configuration Analysis:**
```
App Identifier: 
Build Targets: 
External Binaries: 
Deep Link Schemes: 
```

#### 3.2 Rust Command Inventory

1. Open `packages/desktop/src-tauri/src/lib.rs`
2. List all Tauri commands (functions with `#[tauri::command]`):

| Command | Purpose | Parameters |
|---------|---------|------------|
| | | |
| | | |
| | | |

#### 3.3 Sidecar Management

1. Open `packages/desktop/src-tauri/src/server.rs`
2. Trace the sidecar lifecycle:
   - How is the sidecar spawned?
   - How is health checked?
   - How is it terminated?

**Sidecar Lifecycle:**
```
Spawning:

Health Check:

Termination:
```

#### 3.4 Frontend-Backend Communication

1. Open `packages/desktop/src/index.tsx`
2. Find where Tauri commands are called
3. Trace how `await_initialization` works

**Communication Flow:**
```
1. Frontend calls: 
2. Rust receives: 
3. Rust responds: 
4. Frontend handles: 
```

#### 3.5 Build the Desktop App (Optional)

If you have Rust installed:

```bash
# Install Tauri prerequisites first
# See: https://v2.tauri.app/start/prerequisites/

cd packages/desktop
bun install
bun run tauri dev
```

Document any issues encountered:


---

## Exercise 4: VS Code Extension Examination

### Objective
Understand how the VS Code extension integrates with opencode.

### Tasks

#### 4.1 Extension Manifest Analysis

1. Open `sdks/vscode/package.json`
2. List all contributed features:

**Commands:**
| Command ID | Title | Keybinding |
|------------|-------|------------|
| | | |
| | | |
| | | |

**Menus:**


**Activation Events:**


#### 4.2 Extension Code Analysis

1. Open `sdks/vscode/src/extension.ts`
2. Answer these questions:
   - How does the extension find an existing opencode terminal?
   - How is the port generated for new terminals?
   - What environment variables are set?
   - How does it wait for opencode to be ready?

**Code Analysis:**
```
Terminal Detection: 

Port Generation: 

Environment Variables: 

Server Ready Check: 
```

#### 4.3 File Reference Format

1. Trace the `getActiveFile()` function
2. Document the file reference format:

**Format Examples:**
```
Whole file: 
Single line: 
Line range: 
```

#### 4.4 API Communication

1. Find the `appendPrompt` function
2. Document the API endpoint and payload:

**API Details:**
```
Endpoint: 
Method: 
Payload: 
```

#### 4.5 Extension Testing (Optional)

1. Clone the extension code
2. Open in VS Code
3. Press F5 to launch Extension Development Host
4. Test the commands

Document your testing:


---

## Exercise 5: Integration Challenge

### Objective
Understand how all these components work together.

### Tasks

#### 5.1 Data Flow Diagram

Create a diagram showing how data flows when:
1. User opens opencode from VS Code
2. Types a question about a file
3. opencode uses a subagent to explore
4. Returns the answer

```
VS Code Extension
       │
       ▼
   [Your diagram here]
```

#### 5.2 Plugin-Agent Interaction

Explain how a plugin could modify agent behavior:
1. What hooks would you use?
2. How would you target specific agents?
3. What are the limitations?

**Analysis:**


#### 5.3 Desktop vs Web vs CLI

Compare the three interfaces:

| Feature | Desktop | Web | CLI |
|---------|---------|-----|-----|
| Sidecar | | | |
| File Access | | | |
| Notifications | | | |
| Auto-Update | | | |
| Deep Links | | | |

---

## Reflection Questions

After completing these exercises, answer:

1. **Plugin System**: What would you build with the plugin system?


2. **Agent System**: How would you customize agents for your workflow?


3. **Desktop App**: What are the tradeoffs of Tauri vs Electron?


4. **VS Code Extension**: How could the extension be improved?


5. **Overall Architecture**: What design patterns did you notice across these components?


---

## Bonus Challenges

### Challenge 1: Create a Custom Plugin
Build a plugin that:
- Adds a custom tool
- Modifies chat parameters based on file type
- Logs usage statistics

### Challenge 2: Design a New Agent
Create an agent specialized for:
- Code review
- Documentation generation
- Test writing

### Challenge 3: Extend the VS Code Extension
Add a feature that:
- Shows opencode status in the status bar
- Provides code lens for quick actions
- Integrates with VS Code's problems panel

---

## Resources

- [Tauri v2 Documentation](https://v2.tauri.app/)
- [VS Code Extension API](https://code.visualstudio.com/api)
- [Effect-TS Documentation](https://effect.website/)
- [Zod Documentation](https://zod.dev/)
