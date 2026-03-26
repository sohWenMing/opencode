# Lesson 01: Drizzle ORM

## Learning Objectives

By the end of this lesson, you will be able to:

1. Explain what Drizzle ORM is and its design philosophy
2. Understand why opencode chose Drizzle over alternatives
3. Identify the key Drizzle packages and their purposes
4. Write basic Drizzle queries using the opencode patterns
5. Understand how Drizzle integrates with SQLite

## What is Drizzle ORM?

Drizzle is a TypeScript ORM (Object-Relational Mapping) that provides:

- **Type-safe queries** - SQL queries that are checked at compile time
- **SQL-like syntax** - Familiar SQL patterns, not a custom query language
- **Lightweight** - Minimal runtime overhead, no heavy abstractions
- **Schema-as-code** - Database structure defined in TypeScript

```
┌─────────────────────────────────────────────────────────────┐
│                    Traditional ORM                           │
│  TypeScript → Custom Query Language → SQL → Database        │
│  (e.g., Prisma, TypeORM)                                    │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                      Drizzle ORM                             │
│  TypeScript → SQL-like API → SQL → Database                 │
│  (Minimal abstraction, maximum type safety)                 │
└─────────────────────────────────────────────────────────────┘
```

## Why opencode Uses Drizzle

### 1. Type Safety Without Code Generation

Unlike Prisma which requires a build step to generate types, Drizzle infers types directly from your schema definitions:

```typescript
// Schema definition IS the type definition
export const ProjectTable = sqliteTable("project", {
  id: text().$type<ProjectID>().primaryKey(),
  worktree: text().notNull(),
  vcs: text(),
})

// Types are inferred automatically
type Project = typeof ProjectTable.$inferSelect
// { id: ProjectID; worktree: string; vcs: string | null }
```

### 2. SQL-Like Syntax

Drizzle queries read like SQL, making them familiar and predictable:

```typescript
// Drizzle
db.select()
  .from(SessionTable)
  .where(eq(SessionTable.project_id, projectID))
  .orderBy(desc(SessionTable.time_created))
  .all()

// Equivalent SQL
// SELECT * FROM session
// WHERE project_id = ?
// ORDER BY time_created DESC
```

### 3. Lightweight Runtime

Drizzle has minimal runtime overhead:

- No query parsing at runtime
- No heavy ORM abstractions
- Direct SQL generation
- Small bundle size

### 4. SQLite Compatibility

Drizzle has first-class SQLite support, which is important for opencode's embedded database approach:

```typescript
import { sqliteTable, text, integer } from "drizzle-orm/sqlite-core"
```

## Drizzle Packages

opencode uses two main Drizzle packages:

### drizzle-orm

The core ORM library for runtime queries:

```typescript
// packages/opencode/src/storage/db.ts
export * from "drizzle-orm"  // Re-exports eq, and, or, desc, etc.
```

Key exports:
- `eq`, `ne`, `lt`, `gt`, `lte`, `gte` - Comparison operators
- `and`, `or`, `not` - Logical operators
- `inArray`, `notInArray` - Array operators
- `desc`, `asc` - Ordering

### drizzle-kit

The CLI tool for migrations (development dependency):

```typescript
// packages/opencode/drizzle.config.ts
import { defineConfig } from "drizzle-kit"

export default defineConfig({
  dialect: "sqlite",
  schema: "./src/**/*.sql.ts",
  out: "./migration",
})
```

## Database Initialization

opencode initializes SQLite with specific pragmas for optimal performance:

```typescript
// packages/opencode/src/storage/db.ts (lines 85-96)
export const Client = lazy(() => {
  const db = init(Path)

  db.run("PRAGMA journal_mode = WAL")      // Write-Ahead Logging
  db.run("PRAGMA synchronous = NORMAL")    // Balance safety/speed
  db.run("PRAGMA busy_timeout = 5000")     // Wait 5s on locks
  db.run("PRAGMA cache_size = -64000")     // 64MB cache
  db.run("PRAGMA foreign_keys = ON")       // Enforce FK constraints
  db.run("PRAGMA wal_checkpoint(PASSIVE)") // Checkpoint WAL file

  // Apply migrations...
  return db
})
```

