# Lesson 03: WebSocket and Events

## Learning Objectives

By the end of this lesson, you will be able to:
- Understand how real-time updates work in opencode
- Explain the difference between SSE and WebSocket
- Trace how events flow from core logic to clients
- Understand the Bus event system
- Identify different event types and their payloads

---

## Real-Time Communication Overview

opencode uses two mechanisms for real-time communication:

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Real-Time Communication                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────────────────────┐  ┌─────────────────────────────┐  │
│  │   Server-Sent Events (SSE)  │  │       WebSocket             │  │
│  ├─────────────────────────────┤  ├─────────────────────────────┤  │
│  │  • One-way (server→client)  │  │  • Bidirectional            │  │
│  │  • HTTP-based               │  │  • Full-duplex              │  │
│  │  • Auto-reconnect           │  │  • Lower latency            │  │
│  │  • Used for: Events, status │  │  • Used for: PTY terminals  │  │
│  └─────────────────────────────┘  └─────────────────────────────┘  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Server-Sent Events (SSE)

SSE is the primary mechanism for pushing updates to clients (TUI, web app).

### Event Route Implementation

```1:84:packages/opencode/src/server/routes/event.ts
import { Hono } from "hono"
import { describeRoute, resolver } from "hono-openapi"
import { streamSSE } from "hono/streaming"
import { Log } from "@/util/log"
import { BusEvent } from "@/bus/bus-event"
import { Bus } from "@/bus"
import { lazy } from "../../util/lazy"
import { AsyncQueue } from "../../util/queue"

const log = Log.create({ service: "server" })

export const EventRoutes = lazy(() =>
  new Hono().get(
    "/event",
    describeRoute({
      summary: "Subscribe to events",
      description: "Get events",
      operationId: "event.subscribe",
      responses: {
        200: {
          description: "Event stream",
          content: {
            "text/event-stream": {
              schema: resolver(BusEvent.payloads()),
            },
          },
        },
      },
    }),
    async (c) => {
      log.info("event connected")
      c.header("X-Accel-Buffering", "no")
      c.header("X-Content-Type-Options", "nosniff")
      return streamSSE(c, async (stream) => {
        const q = new AsyncQueue<string | null>()
        let done = false

        q.push(
          JSON.stringify({
            type: "server.connected",
            properties: {},
          }),
        )

        // Send heartbeat every 10s to prevent stalled proxy streams.
        const heartbeat = setInterval(() => {
          q.push(
            JSON.stringify({
              type: "server.heartbeat",
              properties: {},
            }),
          )
        }, 10_000)

        const stop = () => {
          if (done) return
          done = true
          clearInterval(heartbeat)
          unsub()
          q.push(null)
          log.info("event disconnected")
        }

        const unsub = Bus.subscribeAll((event) => {
          q.push(JSON.stringify(event))
          if (event.type === Bus.InstanceDisposed.type) {
            stop()
          }
        })

        stream.onAbort(stop)

        try {
          for await (const data of q) {
            if (data === null) return
            await stream.writeSSE({ data })
          }
        } finally {
          stop()
        }
      })
    },
  ),
)
```

### SSE Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                         SSE Event Flow                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Client                          Server                             │
│    │                               │                                │
│    │  GET /event                   │                                │
│    │ ─────────────────────────────▶│                                │
│    │                               │                                │
│    │  HTTP 200 (text/event-stream) │                                │
│    │ ◀─────────────────────────────│                                │
│    │                               │                                │
│    │  data: {"type":"server.connected",...}                        │
│    │ ◀─────────────────────────────│                                │
│    │                               │                                │
│    │  data: {"type":"session.updated",...}                         │
│    │ ◀─────────────────────────────│  (when session changes)       │
│    │                               │                                │
│    │  data: {"type":"server.heartbeat",...}                        │
│    │ ◀─────────────────────────────│  (every 10 seconds)           │
│    │                               │                                │
│    │  data: {"type":"message.part.updated",...}                    │
│    │ ◀─────────────────────────────│  (during AI response)         │
│    │                               │                                │
│    │  Connection closed            │                                │
│    │ ◀─────────────────────────────│  (on instance dispose)        │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Key SSE Features

