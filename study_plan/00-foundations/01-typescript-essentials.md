# TypeScript Essentials

## Learning Objectives

By the end of this lesson, you will be able to:

- Explain what TypeScript is and why it's used in the opencode project
- Use basic types: string, number, boolean, arrays, and objects
- Understand type annotations vs type inference
- Define interfaces and type aliases
- Use generics for reusable, type-safe code
- Work with async/await and Promises
- Use ES module import/export syntax

---

## What is TypeScript?

TypeScript is a **superset of JavaScript** that adds static typing. This means:

1. All valid JavaScript is valid TypeScript
2. TypeScript adds optional type annotations
3. Types are checked at compile time, not runtime
4. TypeScript compiles to plain JavaScript

### Why opencode Uses TypeScript

The opencode project is a complex codebase with:

- Multiple packages working together
- Many contributors
- Complex data structures (sessions, messages, providers)
- API contracts between frontend and backend

TypeScript helps by:

- **Catching errors early**: Find bugs before running the code
- **Better tooling**: Autocomplete, refactoring, and documentation
- **Self-documenting code**: Types describe what functions expect and return
- **Safer refactoring**: The compiler tells you what breaks when you change things

---

## Basic Types

### Primitive Types

```typescript
// String - text values
const name: string = "opencode"

// Number - integers and decimals
const version: number = 1.3
const port: number = 4096

// Boolean - true/false
const isEnabled: boolean = true
```

### Arrays

```typescript
// Array of strings
const providers: string[] = ["anthropic", "openai", "google"]

// Array of numbers
const ports: number[] = [4096, 4097, 4098]

// Alternative syntax using generic
const messages: Array<string> = ["hello", "world"]
```

### Objects

```typescript
// Object with specific shape
const config: { port: number; host: string } = {
  port: 4096,
  host: "localhost"
}
```

### Real Example from opencode

Look at `packages/opencode/src/project/instance.ts`:

```typescript
export interface Shape {
  directory: string
  worktree: string
  project: Project.Info
}
```

This defines the shape of an "instance" - a project that opencode is working with. Every instance must have:
- A `directory` (string) - where the project lives
- A `worktree` (string) - the git worktree path
- A `project` (Project.Info) - project metadata

---

## Type Annotations vs Type Inference

TypeScript can often **infer** types automatically:

```typescript
// TypeScript infers this is a string
const message = "Hello"

// TypeScript infers this is a number
const count = 42

// TypeScript infers this is string[]
const items = ["a", "b", "c"]
```

You add **explicit annotations** when:

1. TypeScript can't infer the type
2. You want to be explicit for documentation
3. Function parameters (always annotate these)

```typescript
// Function parameters need annotations
function greet(name: string): string {
  return `Hello, ${name}`
}

// Return type can often be inferred, but explicit is clearer
function add(a: number, b: number): number {
  return a + b
}
```

### opencode Style Guide

From the project's AGENTS.md:

> Rely on type inference when possible; avoid explicit type annotations or interfaces unless necessary for exports or clarity

This means: let TypeScript infer when it can, but be explicit for public APIs.

---

## Interfaces and Type Aliases

### Interfaces

Interfaces define the **shape** of an object:

```typescript
interface User {
  id: string
  name: string
  email: string
}

const user: User = {
  id: "123",
  name: "Alice",
  email: "alice@example.com"
}
```

### Type Aliases

Type aliases can define any type, not just objects:

```typescript
// Simple alias
type ID = string

// Union type
type Status = "pending" | "active" | "completed"

// Object type (similar to interface)
type Config = {
  port: number
  host: string
}
```

### When to Use Which?

- **Interface**: For object shapes, especially when they might be extended
- **Type alias**: For unions, primitives, or complex type expressions

### Real Example from opencode

From `packages/opencode/src/util/context.ts`:

```typescript
export namespace Context {
  export class NotFound extends Error {
    constructor(public override readonly name: string) {
      super(`No context found for ${name}`)
    }
  }

  export function create<T>(name: string) {
    const storage = new AsyncLocalStorage<T>()
    return {
      use() {
        const result = storage.getStore()
        if (!result) {
          throw new NotFound(name)
        }
        return result
      },
      provide<R>(value: T, fn: () => R) {
        return storage.run(value, fn)
      },
    }
  }
}
```

This shows:
- A custom error class extending `Error`
- A generic function `create<T>` that works with any type
- Methods that return typed values

---

## Generics

Generics let you write **reusable code** that works with multiple types while maintaining type safety.

### Basic Generic Function

```typescript
// Without generics - loses type information
function identity(value: any): any {
  return value
}

// With generics - preserves type information
function identity<T>(value: T): T {
  return value
}

const str = identity("hello")  // TypeScript knows this is string
const num = identity(42)       // TypeScript knows this is number
```

### Real Examples from opencode

**1. Generic timeout function** from `packages/opencode/src/util/timeout.ts`:

```typescript
export function withTimeout<T>(promise: Promise<T>, ms: number): Promise<T> {
  let timeout: NodeJS.Timeout
  return Promise.race([
    promise.then((result) => {
      clearTimeout(timeout)
      return result
    }),
    new Promise<never>((_, reject) => {
      timeout = setTimeout(() => {
        reject(new Error(`Operation timed out after ${ms}ms`))
      }, ms)
    }),
  ])
}
```

The `<T>` means: "whatever type the input Promise resolves to, the output Promise resolves to the same type."

**2. Generic async queue** from `packages/opencode/src/util/queue.ts`:

```typescript
export class AsyncQueue<T> implements AsyncIterable<T> {
  private queue: T[] = []
  private resolvers: ((value: T) => void)[] = []

  push(item: T) {
    const resolve = this.resolvers.shift()
    if (resolve) resolve(item)
    else this.queue.push(item)
  }

  async next(): Promise<T> {
    if (this.queue.length > 0) return this.queue.shift()!
    return new Promise((resolve) => this.resolvers.push(resolve))
  }
}
```

This queue can hold any type of item - strings, numbers, objects, etc.

**3. Generic JSON reader** from `packages/opencode/src/util/filesystem.ts`:

```typescript
export async function readJson<T = any>(p: string): Promise<T> {
  return JSON.parse(await readFile(p, "utf-8"))
}
```

Usage:
```typescript
// TypeScript knows config is { port: number }
const config = await readJson<{ port: number }>("config.json")
```

**4. Generic with constraints** from `packages/opencode/src/util/fn.ts`:

```typescript
import { z } from "zod"

export function fn<T extends z.ZodType, Result>(
  schema: T, 
  cb: (input: z.infer<T>) => Result
) {
  const result = (input: z.infer<T>) => {
    const parsed = schema.parse(input)
    return cb(parsed)
  }
  result.schema = schema
  return result
}
```

The `T extends z.ZodType` means T must be a Zod schema type - you can't pass just anything.

---

## Async/Await and Promises

### Promises

A Promise represents a value that will be available in the future:

```typescript
// Creating a Promise
const promise = new Promise<string>((resolve, reject) => {
  setTimeout(() => {
    resolve("Done!")
  }, 1000)
})

// Using a Promise with .then()
promise.then(result => {
  console.log(result)  // "Done!" after 1 second
})
```

### Async/Await

`async/await` is syntactic sugar that makes Promises easier to work with:

```typescript
// Mark function as async
async function fetchData(): Promise<string> {
  // await pauses until Promise resolves
  const response = await fetch("https://api.example.com/data")
  const data = await response.json()
  return data
}
```

### Real Examples from opencode

**1. Async file operations** from `packages/opencode/src/util/filesystem.ts`:

