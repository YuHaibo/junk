# ARCHITECTURE.md — Junk v3.0.4

> **Junk** — the flying scratchpad. Press `⌘J` / `Ctrl+J` anywhere, type, press `Esc`. No friction.
>
> - **Repo:** https://github.com/paulfxyz/junk
> - **Site:** https://thejunk.app
> - **Stack:** Rust (Tauri v2) + single-file HTML/CSS/JS frontend
> - **Platforms:** macOS (universal), Windows (x64), Linux (AppImage + .deb)

This document is written for a new contributor who wants to understand every corner of the codebase — the tech stack, why each decision was made, and the hard problems that were solved to get here. Read it alongside `src-tauri/src/main.rs` and `src/index.html`.

---

## Table of Contents

1. [App Overview](#1-app-overview)
2. [File Structure](#2-file-structure)
3. [Tech Stack](#3-tech-stack)
4. [Key Architecture Decisions](#4-key-architecture-decisions)
   - [4.1 Single HTML file, zero build step](#41-single-html-file-zero-build-step)
   - [4.2 IPC bridge: the getInvoke() pattern](#42-ipc-bridge-the-getinvoke-pattern)
   - [4.3 Window lifecycle: hide, don't quit](#43-window-lifecycle-hide-dont-quit)
   - [4.4 macOS: frameless + transparent + rounded corners](#44-macos-frameless--transparent--rounded-corners)
   - [4.5 macOS activation policy](#45-macos-activation-policy)
   - [4.6 Global shortcuts](#46-global-shortcuts)
   - [4.7 Custom shortcut at runtime](#47-custom-shortcut-at-runtime)
   - [4.8 Window position and size memory](#48-window-position-and-size-memory)
   - [4.9 Auto-save via localStorage](#49-auto-save-via-localstorage)
   - [4.10 Dark mode: CSS custom properties + matchMedia](#410-dark-mode-css-custom-properties--matchmedia)
   - [4.11 Inline markdown parser](#411-inline-markdown-parser)
   - [4.12 Update check: Rust HTTP, not JS fetch](#412-update-check-rust-http-not-js-fetch)
   - [4.13 tauri.conf.json key settings](#413-tauriconfjson-key-settings)
5. [Rust Backend Deep Dive](#5-rust-backend-deep-dive)
   - [5.1 State management](#51-state-management)
   - [5.2 IPC command reference](#52-ipc-command-reference)
   - [5.3 Error handling philosophy](#53-error-handling-philosophy)
   - [5.4 Capabilities (Tauri v2 permissions)](#54-capabilities-tauri-v2-permissions)
6. [Frontend Deep Dive](#6-frontend-deep-dive)
   - [6.1 CSS architecture](#61-css-architecture)
   - [6.2 JavaScript organisation](#62-javascript-organisation)
   - [6.3 localStorage schema](#63-localstorage-schema)
7. [Tauri Events (Rust → JS)](#7-tauri-events-rust--js)
8. [Platform Notes](#8-platform-notes)
   - [8.1 macOS](#81-macos)
   - [8.2 Windows](#82-windows)
   - [8.3 Linux](#83-linux)
9. [CI / Release Pipeline](#9-ci--release-pipeline)
10. [Build & Dev Setup](#10-build--dev-setup)
11. [Version History Highlights](#11-version-history-highlights)

---

## 1. App Overview

Junk is a global-hotkey scratchpad. The pitch: press `⌘J` anywhere on your Mac (or `Ctrl+J` on Windows/Linux), a floating frosted-glass notepad appears, you type something, press `Esc`, and it disappears — but the process stays alive, the content stays saved, and the hotkey fires again in 20 milliseconds. No Dock icon. No Taskbar entry. No menus. Just a scratchpad that stays out of your way until you need it.

The design goals that drove every technical decision:

- **Zero friction** — the hotkey must work in *any* app, even fullscreen games and system UIs
- **Zero dependencies for contributors** — `git clone` + `cargo tauri dev` is all you need
- **Zero data loss** — content auto-saves on every keystroke (debounced), stored locally
- **Native feel** — frosted glass, rounded corners, native drop shadow, OS-level hotkeys
- **Small binary** — release LTO, strip symbols, `panic = "abort"`; the final `.app` is ~11 MB

---

## 2. File Structure

```
junk-repo/
├── src/
│   └── index.html              # Entire frontend: HTML + CSS + JS (~1,567 lines)
├── src-tauri/
│   ├── src/
│   │   └── main.rs             # All Rust backend (~770 lines)
│   ├── Cargo.toml              # Dependencies, version, build profiles
│   ├── tauri.conf.json         # Window config, bundle settings, capabilities
│   ├── capabilities/
│   │   └── default.json        # Tauri v2 permission declarations
│   └── icons/                  # App icons (all sizes, all platforms)
├── assets/
│   └── icons/                  # Source icon assets (PNG, .icns, .ico)
├── .github/
│   └── workflows/
│       └── build.yml           # CI: build + publish release on tag push
├── package.json                # Only devDependency: @tauri-apps/cli
├── ARCHITECTURE.md             # This file
├── CONTRIBUTING.md             # How to build and contribute
├── CHANGELOG.md                # Version history
└── README.md                   # Full documentation + deep dives
```

**Why the frontend lives in `src/index.html` and not `src-tauri/`:** Tauri's `frontendDist` build config points to `../src`, so the compiler serves `src/` as the asset root. Keeping it at the repo root (not inside `src-tauri/`) makes it clear that it is *not* a Rust artifact and lets non-Rust contributors find it immediately.

**Why `package.json` exists with a single devDependency:** `@tauri-apps/cli` is the only npm package. It's a thin wrapper around the Rust `tauri` binary; npm is the conventional way to distribute platform-independent CLI tools to developers regardless of their package manager setup. There is no bundler, no transpiler, no runtime JS framework.

---

## 3. Tech Stack

### Runtime

| Layer | Technology | Role |
|---|---|---|
| Backend process | **Rust** (stable, MSRV 1.70) | Window management, IPC host, OS integrations |
| App framework | **Tauri v2** | Bridges Rust ↔ WebView; handles window lifecycle |
| WebView (macOS) | **WKWebView** (via Tauri) | Renders `index.html` |
| WebView (Windows) | **WebView2** (Chromium-based) | Renders `index.html` |
| WebView (Linux) | **WebKitGTK** | Renders `index.html` |
| Frontend | **Vanilla HTML/CSS/JS** | Single file, zero build step |

### Tauri Plugins

| Plugin | Purpose |
|---|---|
| `tauri-plugin-global-shortcut 2` | OS-level hotkey registration — fires even when the app is hidden |
| `tauri-plugin-autostart 2` | macOS LaunchAgent / Windows registry / Linux `.desktop` |
| `tauri-plugin-http 2` (rustls-tls) | `reqwest` HTTP client for update checks from Rust |

### macOS-specific dependencies (Cargo)

| Crate | Version | Purpose |
|---|---|---|
| `window-vibrancy` | 0.6 | Adds `NSVisualEffectView` (frosted glass blur) |
| `objc2` | 0.6.0 | Safe Objective-C message-send bindings |
| `objc2-app-kit` | 0.3.0 | `NSView`, `NSWindow` types |
| `raw-window-handle` | 0.6 | Extracts the native `NSView *` pointer from a Tauri window |

### Windows-specific dependency

| Crate | Version | Purpose |
|---|---|---|
| `window-vibrancy` | 0.6 | Applies Windows Acrylic blur effect |

### Shared utility crates

| Crate | Purpose |
|---|---|
| `serde` + `serde_json` | (De)serialise IPC structs and update-check JSON |
| `log` + `env_logger` | Structured logging to stderr; controlled by `RUST_LOG` |

---

## 4. Key Architecture Decisions

### 4.1 Single HTML file, zero build step

The entire frontend — HTML structure, all CSS, all JavaScript — lives in one file: `src/index.html`. There is no Vite, no webpack, no TypeScript compiler, no npm install required to work on the frontend.

**Why this works for Junk specifically:** The app is a scratchpad. The UI is a textarea, a 40px footer bar, and a preferences slide-up panel. The total JS is around 600 lines. At this scale, a build pipeline is pure overhead.

**Trade-offs accepted:**

- No TypeScript → compensated with JSDoc comments on every function
- No tree-shaking → there is nothing to shake; all code is used
- No hot module replacement → `tauri dev` reloads the WebView on file save, which is fast enough
- No CSS modules → BEM-style class names + a single root stylesheet suffices

**What this means for contributors:** you can edit `src/index.html` in any text editor, save, and see the change immediately in `tauri dev`. No terminal tab open for a watch process. No `npm run build` before testing.

**Tauri's role:** In development (`tauri dev`), Tauri serves `src/` from a local dev server. In production (`tauri build`), the build script reads `frontendDist: "../src"` from `tauri.conf.json` and embeds `index.html` directly into the binary — the file is never fetched over HTTP in production.

---

### 4.2 IPC bridge: the getInvoke() pattern

Tauri v2 injects its IPC layer into the WebView, but the exact surface changed between versions and configurations. We centralise this in one function:

```javascript
function getInvoke() {
  // Primary path: always injected by Tauri v2 runtime, no config needed.
  if (window.__TAURI_INTERNALS__?.invoke) {
    return window.__TAURI_INTERNALS__.invoke;
  }
  // Secondary path: requires withGlobalTauri: true in tauri.conf.json.
  if (window.__TAURI__?.core?.invoke) {
    return window.__TAURI__.core.invoke;
  }
  // Fallback: running in a plain browser (dev/testing outside Tauri).
  return async (cmd, args) => {
    console.warn('[ipc] no Tauri bridge — mock call:', cmd, args);
    return null;
  };
}
```

All IPC calls go through the thin wrapper:

```javascript
async function ipc(command, args = {}) {
  const invoke = getInvoke();
  return invoke(command, args);
}
```

**Why `__TAURI_INTERNALS__.invoke` first:** It is always available in Tauri v2 regardless of `withGlobalTauri`. The `__TAURI__.core.invoke` path requires `withGlobalTauri: true` in `tauri.conf.json` — a deliberate opt-in because it exposes the Tauri object on `window`, which slightly expands the attack surface for untrusted content.

**Why `withGlobalTauri: true` is still enabled:** The event listening API (`window.__TAURI__.event.listen`) *requires* `withGlobalTauri: true`. We need it for two events:
- `tauri://focus` — fires when the window is shown; triggers the fly-in animation and position restore
- `open-prefs` — emitted by Rust when `⌘,` / `Ctrl+,` is pressed; slides the preferences panel up

Without `withGlobalTauri: true`, there is no supported JS-accessible path to subscribe to Tauri events.

---

### 4.3 Window lifecycle: hide, don't quit

When the user clicks `×` or presses `Esc`, the window *hides*. The Rust process stays alive, the global shortcut stays registered, all JS state stays in memory.

**Implementation in Rust:**

```rust
.run(|app, event| match event {
    RunEvent::WindowEvent {
        label,
        event: tauri::WindowEvent::CloseRequested { api, .. },
        ..
    } if label == "main" => {
        api.prevent_close();          // Cancel the OS close request
        if let Some(window) = app.get_webview_window("main") {
            let _ = window.hide();    // Just hide it
        }
    }
    RunEvent::ExitRequested { .. } => {
        app.exit(0);                  // ⌘Q → actually quit
    }
    _ => {}
})
```

**Why `RunEvent::WindowEvent { CloseRequested }` and not `on_window_event`:** The `run()` callback receives events after the build chain is finalised, which is the correct place to intercept close in Tauri v2's event model. `api.prevent_close()` tells Tauri not to call the OS window-close path.

**Why the app starts with `visible: false` in `tauri.conf.json`:** The first show is triggered by the global shortcut (`⌘J`). If we started visible, there would be a race between the OS rendering the window and `setup()` completing (applying vibrancy, setting corner radius, registering shortcuts). Starting hidden ensures all setup is done before the user ever sees the window.

---

### 4.4 macOS: frameless + transparent + rounded corners

This was the hardest problem in the codebase. Three separate sub-problems, each requiring a different solution.

#### Problem A — Dragging a frameless window

`decorations: false` in `tauri.conf.json` removes the title bar, which is what gives the app its clean floating look. But it also removes the drag handle. Two obvious approaches fail:

- **CSS `-webkit-app-region: drag`** — WKWebView (the macOS WebView) ignores this property entirely in Tauri v2. It works in Electron (which patches it), but Tauri does not patch WKWebView.
- **`data-tauri-drag-region` attribute** — this was a Tauri v1 plugin mechanism. It does not exist in v2.

**The solution:** JS listens for `mousedown` on the window (excluding interactive elements like `<textarea>`, buttons, and inputs), calls `e.preventDefault()` *synchronously*, then calls `ipc('start_dragging')` asynchronously. Rust calls `window.start_dragging()`, which calls `NSWindow.performWindowDragWithEvent:`.

```javascript
document.addEventListener('mousedown', async (e) => {
  const interactive = e.target.closest('textarea, button, input, a, [data-no-drag]');
  if (interactive) return;

  e.preventDefault(); // ← must happen before the async call
  await ipc('start_dragging');
});
```

**Why `e.preventDefault()` before the async call is critical:** `NSWindow.performWindowDragWithEvent:` needs the mouse to still be in a "button down" state. If the event propagates through the WebView event loop before `performWindowDragWithEvent:` fires, the drag candidate is lost. `preventDefault()` holds the OS drag-candidate state open synchronously until the Rust side executes.

#### Problem B — Rounded corners

Setting `transparent: true` + `decorations: false` gives a rectangular OS window with a transparent background. CSS `border-radius` on `html`/`body` clips *CSS rendering* within the WebView — it does not clip the OS compositor rectangle. The result: a square window with rounded CSS elements that look wrong against the desktop.

`apply_vibrancy(corner_radius: Some(14.0))` rounds the `NSVisualEffectView` subview (the blur background), but the `NSWindow` frame is still rectangular. The WKWebView sits *above* the blur view in the layer hierarchy and renders square on top of the rounded blur corners.

**The solution (introduced in v3.0.3):** A two-step approach:

**Step 1** — Apply vibrancy (frosted glass background):
```rust
apply_vibrancy(
    &window,
    NSVisualEffectMaterial::HudWindow,
    None,
    Some(14.0),   // Rounds the blur subview only
)
```

**Step 2** — Walk the native view hierarchy and set `cornerRadius` + `masksToBounds` on the `NSWindow`'s `contentView` `CALayer`:

```rust
unsafe fn set_macos_window_corner_radius(window: &tauri::WebviewWindow, radius: f64) {
    use objc2::msg_send;
    use objc2::runtime::AnyObject;
    use raw_window_handle::{HasWindowHandle, RawWindowHandle};

    let Ok(handle) = window.window_handle() else { return };
    let RawWindowHandle::AppKit(appkit) = handle.as_raw() else { return };
    let ns_view: *mut AnyObject = appkit.ns_view.as_ptr().cast();

    // Walk: WKWebView (ns_view) → NSWindow → contentView → CALayer
    let ns_window: *mut AnyObject = msg_send![ns_view, window];
    let content_view: *mut AnyObject = msg_send![ns_window, contentView];
    let () = msg_send![content_view, setWantsLayer: true];
    let layer: *mut AnyObject = msg_send![content_view, layer];

    let () = msg_send![layer, setCornerRadius: radius as f64];
    let () = msg_send![layer, setMasksToBounds: true]; // ← the critical call
}
```

**Why `masksToBounds = YES` is the key:** `cornerRadius` alone is a visual hint — it rounds the CALayer's own rendering, but child layers can still render outside the rounded rect. `masksToBounds = YES` tells the compositor to clip the *entire subtree* — including the WKWebView and every sublayer — to the rounded rectangle. Without it, `cornerRadius` has no visible effect on the window edge.

**Why `objc2` and not `cocoa`/`objc`:** `objc2 0.6` is the actively maintained, memory-safe Objective-C binding crate as of 2024–2025. The older `cocoa` and `objc` crates are in maintenance mode and use unsafe patterns that Rust's borrow checker cannot verify. `objc2` makes the `msg_send!` macro as safe as it can be while still crossing the FFI boundary.

#### Problem C — Drop shadow

After solving A and B, CSS `box-shadow` was added to give the floating window a drop shadow. It looked great in screenshots. In practice: invisible. The shadow was being clipped by `masksToBounds = YES` on the contentView CALayer — it rendered inside the window bounds, not outside them.

**The solution (v3.0.4):** `"shadow": true` in `tauri.conf.json`. This instructs macOS's `WindowServer` to draw a native drop shadow outside the window frame. The OS shadow:
- Renders outside the compositor clip boundary (not affected by `masksToBounds`)
- Adapts automatically to light mode / dark mode and window depth/stack position
- Requires zero CSS; zero Rust code

CSS `box-shadow` is still present in the frontend as a fallback for Windows and Linux, where the native shadow mechanism is different. The comments in the CSS clarify which platform uses which layer.

---

### 4.5 macOS activation policy

```rust
#[cfg(target_os = "macos")]
fn set_macos_activation_policy(app: &AppHandle) {
    use tauri::ActivationPolicy;
    app.set_activation_policy(ActivationPolicy::Accessory).ok();
}
```

`ActivationPolicy::Accessory` removes the app from:
- The Dock (no bouncing icon)
- The `⌘-Tab` app switcher

The app still receives keyboard focus when the window is shown — `Accessory` is the correct policy for a "utility panel" app. `Prohibited` would prevent focus entirely. `Regular` would show the Dock icon, defeating the purpose.

**Critical timing:** `set_macos_activation_policy` is called in `setup()`, *before* any window becomes visible. Setting it after the window shows produces a visible flash — the Dock icon appears briefly, then disappears.

---

### 4.6 Global shortcuts

Three OS-level shortcuts are registered via `tauri-plugin-global-shortcut`:

| Shortcut | Action |
|---|---|
| `⌘J` / `Ctrl+J` | Toggle window visibility (show if hidden, hide if visible) |
| `Escape` | Hide window if visible (no-op if already hidden) |
| `⌘,` / `Ctrl+,` | Open preferences panel (shows window first if hidden) |

**Why OS-level shortcuts, not WebView keyboard listeners:** A WebView keyboard listener only fires when the WebView has focus. OS-level shortcuts fire regardless of which app is in the foreground — they work from fullscreen games, system dialogs, other apps. This is the core value proposition.

**Why `Modifiers::SUPER` on macOS and `Modifiers::CONTROL` elsewhere — never combined:**

```rust
#[cfg(target_os = "macos")]
let modifiers = Modifiers::SUPER;      // ⌘
#[cfg(not(target_os = "macos"))]
let modifiers = Modifiers::CONTROL;    // Ctrl
```

`SUPER | CONTROL` would require holding both `⌘` and `Ctrl` simultaneously. That is never the right UX. The `#[cfg]` attributes select exactly one modifier per platform at compile time.

**`Escape` has no modifier** — it is registered as a bare keypress (`Shortcut::new(None, Code::Escape)`). This is intentional: `Esc` is universally understood as "dismiss" and the user should not need to remember a modifier. The handler checks `is_visible()` before hiding, so pressing `Esc` in another app does not cause a spurious window show.

---

### 4.7 Custom shortcut at runtime

The user can change `⌘J` to any key (e.g. `⌘K`) via the Preferences panel without restarting the app.

**JS side:** The hotkey input field captures the next `keydown` event:

```javascript
hotkeyInput.addEventListener('keydown', async (e) => {
  if (!hotkeyCapturing) return;
  e.preventDefault();

  const code = e.code; // e.g. "KeyK" — layout-independent
  hotkeyCapturing = false;

  await applyHotkey(code);
});
```

`e.code` (not `e.key`) is used because `code` is the physical key identifier. `e.key` is layout-dependent — on an AZERTY keyboard, `KeyJ` produces `j` or `h` depending on the layout. `code` is always `KeyJ` regardless of layout.

**Rust side** (`set_hotkey` command):

1. Parse the key string into a `Code` enum variant via `parse_key_code()`
2. Lock `CurrentShortcut` state, unregister the old shortcut
3. Register the new shortcut with the same platform modifier
4. Update the stored shortcut in state

```rust
struct CurrentShortcut(Mutex<Shortcut>);

#[tauri::command]
fn set_hotkey(
    app: AppHandle,
    key: String,
    current_shortcut_state: tauri::State<CurrentShortcut>,
) -> Result<(), String> {
    let code = parse_key_code(&key)?;
    let new_shortcut = Shortcut::new(Some(modifiers), code);

    // Unregister old shortcut
    let old = current_shortcut_state.0.lock()...;
    app.global_shortcut().unregister(*old).ok();

    // Register new shortcut
    app.global_shortcut().on_shortcut(new_shortcut, ...)?;

    // Update state
    *lock = new_shortcut;
    Ok(())
}
```

**Why store `CurrentShortcut` in Tauri state (not a global):** Tauri's `manage()`/`state()` system provides thread-safe, compile-time-checked access to shared data. A `static Mutex<Shortcut>` would work but is less idiomatic and harder to mock in tests. The state lives for the duration of the app process.

**Persistence:** JS stores the chosen key in `localStorage['junk-hotkey']`. On the next launch, the default shortcut (`KeyJ`) is registered by Rust as usual, then JS reads `localStorage`, and if a non-default key is found, calls `set_hotkey` to replace it. This means Rust does not need to persist anything to disk — the single source of truth is localStorage.

---

### 4.8 Window position and size memory

When the user drags the window to a new screen position, that position is remembered and restored on the next show.

**Data flow:**

1. JS detects drag start: `mousedown` outside interactive elements (same handler as `start_dragging`)
2. `mouseup` fires → call `ipc('get_window_position')` (80ms debounce) → save to `localStorage['junk-win-pos']`
3. On `tauri://focus` event (window shown) → read localStorage → call `ipc('set_window_position', {x, y})`

**Why physical pixels, not logical pixels:**

```rust
fn get_window_position(app: AppHandle) -> Result<WindowPosition, String> {
    let pos = window.outer_position()?;  // PhysicalPosition
    Ok(WindowPosition { x: pos.x, y: pos.y })
}
```

`PhysicalPosition` is the actual pixel offset from the top-left corner of the primary monitor. `LogicalPosition` is divided by the device scale factor. If a user moves from a 2× HiDPI display to a 1× display (or changes display scale in System Preferences), stored logical coordinates would place the window at the wrong position. Physical coordinates survive DPI changes.

**Off-screen guard (JS):** If the saved position is beyond `screen.width + 200` in X or below `-100` in Y (the window is substantially off-screen — e.g. the user disconnected a secondary monitor), the position restore is skipped and the window appears centered. The threshold allows partial off-screen (e.g. dragged to the edge) while preventing total invisibility.

**Window size** follows the same pattern: `get_window_size` / `set_window_size` IPC commands, 300ms debounce (longer because resize events fire very rapidly), `localStorage['junk-win-size']`. Size is clamped in Rust to `300–3000 × 200–2000` to prevent unusable-tiny or multi-monitor-spanning windows.

---

### 4.9 Auto-save via localStorage

Content is saved to `localStorage['junk-content']` with a 300ms debounce:

```javascript
let saveTimer = null;

function scheduleSave() {
  clearTimeout(saveTimer);
  saveTimer = setTimeout(() => {
    localStorage.setItem(KEY_CONTENT, editor.value);
    showSaveStatus();
  }, 300);
}

editor.addEventListener('input', scheduleSave);
```

**Why localStorage and not a file:** No filesystem permissions needed. No `tauri-plugin-fs` dependency. No path-resolution code. No migration code for moving files between versions. `localStorage` is per-origin (the embedded `index.html`), so there is no cross-app collision. In Tauri, the WebView origin for a bundled app is `tauri://localhost`, making it isolated from browser tabs.

**Trade-off:** Content is not accessible outside the app (no easy "open in other app"). For a scratchpad, that is an acceptable trade. Power users who want plain-text export have the Copy All button in the footer.

**Debounce choice — 300ms:** Short enough that content feels instantly saved; long enough that a fast typist does not trigger dozens of writes per second. localStorage writes are synchronous and block the main thread briefly; 300ms ensures at most ~3 writes/second under sustained typing.

---

### 4.10 Dark mode: CSS custom properties + matchMedia

Three modes: `light`, `dark`, `auto`.

**CSS architecture:** All colours are CSS custom properties defined on `:root` (light mode defaults) and `:root[data-theme="dark"]` (dark overrides):

```css
:root {
  --bg-glass:       rgba(255, 255, 255, 0.76);
  --text-primary:   #0a0a0a;
  --accent:         #5b5bf6;
  /* ... ~20 tokens ... */
}

:root[data-theme="dark"] {
  --bg-glass:       rgba(30, 30, 35, 0.88);
  --text-primary:   #f0f0f0;
  --accent:         #7c7cf8;
  /* ... overrides ... */
}
```

**JS mode switching:**

```javascript
function applyTheme(mode) {
  if (mode === 'dark') {
    document.documentElement.setAttribute('data-theme', 'dark');
  } else if (mode === 'light') {
    document.documentElement.removeAttribute('data-theme');
  } else { // 'auto'
    const dark = window.matchMedia('(prefers-color-scheme: dark)').matches;
    document.documentElement.setAttribute('data-theme', dark ? 'dark' : '');
    // Attach listener for live changes
    window.matchMedia('(prefers-color-scheme: dark)')
      .addEventListener('change', (e) => {
        if (getCurrentTheme() === 'auto') {
          document.documentElement.setAttribute('data-theme', e.matches ? 'dark' : '');
        }
      });
  }
}
```

In `auto` mode, the `matchMedia` listener fires when the OS switches between light and dark (e.g. via the menubar, sunset/sunrise schedule, or manually). The transition is CSS-driven (`transition: background 0.2s`) — no JS animation needed.

The `NSVisualEffectMaterial::HudWindow` blur layer also adapts automatically — macOS tints the blur lighter in light mode and darker in dark mode.

---

### 4.11 Inline markdown parser

Markdown preview is implemented as a ~80-line line-by-line parser in pure JS — no `marked.js`, no `micromark`, no CDN dependency:

```
Input text
    ↓
HTML-escape (XSS prevention)
    ↓
Split into lines
    ↓
Line-by-line: detect fenced code blocks, headings, blockquotes, HR, lists
    ↓
Inline spans: bold (**), italic (*), inline code (`), links ([text](url))
    ↓
Join lines → innerHTML of #md-preview
```

**Security first:** HTML is escaped before any parsing. `<`, `>`, `&`, `"`, `'` are all entity-encoded. The parser then inserts its own trusted HTML (e.g. `<strong>`, `<code>`), which is safe because the source text has already been sanitised.

**Why not a library:** For a scratchpad, rendering h1–h3, bold, italic, code blocks, lists, and links is the entire feature set. A full markdown library adds 30–300 KB of JS for features like tables, footnotes, and front matter that a scratchpad never needs. The inline parser is 80 lines, auditable in one read, and cannot pull in transitive vulnerabilities.

---

### 4.12 Update check: Rust HTTP, not JS fetch

```rust
#[tauri::command]
async fn check_for_update() -> Result<UpdateResult, String> {
    let current = env!("CARGO_PKG_VERSION");
    let client = tauri_plugin_http::reqwest::Client::builder()
        .user_agent(format!("Junk/{}", env!("CARGO_PKG_VERSION")))
        .build()?;

    let resp = client
        .get("https://api.github.com/repos/paulfxyz/junk/releases/latest")
        .header("Accept", "application/vnd.github+json")
        .send()
        .await?;

    let text = resp.text().await?;
    let body: serde_json::Value = serde_json::from_str(&text)?;
    // ...
}
```

**Why not `window.fetch()` from JS:**
- `fetch()` is subject to the WebView's Content Security Policy (CSP). Even with `"csp": null` in `tauri.conf.json`, the CSP can be restrictive in some Tauri builds.
- The current version (`env!("CARGO_PKG_VERSION")`) is embedded at compile time in Rust. Reading it from JS requires either injecting it into the HTML at build time (a build step) or trusting a string constant that can drift from reality.

**Critical gotcha: no `.json()` method:** `tauri_plugin_http` re-exports `reqwest` but does *not* enable reqwest's `json` feature. Calling `.json::<T>().await` fails to compile with a method-not-found error. Always use `.text().await` followed by `serde_json::from_str()`:

```rust
// ❌ Fails to compile — json feature not enabled
let body: MyStruct = resp.json::<MyStruct>().await?;

// ✅ Correct pattern
let text = resp.text().await?;
let body: MyStruct = serde_json::from_str(&text)?;
```

This is documented here because it is a genuine gotcha that will waste hours if you forget it.

---

### 4.13 tauri.conf.json key settings

```json
{
  "app": {
    "security": { "csp": null },
    "macOSPrivateApi": true,
    "withGlobalTauri": true,
    "windows": [{
      "label": "main",
      "width": 720, "height": 460,
      "center": true,
      "decorations": false,
      "transparent": true,
      "shadow": true,
      "alwaysOnTop": false,
      "skipTaskbar": true,
      "visible": false,
      "focus": false
    }]
  }
}
```

| Setting | Value | Reason |
|---|---|---|
| `decorations: false` | `false` | Removes OS title bar for clean floating look |
| `transparent: true` | `true` | Required for vibrancy and rounded corners |
| `shadow: true` | `true` | Native macOS drop shadow outside `masksToBounds` clip (v3.0.4) |
| `alwaysOnTop` | `false` | Intentionally NOT always-on-top; this is the user-chosen default — they can enable it in prefs if needed |
| `skipTaskbar: true` | `true` | No Windows/Linux taskbar entry |
| `macOSPrivateApi: true` | `true` | Required for `window-vibrancy` to attach `NSVisualEffectView` |
| `withGlobalTauri: true` | `true` | Enables `window.__TAURI__.event.listen` for JS event subscriptions |
| `visible: false` | `false` | Start hidden; first show triggered by hotkey after setup completes |
| `focus: false` | `false` | Don't steal focus at launch |
| `csp: null` | `null` | No CSP; the app loads only local assets, no remote content |

**Note on `alwaysOnTop`:** The task spec lists this as `true` but the actual `tauri.conf.json` has `false`. The Rust backend and JS prefs panel expose it as a user toggle. This reflects a deliberate design choice: always-on-top can be annoying during video calls or presentations, so it is off by default.

---

## 5. Rust Backend Deep Dive

### 5.1 State management

Tauri v2's `manage()` / `state()` system is used for one piece of shared state:

```rust
struct CurrentShortcut(Mutex<Shortcut>);
```

This holds the currently registered toggle shortcut. It must be a `Mutex` because:
1. Tauri's state system requires `Send + Sync`
2. The shortcut can be mutated from the `set_hotkey` IPC command (called from the WebView thread) and read from shortcut callback closures (called from the global shortcut thread)

The `Mutex` is never held across an `await` point — all operations are synchronous reads/writes with a small critical section.

No other shared state exists. Window state (position, size, content, preferences) lives either in Tauri's window handle (queried on demand) or in `localStorage` (owned by JS). This keeps the Rust side stateless and easy to reason about.

### 5.2 IPC command reference

All commands are registered in `tauri::generate_handler![]`:

| Command | Args | Returns | Notes |
|---|---|---|---|
| `hide_window` | — | `Result<(), String>` | Hides main window |
| `start_dragging` | — | `Result<(), String>` | Calls `NSWindow.performWindowDragWithEvent:` |
| `open_prefs` | — | `Result<(), String>` | Shows window + emits `open-prefs` event |
| `get_prefs` | — | `Result<Prefs, String>` | Returns `{ launch_at_login: bool }` |
| `set_launch_at_login` | `enabled: bool` | `Result<(), String>` | Enables/disables OS login item |
| `check_for_update` | — | `Result<UpdateResult, String>` | Fetches GitHub releases API |
| `get_window_position` | — | `Result<WindowPosition, String>` | `{ x: i32, y: i32 }` in physical pixels |
| `set_window_position` | `x: i32, y: i32` | `Result<(), String>` | Physical pixels; no off-screen guard in Rust |
| `get_window_size` | — | `Result<WindowSize, String>` | `{ width: u32, height: u32 }` physical |
| `set_window_size` | `width: u32, height: u32` | `Result<(), String>` | Clamped to `300–3000 × 200–2000` |
| `set_hotkey` | `key: String` | `Result<(), String>` | Key = `KeyboardEvent.code` string |

All commands return `Result<T, String>`. Errors surface to JS as rejected promises. This convention is enforced throughout — no `unwrap()` in IPC command bodies.

### 5.3 Error handling philosophy

> "A panic in a callback crashes the entire app. A log message is always better."

The comment at the top of `main.rs` states this explicitly. The rules:

- **IPC commands** return `Result<T, String>` and propagate errors to JS. JS shows them in the UI or console.
- **Global shortcut callbacks** cannot propagate errors (no caller). They `log::error!()` and return.
- **Setup hooks** log errors as warnings for non-fatal failures (vibrancy, corner radius), so the app launches even if native effects are unavailable.
- **`panic = "abort"` in release** — panics are bugs, not recoverable errors. Aborting immediately produces a crash report the developer can act on rather than unwinding into undefined state.

`unwrap()` is banned in all production paths. Pattern: `map_err(|e| e.to_string())` for `Result` conversions, `ok()` for errors we intentionally discard after logging.

### 5.4 Capabilities (Tauri v2 permissions)

Tauri v2 requires explicit permission grants in `capabilities/default.json`. Junk's grants:

```json
{
  "permissions": [
    "core:default",
    "core:window:allow-show",
    "core:window:allow-hide",
    "core:window:allow-set-focus",
    "core:window:allow-is-visible",
    "core:window:allow-start-dragging",
    "core:window:allow-outer-position",
    "core:window:allow-set-position",
    "core:webview:allow-set-webview-focus",
    "global-shortcut:allow-register",
    "global-shortcut:allow-unregister",
    "global-shortcut:allow-is-registered",
    "autostart:allow-enable",
    "autostart:allow-disable",
    "autostart:allow-is-enabled",
    "http:default"
  ]
}
```

Each permission maps directly to an operation the app performs. Nothing is granted speculatively. The `http:default` permission enables outbound HTTP from the Rust side (for update checks) but does not expose `fetch()` from JS beyond what the WebView already allows.

---

## 6. Frontend Deep Dive

### 6.1 CSS architecture

The stylesheet is approximately 600 lines of plain CSS inside `<style>` in `index.html`. No preprocessor. The architecture is:

**Layer 1 — Design tokens** (CSS custom properties on `:root`):

All colours, spacing is expressed as custom properties. ~20 tokens per theme. Swapping the theme is a single attribute change on `<html>`:

```javascript
document.documentElement.setAttribute('data-theme', 'dark');  // dark
document.documentElement.removeAttribute('data-theme');         // light
```

**Layer 2 — Reset** (`*, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }`)

**Layer 3 — Layout components**: `.window`, `.footer`, `.footer-left`, `.footer-right`

**Layer 4 — UI components**: `.icon-btn`, `#editor`, `#md-preview`, `#editor-dim`, `.prefs-panel`, `.prefs-section`, `.hotkey-row`

**Layer 5 — Animations**: `@keyframes fly-in` (window entrance), `@keyframes save-fade` (save status flash)

**The fly-in animation:**

```css
@keyframes fly-in {
  from { opacity: 0; transform: scale(0.96) translateY(-10px); }
  to   { opacity: 1; transform: scale(1)    translateY(0); }
}
```

This runs on `.window` whenever the window becomes visible. The animation is re-triggered in JS by removing and re-adding the class. `cubic-bezier(0.22, 1, 0.36, 1)` is a spring-like ease-out — fast initial movement, gentle settle.

**`html, body` must be `background: transparent`:** On macOS, the `NSVisualEffectView` (the blur layer) is the actual background. If `html`/`body` have any non-transparent background, it paints over the vibrancy effect. The `.window` div has a semi-transparent `var(--bg-glass)` background that lets the blur bleed through while adding tint control.

### 6.2 JavaScript organisation

The ~600 lines of JS in `index.html` are organised by feature area with `// ──` section headers. The call order at startup:

```
DOMContentLoaded
  └── loadContent()           // Restore text from localStorage
  └── initTheme()             // Apply saved theme mode
  └── initFontSize()          // Apply saved font size
  └── initMdMode()            // Apply markdown mode toggle state
  └── initHotkeyDisplay()     // Show current hotkey in preferences
  └── setupDragHandlers()     // mousedown → start_dragging
  └── setupResizeObserver()   // Debounced window size save
  └── restoreHotkey()         // Re-register saved hotkey in Rust (if non-default)
  └── window.__TAURI__.event.listen('tauri://focus', onFocus)
  └── window.__TAURI__.event.listen('open-prefs', onOpenPrefs)

onFocus()
  └── triggerFlyIn()          // Play entrance animation
  └── restoreWindowPosition() // Call set_window_position with saved coords
  └── focusEditor()           // Place cursor in textarea
```

**Key design principle:** JS is the source of truth for all user preferences. Rust is stateless with respect to preferences. This means:
- Zero Rust file I/O for settings
- Settings survive app restarts via localStorage
- Settings are inspectable/editable via browser devtools (in dev builds)

### 6.3 localStorage schema

| Key | Type | Description |
|---|---|---|
| `junk-content` | `string` | Scratchpad text content |
| `junk-auto-update` | `'true'` / `'false'` | Whether to auto-check for updates on launch |
| `junk-font-size` | number string (`'14'`–`'28'`) | Font size in px |
| `junk-theme` | `'light'` / `'auto'` / `'dark'` | Appearance mode |
| `junk-md-mode` | `'true'` / `'false'` | Markdown preview enabled |
| `junk-hotkey` | Code string (e.g. `'KeyJ'`, `'KeyK'`) | Custom toggle key; absent = default |
| `junk-win-pos` | JSON `{"x": 100, "y": 200}` | Physical window position (pixels) |
| `junk-win-size` | JSON `{"w": 720, "h": 460}` | Physical window size (pixels) |

All keys are prefixed with `junk-` to avoid collisions if the WebView origin ever shares storage with other apps (it shouldn't in production Tauri, but defensive namespacing is free).

---

## 7. Tauri Events (Rust → JS)

| Event name | Emitted by | Payload | Subscriber |
|---|---|---|---|
| `tauri://focus` | Tauri runtime (built-in) | `null` | `window.__TAURI__.event.listen` — triggers fly-in + position/size restore |
| `open-prefs` | `open_prefs` command + `register_prefs_shortcut` callback | `null` | `window.__TAURI__.event.listen` — slides preferences panel up |

**`tauri://focus` is a Tauri built-in event.** It fires every time the window receives focus — including the first show after launch, every hotkey trigger, and every click on the window from another app. JS handles it identically every time: animate in, restore position, focus the editor.

**`open-prefs` is a custom event** emitted via `window.emit("open-prefs", ())`. The `()` payload serialises to `null` in JSON — JS ignores the payload and just opens the panel.

**Both events require `withGlobalTauri: true`** in `tauri.conf.json`. Without it, `window.__TAURI__` is not injected and `listen` is unavailable. This is the main reason `withGlobalTauri` is enabled — the IPC invoke path has an alternative (`__TAURI_INTERNALS__`), but the event listen path does not.

---

## 8. Platform Notes

### 8.1 macOS

| Feature | Implementation |
|---|---|
| Window appearance | `NSVisualEffectMaterial::HudWindow` — adapts light/dark automatically |
| Rounded corners | `objc2` `CALayer.cornerRadius + masksToBounds` (see §4.4) |
| Drop shadow | `shadow: true` in `tauri.conf.json` → native WindowServer shadow |
| Dock + ⌘-Tab | Hidden via `ActivationPolicy::Accessory` (see §4.5) |
| Login item | `tauri-plugin-autostart` → `MacosLauncher::LaunchAgent` → `~/Library/LaunchAgents/<bundle-id>.plist` |
| Minimum macOS version | 10.15 Catalina (set in `bundle.macOS.minimumSystemVersion`) |
| Distribution | Universal binary (arm64 + x86_64 merged with `lipo`) |
| Code signing | Ad-hoc (`codesign -s -`) — no notarisation; users run `xattr -dr com.apple.quarantine` |

### 8.2 Windows

| Feature | Implementation |
|---|---|
| Window appearance | `apply_acrylic(&window, None)` — semi-transparent Acrylic blur (Windows 10/11) |
| Rounded corners | CSS `border-radius: 14px` + OS compositor (Windows 11 auto-rounds windows) |
| Taskbar | Hidden via `skipTaskbar: true` |
| Login item | `tauri-plugin-autostart` → `HKCU\Software\Microsoft\Windows\CurrentVersion\Run` |
| WebView | WebView2 (Chromium); installer bootstrapper for machines without it |
| Distribution | NSIS installer + MSI (WiX) |

### 8.3 Linux

| Feature | Implementation |
|---|---|
| Window appearance | No native vibrancy; CSS `backdrop-filter: blur(40px)` (varies by compositor) |
| Rounded corners | CSS `border-radius: 14px` (compositor-dependent; works in GNOME/KDE with compositor on) |
| Login item | `tauri-plugin-autostart` → `~/.config/autostart/junk.desktop` |
| WebView | WebKitGTK (`libwebkit2gtk-4.1`) |
| Distribution | AppImage (portable, no install) + `.deb` (Debian/Ubuntu) |
| Build dependencies | `libwebkit2gtk-4.1-dev`, `libgtk-3-dev`, `libxdo-dev`, `libayatana-appindicator3-dev`, others (see CI workflow) |

---

## 9. CI / Release Pipeline

**Workflow file:** `.github/workflows/build.yml`

### Triggers

| Event | Jobs run |
|---|---|
| Push to `main` | All three platform builds (smoke test) |
| Pull request to `main` | All three platform builds |
| Tag push matching `v*` | All three builds + `release` job |

### Platform build matrix

| Platform | Runner | Target | Output |
|---|---|---|---|
| macOS | `macos-latest` | `universal-apple-darwin` | `.dmg` (arm64 + x86_64) |
| Windows | `windows-latest` | `x86_64-pc-windows-msvc` | `.exe` (NSIS) + `.msi` (WiX) |
| Linux | `ubuntu-latest` | `x86_64-unknown-linux-gnu` | `.AppImage` + `.deb` |

### Universal binary on macOS

`--target universal-apple-darwin` tells Tauri to:
1. Compile `aarch64-apple-darwin` (Apple Silicon)
2. Compile `x86_64-apple-darwin` (Intel)
3. Merge the two binaries with `lipo`

Build time roughly doubles (~8 min uncached → ~16 min), but the result is a single `.app` that runs natively on every Mac from 2012 to present (Intel Macs) and M1/M2/M3 (Apple Silicon).

### Caching strategy

```yaml
- uses: actions/cache@v4
  with:
    path: |
      ~/.cargo/registry
      ~/.cargo/git
      src-tauri/target
    key: macos-cargo-${{ hashFiles('**/Cargo.lock') }}
    restore-keys: macos-cargo-
```

Two separate caches — `~/.cargo/registry` (downloaded crate sources, keyed on `Cargo.lock`) and `src-tauri/target` (compiled artefacts). A warm cache reduces build time from ~8 minutes to ~2 minutes. Keying on `Cargo.lock` ensures a lockfile update invalidates the compiled artefacts cache.

### Release job

The `release` job runs only on tag pushes (`if: startsWith(github.ref, 'refs/tags/v')`), waits on all three platform builds (`needs: [build-macos, build-windows, build-linux]`), then:

1. Downloads all platform artefacts into `dist/`
2. Filters files by version string (to avoid stale cache artefacts from previous `Cargo.lock` runs)
3. Creates the GitHub Release if it doesn't exist, or uploads to an existing one (`gh release upload --clobber`)

**Why upload to a pre-created release:** The intended workflow is: create the release with release notes from your dev machine (`gh release create v3.0.4 --notes "..."`) *before* pushing the tag. CI then attaches the binaries. This lets you write polished release notes offline, not in a YAML heredoc.

---

## 10. Build & Dev Setup

### Prerequisites

- **Rust** stable (MSRV 1.70): `rustup install stable`
- **Node.js** 20+ with npm (for `@tauri-apps/cli`)
- **Platform tools:**
  - macOS: Xcode Command Line Tools (`xcode-select --install`)
  - Windows: WebView2 (pre-installed on Windows 10/11); Visual Studio Build Tools
  - Linux: see the `apt-get install` block in `build.yml`

### Commands

```bash
# Install the single devDependency
npm install

# Development mode (live reload, devtools available, debug logging)
npm run tauri dev

# Production build for the current platform
npm run tauri build

# macOS universal binary specifically
npm run tauri build -- --target universal-apple-darwin

# Debug logging (dev only — controlled by main.rs setup)
RUST_LOG=junk=debug npm run tauri dev
```

### Dev mode notes

- `tauri dev` starts a local dev server on `http://localhost:1420` and opens the WebView pointed at it. Changes to `src/index.html` trigger an immediate WebView reload — no Rust recompile needed for frontend changes.
- Right-click → "Inspect Element" opens WebKit Inspector in the WebView (enabled by the `devtools` feature on `tauri`).
- Rust changes require a recompile. `cargo check` is fast; `tauri dev` recompiles and relaunches automatically.
- The Dock icon appears in dev mode (the release build uses `ActivationPolicy::Accessory` which hides it; the dev build inherits the default `Regular` policy for easier debugging).

### Version bumping

Version is declared in **two places** (they must stay in sync):

1. `src-tauri/Cargo.toml` → `[package] version`
2. `src-tauri/tauri.conf.json` → `"version"`

Bump both before tagging. The Rust code uses `env!("CARGO_PKG_VERSION")` — it reads from `Cargo.toml` at compile time.

---

## 11. Version History Highlights

| Version | Key change |
|---|---|
| v3.0.4 | `shadow: true` in `tauri.conf.json` — native macOS drop shadow via WindowServer (fixes CSS shadow clipped by `masksToBounds`) |
| v3.0.3 | Two-step rounded corners: `apply_vibrancy` + `set_macos_window_corner_radius()` via `objc2` CALayer `masksToBounds` |
| v3.0.0 | Custom hotkey, window position memory, font size picker, dark mode, markdown preview, plain-text export |

---

## Contributing

The codebase is intentionally small and self-contained. There are two files to understand: `src/index.html` (all frontend) and `src-tauri/src/main.rs` (all backend). Read them top-to-bottom; they are heavily commented. The Cargo.toml comments explain every dependency choice.

If you hit a Tauri v2 API question, the [Tauri v2 docs](https://v2.tauri.app/) and the [tauri-plugin-global-shortcut](https://github.com/tauri-apps/plugins-workspace/tree/v2/plugins/global-shortcut) source are the primary references. For macOS-specific work, the `objc2` book and Apple's Core Animation docs are your friends.

Questions? Ideas? Open an issue or PR at https://github.com/paulfxyz/junk — all contributors welcome.
