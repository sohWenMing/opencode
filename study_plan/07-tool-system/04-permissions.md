# Lesson 04: Permission System

## Learning Objectives

By the end of this lesson, you will be able to:
- Explain why permissions exist in opencode
- Understand the permission folder structure
- Describe permission types and how they work
- Explain how permissions are requested and granted
- Understand auto-approve vs manual approval

## Why Permissions Exist

Permissions are a **safety mechanism** that prevents the AI from performing potentially dangerous actions without user consent. They provide:

1. **User Control**: Users decide what actions are allowed
2. **Transparency**: Users see what the AI wants to do
3. **Safety**: Prevents accidental file deletion, command execution, etc.
4. **Auditability**: Creates a record of approved actions

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Permission Flow                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│   AI wants to run: rm -rf node_modules                              │
│                        │                                             │
│                        ▼                                             │
│   ┌────────────────────────────────────────┐                        │
│   │         Permission Check               │                        │
│   │  • Check config rules                  │                        │
│   │  • Check session approvals             │                        │
│   └────────────────────────────────────────┘                        │
│                        │                                             │
│         ┌──────────────┼──────────────┐                             │
│         ▼              ▼              ▼                             │
│    ┌─────────┐   ┌─────────┐   ┌─────────┐                         │
│    │ ALLOW   │   │  ASK    │   │  DENY   │                         │
│    │ Execute │   │ Prompt  │   │ Reject  │                         │
│    │ action  │   │ user    │   │ action  │                         │
│    └─────────┘   └────┬────┘   └─────────┘                         │
│                       │                                              │
│                       ▼                                              │
│              ┌─────────────────┐                                    │
│              │  User Decision  │                                    │
│              │  • Once         │                                    │
│              │  • Always       │                                    │
│              │  • Reject       │                                    │
│              └─────────────────┘                                    │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

## Permission Folder Structure

```
packages/opencode/src/permission/
├── index.ts      # Main Permission namespace
├── evaluate.ts   # Rule evaluation logic
├── schema.ts     # Permission ID schema
└── arity.ts      # Bash command arity detection
```

## Permission Types

Permissions are categorized by the type of action:

| Permission | Description | Example |
|------------|-------------|---------|
| `read` | Reading files | `read /etc/passwd` |
| `edit` | Modifying files | `edit src/index.ts` |
| `bash` | Running commands | `bash npm install` |
| `glob` | Finding files | `glob *.ts` |
| `grep` | Searching code | `grep TODO` |
| `task` | Spawning subagents | `task explore` |
| `todowrite` | Updating todos | `todowrite` |
| `webfetch` | Fetching URLs | `webfetch https://...` |
| `websearch` | Web searches | `websearch react hooks` |
| `external_directory` | Accessing outside project | `/home/user/other` |

## Permission Rules

Rules define what actions are allowed, denied, or need asking.

### Rule Structure

From `packages/opencode/src/permission/index.ts`:

```typescript
export const Rule = z.object({
  permission: z.string(),  // Permission type (e.g., "bash", "read")
  pattern: z.string(),     // Pattern to match (e.g., "npm *", "*.ts")
  action: Action,          // "allow", "deny", or "ask"
})

export const Action = z.enum(["allow", "deny", "ask"])
```

### Example Rules

```typescript
const rules: Ruleset = [
  // Allow all read operations
  { permission: "read", pattern: "*", action: "allow" },
  
  // Allow npm commands
  { permission: "bash", pattern: "npm *", action: "allow" },
  
  // Deny rm commands
  { permission: "bash", pattern: "rm *", action: "deny" },
  
  // Ask for other bash commands
  { permission: "bash", pattern: "*", action: "ask" },
]
```

## Permission Evaluation

The `evaluate` function determines what action to take:

From `packages/opencode/src/permission/evaluate.ts`:

```typescript
export function evaluate(permission: string, pattern: string, ...rulesets: Rule[][]): Rule {
  const rules = rulesets.flat()
  
  // Find the last matching rule (later rules take precedence)
  const match = rules.findLast(
    (rule) => Wildcard.match(permission, rule.permission) && Wildcard.match(pattern, rule.pattern),
  )
  
  // Default to "ask" if no rule matches
  return match ?? { action: "ask", permission, pattern: "*" }
}
```

### Evaluation Order

Rules are evaluated in order, with **later rules taking precedence**:

```typescript
const rules = [
  { permission: "bash", pattern: "*", action: "ask" },      // General rule
  { permission: "bash", pattern: "npm *", action: "allow" }, // Specific override
]

// "npm install" matches both, but the later rule wins → "allow"
evaluate("bash", "npm install", rules)  // Returns: { action: "allow", ... }
```

