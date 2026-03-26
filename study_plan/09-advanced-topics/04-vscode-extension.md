# Module 09, Lesson 04: VS Code Extension

## Learning Objectives

By the end of this lesson, you will be able to:

1. Navigate the `sdks/vscode/` structure
2. Understand the extension entry point and activation
3. Explain VS Code extension concepts (commands, views, keybindings)
4. Understand how the extension communicates with opencode
5. Configure and use the extension

## VS Code Extension Overview

The opencode VS Code extension provides a lightweight integration that:

- Opens opencode in a VS Code terminal
- Passes file context to opencode
- Supports keyboard shortcuts for quick access

```
┌─────────────────────────────────────────────────────────────────┐
│                    VS Code Extension                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                    VS Code Editor                        │   │
│  │                                                          │   │
│  │  ┌──────────────────────────────────────────────────┐   │   │
│  │  │                 Editor Panel                      │   │   │
│  │  │                                                   │   │   │
│  │  │  @src/auth.ts                                    │   │   │
│  │  │  ┌─────────────────────────────────────────┐    │   │   │
│  │  │  │ export function login() {               │    │   │   │
│  │  │  │   // Selected code                      │    │   │   │
│  │  │  │ }                                       │    │   │   │
│  │  │  └─────────────────────────────────────────┘    │   │   │
│  │  └──────────────────────────────────────────────────┘   │   │
│  │                                                          │   │
│  │  ┌──────────────────────────────────────────────────┐   │   │
│  │  │              Terminal Panel (opencode)            │   │   │
│  │  │                                                   │   │   │
│  │  │  > opencode --port 54321                         │   │   │
│  │  │                                                   │   │   │
│  │  │  ┌─────────────────────────────────────────┐    │   │   │
│  │  │  │  opencode TUI                           │    │   │   │
│  │  │  │                                          │    │   │   │
│  │  │  │  In @src/auth.ts#L5-10                  │    │   │   │
│  │  │  │  > _                                     │    │   │   │
│  │  │  └─────────────────────────────────────────┘    │   │   │
│  │  └──────────────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## Extension Structure

```
sdks/vscode/
├── src/
│   └── extension.ts          # Main extension code
├── images/
│   ├── icon.png              # Extension icon
│   ├── button-dark.svg       # Terminal icon (dark theme)
│   └── button-light.svg      # Terminal icon (light theme)
├── package.json              # Extension manifest
├── tsconfig.json             # TypeScript config
├── esbuild.js                # Build script
├── eslint.config.mjs         # Linting config
├── .vscodeignore             # Files to exclude from package
├── .vscode-test.mjs          # Test configuration
└── script/
    ├── publish               # Publish script
    └── release               # Release script
```

## Extension Manifest (package.json)

```json
// From sdks/vscode/package.json

{
  "name": "opencode",
  "displayName": "opencode",
  "description": "opencode for VS Code",
  "version": "1.3.2",
  "publisher": "sst-dev",
  "repository": {
    "type": "git",
    "url": "https://github.com/anomalyco/opencode"
  },
  "license": "MIT",
  "icon": "images/icon.png",
  
  "engines": {
    "vscode": "^1.94.0"
  },
  
  "categories": ["Other"],
  "activationEvents": [],
  "main": "./dist/extension.js",
  
  "contributes": {
    "commands": [
      {
        "command": "opencode.openTerminal",
        "title": "Open opencode",
        "icon": {
          "light": "images/button-dark.svg",
          "dark": "images/button-light.svg"
        }
      },
      {
        "command": "opencode.openNewTerminal",
        "title": "Open opencode in new tab",
        "icon": {
          "light": "images/button-dark.svg",
          "dark": "images/button-light.svg"
        }
      },
      {
        "command": "opencode.addFilepathToTerminal",
        "title": "Add Filepath to Terminal"
      }
    ],
    
    "menus": {
      "editor/title": [
        {
          "command": "opencode.openNewTerminal",
          "group": "navigation"
        }
      ]
    },
    
    "keybindings": [
      {
        "command": "opencode.openTerminal",
        "title": "Run opencode",
        "key": "cmd+escape",
        "mac": "cmd+escape",
        "win": "ctrl+escape",
        "linux": "ctrl+escape"
      },
      {
        "command": "opencode.openNewTerminal",
        "title": "Run opencode",
        "key": "cmd+shift+escape",
        "mac": "cmd+shift+escape",
        "win": "ctrl+shift+escape",
        "linux": "ctrl+shift+escape"
      },
      {
        "command": "opencode.addFilepathToTerminal",
        "title": "opencode: Insert At-Mentioned",
        "key": "cmd+alt+k",
        "mac": "cmd+alt+k",
        "win": "ctrl+alt+K",
        "linux": "ctrl+alt+K"
      }
    ]
  }
}
```

## Extension Entry Point

```typescript
// From sdks/vscode/src/extension.ts

// Called when extension is deactivated
export function deactivate() {}

