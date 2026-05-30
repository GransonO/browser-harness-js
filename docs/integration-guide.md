# Integration Guide: browser-harness-js in Agentic Workflows and End-User Apps

## Overview

`browser-harness-js` exposes every CDP method as a typed JS call over a simple local HTTP API (`POST /eval` on `127.0.0.1:9876`). That single surface is everything you need to wire it into any agentic system or package it for end users.

---

## Part 1 — Agentic Workflow Integration

### How the pieces fit together

```
┌─────────────────────────────────────┐
│  LLM / Agent Orchestrator           │
│  (Claude, GPT, custom loop, etc.)   │
└────────────────┬────────────────────┘
                 │  tool call: run_browser_js(code)
                 ▼
┌─────────────────────────────────────┐
│  browser-harness-js CLI / HTTP API  │
│  POST http://127.0.0.1:9876/eval    │
│  (auto-started Bun process)         │
└────────────────┬────────────────────┘
                 │  CDP over WebSocket
                 ▼
┌─────────────────────────────────────┐
│  Chrome (remote-debugging port)     │
│  Real tabs, real cookies, real DOM  │
└─────────────────────────────────────┘
```

The Bun server is the only persistent process. The agent sends raw JS snippets to it; the session, active tab, and any `globalThis.*` survive between calls.

---

### 1.1 The simplest integration: shell tool

If your agent can run shell commands, no extra integration is needed. Expose `browser-harness-js` as a tool:

```python
import subprocess

def run_browser_js(code: str) -> str:
    result = subprocess.run(
        ["browser-harness-js", code],
        capture_output=True, text=True
    )
    if result.returncode != 0:
        raise RuntimeError(result.stderr.strip())
    return result.stdout.strip()
```

For multi-line snippets use stdin:

```python
def run_browser_js_multiline(code: str) -> str:
    result = subprocess.run(
        ["browser-harness-js"],
        input=code, capture_output=True, text=True
    )
    if result.returncode != 0:
        raise RuntimeError(result.stderr.strip())
    return result.stdout.strip()
```

---

### 1.2 Direct HTTP integration (no CLI dependency)

The Bun server's HTTP API is intentionally minimal. Once the server is running, any HTTP client works — no shell required.

**Start the server once at agent startup:**

```bash
browser-harness-js --start
```

Or start it programmatically:

```python
import subprocess, time, urllib.request

def ensure_server():
    try:
        urllib.request.urlopen("http://127.0.0.1:9876/health", timeout=1)
        return  # already up
    except Exception:
        pass
    subprocess.Popen(
        ["bun", "/path/to/sdk/repl.ts"],
        stdout=open("/tmp/browser-harness-js.log", "w"),
        stderr=subprocess.STDOUT
    )
    for _ in range(50):
        time.sleep(0.1)
        try:
            urllib.request.urlopen("http://127.0.0.1:9876/health", timeout=1)
            return
        except Exception:
            pass
    raise RuntimeError("CDP server failed to start")
```

**Eval endpoint:**

```
POST http://127.0.0.1:9876/eval
Content-Type: text/plain

<raw JS, top-level await supported>
```

- `200 OK` → result (string, JSON, or empty)
- `500` → error message + JS stack on stderr

**Health endpoint:**

```
GET http://127.0.0.1:9876/health
→ {"ok":true,"uptime":42,"connected":true,"sessionId":"..."}
```

**Node.js example:**

```ts
async function evalCDP(code: string): Promise<string> {
  const res = await fetch("http://127.0.0.1:9876/eval", {
    method: "POST",
    body: code,
  });
  const text = await res.text();
  if (!res.ok) throw new Error(text);
  return text;
}
```

---

### 1.3 Defining it as an LLM tool

Give the model a single tool that runs a JS snippet against the live browser session. The description is the key — it should tell the model exactly what globals and methods are available.

**Anthropic SDK (Claude):**

```python
browser_tool = {
    "name": "browser_js",
    "description": (
        "Execute JavaScript against a persistent Chrome CDP session. "
        "Globals: `session` (652 typed CDP methods across 56 domains), "
        "`listPageTargets()`, `detectBrowsers()`. "
        "Top-level await supported. Single expressions auto-return. "
        "Multi-statement snippets need an explicit `return`. "
        "State (active tab, globalThis.*) persists across calls. "
        "On error: stderr + exit 1."
    ),
    "input_schema": {
        "type": "object",
        "properties": {
            "code": {
                "type": "string",
                "description": "JS snippet to evaluate. Use `await session.<Domain>.<method>(params)` pattern."
            }
        },
        "required": ["code"]
    }
}
```