## Permission Request Structure

When a tool needs permission, it creates a request:

```typescript
export const Request = z.object({
  id: PermissionID.zod,
  sessionID: SessionID.zod,
  permission: z.string(),           // Type of permission
  patterns: z.string().array(),     // Patterns being requested
  metadata: z.record(z.string(), z.any()),  // Additional context
  always: z.string().array(),       // Patterns to remember if "always" chosen
  tool: z.object({
    messageID: MessageID.zod,
    callID: z.string(),
  }).optional(),
})
```

### Example Request

```typescript
{
  id: "permission_abc123",
  sessionID: "session_xyz",
  permission: "bash",
  patterns: ["rm -rf node_modules"],
  metadata: { description: "Remove node_modules directory" },
  always: ["rm *"],  // If user says "always", approve all rm commands
  tool: { messageID: "msg_123", callID: "call_456" },
}
```

## The Permission.ask Function

Tools use `ctx.ask()` to request permission:

```typescript
// From a tool's execute function
await ctx.ask({
  permission: "bash",
  patterns: ["npm install"],
  always: ["npm *"],  // Remember approval for all npm commands
  metadata: { description: "Install dependencies" },
})
```

### How ask() Works

From `packages/opencode/src/permission/index.ts`:

```typescript
const ask = Effect.fn("Permission.ask")(function* (input: z.infer<typeof AskInput>) {
  const { approved, pending } = yield* InstanceState.get(state)
  const { ruleset, ...request } = input
  let needsAsk = false

  // Check each pattern against rules
  for (const pattern of request.patterns) {
    const rule = evaluate(request.permission, pattern, ruleset, approved)
    
    if (rule.action === "deny") {
      // Immediately reject
      return yield* new DeniedError({ ruleset: relevantRules })
    }
    
    if (rule.action === "allow") {
      continue  // Already approved
    }
    
    needsAsk = true  // Need to ask user
  }

  if (!needsAsk) return  // All patterns approved

  // Create pending request
  const id = request.id ?? PermissionID.ascending()
  const info: Request = { id, ...request }
  
  const deferred = yield* Deferred.make<void, RejectedError | CorrectedError>()
  pending.set(id, { info, deferred })
  
  // Publish event for UI
  void Bus.publish(Event.Asked, info)
  
  // Wait for user response
  return yield* Deferred.await(deferred)
})
```

## User Responses

Users can respond to permission requests in three ways:

### 1. Once (Approve this specific request)

```typescript
if (input.reply === "once") {
  yield* Deferred.succeed(existing.deferred, undefined)
  return  // Don't remember the approval
}
```

### 2. Always (Approve and remember)

```typescript
if (input.reply === "always") {
  yield* Deferred.succeed(existing.deferred, undefined)
  
  // Add patterns to approved list
  for (const pattern of existing.info.always) {
    approved.push({
      permission: existing.info.permission,
      pattern,
      action: "allow",
    })
  }
  
  // Auto-approve other pending requests that now match
  for (const [id, item] of pending.entries()) {
    const ok = item.info.patterns.every(
      (pattern) => evaluate(item.info.permission, pattern, approved).action === "allow",
    )
    if (ok) {
      pending.delete(id)
      yield* Deferred.succeed(item.deferred, undefined)
    }
  }
}
```

### 3. Reject (Deny the request)

```typescript
if (input.reply === "reject") {
  yield* Deferred.fail(
    existing.deferred,
    input.message ? new CorrectedError({ feedback: input.message }) : new RejectedError(),
  )

  // Also reject all other pending requests in this session
  for (const [id, item] of pending.entries()) {
    if (item.info.sessionID !== existing.info.sessionID) continue
    pending.delete(id)
    yield* Deferred.fail(item.deferred, new RejectedError())
  }
}
```

## Permission Errors

Three types of permission errors:

```typescript
// User explicitly rejected
export class RejectedError extends Schema.TaggedErrorClass<RejectedError>()("PermissionRejectedError", {}) {
  override get message() {
    return "The user rejected permission to use this specific tool call."
  }
}

// User rejected with feedback
export class CorrectedError extends Schema.TaggedErrorClass<CorrectedError>()("PermissionCorrectedError", {
  feedback: Schema.String,
}) {
  override get message() {
    return `The user rejected permission with feedback: ${this.feedback}`
  }
}

// Rule explicitly denies
export class DeniedError extends Schema.TaggedErrorClass<DeniedError>()("PermissionDeniedError", {
  ruleset: Schema.Any,
}) {
  override get message() {
    return `A rule prevents this tool call. Relevant rules: ${JSON.stringify(this.ruleset)}`
  }
}
```

