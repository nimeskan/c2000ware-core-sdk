# SysConfig MCP Server — Tool Reference

This documents the **`sysconfig`** MCP server bundled with CCS 21's `ccs-ai`
extension (one of four: `debug`, `project`, `sysconfig`, `serial`). Unlike
the other three servers, this reference was captured by querying a
**live, running instance** via `tools/list` — not from static analysis of
the bundled JS — so it reflects exactly what the server reports about
itself.

See `ccs_headless_gui_and_mcp_walkthrough.md` for how to get a live CCS
instance running headlessly and reach this server at all, and
`get_started_userguide.md` for the base CCS/SysConfig install.

---

## How to reach it

Once CCS's AI assistant feature has been enabled for a workspace (via the
"Set up an AI assistant" panel, or because it was already enabled in a
previous session against the same workspace), a port-discovery file
appears at:

```
$TMPDIR/ccs-ai/ccs-ai-port-<hash-of-workspace-path>
```

containing a `sysconfig` entry like:

```json
"sysconfig": { "serverUrl": "http://127.0.0.1:<port>/mcp", "port": <port> }
```

The bundled proxy talks to that live server over stdio↔HTTP:

```bash
cd <the-workspace-directory>   # must match the workspace the port file was generated for
node /root/ti/ccs2100/ccs/theia/resources/app/plugins/ccs-ai/extension/dist/mcp-server-proxy.js sysconfig
```

It speaks standard MCP over stdio (JSON-RPC: `initialize` →
`notifications/initialized` → `tools/list` / `tools/call`).

---

## Server identity & instructions

```
serverInfo: { name: "ccs-mcp-sysconfig-proxy", version: "1.0.0" }
```

The server's own system prompt (returned in the `initialize` response's
`instructions` field):

