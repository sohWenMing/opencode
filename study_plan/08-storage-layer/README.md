# Module 08: Storage Layer

This module explains how opencode persists data using SQLite and Drizzle ORM.

## Overview

opencode uses a modern, type-safe approach to data persistence:

- **SQLite** - Lightweight, embedded database requiring no separate server
- **Drizzle ORM** - TypeScript-first ORM with excellent type inference
- **Schema-as-Code** - Database structure defined in TypeScript files

## Module Structure

```
08-storage-layer/
├── 01-drizzle-orm.md           # Drizzle ORM fundamentals
├── 02-schema-design.md         # Database schema and tables
├── 03-migrations.md            # Database migrations
└── exercises/
    └── 01-database-exploration.md
```

## Learning Path

1. **Drizzle ORM** - Understand the ORM layer and why it was chosen
2. **Schema Design** - Learn the database structure and relationships
3. **Migrations** - Understand how schema changes are managed

## Key Files

| File | Purpose |
|------|---------|
| `packages/opencode/src/storage/db.ts` | Database client and connection |
| `packages/opencode/src/storage/schema.ts` | Schema exports |
| `packages/opencode/src/**/*.sql.ts` | Individual table definitions |
| `packages/opencode/migration/` | SQL migration files |
| `packages/opencode/drizzle.config.ts` | Drizzle Kit configuration |

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                    Application Layer                         │
│  (Session, Project, Account, etc.)                          │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                     Database.use()                           │
│  Type-safe queries via Drizzle ORM                          │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    Drizzle ORM Layer                         │
│  - Query building                                            │
│  - Type inference                                            │
│  - Transaction management                                    │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    SQLite Database                           │
│  ~/.local/share/opencode/opencode.db                        │
└─────────────────────────────────────────────────────────────┘
```

## Prerequisites

- Understanding of TypeScript types and generics
- Basic SQL knowledge (SELECT, INSERT, UPDATE, DELETE)
- Familiarity with the session system (Module 06)
