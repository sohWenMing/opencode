# Exercise 01: Installation Trace

## Overview

In this exercise, you'll trace through the opencode installation flow manually, find where binaries are cached on your system, and experiment with the `OPENCODE_BIN_PATH` environment variable.

---

## Prerequisites

- Node.js installed (v18+)
- npm, yarn, pnpm, or bun available
- Terminal access
- A text editor

---

## Exercise 1: Trace the Installation Flow

### Objective

Manually trace what happens when you run `npx opencode --version`.

### Steps

1. **Create a fresh directory**:
   ```bash
   mkdir opencode-trace-test
   cd opencode-trace-test
   npm init -y
   ```

2. **Install opencode locally**:
   ```bash
   npm install opencode
   ```

3. **Find the bin script**:
   ```bash
   # List the bin directory
   ls -la node_modules/.bin/
   
   # On Windows:
   dir node_modules\.bin\
   ```

   **Question**: What files do you see? Is `opencode` a symlink or a script?

4. **Examine the symlink target**:
   ```bash
   # Linux/macOS
   readlink node_modules/.bin/opencode
   
   # Or use ls -la to see symlink target
   ls -la node_modules/.bin/opencode
   ```

5. **Read the actual bin script**:
   ```bash
   cat node_modules/opencode/bin/opencode
   ```

   **Question**: Can you identify the three main resolution strategies in the script?

### Expected Output

Document your findings:

```
Bin symlink points to: _______________
Script location: _______________
Resolution strategies found:
1. _______________
2. _______________
3. _______________
```

---

## Exercise 2: Find the Native Binary

### Objective

Locate the actual native binary that gets executed.

### Steps

1. **List platform-specific packages**:
   ```bash
   ls node_modules/ | grep opencode
   ```

   **Question**: Which platform-specific package was installed?

2. **Find the binary inside**:
   ```bash
   # Replace with your actual platform package
   ls -la node_modules/opencode-darwin-arm64/bin/
   # or
   ls -la node_modules/opencode-linux-x64/bin/
   # or
   dir node_modules\opencode-windows-x64\bin\
   ```

3. **Check the binary type**:
   ```bash
   # Linux/macOS
   file node_modules/opencode-*/bin/opencode
   
   # Windows (PowerShell)
   Get-Item node_modules\opencode-windows-x64\bin\opencode.exe | Select-Object Name, Length
   ```

4. **Verify it's executable**:
   ```bash
   # Direct execution
   ./node_modules/opencode-darwin-arm64/bin/opencode --version
   # or your platform equivalent
   ```

### Document Your Findings

```
Platform package installed: _______________
Binary path: _______________
Binary type (from `file` command): _______________
Binary size: _______________
Version output: _______________
```

---

## Exercise 3: Experiment with OPENCODE_BIN_PATH

### Objective

Understand how the environment variable override works.

### Steps

1. **Test with a valid path**:
   ```bash
   # Set to the actual binary location
   export OPENCODE_BIN_PATH="$(pwd)/node_modules/opencode-darwin-arm64/bin/opencode"
   # Adjust path for your platform
   
   npx opencode --version
   ```

2. **Test with an invalid path**:
   ```bash
   export OPENCODE_BIN_PATH="/nonexistent/path/opencode"
   npx opencode --version
   ```

   **Question**: What error message do you see?

3. **Test with a custom script**:
   ```bash
   # Create a mock binary
   echo '#!/bin/bash
   echo "Mock opencode v999.0.0"
   echo "Arguments: $@"' > mock-opencode
   chmod +x mock-opencode
   
   export OPENCODE_BIN_PATH="$(pwd)/mock-opencode"
   npx opencode --version
   npx opencode some arguments here
   ```

   **Question**: How are arguments passed through?

4. **Clear the environment variable**:
   ```bash
   unset OPENCODE_BIN_PATH
   ```

### Document Your Findings

```
Valid path behavior: _______________
Invalid path error: _______________
Arguments passed correctly: yes/no
```

---

## Exercise 4: Understand Platform Detection

### Objective

See exactly what platform values your system reports.

### Steps

