# Exercise 01: Tool System Exploration

## Overview

In this exercise, you will explore the tool system by tracing tool calls, examining parameters and validation, understanding the permission flow, and optionally creating a simple custom tool.

## Prerequisites

- Completed lessons 01-04 of Module 07
- Familiarity with TypeScript and Zod
- Understanding of async/await patterns

## Exercise 1: Trace a Tool Call from Request to Execution

### Objective
Follow the complete path of a tool call from when the AI decides to use it until the result is returned.

### Scenario
The AI wants to read the file `src/index.ts`.

### Tasks

1. **Identify the entry point**
   
   Look at `packages/opencode/src/session/prompt.ts` and find where tool calls are processed. Answer:
   - How does the system receive tool call requests from the LLM?
   - What function handles tool execution?

2. **Trace through Tool.define**
   
   Examine `packages/opencode/src/tool/tool.ts`:
   - What happens before the actual `execute` function runs?
   - How is parameter validation performed?
   - When does truncation occur?

3. **Follow the read tool**
   
   In `packages/opencode/src/tool/read.ts`:
   - What parameters does it accept?
   - What permission does it request?
   - How does it handle different file types (text, image, directory)?

4. **Document the flow**
   
   Create a diagram or numbered list showing:
   ```
   1. LLM generates tool call JSON
   2. ???
   3. ???
   ...
   N. Result returned to LLM
   ```

### Expected Output

A written trace document (markdown) that shows:
- Each function/method called
- Key decisions made at each step
- Where errors could occur
- What data is passed between steps

---

## Exercise 2: Examine Tool Parameters and Validation

### Objective
Understand how tools define and validate their parameters using Zod schemas.

### Tasks

1. **Compare parameter schemas**
   
   Look at these tools and compare their parameter schemas:
   - `packages/opencode/src/tool/read.ts`
   - `packages/opencode/src/tool/bash.ts`
   - `packages/opencode/src/tool/edit.ts`
   - `packages/opencode/src/tool/grep.ts`

   Create a table:
   | Tool | Required Params | Optional Params | Special Validation |
   |------|-----------------|-----------------|-------------------|
   | read | | | |
   | bash | | | |
   | edit | | | |
   | grep | | | |

2. **Test validation errors**
   
   For each tool, identify what would happen with these invalid inputs:
   
   **read:**
   ```json
   { "filePath": 123 }
   ```
   
   **bash:**
   ```json
   { "command": "", "timeout": -1 }
   ```
   
   **edit:**
   ```json
   { "filePath": "test.ts", "oldString": "hello", "newString": "hello" }
   ```

3. **Understand .describe()**
   
   Find 5 examples of `.describe()` usage in tool parameters. Explain why the description matters for the AI.

### Expected Output

- Completed parameter comparison table
- Written explanation of each validation error
- List of 5 `.describe()` examples with explanations

---

## Exercise 3: Understand the Permission Flow

### Objective
Trace how permissions are requested, evaluated, and granted.

### Tasks

1. **Map permission types to tools**
   
   Create a mapping of which tools use which permission types:
   ```
   Permission Type    | Tools Using It
   -------------------|---------------
   read               | read, ???
   edit               | edit, write, ???
   bash               | bash
   ...
   ```

2. **Trace a permission request**
   
   Given this bash command: `npm install express`
   
   Trace through:
   - `packages/opencode/src/tool/bash.ts` - How is the permission request created?
   - `packages/opencode/src/permission/arity.ts` - What is the command arity?
   - `packages/opencode/src/permission/index.ts` - How is the request evaluated?
   - `packages/opencode/src/permission/evaluate.ts` - What rules are checked?

3. **Simulate rule evaluation**
   
   Given these rules:
   ```typescript
   const rules = [
     { permission: "bash", pattern: "*", action: "ask" },
     { permission: "bash", pattern: "npm *", action: "allow" },
     { permission: "bash", pattern: "npm publish *", action: "deny" },
   ]
   ```
   
   What is the result for each command?
   - `npm install`
   - `npm run build`
   - `npm publish --access public`
   - `yarn add react`
   - `git push`

