# Lesson 03: Database Migrations

## Learning Objectives

By the end of this lesson, you will be able to:

1. Explain what database migrations are and why they're needed
2. Understand the migration folder structure in opencode
3. Create new migrations using Drizzle Kit
4. Understand how migrations are applied at runtime
5. Read and interpret migration SQL files

## What Are Database Migrations?

Database migrations are versioned changes to your database schema. They allow you to:

- **Track schema changes** over time in version control
- **Apply changes safely** to existing databases
- **Roll forward** to new schema versions
- **Maintain consistency** across development, testing, and production

```
┌─────────────────────────────────────────────────────────────┐
│                    Schema Evolution                          │
├─────────────────────────────────────────────────────────────┤
│  v1: Initial schema (project, session, message)             │
│       │                                                      │
│       ▼                                                      │
│  v2: Add workspace table                                     │
│       │                                                      │
│       ▼                                                      │
│  v3: Add session_message_cursor index                        │
│       │                                                      │
│       ▼                                                      │
│  v4: Add event tables for sync                               │
└─────────────────────────────────────────────────────────────┘
```

## Migration Folder Structure

opencode stores migrations in `packages/opencode/migration/`:

```
migration/
├── 20260127222353_familiar_lady_ursula/
│   ├── migration.sql      # SQL statements to apply
│   └── snapshot.json      # Schema snapshot after migration
├── 20260211171708_add_project_commands/
│   ├── migration.sql
│   └── snapshot.json
├── 20260213144116_wakeful_the_professor/
│   ├── migration.sql
│   └── snapshot.json
├── 20260225215848_workspace/
│   ├── migration.sql
│   └── snapshot.json
├── 20260227213759_add_session_workspace_id/
│   ├── migration.sql
│   └── snapshot.json
├── 20260228203230_blue_harpoon/
│   ├── migration.sql
│   └── snapshot.json
├── 20260303231226_add_workspace_fields/
│   ├── migration.sql
│   └── snapshot.json
├── 20260309230000_move_org_to_state/
│   ├── migration.sql
│   └── snapshot.json
├── 20260312043431_session_message_cursor/
│   ├── migration.sql
│   └── snapshot.json
└── 20260323234822_events/
    ├── migration.sql
    └── snapshot.json
```

### Naming Convention

```
YYYYMMDDHHMMSS_slug/
└── 20260127222353_familiar_lady_ursula/
    │              │
    │              └── Human-readable slug
    └── Timestamp (UTC)
```

The timestamp ensures migrations are applied in chronological order.

## Migration File Contents

### migration.sql

Contains the SQL statements to apply:

```sql
-- packages/opencode/migration/20260127222353_familiar_lady_ursula/migration.sql
CREATE TABLE `project` (
	`id` text PRIMARY KEY,
	`worktree` text NOT NULL,
	`vcs` text,
	`name` text,
	`icon_url` text,
	`icon_color` text,
	`time_created` integer NOT NULL,
	`time_updated` integer NOT NULL,
	`time_initialized` integer,
	`sandboxes` text NOT NULL
);
--> statement-breakpoint
CREATE TABLE `message` (
	`id` text PRIMARY KEY,
	`session_id` text NOT NULL,
	`time_created` integer NOT NULL,
	`time_updated` integer NOT NULL,
	`data` text NOT NULL,
	CONSTRAINT `fk_message_session_id_session_id_fk` 
	  FOREIGN KEY (`session_id`) REFERENCES `session`(`id`) ON DELETE CASCADE
);
--> statement-breakpoint
CREATE INDEX `message_session_idx` ON `message` (`session_id`);
```

### Statement Breakpoints

The `--> statement-breakpoint` comment separates individual SQL statements. This is important because:

1. SQLite executes statements one at a time
2. Errors can be traced to specific statements
3. Some statements must complete before others can run

### snapshot.json

Contains a complete snapshot of the schema after this migration. This is used by Drizzle Kit to:

- Generate the next migration by comparing current schema to snapshot
- Validate that migrations produce the expected schema

## Creating New Migrations

### Step 1: Modify Schema Files

Edit the relevant `.sql.ts` file:

```typescript
// packages/opencode/src/session/session.sql.ts
export const SessionTable = sqliteTable("session", {
  // ... existing columns
  new_column: text(),  // Add new column
})
```

### Step 2: Generate Migration

Run Drizzle Kit from the package directory:

```bash
cd packages/opencode
bun run db generate --name add_new_column
```

