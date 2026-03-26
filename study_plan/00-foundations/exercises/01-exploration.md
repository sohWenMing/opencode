# Module 00 Exercises: Foundations Exploration

These exercises will help you solidify your understanding of TypeScript, Bun, and the opencode monorepo structure through hands-on exploration.

---

## Prerequisites

Before starting, ensure you have:

1. Cloned the opencode repository
2. Installed Bun (`curl -fsSL https://bun.sh/install | bash`)
3. Run `bun install` at the repository root
4. A code editor with TypeScript support (VS Code recommended)

---

## Part 1: TypeScript Exploration

### Exercise 1.1: Type Scavenger Hunt

Find examples of each TypeScript concept in the codebase. Record the file path and line numbers.

| Concept | File Path | Line Numbers |
|---------|-----------|--------------|
| Interface with 3+ properties | | |
| Type alias with union (`\|`) | | |
| Generic function `<T>` | | |
| Generic with constraint `<T extends ...>` | | |
| async/await function | | |
| Promise.all usage | | |
| import type statement | | |
| Namespace (export namespace) | | |

**Hints:**
- Start with `packages/opencode/src/util/`
- Use grep: `grep -r "interface " packages/opencode/src/`
- Use grep: `grep -r "<T extends" packages/opencode/src/`

### Exercise 1.2: Read and Understand

Open `packages/opencode/src/util/context.ts` and answer:

1. What does the `Context.create<T>()` function return?
2. What error is thrown when `use()` is called without a context?
3. How does `provide()` make a value available to `use()`?
4. What Node.js API does this use under the hood?

**Write your answers here:**

```
1. 

2. 

3. 

4. 
```

### Exercise 1.3: Type the Function

Given this untyped JavaScript function, add TypeScript types:

```typescript
// Original (untyped)
function processItems(items, transform) {
  return items.map(transform)
}

// Your typed version:
function processItems(/* add types */): /* return type */ {
  return items.map(transform)
}
```

**Bonus:** Make it generic so it works with any input/output types.

---

## Part 2: Bun Exploration

### Exercise 2.1: Bun Commands

Run each command and record what happens:

```bash
# 1. Check Bun version
bun --version
# Result: 

# 2. Run the root test script (should fail)
bun test
# Result: 

# 3. Run typecheck across all packages
bun turbo typecheck
# Result: 

# 4. Start the dev server
cd packages/opencode
bun run dev
# Result: (press Ctrl+C to stop)
```

### Exercise 2.2: Bun File API

Create a file `scripts/bun-explore.ts` with this content:

```typescript
// Read package.json
const pkg = await Bun.file("package.json").json()
console.log("Project name:", pkg.name)
console.log("Package manager:", pkg.packageManager)

// List workspace packages
console.log("\nWorkspace packages:")
for (const pattern of pkg.workspaces.packages) {
  console.log(" -", pattern)
}

// Read bunfig.toml
const bunfig = await Bun.file("bunfig.toml").text()
console.log("\nbunfig.toml contents:")
console.log(bunfig)
```

Run it:
```bash
bun scripts/bun-explore.ts
```

**Questions:**
1. What version of Bun does the project use?
2. How many workspace patterns are defined?
3. What does `exact = true` in bunfig.toml do?

### Exercise 2.3: Bun Spawn

Extend your script to run a shell command:

```typescript
// Add to scripts/bun-explore.ts

// List packages directory
const proc = Bun.spawn(["ls", "-la", "packages"])
const output = await new Response(proc.stdout).text()
console.log("\nPackages directory:")
console.log(output)

// Get git status
const git = Bun.spawn(["git", "status", "--short"])
const status = await new Response(git.stdout).text()
console.log("\nGit status:")
console.log(status || "(clean)")
```

### Exercise 2.4: Find Bun API Usage

Search the codebase for Bun API usage:

```bash
# Find Bun.file usage
grep -r "Bun.file" packages/ --include="*.ts" | head -10

# Find Bun.spawn usage
grep -r "Bun.spawn" packages/ --include="*.ts" | head -10

# Find Bun.serve usage
grep -r "Bun.serve" packages/ --include="*.ts" | head -10
```

Record 3 interesting examples you found:

```
1. File: 
   Usage: 

2. File: 
   Usage: 

3. File: 
   Usage: 
```

---

## Part 3: Monorepo Exploration

### Exercise 3.1: Map the Packages

Fill in this table by examining each package's `package.json`:

| Package | Name | Main Export | Key Dependencies |
|---------|------|-------------|------------------|
| packages/opencode | opencode | ./src/index.ts | effect, hono, drizzle-orm |
| packages/app | | | |
| packages/sdk/js | | | |
| packages/ui | | | |
| packages/plugin | | | |

### Exercise 3.2: Trace workspace:* Dependencies

Starting from `packages/opencode/package.json`:

1. List all `workspace:*` dependencies
2. For each, find the actual package location
3. Check what that package exports

```
packages/opencode depends on:
├── @opencode-ai/plugin -> packages/plugin
│   └── exports: ???
├── @opencode-ai/sdk -> packages/sdk/js
│   └── exports: ???
├── @opencode-ai/util -> packages/util
│   └── exports: ???
└── @opencode-ai/script -> packages/script
    └── exports: ???
```

### Exercise 3.3: Catalog Analysis

From the root `package.json` catalog:

1. Find 5 packages that use `catalog:` versions
2. Verify they all resolve to the same version

```bash
# Example: Check zod versions
grep -r '"zod"' packages/*/package.json
```

**Your findings:**

| Package | Dependency | Catalog Version |
|---------|------------|-----------------|
| | zod | |
| | zod | |
| | typescript | |
| | typescript | |
| | solid-js | |

### Exercise 3.4: Turbo Task Analysis

Read `turbo.json` and answer:

1. Which tasks have no dependencies and can run in parallel?
2. Which tasks depend on other tasks completing first?
3. What outputs are cached for the `build` task?
4. Why do `opencode#test` and `@opencode-ai/app#test` have special configurations?

**Your answers:**

```
1. Parallel tasks: 

2. Tasks with dependencies: 

3. Cached outputs: 

4. Special test configs because: 
```

### Exercise 3.5: Create a Dependency Graph

Draw (or describe) the dependency relationships between these packages:

- opencode
- app
- sdk
- ui
- util
- plugin

Use arrows to show "depends on" relationships.

```
Example format:
opencode -> sdk (opencode depends on sdk)
app -> ui (app depends on ui)

Your graph:




```

---

## Part 4: Integration Challenge

### Exercise 4.1: End-to-End Trace

Trace how a user's message flows through the system:

1. User opens the web app (`packages/app`)
2. App connects to the server (`packages/opencode`)
3. Server processes the request using various utilities

**Questions:**

1. What SDK function does the app use to connect to the server?
   - Hint: Look in `packages/sdk/js/src/`

2. What API endpoint does the app call?
   - Hint: Look for Hono routes in `packages/opencode/src/`

3. What database is used to store sessions?
   - Hint: Look for Drizzle schemas in `packages/opencode/src/`

### Exercise 4.2: Build the Project

Run the full build process and observe:

```bash
# Clean any previous builds
rm -rf packages/*/dist

# Run the build
bun turbo build

# Check what was created
ls -la packages/opencode/dist/
ls -la packages/app/dist/
```

**Questions:**

1. How long did the build take?
2. What files were created in each dist folder?
3. Run the build again - was it faster? Why?

---

## Part 5: Reflection

### What I Learned

Write 3-5 sentences about what you learned from these exercises:

```




```

### Questions I Still Have

List any questions that came up during exploration:

```
1. 
2. 
3. 
```

### Next Steps

Based on what you learned, what would you like to explore next?

```


```

---

## Solutions Checklist

After completing the exercises, verify your understanding:

- [ ] I can find and read TypeScript type definitions in the codebase
- [ ] I understand the difference between interfaces and type aliases
- [ ] I can run Bun commands and scripts
- [ ] I understand what `Bun.file()`, `Bun.spawn()`, and `Bun.serve()` do
- [ ] I can navigate the monorepo structure
- [ ] I understand how `workspace:*` and `catalog:` work
- [ ] I can trace dependencies between packages
- [ ] I understand what Turborepo does and how tasks are configured

---

## Bonus Challenges

If you finish early, try these:

### Bonus 1: Write a Utility Function

Add a new utility function to `packages/util/src/`:

```typescript
// packages/util/src/string.ts
export function truncate(str: string, maxLength: number): string {
  // Your implementation
}
```

### Bonus 2: Explore Effect

The opencode project uses Effect heavily. Find 3 examples of Effect usage:

```bash
grep -r "Effect.gen" packages/opencode/src/ | head -5
grep -r "Effect.fn" packages/opencode/src/ | head -5
```

### Bonus 3: Run the Tests

```bash
cd packages/opencode
bun test

# Run a specific test file
bun test test/util/timeout.test.ts
```

What tests exist? What do they test?
