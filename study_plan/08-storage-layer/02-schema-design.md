# Lesson 02: Schema Design

## Learning Objectives

By the end of this lesson, you will be able to:

1. Understand the database schema structure in opencode
2. Identify all tables and their purposes
3. Explain the relationships between tables
4. Read and write Drizzle schema definitions
5. Understand the `.sql.ts` file pattern

## Schema Overview

opencode's database schema is distributed across multiple `.sql.ts` files, each defining tables for a specific domain:

```
┌─────────────────────────────────────────────────────────────┐
│                    schema.ts (exports)                       │
│  Re-exports all tables from domain-specific files           │
└─────────────────────────────────────────────────────────────┘
                              │
        ┌─────────────────────┼─────────────────────┐
        ▼                     ▼                     ▼
┌───────────────┐   ┌───────────────┐   ┌───────────────┐
│ session.sql.ts│   │ project.sql.ts│   │ account.sql.ts│
│ - Session     │   │ - Project     │   │ - Account     │
│ - Message     │   │               │   │ - AccountState│
│ - Part        │   │               │   │               │
│ - Todo        │   │               │   │               │
│ - Permission  │   │               │   │               │
└───────────────┘   └───────────────┘   └───────────────┘
        │                     │                     │
        ▼                     ▼                     ▼
┌───────────────┐   ┌───────────────┐   ┌───────────────┐
│  share.sql.ts │   │workspace.sql.ts│   │  event.sql.ts │
│ - SessionShare│   │ - Workspace   │   │ - Event       │
│               │   │               │   │ - EventSeq    │
└───────────────┘   └───────────────┘   └───────────────┘
```

## The Schema Export Pattern

All tables are re-exported from a central schema file:

```typescript
// packages/opencode/src/storage/schema.ts
export { AccountTable, AccountStateTable, ControlAccountTable } from "../account/account.sql"
export { ProjectTable } from "../project/project.sql"
export { SessionTable, MessageTable, PartTable, TodoTable, PermissionTable } from "../session/session.sql"
export { SessionShareTable } from "../share/share.sql"
export { WorkspaceTable } from "../control-plane/workspace.sql"
```

## Common Schema Patterns

### Timestamps Mixin

A reusable pattern for created/updated timestamps:

```typescript
// packages/opencode/src/storage/schema.sql.ts
import { integer } from "drizzle-orm/sqlite-core"

export const Timestamps = {
  time_created: integer()
    .notNull()
    .$default(() => Date.now()),
  time_updated: integer()
    .notNull()
    .$onUpdate(() => Date.now()),
}
```

Usage in tables:

```typescript
export const SessionTable = sqliteTable("session", {
  id: text().primaryKey(),
  // ... other columns
  ...Timestamps,  // Spreads time_created and time_updated
})
```

### Branded Types

Tables use branded types for type-safe IDs:

```typescript
// Type-safe ID that can't be confused with other string IDs
id: text().$type<SessionID>().primaryKey(),
project_id: text().$type<ProjectID>().notNull(),
```

This prevents accidentally passing a `SessionID` where a `ProjectID` is expected.

## Table Definitions

### ProjectTable

The root entity representing a code project:

```typescript
// packages/opencode/src/project/project.sql.ts
export const ProjectTable = sqliteTable("project", {
  id: text().$type<ProjectID>().primaryKey(),
  worktree: text().notNull(),           // Path to project root
  vcs: text(),                          // Version control system (e.g., "git")
  name: text(),                         // Display name
  icon_url: text(),                     // Project icon URL
  icon_color: text(),                   // Icon color
  ...Timestamps,
  time_initialized: integer(),          // When project was first opened
  sandboxes: text({ mode: "json" })     // JSON array of sandbox IDs
    .notNull()
    .$type<string[]>(),
  commands: text({ mode: "json" })      // Custom commands config
    .$type<{ start?: string }>(),
})
```

### SessionTable

Chat sessions within a project:

