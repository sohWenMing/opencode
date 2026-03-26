# Module 09, Lesson 03: Desktop App with Tauri

## Learning Objectives

By the end of this lesson, you will be able to:

1. Understand what Tauri is and why it's used for the desktop app
2. Navigate the `packages/desktop/` structure
3. Explain how Tauri wraps the web app
4. Read and understand the Rust code in `src-tauri/`
5. Understand Tauri commands, events, and plugins
6. Build the desktop app

## What is Tauri?

Tauri is a **Rust-based framework** for building desktop applications with web technologies. Unlike Electron, which bundles Chromium, Tauri uses the operating system's native webview:

- **macOS**: WebKit (Safari engine)
- **Windows**: WebView2 (Edge/Chromium)
- **Linux**: WebKitGTK

```
┌─────────────────────────────────────────────────────────────────┐
│                    Tauri Architecture                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                    Frontend (Web)                        │   │
│  │                                                          │   │
│  │  ┌────────────┐  ┌────────────┐  ┌────────────┐        │   │
│  │  │  Solid.js  │  │    CSS     │  │ TypeScript │        │   │
│  │  │    UI      │  │   Styles   │  │   Logic    │        │   │
│  │  └────────────┘  └────────────┘  └────────────┘        │   │
│  │                                                          │   │
│  │              @opencode-ai/app (shared)                   │   │
│  └─────────────────────────────────────────────────────────┘   │
│                           │                                     │
│                           │ IPC (invoke/events)                 │
│                           ▼                                     │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                    Tauri Core (Rust)                     │   │
│  │                                                          │   │
│  │  ┌────────────┐  ┌────────────┐  ┌────────────┐        │   │
│  │  │  Commands  │  │   Events   │  │  Plugins   │        │   │
│  │  │ (IPC API)  │  │ (Pub/Sub)  │  │ (Features) │        │   │
│  │  └────────────┘  └────────────┘  └────────────┘        │   │
│  │                                                          │   │
│  │  ┌────────────┐  ┌────────────┐  ┌────────────┐        │   │
│  │  │  Sidecar   │  │   Window   │  │   System   │        │   │
│  │  │ Management │  │ Management │  │   Access   │        │   │
│  │  └────────────┘  └────────────┘  └────────────┘        │   │
│  └─────────────────────────────────────────────────────────┘   │
│                           │                                     │
│                           ▼                                     │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                  Native Webview                          │   │
│  │                                                          │   │
│  │  macOS: WebKit  │  Windows: WebView2  │  Linux: WebKitGTK│   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Why Tauri Over Electron?

| Feature | Tauri | Electron |
|---------|-------|----------|
| Binary Size | ~10-20 MB | ~150+ MB |
| Memory Usage | Lower | Higher |
| Security | Rust memory safety | JavaScript |
| Native Features | Via Rust | Via Node.js |
| Startup Time | Faster | Slower |

## Desktop Package Structure

```
packages/desktop/
├── src/                        # Frontend TypeScript/Solid.js
│   ├── index.tsx               # Main app entry point
│   ├── entry.tsx               # Entry router (loading vs main)
│   ├── loading.tsx             # Loading screen component
│   ├── bindings.ts             # Auto-generated Rust bindings
│   ├── menu.ts                 # Application menu
│   ├── updater.ts              # Auto-update logic
│   ├── cli.ts                  # CLI integration
│   ├── webview-zoom.ts         # Zoom controls
│   ├── styles.css              # Global styles
│   └── i18n/                   # Internationalization
│       ├── index.ts
│       ├── en.ts, de.ts, ...   # Language files
│
├── src-tauri/                  # Rust backend
│   ├── src/
│   │   ├── main.rs             # Application entry point
│   │   ├── lib.rs              # Main library (commands, setup)
│   │   ├── server.rs           # Sidecar server management
│   │   ├── cli.rs              # CLI sidecar spawning
│   │   ├── windows.rs          # Window management
│   │   ├── markdown.rs         # Markdown parsing
│   │   ├── logging.rs          # Log management
│   │   ├── constants.rs        # Constants
│   │   ├── window_customizer.rs # Window customization
│   │   └── os/                 # OS-specific code
│   │       ├── mod.rs
│   │       └── windows.rs
│   │
│   ├── Cargo.toml              # Rust dependencies
│   ├── tauri.conf.json         # Tauri configuration
│   ├── capabilities/           # Security capabilities
│   └── icons/                  # App icons
│
├── scripts/                    # Build scripts
├── package.json                # Node.js dependencies
├── vite.config.ts              # Vite bundler config
└── README.md
```

## Tauri Configuration

```json
// From packages/desktop/src-tauri/tauri.conf.json

