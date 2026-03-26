# Module 06: Session System - Lesson 01: Session Lifecycle

## Learning Objectives

By the end of this lesson, you will be able to:

1. Understand what a session represents in opencode
2. Explain the Session.Info type and its fields
3. Trace session CRUD operations through the codebase
4. Understand how sessions are persisted to the database
5. Identify session events and their role in the system

---

## What is a Session?

A **session** in opencode represents a single conversation with the AI. Think of it as a chat thread that:

- Contains a sequence of messages (user prompts and assistant responses)
- Tracks metadata like title, timestamps, and cost
- Can have child sessions (subagents/tasks)
- Persists across application restarts

```
┌─────────────────────────────────────────────────────────────┐
│                         SESSION                              │
├─────────────────────────────────────────────────────────────┤
│  id: session_01HWXYZ...                                     │
│  title: "Implement user authentication"                     │
│  projectID: project_abc123                                  │
│                                                             │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐     │
│  │ User Msg 1  │───▶│ Asst Msg 1  │───▶│ User Msg 2  │ ... │
│  └─────────────┘    └─────────────┘    └─────────────┘     │
│                                                             │
│  time.created: 1710000000000                                │
│  time.updated: 1710001000000                                │
│  cost: 0.0234                                               │
└─────────────────────────────────────────────────────────────┘
```

---

## The Session.Info Type

The core data structure for a session is defined using Zod schemas. Here's the complete type:

```typescript
// From packages/opencode/src/session/index.ts

export const Info = z
  .object({
    id: SessionID.zod,                    // Unique identifier (descending ULID)
    slug: z.string(),                     // Human-readable slug
    projectID: ProjectID.zod,             // Which project this belongs to
    workspaceID: WorkspaceID.zod.optional(), // Optional workspace grouping
    directory: z.string(),                // Working directory path
    parentID: SessionID.zod.optional(),   // Parent session (for subagents)
    summary: z
      .object({
        additions: z.number(),            // Lines added
        deletions: z.number(),            // Lines deleted
        files: z.number(),                // Files changed
        diffs: Snapshot.FileDiff.array().optional(),
      })
      .optional(),
    share: z
      .object({
        url: z.string(),                  // Share URL if shared
      })
      .optional(),
    title: z.string(),                    // Session title
    version: z.string(),                  // opencode version
    time: z.object({
      created: z.number(),                // Creation timestamp
      updated: z.number(),                // Last update timestamp
      compacting: z.number().optional(),  // When compaction started
      archived: z.number().optional(),    // When archived
    }),
    permission: Permission.Ruleset.optional(), // Permission rules
    revert: z
      .object({
        messageID: MessageID.zod,
        partID: PartID.zod.optional(),
        snapshot: z.string().optional(),
        diff: z.string().optional(),
      })
      .optional(),                        // Revert state
  })
```

### Key Fields Explained

| Field | Purpose |
|-------|---------|
| `id` | Descending ULID - newer sessions sort first |
| `parentID` | Links child sessions to parent (for Task tool) |
| `summary` | Aggregated diff statistics for the session |
| `permission` | Per-session permission overrides |
| `revert` | Tracks pending revert operations |

---

## Session ID Generation

Session IDs use a **descending ULID** pattern so newer sessions appear first when sorted:

```typescript
// From packages/opencode/src/session/schema.ts

export const SessionID = Schema.String.pipe(
  Schema.brand("SessionID"),
  withStatics((s) => ({
    make: (id: string) => s.makeUnsafe(id),
    descending: (id?: string) => s.makeUnsafe(Identifier.descending("session", id)),
    zod: Identifier.schema("session").pipe(z.custom<Schema.Schema.Type<typeof s>>()),
  })),
)
```

The `descending` function creates IDs that sort in reverse chronological order, making "most recent" queries efficient.

---

## Session CRUD Operations

### Creating a Session

```typescript
// From packages/opencode/src/session/index.ts

export async function createNext(input: {
  id?: SessionID
  title?: string
  parentID?: SessionID
  workspaceID?: WorkspaceID
  directory: string
  permission?: Permission.Ruleset
}) {
  const result: Info = {
    id: SessionID.descending(input.id),
    slug: Slug.create(),
    version: Installation.VERSION,
    projectID: Instance.project.id,
    directory: input.directory,
    workspaceID: input.workspaceID,
    parentID: input.parentID,
    title: input.title ?? createDefaultTitle(!!input.parentID),
    permission: input.permission,
    time: {
      created: Date.now(),
      updated: Date.now(),
    },
  }
  log.info("created", result)

  // Emit sync event (persists to database)
  SyncEvent.run(Event.Created, { sessionID: result.id, info: result })

  // Auto-share if configured
  const cfg = await Config.get()
  if (!result.parentID && (Flag.OPENCODE_AUTO_SHARE || cfg.share === "auto")) {
    share(result.id).catch(() => {})
  }

  return result
}
```

