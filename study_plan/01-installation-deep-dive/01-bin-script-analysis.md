# Lesson 01: The Bin Script Analysis

## Learning Objectives

By the end of this lesson, you will be able to:

- Explain what happens when a user runs `npx opencode` or installs globally with `npm install -g opencode`
- Understand how npm/bun bin scripts work and how they're registered
- Trace through the `bin/opencode` script line-by-line
- Explain the role of the `OPENCODE_BIN_PATH` environment variable
- Describe how the script detects the current platform and architecture

---

## How npm/Bun Bin Scripts Work

When you install a package that has a `bin` field in its `package.json`, npm/bun creates executable symlinks in your `node_modules/.bin/` directory (for local installs) or in your global bin directory (for global installs).

### The package.json bin Entry

```json
// packages/opencode/package.json (lines 22-24)
"bin": {
  "opencode": "./bin/opencode"
}
```

This tells npm: "When someone runs the command `opencode`, execute the file at `./bin/opencode`."

### Installation Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           User runs: npx opencode                           │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  npm/npx downloads the opencode package from registry                       │
│  - Reads package.json                                                       │
│  - Finds "bin": { "opencode": "./bin/opencode" }                           │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  npm creates symlink/shim:                                                  │
│  - Local: node_modules/.bin/opencode → ../opencode/bin/opencode            │
│  - Global: /usr/local/bin/opencode → <global_modules>/opencode/bin/opencode│
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  Node.js executes bin/opencode via the shebang line                        │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## The Shebang Line

```javascript
// packages/opencode/bin/opencode (line 1)
#!/usr/bin/env node
```

### What This Does

1. **`#!`** - The "shebang" tells Unix-like systems this file is a script
2. **`/usr/bin/env`** - A utility that searches PATH for the specified program
3. **`node`** - The program to execute this script with

Using `/usr/bin/env node` instead of a hardcoded path like `/usr/local/bin/node` ensures the script works regardless of where Node.js is installed on the system.

### Cross-Platform Behavior

| Platform | Behavior |
|----------|----------|
| Linux/macOS | Kernel reads shebang, executes with Node.js |
| Windows | npm creates a `.cmd` wrapper that invokes Node.js |

---

## Line-by-Line Walkthrough

### Module Imports (Lines 3-6)

```javascript
// packages/opencode/bin/opencode (lines 3-6)
const childProcess = require("child_process")
const fs = require("fs")
const path = require("path")
const os = require("os")
```

The script uses CommonJS (`require`) instead of ES modules (`import`) for maximum compatibility. These are Node.js built-in modules:

| Module | Purpose |
|--------|---------|
| `child_process` | Spawn the native binary as a subprocess |
| `fs` | Check if files/directories exist |
| `path` | Build cross-platform file paths |
| `os` | Detect platform (darwin/linux/win32) and architecture (x64/arm64) |

### The `run()` Function (Lines 8-18)

```javascript
// packages/opencode/bin/opencode (lines 8-18)
function run(target) {
  const result = childProcess.spawnSync(target, process.argv.slice(2), {
    stdio: "inherit",
  })
  if (result.error) {
    console.error(result.error.message)
    process.exit(1)
  }
  const code = typeof result.status === "number" ? result.status : 0
  process.exit(code)
}
```

This is the core execution function. Let's break it down:

| Line | Explanation |
|------|-------------|
| `childProcess.spawnSync(target, ...)` | Synchronously spawn the native binary |
| `process.argv.slice(2)` | Pass all CLI arguments (skip `node` and script path) |
| `stdio: "inherit"` | Share stdin/stdout/stderr with parent process |
| `process.exit(code)` | Exit with the same code as the native binary |

**Why `spawnSync` instead of `spawn`?**

The script needs to wait for the binary to complete and propagate its exit code. Using synchronous spawn ensures the Node.js wrapper exits with the same status as the native binary.

### Environment Variable Override (Lines 20-23)

```javascript
// packages/opencode/bin/opencode (lines 20-23)
const envPath = process.env.OPENCODE_BIN_PATH
if (envPath) {
  run(envPath)
}
```

**Purpose**: Allows developers to override which binary gets executed.

**Use Cases**:
- Testing a development build
- Using a custom-compiled binary
- Debugging installation issues

**Example**:
```bash
OPENCODE_BIN_PATH=/path/to/custom/opencode opencode --version
```

### Cached Binary Check (Lines 25-32)

```javascript
// packages/opencode/bin/opencode (lines 25-32)
const scriptPath = fs.realpathSync(__filename)
const scriptDir = path.dirname(scriptPath)

const cached = path.join(scriptDir, ".opencode")
if (fs.existsSync(cached)) {
  run(cached)
}
```

**What's happening**:

1. `fs.realpathSync(__filename)` - Resolve symlinks to get the actual script location
2. Look for a `.opencode` binary in the same directory as the script
3. If found, run it immediately (skip platform detection)

**Why `realpathSync`?**