{
  "$schema": "https://schema.tauri.app/config/2",
  "productName": "OpenCode Dev",
  "identifier": "ai.opencode.desktop.dev",
  "mainBinaryName": "OpenCode",
  "version": "../package.json",
  
  "build": {
    "beforeDevCommand": "bun run dev",      // Start Vite dev server
    "devUrl": "http://localhost:1420",      // Dev server URL
    "beforeBuildCommand": "bun run build",  // Build frontend
    "frontendDist": "../dist"               // Built frontend location
  },
  
  "app": {
    "windows": [
      {
        "label": "main",
        "create": false    // Created programmatically
      }
    ],
    "withGlobalTauri": true,
    "security": {
      "csp": null          // Content Security Policy
    },
    "macOSPrivateApi": true
  },
  
  "bundle": {
    "icon": ["icons/dev/32x32.png", "icons/dev/icon.icns", "icons/dev/icon.ico"],
    "active": true,
    "targets": ["deb", "rpm", "dmg", "nsis", "app"],
    "externalBin": ["sidecars/opencode-cli"],  // Bundled CLI binary
    "macOS": {
      "entitlements": "./entitlements.plist"
    }
  },
  
  "plugins": {
    "deep-link": {
      "desktop": {
        "schemes": ["opencode"]  // Handle opencode:// URLs
      }
    }
  }
}
```

## Rust Entry Point

```rust
// From packages/desktop/src-tauri/src/main.rs

// Prevents console window on Windows in release builds
#![cfg_attr(not(debug_assertions), windows_subsystem = "windows")]

fn main() {
    // Ensure loopback connections bypass proxy settings
    const LOOPBACK: [&str; 3] = ["127.0.0.1", "localhost", "::1"];

    let upsert = |key: &str| {
        let mut items = std::env::var(key)
            .unwrap_or_default()
            .split(',')
            .filter(|v| !v.is_empty())
            .map(|v| v.to_string())
            .collect::<Vec<_>>();

        for host in LOOPBACK {
            if !items.iter().any(|v| v.eq_ignore_ascii_case(host)) {
                items.push(host.to_string());
            }
        }
        unsafe { std::env::set_var(key, items.join(",")) };
    };

    upsert("NO_PROXY");
    upsert("no_proxy");

    // Linux-specific display backend configuration
    #[cfg(target_os = "linux")]
    {
        if let Some(backend_note) = configure_display_backend() {
            eprintln!("{backend_note}");
        }
    }

    opencode_lib::run()
}
```

## Main Library and Commands

```rust
// From packages/desktop/src-tauri/src/lib.rs

mod cli;
mod constants;
mod logging;
mod markdown;
mod os;
mod server;
mod window_customizer;
mod windows;

use tauri::{AppHandle, Manager, RunEvent, State, ipc::Channel};
use tauri_specta::Event;

// State structures for managing app state
struct ServerState {
    child: Arc<Mutex<Option<CommandChild>>>,
}

struct SidecarReady(futures::future::Shared<oneshot::Receiver<ServerReadyData>>);

// Tauri command: Kill the sidecar process
#[tauri::command]
#[specta::specta]
fn kill_sidecar(app: AppHandle) {
    let Some(server_state) = app.try_state::<ServerState>() else {
        tracing::info!("Server not running");
        return;
    };

    let Some(server_state) = server_state.child.lock().unwrap().take() else {
        tracing::info!("Server state missing");
        return;
    };

    let _ = server_state.kill();
    tracing::info!("Killed server");
}

