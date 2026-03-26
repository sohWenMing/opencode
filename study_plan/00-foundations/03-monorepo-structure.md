# Monorepo Structure

## Learning Objectives

By the end of this lesson, you will be able to:

- Explain what a monorepo is and its benefits
- Understand Bun workspaces and how they're configured
- Navigate the root `package.json` and its `workspaces` field
- Use the `catalog` for shared dependency versions
- Understand Turborepo and the `turbo.json` configuration
- Trace how packages reference each other with `workspace:*`
- Navigate the `packages/` folder structure

---

## What is a Monorepo?

A **monorepo** (monolithic repository) is a single repository containing multiple projects or packages that may be related but are independently deployable.

### Monorepo vs Multi-repo

```
Multi-repo approach:
├── opencode-cli/          (separate git repo)
├── opencode-app/          (separate git repo)
├── opencode-sdk/          (separate git repo)
└── opencode-ui/           (separate git repo)

Monorepo approach (opencode):
└── opencode/              (single git repo)
    └── packages/
        ├── opencode/      (CLI)
        ├── app/           (Web UI)
        ├── sdk/           (SDK)
        └── ui/            (Shared components)
```

### Benefits of Monorepos

1. **Atomic changes**: Update multiple packages in a single commit
2. **Code sharing**: Easy to share utilities, types, and components
3. **Consistent tooling**: Same linting, testing, and build setup
4. **Simplified dependencies**: No need to publish internal packages
5. **Better discoverability**: All code in one place

### Challenges

1. **Build complexity**: Need tools to manage builds across packages
2. **CI/CD**: Must be smart about what to build/test on changes
3. **Repository size**: Can grow large over time

---

## Bun Workspaces

Bun workspaces allow you to manage multiple packages in a single repository. They're configured in the root `package.json`.

### opencode's Workspace Configuration

From `package.json`:

```json
{
  "name": "opencode",
  "private": true,
  "type": "module",
  "packageManager": "bun@1.3.11",
  "workspaces": {
    "packages": [
      "packages/*",
      "packages/console/*",
      "packages/sdk/js",
      "packages/slack"
    ],
    "catalog": {
      // ... shared versions
    }
  }
}
```

### Understanding the Configuration

**`"private": true`**: This package won't be published to npm. The root is just for orchestration.

**`"type": "module"`**: Use ES modules (import/export) by default.

**`"packageManager": "bun@1.3.11"`**: Specifies the exact Bun version for the project.

**`"workspaces.packages"`**: Glob patterns for package locations:

- `"packages/*"` - All direct children of `packages/`
- `"packages/console/*"` - All children of `packages/console/`
- `"packages/sdk/js"` - Specific path for the JS SDK
- `"packages/slack"` - Specific path for Slack integration

### How Workspaces Work

When you run `bun install` at the root:

1. Bun finds all packages matching the workspace patterns
2. Creates symlinks in `node_modules` for each workspace package
3. Installs external dependencies once (hoisted to root when possible)
4. Links workspace packages to each other

```
node_modules/
├── @opencode-ai/
│   ├── app -> ../../packages/app        (symlink)
│   ├── sdk -> ../../packages/sdk/js     (symlink)
│   └── ui -> ../../packages/ui          (symlink)
├── zod/                                  (real package)
└── typescript/                           (real package)
```

---

## The Catalog: Shared Dependency Versions

The `catalog` is a Bun feature for centralizing dependency versions across all packages.

### From `package.json`:

```json
{
  "workspaces": {
    "catalog": {
      "typescript": "5.8.2",
      "zod": "4.1.8",
      "effect": "4.0.0-beta.35",
      "hono": "4.10.7",
      "solid-js": "1.9.10",
      "vite": "7.1.4",
      "tailwindcss": "4.1.11",
      "drizzle-orm": "1.0.0-beta.19-d95b7a4",
      "@types/bun": "1.3.11",
      "@types/node": "22.13.9",
      "@playwright/test": "1.51.0"
      // ... more
    }
  }
}
```

### Using Catalog Versions

In any package's `package.json`, use `"catalog:"` to reference the centralized version:

```json
{
  "name": "@opencode-ai/app",
  "dependencies": {
    "solid-js": "catalog:",
    "zod": "catalog:"
  },
  "devDependencies": {
    "typescript": "catalog:",
    "@types/bun": "catalog:"
  }
}
```