> # SysConfig
>
> SysConfig is a tool that captures user configuration and generates
> textual artifacts based on configuration. The configuration may be for
> low level hardware components, or higher level software abstractions.
> The textual artifacts are often `.c` and `.h` files that are used in a
> build, but they could be of any file type. The configuration is stored
> in a `.syscfg` file for later use.
>
> **Important**: Always use the provided tools to read and modify
> configuration. Never directly edit `.syscfg` files, as these tools
> ensure proper validation, maintain internal consistency, and provide
> immediate feedback on errors.
>
> ## Key Concepts
> - **Module**: A type of configurable component
> - **Instance**: A specific instantiation of a module with its own
>   unique settings
> - **Configurable**: A name/value pair that controls instance behavior
>
> ## Required First Steps
> 1. Use `openFile` to select the `.syscfg` file you want to work with
> 2. **After `openFile` returns, new tools will appear in your tool
>    list** — configuration tools such as `listModules`,
>    `getInstanceConfiguration`, `changeConfiguration`, and others are
>    dynamically added once a file is open
> 3. **Read the `additionalInstructions` field in the `openFile`
>    result** — it contains essential guidance for using the
>    configuration tools now available. Follow those instructions when
>    working with the opened file.
>
> **Important**: All configuration tools operate **exclusively** on the
> currently selected file. If no file is selected, configuration tools
> will return an error directing you to use `openFile`.
>
> ### When to Use `listFiles`
> Only use `listFiles` if you need to discover which `.syscfg` files are
> available in the workspace. If you already know the file path (e.g.,
> from the user's message or context), use `openFile` directly with that
> path.

### Practical consequence of this design

`tools/list` immediately after `initialize` only returns **4 tools**:
`listFiles`, `openFile`, `closeFile`, `inspectExample`. The other 19 tools
documented below **do not exist in the tool list until `openFile` has
successfully opened a `.syscfg` file** — they are dynamically registered
per-session, scoped to whichever file is currently open. Calling any of
them before opening a file, or after `closeFile`, returns an error
telling you to call `openFile` first.

---

## Tool reference

Total: **23 tools** (4 always available + 19 that appear after `openFile`).

### Always available

#### `listFiles`
- **Params:** none
- **Description:** Lists all `.syscfg` configuration files available in
  the workspace. Indicates which file is currently selected. No file may
  be selected initially or after calling `closeFile`. Each file contains
  the configuration for its project and generates build artifacts (`.c`
  and `.h` files).

#### `openFile`
- **Params:** `filePath` (required)
- **Description:** Opens a `.syscfg` configuration file and loads the
  appropriate SysConfig version for it. This must be called before using
  configuration tools such as `listModules`, `getInstanceConfiguration`,
  or `changeConfiguration`. If you know the file path, pass it directly
  to this tool. If you need to discover available files, use `listFiles`
  first. Returns instructions for how to use the configuration tools now
  available (in `additionalInstructions`).

  > **⚠️ Known headless limitation — see "Known limitation: first-time
  > device-selection dialog" below.** The very first time a given
  > `.syscfg` file is opened by *any* client (MCP or the GUI itself) in a
  > CCS session, if the project's device ID isn't a fully specific part
  > number, `openFile` **hangs indefinitely** waiting on a GUI modal
  > dialog that a pure MCP client cannot answer. It is not a timing
  > fluke — it reproduced identically across three separate attempts.

#### `closeFile`
- **Params:** none
- **Description:** Closes the currently open file. Configuration tools
  will no longer be available until another file is opened.

#### `inspectExample`
- **Params:** `fileName` (required), `force`
- **Description:** Reads the configuration from an example `.syscfg` file
  in the SDK. Use this for complex configuration tasks when you need
  guidance on how to configure modules for specific use cases. Only
  non-default values are returned — settings that match their initial
  default are omitted. Each setting includes a `setByUser` boolean: true
  means explicitly set by the user, false means automatically changed as
  a side effect of other settings.

### Available only after `openFile` — module discovery

#### `listModules`
- **Params:** none
- **Description:** Returns a list of all available modules with a short
  description of each. This list is fixed for the lifetime of the
  session. Use `getModuleDescription` to get a more detailed description
  for specific modules.

#### `getModuleDescription`
- **Params:** `moduleIds` (required)
- **Description:** Returns detailed descriptions for one or more modules.
  Descriptions are fixed for the lifetime of the session.

#### `getModuleInstances`
- **Params:** `moduleIds` (optional)
- **Description:** Returns all module instances in the configuration,
  optionally filtered to specific modules.

#### `getInstanceConfiguration`
- **Params:** `instances` (required), `changesOnly`, `ids`, `searchText`,
  `includeDescriptions`, `includeChoices`
- **Description:** Returns the detailed configuration for one or more
  specific module instances, including configurables, pinmux settings,
  child modules, and validation messages (errors/warnings). Efficient
  usage pattern: first call with full details (`includeDescriptions:
  true`), then use `changesOnly: true` with `includeDescriptions: false`
  and `includeChoices: false` for subsequent status checks. Use `ids` to
  query specific configurables, or `searchText` to explore/discover
  configurables.

### Available only after `openFile` — editing configuration

#### `addModuleInstances`
- **Params:** `moduleIds` (required), `maxConfigurables`
- **Description:** Adds one or more instances of the specified modules
  to the system configuration. Returns all created instances — both
  directly requested and auto-created as dependencies — with identity,
  relationship information, and `configurableCount`. Automatically
  returns a lightweight representative configuration (`id`, `name`,
  `value`, `choices`, `readOnly` — no descriptions) for each new module
  type with `configurableCount` ≤ `maxConfigurables` (default 50). For
  module types exceeding the threshold, use `configurableCount` from
  `addedInstances` to estimate token cost (~30 tokens per configurable),
  then call `getInstanceConfiguration` with `searchText` or `ids` filters.

#### `removeModuleInstances`
- **Params:** `instances` (required)
- **Description:** Removes one or more module instances from the system
  configuration. Returns all removed instances (including any removed as
  side-effects) and any instances auto-created as a reaction. All arrays
  are omitted when empty.

#### `changeConfiguration`
- **Params:** `changes` (required), `description` (required)
- **Description:** Changes the values of one or more configurables
  atomically — if any change fails, all are reverted. Returns a delta of
  what changed: per-instance arrays of available/changed/unavailable
  configurables, plus `addedInstances`/`removedInstances` for
  module-level side-effects. All arrays are omitted when empty.

#### `getErrorsAndWarnings`
- **Params:** none
- **Description:** Returns all error and warning messages across all
  module instances. Each entry is a single message with the configurable
  ID where it appears (when applicable). Module-level messages not
  associated with a specific configurable omit `configurableId`. Use this
  instead of `getInstanceConfiguration` when you only need error/warning
  messages.

#### `save`
- **Params:** none
- **Description:** Saves the current configuration to the `.syscfg` file
  and regenerates all artifacts. Call this after making configuration
  changes to persist them and update the generated artifacts. Returns the
  saved `.syscfg` path, the list of generated artifacts, and any files
  skipped due to configuration errors.

### Available only after `openFile` — device migration

#### `listMigrationTargets`
- **Params:** `includeCompatibility` (optional)
- **Description:** Lists all possible targets (device, package, variant)
  that the current configuration could be migrated to (these fields are
  simply absent if not available). Use this tool before performing a
  migration to discover what the user can migrate to.

#### `migrate`
- **Params:** `device` (required), `package` (required), `variant`
  (optional)
- **Description:** Migrates the current device configuration to the
  specified target device, package, and variant. Use
  `listMigrationTargets` first to discover valid targets.

### Available only after `openFile` — pin/connection graph

#### `getInstanceConnections`
- **Params:** `instances` (optional), `includeConnectionTypes` (optional)
- **Description:** Returns the connection ports available on module
  instances and their current connection state. Ports have types that
  determine compatibility. Use the connection-types rules to understand
  which port types can connect to each other.

#### `manageConnections`
- **Params:** `disconnect` (optional), `connect` (optional)
- **Description:** Creates or removes connections between module
  instance ports. This tool exclusively manages connections — it cannot
  change configurable values or add/remove module instances. All
  operations are applied atomically — if any fails, all are reverted.

#### `getInstanceLayout`
- **Params:** `instances` (optional)
- **Description:** Returns the position and size of module instances in
  the graphical view. Groups create isolated layout contexts: members of
  a group have positions relative to their group's coordinate space,
  while non-members and groups themselves share the top-level coordinate
  space. When laying out, treat group members as a completely separate
  graph from non-members.

#### `setInstancePositions`
- **Params:** `positions` (required)
- **Description:** Sets the positions of module instances in the
  graphical view. Group members and top-level instances occupy separate
  coordinate spaces — see layout instructions for correct positioning
  rules.

### Available only after `openFile` — clock tree

#### `getClockTreeInstances`
- **Params:** `includeFilter` (optional), `excludeFilter` (optional),
  `instances` (optional)
- **Description:** Returns clock tree instances and their connection
  topology.

#### `traceClockSignal`
- **Params:** `toInstance` (required), `toPin` (required)
- **Description:** Traces backward from a specific pin to show the
  complete signal path with frequencies. Returns a sequential list of
  pins from the specified destination to the source. Adjacent pins with
  the same frequency represent a connection (wire). When the same
  instance appears with different frequencies on different pins, a
  transformation occurred inside that instance.

### Available only after `openFile` — generated output

#### `listGeneratedArtifacts`
- **Params:** none
- **Description:** Lists all artifacts that will be generated based on
  the current configuration. Artifacts are included only if the required
  modules are present. Artifacts that would be generated but are blocked
  by configuration errors are also indicated.

#### `readGeneratedArtifact`
- **Params:** `filePath` (required)
- **Description:** Reads the contents of a specific generated artifact by
  its file path.

---

## Quick-reference table

| Tool | Required params | Available before `openFile`? |
|---|---|---|
| `listFiles` | — | Yes |
| `openFile` | `filePath` | Yes |
| `closeFile` | — | Yes |
| `inspectExample` | `fileName` | Yes |
| `listModules` | — | No |
| `getModuleDescription` | `moduleIds` | No |
| `getModuleInstances` | — | No |
| `getInstanceConfiguration` | `instances` | No |
| `addModuleInstances` | `moduleIds` | No |
| `removeModuleInstances` | `instances` | No |
| `changeConfiguration` | `changes`, `description` | No |
| `getErrorsAndWarnings` | — | No |
| `save` | — | No |
| `listMigrationTargets` | — | No |
| `migrate` | `device`, `package` | No |
| `getInstanceConnections` | — | No |
| `manageConnections` | — | No |
| `getInstanceLayout` | — | No |
| `setInstancePositions` | `positions` | No |
| `getClockTreeInstances` | — | No |
| `traceClockSignal` | `toInstance`, `toPin` | No |
| `listGeneratedArtifacts` | — | No |
| `readGeneratedArtifact` | `filePath` | No |

---

## Known limitation: first-time device-selection dialog blocks `openFile`

### Symptom

Calling `openFile` on a `.syscfg` file whose owning `.projectspec` used a
generic device ID (e.g. `F280013x` rather than a concrete part like
`TMS320F2800137`) **never returns**. The MCP client eventually gives up
with:

```json
{"error": {"code": -32001, "message": "MCP error -32001: MCP error -32001: Request timed out"}}
```

This is not a slow response — it is a genuine hang. It reproduced
identically across three separate, independent attempts, each timing out
at exactly 60 seconds:

```
2026-07-18T03:30:31.798Z -> Tool call: openFile
2026-07-18T03:31:31.809Z x- Tool call: openFile: MCP error -32001: Request timed out

2026-07-18T03:32:48.114Z -> Tool call: openFile
2026-07-18T03:33:48.126Z x- Tool call: openFile: MCP error -32001: Request timed out

2026-07-18T03:35:47.419Z -> Tool call: openFile
2026-07-18T03:36:47.433Z x- Tool call: openFile: MCP error -32001: Request timed out
```

(from `~/.config/Texas Instruments/CCS/ccs2100/0/theia/ccs_theia.log`).
Every other tool call in the same log completes in milliseconds — only
`openFile` ever hangs.

### Root cause

`openFile` launches a real SysConfig engine subprocess
(`.../sysconfig_1.28.0/dist/runtimeServer.js --uiPort <n> --proxyPort
<n> ...`), confirmed alive in `ps aux` for the entire duration of every
hung attempt. But neither of its advertised ports ever actually accepts
a connection (`curl` to both returned `Connection refused` throughout).

The actual blocker, found by screenshotting the live CCS window while an
`openFile` call was in flight: CCS pops up a **"Select Device" modal
dialog**:

> SysConfig has been updated to use standard TI part numbers. The device
> `F280013x` was specified on the command line. Please select the
> specific device to use: **Device:** `TMS320F2800137` **Package:**
> `64PM` **Variant:** `default` — **[CONFIRM]**

This dialog requires a human (or GUI-automation) click. A pure MCP
client — including CCS's own bundled `Claude Code` agent integration,
unless it's also independently driving the GUI — has no channel to
answer it, so the underlying operation waits forever. The 60-second
error is just the MCP transport's own client-side request timeout firing
and reporting failure back to the caller; it does **not** mean the
server gave up — the hung `runtimeServer.js` process and unopened ports
persist unless the process is killed.

### Workaround (used to confirm this)

Resolve the dialog once, by hand, through the actual GUI (see
`ccs_headless_gui_and_mcp_walkthrough.md` for how the GUI is driven
headlessly via `Xvfb` + `xdotool`):

```bash
export DISPLAY=:99
xwd -root -out /tmp/shot.xwd && convert /tmp/shot.xwd /tmp/shot.png   # find the CONFIRM button's coordinates
xdotool mousemove <x> <y> click 1                                     # click CONFIRM
```

After that one-time click, calling `openFile` again for the **same
file** — from a brand-new `mcp-server-proxy.js sysconfig` connection —
completed successfully in **1.6 seconds**, returning the full
`additionalInstructions` payload and expanding the tool list. A
follow-up `getModuleInstances` call then returned genuine live data:

```json
{
  "instances": [
    {"moduleId": "/driverlib/epwm.js", "instances": [
      {"moduleInstanceId": "myEPWM1", "configurableCount": 168},
      {"moduleInstanceId": "myEPWM2", "configurableCount": 171},
      {"moduleInstanceId": "myEPWM3", "configurableCount": 171},
      {"moduleInstanceId": "myEPWM4", "configurableCount": 171}
    ]},
    {"moduleId": "/driverlib/sync.js", "instances": [
      {"moduleInstanceId": "$static", "configurableCount": 5}
    ]},
    {"moduleId": "/driverlib/inputxbar_input.js", "instances": [
      {"moduleInstanceId": "myINPUTXBARINPUT0", "configurableCount": 5},
      {"moduleInstanceId": "myINPUTXBARINPUT1", "configurableCount": 5}
    ]}
  ]
}
```

### Practical implications

- **Each fresh `.syscfg` file** whose device isn't already fully
  resolved will hit this dialog exactly once. Projects built from
  `.projectspec` files with a concrete device ID (not a generic family
  name) may never trigger it at all — worth testing per-project.
- **MCP session state is per-connection, not global.** Resolving the
  dialog via the GUI, or via one MCP connection's `openFile` call, does
  **not** carry over to a different MCP connection querying the same
  file — each new `mcp-server-proxy.js sysconfig` process is its own
  session against the backend and must independently call `openFile`
  again (though it's fast once the device is already resolved for that
  file).
- **A pure MCP-only agent, with no GUI access, cannot get past this by
  itself** on a file hitting it for the first time. It needs either a
  human at the actual CCS window, or a GUI-automation layer (as
  documented in `ccs_headless_gui_and_mcp_walkthrough.md`) sitting
  alongside it, watching for and dismissing this dialog.

---

## How this was captured

A persistent Node.js child-process client (see
`ccs_headless_gui_and_mcp_walkthrough.md` §7.2 for the pattern) sent a
real `initialize` → `notifications/initialized` → `tools/list` sequence
to `mcp-server-proxy.js sysconfig` while a live CCS instance was running,
and the exact JSON response was parsed to produce this document.

**Discrepancy worth flagging:** the server's own `initialize` instructions
(quoted above) explicitly state that only `listFiles`/`openFile`/
`closeFile`/`inspectExample` are available until a `.syscfg` file is
opened, and that the rest are "dynamically added" afterward. In practice,
the single `tools/list` call in this session — made without ever calling
`openFile` first — returned **all 23 tools already**, including
`listModules`, `getInstanceConfiguration`, `changeConfiguration`, etc.
This may mean the tool *list* is static from server start (openFile only
gates whether calling them *succeeds*, not whether they're listed), or
that some prior state in this CCS instance left a file already
associated with the session. This was not independently re-tested against
a freshly-started server with zero prior activity, so treat the
"available before `openFile`?" column above as documented intent from the
server's own instructions, not as independently confirmed behavior for
every tool.