// Tauri command: Wait for initialization to complete
#[tauri::command]
#[specta::specta]
async fn await_initialization(
    state: State<'_, SidecarReady>,
    init_state: State<'_, InitState>,
    events: Channel<InitStep>,
) -> Result<ServerReadyData, String> {
    let mut rx = init_state.current.clone();

    // Stream initialization progress to frontend
    let stream = async {
        let e = *rx.borrow();
        let _ = events.send(e);

        while rx.changed().await.is_ok() {
            let step = *rx.borrow_and_update();
            let _ = events.send(step);
            if matches!(step, InitStep::Done) {
                break;
            }
        }
    };

    // Wait for sidecar credentials
    let data = state.inner().0.clone().await
        .map_err(|_| "Failed to get sidecar data".to_string());

    let (result, _) = futures::future::join(data, stream).await;
    result
}

// Tauri command: Check if an app exists
#[tauri::command]
#[specta::specta]
fn check_app_exists(app_name: &str) -> bool {
    #[cfg(target_os = "windows")]
    { os::windows::check_windows_app(app_name) }

    #[cfg(target_os = "macos")]
    { check_macos_app(app_name) }

    #[cfg(target_os = "linux")]
    { true }
}

// Main run function
pub fn run() {
    let builder = make_specta_builder();

    #[cfg(debug_assertions)]
    export_types(&builder);  // Generate TypeScript bindings in dev

    let mut builder = tauri::Builder::default()
        .plugin(tauri_plugin_single_instance::init(|app, _args, _cwd| {
            // Focus existing window when another instance is launched
            if let Some(window) = app.get_webview_window(MainWindow::LABEL) {
                let _ = window.set_focus();
                let _ = window.unminimize();
            }
        }))
        .plugin(tauri_plugin_deep_link::init())
        .plugin(tauri_plugin_os::init())
        .plugin(tauri_plugin_window_state::Builder::new()
            .with_state_flags(window_state_flags())
            .with_denylist(&[LoadingWindow::LABEL])
            .build())
        .plugin(tauri_plugin_store::Builder::new().build())
        .plugin(tauri_plugin_dialog::init())
        .plugin(tauri_plugin_shell::init())
        .plugin(tauri_plugin_process::init())
        .plugin(tauri_plugin_opener::init())
        .plugin(tauri_plugin_clipboard_manager::init())
        .plugin(tauri_plugin_http::init())
        .plugin(tauri_plugin_notification::init())
        .plugin(crate::window_customizer::PinchZoomDisablePlugin)
        .plugin(tauri_plugin_decorum::init())
        .invoke_handler(builder.invoke_handler())
        .setup(move |app| {
            let handle = app.handle().clone();
            
            // Initialize logging
            let log_dir = app.path().app_log_dir().expect("failed to resolve app log dir");
            handle.manage(logging::init(&log_dir));

            builder.mount_events(&handle);
            tauri::async_runtime::spawn(initialize(handle));

            Ok(())
        });

    if UPDATER_ENABLED {
        builder = builder.plugin(tauri_plugin_updater::Builder::new().build());
    }

    builder
        .build(tauri::generate_context!())
        .expect("error while running tauri application")
        .run(|app, event| {
            if let RunEvent::Exit = event {
                tracing::info!("Received Exit");
                kill_sidecar(app.clone());
            }
        });
}
```

## Sidecar Server Management

```rust
// From packages/desktop/src-tauri/src/server.rs

pub fn spawn_local_server(
    app: AppHandle,
    hostname: String,
    port: u32,
    password: String,
) -> (CommandChild, HealthCheck) {
    let (child, exit) = cli::serve(&app, &hostname, port, &password);

    let health_check = HealthCheck(tokio::spawn(async move {
        let url = format!("http://{hostname}:{port}");
        let timestamp = Instant::now();

        let ready = async {
            loop {
                tokio::time::sleep(Duration::from_millis(100)).await;

                if check_health(&url, Some(&password)).await {
                    tracing::info!(elapsed = ?timestamp.elapsed(), "Server ready");
                    return Ok(());
                }
            }
        };

        let terminated = async {
            match exit.await {
                Ok(payload) => Err(format!(
                    "Sidecar terminated before becoming healthy (code={:?})",
                    payload.code
                )),
                Err(_) => Err("Sidecar terminated before becoming healthy".to_string()),
            }
        };

        tokio::select! {
            res = ready => res,
            res = terminated => res,
        }
    }));

    (child, health_check)
}

