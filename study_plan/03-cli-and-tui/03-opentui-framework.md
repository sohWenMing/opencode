# Lesson 03: OpenTUI Framework

## Learning Objectives

By the end of this lesson, you will be able to:
- Understand what OpenTUI is and how it differs from browser rendering
- Identify the @opentui/* packages used in opencode
- Explain how layout and styling work in the terminal
- Understand input handling (keyboard and mouse)
- Navigate the theme system and customize colors

## What is OpenTUI?

OpenTUI is a terminal UI framework built on top of Solid.js. It provides:

- **Terminal rendering**: Draws UI to the terminal instead of a browser
- **Flexbox layout**: CSS-like layout system for terminal cells
- **Input handling**: Keyboard, mouse, and paste events
- **Theming**: Color and style management
- **Components**: Box, text, textarea, scrollbox, etc.

```
┌─────────────────────────────────────────────────────────────┐
│                    Browser Rendering                        │
│  Solid.js → DOM Elements → Browser Paint → Pixels           │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                   Terminal Rendering                        │
│  Solid.js → OpenTUI → ANSI Escape Codes → Terminal Cells    │
└─────────────────────────────────────────────────────────────┘
```

## OpenTUI Packages

OpenCode uses two main OpenTUI packages:

```json
// packages/opencode/package.json
{
  "dependencies": {
    "@opentui/core": "0.1.90",
    "@opentui/solid": "0.1.90"
  }
}
```

### @opentui/core

Low-level terminal rendering primitives:
- `RGBA` - Color representation
- `TextAttributes` - Bold, italic, underline
- `MouseButton`, `MouseEvent` - Mouse handling
- `ParsedKey` - Keyboard event parsing
- `SyntaxStyle` - Syntax highlighting
- `TerminalColors` - Terminal palette

### @opentui/solid

Solid.js integration:
- `render()` - Start the TUI application
- `useKeyboard()` - Keyboard event hook
- `useRenderer()` - Access the renderer
- `useTerminalDimensions()` - Terminal size hook
- JSX components: `<box>`, `<text>`, `<textarea>`, `<scrollbox>`

## Rendering to the Terminal

The `render()` function starts the TUI:

```typescript
// packages/opencode/src/cli/cmd/tui/app.tsx (lines 135-201)
render(
  () => {
    return (
      <ErrorBoundary fallback={...}>
        <ArgsProvider {...input.args}>
          {/* ... nested providers ... */}
          <App />
        </ArgsProvider>
      </ErrorBoundary>
    )
  },
  {
    targetFps: 60,
    gatherStats: false,
    exitOnCtrlC: false,
    useKittyKeyboard: { events: process.platform === "win32" },
    autoFocus: false,
    openConsoleOnError: false,
    consoleOptions: {
      keyBindings: [{ name: "y", ctrl: true, action: "copy-selection" }],
      onCopySelection: (text) => {
        Clipboard.copy(text).catch((error) => {
          console.error(`Failed to copy: ${error}`)
        })
      },
    },
  },
)
```

### Render Options

| Option | Description |
|--------|-------------|
| `targetFps` | Target frames per second (60) |
| `exitOnCtrlC` | Whether Ctrl+C exits (false - handled manually) |
| `useKittyKeyboard` | Enable Kitty keyboard protocol |
| `autoFocus` | Auto-focus first focusable element |
| `openConsoleOnError` | Show console on errors |

## Layout System

OpenTUI uses a flexbox-like layout system:

```typescript
<box 
  flexDirection="column"    // or "row"
  flexGrow={1}              // Grow to fill space
  flexShrink={0}            // Don't shrink
  alignItems="center"       // Cross-axis alignment
  justifyContent="flex-start"  // Main-axis alignment
  gap={1}                   // Space between children
  padding={2}               // All sides
  paddingLeft={2}           // Specific side
  width={80}                // Fixed width
  maxWidth={100}            // Maximum width
  height="100%"             // Percentage
>
  {children}