import * as vscode from "vscode"

const TERMINAL_NAME = "opencode"

// Called when extension is activated
export function activate(context: vscode.ExtensionContext) {
  
  // Command: Open opencode in a new terminal tab
  let openNewTerminalDisposable = vscode.commands.registerCommand(
    "opencode.openNewTerminal", 
    async () => {
      await openTerminal()
    }
  )

  // Command: Open or focus existing opencode terminal
  let openTerminalDisposable = vscode.commands.registerCommand(
    "opencode.openTerminal", 
    async () => {
      // Check if an opencode terminal already exists
      const existingTerminal = vscode.window.terminals.find(
        (t) => t.name === TERMINAL_NAME
      )
      
      if (existingTerminal) {
        existingTerminal.show()
        return
      }

      await openTerminal()
    }
  )

  // Command: Add current file path to opencode terminal
  let addFilepathDisposable = vscode.commands.registerCommand(
    "opencode.addFilepathToTerminal", 
    async () => {
      const fileRef = getActiveFile()
      if (!fileRef) return

      const terminal = vscode.window.activeTerminal
      if (!terminal) return

      if (terminal.name === TERMINAL_NAME) {
        // Get the port from terminal environment
        // @ts-ignore
        const port = terminal.creationOptions.env?.["_EXTENSION_OPENCODE_PORT"]
        
        // Use API if available, otherwise send text directly
        port 
          ? await appendPrompt(parseInt(port), fileRef) 
          : terminal.sendText(fileRef, false)
        
        terminal.show()
      }
    }
  )

  // Register all disposables
  context.subscriptions.push(
    openTerminalDisposable, 
    addFilepathDisposable
  )

  // Helper: Create and configure a new opencode terminal
  async function openTerminal() {
    // Generate random port for this terminal instance
    const port = Math.floor(Math.random() * (65535 - 16384 + 1)) + 16384
    
    const terminal = vscode.window.createTerminal({
      name: TERMINAL_NAME,
      iconPath: {
        light: vscode.Uri.file(context.asAbsolutePath("images/button-dark.svg")),
        dark: vscode.Uri.file(context.asAbsolutePath("images/button-light.svg")),
      },
      location: {
        viewColumn: vscode.ViewColumn.Beside,
        preserveFocus: false,
      },
      env: {
        _EXTENSION_OPENCODE_PORT: port.toString(),
        OPENCODE_CALLER: "vscode",
      },
    })

    terminal.show()
    terminal.sendText(`opencode --port ${port}`)

    // If there's an active file, add it to the prompt
    const fileRef = getActiveFile()
    if (!fileRef) return

    // Wait for opencode server to be ready
    let tries = 10
    let connected = false
    do {
      await new Promise((resolve) => setTimeout(resolve, 200))
      try {
        await fetch(`http://localhost:${port}/app`)
        connected = true
        break
      } catch (e) {}
      tries--
    } while (tries > 0)

    // Append the file reference to the prompt
    if (connected) {
      await appendPrompt(port, `In ${fileRef}`)
      terminal.show()
    }
  }

  // Helper: Append text to opencode prompt via API
  async function appendPrompt(port: number, text: string) {
    await fetch(`http://localhost:${port}/tui/append-prompt`, {
      method: "POST",
      headers: {
        "Content-Type": "application/json",
      },
      body: JSON.stringify({ text }),
    })
  }

  // Helper: Get the active file with optional line selection
  function getActiveFile() {
    const activeEditor = vscode.window.activeTextEditor
    if (!activeEditor) return

    const document = activeEditor.document
    const workspaceFolder = vscode.workspace.getWorkspaceFolder(document.uri)
    if (!workspaceFolder) return

    // Get relative path from workspace root
    const relativePath = vscode.workspace.asRelativePath(document.uri)
    let filepathWithAt = `@${relativePath}`

    // Add line numbers if there's a selection
    const selection = activeEditor.selection
    if (!selection.isEmpty) {
      const startLine = selection.start.line + 1  // Convert to 1-based
      const endLine = selection.end.line + 1

      if (startLine === endLine) {
        filepathWithAt += `#L${startLine}`           // Single line
      } else {
        filepathWithAt += `#L${startLine}-${endLine}` // Range
      }
    }

    return filepathWithAt
  }
}
```

## Key Concepts

### 1. Extension Activation

VS Code extensions are activated lazily. The `activate` function is called when:
- A registered command is executed
- An activation event occurs
- The extension is explicitly activated

```typescript
export function activate(context: vscode.ExtensionContext) {
  // Extension initialization code
  // Register commands, providers, etc.
}

export function deactivate() {
  // Cleanup code (optional)
}
```

### 2. Commands

Commands are actions that can be triggered by:
- Command Palette (Ctrl+Shift+P)
- Keyboard shortcuts
- Menu items
- Other extensions

```typescript
// Register a command
const disposable = vscode.commands.registerCommand(
  "opencode.openTerminal",  // Command ID
  async () => {              // Handler function
    // Command implementation
  }
)