1. **Create a detection script**:
   ```bash
   cat > detect-platform.js << 'EOF'
   const os = require('os')
   const childProcess = require('child_process')
   const fs = require('fs')
   
   console.log('=== Platform Detection ===')
   console.log('os.platform():', os.platform())
   console.log('os.arch():', os.arch())
   console.log('os.type():', os.type())
   console.log('os.release():', os.release())
   
   // Map to opencode naming
   const platformMap = { darwin: 'darwin', linux: 'linux', win32: 'windows' }
   const archMap = { x64: 'x64', arm64: 'arm64', arm: 'arm' }
   
   const platform = platformMap[os.platform()] || os.platform()
   const arch = archMap[os.arch()] || os.arch()
   
   console.log('\n=== Opencode Binary Name ===')
   console.log('Expected base:', `opencode-${platform}-${arch}`)
   
   // AVX2 detection (x64 only)
   if (arch === 'x64') {
     console.log('\n=== AVX2 Detection ===')
     
     if (platform === 'linux') {
       try {
         const cpuinfo = fs.readFileSync('/proc/cpuinfo', 'utf8')
         const hasAvx2 = /(^|\s)avx2(\s|$)/i.test(cpuinfo)
         console.log('AVX2 supported:', hasAvx2)
       } catch (e) {
         console.log('Could not read /proc/cpuinfo:', e.message)
       }
     }
     
     if (platform === 'darwin') {
       try {
         const result = childProcess.spawnSync('sysctl', ['-n', 'hw.optional.avx2_0'], {
           encoding: 'utf8',
           timeout: 1500
         })
         console.log('sysctl hw.optional.avx2_0:', result.stdout.trim())
         console.log('AVX2 supported:', result.stdout.trim() === '1')
       } catch (e) {
         console.log('sysctl failed:', e.message)
       }
     }
   }
   
   // Musl detection (Linux only)
   if (platform === 'linux') {
     console.log('\n=== Musl Detection ===')
     
     if (fs.existsSync('/etc/alpine-release')) {
       console.log('Alpine detected: yes')
     } else {
       console.log('Alpine detected: no')
     }
     
     try {
       const result = childProcess.spawnSync('ldd', ['--version'], { encoding: 'utf8' })
       const output = ((result.stdout || '') + (result.stderr || '')).toLowerCase()
       console.log('ldd output contains musl:', output.includes('musl'))
     } catch (e) {
       console.log('ldd check failed:', e.message)
     }
   }
   EOF
   ```

2. **Run the detection**:
   ```bash
   node detect-platform.js
   ```

3. **Compare with installed package**:
   ```bash
   ls node_modules/ | grep opencode-
   ```

### Document Your Findings

```
Platform: _______________
Architecture: _______________
AVX2 supported: _______________
Musl-based: _______________
Expected binary: _______________
Actually installed: _______________
Match: yes/no
```

---

## Exercise 5: Simulate Different Platforms

### Objective

Understand how the resolution logic would work on different platforms.

### Steps

1. **Create a simulation script**:
   ```bash
   cat > simulate-resolution.js << 'EOF'
   // Simulate the names[] resolution for different scenarios
   
   function getNames(platform, arch, hasAvx2, isMusl) {
     const base = `opencode-${platform}-${arch}`
     const baseline = arch === 'x64' && !hasAvx2
     
     if (platform === 'linux') {
       if (isMusl) {
         if (arch === 'x64') {
           if (baseline) return [`${base}-baseline-musl`, `${base}-musl`, `${base}-baseline`, base]
           return [`${base}-musl`, `${base}-baseline-musl`, base, `${base}-baseline`]
         }
         return [`${base}-musl`, base]
       }
       
       if (arch === 'x64') {
         if (baseline) return [`${base}-baseline`, base, `${base}-baseline-musl`, `${base}-musl`]
         return [base, `${base}-baseline`, `${base}-musl`, `${base}-baseline-musl`]
       }
       return [base, `${base}-musl`]
     }
     
     if (arch === 'x64') {
       if (baseline) return [`${base}-baseline`, base]
       return [base, `${base}-baseline`]
     }
     return [base]
   }
   
   // Test scenarios
   const scenarios = [
     { name: 'macOS Apple Silicon', platform: 'darwin', arch: 'arm64', avx2: false, musl: false },
     { name: 'macOS Intel with AVX2', platform: 'darwin', arch: 'x64', avx2: true, musl: false },
     { name: 'macOS Intel without AVX2', platform: 'darwin', arch: 'x64', avx2: false, musl: false },
     { name: 'Ubuntu x64 with AVX2', platform: 'linux', arch: 'x64', avx2: true, musl: false },
     { name: 'Ubuntu x64 without AVX2', platform: 'linux', arch: 'x64', avx2: false, musl: false },
     { name: 'Alpine x64 with AVX2', platform: 'linux', arch: 'x64', avx2: true, musl: true },
     { name: 'Alpine x64 without AVX2', platform: 'linux', arch: 'x64', avx2: false, musl: true },
     { name: 'Linux arm64', platform: 'linux', arch: 'arm64', avx2: false, musl: false },
     { name: 'Windows x64 with AVX2', platform: 'windows', arch: 'x64', avx2: true, musl: false },
     { name: 'Windows x64 without AVX2', platform: 'windows', arch: 'x64', avx2: false, musl: false },
   ]
   
   console.log('Binary Resolution Priority by Platform\n')
   console.log('=' .repeat(60))
   
   for (const s of scenarios) {
     const names = getNames(s.platform, s.arch, s.avx2, s.musl)
     console.log(`\n${s.name}:`)
     names.forEach((name, i) => console.log(`  ${i + 1}. ${name}`))
   }
   EOF
   ```