```typescript
export async function readText(p: string): Promise<string> {
  return readFile(p, "utf-8")
}

export async function write(
  p: string, 
  content: string | Buffer | Uint8Array, 
  mode?: number
): Promise<void> {
  try {
    await writeFile(p, content)
  } catch (e) {
    if (isEnoent(e)) {
      await mkdir(dirname(p), { recursive: true })
      await writeFile(p, content)
      return
    }
    throw e
  }
}
```

**2. Concurrent work** from `packages/opencode/src/util/queue.ts`:

```typescript
export async function work<T>(
  concurrency: number, 
  items: T[], 
  fn: (item: T) => Promise<void>
) {
  const pending = [...items]
  await Promise.all(
    Array.from({ length: concurrency }, async () => {
      while (true) {
        const item = pending.pop()
        if (item === undefined) return
        await fn(item)
      }
    }),
  )
}
```

This processes items with limited concurrency - useful for rate-limited APIs.

**3. Promise.race for timeouts** from `packages/opencode/src/util/timeout.ts`:

```typescript
export function withTimeout<T>(promise: Promise<T>, ms: number): Promise<T> {
  return Promise.race([
    promise,
    new Promise<never>((_, reject) => {
      setTimeout(() => reject(new Error(`Timed out`)), ms)
    }),
  ])
}
```

---

## Import/Export Syntax (ES Modules)

opencode uses ES modules (ESM), the modern JavaScript module system.

### Named Exports

```typescript
// filesystem.ts - exporting
export function readText(p: string): Promise<string> { ... }
export function write(p: string, content: string): Promise<void> { ... }

// another-file.ts - importing
import { readText, write } from "./filesystem"
```

### Default Exports

```typescript
// config.ts - exporting
export default {
  port: 4096,
  host: "localhost"
}

// another-file.ts - importing
import config from "./config"
```

### Re-exports (Barrel Files)

From `packages/sdk/js/src/index.ts`:

```typescript
export * from "./client.js"
export * from "./server.js"

import { createOpencodeClient } from "./client.js"
import { createOpencodeServer } from "./server.js"

export async function createOpencode(options?: ServerOptions) {
  const server = await createOpencodeServer(options)
  const client = createOpencodeClient({ baseUrl: server.url })
  return { client, server }
}
```

This file:
1. Re-exports everything from `client.js` and `server.js`
2. Imports specific items for use in this file
3. Exports a new function that combines them

### Type-Only Imports

```typescript
// Import only the type, not the runtime value
import type { ServerOptions } from "./server.js"
```

This is useful when you only need the type for annotations, not the actual code.

---

## Self-Check Questions

1. What's the difference between `interface` and `type`?
2. When would you use `<T>` in a function signature?
3. What does `async` do to a function's return type?
4. Why use `import type` instead of regular `import`?
5. What's the purpose of `Promise.race()`?

---

## Exercises

### Exercise 1: Find Type Examples

Search the codebase for examples of:

1. An interface with more than 3 properties
2. A generic function with constraints (`extends`)
3. A function that returns `Promise<void>`
4. A type alias using union types (`|`)

Hint: Use your editor's search or grep for patterns like `interface `, `<T extends`, `Promise<void>`.

### Exercise 2: Trace the Types

Open `packages/opencode/src/project/instance.ts` and trace:

1. What type does `Instance.provide()` return?
2. What type does `Instance.directory` return?
3. What is the type of the `cache` variable?

### Exercise 3: Write Your Own

Create a file with:

1. An interface for a `Message` with `id`, `content`, and `timestamp`
2. A generic function `first<T>(items: T[]): T | undefined` that returns the first item
3. An async function that reads a JSON file and returns a typed result

---

## Further Reading

- [TypeScript Handbook](https://www.typescriptlang.org/docs/handbook/intro.html)
- [TypeScript Deep Dive](https://basarat.gitbook.io/typescript/)
- [Effect-TS](https://effect.website/) - Used heavily in opencode for advanced patterns
- [Zod](https://zod.dev/) - Runtime validation library used with TypeScript
