# Contributing to Junk

Thanks for your interest in contributing. Junk is a small, focused tool — contributions should keep it that way. Read this guide before opening a PR.

---

## Prerequisites

| Requirement | Version | Install |
|---|---|---|
| Rust (stable) | 1.70+ | `curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs \| sh` |
| Tauri CLI | v2 | `cargo install tauri-cli --version "^2"` |

**macOS**
```bash
xcode-select --install
```

**Windows**

Install [Microsoft C++ Build Tools](https://visualstudio.microsoft.com/visual-cpp-build-tools/) — select the **Desktop development with C++** workload (MSVC toolchain).

**Linux (Debian/Ubuntu)**
```bash
sudo apt-get install \
  libwebkit2gtk-4.1-dev \
  libssl-dev \
  libayatana-appindicator3-dev \
  librsvg2-dev
```

**No Node.js required.** The frontend is a single HTML file. There is no npm install step, no bundler, no build pipeline.

---

## Dev Setup

```bash
git clone https://github.com/paulfxyz/junk
cd junk
cargo tauri dev   # launches the app with hot-reload
```

`cargo tauri dev` compiles the Rust backend and opens the app window. From there:

- **Frontend changes** (`src/index.html`) — save the file and the WebView reloads instantly.
- **Rust changes** (`src-tauri/src/main.rs`) — triggers a full recompile. Expect 5–30s depending on your machine.

---

## Building a Release

```bash
cargo tauri build
```

Output lands in `src-tauri/target/release/bundle/`:

| Platform | Output |
|---|---|
| macOS | `.dmg` (arm64 and x64 built separately, merged by CI with `lipo`) |
| Windows | `.exe` installer + `.exe` portable |
| Linux | `.AppImage` + `.deb` |

For universal macOS binaries: CI builds arm64 and x64 separately and merges them with `lipo`. You don't need to do this locally — just build for your native arch during development.

---

## Project Structure

```
src/
  index.html          # Entire frontend (HTML + CSS + JS, ~1500 lines)
src-tauri/
  src/
    main.rs           # All backend Rust (~770 lines)
  Cargo.toml          # Version + dependencies
  tauri.conf.json     # Window config + capabilities
  capabilities/       # Tauri v2 permission declarations
  icons/              # All icon sizes (all platforms)
.github/workflows/
  release.yml         # CI: build + release on tag push
```

The project is intentionally monolithic. Both the frontend and backend are single files. This is a feature, not a flaw — it keeps the codebase easy to navigate and contribute to.

---

## Adding a New IPC Command

IPC is how the JS frontend calls Rust backend functions. Here's the full workflow:

1. **Define a return type** in `main.rs` (if the command returns structured data):
   ```rust
   #[derive(serde::Serialize)]
   struct MyResult {
       value: String,
   }
   ```

2. **Write the command function**:
   ```rust
   #[tauri::command]
   fn my_command(arg: String) -> Result<MyResult, String> {
       // ...
   }
   ```

3. **Register it** in `tauri::generate_handler![]`:
   ```rust
   tauri::generate_handler![
       // existing commands...
       my_command,
   ]
   ```

4. **Call it from JS** using the `ipc()` helper:
   ```js
   const result = await ipc('my_command', { arg: 'value' });
   ```

5. **Document it** — add a row to the IPC Command Reference table in `ARCHITECTURE.md`.

---

## Code Style

### Rust

- Run `cargo fmt` before every commit.
- Run `cargo clippy --all -- -D warnings` — zero warnings allowed.
- No `unwrap()` in production code paths. Use `?` for propagation, or `.map_err()` + `log::error!()` for recoverable errors that shouldn't crash.

### JavaScript

- Vanilla ES2020. No framework. No bundler. No dependencies.
- Keep it that way. If you find yourself reaching for a library, reconsider the approach.
- Every new feature gets a `// FEATURE N:` block comment in the JS section of `index.html`, where `N` is the next sequential number. This makes the file grep-navigable.

### Comments

Explain **why**, not what. The code says what. The comment says why.

```js
// BAD: Increment the counter
counter++;

// GOOD: Counter tracks unsaved changes; triggers the dirty indicator in the footer
counter++;
```

---

## Committing

Before you commit, run:

```bash
cargo test                          # no tests yet, but confirms clean compile
cargo clippy --all -- -D warnings   # zero warnings
```

Commit message prefixes:

| Prefix | When to use |
|---|---|
| `feat:` | New user-facing feature |
| `fix:` | Bug fix |
| `docs:` | Documentation only |
| `chore:` | Tooling, deps, version bumps, CI |
| `refactor:` | Code change with no behavior change |

Keep commits focused. One logical change per commit. If you're fixing a bug and cleaning up unrelated code, that's two commits.

---

## Releasing

**Releases are made by the repo owner (@paulfxyz) only.** If you think a release is warranted, open an issue.

The release process:

1. Bump the version in **both** places:
   - `src-tauri/Cargo.toml` → `[package] version = "X.Y.Z"`
   - `src-tauri/tauri.conf.json` → `"version": "X.Y.Z"`
2. Update `CHANGELOG.md` — add the new version section at the top.
3. Commit: `chore: bump version to vX.Y.Z`
4. Tag:
   ```bash
   git tag -a vX.Y.Z -m "vX.Y.Z — short description"
   ```
5. Push:
   ```bash
   git push origin main && git push origin vX.Y.Z
   ```
6. CI picks up the tag, builds all platforms, and publishes the GitHub Release automatically.

---

## Key Gotchas

Read this section before writing any code. These are hard-won lessons — ignoring them will cost you time.

### IPC Bridge

Always use the `getInvoke()` helper to call Tauri commands. It tries `__TAURI_INTERNALS__.invoke` first, then falls back to `__TAURI__.core.invoke`. Never call `window.__TAURI__.invoke` directly — it doesn't exist in Tauri v2.

```js
// CORRECT
const invoke = getInvoke();
const result = await invoke('my_command', { arg: 'value' });

// WRONG — will throw in Tauri v2
const result = await window.__TAURI__.invoke('my_command');
```

### JSON in reqwest

`tauri_plugin_http`'s bundled reqwest does **not** enable the `json` feature. Calling `.json::<T>().await` will always fail silently or panic. Use `.text().await` and then `serde_json::from_str()`:

```rust
// CORRECT
let body = resp.text().await.map_err(|e| e.to_string())?;
let data: MyStruct = serde_json::from_str(&body).map_err(|e| e.to_string())?;

// WRONG — json feature not enabled
let data = resp.json::<MyStruct>().await?;
```

### macOS Window Drag

When calling `start_dragging()` via IPC, you **must** call `e.preventDefault()` before the IPC call, not after. The OS needs drag state to still be open when the Rust side processes the event.

```js
// CORRECT
header.addEventListener('mousedown', async (e) => {
    e.preventDefault();              // must be first
    await ipc('start_dragging');
});
```

### Global Shortcuts

Use `Modifiers::SUPER` on macOS and `Modifiers::CONTROL` on Windows/Linux. Never combine them — the shortcut will fail to register on the platform where the modifier doesn't apply.

### macOS Rounded Corners and Shadows

`CALayer`'s `masksToBounds = YES` (set for rounded corners) clips CSS `box-shadow` — the shadow renders inside the window bounds and is invisible. Use `"shadow": true` in `tauri.conf.json` to get a native WindowServer drop shadow instead. CSS `box-shadow` is fine for decorative edge rings (0.5px), but not for actual shadows on transparent frameless windows.

### withGlobalTauri

`withGlobalTauri: true` in `tauri.conf.json` is required for `window.__TAURI__.event.listen` to work. Tauri events (like `tauri://focus`) will silently never fire without it. Don't remove it.

---

## Reporting Bugs

Open an issue at [github.com/paulfxyz/junk/issues](https://github.com/paulfxyz/junk/issues).

Include:
- OS + version (e.g. macOS 15.3, Windows 11 23H2)
- Junk version (visible in Preferences)
- Steps to reproduce
- If possible, output from `RUST_LOG=debug cargo tauri dev`

The debug log almost always contains the root cause.