2. **Run the simulation**:
   ```bash
   node simulate-resolution.js
   ```

### Questions

1. Why does Alpine Linux have different priority than Ubuntu?
2. Why do x64 platforms have more fallback options than arm64?
3. What would happen if only `opencode-linux-x64-baseline-musl` was installed on an Ubuntu system with AVX2?

---

## Exercise 6: The Cache Directory

### Objective

Understand the `.opencode` cache mechanism.

### Steps

1. **Check if cache exists**:
   ```bash
   ls -la node_modules/opencode/bin/
   ```

   **Question**: Do you see a `.opencode` file?

2. **Create a mock cache**:
   ```bash
   # Create a mock cached binary
   cp node_modules/opencode-*/bin/opencode node_modules/opencode/bin/.opencode 2>/dev/null || \
   cp node_modules/opencode-*/bin/opencode.exe node_modules/opencode/bin/.opencode 2>/dev/null
   
   # Verify
   ls -la node_modules/opencode/bin/
   ```

3. **Test cache priority**:
   ```bash
   # The script should now use .opencode instead of searching node_modules
   npx opencode --version
   ```

4. **Clean up**:
   ```bash
   rm node_modules/opencode/bin/.opencode
   ```

---

## Challenge Exercise: Cross-Platform Testing with Docker

### Objective

Test the installation on different Linux environments.

### Steps

1. **Test on Ubuntu (glibc)**:
   ```bash
   docker run --rm -it node:20 bash -c "
     npm install -g opencode &&
     which opencode &&
     ls -la /usr/local/lib/node_modules/opencode/bin/ &&
     ls /usr/local/lib/node_modules/ | grep opencode &&
     opencode --version
   "
   ```

2. **Test on Alpine (musl)**:
   ```bash
   docker run --rm -it node:20-alpine sh -c "
     npm install -g opencode &&
     which opencode &&
     ls -la /usr/local/lib/node_modules/opencode/bin/ &&
     ls /usr/local/lib/node_modules/ | grep opencode &&
     opencode --version
   "
   ```

### Questions

1. Which binary package was installed on Ubuntu vs Alpine?
2. How did the musl detection work on Alpine?

---

## Summary Checklist

After completing these exercises, you should be able to:

- [ ] Locate the bin script symlink in node_modules/.bin/
- [ ] Find the actual bin/opencode script
- [ ] Identify the platform-specific binary package
- [ ] Use OPENCODE_BIN_PATH to override binary resolution
- [ ] Explain the three resolution strategies (env var, cache, node_modules)
- [ ] Predict which binary will be selected for a given platform
- [ ] Understand the AVX2 and musl detection mechanisms

---

## Troubleshooting

### "Command not found" after installation

1. Check if node_modules/.bin is in your PATH
2. Use `npx opencode` instead of just `opencode`
3. Verify the symlink exists: `ls -la node_modules/.bin/opencode`

### "Package manager failed to install" error

1. Check which platform packages are available: `npm view opencode optionalDependencies`
2. Manually install your platform package: `npm install opencode-linux-x64`
3. Verify the binary exists in the package

### Binary won't execute

1. Check file permissions: `chmod +x node_modules/opencode-*/bin/opencode`
2. Verify binary architecture matches your system: `file node_modules/opencode-*/bin/opencode`
3. On Linux, check for missing shared libraries: `ldd node_modules/opencode-*/bin/opencode`

---

## Next Steps

Now that you understand the installation flow, proceed to Module 02 to learn about the opencode architecture and how the binary is structured internally.