Key points:
1. IDs are generated with `SessionID.descending()` for reverse-chronological sorting
2. A slug is created for human-readable URLs
3. The `SyncEvent.run()` call persists to the database via event sourcing
4. Auto-sharing can be configured

### Reading a Session

```typescript
export const get = fn(SessionID.zod, async (id) => {
  const row = Database.use((db) => 
    db.select().from(SessionTable).where(eq(SessionTable.id, id)).get()
  )
  if (!row) throw new NotFoundError({ message: `Session not found: ${id}` })
  return fromRow(row)
})
```

### Updating a Session

Updates are done via sync events that modify specific fields:

```typescript
export const setTitle = fn(
  z.object({
    sessionID: SessionID.zod,
    title: z.string(),
  }),
  async (input) => {
    SyncEvent.run(Event.Updated, { 
      sessionID: input.sessionID, 
      info: { title: input.title } 
    })
  },
)

export const touch = fn(SessionID.zod, async (sessionID) => {
  const time = Date.now()
  SyncEvent.run(Event.Updated, { 
    sessionID, 
    info: { time: { updated: time } } 
  })
})
```

### Deleting a Session

Deletion cascades to children and cleans up related data:

```typescript
export const remove = fn(SessionID.zod, async (sessionID) => {
  try {
    const session = await get(sessionID)
    
    // Recursively delete child sessions
    for (const child of await children(sessionID)) {
      await remove(child.id)
    }
    
    // Remove share if exists
    await unshare(sessionID).catch(() => {})

    // Emit delete event
    SyncEvent.run(Event.Deleted, { sessionID, info: session })

    // Clean up event sourcing data
    SyncEvent.remove(sessionID)
  } catch (e) {
    log.error(e)
  }
})
```

---

## Database Schema

Sessions are stored in SQLite using Drizzle ORM:

```typescript
// From packages/opencode/src/session/session.sql.ts

export const SessionTable = sqliteTable(
  "session",
  {
    id: text().$type<SessionID>().primaryKey(),
    project_id: text()
      .$type<ProjectID>()
      .notNull()
      .references(() => ProjectTable.id, { onDelete: "cascade" }),
    workspace_id: text().$type<WorkspaceID>(),
    parent_id: text().$type<SessionID>(),
    slug: text().notNull(),
    directory: text().notNull(),
    title: text().notNull(),
    version: text().notNull(),
    share_url: text(),
    summary_additions: integer(),
    summary_deletions: integer(),
    summary_files: integer(),
    summary_diffs: text({ mode: "json" }).$type<Snapshot.FileDiff[]>(),
    revert: text({ mode: "json" }).$type<{...}>(),
    permission: text({ mode: "json" }).$type<Permission.Ruleset>(),
    ...Timestamps,
    time_compacting: integer(),
    time_archived: integer(),
  },
  (table) => [
    index("session_project_idx").on(table.project_id),
    index("session_workspace_idx").on(table.workspace_id),
    index("session_parent_idx").on(table.parent_id),
  ],
)
```

### Row-to-Info Conversion

The `fromRow` function converts database rows to the `Session.Info` type:

```typescript
export function fromRow(row: SessionRow): Info {
  const summary =
    row.summary_additions !== null || row.summary_deletions !== null || row.summary_files !== null
      ? {
          additions: row.summary_additions ?? 0,
          deletions: row.summary_deletions ?? 0,
          files: row.summary_files ?? 0,
          diffs: row.summary_diffs ?? undefined,
        }
      : undefined
  const share = row.share_url ? { url: row.share_url } : undefined
  const revert = row.revert ?? undefined
  return {
    id: row.id,
    slug: row.slug,
    projectID: row.project_id,
    workspaceID: row.workspace_id ?? undefined,
    directory: row.directory,
    parentID: row.parent_id ?? undefined,
    title: row.title,
    version: row.version,
    summary,
    share,
    revert,
    permission: row.permission ?? undefined,
    time: {
      created: row.time_created,
      updated: row.time_updated,
      compacting: row.time_compacting ?? undefined,
      archived: row.time_archived ?? undefined,
    },
  }
}
```

---

## Session Events

The session system uses an event-driven architecture for state changes:

```typescript
// From packages/opencode/src/session/index.ts

export const Event = {
  // Sync events (persisted, replayed)
  Created: SyncEvent.define({
    type: "session.created",
    version: 1,
    aggregate: "sessionID",
    schema: z.object({
      sessionID: SessionID.zod,
      info: Info,
    }),
  }),
  
  Updated: SyncEvent.define({
    type: "session.updated",
    version: 1,
    aggregate: "sessionID",
    schema: z.object({
      sessionID: SessionID.zod,
      info: updateSchema(Info).extend({...}),
    }),
    busSchema: z.object({
      sessionID: SessionID.zod,
      info: Info,
    }),
  }),
  
  Deleted: SyncEvent.define({
    type: "session.deleted",
    version: 1,
    aggregate: "sessionID",
    schema: z.object({
      sessionID: SessionID.zod,
      info: Info,
    }),
  }),

  // Bus events (not persisted, real-time only)
  Diff: BusEvent.define(
    "session.diff",
    z.object({
      sessionID: SessionID.zod,
      diff: Snapshot.FileDiff.array(),
    }),
  ),
  
  Error: BusEvent.define(
    "session.error",
    z.object({
      sessionID: SessionID.zod.optional(),
      error: MessageV2.Assistant.shape.error,
    }),
  ),
}
```