</box>
```

### Layout Example: Home Screen

```typescript
// packages/opencode/src/cli/cmd/tui/routes/home.tsx (lines 108-158)
return (
  <>
    <box flexGrow={1} alignItems="center" paddingLeft={2} paddingRight={2}>
      <box flexGrow={1} minHeight={0} />
      <box height={4} minHeight={0} flexShrink={1} />
      <box flexShrink={0}>
        <Logo />
      </box>
      <box height={1} minHeight={0} flexShrink={1} />
      <box width="100%" maxWidth={75} zIndex={1000} paddingTop={1} flexShrink={0}>
        <Prompt ref={...} hint={Hint} workspaceID={route.workspaceID} />
      </box>
      <box height={4} minHeight={0} width="100%" maxWidth={75} alignItems="center" paddingTop={3} flexShrink={1}>
        <Show when={showTips()}>
          <Tips />
        </Show>
      </box>
      <box flexGrow={1} minHeight={0} />
      <Toast />
    </box>
    <box paddingTop={1} paddingBottom={1} paddingLeft={2} paddingRight={2} flexDirection="row" flexShrink={0} gap={2}>
      <text fg={theme.textMuted}>{directory()}</text>
      {/* ... footer content ... */}
    </box>
  </>
)
```

### Layout Diagram

```
┌─────────────────────────────────────────────────────────────┐
│  flexGrow={1} (spacer)                                      │
├─────────────────────────────────────────────────────────────┤
│                          Logo                               │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────────────────────────────────────────────┐    │
│  │                    Prompt                           │    │
│  │  maxWidth={75}                                      │    │
│  └─────────────────────────────────────────────────────┘    │
├─────────────────────────────────────────────────────────────┤
│                          Tips                               │
├─────────────────────────────────────────────────────────────┤
│  flexGrow={1} (spacer)                                      │
├─────────────────────────────────────────────────────────────┤
│  Footer: directory | version                                │
└─────────────────────────────────────────────────────────────┘
```

## Text and Styling

### Basic Text

```typescript
<text fg={theme.text}>Hello, World!</text>

<text 
  fg={theme.primary}           // Foreground color
  bg={theme.background}        // Background color
  attributes={TextAttributes.BOLD}  // Text style
>
  Bold text
</text>
```

### Inline Styling with Spans

```typescript
<text fg={theme.text}>
  Normal text with <span style={{ fg: theme.error }}>error</span> and 
  <span style={{ fg: theme.success, bold: true }}>success</span>
</text>
```

### Real Example: Status Display

```typescript
// packages/opencode/src/cli/cmd/tui/routes/home.tsx (lines 62-75)
<text fg={theme.text}>
  <Switch>
    <Match when={mcpError()}>
      <span style={{ fg: theme.error }}>•</span> mcp errors{" "}
      <span style={{ fg: theme.textMuted }}>ctrl+x s</span>
    </Match>
    <Match when={true}>
      <span style={{ fg: theme.success }}>•</span>{" "}
      {Locale.pluralize(connectedMcpCount(), "{} mcp server", "{} mcp servers")}
    </Match>
  </Switch>
</text>
```

## Input Handling

### Keyboard Events

```typescript
import { useKeyboard } from "@opentui/solid"

function MyComponent() {
  useKeyboard((evt) => {
    if (evt.ctrl && evt.name === "c") {
      // Handle Ctrl+C
      evt.preventDefault()
      evt.stopPropagation()
    }
    
    if (evt.name === "escape") {
      // Handle Escape
    }
  })
}
```

### Real Example: Selection Handling

```typescript
// packages/opencode/src/cli/cmd/tui/app.tsx (lines 221-248)
useKeyboard((evt) => {
  if (!Flag.OPENCODE_EXPERIMENTAL_DISABLE_COPY_ON_SELECT) return
  if (!renderer.getSelection()) return

  // Windows Terminal-like behavior:
  // - Ctrl+C copies and dismisses selection
  // - Esc dismisses selection
  if (evt.ctrl && evt.name === "c") {
    if (!Selection.copy(renderer, toast)) {
      renderer.clearSelection()
      return
    }
    evt.preventDefault()
    evt.stopPropagation()
    return
  }

  if (evt.name === "escape") {
    renderer.clearSelection()
    evt.preventDefault()
    evt.stopPropagation()
    return
  }

  renderer.clearSelection()
})
```

### Mouse Events

```typescript
<box
  onMouseDown={(evt) => {
    if (evt.button === MouseButton.LEFT) {
      // Left click
    }
    if (evt.button === MouseButton.RIGHT) {
      // Right click
    }
  }}
  onMouseUp={(evt) => {
    // Mouse released
  }}
>
  Click me
</box>
```

### Textarea Component

```typescript
// packages/opencode/src/cli/cmd/tui/component/prompt/index.tsx (lines 849-1028)
<textarea
  placeholder={placeholderText()}
  textColor={theme.text}
  focusedTextColor={theme.text}
  minHeight={1}
  maxHeight={6}
  onContentChange={() => {
    const value = input.plainText
    setStore("prompt", "input", value)
    autocomplete.onInput(value)
  }}
  onKeyDown={async (e) => {
    if (props.disabled) {
      e.preventDefault()
      return
    }
    // Handle special keys...
  }}
  onSubmit={submit}
  onPaste={async (event: PasteEvent) => {
    // Handle paste...
  }}
  ref={(r: TextareaRenderable) => {
    input = r
    // Setup...
  }}
  cursorColor={theme.text}
  syntaxStyle={syntax()}
