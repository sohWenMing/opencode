# Exercise 01: Server Exploration

## Overview

In this exercise, you'll explore the opencode HTTP server by starting it, making API calls, and tracing request flows through the codebase.

---

## Prerequisites

- opencode installed and working
- `curl` or similar HTTP client
- A terminal for running commands
- Your favorite code editor for reading source files

---

## Part 1: Starting the Server

### Exercise 1.1: Start opencode in a Project Directory

1. Navigate to any git repository (or create a test one):
   ```bash
   mkdir ~/test-project && cd ~/test-project
   git init
   ```

2. Start opencode:
   ```bash
   opencode
   ```

3. Note the port number displayed (usually 4096 or auto-assigned)

### Exercise 1.2: Find the Server Port

If you're running opencode in TUI mode, the server is already running. Find the port:

```bash
# Option 1: Check the opencode state directory
cat ~/.opencode/state/server.json

# Option 2: Look for the process
lsof -i -P | grep opencode
```

**Question**: What port is your server running on?

---

## Part 2: Health Check and Basic Endpoints

### Exercise 2.1: Health Check

Make a health check request:

```bash
curl http://localhost:4096/global/health
```

**Expected Response**:
```json
{"healthy":true,"version":"0.x.x"}
```

**Question**: What version of opencode are you running?

### Exercise 2.2: Get Current Project

```bash
curl http://localhost:4096/project/current
```

**Questions**:
1. What fields are in the project response?
2. What is the project ID format?
3. Is there VCS (git) information?

### Exercise 2.3: Get Configuration

```bash
curl http://localhost:4096/config
```

**Questions**:
1. What configuration options are available?
2. Are any providers configured?
3. What's the default model (if set)?

---

## Part 3: Session Management

### Exercise 3.1: List Sessions

```bash
curl http://localhost:4096/session
```

**Questions**:
1. How many sessions exist?
2. What fields does each session have?
3. What's the format of session IDs?

### Exercise 3.2: Create a Session

```bash
curl -X POST http://localhost:4096/session \
  -H "Content-Type: application/json" \
  -d '{}'
```

**Questions**:
1. What's the ID of the new session?
2. What's the default title?
3. What timestamps are included?

### Exercise 3.3: Get Session Details

Using the session ID from 3.2:

```bash
curl http://localhost:4096/session/YOUR_SESSION_ID
```

### Exercise 3.4: Update Session Title

```bash
curl -X PATCH http://localhost:4096/session/YOUR_SESSION_ID \
  -H "Content-Type: application/json" \
  -d '{"title": "My Test Session"}'
```

**Question**: Verify the title changed by fetching the session again.

### Exercise 3.5: Delete Session

```bash
curl -X DELETE http://localhost:4096/session/YOUR_SESSION_ID
```

**Question**: What response do you get? Verify it's deleted by listing sessions.

---

## Part 4: Provider and Model Information

### Exercise 4.1: List Providers

```bash
curl http://localhost:4096/provider
```

**Questions**:
1. What providers are available?
2. Which providers are "connected" (have credentials)?
3. What's the default model for each provider?

### Exercise 4.2: Get Auth Methods

```bash
curl http://localhost:4096/provider/auth
```

**Question**: What authentication methods are available for each provider?

---

## Part 5: File Operations

### Exercise 5.1: List Files

```bash
curl "http://localhost:4096/file?path=."
```

**Questions**:
1. What files are in the current directory?
2. What information is provided for each file?

### Exercise 5.2: Read File Content

```bash
curl "http://localhost:4096/file/content?path=./README.md"
```

(Adjust the path to an existing file in your project)

### Exercise 5.3: Search for Text

```bash
curl "http://localhost:4096/find?pattern=TODO"
```

**Question**: What format are the search results in?

### Exercise 5.4: Find Files

```bash
curl "http://localhost:4096/find/file?query=.ts"
```

**Question**: How many TypeScript files are found?

---

## Part 6: Event Streaming

### Exercise 6.1: Connect to Event Stream

Open a new terminal and run:

```bash
curl -N http://localhost:4096/event
```

**Questions**:
1. What's the first event you receive?
2. How often do heartbeat events arrive?

### Exercise 6.2: Trigger Events

While the event stream is open, in another terminal:

1. Create a session:
   ```bash
   curl -X POST http://localhost:4096/session -H "Content-Type: application/json" -d '{}'
   ```

2. Watch the event stream for `session.updated` events

**Question**: What events did you see when the session was created?

### Exercise 6.3: Global Event Stream

```bash
curl -N http://localhost:4096/global/event
```

**Question**: How does this differ from `/event`?

---

## Part 7: MCP Status

### Exercise 7.1: Check MCP Servers

```bash
curl http://localhost:4096/mcp
```

**Questions**:
1. Are any MCP servers configured?
2. What status values are possible?

---

## Part 8: OpenAPI Documentation

### Exercise 8.1: Get API Spec

```bash
curl http://localhost:4096/doc
```

**Questions**:
1. What format is the documentation in?
2. How many endpoints are documented?
3. Find the schema for `Session.Info`

---

## Part 9: Request Tracing

### Exercise 9.1: Trace a Request Through the Code

For the request `GET /session`:

1. Find the route handler in `packages/opencode/src/server/routes/session.ts`
2. Identify what validation occurs
3. Find the core function that's called (`Session.list`)
4. Trace to `packages/opencode/src/session/index.ts`
5. Find where the database query happens

**Document your findings**:
- Route file: ___
- Handler line number: ___
- Core function: ___
- Database query location: ___

### Exercise 9.2: Trace Event Publishing

For the `session.updated` event:

1. Find where `Bus.publish(Session.Event.Updated, ...)` is called
2. Trace how it reaches the `/event` SSE endpoint
3. Document the flow

---

## Part 10: Error Handling

### Exercise 10.1: Trigger a 404

```bash
curl http://localhost:4096/session/nonexistent-id
```

**Questions**:
1. What's the HTTP status code?
2. What's the error response format?

### Exercise 10.2: Trigger a 400

```bash
curl -X POST http://localhost:4096/session \
  -H "Content-Type: application/json" \
  -d '{"invalid": "field"}'
```

**Question**: What validation error do you get?

---

## Bonus Challenges

### Challenge 1: Permission Flow

1. Start a session and send a message that would trigger a permission request
2. Use `/permission` to list pending permissions
3. Use `/permission/:id/reply` to respond

### Challenge 2: PTY Session

1. Create a PTY session: `POST /pty`
2. Connect via WebSocket (requires a WebSocket client)
3. Send commands and receive output

### Challenge 3: Build a Simple Client

Write a script (in any language) that:
1. Creates a session
2. Subscribes to the event stream
3. Sends a message
4. Prints streaming responses as they arrive

---

## Reflection Questions

1. What patterns do you notice across all the route handlers?
2. How does the validation layer help ensure data integrity?
3. What's the relationship between the HTTP API and the TUI?
4. How would you add a new endpoint to the server?

---

## Summary

In this exercise, you explored:
- Server startup and port discovery
- Basic CRUD operations on sessions
- Provider and configuration endpoints
- File operations and search
- Real-time event streaming
- Error handling patterns
- Request tracing through the codebase

These skills will help you understand how clients interact with opencode and how to debug issues in the server layer.