### SQLite Pragmas Explained

| Pragma | Value | Purpose |
|--------|-------|---------|
| `journal_mode` | WAL | Enables concurrent reads during writes |
| `synchronous` | NORMAL | Good durability without excessive fsync |
| `busy_timeout` | 5000 | Retry for 5 seconds if database is locked |
| `cache_size` | -64000 | Use 64MB of memory for page cache |
| `foreign_keys` | ON | Enforce referential integrity |

## Runtime Adapters

opencode supports both Bun and Node.js runtimes with different SQLite bindings:

```
┌─────────────────────────────────────────────────────────────┐
│                    db.ts (common logic)                      │
│  - Migration handling                                        │
│  - Transaction management                                    │
│  - Query helpers                                             │
└─────────────────────────────────────────────────────────────┘
                              │
              ┌───────────────┴───────────────┐
              ▼                               ▼
┌─────────────────────────┐     ┌─────────────────────────┐
│      db.bun.ts          │     │      db.node.ts         │
│  import { Database }    │     │  import { DatabaseSync }│
│    from "bun:sqlite"    │     │    from "node:sqlite"   │
│  drizzle/bun-sqlite     │     │  drizzle/node-sqlite    │
└─────────────────────────┘     └─────────────────────────┘
```

### Bun Adapter

```typescript
// packages/opencode/src/storage/db.bun.ts
import { Database } from "bun:sqlite"
import { drizzle } from "drizzle-orm/bun-sqlite"

export function init(path: string) {
  const sqlite = new Database(path, { create: true })
  const db = drizzle({ client: sqlite })
  return db
}
```

### Node.js Adapter

```typescript
// packages/opencode/src/storage/db.node.ts
import { DatabaseSync } from "node:sqlite"
import { drizzle } from "drizzle-orm/node-sqlite"

export function init(path: string) {
  const sqlite = new DatabaseSync(path)
  const db = drizzle({ client: sqlite })
  return db
}
```

## Database Access Pattern

opencode provides a `Database.use()` helper for all database operations:

```typescript
// packages/opencode/src/storage/db.ts (lines 130-142)
export function use<T>(callback: (trx: TxOrDb) => T): T {
  try {
    return callback(ctx.use().tx)  // Use existing transaction if available
  } catch (err) {
    if (err instanceof Context.NotFound) {
      const effects: (() => void | Promise<void>)[] = []
      const result = ctx.provide({ effects, tx: Client() }, () => callback(Client()))
      for (const effect of effects) effect()
      return result
    }
    throw err
  }
}
```

### Usage Examples

**Simple Query:**

```typescript
// packages/opencode/src/project/project.ts
const row = Database.use((db) =>
  db.select()
    .from(ProjectTable)
    .where(eq(ProjectTable.id, id))
    .get()
)
```

**Query with Multiple Tables:**

```typescript
// packages/opencode/src/session/message-v2.ts (lines 553-560)
const partRows = Database.use((db) =>
  db
    .select()
    .from(PartTable)
    .where(inArray(PartTable.message_id, ids))
    .orderBy(PartTable.message_id, PartTable.id)
    .all(),
)
```

## Transactions

For operations that need atomicity, use `Database.transaction()`:

```typescript
// packages/opencode/src/storage/db.ts (lines 154-176)
export function transaction<T>(
  callback: (tx: TxOrDb) => NotPromise<T>,
  options?: {
    behavior?: "deferred" | "immediate" | "exclusive"
  },
): NotPromise<T>
```

### Transaction Example

