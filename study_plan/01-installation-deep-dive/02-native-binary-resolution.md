# Lesson 02: Native Binary Resolution

## Learning Objectives

By the end of this lesson, you will be able to:

- Explain why opencode uses a native binary instead of pure JavaScript
- Understand the binary naming convention and platform variants
- Trace through the AVX2 detection logic for different platforms
- Explain how musl vs glibc detection works on Linux
- Describe the fallback chain for binary resolution
- Understand the `findBinary()` function and node_modules traversal

---

## Why a Native Binary?

opencode ships as a **native binary** rather than pure JavaScript for several reasons:

| Benefit | Explanation |
|---------|-------------|
| **Performance** | Native code runs faster than JavaScript for CPU-intensive tasks |
| **Bundling** | All dependencies compiled into a single executable |
| **Startup time** | No Node.js module resolution overhead |
| **Distribution** | Users don't need Node.js installed to run the binary |

The JavaScript `bin/opencode` script is just a **thin launcher** that finds and executes the correct native binary for the user's platform.

---

## Binary Naming Convention

### Base Pattern

```
opencode-{platform}-{arch}[-{variant}]
```

### Platform Values

| Node.js `os.platform()` | Binary Name |
|------------------------|-------------|
| `darwin` | `opencode-darwin-*` |
| `linux` | `opencode-linux-*` |
| `win32` | `opencode-windows-*` |

### Architecture Values

| Node.js `os.arch()` | Binary Name |
|--------------------|-------------|
| `x64` | `opencode-*-x64` |
| `arm64` | `opencode-*-arm64` |
| `arm` | `opencode-*-arm` |

### Variants

| Variant | When Used |
|---------|-----------|
| `-baseline` | x64 CPUs without AVX2 support |
| `-musl` | Linux with musl libc (Alpine, etc.) |
| `-baseline-musl` | x64 musl Linux without AVX2 |

### Examples

```
opencode-darwin-arm64          # macOS on Apple Silicon
opencode-darwin-x64            # macOS on Intel
opencode-linux-x64             # Linux x64 with AVX2
opencode-linux-x64-baseline    # Linux x64 without AVX2
opencode-linux-x64-musl        # Alpine Linux x64 with AVX2
opencode-windows-x64           # Windows x64
```

---

## AVX2 Detection Deep Dive

AVX2 (Advanced Vector Extensions 2) is a CPU instruction set that enables faster SIMD operations. The script detects AVX2 support to choose between optimized and baseline binaries.

### The supportsAvx2() Function

```javascript
// packages/opencode/bin/opencode (lines 56-104)
function supportsAvx2() {
  if (arch !== "x64") return false

  if (platform === "linux") {
    try {
      return /(^|\s)avx2(\s|$)/i.test(fs.readFileSync("/proc/cpuinfo", "utf8"))
    } catch {
      return false
    }
  }

  if (platform === "darwin") {
    try {
      const result = childProcess.spawnSync("sysctl", ["-n", "hw.optional.avx2_0"], {
        encoding: "utf8",
        timeout: 1500,
      })
      if (result.status !== 0) return false
      return (result.stdout || "").trim() === "1"
    } catch {
      return false
    }
  }

  if (platform === "windows") {
    const cmd =
      '(Add-Type -MemberDefinition "[DllImport(""kernel32.dll"")] public static extern bool IsProcessorFeaturePresent(int ProcessorFeature);" -Name Kernel32 -Namespace Win32 -PassThru)::IsProcessorFeaturePresent(40)'

    for (const exe of ["powershell.exe", "pwsh.exe", "pwsh", "powershell"]) {
      try {
        const result = childProcess.spawnSync(exe, ["-NoProfile", "-NonInteractive", "-Command", cmd], {
          encoding: "utf8",
          timeout: 3000,
          windowsHide: true,
        })
        if (result.status !== 0) continue
        const out = (result.stdout || "").trim().toLowerCase()
        if (out === "true" || out === "1") return true
        if (out === "false" || out === "0") return false
      } catch {
        continue
      }
    }

    return false
  }

  return false
}
```

### Platform-Specific Detection

#### Linux Detection

```javascript
// packages/opencode/bin/opencode (lines 59-65)
if (platform === "linux") {
  try {
    return /(^|\s)avx2(\s|$)/i.test(fs.readFileSync("/proc/cpuinfo", "utf8"))
  } catch {
    return false
  }
}
```