1. **Initial Connection Event**: Sends `server.connected` immediately
2. **Heartbeat**: Sends `server.heartbeat` every 10 seconds to keep connection alive
3. **Event Subscription**: Uses `Bus.subscribeAll()` to receive all events
4. **Clean Shutdown**: Stops on `InstanceDisposed` event or client disconnect

---

## The Bus Event System

The Bus is the central event pub/sub system in opencode.

### Bus Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                         Bus Event System                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────────┐                                               │
│  │  Event Source   │  (Session, Message, Permission, etc.)         │
│  └────────┬────────┘                                               │
│           │                                                         │
│           │ Bus.publish(EventDef, properties)                       │
│           ▼                                                         │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                        Bus                                   │   │
│  │  ┌─────────────────┐  ┌─────────────────────────────────┐   │   │
│  │  │  Typed PubSub   │  │      Wildcard PubSub            │   │   │
│  │  │  (per event)    │  │      (all events)               │   │   │
│  │  └────────┬────────┘  └──────────────┬──────────────────┘   │   │
│  │           │                          │                       │   │
│  │           ▼                          ▼                       │   │
│  │  ┌─────────────────┐  ┌─────────────────────────────────┐   │   │
│  │  │ Typed Listeners │  │      GlobalBus.emit()           │   │   │
│  │  │ (specific event)│  │      (cross-instance events)    │   │   │
│  │  └─────────────────┘  └──────────────┬──────────────────┘   │   │
│  └──────────────────────────────────────┼───────────────────────┘   │
│                                         │                           │
│                                         ▼                           │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                    SSE Endpoint                              │   │
│  │              (streams to TUI/Web clients)                    │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Bus Implementation

```1:184:packages/opencode/src/bus/index.ts
import z from "zod"
import { Effect, Exit, Layer, PubSub, Scope, ServiceMap, Stream } from "effect"
import { Log } from "../util/log"
import { Instance } from "../project/instance"
import { BusEvent } from "./bus-event"
import { GlobalBus } from "./global"
import { InstanceState } from "@/effect/instance-state"
import { makeRuntime } from "@/effect/run-service"

export namespace Bus {
  const log = Log.create({ service: "bus" })

  export const InstanceDisposed = BusEvent.define(
    "server.instance.disposed",
    z.object({
      directory: z.string(),
    }),
  )

  // ... Effect-based implementation

  // Public API
  export async function publish<D extends BusEvent.Definition>(def: D, properties: z.output<D["properties"]>) {
    return runPromise((svc) => svc.publish(def, properties))
  }

  export function subscribe<D extends BusEvent.Definition>(
    def: D,
    callback: (event: { type: D["type"]; properties: z.infer<D["properties"]> }) => unknown,
  ) {
    return runSync((svc) => svc.subscribeCallback(def, callback))
  }

  export function subscribeAll(callback: (event: any) => unknown) {
    return runSync((svc) => svc.subscribeAllCallback(callback))
  }
}
```

### Publishing Events

Events are published from various parts of the codebase:

```typescript
// Publishing a session update
Bus.publish(Session.Event.Updated, {
  sessionID,
  info: result,
})

// Publishing a message part update
Bus.publish(MessageV2.Event.PartUpdated, {
  part: updatedPart,
})

// Publishing a file edit
Bus.publish(File.Event.Edited, { file: filepath })
```

### Subscribing to Events

```typescript
// Subscribe to specific event type
const unsub = Bus.subscribe(Session.Event.Updated, (evt) => {
  console.log("Session updated:", evt.properties.sessionID)
})

// Subscribe to all events
const unsub = Bus.subscribeAll((event) => {
  console.log("Event:", event.type, event.properties)
})

// Unsubscribe when done
unsub()
```

---

## Event Definition System

Events are defined using `BusEvent.define`:

```1:40:packages/opencode/src/bus/bus-event.ts
import z from "zod"
import type { ZodType } from "zod"

export namespace BusEvent {
  export type Definition = ReturnType<typeof define>

  const registry = new Map<string, Definition>()

  export function define<Type extends string, Properties extends ZodType>(type: Type, properties: Properties) {
    const result = {
      type,
      properties,
    }
    registry.set(type, result)
    return result
  }

  export function payloads() {
    return z
      .discriminatedUnion(
        "type",
        registry
          .entries()
          .map(([type, def]) => {
            return z
              .object({
                type: z.literal(type),
                properties: def.properties,
              })
              .meta({
                ref: "Event" + "." + def.type,
              })
          })
          .toArray() as any,
      )
      .meta({
        ref: "Event",
      })
  }
}
```

### Example Event Definitions

```typescript
// Session events
export const Event = {
  Updated: BusEvent.define(
    "session.updated",
    z.object({
      sessionID: SessionID.zod,
      info: Session.Info,
    }),
  ),
  Error: BusEvent.define(
    "session.error",
    z.object({
      sessionID: SessionID.zod.optional(),
      error: NamedError.Schema,
    }),
  ),
  Diff: BusEvent.define(
    "session.diff",
    z.object({
      sessionID: SessionID.zod,
      diff: Snapshot.FileDiff.array(),
    }),
  ),
}

// Message events
export const Event = {
  Updated: BusEvent.define(
    "message.updated",
    z.object({
      info: MessageV2.Info,
    }),
  ),
  PartUpdated: BusEvent.define(
    "message.part.updated",
    z.object({
      part: MessageV2.Part,
    }),
  ),
  PartDelta: BusEvent.define(
    "message.part.delta",
    z.object({
      sessionID: SessionID.zod,
      messageID: MessageID.zod,
      partID: PartID.zod,
      delta: z.string(),
    }),
  ),
}
```

---

## Common Event Types

### Session Events

| Event Type | Description | Properties |
|------------|-------------|------------|
| `session.updated` | Session metadata changed | sessionID, info |
| `session.error` | Error occurred | sessionID?, error |
| `session.diff` | File diff computed | sessionID, diff |

### Message Events

| Event Type | Description | Properties |
|------------|-------------|------------|
| `message.updated` | Message created/updated | info |
| `message.part.updated` | Message part changed | part |
| `message.part.delta` | Streaming text delta | sessionID, messageID, partID, delta |

### Permission Events

| Event Type | Description | Properties |
|------------|-------------|------------|
| `permission.asked` | Permission requested | sessionID, id, type, ... |
| `permission.replied` | Permission answered | sessionID, requestID, reply |

### Question Events

| Event Type | Description | Properties |
|------------|-------------|------------|
| `question.asked` | Question requested | sessionID, id, questions |
| `question.replied` | Question answered | sessionID, requestID, answers |
| `question.rejected` | Question rejected | sessionID, requestID |

### File Events

| Event Type | Description | Properties |
|------------|-------------|------------|
| `file.edited` | File was edited | file |
| `file.watcher.updated` | File changed on disk | file, event |

### Server Events

| Event Type | Description | Properties |
|------------|-------------|------------|
| `server.connected` | Client connected | (none) |
| `server.heartbeat` | Keep-alive ping | (none) |
| `server.instance.disposed` | Instance shutting down | directory |

---

## Global Bus

For cross-instance events, there's a GlobalBus:

```1:10:packages/opencode/src/bus/global.ts
import { EventEmitter } from "events"

export const GlobalBus = new EventEmitter<{
  event: [
    {
      directory?: string
      payload: any
    },
  ]
}>()
```

### Global Event Flow

