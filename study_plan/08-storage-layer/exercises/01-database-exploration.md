# Exercise 01: Database Exploration

## Overview

In this exercise, you'll explore the opencode SQLite database directly. You'll locate the database file, examine its schema, query data, and understand the migration history.

## Prerequisites

- SQLite CLI installed (`sqlite3` command)
- opencode installed and run at least once (to create the database)
- Basic SQL knowledge

## Part 1: Locate the Database

### Task 1.1: Find the Database Path

The database location depends on your operating system and configuration.

**Default Locations:**

| OS | Path |
|----|------|
| Linux | `~/.local/share/opencode/opencode.db` |
| macOS | `~/Library/Application Support/opencode/opencode.db` |
| Windows | `%APPDATA%\opencode\opencode.db` |

**Questions:**
1. What is the full path to your opencode database?
2. What is the file size of the database?
3. Are there any related files (like `-wal` or `-shm`)?

**Hint:** Look at `packages/opencode/src/storage/db.ts` lines 30-44 to understand how the path is determined.

### Task 1.2: Understand Channel-Specific Databases

opencode can create separate databases for different release channels.

**Questions:**
1. What environment variable can override the database path?
2. How does the channel affect the database filename?
3. What is the purpose of `OPENCODE_DISABLE_CHANNEL_DB`?

**Reference:** `packages/opencode/src/storage/db.ts` lines 30-44

## Part 2: Explore the Schema

### Task 2.1: Connect to the Database

```bash
# Open the database
sqlite3 ~/.local/share/opencode/opencode.db

# Enable headers and column mode for better output
.headers on
.mode column
```

### Task 2.2: List All Tables

```sql
-- List all tables
.tables

-- Or use SQL
SELECT name FROM sqlite_master WHERE type='table' ORDER BY name;
```

**Questions:**
1. How many tables exist in the database?
2. Which table is used by Drizzle to track migrations?
3. Are there any tables not defined in the `.sql.ts` files?

### Task 2.3: Examine Table Schemas

```sql
-- Show schema for a specific table
.schema session

-- Or use PRAGMA
PRAGMA table_info(session);
```

**Exercise:** Document the schema for these tables:
- `project`
- `session`
- `message`
- `part`

For each table, note:
- Column names and types
- Primary key
- Foreign keys
- Indexes

### Task 2.4: View Indexes

```sql
-- List all indexes
SELECT name, tbl_name, sql FROM sqlite_master WHERE type='index';

-- Or for a specific table
PRAGMA index_list(session);
PRAGMA index_info(session_project_idx);
```

**Questions:**
1. What indexes exist on the `message` table?
2. Why is there a composite index on `(session_id, time_created, id)`?
3. Which tables have the most indexes?

## Part 3: Query Data

### Task 3.1: Count Records

```sql
-- Count projects
SELECT COUNT(*) as project_count FROM project;

-- Count sessions per project
SELECT project_id, COUNT(*) as session_count 
FROM session 
GROUP BY project_id;

-- Count messages per session
SELECT session_id, COUNT(*) as message_count 
FROM message 
GROUP BY session_id 
ORDER BY message_count DESC 
LIMIT 10;
```

**Questions:**
1. How many projects do you have?
2. Which project has the most sessions?
3. What is the average number of messages per session?

### Task 3.2: Explore Relationships

```sql
-- Find sessions with their project names
SELECT s.id, s.title, p.worktree 
FROM session s 
JOIN project p ON s.project_id = p.id 
LIMIT 10;

-- Find messages with part counts
SELECT m.id, m.session_id, COUNT(p.id) as part_count
FROM message m
LEFT JOIN part p ON m.id = p.message_id
GROUP BY m.id
ORDER BY part_count DESC
LIMIT 10;
```

**Exercise:** Write a query that shows:
- Session title
- Project worktree
- Number of messages
- Total number of parts

### Task 3.3: Explore JSON Data

opencode stores complex data as JSON in TEXT columns.

```sql
-- View message data
SELECT id, json_extract(data, '$.role') as role
FROM message
LIMIT 10;

-- View session summary
SELECT id, title, summary_additions, summary_deletions
FROM session
WHERE summary_additions IS NOT NULL
LIMIT 10;
```

**Questions:**
1. What roles appear in message data?
2. What fields are stored in the `data` column of `part`?
3. How is permission data structured?

### Task 3.4: Time-Based Queries

```sql
-- Find recent sessions
SELECT id, title, datetime(time_created/1000, 'unixepoch') as created
FROM session
ORDER BY time_created DESC
LIMIT 10;

-- Find sessions modified today
SELECT id, title
FROM session
WHERE time_updated > (strftime('%s', 'now') - 86400) * 1000;
```

**Questions:**
1. Why are timestamps stored as integers?
2. What is the timestamp format (seconds or milliseconds)?
3. How would you find sessions created in the last week?

## Part 4: Migration History

### Task 4.1: View Applied Migrations