```typescript
// packages/opencode/src/session/session.sql.ts (lines 14-44)
export const SessionTable = sqliteTable(
  "session",
  {
    id: text().$type<SessionID>().primaryKey(),
    project_id: text()
      .$type<ProjectID>()
      .notNull()
      .references(() => ProjectTable.id, { onDelete: "cascade" }),
    workspace_id: text().$type<WorkspaceID>(),
    parent_id: text().$type<SessionID>(),     // For branched sessions
    slug: text().notNull(),                    // URL-friendly identifier
    directory: text().notNull(),               // Working directory
    title: text().notNull(),                   // Session title
    version: text().notNull(),                 // Schema version
    share_url: text(),                         // Public share URL
    summary_additions: integer(),              // Lines added
    summary_deletions: integer(),              // Lines deleted
    summary_files: integer(),                  // Files changed
    summary_diffs: text({ mode: "json" })      // Detailed diff data
      .$type<Snapshot.FileDiff[]>(),
    revert: text({ mode: "json" })             // Revert information
      .$type<{ messageID: MessageID; partID?: PartID; snapshot?: string; diff?: string }>(),
    permission: text({ mode: "json" })         // Session-specific permissions
      .$type<Permission.Ruleset>(),
    ...Timestamps,
    time_compacting: integer(),                // When compaction started
    time_archived: integer(),                  // When session was archived
  },
  (table) => [
    index("session_project_idx").on(table.project_id),
    index("session_workspace_idx").on(table.workspace_id),
    index("session_parent_idx").on(table.parent_id),
  ],
)
```

### MessageTable

Messages within a session:

```typescript
// packages/opencode/src/session/session.sql.ts (lines 46-58)
export const MessageTable = sqliteTable(
  "message",
  {
    id: text().$type<MessageID>().primaryKey(),
    session_id: text()
      .$type<SessionID>()
      .notNull()
      .references(() => SessionTable.id, { onDelete: "cascade" }),
    ...Timestamps,
    data: text({ mode: "json" }).notNull().$type<InfoData>(),  // Message metadata
  },
  (table) => [
    index("message_session_time_created_id_idx")
      .on(table.session_id, table.time_created, table.id)
  ],
)
```

### PartTable

Individual parts of a message (text, tool calls, tool results):

```typescript
// packages/opencode/src/session/session.sql.ts (lines 60-76)
export const PartTable = sqliteTable(
  "part",
  {
    id: text().$type<PartID>().primaryKey(),
    message_id: text()
      .$type<MessageID>()
      .notNull()
      .references(() => MessageTable.id, { onDelete: "cascade" }),
    session_id: text().$type<SessionID>().notNull(),  // Denormalized for queries
    ...Timestamps,
    data: text({ mode: "json" }).notNull().$type<PartData>(),  // Part content
  },
  (table) => [
    index("part_message_id_id_idx").on(table.message_id, table.id),
    index("part_session_idx").on(table.session_id),
  ],
)
```

### TodoTable

Task tracking within sessions:

```typescript
// packages/opencode/src/session/session.sql.ts (lines 78-95)
export const TodoTable = sqliteTable(
  "todo",
  {
    session_id: text()
      .$type<SessionID>()
      .notNull()
      .references(() => SessionTable.id, { onDelete: "cascade" }),
    content: text().notNull(),      // Todo description
    status: text().notNull(),       // pending, in_progress, completed, cancelled
    priority: text().notNull(),     // Priority level
    position: integer().notNull(),  // Order in list
    ...Timestamps,
  },
  (table) => [
    primaryKey({ columns: [table.session_id, table.position] }),
    index("todo_session_idx").on(table.session_id),
  ],
)
```

### PermissionTable

Project-level permission rules:

```typescript
// packages/opencode/src/session/session.sql.ts (lines 97-103)
export const PermissionTable = sqliteTable("permission", {
  project_id: text()
    .primaryKey()
    .references(() => ProjectTable.id, { onDelete: "cascade" }),
  ...Timestamps,
  data: text({ mode: "json" }).notNull().$type<Permission.Ruleset>(),
})
```