**Tool handler:**

```python
def handle_tool(tool_name: str, tool_input: dict) -> str:
    if tool_name == "browser_js":
        return run_browser_js_multiline(tool_input["code"])
```

---

### 1.4 Standard startup sequence

Every agentic session should begin with these three calls before doing any page-level work:

```js
// 1. Connect to the running browser (auto-detects Chrome/Edge/Brave/etc.)
await session.connect()

// 2. List real page targets (chrome:// internals filtered out)
const tabs = await listPageTargets()

// 3. Route all page-level calls to the first tab
await session.use(tabs[0].targetId)
```

If no tabs exist yet:

```js
await session.connect()
let tabs = await listPageTargets()
if (!tabs.length) {
  const { targetId } = await session.Target.createTarget({ url: 'about:blank' })
  await session.use(targetId)
} else {
  await session.use(tabs[0].targetId)
}
```

---

### 1.5 Persisting state across agent turns

Each snippet runs inside a fresh `async` wrapper — `let`/`const` variables vanish when it returns. To carry data between agent turns, attach to `globalThis`:

```js
// Turn 1: store the tab ID
globalThis.tid = (await listPageTargets())[0].targetId
await session.use(globalThis.tid)

// Turn 2: it's still there
await session.Page.navigate({ url: 'https://example.com' })
```

The session WebSocket, active sessionId, and event subscribers are preserved by the server automatically — `globalThis` is only needed for ad-hoc agent data.

---

### 1.6 Event-driven patterns

For actions that trigger async page changes (navigation, form submission, file download), subscribe to events rather than polling:

```js
// Wait for navigation to complete before reading the page
await session.Page.enable()
const nav = session.waitFor('Page.frameNavigated', p => !p.frame.parentId, 10_000)
await session.Page.navigate({ url: 'https://example.com' })
await nav
const { result } = await session.Runtime.evaluate({
  expression: 'document.title',
  returnByValue: true
})
return result.value
```

```js
// Wait for a network response matching a pattern
await session.Network.enable({})
const resp = session.waitFor(
  'Network.responseReceived',
  p => p.response.url.includes('/api/data'),
  15_000
)
await session.Runtime.evaluate({ expression: 'document.querySelector("button").click()' })
const { response } = await resp
return response.url
```

---

### 1.7 Multi-tab orchestration

```js
// Store tab IDs across calls
const tabs = await listPageTargets()
tabs.forEach((t, i) => { globalThis['tab' + i] = t.targetId })

// Open a new tab in a later call
const { targetId } = await session.Target.createTarget({ url: 'https://example.com' })
globalThis.newTab = targetId

// Switch between tabs
await session.use(globalThis.tab0)
const title0 = (await session.Runtime.evaluate({ expression: 'document.title', returnByValue: true })).result.value

await session.use(globalThis.newTab)
await session.Page.navigate({ url: 'https://other.com' })
```

---

### 1.8 Error handling in agent loops

The server returns HTTP 500 with the CDP error and JS stack on failure. A robust agent loop should:

1. Catch the error text and pass it back to the LLM as tool result content (not as an exception that crashes the loop).
2. Let the model decide whether to retry, adjust params, or give up.

```python
def run_browser_js(code: str) -> dict:
    try:
        result = subprocess.run(["browser-harness-js", code],
                                capture_output=True, text=True)
        if result.returncode == 0:
            return {"ok": True, "result": result.stdout.strip()}
        else:
            return {"ok": False, "error": result.stderr.strip()}
    except FileNotFoundError:
        return {"ok": False, "error": "browser-harness-js not found on PATH"}
```

Return the `{"ok": false, "error": "..."}` object as the tool result content so the model sees the CDP error and can adapt.

---

## Part 2 — Packaging for Desktop Apps

### 2.1 Architecture for desktop packaging

The Bun server is the core. A desktop app wraps it:

```
┌──────────────────────────────────────────────┐
│  Desktop App Shell (Electron / Tauri)        │
│  ┌──────────────────┐  ┌──────────────────┐  │
│  │  UI (React/Vue)  │  │  Bun subprocess  │  │
│  │  - Status tray   │  │  repl.ts server  │  │
│  │  - Settings      │  │  port 9876       │  │
│  └────────┬─────────┘  └────────┬─────────┘  │
│           │ IPC / fetch          │ CDP WS     │
└───────────┼──────────────────────┼────────────┘
            │                      ▼
            │              Chrome (user's browser)
            ▼
        LLM API (cloud or local)
```