4. **Understand "always" patterns**
   
   Look at how different tools set their `always` patterns:
   - Why does `read` use `["*"]` for always?
   - Why does `bash` use command-based patterns?
   - What happens when a user approves with "always"?

### Expected Output

- Permission type to tool mapping
- Step-by-step trace of the npm install permission flow
- Answers for all 5 rule evaluation scenarios
- Explanation of "always" pattern behavior

---

## Exercise 4: Create a Simple Custom Tool (Advanced)

### Objective
Create a custom tool that counts lines of code in files matching a pattern.

### Requirements

The tool should:
- Accept a glob pattern parameter
- Accept an optional file extension filter
- Return the total line count and file count
- Request appropriate permissions

### Starter Template

Create a file `packages/opencode/src/tool/linecount.ts`:

```typescript
import z from "zod"
import { Tool } from "./tool"
import { Instance } from "../project/instance"
import { Ripgrep } from "../file/ripgrep"
import { Filesystem } from "../util/filesystem"
import path from "path"

export const LineCountTool = Tool.define("linecount", {
  description: `Count lines of code in files matching a pattern.
  
Use this tool when you need to:
- Get statistics about code size
- Compare file sizes
- Understand project scope

The tool returns the total line count and number of files matched.`,

  parameters: z.object({
    // TODO: Define parameters
    // - pattern: glob pattern (required)
    // - extension: file extension filter (optional)
  }),

  async execute(params, ctx) {
    // TODO: Implement
    // 1. Request permission (what type?)
    // 2. Find files matching pattern
    // 3. Count lines in each file
    // 4. Return results
    
    return {
      title: "Line count",
      metadata: {
        // TODO: What metadata to include?
      },
      output: "TODO: Format output",
    }
  },
})
```

### Tasks

1. **Define the parameters**
   - What Zod types should you use?
   - What descriptions should you provide?
   - Are there any constraints (min/max values)?

2. **Implement permission request**
   - What permission type is appropriate?
   - What patterns should be used?
   - What should the "always" patterns be?

3. **Implement the counting logic**
   - Use `Ripgrep.files()` to find files
   - Use `Filesystem.readText()` to read files
   - Handle errors gracefully

4. **Format the output**
   - What information should the AI receive?
   - How should it be formatted?
   - What metadata is useful?

5. **Register the tool**
   - Add it to `packages/opencode/src/tool/registry.ts`
   - Test it works

### Expected Output

- Complete implementation of `linecount.ts`
- Updated `registry.ts` with the new tool
- Documentation of design decisions

---

## Bonus Challenges

### Challenge 1: Add Truncation Handling

Modify your linecount tool to handle the case where there are thousands of files. Implement custom truncation that:
- Shows the first 50 files
- Includes a summary of truncated files
- Saves full results to a file

### Challenge 2: Add Progress Updates

Use `ctx.metadata()` to show progress as files are counted:
```typescript
ctx.metadata({
  metadata: {
    filesProcessed: 10,
    totalFiles: 100,
    currentFile: "src/index.ts",
  },
})
```

### Challenge 3: Create a Permission Rule

Write a configuration that:
- Allows linecount for `*.ts` and `*.js` files
- Asks for other file types
- Denies access to `node_modules`

---

## Submission Checklist

- [ ] Exercise 1: Tool call trace document
- [ ] Exercise 2: Parameter comparison table and validation analysis
- [ ] Exercise 3: Permission flow documentation
- [ ] Exercise 4: Custom tool implementation (if attempted)
- [ ] Bonus challenges (optional)

## Reflection Questions

After completing these exercises, consider:

1. Why does opencode use Zod for parameter validation instead of runtime type checking?

2. What are the trade-offs between strict permissions (deny by default) and permissive permissions (allow by default)?

3. How does the tool abstraction make it easy to add new capabilities?

4. What would you change about the tool system design?

5. How would you implement a tool that needs to run for a long time (minutes)?