### Event Types

| Event | Type | Purpose |
|-------|------|---------|
| `Created` | SyncEvent | Session created, persisted to DB |
| `Updated` | SyncEvent | Session modified, persisted to DB |
| `Deleted` | SyncEvent | Session removed, cascades cleanup |
| `Diff` | BusEvent | Real-time diff updates for UI |
| `Error` | BusEvent | Error notifications for UI |

---

## Listing Sessions

Sessions can be listed with various filters:

```typescript
export function* list(input?: {
  directory?: string
  workspaceID?: WorkspaceID
  roots?: boolean        // Only top-level sessions (no children)
  start?: number         // Filter by time_updated
  search?: string        // Title search
  limit?: number
}) {
  const project = Instance.project
  const conditions = [eq(SessionTable.project_id, project.id)]

  if (WorkspaceContext.workspaceID) {
    conditions.push(eq(SessionTable.workspace_id, WorkspaceContext.workspaceID))
  }
  if (input?.directory) {
    conditions.push(eq(SessionTable.directory, input.directory))
  }
  if (input?.roots) {
    conditions.push(isNull(SessionTable.parent_id))
  }
  // ... more conditions

  const rows = Database.use((db) =>
    db
      .select()
      .from(SessionTable)
      .where(and(...conditions))
      .orderBy(desc(SessionTable.time_updated))
      .limit(limit)
      .all(),
  )
  for (const row of rows) {
    yield fromRow(row)
  }
}
```

---

## Session Forking

Sessions can be forked to create a copy with a new conversation branch:

```typescript
export const fork = fn(
  z.object({
    sessionID: SessionID.zod,
    messageID: MessageID.zod.optional(), // Fork from this point
  }),
  async (input) => {
    const original = await get(input.sessionID)
    if (!original) throw new Error("session not found")
    
    const title = getForkedTitle(original.title) // "Title (fork #1)"
    const session = await createNext({
      directory: Instance.directory,
      workspaceID: original.workspaceID,
      title,
    })
    
    // Copy messages up to the fork point
    const msgs = await messages({ sessionID: input.sessionID })
    const idMap = new Map<string, MessageID>()

    for (const msg of msgs) {
      if (input.messageID && msg.info.id >= input.messageID) break
      const newID = MessageID.ascending()
      idMap.set(msg.info.id, newID)

      // Clone message with new IDs
      const cloned = await updateMessage({
        ...msg.info,
        sessionID: session.id,
        id: newID,
        ...(parentID && { parentID }),
      })

      // Clone parts
      for (const part of msg.parts) {
        await updatePart({
          ...part,
          id: PartID.ascending(),
          messageID: cloned.id,
          sessionID: session.id,
        })
      }
    }
    return session
  },
)
```

---

## Self-Check Questions

1. **Why do session IDs use descending ULIDs instead of ascending?**
   <details>
   <summary>Answer</summary>
   Descending ULIDs ensure newer sessions sort first, making "most recent" queries efficient without needing to reverse sort.
   </details>

2. **What's the difference between SyncEvent and BusEvent?**
   <details>
   <summary>Answer</summary>
   SyncEvents are persisted to the database and can be replayed. BusEvents are ephemeral, used only for real-time notifications to the UI.
   </details>

3. **How does session deletion handle child sessions?**
   <details>
   <summary>Answer</summary>
   The `remove` function recursively deletes all child sessions before deleting the parent, ensuring no orphaned sessions remain.
   </details>

4. **What triggers auto-sharing of a session?**
   <details>
   <summary>Answer</summary>
   Auto-sharing is triggered when creating a top-level session (no parentID) and either `OPENCODE_AUTO_SHARE` flag is set or `cfg.share === "auto"`.
   </details>

---

## Exercises

1. **Trace a session creation**: Set a breakpoint in `createNext` and trace the flow from user prompt to database persistence.

2. **Examine the event log**: Look at the sync event storage to see how session events are recorded and replayed.

3. **Fork a session**: Use the API to fork an existing session and verify the message copying behavior.

---

## Further Reading

- `packages/opencode/src/sync/index.ts` - SyncEvent implementation
- `packages/opencode/src/bus/index.ts` - Bus event system
- `packages/opencode/src/storage/db.ts` - Database layer
- `packages/opencode/src/id/id.ts` - Identifier generation