```typescript
// packages/opencode/src/sync/index.ts (lines 117-159)
Database.transaction((tx) => {
  projector(tx, event.data)

  tx.insert(EventSequenceTable)
    .values({
      aggregate_id: event.aggregateID,
      seq: event.seq,
    })
    .onConflictDoUpdate({
      target: EventSequenceTable.aggregate_id,
      set: { seq: event.seq },
    })
    .run()

  tx.insert(EventTable)
    .values({
      id: event.id,
      seq: event.seq,
      aggregate_id: event.aggregateID,
      type: versionedType(def.type, def.version),
      data: event.data as Record<string, unknown>,
    })
    .run()

  Database.effect(() => {
    Bus.emit("event", { def, event })
  })
})
```

### Transaction Behaviors

| Behavior | Description |
|----------|-------------|
| `deferred` | Lock acquired on first write (default) |
| `immediate` | Lock acquired immediately |
| `exclusive` | Exclusive lock, no concurrent readers |

## Side Effects in Transactions

The `Database.effect()` function defers side effects until after the transaction commits:

```typescript
// packages/opencode/src/storage/db.ts (lines 144-150)
export function effect(fn: () => any | Promise<any>) {
  try {
    ctx.use().effects.push(fn)  // Queue for after commit
  } catch {
    fn()  // No transaction, run immediately
  }
}
```

This ensures events are only published after data is safely persisted.

## Common Query Patterns

### Select All

```typescript
db.select().from(SessionTable).all()
```

### Select with Where

```typescript
db.select()
  .from(SessionTable)
  .where(eq(SessionTable.project_id, projectID))
  .all()
```

### Select Single Row

```typescript
db.select()
  .from(ProjectTable)
  .where(eq(ProjectTable.id, id))
  .get()  // Returns single row or undefined
```

### Insert

```typescript
db.insert(SessionTable)
  .values({
    id: sessionID,
    project_id: projectID,
    title: "New Session",
    // ...
  })
  .run()
```

### Update

```typescript
db.update(SessionTable)
  .set({ title: "Updated Title" })
  .where(eq(SessionTable.id, sessionID))
  .run()
```

### Delete

```typescript
db.delete(SessionTable)
  .where(eq(SessionTable.id, sessionID))
  .run()
```

### Upsert (Insert or Update)

```typescript
db.insert(EventSequenceTable)
  .values({ aggregate_id: id, seq: 1 })
  .onConflictDoUpdate({
    target: EventSequenceTable.aggregate_id,
    set: { seq: 1 },
  })
  .run()
```

## Self-Check Questions

1. What are the two main Drizzle packages and their purposes?
2. Why does opencode use `PRAGMA journal_mode = WAL`?
3. What is the difference between `.get()` and `.all()` in Drizzle queries?
4. Why does `Database.effect()` exist? What problem does it solve?
5. How does opencode support both Bun and Node.js runtimes for SQLite?

## Exercises

### Exercise 1: Trace a Query

Find a `Database.use()` call in `packages/opencode/src/session/session.ts` and:
1. Identify the table being queried
2. List the conditions in the WHERE clause
3. Determine what type the query returns

### Exercise 2: Understand Transaction Flow

In `packages/opencode/src/sync/index.ts`, trace the transaction at line 117:
1. What tables are modified?
2. What side effect is deferred with `Database.effect()`?
3. Why is this a transaction instead of separate queries?

### Exercise 3: Compare Adapters

Compare `db.bun.ts` and `db.node.ts`:
1. What is the key difference in the SQLite import?
2. Why does opencode need both adapters?
3. How does the build system choose which adapter to use?

## Further Reading

- [Drizzle ORM Documentation](https://orm.drizzle.team/)
- [SQLite WAL Mode](https://www.sqlite.org/wal.html)
- [SQLite Pragma Reference](https://www.sqlite.org/pragma.html)
- [Bun SQLite Documentation](https://bun.sh/docs/api/sqlite)