/>
```

## The Theme System

### Theme Structure

Themes define colors for all UI elements:

```typescript
// packages/opencode/src/cli/cmd/tui/context/theme.tsx (lines 46-99)
type ThemeColors = {
  primary: RGBA
  secondary: RGBA
  accent: RGBA
  error: RGBA
  warning: RGBA
  success: RGBA
  info: RGBA
  text: RGBA
  textMuted: RGBA
  selectedListItemText: RGBA
  background: RGBA
  backgroundPanel: RGBA
  backgroundElement: RGBA
  backgroundMenu: RGBA
  border: RGBA
  borderActive: RGBA
  borderSubtle: RGBA
  diffAdded: RGBA
  diffRemoved: RGBA
  // ... more colors for diffs, markdown, syntax
}
```

### Built-in Themes

```typescript
// packages/opencode/src/cli/cmd/tui/context/theme.tsx (lines 141-175)
export const DEFAULT_THEMES: Record<string, ThemeJson> = {
  aura,
  ayu,
  catppuccin,
  ["catppuccin-frappe"]: catppuccinFrappe,
  ["catppuccin-macchiato"]: catppuccinMacchiato,
  cobalt2,
  cursor,
  dracula,
  everforest,
  flexoki,
  github,
  gruvbox,
  kanagawa,
  material,
  matrix,
  mercury,
  monokai,
  nightowl,
  nord,
  ["one-dark"]: onedark,
  ["osaka-jade"]: osakaJade,
  opencode,
  orng,
  ["lucent-orng"]: lucentOrng,
  palenight,
  rosepine,
  solarized,
  synthwave84,
  tokyonight,
  vesper,
  vercel,
  zenburn,
  carbonfox,
}
```

### Theme JSON Format

```json
// packages/opencode/src/cli/cmd/tui/context/theme/opencode.json
{
  "$schema": "...",
  "defs": {
    "orange": "#fab283",
    "bg": "#0a0a0a"
  },
  "theme": {
    "primary": "orange",
    "secondary": "#c4a7e7",
    "accent": "orange",
    "error": "#eb6f92",
    "warning": "#f6c177",
    "success": "#9ccfd8",
    "info": "#31748f",
    "text": "#e0def4",
    "textMuted": "#6e6a86",
    "background": "bg",
    "backgroundPanel": "#1a1a1a",
    "backgroundElement": "#1f1f1f"
  }
}
```

### Using the Theme

```typescript
// packages/opencode/src/cli/cmd/tui/routes/home.tsx
import { useTheme } from "@tui/context/theme"

function Home() {
  const { theme } = useTheme()
  
  return (
    <box backgroundColor={theme.background}>
      <text fg={theme.text}>Normal text</text>
      <text fg={theme.textMuted}>Muted text</text>
      <text fg={theme.primary}>Primary color</text>
      <text fg={theme.error}>Error message</text>
    </box>
  )
}
```

### Theme Provider

```typescript
// packages/opencode/src/cli/cmd/tui/context/theme.tsx (lines 280-433)
export const { use: useTheme, provider: ThemeProvider } = createSimpleContext({
  name: "Theme",
  init: (props: { mode: "dark" | "light" }) => {
    const renderer = useRenderer()
    const config = useTuiConfig()
    const kv = useKV()
    
    const [store, setStore] = createStore({
      themes: DEFAULT_THEMES,
      mode: kv.get("theme_mode", props.mode) ?? props.mode,
      active: (config.theme ?? kv.get("theme", "opencode")) as string,
      ready: false,
    })

    // ... theme resolution logic ...

    return {
      theme: new Proxy(values(), {
        get(_target, prop) {
          return values()[prop]
        },
      }),
      get selected() {
        return store.active
      },
      all() {
        return store.themes
      },
      syntax,
      mode() {
        return store.mode
      },
      set(theme: string) {
        setStore("active", theme)
        kv.set("theme", theme)
      },
    }
  },
})
```

### Dark/Light Mode Detection

```typescript
// packages/opencode/src/cli/cmd/tui/app.tsx (lines 46-104)
async function getTerminalBackgroundColor(): Promise<"dark" | "light"> {
  if (!process.stdin.isTTY) return "dark"

  return new Promise((resolve) => {
    const handler = (data: Buffer) => {
      const str = data.toString()
      const match = str.match(/\x1b]11;([^\x07\x1b]+)/)
      if (match) {
        const color = match[1]
        // Parse RGB and calculate luminance
        const luminance = (0.299 * r + 0.587 * g + 0.114 * b) / 255
        resolve(luminance > 0.5 ? "light" : "dark")
      }
    }

    process.stdin.setRawMode(true)
    process.stdin.on("data", handler)
    // Query terminal for background color
    process.stdout.write("\x1b]11;?\x07")

    setTimeout(() => {
      resolve("dark")  // Default to dark
    }, 1000)
  })
}
```

## Dialog System

The dialog system provides modal overlays:

```typescript
// packages/opencode/src/cli/cmd/tui/ui/dialog.tsx (lines 10-57)
export function Dialog(props: ParentProps<{
  size?: "medium" | "large"
  onClose: () => void
}>) {
  const dimensions = useTerminalDimensions()
  const { theme } = useTheme()

  return (
    <box
      onMouseUp={() => props.onClose?.()}
      width={dimensions().width}
      height={dimensions().height}
      alignItems="center"
      position="absolute"
      paddingTop={dimensions().height / 4}
      left={0}
      top={0}
      backgroundColor={RGBA.fromInts(0, 0, 0, 150)}  // Semi-transparent overlay
    >
      <box
        onMouseUp={(e) => e.stopPropagation()}  // Prevent closing when clicking dialog
        width={props.size === "large" ? 80 : 60}
        maxWidth={dimensions().width - 2}
        backgroundColor={theme.backgroundPanel}
        paddingTop={1}
      >
        {props.children}
      </box>
    </box>
  )
}
```

### Using Dialogs

```typescript
import { useDialog } from "@tui/ui/dialog"