The app manages:
- Bundling and starting the Bun server
- Connecting to Chrome (prompting user to enable remote debugging if needed)
- Providing a UI for the agent conversation
- Calling the LLM API and routing `browser_js` tool calls to the local server

---

### 2.2 Electron

Electron is the most portable choice (macOS + Windows + Linux from one codebase). Bun doesn't need to be installed globally — bundle the binary.

**Project structure:**

```
my-app/
├── package.json
├── main.js          ← Electron main process
├── preload.js       ← Contextbridge for renderer ↔ main IPC
├── renderer/        ← React/Vue UI
└── resources/
    ├── bun           ← bundled Bun binary (platform-specific)
    └── sdk/
        ├── repl.ts
        ├── session.ts
        └── generated.ts
```

**main.js — start/stop the Bun server as a child process:**

```js
const { app, BrowserWindow, ipcMain } = require('electron')
const { spawn } = require('child_process')
const path = require('path')
const fetch = require('node-fetch')

let replProcess = null

function getBunPath() {
  // Use bundled Bun binary at runtime
  return path.join(process.resourcesPath, 'bun')
}

async function startReplServer() {
  if (replProcess) return
  const bun = getBunPath()
  const repl = path.join(process.resourcesPath, 'sdk', 'repl.ts')
  replProcess = spawn(bun, [repl], {
    env: { ...process.env, CDP_REPL_PORT: '9876' },
    stdio: ['ignore', 'pipe', 'pipe'],
  })
  replProcess.stdout.on('data', d => console.log('[repl]', d.toString()))
  replProcess.stderr.on('data', d => console.error('[repl]', d.toString()))

  // Wait for health check
  for (let i = 0; i < 50; i++) {
    await new Promise(r => setTimeout(r, 100))
    try {
      await fetch('http://127.0.0.1:9876/health')
      console.log('CDP REPL ready')
      return
    } catch {}
  }
  throw new Error('CDP REPL failed to start')
}

function stopReplServer() {
  if (replProcess) { replProcess.kill(); replProcess = null }
}

app.on('ready', async () => {
  await startReplServer()
  const win = new BrowserWindow({ width: 900, height: 700, webPreferences: { preload: path.join(__dirname, 'preload.js') } })
  win.loadFile('renderer/index.html')
})

app.on('will-quit', stopReplServer)

// IPC: renderer calls this to eval CDP JS
ipcMain.handle('browser-eval', async (_event, code) => {
  const res = await fetch('http://127.0.0.1:9876/eval', { method: 'POST', body: code })
  const text = await res.text()
  return { ok: res.ok, result: text }
})
```

**Bundling platform-specific Bun binaries:**

Use `electron-builder` with platform targets. Download the correct Bun binary for each platform during the build:

```json
// package.json (electron-builder config)
{
  "build": {
    "extraResources": [
      { "from": "resources/", "to": "resources/" }
    ],
    "mac": { "target": "dmg" },
    "win": { "target": "nsis" },
    "linux": { "target": "AppImage" }
  }
}
```

Download Bun binaries in a pre-build script:

```bash
# scripts/download-bun.sh
BUN_VERSION="1.1.0"
mkdir -p resources

# macOS arm64
curl -Lo resources/bun-macos-arm64.zip \
  "https://github.com/oven-sh/bun/releases/download/bun-v${BUN_VERSION}/bun-darwin-aarch64.zip"

# Linux x64
curl -Lo resources/bun-linux-x64.zip \
  "https://github.com/oven-sh/bun/releases/download/bun-v${BUN_VERSION}/bun-linux-x64.zip"
```

Select the right binary at runtime:

```js
function getBunPath() {
  const platform = process.platform  // 'darwin', 'win32', 'linux'
  const arch = process.arch          // 'x64', 'arm64'
  return path.join(process.resourcesPath, `bun-${platform}-${arch}`)
}
```

---

### 2.3 Tauri (Rust + WebView)