**How it works**:
1. Read `/proc/cpuinfo` (Linux virtual filesystem)
2. Search for "avx2" as a word boundary (not substring)
3. The regex `/(^|\s)avx2(\s|$)/i` ensures we match "avx2" as a complete flag

**Example `/proc/cpuinfo` excerpt**:
```
flags : fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush mmx fxsr sse sse2 ss ht syscall nx pdpe1gb rdtscp lm constant_tsc rep_good nopl xtopology cpuid pni pclmulqdq ssse3 fma cx16 pcid sse4_1 sse4_2 x2apic movbe popcnt aes xsave avx f16c rdrand hypervisor lahf_lm abm 3dnowprefetch cpuid_fault ssbd ibrs ibpb stibp ibrs_enhanced fsgsbase bmi1 avx2 smep bmi2 erms invpcid rdseed adx smap clflushopt clwb sha_ni xsaveopt xsavec xgetbv1 xsaves
```

#### macOS Detection

```javascript
// packages/opencode/bin/opencode (lines 67-78)
if (platform === "darwin") {
  try {
    const result = childProcess.spawnSync("sysctl", ["-n", "hw.optional.avx2_0"], {
      encoding: "utf8",
      timeout: 1500,
    })
    if (result.status !== 0) return false
    return (result.stdout || "").trim() === "1"
  } catch {
    return false
  }
}
```

**How it works**:
1. Run `sysctl -n hw.optional.avx2_0`
2. Returns "1" if AVX2 is supported, "0" otherwise
3. Timeout of 1500ms prevents hanging

**Note**: Apple Silicon Macs (arm64) skip this check entirely since AVX2 is an x86 feature.

#### Windows Detection

```javascript
// packages/opencode/bin/opencode (lines 80-101)
if (platform === "windows") {
  const cmd =
    '(Add-Type -MemberDefinition "[DllImport(""kernel32.dll"")] public static extern bool IsProcessorFeaturePresent(int ProcessorFeature);" -Name Kernel32 -Namespace Win32 -PassThru)::IsProcessorFeaturePresent(40)'

  for (const exe of ["powershell.exe", "pwsh.exe", "pwsh", "powershell"]) {
    try {
      const result = childProcess.spawnSync(exe, ["-NoProfile", "-NonInteractive", "-Command", cmd], {
        encoding: "utf8",
        timeout: 3000,
        windowsHide: true,
      })
      // ...
    } catch {
      continue
    }
  }
  return false
}
```

**How it works**:
1. Use PowerShell to call Windows API `IsProcessorFeaturePresent(40)`
2. Feature ID 40 = `PF_AVX2_INSTRUCTIONS_AVAILABLE`
3. Try multiple PowerShell executables for compatibility
4. `windowsHide: true` prevents console window flash

---

## Musl vs Glibc Detection (Linux)

Linux systems use different C libraries. The script detects which one to choose the correct binary.

```javascript
// packages/opencode/bin/opencode (lines 110-127)
const musl = (() => {
  try {
    if (fs.existsSync("/etc/alpine-release")) return true
  } catch {
    // ignore
  }

  try {
    const result = childProcess.spawnSync("ldd", ["--version"], { encoding: "utf8" })
    const text = ((result.stdout || "") + (result.stderr || "")).toLowerCase()
    if (text.includes("musl")) return true
  } catch {
    // ignore
  }

  return false
})()
```

### Detection Methods

| Method | What It Checks |
|--------|----------------|
| `/etc/alpine-release` | File exists only on Alpine Linux |
| `ldd --version` | Output contains "musl" for musl-based systems |

### Why This Matters

| C Library | Common Distros | Binary Suffix |
|-----------|----------------|---------------|
| glibc | Ubuntu, Debian, Fedora, CentOS | (none) |
| musl | Alpine, Void Linux | `-musl` |

Binaries compiled against glibc won't run on musl systems and vice versa.

---

## The Binary Name Resolution Chain

The `names` IIFE builds an ordered list of binary names to try:

```javascript
// packages/opencode/bin/opencode (lines 106-149)
const names = (() => {
  const avx2 = supportsAvx2()
  const baseline = arch === "x64" && !avx2

  if (platform === "linux") {
    const musl = (() => { /* ... */ })()

    if (musl) {
      if (arch === "x64") {
        if (baseline) return [`${base}-baseline-musl`, `${base}-musl`, `${base}-baseline`, base]
        return [`${base}-musl`, `${base}-baseline-musl`, base, `${base}-baseline`]
      }
      return [`${base}-musl`, base]
    }

    if (arch === "x64") {
      if (baseline) return [`${base}-baseline`, base, `${base}-baseline-musl`, `${base}-musl`]
      return [base, `${base}-baseline`, `${base}-musl`, `${base}-baseline-musl`]
    }
    return [base, `${base}-musl`]
  }

  if (arch === "x64") {
    if (baseline) return [`${base}-baseline`, base]
    return [base, `${base}-baseline`]
  }
  return [base]
})()
```

### Resolution Priority Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    Linux x64 with AVX2 (glibc)                              │
│  Try: opencode-linux-x64 → opencode-linux-x64-baseline →                    │
│       opencode-linux-x64-musl → opencode-linux-x64-baseline-musl            │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                    Linux x64 without AVX2 (glibc)                           │
│  Try: opencode-linux-x64-baseline → opencode-linux-x64 →                    │
│       opencode-linux-x64-baseline-musl → opencode-linux-x64-musl            │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                    Alpine Linux x64 with AVX2 (musl)                        │
│  Try: opencode-linux-x64-musl → opencode-linux-x64-baseline-musl →          │
│       opencode-linux-x64 → opencode-linux-x64-baseline                      │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                    macOS arm64 (Apple Silicon)                              │
│  Try: opencode-darwin-arm64                                                 │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                    Windows x64 with AVX2                                    │
│  Try: opencode-windows-x64 → opencode-windows-x64-baseline                  │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## The findBinary() Function

This function searches for the binary in node_modules directories:

```javascript
// packages/opencode/bin/opencode (lines 151-167)
function findBinary(startDir) {
  let current = startDir
  for (;;) {
    const modules = path.join(current, "node_modules")
    if (fs.existsSync(modules)) {
      for (const name of names) {
        const candidate = path.join(modules, name, "bin", binary)
        if (fs.existsSync(candidate)) return candidate
      }
    }
    const parent = path.dirname(current)
    if (parent === current) {
      return
    }
    current = parent
  }
}
```

### Search Algorithm

```
Starting from script directory, walk up the tree:

/home/user/project/node_modules/opencode/bin/
                    │
                    ▼
/home/user/project/node_modules/
    ├── opencode-linux-x64/bin/opencode     ← Check first (if AVX2)
    ├── opencode-linux-x64-baseline/bin/    ← Check second (fallback)
    └── ...
                    │
                    ▼ (not found)
/home/user/node_modules/                     ← Check parent
                    │
                    ▼ (not found)
/home/node_modules/                          ← Keep walking up
                    │
                    ▼ (not found)
/node_modules/                               ← Root reached
                    │
                    ▼ (not found)
return undefined                             ← Binary not found
```

### Why Walk Up the Tree?

npm/yarn/pnpm may hoist packages to different levels depending on:
- Workspace configuration
- Deduplication settings
- Package manager version

Walking up ensures we find the binary regardless of hoisting.

---

## Final Resolution and Execution

```javascript
// packages/opencode/bin/opencode (lines 169-179)
const resolved = findBinary(scriptDir)
if (!resolved) {
  console.error(
    "It seems that your package manager failed to install the right version of the opencode CLI for your platform. You can try manually installing " +
      names.map((n) => `\"${n}\"`).join(" or ") +
      " package",
  )
  process.exit(1)
}

run(resolved)
```

### Error Message

If no binary is found, the error message tells the user exactly which packages to try:

```
It seems that your package manager failed to install the right version of the opencode CLI for your platform. You can try manually installing "opencode-linux-x64" or "opencode-linux-x64-baseline" or "opencode-linux-x64-musl" or "opencode-linux-x64-baseline-musl" package
```

---

## Complete Resolution Flow

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         bin/opencode executed                               │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                    ┌─────────────────┴─────────────────┐
                    │  OPENCODE_BIN_PATH set?           │
                    └─────────────────┬─────────────────┘
                           yes │              │ no
                               ▼              ▼
                    ┌──────────────┐  ┌─────────────────────────────────┐
                    │ run(envPath) │  │ Check for .opencode cache       │
                    └──────────────┘  └─────────────────┬───────────────┘
                                              found │          │ not found
                                                    ▼          ▼
                                      ┌──────────────┐  ┌─────────────────┐
                                      │ run(cached)  │  │ Detect platform │
                                      └──────────────┘  │ Detect arch     │
                                                        │ Check AVX2      │
                                                        │ Check musl      │
                                                        └────────┬────────┘
                                                                 │
                                                                 ▼
                                                        ┌─────────────────┐
                                                        │ Build names[]   │
                                                        │ priority list   │
                                                        └────────┬────────┘
                                                                 │
                                                                 ▼
                                                        ┌─────────────────┐
                                                        │ findBinary()    │
                                                        │ walk up tree    │
                                                        └────────┬────────┘
                                                        found │    │ not found
                                                              ▼    ▼
                                                  ┌────────────┐  ┌─────────┐
                                                  │run(resolved)│  │ Error   │
                                                  └────────────┘  │ exit(1) │
                                                                  └─────────┘
```