### AccountTable

User account information:

```typescript
// packages/opencode/src/account/account.sql.ts (lines 6-14)
export const AccountTable = sqliteTable("account", {
  id: text().$type<AccountID>().primaryKey(),
  email: text().notNull(),
  url: text().notNull(),                              // Auth server URL
  access_token: text().$type<AccessToken>().notNull(),
  refresh_token: text().$type<RefreshToken>().notNull(),
  token_expiry: integer(),
  ...Timestamps,
})
```

### AccountStateTable

Singleton table for active account state:

```typescript
// packages/opencode/src/account/account.sql.ts (lines 16-22)
export const AccountStateTable = sqliteTable("account_state", {
  id: integer().primaryKey(),  // Always 1 (singleton)
  active_account_id: text()
    .$type<AccountID>()
    .references(() => AccountTable.id, { onDelete: "set null" }),
  active_org_id: text().$type<OrgID>(),
})
```

### WorkspaceTable

Workspace configurations (git branches, worktrees):

```typescript
// packages/opencode/src/control-plane/workspace.sql.ts
export const WorkspaceTable = sqliteTable("workspace", {
  id: text().$type<WorkspaceID>().primaryKey(),
  type: text().notNull(),        // Workspace type
  branch: text(),                // Git branch
  name: text(),                  // Display name
  directory: text(),             // Working directory
  extra: text({ mode: "json" }), // Additional config
  project_id: text()
    .$type<ProjectID>()
    .notNull()
    .references(() => ProjectTable.id, { onDelete: "cascade" }),
})
```

### SessionShareTable

Shared session information:

```typescript
// packages/opencode/src/share/share.sql.ts
export const SessionShareTable = sqliteTable("session_share", {
  session_id: text()
    .primaryKey()
    .references(() => SessionTable.id, { onDelete: "cascade" }),
  id: text().notNull(),      // Share ID
  secret: text().notNull(),  // Access secret
  url: text().notNull(),     // Public URL
  ...Timestamps,
})
```

### Event Tables (Sync System)

For event sourcing and synchronization:

```typescript
// packages/opencode/src/sync/event.sql.ts
export const EventSequenceTable = sqliteTable("event_sequence", {
  aggregate_id: text().notNull().primaryKey(),
  seq: integer().notNull(),  // Current sequence number
})

export const EventTable = sqliteTable("event", {
  id: text().primaryKey(),
  aggregate_id: text()
    .notNull()
    .references(() => EventSequenceTable.aggregate_id, { onDelete: "cascade" }),
  seq: integer().notNull(),
  type: text().notNull(),
  data: text({ mode: "json" }).$type<Record<string, unknown>>().notNull(),
})
```

## Entity Relationship Diagram

```
┌─────────────┐
│   Account   │
├─────────────┤
│ id (PK)     │
│ email       │
│ url         │
│ tokens...   │
└─────────────┘
       │
       ▼
┌─────────────┐
│AccountState │
├─────────────┤
│ id (PK)     │
│ active_id   │──▶ Account
│ active_org  │
└─────────────┘

┌─────────────┐     ┌─────────────┐
│   Project   │◀────│  Workspace  │
├─────────────┤     ├─────────────┤
│ id (PK)     │     │ id (PK)     │
│ worktree    │     │ project_id  │──▶ Project
│ vcs         │     │ branch      │
│ name        │     │ type        │
└─────────────┘     └─────────────┘
       │
       ├────────────────────────────┐
       ▼                            ▼
┌─────────────┐              ┌─────────────┐
│  Permission │              │   Session   │
├─────────────┤              ├─────────────┤
│ project_id  │──▶ Project   │ id (PK)     │
│ data        │              │ project_id  │──▶ Project
└─────────────┘              │ workspace_id│──▶ Workspace
                             │ parent_id   │──▶ Session (self)
                             │ title       │
                             └─────────────┘
                                    │
       ┌────────────────────────────┼────────────────┐
       ▼                            ▼                ▼
┌─────────────┐              ┌─────────────┐  ┌─────────────┐
│   Message   │              │    Todo     │  │SessionShare │
├─────────────┤              ├─────────────┤  ├─────────────┤
│ id (PK)     │              │ session_id  │  │ session_id  │──▶ Session
│ session_id  │──▶ Session   │ position    │  │ id          │
│ data        │              │ content     │  │ secret      │
└─────────────┘              │ status      │  │ url         │
       │                     └─────────────┘  └─────────────┘
       ▼
┌─────────────┐
│    Part     │
├─────────────┤
│ id (PK)     │
│ message_id  │──▶ Message
│ session_id  │──▶ Session (denormalized)
│ data        │
└─────────────┘
```