This command:
1. Reads all `*.sql.ts` files (per `drizzle.config.ts`)
2. Compares to the latest snapshot
3. Generates SQL to transform the schema
4. Creates a new migration folder

### Drizzle Configuration

```typescript
// packages/opencode/drizzle.config.ts
import { defineConfig } from "drizzle-kit"

export default defineConfig({
  dialect: "sqlite",
  schema: "./src/**/*.sql.ts",  // Find all schema files
  out: "./migration",           // Output directory
  dbCredentials: {
    url: "/home/thdxr/.local/share/opencode/opencode.db",
  },
})
```

## How Migrations Are Applied

opencode applies migrations at startup in `db.ts`:

```typescript
// packages/opencode/src/storage/db.ts (lines 65-83)
function migrations(dir: string): Journal {
  const dirs = readdirSync(dir, { withFileTypes: true })
    .filter((entry) => entry.isDirectory())
    .map((entry) => entry.name)

  const sql = dirs
    .map((name) => {
      const file = path.join(dir, name, "migration.sql")
      if (!existsSync(file)) return
      return {
        sql: readFileSync(file, "utf-8"),
        timestamp: time(name),  // Parse timestamp from folder name
        name,
      }
    })
    .filter(Boolean) as Journal

  return sql.sort((a, b) => a.timestamp - b.timestamp)  // Apply in order
}
```

### Migration Application Flow

```
┌─────────────────────────────────────────────────────────────┐
│                    Database.Client()                         │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│              1. Open SQLite database                         │
│                 init(Path)                                   │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│              2. Configure pragmas                            │
│                 PRAGMA journal_mode = WAL                    │
│                 PRAGMA foreign_keys = ON                     │
│                 ...                                          │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│              3. Load migrations                              │
│                 - Read migration folders                     │
│                 - Sort by timestamp                          │
│                 - Parse SQL files                            │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│              4. Apply migrations                             │
│                 migrate(db, entries)                         │
│                 - Drizzle tracks applied migrations          │
│                 - Only runs new migrations                   │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│              5. Return database client                       │
└─────────────────────────────────────────────────────────────┘
```

### Bundled vs Development Migrations

```typescript
// packages/opencode/src/storage/db.ts (lines 97-113)
const entries =
  typeof OPENCODE_MIGRATIONS !== "undefined"
    ? OPENCODE_MIGRATIONS           // Production: bundled migrations
    : migrations(path.join(import.meta.dirname, "../../migration"))  // Dev: read from disk

if (entries.length > 0) {
  log.info("applying migrations", {
    count: entries.length,
    mode: typeof OPENCODE_MIGRATIONS !== "undefined" ? "bundled" : "dev",
  })
  migrate(db, entries)
}
```

In production builds, migrations are bundled into the binary via `OPENCODE_MIGRATIONS`.

## Migration Examples

### Initial Schema Migration

The first migration creates all core tables:

```sql
-- packages/opencode/migration/20260127222353_familiar_lady_ursula/migration.sql
CREATE TABLE `project` (
	`id` text PRIMARY KEY,
	`worktree` text NOT NULL,
	`vcs` text,
	`name` text,
	`icon_url` text,
	`icon_color` text,
	`time_created` integer NOT NULL,
	`time_updated` integer NOT NULL,
	`time_initialized` integer,
	`sandboxes` text NOT NULL
);
--> statement-breakpoint
CREATE TABLE `session` (
	`id` text PRIMARY KEY,
	`project_id` text NOT NULL,
	`parent_id` text,
	`slug` text NOT NULL,
	`directory` text NOT NULL,
	`title` text NOT NULL,
	`version` text NOT NULL,
	`share_url` text,
	`summary_additions` integer,
	`summary_deletions` integer,
	`summary_files` integer,
	`summary_diffs` text,
	`revert` text,
	`permission` text,
	`time_created` integer NOT NULL,
	`time_updated` integer NOT NULL,
	`time_compacting` integer,
	`time_archived` integer,
	CONSTRAINT `fk_session_project_id_project_id_fk` 
	  FOREIGN KEY (`project_id`) REFERENCES `project`(`id`) ON DELETE CASCADE
);
--> statement-breakpoint
CREATE INDEX `session_project_idx` ON `session` (`project_id`);
```

### Adding a New Table

