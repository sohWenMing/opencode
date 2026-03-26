# Contributing to OpenCode

This guide covers everything you need to know to contribute to the opencode project, from setting up your development environment to getting your PR merged.

## Types of Contributions Welcome

The opencode team welcomes these types of contributions:

- **Bug fixes** - Found something broken? Fix it!
- **Additional LSPs / Formatters** - Expand language support
- **LLM performance improvements** - Make the AI better
- **New provider support** - Add more LLM providers
- **Environment-specific fixes** - Platform quirks, edge cases
- **Missing standard behavior** - Features users expect
- **Documentation improvements** - Help others learn

> **Important:** UI changes and core product features require design review with the core team before implementation. Open an issue first to discuss.

## Finding Issues to Work On

Look for issues with these labels:

- [`help wanted`](https://github.com/anomalyco/opencode/issues?q=is%3Aissue%20state%3Aopen%20label%3Ahelp-wanted) - Maintainers want community help
- [`good first issue`](https://github.com/anomalyco/opencode/issues?q=is%3Aissue%20state%3Aopen%20label%3A%22good%20first%20issue%22) - Great for newcomers
- [`bug`](https://github.com/anomalyco/opencode/issues?q=is%3Aissue%20state%3Aopen%20label%3Abug) - Something is broken
- [`perf`](https://github.com/anomalyco/opencode/issues?q=is%3Aopen%20is%3Aissue%20label%3A%22perf%22) - Performance improvements needed

**Before starting work:** Comment on the issue to express interest. A maintainer may assign it to you.

## Development Environment Setup

### Prerequisites

- **Bun 1.3+** - The JavaScript runtime
  ```bash
  curl -fsSL https://bun.sh/install | bash
  ```
- **Git** - Version control
- **A code editor** - VS Code recommended (has example configs)

### Initial Setup

1. **Fork the repository** on GitHub

2. **Clone your fork:**
   ```bash
   git clone https://github.com/YOUR_USERNAME/opencode.git
   cd opencode
   ```

3. **Install dependencies:**
   ```bash
   bun install
   ```

4. **Start the development server:**
   ```bash
   bun dev
   ```

### Running Against Different Directories

By default, `bun dev` runs opencode in `packages/opencode`. To run against a different directory:

```bash
# Run against a specific directory
bun dev /path/to/your/project

# Run against the opencode repo itself
bun dev .
```

### Understanding bun dev vs opencode

During development, `bun dev` is equivalent to the built `opencode` command:

```bash
# Development commands
bun dev --help           # Show all available commands
bun dev serve            # Start headless API server
bun dev web              # Start server + open web interface
bun dev <directory>      # Start TUI in specific directory

# Production equivalents
opencode --help
opencode serve
opencode web
opencode <directory>
```

### Building a Local Binary

To compile a standalone executable:

```bash
./packages/opencode/script/build.ts --single
```

Run it with:

```bash
./packages/opencode/dist/opencode-<platform>/bin/opencode
```

Replace `<platform>` with your platform (e.g., `darwin-arm64`, `linux-x64`, `win32-x64`).

### Running the API Server

```bash
# Default port (4096)
bun dev serve

# Custom port
bun dev serve --port 8080
```

### Running the Web App

1. Start the opencode server (see above)
2. Run the web app:
   ```bash
   bun run --cwd packages/app dev
   ```
3. Open http://localhost:5173

### Running the Desktop App

The desktop app uses Tauri (requires Rust toolchain):

```bash
# Full native app
bun run --cwd packages/desktop tauri dev

# Web dev server only (no native shell)
bun run --cwd packages/desktop dev
```

See [Tauri prerequisites](https://v2.tauri.app/start/prerequisites/) for setup.

### Regenerating the SDK

After changing the API or SDK:

```bash
./script/generate.ts
```

## Coding Style Guide

The opencode codebase follows specific conventions. **Following these is important for PR acceptance.**

### General Principles

- Keep logic in one function unless composable or reusable
- Avoid `try`/`catch` where possible (use `.catch()` instead)
- Avoid the `any` type - use precise types
- Use Bun APIs when possible (e.g., `Bun.file()`)
- Rely on type inference - avoid explicit annotations unless necessary
- Prefer functional array methods (`flatMap`, `filter`, `map`) over `for` loops

### Naming Conventions

**Use single-word names by default.** Multi-word names only when a single word would be unclear.

```typescript
// Good
const foo = 1
function journal(dir: string) {}

// Bad
const fooBar = 1
function prepareJournal(dir: string) {}
```

**Good short names:** `pid`, `cfg`, `err`, `opts`, `dir`, `root`, `child`, `state`, `timeout`

**Avoid unless necessary:** `inputPID`, `existingClient`, `connectTimeout`, `workerPath`

### Inlining

Reduce variable count by inlining values used only once:

```typescript
// Good
const journal = await Bun.file(path.join(dir, "journal.json")).json()

// Bad
const journalPath = path.join(dir, "journal.json")
const journal = await Bun.file(journalPath).json()
```

### Destructuring

Avoid unnecessary destructuring. Use dot notation:

```typescript
// Good
obj.a
obj.b

// Bad
const { a, b } = obj
```

### Variables

Prefer `const` over `let`. Use ternaries or early returns:

```typescript
// Good
const foo = condition ? 1 : 2

// Bad
let foo
if (condition) foo = 1
else foo = 2
```

### Control Flow

Avoid `else` statements. Use early returns:

```typescript
// Good
function foo() {
  if (condition) return 1
  return 2
}

// Bad
function foo() {
  if (condition) return 1
  else return 2
}
```

### Schema Definitions (Drizzle)

Use snake_case for field names:

```typescript
// Good
const table = sqliteTable("session", {
  id: text().primaryKey(),
  project_id: text().notNull(),
  created_at: integer().notNull(),
})

// Bad
const table = sqliteTable("session", {
  id: text("id").primaryKey(),
  projectID: text("project_id").notNull(),
  createdAt: integer("created_at").notNull(),
})
```

## Running Tests

Tests must be run from package directories, not the repo root:

```bash
# Correct
cd packages/opencode
bun test

# Wrong - will fail
bun test  # from repo root
```

### Testing Philosophy

- **Avoid mocks** as much as possible
- Test actual implementation, don't duplicate logic in tests
- Write tests that verify behavior, not implementation details

## Type Checking

Always run type checking from package directories:

```bash
# Correct
cd packages/opencode
bun typecheck

# Wrong
tsc  # Never run tsc directly
```

## Creating Pull Requests

### Issue First Policy

**All PRs must reference an existing issue.** Before opening a PR:

1. Open an issue describing the bug or feature
2. Wait for maintainer feedback on complex features
3. Link the issue in your PR with `Fixes #123` or `Closes #123`

### PR Requirements

- **Keep PRs small and focused** - One logical change per PR
- **Explain the issue** and why your change fixes it
- **Check for existing functionality** before adding new code

### PR Titles

Follow conventional commit format:

| Prefix | Use Case |
|--------|----------|
| `feat:` | New feature or functionality |
| `fix:` | Bug fix |
| `docs:` | Documentation changes |
| `chore:` | Maintenance, dependencies |
| `refactor:` | Code refactoring (no behavior change) |
| `test:` | Adding or updating tests |

Optional scope for package:

```
feat(app): add dark mode support
fix(desktop): resolve crash on startup
chore(opencode): bump dependency versions
```

### For UI Changes

Include screenshots or videos showing before and after. This speeds up review significantly.

### For Logic Changes

Explain how you verified it works:
- What did you test?
- How can a reviewer reproduce/confirm the fix?

### What to Avoid

**No AI-generated walls of text.** Long, AI-generated PR descriptions will be ignored.

- Write short, focused descriptions
- Explain in your own words
- If you can't explain it briefly, your PR might be too large

## Code Review Process

1. **Submit your PR** following the guidelines above
2. **Automated checks** run (linting, tests, type checking)
3. **Maintainer review** - expect feedback within a few days
4. **Address feedback** - make requested changes
5. **Approval and merge** - maintainer merges when ready

### Tips for Faster Reviews

- Keep PRs small (under 400 lines ideal)
- Write clear commit messages
- Respond promptly to feedback
- Be open to suggestions

## Adding New Providers

New providers shouldn't require code changes in most cases. First, make a PR to:

https://github.com/anomalyco/models.dev

This repository defines provider configurations that opencode uses.

## Setting Up a Debugger

Bun debugging can be tricky. The most reliable method:

```bash
bun run --inspect=ws://localhost:6499/ dev ...
```

Then attach your debugger to that URL.

### Tips

- Use `--inspect-wait` or `--inspect-brk` depending on your workflow
- Set `export BUN_OPTIONS=--inspect=ws://localhost:6499/` to avoid typing it each time
- For TUI + server debugging, you may need `bun dev spawn` instead of `bun dev`

### Debugging Server Separately

```bash
# Debug server
bun run --inspect=ws://localhost:6499/ --cwd packages/opencode ./src/index.ts serve --port 4096

# Attach TUI
opencode attach http://localhost:4096
```

### VS Code Setup

Copy the example configurations:
- `.vscode/settings.example.json` → `.vscode/settings.json`
- `.vscode/launch.example.json` → `.vscode/launch.json`

**Note:** Debug configurations with `"request": "launch"` may have breakpoint mapping issues.

## Community Guidelines

### Trust & Vouch System

The project uses [vouch](https://github.com/mitchellh/vouch) to manage contributor trust.

- **Vouched users** - Explicitly trusted contributors
- **Denounced users** - Blocked from contributing
- **Everyone else** - Can participate normally

You don't need to be vouched to open issues or PRs.

### Denouncement Policy

Denouncement is reserved for:
- Repeatedly submitting low-quality AI-generated contributions
- Spam
- Acting in bad faith

It is **not** used for disagreements or honest mistakes.

### Issue Requirements

All issues must use a template:
- **Bug report** - For reporting bugs
- **Feature request** - For suggesting enhancements
- **Question** - For asking questions

Issues not following templates have 2 hours to be fixed before automatic closure.

## Getting Help

- **Discord** - [opencode.ai/discord](https://opencode.ai/discord)
- **GitHub Discussions** - For longer-form questions
- **Issue comments** - For specific issue questions

## Quick Reference

| Task | Command |
|------|---------|
| Install dependencies | `bun install` |
| Start dev server | `bun dev` |
| Run against directory | `bun dev /path/to/dir` |
| Start API server | `bun dev serve` |
| Run web app | `bun run --cwd packages/app dev` |
| Run tests | `cd packages/opencode && bun test` |
| Type check | `cd packages/opencode && bun typecheck` |
| Build binary | `./packages/opencode/script/build.ts --single` |
| Regenerate SDK | `./script/generate.ts` |

## Checklist Before Submitting

- [ ] Issue exists and is linked in PR
- [ ] Code follows style guide
- [ ] Tests pass locally
- [ ] Type checking passes
- [ ] PR title follows conventional commit format
- [ ] Description explains what and why
- [ ] Screenshots included (for UI changes)
- [ ] No AI-generated walls of text

Good luck with your contribution!