Tauri produces smaller binaries (~5 MB vs Electron's ~150 MB) but requires Rust tooling at build time. Bun is managed the same way — bundled as a sidecar binary.

**src-tauri/tauri.conf.json — declare Bun as a sidecar:**

```json
{
  "tauri": {
    "bundle": {
      "externalBin": ["binaries/bun"]
    }
  }
}
```

**src-tauri/src/main.rs — spawn Bun at startup:**

```rust
use std::process::{Child, Command};
use tauri::Manager;

struct ReplHandle(Mutex<Option<Child>>);

fn start_repl(app: &tauri::AppHandle) -> Result<Child, String> {
    let bun = app.path_resolver()
        .resolve_resource("binaries/bun")
        .ok_or("bun not found")?;
    let repl = app.path_resolver()
        .resolve_resource("sdk/repl.ts")
        .ok_or("repl.ts not found")?;

    Command::new(bun)
        .arg(repl)
        .env("CDP_REPL_PORT", "9876")
        .spawn()
        .map_err(|e| e.to_string())
}

#[tauri::command]
async fn browser_eval(code: String) -> Result<String, String> {
    let client = reqwest::Client::new();
    let resp = client
        .post("http://127.0.0.1:9876/eval")
        .body(code)
        .send()
        .await
        .map_err(|e| e.to_string())?;
    resp.text().await.map_err(|e| e.to_string())
}
```

**Frontend (Svelte/React) — call the Tauri command:**

```ts
import { invoke } from '@tauri-apps/api/tauri'

async function evalCDP(code: string) {
  return invoke<string>('browser_eval', { code })
}
```

---

### 2.4 UX: guiding users through remote debugging setup

The biggest end-user friction is enabling Chrome's remote debugging. Handle this in your app's onboarding:

```js
// Electron: check if Chrome is reachable, guide user if not
async function checkChromeConnection() {
  try {
    const res = await fetch('http://127.0.0.1:9876/eval', {
      method: 'POST',
      body: 'await session.connect()'
    })
    if (res.ok) return 'connected'
  } catch {}

  // Chrome not reachable — open the setup page
  const { shell } = require('electron')
  shell.openExternal('chrome://inspect/#remote-debugging')
  return 'needs_setup'
}
```

Show a step-by-step prompt in the UI:

1. Open `chrome://inspect` in Chrome
2. Tick "Discover network targets"
3. Click **Allow** when Chrome prompts
4. Return to this app and click **Connect**

---

### 2.5 Auto-connecting to the right Chrome profile

If the user has multiple Chrome profiles or browsers, use `detectBrowsers()` to let them pick:

```js
const browsers = await detectBrowsers()
// Returns [{name, profileDir, port, wsUrl, mtimeMs}, ...]
// Sorted by most-recently-launched
```

Show this list in a settings screen. Store the chosen `profileDir` and connect explicitly:

```js
await session.connect({ profileDir: storedProfileDir })
```

---

## Part 3 — Android

### 3.1 Why Android is different

The harness server is Bun-native and CDP speaks WebSocket over a TCP port — neither is directly available in a standard Android app. There are four practical approaches depending on who the end user is.

---

### 3.2 Option A — Cloud-mediated (best for consumer apps)

The Android app is a thin client. The browser automation runs in the cloud on a headless Chrome instance.

```
Android App
    │  HTTPS (LLM API calls + tool results)
    ▼
Cloud Backend
    ├── LLM API (Claude/GPT)
    └── browser-harness-js server + headless Chrome
            │  CDP over WS
            ▼
        Headless Chromium (linux/docker)
```

**Cloud setup:**

```bash
# Dockerfile
FROM debian:bookworm-slim

RUN apt-get update && apt-get install -y chromium curl unzip \
    && curl -fsSL https://bun.sh/install | bash

COPY sdk/ /app/sdk/
WORKDIR /app

EXPOSE 9876

CMD ["bash", "-c", \
  "chromium --headless --remote-debugging-port=9222 --no-sandbox &\
   sleep 1 &&\
   ~/.bun/bin/bun sdk/repl.ts"]
```

The Android app calls your backend API, which calls the LLM, which calls the local `browser-harness-js` server inside the container. The app never talks to Chrome directly.

**Pros:** No Android-native constraints. Full CDP surface. Sharable sessions.
**Cons:** Requires cloud infra. Not offline. Latency.

---

### 3.3 Option B — ADB bridge (developer / power-user tools)

Android Chrome supports remote debugging over ADB. Port-forward Chrome's debug socket to the desktop, run the harness there, and the agent on the desktop drives the Android browser.

```
Android device
    └── Chrome (remote debugging via abstract socket)
            │  adb forward
            ▼
Desktop machine (port 9222)
    └── browser-harness-js → session.connect({ port: 9222 })
            │  tool calls
            ▼
        Agent / LLM (desktop or cloud)
```

**Setup:**

```bash
# 1. Enable USB debugging on the Android device
# 2. Enable remote debugging in Chrome (Settings → Developer tools → Enable)

# 3. Forward Chrome's debug socket to the desktop
adb forward tcp:9222 localabstract:chrome_devtools_remote

# 4. Connect the harness to the forwarded port
browser-harness-js 'await session.connect({ port: 9222 })'

# 5. List tabs on the Android Chrome
browser-harness-js 'return JSON.stringify(await listPageTargets())'
```

**Pros:** Full CDP on real Android Chrome. Zero cloud.
**Cons:** Requires ADB + USB/WiFi pairing. Not for end users.

---

### 3.4 Option C — Android app with embedded WebView (self-contained)

For a self-contained Android app that the agent controls, embed a `WebView` with remote debugging enabled and build a lightweight bridge server in Kotlin/Java.

**Enable WebView remote debugging (Kotlin):**

```kotlin
// In Application.onCreate() or Activity.onCreate()
if (BuildConfig.DEBUG) {
    WebView.setWebContentsDebuggingEnabled(true)
}

val webView = WebView(this)
webView.settings.javaScriptEnabled = true
webView.loadUrl("https://your-start-url.com")
```

This exposes a Unix abstract socket that ADB can forward. For production (no ADB), build an in-process HTTP bridge:

```kotlin
// Minimal NanoHTTPD server that evaluates JS in the WebView
class CdpBridgeServer(private val webView: WebView) : NanoHTTPD("127.0.0.1", 9876) {
    override fun serve(session: IHTTPSession): Response {
        if (session.method == Method.POST && session.uri == "/eval") {
            val body = session.inputStream.bufferedReader().readText()
            var result = ""
            val latch = CountDownLatch(1)

            webView.post {
                webView.evaluateJavascript(body) { value ->
                    result = value ?: ""
                    latch.countDown()
                }
            }

            latch.await(10, TimeUnit.SECONDS)
            return newFixedLengthResponse(result)
        }
        return newFixedLengthResponse(Response.Status.NOT_FOUND, MIME_PLAINTEXT, "")
    }
}
```

The app starts this bridge at launch. The agent POSTs JS snippets to it, which runs them inside the embedded WebView. The JavaScript API is the browser DOM — not the full CDP surface, but sufficient for most page interactions.

**Limitations vs full CDP:** `evaluateJavascript` runs in the page context, not the browser context. You get DOM/BOM APIs but not `Target.*`, `Network.*` (for interception), or multi-tab. Use this for agents that drive a single embedded browser view.

---

### 3.5 Option D — Termux (Android power users)

Bun has an arm64 Linux build. Users running Termux on a rooted or developer Android device can run the full harness stack:

```bash
# In Termux
pkg install curl git
curl -fsSL https://bun.sh/install | bash
source ~/.bashrc

# Clone and run
git clone https://github.com/browser-use/browser-harness-js
cd browser-harness-js/sdk
bun repl.ts &

# ADB-forward Android Chrome to the Termux server
# (from desktop, or use Termux's local port directly)
```

**Pros:** Full CDP stack on-device, no cloud.
**Cons:** Requires Termux + developer setup. Not a consumer packaging path.

---

## Part 4 — Decision Matrix

| Scenario | Recommended approach |
|---|---|
| AI coding agent (Claude Code, Cursor, Windsurf) | Shell tool via `browser-harness-js` CLI |
| Custom Python/Node agent | Direct HTTP to `/eval` endpoint |
| Consumer desktop app (macOS/Win/Linux) | Electron with bundled Bun binary |
| Performance-sensitive desktop app | Tauri with Bun sidecar |
| Consumer Android app | Cloud-mediated (Option A) |
| Enterprise Android app with controlled devices | ADB bridge (Option B) |
| Android app with single-page browser view | Embedded WebView bridge (Option C) |
| Developer / power-user Android tool | Termux + Bun (Option D) |

---

## Part 5 — Security Considerations

- The REPL server binds to `127.0.0.1` only — it is not reachable from other machines on the network.
- In a desktop app, keep the server on localhost and use IPC/local fetch to reach it from the renderer. Never expose port 9876 externally.
- In a cloud deployment, run the Bun server and Chrome inside a container with no external port exposure. The LLM API calls your backend; the backend calls localhost.
- The `/eval` endpoint executes arbitrary JS. In multi-user or cloud scenarios, isolate each user's session in a separate container or VM.
- Chrome remote debugging gives full control of the browser (cookies, stored passwords, extensions). Inform end users of this and obtain explicit consent before enabling it.
