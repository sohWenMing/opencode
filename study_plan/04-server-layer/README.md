# Module 04: Server Layer

## Overview

This module covers the HTTP API layer that enables communication between clients (TUI, web app, external tools) and the opencode core functionality. You'll learn how the server is built using the Hono framework, how routes are organized, and how real-time updates flow to clients.

---

## Module Structure

```
04-server-layer/
├── README.md                          # This file
├── 01-hono-framework.md              # Hono basics and server setup
├── 02-routes-and-handlers.md         # Route organization and patterns
├── 03-websocket-events.md            # Real-time communication
└── exercises/
    └── 01-server-exploration.md      # Hands-on API exploration
```

---

## Learning Path

### Lesson 1: Hono Framework
**Time: 45-60 minutes**

Learn the fundamentals of Hono and how opencode uses it:
- What Hono is and why it was chosen
- Basic concepts: app, routes, middleware, context
- Server initialization and configuration
- CORS, authentication, and error handling
- OpenAPI documentation generation

### Lesson 2: Routes and Handlers
**Time: 60-90 minutes**

Deep dive into the route structure:
- Routes folder organization
- Common route patterns with validation
- Session, provider, config, MCP routes
- Permission and question handling
- Request/response flow

### Lesson 3: WebSocket and Events
**Time: 45-60 minutes**

Understand real-time communication:
- Server-Sent Events (SSE) vs WebSocket
- The Bus event system
- Event types and payloads
- Global vs instance events
- PTY WebSocket connections

### Exercise: Server Exploration
**Time: 90-120 minutes**

Hands-on practice:
- Start the server and explore endpoints
- Make API calls with curl
- Trace requests through the code
- Understand event streaming

---

## Key Concepts

### Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                         Clients                                     │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────────────────┐   │
│  │   TUI   │  │ Web App │  │  curl   │  │ External Tools      │   │
│  └────┬────┘  └────┬────┘  └────┬────┘  └──────────┬──────────┘   │
└───────┼────────────┼────────────┼───────────────────┼──────────────┘
        │            │            │                   │
        └────────────┴────────────┴───────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      HTTP Server (Hono)                             │
├─────────────────────────────────────────────────────────────────────┤
│  Middleware: Error → Auth → Logging → CORS → Context               │
├─────────────────────────────────────────────────────────────────────┤
│  Routes:                                                            │
│  /session  /provider  /config  /mcp  /permission  /question        │
│  /project  /pty       /event   /file /global      /tui             │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      Core Business Logic                            │
│  Session, Provider, Config, MCP, Permission, Question, etc.        │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      Storage & External Services                    │
│  SQLite Database, File System, LLM APIs, MCP Servers               │
└─────────────────────────────────────────────────────────────────────┘
```

### Key Files

| File | Purpose |
|------|---------|
| `packages/opencode/src/server/server.ts` | Main server setup and middleware |
| `packages/opencode/src/server/routes/*.ts` | Route handlers |
| `packages/opencode/src/server/error.ts` | Error response schemas |
| `packages/opencode/src/bus/index.ts` | Event pub/sub system |
| `packages/opencode/src/bus/bus-event.ts` | Event definition system |

### Request Flow

1. **HTTP Request** arrives at the server
2. **Middleware Stack** processes the request:
   - Error handler wraps everything
   - Auth check (if password configured)
   - Request logging with timing
   - CORS validation
   - Workspace/Instance context setup
3. **Route Matching** finds the handler
4. **Validation** checks params, query, body
5. **Handler** calls core business logic
6. **Response** is sent back to client
7. **Events** are published to Bus for real-time updates

### Event Flow

1. **Core Logic** publishes event via `Bus.publish()`
2. **Bus** distributes to:
   - Typed subscribers (specific event)
   - Wildcard subscribers (all events)
   - GlobalBus (cross-instance)
3. **SSE Endpoint** streams to connected clients
4. **Clients** (TUI/Web) update their UI

---

## Prerequisites

Before starting this module, you should have completed:
- Module 00: Foundations (TypeScript, Bun basics)
- Module 02: Architecture Overview (package relationships)

Helpful but not required:
- Module 03: CLI and TUI (understanding how clients use the API)

---

## Learning Outcomes

After completing this module, you will be able to:

1. **Explain** how the Hono server is configured and started
2. **Navigate** the routes folder and find specific endpoints
3. **Read** route handlers and understand validation patterns
4. **Trace** requests from HTTP to core logic to response
5. **Understand** how real-time events flow to clients
6. **Debug** server issues by understanding the middleware stack
7. **Add** new endpoints following existing patterns

---

## Practical Applications

This knowledge enables you to:

- **Build integrations** that call the opencode API
- **Debug issues** in the server layer
- **Add new endpoints** for custom functionality
- **Understand** how the TUI and web app communicate with the backend
- **Monitor** events for debugging or logging

---

## Next Steps

After completing this module, consider:

1. **Module 05: Session System** - Deep dive into chat session management
2. **Module 06: Provider Integration** - How LLM providers are connected
3. **Module 07: Tool System** - How AI tools are implemented

---

## Quick Reference

### Common Endpoints

```bash
# Health check
GET /global/health

# Sessions
GET /session                    # List sessions
POST /session                   # Create session
GET /session/:id                # Get session
POST /session/:id/message       # Send message
POST /session/:id/abort         # Abort processing

# Configuration
GET /config                     # Get config
PATCH /config                   # Update config

# Providers
GET /provider                   # List providers
GET /provider/auth              # Get auth methods

# Events
GET /event                      # SSE event stream
GET /global/event               # Global SSE stream
```

### Event Types

```
session.updated          # Session metadata changed
session.error            # Error occurred
message.updated          # Message created/updated
message.part.updated     # Message part changed
message.part.delta       # Streaming text
permission.asked         # Permission requested
permission.replied       # Permission answered
question.asked           # Question requested
question.replied         # Question answered
server.connected         # Client connected
server.heartbeat         # Keep-alive
```

---

## Resources

- [Hono Documentation](https://hono.dev/)
- [Hono OpenAPI Middleware](https://github.com/honojs/middleware/tree/main/packages/openapi)
- [Server-Sent Events MDN](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events)
- [WebSocket API MDN](https://developer.mozilla.org/en-US/docs/Web/API/WebSocket)
- [Zod Documentation](https://zod.dev/)