function MyComponent() {
  const dialog = useDialog()

  const showHelp = () => {
    dialog.replace(() => <DialogHelp />)
  }

  const closeDialog = () => {
    dialog.clear()
  }
}
```

## Borders and Styling

### Custom Border Characters

```typescript
// packages/opencode/src/cli/cmd/tui/component/border.tsx
export const EmptyBorder = {
  topLeft: " ",
  topRight: " ",
  bottomLeft: " ",
  bottomRight: " ",
  horizontal: " ",
  vertical: " ",
}
```

### Using Borders

```typescript
// packages/opencode/src/cli/cmd/tui/component/prompt/index.tsx (lines 831-845)
<box
  border={["left"]}
  borderColor={highlight()}
  customBorderChars={{
    ...EmptyBorder,
    vertical: "┃",
    bottomLeft: "╹",
  }}
>
  {/* Content */}
</box>
```

## Self-Check Questions

1. What is the difference between `@opentui/core` and `@opentui/solid`?
2. How does OpenTUI's layout system compare to CSS flexbox?
3. What render options are important for the TUI?
4. How does the theme system support both dark and light modes?
5. How do dialogs work in OpenTUI?

## Exercises

### Exercise 1: Explore Theme Files
1. Navigate to `packages/opencode/src/cli/cmd/tui/context/theme/`
2. Open `opencode.json` and examine the structure
3. Compare with another theme like `dracula.json`
4. Identify the color definitions and references

### Exercise 2: Trace Input Handling
1. Open `packages/opencode/src/cli/cmd/tui/app.tsx`
2. Find the `useKeyboard` hook usage
3. Trace how keyboard events are processed
4. Find where specific keys (Escape, Ctrl+C) are handled

### Exercise 3: Understand Layout
1. Open `packages/opencode/src/cli/cmd/tui/routes/home.tsx`
2. Draw a diagram of the layout structure
3. Identify flexbox properties used
4. Understand how the prompt is centered

### Exercise 4: Examine the Dialog System
1. Open `packages/opencode/src/cli/cmd/tui/ui/dialog.tsx`
2. Understand how the overlay is created
3. Find how dialogs are opened and closed
4. Look at a specific dialog like `dialog-help.tsx`

## Key Takeaways

1. **OpenTUI bridges Solid and terminal**: It translates Solid's reactive updates to ANSI escape codes

2. **Flexbox-like layout**: The layout system is familiar to web developers

3. **Comprehensive input handling**: Keyboard, mouse, and paste events are all supported

4. **Theme system is JSON-based**: Easy to create and customize themes

5. **Components are composable**: Build complex UIs from simple primitives

## Further Reading

- OpenTUI source code (if available)
- [ANSI Escape Codes](https://en.wikipedia.org/wiki/ANSI_escape_code)
- [Kitty Keyboard Protocol](https://sw.kovidgoyal.net/kitty/keyboard-protocol/)
- `packages/opencode/src/cli/cmd/tui/context/theme/` - All theme files
- `packages/opencode/src/cli/cmd/tui/ui/` - UI components