```sql
-- Check migration tracking table
SELECT * FROM __drizzle_migrations ORDER BY created_at;
```

**Questions:**
1. How many migrations have been applied?
2. What is the hash used for?
3. When was the most recent migration applied?

### Task 4.2: Compare Schema to Code

**Exercise:** Compare the actual database schema to the code:

1. Run `.schema session` in SQLite
2. Read `packages/opencode/src/session/session.sql.ts`
3. Note any differences

**Questions:**
1. Do all columns in the code exist in the database?
2. Are the types consistent?
3. Are all indexes present?

### Task 4.3: Trace a Migration

Pick a migration folder (e.g., `20260312043431_session_message_cursor`) and:

1. Read the `migration.sql` file
2. Verify the changes exist in the database
3. Check the `snapshot.json` for the expected schema

## Part 5: Advanced Exploration

### Task 5.1: Database Statistics

```sql
-- Table sizes (approximate)
SELECT name, 
       (SELECT COUNT(*) FROM sqlite_master WHERE type='index' AND tbl_name=m.name) as index_count
FROM sqlite_master m 
WHERE type='table' 
AND name NOT LIKE 'sqlite_%'
AND name NOT LIKE '__%';

-- Database page count and size
PRAGMA page_count;
PRAGMA page_size;
```

### Task 5.2: Foreign Key Verification

```sql
-- Check foreign key integrity
PRAGMA foreign_key_check;

-- If any issues, investigate
PRAGMA foreign_key_check(session);
```

### Task 5.3: WAL Mode Verification

```sql
-- Check journal mode
PRAGMA journal_mode;

-- Check WAL checkpoint status
PRAGMA wal_checkpoint;
```

**Questions:**
1. Is WAL mode enabled?
2. What are the benefits of WAL mode for opencode?
3. What do the `-wal` and `-shm` files contain?

## Challenges

### Challenge 1: Session Analytics

Write SQL queries to answer:
1. What is the total number of code additions across all sessions?
2. Which session has the most tool calls (parts with tool data)?
3. What is the distribution of session lengths (by message count)?

### Challenge 2: Data Integrity Check

Write queries to find:
1. Sessions without any messages
2. Messages without any parts
3. Parts referencing non-existent messages

### Challenge 3: Schema Documentation

Create a complete schema documentation including:
1. All tables with their columns
2. All foreign key relationships
3. All indexes and their purposes
4. Sample data for each table

## Reflection Questions

1. Why does opencode use SQLite instead of a client-server database?
2. What are the trade-offs of storing JSON in TEXT columns?
3. How does the schema design support the session/message/part hierarchy?
4. What would you change about the schema design?

## Solutions

<details>
<summary>Click to reveal solutions</summary>

### Task 3.2 Solution

```sql
SELECT 
  s.title,
  p.worktree,
  COUNT(DISTINCT m.id) as message_count,
  COUNT(pt.id) as part_count
FROM session s
JOIN project p ON s.project_id = p.id
LEFT JOIN message m ON s.id = m.session_id
LEFT JOIN part pt ON m.id = pt.message_id
GROUP BY s.id
ORDER BY message_count DESC
LIMIT 10;
```

### Challenge 1 Solution

```sql
-- Total additions
SELECT SUM(summary_additions) as total_additions FROM session;

-- Most tool calls (assuming tool parts have specific structure)
SELECT m.session_id, COUNT(*) as tool_count
FROM part p
JOIN message m ON p.message_id = m.id
WHERE json_extract(p.data, '$.type') = 'tool-invocation'
GROUP BY m.session_id
ORDER BY tool_count DESC
LIMIT 5;

-- Session length distribution
SELECT 
  CASE 
    WHEN msg_count <= 5 THEN '1-5'
    WHEN msg_count <= 20 THEN '6-20'
    WHEN msg_count <= 50 THEN '21-50'
    ELSE '50+'
  END as bucket,
  COUNT(*) as session_count
FROM (
  SELECT session_id, COUNT(*) as msg_count
  FROM message
  GROUP BY session_id
) 
GROUP BY bucket;
```

### Challenge 2 Solution

```sql
-- Sessions without messages
SELECT s.id, s.title
FROM session s
LEFT JOIN message m ON s.id = m.session_id
WHERE m.id IS NULL;

-- Messages without parts
SELECT m.id, m.session_id
FROM message m
LEFT JOIN part p ON m.id = p.message_id
WHERE p.id IS NULL;

-- Orphaned parts (shouldn't exist with FK constraints)
SELECT p.id, p.message_id
FROM part p
LEFT JOIN message m ON p.message_id = m.id
WHERE m.id IS NULL;
```

</details>

## Next Steps

After completing this exercise, you should:
1. Understand the database structure
2. Be comfortable querying opencode data
3. Know how to verify schema consistency
4. Be ready to trace data flow through the application

Continue to Module 09 to learn about the sync system and event sourcing.