async fn check_health(url: &str, password: Option<&str>) -> bool {
    let client = reqwest::Client::builder()
        .timeout(Duration::from_secs(7))
        .no_proxy()  // Bypass proxy for localhost
        .build()
        .ok()?;
    
    let health_url = reqwest::Url::parse(url)?.join("/global/health")?;
    let mut req = client.get(health_url);

    if let Some(password) = password {
        req = req.basic_auth("opencode", Some(password));
    }

    req.send().await.map(|r| r.status().is_success()).unwrap_or(false)
}
```

## Cargo Dependencies

```toml
# From packages/desktop/src-tauri/Cargo.toml

[package]
name = "opencode-desktop"
version = "0.0.0"
description = "The open source AI coding agent"
edition = "2024"

[lib]
name = "opencode_lib"
crate-type = ["staticlib", "cdylib", "rlib"]

[dependencies]
# Tauri core and plugins
tauri = { version = "2.9.5", features = ["macos-private-api"] }
tauri-plugin-opener = "2"
tauri-plugin-deep-link = "2.4.6"
tauri-plugin-shell = "2"
tauri-plugin-dialog = "2"
tauri-plugin-updater = "2"
tauri-plugin-process = "2"
tauri-plugin-store = "2"
tauri-plugin-window-state = "2"
tauri-plugin-clipboard-manager = "2"
tauri-plugin-http = "2.5.6"
tauri-plugin-notification = "2"
tauri-plugin-single-instance = { version = "2", features = ["deep-link"] }

# Serialization
serde = { version = "1", features = ["derive"] }
serde_json = "1"

# Async runtime
tokio = { version = "1.48.0", features = ["process"] }
futures = "0.3.31"

# Type generation for TypeScript
specta = "=2.0.0-rc.22"
specta-typescript = "0.0.9"
tauri-specta = { version = "=2.0.0-rc.21", features = ["derive", "typescript"] }

# Utilities
reqwest = { version = "0.12", default-features = false, features = ["rustls-tls"] }
uuid = { version = "1.19.0", features = ["v4"] }
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }

# Platform-specific
[target.'cfg(windows)'.dependencies]
windows-sys = { version = "0.61", features = ["Win32_System_Threading"] }

[target.'cfg(target_os = "linux")'.dependencies]
gtk = "0.18.2"
webkit2gtk = "=2.0.2"

[target.'cfg(target_os = "macos")'.dependencies]
objc2 = "0.6"
objc2-web-kit = "0.3"
```

## Frontend Integration

```typescript
// From packages/desktop/src/index.tsx

import { commands, type InitStep } from "./bindings"

const createPlatform = (): Platform => {
  const os = (() => {
    const type = ostype()
    if (type === "macos" || type === "windows" || type === "linux") return type
    return undefined
  })()

  return {
    platform: "desktop",
    os,
    version: pkg.version,

    // File picker dialogs
    async openDirectoryPickerDialog(opts) {
      return await open({
        directory: true,
        multiple: opts?.multiple ?? false,
        title: opts?.title ?? t("desktop.dialog.chooseFolder"),
      })
    },

    // Open external links
    openLink(url: string) {
      void shellOpen(url).catch(() => undefined)
    },

    // Open files with specific apps
    async openPath(path: string, app?: string) {
      await commands.openPath(path, app ?? null)
    },

    // Persistent storage using Tauri Store
    storage: (() => {
      const getStore = (name: string) => Store.load(name)
      
      return (name = "default.dat") => ({
        getItem: async (key) => (await getStore(name)).get(key),
        setItem: async (key, value) => (await getStore(name)).set(key, value),
        removeItem: async (key) => (await getStore(name)).delete(key),
        // ...
      })
    })(),

    // Auto-update
    checkUpdate: async () => {
      if (!UPDATER_ENABLED) return { updateAvailable: false }
      const next = await check().catch(() => null)
      if (!next) return { updateAvailable: false }
      await next.download()
      return { updateAvailable: true, version: next.version }
    },

    // Restart app
    restart: async () => {
      await commands.killSidecar().catch(() => undefined)
      await relaunch()
    },

    // Desktop notifications
    notify: async (title, description, href) => {
      const granted = await isPermissionGranted()
      if (!granted) return
      
      const notification = new Notification(title, {
        body: description ?? "",
        icon: "https://opencode.ai/favicon-96x96-v3.png",
      })
      notification.onclick = () => {
        getCurrentWindow().show()
        handleNotificationClick(href)
      }
    },
  }
}