```sql
-- packages/opencode/migration/20260225215848_workspace/migration.sql
CREATE TABLE `workspace` (
	`id` text PRIMARY KEY,
	`branch` text,
	`project_id` text NOT NULL,
	`config` text NOT NULL,
	CONSTRAINT `fk_workspace_project_id_project_id_fk` 
	  FOREIGN KEY (`project_id`) REFERENCES `project`(`id`) ON DELETE CASCADE
);
```

### Modifying Indexes

```sql
-- packages/opencode/migration/20260312043431_session_message_cursor/migration.sql
DROP INDEX IF EXISTS `message_session_idx`;
--> statement-breakpoint
DROP INDEX IF EXISTS `part_message_idx`;
--> statement-breakpoint
CREATE INDEX `message_session_time_created_id_idx` 
  ON `message` (`session_id`,`time_created`,`id`);
--> statement-breakpoint
CREATE INDEX `part_message_id_id_idx` 
  ON `part` (`message_id`,`id`);
```

### Adding Event Sourcing Tables

```sql
-- packages/opencode/migration/20260323234822_events/migration.sql
CREATE TABLE `event_sequence` (
	`aggregate_id` text PRIMARY KEY,
	`seq` integer NOT NULL
);
--> statement-breakpoint
CREATE TABLE `event` (
	`id` text PRIMARY KEY,
	`aggregate_id` text NOT NULL,
	`seq` integer NOT NULL,
	`type` text NOT NULL,
	`data` text NOT NULL,
	CONSTRAINT `fk_event_aggregate_id_event_sequence_aggregate_id_fk` 
	  FOREIGN KEY (`aggregate_id`) REFERENCES `event_sequence`(`aggregate_id`) ON DELETE CASCADE
);
```

## Migration Tracking

Drizzle tracks which migrations have been applied using an internal table:

```sql
-- Created automatically by Drizzle
CREATE TABLE IF NOT EXISTS __drizzle_migrations (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  hash TEXT NOT NULL,
  created_at INTEGER
);
```

This ensures:
- Migrations are only applied once
- The order is preserved
- You can see migration history

## JSON to SQLite Migration

opencode also has a legacy JSON storage system. The `JsonMigration` module handles migrating from JSON files to SQLite:

```typescript
// packages/opencode/src/storage/json-migration.ts (lines 26-42)
export async function run(sqlite: Database, options?: Options) {
  const storageDir = path.join(Global.Path.data, "storage")

  if (!existsSync(storageDir)) {
    log.info("storage directory does not exist, skipping migration")
    return { /* stats */ }
  }

  // Migrate projects, sessions, messages, parts, todos, permissions, shares
  // from JSON files to SQLite tables
}
```

This is a one-time data migration, separate from schema migrations.

## Best Practices

### 1. Never Edit Existing Migrations

Once a migration is committed, treat it as immutable. Create a new migration for fixes.

### 2. Test Migrations

```bash
# Create a test database
OPENCODE_DB=:memory: bun test migration.test.ts
```

### 3. Use Descriptive Slugs

```bash
# Good
bun run db generate --name add_workspace_fields

# Bad
bun run db generate --name fix
```

### 4. Keep Migrations Small

One logical change per migration makes debugging easier.

### 5. Handle Data Migration Separately

Schema changes (DDL) and data changes (DML) should be separate concerns.

## Self-Check Questions

1. Why does opencode use timestamps in migration folder names?
2. What is the purpose of `snapshot.json` files?
3. How does Drizzle know which migrations have already been applied?
4. What does `--> statement-breakpoint` do in migration files?
5. Why are migrations bundled in production builds?

## Exercises

### Exercise 1: Read a Migration

Examine `packages/opencode/migration/20260312043431_session_message_cursor/migration.sql`:
1. What indexes are being dropped?
2. What new indexes are being created?
3. Why might this change improve query performance?

### Exercise 2: Trace Migration History

List all migrations in chronological order and describe what each one does:
1. What was the initial schema?
2. What features were added over time?
3. Are there any index optimizations?

### Exercise 3: Plan a Migration

You need to add a `starred` boolean column to the `session` table. Write out:
1. The schema change in `session.sql.ts`
2. The expected SQL in the migration file
3. Any index changes that might be needed

## Further Reading

- [Drizzle Kit Documentation](https://orm.drizzle.team/kit-docs/overview)
- [SQLite ALTER TABLE](https://www.sqlite.org/lang_altertable.html)
- [Database Migration Patterns](https://martinfowler.com/articles/evodb.html)
- [Drizzle Migrations Guide](https://orm.drizzle.team/docs/migrations)