When npm creates symlinks in `node_modules/.bin/`, `__filename` would point to the symlink. We need the real path to find sibling files correctly.

### Platform Detection (Lines 34-52)

```javascript
// packages/opencode/bin/opencode (lines 34-52)
const platformMap = {
  darwin: "darwin",
  linux: "linux",
  win32: "windows",
}
const archMap = {
  x64: "x64",
  arm64: "arm64",
  arm: "arm",
}

let platform = platformMap[os.platform()]
if (!platform) {
  platform = os.platform()
}
let arch = archMap[os.arch()]
if (!arch) {
  arch = os.arch()
}
const base = "opencode-" + platform + "-" + arch
const binary = platform === "windows" ? "opencode.exe" : "opencode"
```

**Platform Detection Flow**:

```
┌────────────────────┐     ┌─────────────────────────────────────────┐
│   os.platform()    │────▶│  darwin → "darwin"                      │
│                    │     │  linux  → "linux"                       │
│                    │     │  win32  → "windows"                     │
└────────────────────┘     └─────────────────────────────────────────┘

┌────────────────────┐     ┌─────────────────────────────────────────┐
│    os.arch()       │────▶│  x64    → "x64"                         │
│                    │     │  arm64  → "arm64"                       │
│                    │     │  arm    → "arm"                         │
└────────────────────┘     └─────────────────────────────────────────┘

                           ┌─────────────────────────────────────────┐
                           │  base = "opencode-darwin-arm64"         │
                           │  binary = "opencode" (or "opencode.exe")│
                           └─────────────────────────────────────────┘
```

**Why the mapping?**

Node.js uses `win32` for Windows, but the binary packages use `windows` in their names for clarity.

---

## Execution Priority Summary

The script tries to find and run the binary in this order:

```
1. OPENCODE_BIN_PATH environment variable (if set)
         │
         ▼ (not set)
2. Cached .opencode binary next to script
         │
         ▼ (not found)
3. Platform-specific binary in node_modules
         │
         ▼ (not found)
4. Error: "package manager failed to install"
```

---

## Self-Check Questions

1. **Why does the script use `#!/usr/bin/env node` instead of `#!/usr/bin/node`?**
   <details>
   <summary>Answer</summary>
   Using `/usr/bin/env node` searches the PATH for Node.js, making the script work regardless of where Node.js is installed. A hardcoded path would fail on systems with different Node.js installation locations.
   </details>

2. **What happens if you set `OPENCODE_BIN_PATH` to a non-existent file?**
   <details>
   <summary>Answer</summary>
   The `run()` function will attempt to spawn the non-existent file, `result.error` will be set (ENOENT), and the script will print the error message and exit with code 1.
   </details>

3. **Why does the script use CommonJS (`require`) instead of ES modules (`import`)?**
   <details>
   <summary>Answer</summary>
   CommonJS has broader compatibility across Node.js versions and doesn't require the `"type": "module"` setting in package.json. For a simple launcher script, CommonJS is more reliable.
   </details>

4. **What is the purpose of `fs.realpathSync(__filename)`?**
   <details>
   <summary>Answer</summary>
   When the script is executed via a symlink (like those in `node_modules/.bin/`), `__filename` points to the symlink. `realpathSync` resolves to the actual file location, which is needed to find sibling files like the `.opencode` cache.
   </details>

---

## Exercises

### Exercise 1: Trace the Shebang

1. Create a simple script with a shebang:
   ```bash
   echo '#!/usr/bin/env node
   console.log("Hello from shebang!")
   console.log("Node version:", process.version)' > test-shebang.js
   chmod +x test-shebang.js
   ./test-shebang.js
   ```

2. What happens if you remove the shebang line and try to run `./test-shebang.js`?

### Exercise 2: Explore Platform Detection

Run this in Node.js to see your system's values:
```javascript
const os = require('os')
console.log('Platform:', os.platform())
console.log('Architecture:', os.arch())
console.log('Expected binary:', `opencode-${os.platform()}-${os.arch()}`)
```

### Exercise 3: Test OPENCODE_BIN_PATH

1. Set the environment variable to a non-existent path:
   ```bash
   OPENCODE_BIN_PATH=/nonexistent/binary npx opencode --version
   ```
2. Observe the error message
3. What does this tell you about the execution order?

---

## Further Reading

- [npm package.json bin field](https://docs.npmjs.com/cli/v10/configuring-npm/package-json#bin)
- [Node.js child_process.spawnSync](https://nodejs.org/api/child_process.html#child_processspawnsynccommand-args-options)
- [Node.js os module](https://nodejs.org/api/os.html)
- [Shebang (Unix)](https://en.wikipedia.org/wiki/Shebang_(Unix))

---

## Next Lesson

In [02-native-binary-resolution.md](./02-native-binary-resolution.md), we'll explore how the script finds and downloads the correct native binary for your platform, including AVX2 detection and musl/glibc handling.