### Benefits of the Catalog

1. **Single source of truth**: Update a version in one place
2. **Consistency**: All packages use the same version
3. **Easier upgrades**: Change once, applies everywhere
4. **No version conflicts**: Prevents accidental mismatches

### Real Example

From `packages/app/package.json`:

```json
{
  "devDependencies": {
    "@tailwindcss/vite": "catalog:",
    "@types/bun": "catalog:",
    "typescript": "catalog:",
    "vite": "catalog:",
    "vite-plugin-solid": "catalog:"
  },
  "dependencies": {
    "solid-js": "catalog:",
    "tailwindcss": "catalog:",
    "zod": "catalog:"
  }
}
```

All these resolve to the versions defined in the root catalog.

---

## Turborepo for Task Orchestration

[Turborepo](https://turbo.build/) is a build system for JavaScript/TypeScript monorepos. It handles:

- Running tasks across packages
- Caching build outputs
- Parallel execution
- Dependency-aware task ordering

### opencode's turbo.json

```json
{
  "$schema": "https://v2-8-13.turborepo.dev/schema.json",
  "globalEnv": ["CI", "OPENCODE_DISABLE_SHARE"],
  "globalPassThroughEnv": ["CI", "OPENCODE_DISABLE_SHARE"],
  "tasks": {
    "typecheck": {},
    "build": {
      "dependsOn": [],
      "outputs": ["dist/**"]
    },
    "opencode#test": {
      "dependsOn": ["^build"],
      "outputs": []
    },
    "@opencode-ai/app#test": {
      "dependsOn": ["^build"],
      "outputs": []
    }
  }
}
```

### Understanding the Configuration

**`globalEnv`**: Environment variables that affect all tasks. If these change, caches are invalidated.

**`globalPassThroughEnv`**: Environment variables passed to tasks but don't affect caching.

**`tasks`**: Defines how each task runs:

#### `typecheck` Task

```json
"typecheck": {}
```

- No special configuration
- Runs `typecheck` script in each package that has one
- Can run in parallel (no dependencies)

#### `build` Task

```json
"build": {
  "dependsOn": [],
  "outputs": ["dist/**"]
}
```

- `dependsOn: []` - No dependencies, can run immediately
- `outputs: ["dist/**"]` - Cache the `dist/` folder

#### Package-Specific `test` Tasks

```json
"opencode#test": {
  "dependsOn": ["^build"],
  "outputs": []
}
```

- `opencode#test` - Specific to the `opencode` package
- `dependsOn: ["^build"]` - Wait for dependencies' `build` tasks first
- `outputs: []` - Don't cache test results

### Running Turbo Tasks

```bash
# Run typecheck across all packages
bun turbo typecheck

# Run build across all packages
bun turbo build

# Run a specific package's task
bun turbo test --filter=opencode

# Run with verbose output
bun turbo build --verbosity=2
```

### Turbo Caching

Turbo caches task outputs based on:

1. Input files (source code)
2. Environment variables
3. Dependencies' outputs

If nothing changed, Turbo replays cached output instead of re-running:

```
$ bun turbo build
• Packages in scope: opencode, @opencode-ai/app, ...
• Running build in 5 packages
• Remote caching disabled
opencode:build: cache hit, replaying output
@opencode-ai/app:build: cache hit, replaying output
```

---

## Package References: workspace:*

Packages reference each other using the `workspace:*` protocol:

### From `packages/opencode/package.json`:

```json
{
  "dependencies": {
    "@opencode-ai/plugin": "workspace:*",
    "@opencode-ai/script": "workspace:*",
    "@opencode-ai/sdk": "workspace:*",
    "@opencode-ai/util": "workspace:*"
  }
}
```

### How It Works

- `workspace:*` means "use the version from the workspace"
- Bun creates symlinks instead of downloading from npm
- Changes in one package are immediately available in others
- No need to publish/version internal packages

### Import Example

```typescript
// In packages/opencode/src/some-file.ts
import { createOpencodeClient } from "@opencode-ai/sdk"
import { someUtil } from "@opencode-ai/util"
```

This imports from the local workspace packages, not from npm.

---

## The packages/ Folder Structure

```
packages/
├── opencode/           # Core CLI and server
├── app/                # Web UI (SolidJS)
├── sdk/
│   └── js/             # JavaScript/TypeScript SDK
├── plugin/             # Plugin API
├── script/             # Shared build scripts
├── util/               # Shared utilities
├── ui/                 # Shared UI components
├── web/                # Documentation site (Astro)
├── desktop/            # Tauri desktop app
├── desktop-electron/   # Electron desktop app
├── enterprise/         # Enterprise features
├── function/           # Serverless functions
├── slack/              # Slack integration
├── storybook/          # Component storybook
├── containers/         # Docker images
└── console/
    ├── core/           # Console backend
    ├── app/            # Console web app
    ├── mail/           # Email templates
    ├── resource/       # Resource bindings
    └── function/       # Console functions
```

### Package Roles

| Package | Description | Key Technologies |
|---------|-------------|------------------|
| `opencode` | Main CLI, server, AI providers | Effect, Drizzle, Hono |
| `app` | Web interface | SolidJS, Vite, TailwindCSS |
| `sdk/js` | Client/server SDK | TypeScript, OpenAPI |
| `plugin` | Plugin API definitions | Zod |
| `util` | Shared utilities | TypeScript |
| `ui` | Shared components | SolidJS, TailwindCSS |
| `web` | Documentation | Astro, Starlight |
| `desktop` | Desktop app | Tauri |

### Package Naming Convention

- Internal packages: `@opencode-ai/<name>`
- Published packages: `opencode` (the CLI)

### Package Dependencies Graph

```
                    ┌─────────────┐
                    │   opencode  │ (CLI/Server)
                    └──────┬──────┘
                           │
        ┌──────────────────┼──────────────────┐
        │                  │                  │
        ▼                  ▼                  ▼
   ┌─────────┐       ┌─────────┐       ┌─────────┐
   │   sdk   │       │  plugin │       │  util   │
   └─────────┘       └────┬────┘       └─────────┘
                          │
                          ▼
                    ┌─────────┐
                    │   sdk   │
                    └─────────┘

   ┌─────────┐
   │   app   │ (Web UI)
   └────┬────┘
        │
        ├──────────────────┐
        │                  │
        ▼                  ▼
   ┌─────────┐       ┌─────────┐
   │   sdk   │       │   ui    │
   └─────────┘       └─────────┘
```

---

## Self-Check Questions

1. What's the difference between `packages/*` and `packages/console/*` in the workspaces config?
2. How does `catalog:` help maintain consistency across packages?
3. What does `dependsOn: ["^build"]` mean in turbo.json?
4. Why use `workspace:*` instead of a version number?
5. Which package contains the main CLI code?

---

## Exercises

### Exercise 1: Explore the Workspace

Run these commands from the repository root:

```bash
# List all workspace packages
bun pm ls

# See the dependency tree
bun pm ls --all

# Check which packages depend on zod
grep -r '"zod"' packages/*/package.json
```

### Exercise 2: Trace a Dependency

Starting from `packages/app/package.json`:

1. Find all `workspace:*` dependencies
2. Locate each referenced package in the `packages/` folder
3. Check what external dependencies each has

### Exercise 3: Understand the Build Order

Given the turbo.json configuration:

1. What tasks can run in parallel?
2. What must complete before `opencode#test` runs?
3. What gets cached after a `build` task?

### Exercise 4: Add a Hypothetical Package

If you were to add a new package `packages/analytics/`:

1. What would you add to the root `package.json` workspaces?
2. What would the package's `package.json` look like?
3. How would another package import from it?

Create a mental model (don't actually create the files):

```json
// packages/analytics/package.json
{
  "name": "@opencode-ai/analytics",
  "version": "1.0.0",
  "type": "module",
  "exports": {
    ".": "./src/index.ts"
  },
  "dependencies": {
    // What would go here?
  }
}
```

### Exercise 5: Catalog Version Update

Imagine you need to update `zod` from `4.1.8` to `4.2.0`:

1. Where would you make the change?
2. How many files would you need to modify?
3. What command would verify all packages still build?

---

## Further Reading

- [Bun Workspaces](https://bun.sh/docs/install/workspaces)
- [Turborepo Documentation](https://turbo.build/repo/docs)
- [Monorepo Tools Comparison](https://monorepo.tools/)
- [npm Workspaces](https://docs.npmjs.com/cli/using-npm/workspaces) (similar concept)