## Configuration-Based Permissions

Permissions can be configured in `opencode.json`:

```json
{
  "permission": {
    "read": "allow",
    "edit": "ask",
    "bash": {
      "npm *": "allow",
      "git *": "allow",
      "rm *": "deny",
      "*": "ask"
    }
  }
}
```

### Converting Config to Rules

From `packages/opencode/src/permission/index.ts`:

```typescript
export function fromConfig(permission: Config.Permission) {
  const ruleset: Ruleset = []
  
  for (const [key, value] of Object.entries(permission)) {
    if (typeof value === "string") {
      // Simple: "read": "allow"
      ruleset.push({ permission: key, action: value, pattern: "*" })
      continue
    }
    
    // Complex: "bash": { "npm *": "allow", ... }
    ruleset.push(
      ...Object.entries(value).map(([pattern, action]) => ({
        permission: key,
        pattern: expand(pattern),  // Expand ~ to home directory
        action,
      })),
    )
  }
  
  return ruleset
}

function expand(pattern: string): string {
  if (pattern.startsWith("~/")) return os.homedir() + pattern.slice(1)
  if (pattern === "~") return os.homedir()
  return pattern
}
```

## External Directory Permissions

Special handling for accessing files outside the project:

From `packages/opencode/src/tool/external-directory.ts`:

```typescript
export async function assertExternalDirectory(ctx: Tool.Context, target?: string, options?: Options) {
  if (!target) return
  if (options?.bypass) return
  if (Instance.containsPath(target)) return  // Inside project, no special permission needed

  const kind = options?.kind ?? "file"
  const parentDir = kind === "directory" ? target : path.dirname(target)
  const glob = path.join(parentDir, "*").replaceAll("\\", "/")

  await ctx.ask({
    permission: "external_directory",
    patterns: [glob],
    always: [glob],
    metadata: { filepath: target, parentDir },
  })
}
```

## Disabled Tools

Some tools can be completely disabled by permissions:

```typescript
const EDIT_TOOLS = ["edit", "write", "apply_patch", "multiedit"]

export function disabled(tools: string[], ruleset: Ruleset): Set<string> {
  const result = new Set<string>()
  
  for (const tool of tools) {
    const permission = EDIT_TOOLS.includes(tool) ? "edit" : tool
    const rule = ruleset.findLast((rule) => Wildcard.match(permission, rule.permission))
    
    if (rule?.pattern === "*" && rule.action === "deny") {
      result.add(tool)
    }
  }
  
  return result
}
```

## Permission Events

The permission system publishes events for the UI:

```typescript
export const Event = {
  Asked: BusEvent.define("permission.asked", Request),
  Replied: BusEvent.define(
    "permission.replied",
    z.object({
      sessionID: SessionID.zod,
      requestID: PermissionID.zod,
      reply: Reply,
    }),
  ),
}
```

## Self-Check Questions

1. What are the three possible actions for a permission rule?
2. How does rule precedence work in evaluation?
3. What's the difference between "once" and "always" approval?
4. When does a `DeniedError` occur vs a `RejectedError`?
5. How are external directory accesses handled?

## Exercises

### Exercise 1: Write Permission Rules
Write a ruleset that:
- Allows all read operations
- Allows git and npm commands
- Denies any command containing "sudo"
- Asks for everything else

### Exercise 2: Trace Permission Evaluation
Given these rules:
```typescript
[
  { permission: "bash", pattern: "*", action: "ask" },
  { permission: "bash", pattern: "npm *", action: "allow" },
  { permission: "bash", pattern: "npm publish", action: "deny" },
]
```
What would be the result for:
- `npm install`
- `npm publish`
- `yarn add react`

### Exercise 3: Permission Flow Diagram
Draw a sequence diagram showing what happens when:
1. Tool calls `ctx.ask()` for a bash command
2. No matching rule exists (defaults to "ask")
3. User approves with "always"
4. Another tool calls `ctx.ask()` for a similar command

## Further Reading

- `packages/opencode/src/permission/index.ts` - Main permission logic
- `packages/opencode/src/permission/evaluate.ts` - Rule evaluation
- `packages/opencode/src/permission/arity.ts` - Bash command arity
- `packages/opencode/src/util/wildcard.ts` - Pattern matching
- `packages/opencode/src/tool/external-directory.ts` - External directory handling
