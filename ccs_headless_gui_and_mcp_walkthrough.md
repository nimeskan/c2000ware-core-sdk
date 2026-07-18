# Driving the CCS 21 GUI Headlessly, and Activating Its Built-In MCP Servers

This document is a step-by-step technical record of how the full CCS 21
Electron/Theia desktop application — not just its CLI — was launched,
screenshotted, and driven with simulated mouse/keyboard input inside a
headless Linux container with **no display, no window manager, and no
physical GPU**, and how that was used to activate CCS's built-in
Model Context Protocol (MCP) servers and verify them against a live
instance.

It assumes CCS 21 is already installed per `get_started_userguide.md`
(install path `/root/ti/ccs2100/ccs`). Everything below is additional —
none of it is required for the CLI-only workflow in that guide.

---

## 1. Why this was necessary

`ccs-server-cli.sh` (the headless build/import CLI) is one integration
point. CCS 21 ships a *second*, separate integration: a bundled AI/MCP
extension (`ccs-ai`) at

```
/root/ti/ccs2100/ccs/theia/resources/app/plugins/ccs-ai/extension/
```

containing `dist/mcp-server-proxy.js`, documented in the extension's own
`README.md` as a Claude Code MCP server config. Running that proxy
directly just hangs — it's a stdio↔HTTP bridge to a **live, running CCS
instance**, and nothing was running one. Getting a real answer out of it
required actually starting the full CCS application, driving its GUI to
turn the feature on, and then reconnecting.

---

## 2. Tools installed to make this possible

None of these are part of a normal CCS install. All were added on top of
the base image:

| Tool | Package | Purpose |
|---|---|---|
| `Xvfb` | `xvfb` (pre-installed in this environment) | Virtual X11 framebuffer — a fake display CCS's Electron process can open, with no real monitor/GPU behind it |
| `xdotool` | `xdotool` | Sends synthetic mouse moves/clicks and keypresses to windows on an X display — used to actually operate the CCS UI |
| `xwd` | `x11-apps` | Dumps the raw pixel contents of an X display/window to a file |
| `convert` | `imagemagick` | Converts the `.xwd` dump to a normal `.png` so it can be viewed |
| `playwright-core` (npm) | — | A CDP-driving browser automation library — used only to load-test the CCS backend's raw HTTP endpoint, **not** to drive the actual app (see §5) |

Install commands used:

```bash
apt-get install -y --no-install-recommends x11-apps   # provides xwd
apt-get update && apt-get install -y --no-install-recommends imagemagick  # provides convert
apt-get install -y --no-install-recommends xdotool
cd /tmp && npm install playwright-core --no-save   # browser binary was already present at /opt/pw-browsers/chromium
```

---

## 3. Getting CCS's Electron app to actually launch

### 3.1 First failure: sandbox

```bash
/root/ti/ccs2100/ccs/theia/ccstudio
```
```
FATAL:electron_main_delegate.cc:290] Running as root without --no-sandbox is not supported.
```

Electron refuses to run its Chromium sandbox as root (this container runs
everything as root, with no unprivileged user configured). Fix: add
`--no-sandbox`.

### 3.2 Second failure: no display

```bash
/root/ti/ccs2100/ccs/theia/ccstudio --no-sandbox
```
```
Segmentation fault
[...] ERROR:ozone_platform_x11.cc:250] Missing X server or $DISPLAY
[...] ERROR:aura/env.cc:257] The platform failed to initialize.  Exiting.
```

There's no X server at all in this container (`$DISPLAY` was empty).
Fix: start one.

### 3.3 Starting a virtual display

```bash
Xvfb :99 -screen 0 1920x1080x24 > /tmp/xvfb.log 2>&1 &
disown
export DISPLAY=:99
```

`Xvfb` creates an X11 display entirely in memory — no real screen, no
GPU — that any X client (including Electron/Chromium) can connect to as
if it were a real monitor. `:99` is just an arbitrary display number
unlikely to collide with anything. `-screen 0 1920x1080x24` sets the
virtual screen's resolution/depth (this was bumped up from an initial
`1280x1024x24` — see §4.4 — because the app's side panels got clipped at
the smaller size).

### 3.4 Successful launch

```bash
mkdir -p /tmp/ccs_home
/root/ti/ccs2100/ccs/theia/ccstudio --no-sandbox --user-data-dir=/tmp/ccs_home \
  --window-size=1900,1050 > /tmp/ccs_launch.log 2>&1 &
disown
```

`--user-data-dir` points Electron's profile storage at a scratch
directory instead of the default `~/.config/...` (kept separate to allow
clean re-runs). `--window-size` requests an initial window size close to
the virtual screen's resolution.