// Add to subscriptions for cleanup
context.subscriptions.push(disposable)
```

### 3. Terminals

VS Code provides a terminal API for creating and managing integrated terminals:

```typescript
const terminal = vscode.window.createTerminal({
  name: "opencode",           // Terminal tab name
  iconPath: {                 // Custom icon
    light: vscode.Uri.file("icon-dark.svg"),
    dark: vscode.Uri.file("icon-light.svg"),
  },
  location: {
    viewColumn: vscode.ViewColumn.Beside,  // Open beside editor
    preserveFocus: false,                   // Focus the terminal
  },
  env: {                      // Environment variables
    MY_VAR: "value",
  },
})

terminal.show()               // Show the terminal
terminal.sendText("command")  // Send text/command
```

### 4. Keybindings

Keybindings are defined in `package.json`:

```json
{
  "keybindings": [
    {
      "command": "opencode.openTerminal",
      "key": "cmd+escape",      // macOS
      "mac": "cmd+escape",
      "win": "ctrl+escape",     // Windows
      "linux": "ctrl+escape"    // Linux
    }
  ]
}
```

### 5. Menus

Menu contributions add items to VS Code's UI:

```json
{
  "menus": {
    "editor/title": [           // Editor title bar
      {
        "command": "opencode.openNewTerminal",
        "group": "navigation"
      }
    ]
  }
}
```

## Communication with opencode

The extension communicates with opencode through:

### 1. Terminal Environment Variables

```typescript
env: {
  _EXTENSION_OPENCODE_PORT: port.toString(),  // Server port
  OPENCODE_CALLER: "vscode",                  // Identify caller
}
```

### 2. HTTP API

opencode exposes an HTTP API that the extension uses:

```typescript
// Health check
await fetch(`http://localhost:${port}/app`)

// Append text to prompt
await fetch(`http://localhost:${port}/tui/append-prompt`, {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({ text: "@src/file.ts" }),
})
```

## File Reference Format

The extension generates file references in the `@file#Lstart-end` format:

```
@src/auth.ts           # Whole file
@src/auth.ts#L5        # Line 5
@src/auth.ts#L5-10     # Lines 5-10
```

This format is recognized by opencode and allows it to:
- Load the file content
- Focus on specific lines
- Provide context-aware assistance

## Extension Development

### Building

```bash
cd sdks/vscode
bun install
bun run compile
```

### Testing

```bash
bun run test
```

### Packaging

```bash
bun run package
```

This creates a `.vsix` file that can be installed in VS Code.

### Publishing

```bash
bun run vscode:prepublish
# Then use vsce to publish
```

## Configuration Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                  Extension Activation Flow                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. User triggers command (Cmd+Escape)                          │
│     │                                                           │
│     ▼                                                           │
│  2. VS Code activates extension (if not already active)         │
│     │                                                           │
│     ▼                                                           │
│  3. activate() registers commands                               │
│     │                                                           │
│     ▼                                                           │
│  4. Command handler executes                                    │
│     │                                                           │
│     ├─── Check for existing terminal                            │
│     │    │                                                      │
│     │    ├─── Found: Focus it                                   │
│     │    │                                                      │
│     │    └─── Not found: Create new terminal                    │
│     │         │                                                 │
│     │         ├─── Generate random port                         │
│     │         │                                                 │
│     │         ├─── Set environment variables                    │
│     │         │                                                 │
│     │         ├─── Run: opencode --port <port>                  │
│     │         │                                                 │
│     │         └─── Wait for server, append file context         │
│     │                                                           │
│     ▼                                                           │
│  5. opencode TUI running in terminal                            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## Self-Check Questions

1. What triggers VS Code extension activation?
2. How does the extension communicate with opencode?
3. What is the purpose of the `_EXTENSION_OPENCODE_PORT` environment variable?
4. How are file references with line numbers formatted?
5. What VS Code APIs are used to create and manage terminals?

## Exercises

### Exercise 1: Explore the Extension

1. Install the opencode extension in VS Code
2. Open a project and use Cmd+Escape to open opencode
3. Select some code and use Cmd+Alt+K to add the file reference
4. Observe how the file reference appears in opencode

### Exercise 2: Add a New Command

1. Fork the extension code
2. Add a new command that:
   - Gets the current selection
   - Sends it to opencode with a specific prompt
3. Test your changes

### Exercise 3: Trace the Communication

1. Add logging to the extension
2. Start opencode with verbose logging
3. Trace the HTTP requests between extension and opencode
4. Document the API endpoints used

## Further Reading

- [VS Code Extension API](https://code.visualstudio.com/api)
- [Extension Manifest Reference](https://code.visualstudio.com/api/references/extension-manifest)
- [VS Code Terminal API](https://code.visualstudio.com/api/references/vscode-api#Terminal)
- [Publishing Extensions](https://code.visualstudio.com/api/working-with-extensions/publishing-extension)