// Main render
render(() => {
  const platform = createPlatform()
  
  // Fetch sidecar credentials from Rust
  const [sidecar] = createResource(() => 
    commands.awaitInitialization(new Channel<InitStep>())
  )

  const servers = () => {
    const data = sidecar()
    if (!data) return []
    return [{
      displayName: t("desktop.server.local"),
      type: "sidecar",
      variant: "base",
      http: {
        url: data.url,
        username: data.username,
        password: data.password,
      },
    }]
  }

  return (
    <PlatformProvider value={platform}>
      <AppBaseProviders>
        <Show when={!sidecar.loading}>
          <AppInterface servers={servers()} />
        </Show>
      </AppBaseProviders>
    </PlatformProvider>
  )
}, root!)
```

## Type-Safe Bindings with tauri-specta

The desktop app uses `tauri-specta` to generate TypeScript bindings from Rust:

```rust
// In lib.rs
fn make_specta_builder() -> tauri_specta::Builder<tauri::Wry> {
    tauri_specta::Builder::<tauri::Wry>::new()
        .commands(tauri_specta::collect_commands![
            kill_sidecar,
            cli::install_cli,
            await_initialization,
            server::get_default_server_url,
            server::set_default_server_url,
            check_app_exists,
            wsl_path,
            open_path
        ])
        .events(tauri_specta::collect_events![
            LoadingWindowComplete,
            SqliteMigrationProgress
        ])
        .error_handling(tauri_specta::ErrorHandlingMode::Throw)
}

fn export_types(builder: &tauri_specta::Builder<tauri::Wry>) {
    builder
        .export(specta_typescript::Typescript::default(), "../src/bindings.ts")
        .expect("Failed to export typescript bindings");
}
```

This generates `packages/desktop/src/bindings.ts` with type-safe command wrappers.

## Building the Desktop App

### Prerequisites

1. Install Rust via [rustup](https://rustup.rs/)
2. Install platform-specific dependencies (see [Tauri prerequisites](https://v2.tauri.app/start/prerequisites/))

### Development

```bash
# From repo root
bun install
bun run --cwd packages/desktop tauri dev
```

### Production Build

```bash
bun run --cwd packages/desktop tauri build
```

This creates platform-specific installers in `packages/desktop/src-tauri/target/release/bundle/`.

## Self-Check Questions

1. What is the advantage of Tauri over Electron?
2. How does the desktop app communicate with the opencode CLI?
3. What is the purpose of the sidecar process?
4. How are TypeScript bindings generated from Rust code?
5. What Tauri plugins are used and what do they provide?

## Exercises

### Exercise 1: Explore the Bindings

1. Open `packages/desktop/src/bindings.ts`
2. List all available commands and events
3. Trace how a command flows from TypeScript to Rust

### Exercise 2: Add a Custom Command

1. Add a new Rust command in `lib.rs`
2. Register it with `tauri_specta`
3. Regenerate bindings and call it from TypeScript

### Exercise 3: Build for Your Platform

1. Install Tauri prerequisites for your OS
2. Run the development build
3. Create a production build
4. Examine the generated installer

## Further Reading

- [Tauri v2 Documentation](https://v2.tauri.app/)
- [Tauri Plugins](https://v2.tauri.app/plugin/)
- [tauri-specta](https://github.com/specta-rs/tauri-specta) - Type-safe bindings
- [Rust Book](https://doc.rust-lang.org/book/) - Learn Rust
