# Module 06: Session System

## Overview

This module covers the **heart of opencode** - the session system that orchestrates conversations with LLMs. Understanding this module is essential for comprehending how opencode works at its core.

The session system is responsible for:
- Managing conversation state and persistence
- Orchestrating the agent loop that drives AI interactions
- Processing LLM streams and handling tool calls
- Managing context windows through compaction
- Tracking costs, tokens, and file changes

---

## Module Structure

```
06-session-system/
├── README.md                        # This file
├── 01-session-lifecycle.md          # Session CRUD and persistence
├── 02-message-format.md             # Message structure and parts
├── 03-prompt-orchestration.md       # The main agent loop (THE HARNESS)
├── 04-processor-streaming.md        # LLM stream processing
├── 05-compaction-and-context.md     # Context window management
└── exercises/
    └── 01-session-tracing.md        # Hands-on debugging exercises
```

---

## Key Source Files

| File | Purpose |
|------|---------|
| `packages/opencode/src/session/index.ts` | Session CRUD operations |
| `packages/opencode/src/session/prompt.ts` | Main orchestration loop |
| `packages/opencode/src/session/processor.ts` | Stream processing |
| `packages/opencode/src/session/message-v2.ts` | Message format |
| `packages/opencode/src/session/compaction.ts` | Context management |
| `packages/opencode/src/session/llm.ts` | LLM calls |
| `packages/opencode/src/session/summary.ts` | Diff tracking |
| `packages/opencode/src/session/schema.ts` | ID types |
| `packages/opencode/src/session/session.sql.ts` | Database schema |
| `packages/opencode/src/session/retry.ts` | Retry logic |
| `packages/opencode/src/session/status.ts` | Session status |
| `packages/opencode/src/session/system.ts` | System prompts |

---

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           SESSION SYSTEM                                 │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                     SessionPrompt.prompt()                       │    │
│  │  ┌─────────────────────────────────────────────────────────┐    │    │
│  │  │                  createUserMessage()                     │    │    │
│  │  └─────────────────────────────────────────────────────────┘    │    │
│  │                            │                                     │    │
│  │                            ▼                                     │    │
│  │  ┌─────────────────────────────────────────────────────────┐    │    │
│  │  │                    loop() - while(true)                  │    │    │
│  │  │  ┌───────────────────────────────────────────────────┐  │    │    │
│  │  │  │ 1. Load messages (filterCompacted)                │  │    │    │
│  │  │  │ 2. Find lastUser, lastAssistant, tasks            │  │    │    │
│  │  │  │ 3. Check exit conditions                          │  │    │    │
│  │  │  │ 4. Process subtask OR compaction OR normal        │  │    │    │
│  │  │  │ 5. Create processor                               │  │    │    │
│  │  │  │ 6. Resolve tools (Registry + MCP)                 │  │    │    │
│  │  │  │ 7. Build system prompt                            │  │    │    │
│  │  │  │ 8. Call processor.process()                       │  │    │    │
│  │  │  │ 9. Handle result (continue/stop/compact)          │  │    │    │
│  │  │  └───────────────────────────────────────────────────┘  │    │    │
│  │  └─────────────────────────────────────────────────────────┘    │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                   SessionProcessor.process()                     │    │
│  │  ┌───────────────────────────────────────────────────────────┐  │    │
│  │  │                    LLM.stream()                            │  │    │
│  │  └───────────────────────────────────────────────────────────┘  │    │
│  │                            │                                     │    │
│  │                            ▼                                     │    │
│  │  ┌───────────────────────────────────────────────────────────┐  │    │
│  │  │              for await (event of fullStream)               │  │    │
│  │  │  ├── start → set status busy                              │  │    │
│  │  │  ├── reasoning-* → create/update ReasoningPart            │  │    │
│  │  │  ├── text-* → create/update TextPart                      │  │    │
│  │  │  ├── tool-* → create/update ToolPart                      │  │    │
│  │  │  ├── start-step → snapshot + StepStartPart                │  │    │
│  │  │  ├── finish-step → usage + StepFinishPart                 │  │    │
│  │  │  └── error → throw                                        │  │    │
│  │  └───────────────────────────────────────────────────────────┘  │    │
│  │                            │                                     │    │
│  │                            ▼                                     │    │
│  │  ┌───────────────────────────────────────────────────────────┐  │    │
│  │  │              Return: "continue" | "stop" | "compact"       │  │    │
│  │  └───────────────────────────────────────────────────────────┘  │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                   SessionCompaction.process()                    │    │
│  │  ├── Generate summary of conversation                           │    │
│  │  ├── Prune old tool outputs                                     │    │
│  │  └── Continue with condensed context                            │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Learning Path

### Lesson 1: Session Lifecycle
- What a session is
- Session.Info type
- CRUD operations
- Database persistence
- Event system

### Lesson 2: Message Format
- Message structure (User vs Assistant)
- Part types (Text, Tool, Reasoning, etc.)
- Converting to AI SDK format
- Handling interrupted tools

### Lesson 3: Prompt Orchestration
- The main while(true) loop
- Task/subtask processing
- Tool assembly
- System prompt construction
- Error handling

### Lesson 4: Processor & Streaming
- LLM.stream() integration
- Stream event handling
- Tool call lifecycle
- Doom loop detection
- Retry logic

### Lesson 5: Compaction & Context
- Why compaction is needed
- Overflow detection
- Summary generation
- Pruning old outputs
- Context filtering

---

## Key Concepts

### Session State Machine

```
┌─────────┐     prompt()      ┌─────────┐
│  IDLE   │──────────────────▶│  BUSY   │
└─────────┘                   └────┬────┘
     ▲                             │
     │                             │
     │         ┌───────────────────┼───────────────────┐
     │         │                   │                   │
     │         ▼                   ▼                   ▼
     │    ┌─────────┐        ┌─────────┐        ┌─────────┐
     │    │  ERROR  │        │  RETRY  │        │ COMPACT │
     │    └────┬────┘        └────┬────┘        └────┬────┘
     │         │                  │                   │
     └─────────┴──────────────────┴───────────────────┘
```

### Message Part Types

| Part Type | Purpose |
|-----------|---------|
| `text` | Plain text content |
| `file` | Attached files/images |
| `tool` | Tool invocations |
| `reasoning` | Model thinking (CoT) |
| `step-start` | LLM step boundary |
| `step-finish` | Step completion with usage |
| `patch` | File changes |
| `subtask` | Task tool invocation |
| `compaction` | Compaction marker |

### Tool States

```
PENDING → RUNNING → COMPLETED
                 → ERROR
```

---

## Prerequisites

Before starting this module, ensure you understand:
- TypeScript and async/await
- Zod schemas
- Event-driven architecture
- Basic LLM concepts (tokens, context windows)

---

## Estimated Time

- Lesson 1: 45 minutes
- Lesson 2: 60 minutes
- Lesson 3: 90 minutes (most complex)
- Lesson 4: 60 minutes
- Lesson 5: 45 minutes
- Exercises: 2-3 hours

**Total: ~6-7 hours**

---

## Next Steps

After completing this module, you should:
1. Be able to trace any user prompt through the system
2. Understand how tool calls are processed
3. Know when and why compaction occurs
4. Be able to debug session-related issues

Continue to **Module 07: Provider System** to learn how LLM providers are integrated.