Log output on success (harmless noise trimmed):
```
Configuration directory URI: 'file:///root/.config/Texas%20Instruments/CCS/ccs2100/0/theia'
CCSGuiComposerBackendContribution loaded...
CCSBackendApplicationContribution loaded...
CCSPluginHttpBackendImpl loaded...
Theia app listening on http://127.0.0.1:<port>.
```

**Errors that appear but are safe to ignore in this environment:**
- `Failed to connect to the bus: ... /run/dbus/system_bus_socket` — no
  D-Bus daemon in this container; only affects desktop-integration
  features (tray icons, etc.), not the app itself.
- `Exiting GPU process due to errors during initialization` /
  `ContextResult::kTransientFailure` — no real GPU; Chromium falls back
  to software rendering and keeps working.
- `CreatePlatformSocket() failed: Address family not supported by
  protocol (97)` — IPv6 socket attempts failing in this container's
  network stack; unrelated to the UI.
- `handshake failed; ... net_error -101` — an outbound HTTPS call (looks
  like a license/update/telemetry check) failing to complete; did not
  block anything we needed.

---

## 4. Getting visuals: why Playwright was the wrong tool, and what worked instead

### 4.1 The instinct: point Playwright at the HTTP port

The Theia backend log showed it listening on a real HTTP port. The
obvious move was to treat it like any other web app and drive it with
Playwright:

```js
const { chromium } = require('playwright-core');
const browser = await chromium.launch({
  executablePath: '/opt/pw-browsers/chromium',
  args: ['--no-sandbox'],
});
const page = await browser.newPage({ viewport: { width: 1280, height: 900 } });
await page.goto('http://127.0.0.1:<port>/', { waitUntil: 'networkidle' });
await page.screenshot({ path: '/tmp/shot.png' });
```

This *did* return `HTTP 200` and a real HTML document (`<title>CCStudio
IDE</title>`), and a screenshot could be taken — but it only ever showed
CCS's loading splash (a red spinning cube), never the actual workbench.

### 4.2 Diagnosing why: it's not a website, it's an Electron app that happens to use HTTP

Capturing the browser's own console/error output revealed the real
cause:

```js
page.on('console', msg => console.log(`[${msg.type()}] ${msg.text()}`));
page.on('pageerror', err => console.log('[pageerror]', err.message));
```
```
[pageerror] Cannot read properties of undefined (reading 'WindowMetadata')
[warning] WebSocket connection to 'ws://127.0.0.1:<port>/socket.io/...' failed
[error] Failed to load resource: the server responded with a status of 403 (Forbidden)
```

Theia frontends built for an Electron desktop shell expect an **Electron
preload script** to have already injected a global bridge object (things
like window metadata, native menu control, IPC to the main process)
*before* the page's own JavaScript runs. That bridge only exists inside
Electron's own `BrowserWindow` renderer process — a `BrowserWindow` is
still a Chromium tab under the hood, but it's launched by Electron's main
process with `contextBridge`/`preload.js` wiring specific to that
instance. An **external** browser (Playwright's own Chromium, launched
independently) connecting to the same URL is just a plain tab: it gets
all the same JS and HTML, but none of that injected bridge, so the app's
own startup code throws immediately trying to read a property off
`undefined` and never gets past the splash screen.

In short: **the HTTP port serves the same assets Electron's window loads,
but only Electron's own window has the context needed to actually run
them.** Playwright was the wrong layer entirely for seeing or driving the
real UI — it's useful here only as a coincidental way to confirm the HTTP
server responds, not as a way to render the app.

### 4.3 What actually worked: screenshot the real X display, not a browser tab

Since Electron's own window was already rendering correctly inside the
`Xvfb` display (the framebuffer doesn't care that no monitor is attached
— it renders regardless), the fix was to capture *that*, using ordinary
X11 tools instead of a browser:

```bash
export DISPLAY=:99
xwd -root -out /tmp/shot.xwd        # dump the whole virtual screen's pixels
convert /tmp/shot.xwd /tmp/shot.png # convert to a normal viewable image
```

`xwd -root` grabs the entire root window (i.e. the whole virtual screen,
including every window on it) rather than a single window ID, which
avoids needing to look up a specific window handle just to get a first
look at what's on screen. This immediately showed the real, fully
rendered CCS IDE — menu bar, activity bar, welcome page, a workspace
trust dialog — everything Playwright's tab never got to.

### 4.4 Iterating on window size/position

The first captures showed the actual app window occupying only a small
region of the virtual screen (its own internal geometry was
853×683 at position 214,171, inside a larger 1280×1024 virtual screen),
which clipped side panels. Two levers fixed this:

```bash
# reposition/resize the existing window
xdotool getwindowgeometry <window-id>
xdotool windowmove <window-id> 10 10
xdotool windowsize <window-id> 1260 1000
```

That alone wasn't enough once a wide side panel (see §6) needed more
room than the window had, so the virtual screen itself was made bigger
and the app relaunched fresh against it:

```bash
pkill -f ccstudio
kill -9 <old-Xvfb-pid>   # `pkill Xvfb` left a zombie in one run; kill -9 by pid was reliable
Xvfb :99 -screen 0 1920x1080x24 &
disown
export DISPLAY=:99
/root/ti/ccs2100/ccs/theia/ccstudio --no-sandbox --user-data-dir=/tmp/ccs_home2 \
  --window-size=1900,1050 &
disown
```

Finding the app's actual window ID (as opposed to Electron's various
helper/zygote windows) was done with:

```bash
xdotool search --name "CCStudio"
```
which returns multiple IDs (main window, a small helper window, etc.) —
`getwindowgeometry` on each was used to identify which one was the real,
large application window.

---

## 5. Driving the UI: `xdotool`

With a correctly sized, visible window, interaction was done purely by
sending synthetic input to absolute screen coordinates read off each
screenshot — there is no accessibility-tree/DOM-based automation here,
just "click at pixel (x, y)":

```bash
export DISPLAY=:99

# click a button/link at a specific pixel position
xdotool mousemove <x> <y> click 1

# scroll a scrollable pane
xdotool mousemove <x> <y>
xdotool click 5      # mouse-wheel down, one notch
xdotool click 4      # mouse-wheel up, one notch

# keyboard paging (used once, over-scrolled — mouse wheel in small
# increments proved more controllable for this UI)
xdotool key --repeat 6 Page_Down
```

Practical workflow used throughout: **screenshot → look at pixel
coordinates of the thing to click → `xdotool mousemove/click` → wait a
couple seconds for the UI to react → screenshot again to confirm.** This
is slow and fully manual (each click requires a fresh screenshot to find
the next target), but reliable, since it's operating on the exact same
rendering the real user would see.

One approach that did *not* work: dragging a panel's border to resize it
(`mousemove` to the border → `mousedown` → `mousemove` to a new position
→ `mouseup`). Instead of resizing the panel, it just selected the text
underneath the drag path — the border's drag-target hit-region was
apparently narrower than assumed. Growing the virtual screen and window
(§4.4) sidestepped the problem instead of finding the exact pixel.

---

## 6. Turning on the MCP servers, end to end

Steps performed against the running, visible window (screenshots omitted
here; see chat history for the actual captures):

1. On the default "Get Started" welcome page, scrolled to a **"Get
   started with AI"** section, then clicked **"Set up an AI assistant."**
2. This opened an **"AI ASSISTANT CONFIGURATOR"** side panel, already
   defaulted to:
   - **Agent:** `Claude Code`
   - **Provider:** `Anthropic`
   - **MCP Servers → "Enable CCStudio IDE ecosystem MCP servers for
     Claude Code"** — checkbox already ticked
3. Clicked **Apply**. This triggered a visible **"Installing Claude
   Code..."** progress notification (the panel installs the `claude` CLI
   itself if not already present — it landed at `/opt/node22/bin/claude`
   in this environment), followed by **"AI assistant configured
   successfully"** and an "✓ Installed" badge next to the Agent dropdown.

### What that click actually changed on disk

```bash
# a new workspace-scoped MCP config, with the exact 4 servers from the README:
cat /root/workspace_ccstheia/.mcp.json
```
```json
{
  "mcpServers": {
    "ccs-debug":     { "command": ".../node", "args": [".../mcp-server-proxy.js", "debug"] },
    "ccs-project":   { "command": ".../node", "args": [".../mcp-server-proxy.js", "project"] },
    "ccs-sysconfig": { "command": ".../node", "args": [".../mcp-server-proxy.js", "sysconfig"] },
    "ccs-serial":    { "command": ".../node", "args": [".../mcp-server-proxy.js", "serial"] }
  }
}
```

It also rewrote the global `~/.claude.json` (a timestamped backup was
auto-created under `~/.claude/backups/`), but did **not** touch the
per-project settings for any other working directory — this was checked
explicitly by inspecting that file's `projects` map before and after.

Most importantly, it started the actual backend MCP HTTP listeners and
wrote a discovery file:

```bash
cat /tmp/ccs-ai/ccs-ai-port-<hash>
```
```json
{
  "servers": {
    "log":       { "serverUrl": "http://127.0.0.1:43575/log" },
    "debug":     { "serverUrl": "http://127.0.0.1:43575/mcp/debug" },
    "project":   { "serverUrl": "http://127.0.0.1:43575/mcp/project" },
    "serial":    { "serverUrl": "http://127.0.0.1:43575/mcp/serial" },
    "sysconfig": { "serverUrl": "http://127.0.0.1:36511/mcp" }
  }
}
```

This file did not exist at all before clicking Apply — its path is
computed by the extension as
`os.tmpdir() + "/ccs-ai/ccs-ai-port-" + sha256(workspacePath)[:7]`, so it
only appears once a workspace's AI feature has actually been activated.