---

## Self-Check Questions

1. **Why does opencode ship platform-specific binaries instead of a single JavaScript implementation?**
   <details>
   <summary>Answer</summary>
   Native binaries provide better performance, faster startup times, and can be distributed as standalone executables without requiring Node.js to be installed.
   </details>

2. **What happens on an Alpine Linux x64 system with AVX2 support?**
   <details>
   <summary>Answer</summary>
   The script detects musl (via `/etc/alpine-release` or `ldd --version`), detects AVX2 support, and tries binaries in this order: `opencode-linux-x64-musl`, `opencode-linux-x64-baseline-musl`, `opencode-linux-x64`, `opencode-linux-x64-baseline`.
   </details>

3. **Why does the Windows AVX2 detection try multiple PowerShell executables?**
   <details>
   <summary>Answer</summary>
   Different Windows versions and configurations may have PowerShell available under different names (`powershell.exe`, `pwsh.exe`, `pwsh`, `powershell`). Trying multiple ensures compatibility across Windows versions.
   </details>

4. **What is the purpose of the `-baseline` binary variant?**
   <details>
   <summary>Answer</summary>
   The baseline variant is compiled without AVX2 instructions, ensuring compatibility with older x64 CPUs that don't support AVX2. This provides a fallback for systems where the optimized binary won't run.
   </details>

5. **Why does `findBinary()` walk up the directory tree instead of just checking one location?**
   <details>
   <summary>Answer</summary>
   Package managers may hoist dependencies to different levels in the node_modules tree depending on workspace configuration and deduplication. Walking up ensures the binary is found regardless of where it was installed.
   </details>

---

## Exercises

### Exercise 1: Trace Your Platform

Run this script to see what binary opencode would look for on your system:

```javascript
const os = require('os')
const childProcess = require('child_process')
const fs = require('fs')

const platform = os.platform() === 'win32' ? 'windows' : os.platform()
const arch = os.arch()
const base = `opencode-${platform}-${arch}`

console.log('Platform:', platform)
console.log('Architecture:', arch)
console.log('Base binary name:', base)

// Check AVX2 (simplified)
if (arch === 'x64') {
  if (platform === 'linux') {
    try {
      const cpuinfo = fs.readFileSync('/proc/cpuinfo', 'utf8')
      console.log('AVX2 supported:', /(^|\s)avx2(\s|$)/i.test(cpuinfo))
    } catch (e) {
      console.log('AVX2 check failed:', e.message)
    }
  }
}
```

### Exercise 2: Find Your Installed Binary

After installing opencode, find where the binary actually lives:

```bash
# Find the bin script
npm ls opencode
# or
which opencode

# Then trace to find the actual binary
node -e "console.log(require('path').dirname(require('fs').realpathSync(process.argv[1])))" $(which opencode)
```

### Exercise 3: Test Fallback Behavior

1. Create a mock node_modules structure with only a baseline binary
2. Observe how the script falls back when the primary binary isn't found

---

## Further Reading

- [AVX2 Wikipedia](https://en.wikipedia.org/wiki/Advanced_Vector_Extensions#AVX2)
- [musl libc](https://musl.libc.org/)
- [npm optional dependencies](https://docs.npmjs.com/cli/v10/configuring-npm/package-json#optionaldependencies)
- [Node.js os.arch()](https://nodejs.org/api/os.html#osarch)

---

## Next Lesson

In [exercises/01-installation-trace.md](./exercises/01-installation-trace.md), you'll get hands-on practice tracing through the installation flow, finding cached binaries, and experimenting with environment variables.