```
┌─────────────────────────────────────────────────────────────────────┐
│                      Global Event Flow                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Instance A                    Instance B                           │
│  (Project 1)                   (Project 2)                          │
│      │                             │                                │
│      │ Bus.publish()               │                                │
│      ▼                             │                                │
│  ┌─────────┐                       │                                │
│  │ Local   │                       │                                │
│  │ PubSub  │                       │                                │
│  └────┬────┘                       │                                │
│       │                            │                                │
│       │ GlobalBus.emit()           │                                │
│       ▼                            │                                │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                      GlobalBus                               │   │
│  │              (shared across all instances)                   │   │
│  └─────────────────────────────────────────────────────────────┘   │
│       │                            │                                │
│       ▼                            ▼                                │
│  /global/event               /global/event                          │
│  (SSE endpoint)              (SSE endpoint)                         │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Global Event Endpoint

```93:132:packages/opencode/src/server/routes/global.ts
    .get(
      "/event",
      describeRoute({
        summary: "Get global events",
        description: "Subscribe to global events from the OpenCode system using server-sent events.",
        operationId: "global.event",
        responses: {
          200: {
            description: "Event stream",
            content: {
              "text/event-stream": {
                schema: resolver(
                  z
                    .object({
                      directory: z.string(),
                      payload: BusEvent.payloads(),
                    })
                    .meta({
                      ref: "GlobalEvent",
                    }),
                ),
              },
            },
          },
        },
      }),
      async (c) => {
        log.info("global event connected")
        c.header("X-Accel-Buffering", "no")
        c.header("X-Content-Type-Options", "nosniff")

        return streamEvents(c, (q) => {
          async function handler(event: any) {
            q.push(JSON.stringify(event))
          }
          GlobalBus.on("event", handler)
          return () => GlobalBus.off("event", handler)
        })
      },
    )
```

---

## WebSocket for PTY

WebSocket is used for bidirectional PTY communication:

```153:209:packages/opencode/src/server/routes/pty.ts
      upgradeWebSocket(async (c) => {
        const id = PtyID.zod.parse(c.req.param("ptyID"))
        const cursor = (() => {
          const value = c.req.query("cursor")
          if (!value) return
          const parsed = Number(value)
          if (!Number.isSafeInteger(parsed) || parsed < -1) return
          return parsed
        })()
        let handler: Awaited<ReturnType<typeof Pty.connect>>
        if (!(await Pty.get(id))) throw new Error("Session not found")

        // ... socket type checking

        return {
          async onOpen(_event, ws) {
            const socket = ws.raw
            if (!isSocket(socket)) {
              ws.close()
              return
            }
            handler = await Pty.connect(id, socket, cursor)
            ready = true
            for (const msg of pending) handler?.onMessage(msg)
            pending.length = 0
          },
          onMessage(event) {
            if (typeof event.data !== "string") return
            if (!ready) {
              pending.push(event.data)
              return
            }
            handler?.onMessage(event.data)
          },
          onClose() {
            handler?.onClose()
          },
          onError() {
            handler?.onClose()
          },
        }
      }),
```

### WebSocket Flow

```
┌─────────────────────────────────────────────────────────────────────┐
│                      PTY WebSocket Flow                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Client (TUI/Web)              Server                  PTY Process  │
│       │                           │                         │       │
│       │  GET /pty/:id/connect     │                         │       │
│       │  Upgrade: websocket       │                         │       │
│       │ ─────────────────────────▶│                         │       │
│       │                           │                         │       │
│       │  101 Switching Protocols  │                         │       │
│       │ ◀─────────────────────────│                         │       │
│       │                           │                         │       │
│       │  WS: user input           │                         │       │
│       │ ─────────────────────────▶│ ───────────────────────▶│       │
│       │                           │                         │       │
│       │                           │  terminal output        │       │
│       │ ◀─────────────────────────│ ◀───────────────────────│       │
│       │  WS: terminal output      │                         │       │
│       │                           │                         │       │
│       │  WS: resize event         │                         │       │
│       │ ─────────────────────────▶│ ───────────────────────▶│       │
│       │                           │                         │       │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Event Consumption in TUI

The TUI subscribes to events to update its display:

```typescript
// Example: TUI subscribing to session events
Bus.subscribe(Session.Event.Updated, (evt) => {
  // Update session list in UI
  updateSessionDisplay(evt.properties.info)
})

// Example: TUI subscribing to message streaming
Bus.subscribe(MessageV2.Event.PartDelta, (evt) => {
  // Append streaming text to message display
  appendToMessage(evt.properties.sessionID, evt.properties.delta)
})

// Example: TUI handling permission requests
Bus.subscribe(Permission.Event.Asked, (evt) => {
  // Show permission dialog
  showPermissionDialog(evt.properties)
})
```

---

## Sync Events

For data synchronization, there's a separate SyncEvent system:

```133:177:packages/opencode/src/server/routes/global.ts
    .get(
      "/sync-event",
      describeRoute({
        summary: "Subscribe to global sync events",
        description: "Get global sync events",
        operationId: "global.sync-event.subscribe",
        responses: {
          200: {
            description: "Event stream",
            content: {
              "text/event-stream": {
                schema: resolver(
                  z
                    .object({
                      payload: SyncEvent.payloads(),
                    })
                    .meta({
                      ref: "SyncEvent",
                    }),
                ),
              },
            },
          },
        },
      }),
      async (c) => {
        log.info("global sync event connected")
        c.header("X-Accel-Buffering", "no")
        c.header("X-Content-Type-Options", "nosniff")
        return streamEvents(c, (q) => {
          return SyncEvent.subscribeAll(({ def, event }) => {
            q.push(
              JSON.stringify({
                payload: {
                  ...event,
                  type: SyncEvent.versionedType(def.type, def.version),
                },
              }),
            )
          })
        })
      },
    )
```

---

## Self-Check Questions

1. **What's the difference between SSE and WebSocket in opencode?**
   <details>
   <summary>Answer</summary>
   SSE is used for one-way server-to-client event streaming (session updates, message deltas). WebSocket is used for bidirectional communication with PTY terminals.
   </details>

2. **Why does the SSE endpoint send heartbeat events?**
   <details>
   <summary>Answer</summary>
   To prevent proxy servers and load balancers from closing idle connections. The 10-second heartbeat keeps the connection alive.
   </details>

3. **What happens when `Bus.publish()` is called?**
   <details>
   <summary>Answer</summary>
   The event is published to both the typed PubSub (for specific subscribers) and the wildcard PubSub (for `subscribeAll`). It's also emitted to GlobalBus for cross-instance visibility.
   </details>

4. **How does a client know when to disconnect from the event stream?**
   <details>
   <summary>Answer</summary>
   The server sends an `InstanceDisposed` event when shutting down, which triggers the cleanup and closes the stream.
   </details>

5. **What's the purpose of the GlobalBus vs the regular Bus?**
   <details>
   <summary>Answer</summary>
   The regular Bus is per-instance (per-project), while GlobalBus aggregates events across all instances, useful for the `/global/event` endpoint.
   </details>

---

## Exercises

### Exercise 1: Event Tracing

When a user sends a message, trace all the events that are published:
1. What event is published when the message is created?
2. What events are published during AI streaming?
3. What event is published when the response completes?

### Exercise 2: Custom Event

Design a custom event for tracking when a user copies code from a message. Define:
1. The event type string
2. The Zod schema for properties
3. Where you would publish this event

### Exercise 3: Event Consumer

Write pseudocode for a client that:
1. Connects to the `/event` SSE endpoint
2. Filters for `message.part.delta` events
3. Aggregates streaming text by message ID
4. Handles reconnection on disconnect

---

## Further Reading

- `packages/opencode/src/bus/index.ts` - Bus implementation
- `packages/opencode/src/bus/bus-event.ts` - Event definition system
- `packages/opencode/src/server/routes/event.ts` - SSE endpoint
- `packages/opencode/src/server/routes/pty.ts` - WebSocket endpoint
- [Server-Sent Events MDN](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events)
- [WebSocket API MDN](https://developer.mozilla.org/en-US/docs/Web/API/WebSocket)