---

## 7. Verifying an MCP server for real

### 7.1 First attempt: piping a static file — worked, but truncated

```bash
cd /root/workspace_ccstheia
cat > requests.jsonl <<'EOF'
{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2024-11-05","capabilities":{},"clientInfo":{"name":"test","version":"1.0"}}}
{"jsonrpc":"2.0","method":"notifications/initialized"}
{"jsonrpc":"2.0","id":2,"method":"tools/list"}
EOF
node .../mcp-server-proxy.js project < requests.jsonl
```

This returned a full, valid `initialize` response and (with enough of a
timeout) a full `tools/list` response — but piping a **static file** as
stdin means stdin hits EOF the instant all lines are written, and with
more than one round-trip queued up, later responses could arrive after
the process had already begun tearing down. Fine for one or two quick
calls, unreliable beyond that.

### 7.2 Reliable approach: a small persistent Node client

```js
// mcp_client.js
const { spawn } = require('child_process');
const proxy = spawn('node', ['.../mcp-server-proxy.js', 'project'], {
  cwd: '/root/workspace_ccstheia',
  stdio: ['pipe', 'pipe', 'pipe'],
});

let buf = '';
proxy.stdout.on('data', (d) => {
  buf += d.toString();
  let idx;
  while ((idx = buf.indexOf('\n')) >= 0) {
    const line = buf.slice(0, idx);
    buf = buf.slice(idx + 1);
    if (line.trim()) console.log('RESPONSE:', line);
  }
});
proxy.stderr.on('data', (d) => console.error('STDERR:', d.toString()));

function send(obj) { proxy.stdin.write(JSON.stringify(obj) + '\n'); }

send({ jsonrpc: '2.0', id: 1, method: 'initialize', params: {
  protocolVersion: '2024-11-05', capabilities: {}, clientInfo: { name: 'test', version: '1.0' },
}});
setTimeout(() => {
  send({ jsonrpc: '2.0', method: 'notifications/initialized' });
  setTimeout(() => send({ jsonrpc: '2.0', id: 2, method: 'tools/call',
    params: { name: 'getActiveProjectName', arguments: {} } }), 500);
  setTimeout(() => send({ jsonrpc: '2.0', id: 3, method: 'tools/call',
    params: { name: 'getCompilers', arguments: {} } }), 1500);
  setTimeout(() => { proxy.stdin.end(); proxy.kill(); process.exit(0); }, 10000);
}, 1000);
```

Keeping stdin open as a real pipe (rather than a file redirect) and
spacing requests out with `setTimeout` let every response come back
before the process was torn down. Results:

- `tools/call getActiveProjectName` → `{"content":[{"type":"text","text":"Success"}],"isError":false}`
- `tools/call getCompilers` → real, live data:
  ```json
  {
    "C2000": {
      "tools": ["C2000:25.11.1.LTS", "C2000:TICLANG_2.2.0.LTS"]
    }
  }
  ```
  matching the actual compiler directories installed under
  `/root/ti/ccs2100/ccs/tools/compiler/`.

This confirmed two things at once: the MCP server is genuinely live and
answering with real IDE state, and the tool list/schemas obtained earlier
by statically reading the minified `extension.js` bundle (see the
session's tool-enumeration work) were byte-for-byte accurate — `tools/list`
returned the identical 14 tool names, descriptions, and input schemas.

---

## 8. Summary: minimal reproduction

```bash
# one-time tool install
apt-get install -y --no-install-recommends x11-apps imagemagick xdotool

# start a virtual display
Xvfb :99 -screen 0 1920x1080x24 & disown
export DISPLAY=:99

# launch CCS
mkdir -p /tmp/ccs_home
/root/ti/ccs2100/ccs/theia/ccstudio --no-sandbox --user-data-dir=/tmp/ccs_home \
  --window-size=1900,1050 & disown
sleep 18   # let the backend + renderer finish initializing

# find and position the real window
xdotool search --name "CCStudio"          # note the window id with real content
xdotool windowmove <id> 10 10
xdotool windowsize <id> 1880 1030

# screenshot loop (repeat as needed while clicking through the UI)
xwd -root -out /tmp/shot.xwd && convert /tmp/shot.xwd /tmp/shot.png

# click at coordinates read off the screenshot
xdotool mousemove <x> <y> click 1

# once "Set up an AI assistant" -> Apply has been clicked in the GUI,
# the MCP servers are live; find the port file:
cat /tmp/ccs-ai/ccs-ai-port-*

# talk to a server for real (see §7.2 for the persistent-client version)
cd /root/workspace_ccstheia
node /root/ti/ccs2100/ccs/theia/resources/app/plugins/ccs-ai/extension/dist/mcp-server-proxy.js project
```