## Column Types and Constraints

### SQLite Column Types in Drizzle

| Drizzle Type | SQLite Type | Usage |
|--------------|-------------|-------|
| `text()` | TEXT | Strings, JSON |
| `integer()` | INTEGER | Numbers, timestamps, booleans |
| `real()` | REAL | Floating point |
| `blob()` | BLOB | Binary data |

### Common Modifiers

```typescript
// Primary key
id: text().primaryKey()

// Not null constraint
worktree: text().notNull()

// Foreign key with cascade delete
project_id: text()
  .references(() => ProjectTable.id, { onDelete: "cascade" })

// Default value
time_created: integer().$default(() => Date.now())

// Update hook
time_updated: integer().$onUpdate(() => Date.now())

// JSON mode (stored as TEXT, parsed as JSON)
data: text({ mode: "json" }).$type<MyType>()

// Boolean mode (stored as INTEGER 0/1)
active: integer({ mode: "boolean" })
```

### Index Definitions

```typescript
// Single column index
index("session_project_idx").on(table.project_id)

// Composite index
index("message_session_time_created_id_idx")
  .on(table.session_id, table.time_created, table.id)

// Composite primary key
primaryKey({ columns: [table.session_id, table.position] })
```

## Naming Conventions

From the AGENTS.md style guide:

| Element | Convention | Example |
|---------|------------|---------|
| Table names | snake_case | `session_share` |
| Column names | snake_case | `project_id` |
| Foreign keys | `<entity>_id` | `session_id` |
| Indexes | `<table>_<column>_idx` | `session_project_idx` |

## JSON Columns

opencode stores complex data as JSON in TEXT columns:

```typescript
// Type-safe JSON column
data: text({ mode: "json" }).notNull().$type<InfoData>()

// Query JSON data (SQLite JSON functions)
db.select()
  .from(SessionTable)
  .where(sql`json_extract(${SessionTable.summary_diffs}, '$[0].file') = 'README.md'`)
```

## Self-Check Questions

1. Why does opencode use branded types like `SessionID` instead of plain strings?
2. What is the purpose of the `Timestamps` mixin?
3. Why is `session_id` denormalized in the `PartTable`?
4. What happens to sessions when a project is deleted?
5. How does the `mode: "json"` option work in Drizzle?

## Exercises

### Exercise 1: Schema Analysis

For the `MessageTable`:
1. List all columns and their types
2. Identify the foreign key relationship
3. Explain the index and why it includes three columns

### Exercise 2: Design a New Table

Design a schema for a `Bookmark` table that:
- Belongs to a session
- Has a title and description
- References a specific message
- Includes timestamps

Write the Drizzle schema definition.

### Exercise 3: Trace Relationships

Starting from a `Part`:
1. How do you find the parent `Message`?
2. How do you find the parent `Session`?
3. How do you find the parent `Project`?
4. What indexes would be used for each query?

## Further Reading

- [Drizzle Schema Documentation](https://orm.drizzle.team/docs/sql-schema-declaration)
- [SQLite Data Types](https://www.sqlite.org/datatype3.html)
- [SQLite Foreign Keys](https://www.sqlite.org/foreignkeys.html)
- [Database Normalization](https://en.wikipedia.org/wiki/Database_normalization)
