# SysConfig MCP Server — Tool Reference

This documents the **`sysconfig`** MCP server bundled with CCS 21's `ccs-ai`
extension (one of four: `debug`, `project`, `sysconfig`, `serial`). This
reference was captured by querying a **live, running instance** directly
via `tools/list` — including each tool's declared JSON Schema
`inputSchema`/`outputSchema` — not from static analysis of the bundled JS,
so it reflects exactly what the server reports about itself.

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

### What actually happens (verified against a live server, not just the instructions text above)

The instructions above claim only 4 tools exist until `openFile` is
called. **That is not what was observed.** Two independent `tools/list`
calls made immediately after `initialize`, with no `openFile` call first,
both returned **all 23 tools already** — including `listModules`,
`getInstanceConfiguration`, `changeConfiguration`, etc. Calling any of
the file-scoped tools before a file is actually open returns a runtime
error (e.g. `"No configuration file is open. Use 'listFiles' to see
available files, then 'openFile' to select one."`), so the *description
text* is right about behavior, just wrong about what shows up in
`tools/list`.

What genuinely does change after `openFile` succeeds: the tool list
observed for this specific project (`epwm_ex3_synchronization`, device
`F280013x`) **shrank from 23 to 19** — four pin/graphical-layout tools
disappeared entirely:

- `getInstanceConnections`
- `manageConnections`
- `getInstanceLayout`
- `setInstancePositions`

`openFile`'s own result includes a `validTools` array — the real,
authoritative list of which tools are usable for that file's resolved
SysConfig version. For this file it listed exactly those same 19 tools.
The four missing ones are documented in `openFile`'s `outputSchema`
description as being "for features not supported by this file's
SysConfig version" — most plausibly the graphical pinmux/connection view,
which may not apply to every device family. This wasn't tested against a
device/file where those 4 tools *do* remain available, so treat "pin
graph tools may disappear for some devices" as confirmed, but the exact
condition that keeps them available as unconfirmed.

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
`additionalInstructions` payload and expanding the tool list. Every
subsequent `openFile` call against that same file (even from further new
connections) has stayed fast (~1.7s) since — confirmed across multiple
follow-up calls including the `save` and `listModules` calls documented
below.

### Practical implications

- **Each fresh `.syscfg` file** whose device isn't already fully
  resolved will hit this dialog exactly once. Projects built from
  `.projectspec` files with a concrete device ID (not a generic family
  name) may never trigger it at all — worth testing per-project.
- **The resolution is per-file, not per-connection.** MCP session state
  (which tools are registered, whether a file is "open") is
  per-connection and does not persist across separate
  `mcp-server-proxy.js sysconfig` invocations — each new connection must
  call `openFile` again. But *device resolution* for a given file, once
  settled, is remembered by the backend and makes every later `openFile`
  call for that same file fast, regardless of which connection makes it.
- **A pure MCP-only agent, with no GUI access, cannot get past this by
  itself** on a file hitting it for the first time. It needs either a
  human at the actual CCS window, or a GUI-automation layer (as
  documented in `ccs_headless_gui_and_mcp_walkthrough.md`) sitting
  alongside it, watching for and dismissing this dialog.

---

## Tool reference

All 23 tools, with full input/output schemas as declared by the live
server itself (not inferred from example responses). Grouped by what
they do; each heading notes whether the tool was present in the
post-`openFile` (19-tool) list for the test file, or only in the
pre-`openFile`/generic-connection-tools (23-tool) list.

### File / session management

#### `listFiles`
*(available in both lists)*
- **Params:** none
- **Returns:**
  | Field | Type | Notes |
  |---|---|---|
  | `files` | array (required) | one entry per `.syscfg` in the workspace |
  | `files[].filePath` | string (required) | absolute path |
  | `files[].name` | string (required) | basename |
  | `files[].selected` | boolean (required) | true if this is the currently-open file for this session |
  | `files[].sysConfigVersion` | string (required) | SysConfig version required for this file |
- **Description:** Lists all `.syscfg` configuration files available in
  the workspace. Indicates which file is currently selected. No file may
  be selected initially or after calling `closeFile`.

#### `openFile`
*(available in both lists)*
- **Params:** `filePath` (string, required) — absolute path to the `.syscfg` file
- **Returns:**
  | Field | Type | Notes |
  |---|---|---|
  | `additionalInstructions` | string (required) | workflow guidance for using the now-available tools |
  | `validTools` | array of strings (required) | "When present, only these tools are valid to call for the opened file. Other tools visible in the tool list are for features not supported by this file's SysConfig version — calling them will fail." |
- **Description:** Opens a `.syscfg` file and loads the appropriate
  SysConfig version for it. Must be called before any file-scoped tool.

  > **⚠️ See "Known limitation: first-time device-selection dialog"
  > above.** The very first time a given `.syscfg` file is opened, if
  > the project's device ID isn't a fully specific part number,
  > `openFile` **hangs indefinitely** waiting on a GUI modal dialog that
  > a pure MCP client cannot answer. Not a timing fluke — reproduced
  > identically across three separate attempts.

#### `closeFile`
*(available in both lists)*
- **Params:** none
- **Returns:** no output schema declared (empty result)
- **Description:** Closes the currently open file. File-scoped tools
  stop working until another file is opened.

#### `inspectExample`
*(available in both lists)*
- **Params:**
  | Field | Type | Required |
  |---|---|---|
  | `fileName` | string | yes — absolute path to an example `.syscfg` in the SDK |
  | `force` | boolean | no — force reading a file configured for a different device/board; without this, mismatched devices throw an error |
- **Returns:**
  | Field | Type | Notes |
  |---|---|---|
  | `configuration` | array (required) | module instances found in the example file |
  | `configuration[].moduleId` / `.moduleInstanceId` | string (required) | identity |
  | `configuration[].configuration` | array (required) | the instance's non-default settings |
  | `configuration[].commonSettingsAffectingAllInstances` | array (optional) | module-wide settings affecting every instance of that type |
- **Description:** Reads configuration from an SDK example `.syscfg` for
  guidance on complex configuration tasks. Only non-default values are
  returned; each setting includes `setByUser` (true = explicit user
  choice, false = automatic side effect).

### Module discovery

#### `listModules`
*(19-tool list only)*
- **Params:** none
- **Returns:**
  | Field | Type | Notes |
  |---|---|---|
  | `modules` | array (required) | |
  | `modules[].moduleId` | string (required) | |
  | `modules[].moduleName` | string (required) | |
  | `modules[].description` | string (optional) | |
- **Description:** Lists all module types available for the currently
  open file's device. Fixed for the session lifetime.

#### `getModuleDescription`
*(19-tool list only)*
- **Params:** `moduleIds` (array of strings, required)
- **Returns:**
  | Field | Type | Notes |
  |---|---|---|
  | `modules` | array (required) | |
  | `modules[].moduleId` | string (required) | |
  | `modules[].description` | string (required) | may be empty if none available |
- **Description:** Detailed description per requested module ID.

#### `getModuleInstances`
*(19-tool list only)*
- **Params:** `moduleIds` (array of strings, optional) — filter to specific modules
- **Returns:**
  | Field | Type | Notes |
  |---|---|---|
  | `instances` | array (required) | grouped by module type |
  | `instances[].moduleId` | string (required) | |
  | `instances[].instances` | array (required) | the actual instances of this module type |
  | `instances[].instances[].moduleInstanceId` | string (required) | |
  | `instances[].instances[].instanceName` | string (required) | |
  | `instances[].instances[].configurableCount` | number (required) | total configurables — for deciding whether to filter `getInstanceConfiguration` |
  | `instances[].instances[].parentInstances` | array of `{moduleId, moduleInstanceId}` (optional, **omitted if empty**) | owning instance(s); multiple = shared ownership |
  | `instances[].instances[].childInstances` | array of `{moduleId, moduleInstanceId}` (optional, **omitted if empty**) | owned instance(s) |
- **Description:** All module instances currently in the configuration,
  optionally filtered. This is the tool that actually reflects
  project state — `listModules` does not.
- **Live example** (from the `epwm_ex3_synchronization` project, after
  `openFile`):
  ```json
  {
    "instances": [
      {"moduleId": "/driverlib/epwm.js", "instances": [
        {"moduleInstanceId": "myEPWM1", "instanceName": "myEPWM1", "configurableCount": 168},
        {"moduleInstanceId": "myEPWM2", "instanceName": "myEPWM2", "configurableCount": 171},
        {"moduleInstanceId": "myEPWM3", "instanceName": "myEPWM3", "configurableCount": 171},
        {"moduleInstanceId": "myEPWM4", "instanceName": "myEPWM4", "configurableCount": 171}
      ]},
      {"moduleId": "/driverlib/sync.js", "instances": [
        {"moduleInstanceId": "$static", "instanceName": "staticInstance", "configurableCount": 5,
         "parentInstances": [{"moduleId": "/driverlib/epwm.js", "moduleInstanceId": "$static"}]}
      ]},
      {"moduleId": "/driverlib/inputxbar_input.js", "instances": [
        {"moduleInstanceId": "myINPUTXBARINPUT0", "instanceName": "myINPUTXBARINPUT0", "configurableCount": 5},
        {"moduleInstanceId": "myINPUTXBARINPUT1", "instanceName": "myINPUTXBARINPUT1", "configurableCount": 5}
      ]}
    ]
  }
  ```
  Note `parentInstances`/`childInstances` only appear on the `sync.js`
  instance (it has a real parent); the EPWM and INPUTXBARINPUT instances
  have neither field at all, since the schema says to omit them when
  empty rather than send empty arrays.

#### `getInstanceConfiguration`
*(19-tool list only)*
- **Params:**
  | Field | Type | Required | Notes |
  |---|---|---|---|
  | `instances` | array of `{moduleId, moduleInstanceId}` | yes | which instances to inspect |
  | `changesOnly` | boolean, default `false` | no | only modified configurables; adds `setByUser` per entry |
  | `ids` | array of strings | no | exact configurable IDs, e.g. `['baudRate','parity']` |
  | `searchText` | string | no | case-insensitive regex across names/descriptions/values/choices, e.g. `'clock\|frequency'` — searched even when descriptions/choices are excluded |
  | `includeDescriptions` | boolean, default `true` | no | omit to save tokens |
  | `includeChoices` | boolean, default `true` | no | omit to save tokens |
- **Returns:**
  | Field | Type | Notes |
  |---|---|---|
  | `configuration` | array (required) | |
  | `configuration[].moduleId` / `.moduleInstanceId` | string (required) | |
  | `configuration[].configuration` | array (required) | the settings themselves |
  | `configuration[].commonSettingsAffectingAllInstances` | array (optional) | module-wide settings shared across all instances of this type |
- **Description:** Detailed configuration for specific instances —
  configurables, pinmux, child modules, validation messages. The
  server's own guidance: call once with full details, then use
  `changesOnly: true` + no descriptions/choices for cheap re-checks.
  Token cost estimate given in the server's instructions: roughly 30
  tokens per configurable.

### Editing configuration

#### `addModuleInstances`
*(19-tool list only)*
- **Params:**
  | Field | Type | Required | Notes |
  |---|---|---|---|
  | `moduleIds` | array of strings | yes | repeat an ID to add multiple instances of the same module |
  | `maxConfigurables` | number, default `50` | no | ceiling for auto-including representative config; `0` disables it, `-1` always includes |
- **Returns:**
  | Field | Type | Notes |
  |---|---|---|
  | `addedInstances` | array (required) | every instance created, including auto-created dependencies |
  | `addedInstances[].moduleId` / `.moduleInstanceId` / `.instanceName` / `.configurableCount` | — | (all required) |
  | `addedInstances[].wasDirectlyAdded` | boolean (required) | true = explicitly requested, false = auto-created dependency |
  | `addedInstances[].parentInstances` | array (optional) | owning instance(s) |
  | `configurationByModuleType` | array (optional) | one entry per requested module type, with a lightweight representative config (`id`, `name`, `value`, `choices`, `readOnly` — no descriptions) for types under the `maxConfigurables` threshold |
- **Description:** Adds module instance(s), returning enough detail to
  avoid an immediate follow-up `getInstanceConfiguration` call for small
  modules.

#### `removeModuleInstances`
*(19-tool list only)*
- **Params:** `instances` (array of `{moduleId, moduleInstanceId}`, required)
- **Returns:**
  | Field | Type | Notes |
  |---|---|---|
  | `removedInstances` | array (required) | every instance removed, including side-effects |
  | `addedInstances` | array (optional, omitted if none) | instances created as a *reaction* to the removal |
- **Description:** Removes instance(s) and reports side-effects both ways.

#### `changeConfiguration`
*(19-tool list only)*
- **Params:**
  | Field | Type | Required | Notes |
  |---|---|---|---|
  | `changes` | array of `{moduleId, moduleInstanceId, configurable, value}` | yes | atomic — any failure reverts all |
  | `changes[].value` | string \| number \| boolean \| string[] \| number[] \| `{moduleId, moduleInstanceId}` | yes | must match the type from `getInstanceConfiguration` |
  | `description` | string | yes | shown in the UI undo history only — has no functional effect |
- **Returns:**
  | Field | Type | Notes |
  |---|---|---|
  | `changes` | array (optional, omitted if nothing changed) | per-instance delta |
  | `changes[].available` | array (optional) | configurables newly visible, full state |
  | `changes[].changed` | array (optional) | configurables that changed — sparse, only the fields that actually changed |
  | `changes[].unavailable` | array of strings (optional) | configurable IDs no longer visible |
  | `addedInstances` / `removedInstances` | array (optional, omitted if none) | module-level side-effects |
- **Description:** Atomic multi-configurable change with full delta
  reporting, including cascading module add/remove side-effects.

#### `getErrorsAndWarnings`
*(19-tool list only)*
- **Params:** none
- **Returns:**
  | Field | Type | Notes |
  |---|---|---|
  | `summary` | array (required) | |
  | `summary[].moduleId` / `.moduleInstanceId` | string (required) | |
  | `summary[].type` | `"error"` \| `"warning"` (required) | |
  | `summary[].message` | string (required) | |
  | `summary[].configurableId` | string (optional) | omitted for module-level (non-configurable-specific) messages |
- **Description:** All current validation errors/warnings, cheaper than
  `getInstanceConfiguration` when that's all you need.

#### `save`
*(19-tool list only)*
- **Params:** none
- **Returns:**
  | Field | Type | Notes |
  |---|---|---|
  | `savedFile` | string (required) | path of the saved `.syscfg` |
  | `generatedArtifacts` | array of strings (required) | paths of files written |
  | `skippedArtifacts` | array of strings (optional) | files not generated due to config errors |
  | `note` | string (optional) | present if generation wasn't possible/failed; the `.syscfg` itself still saves even if a generation error occurs |
- **Description:** Persists to `.syscfg` and regenerates all build
  artifacts.
- **Live example** (from the `epwm_ex3_synchronization` project):
  ```json
  {
    "savedFile": "/root/workspace_ccstheia/epwm_ex3_synchronization/epwm_ex3_synchronization.syscfg",
    "generatedArtifacts": [
      ".../CPU1_RAM/syscfg/migration.json", ".../board.c", ".../board.h",
      ".../board.cmd.genlibs", ".../board.opt", ".../board.json",
      ".../pinmux.csv", ".../epwm.dot", ".../c2000ware_libraries.cmd.genlibs",
      ".../c2000ware_libraries.opt", ".../c2000ware_libraries.c",
      ".../c2000ware_libraries.h", ".../clocktree.h"
    ]
  }
  ```

### Device migration

#### `listMigrationTargets`
*(19-tool list only)*
- **Params:** `includeCompatibility` (boolean, default `false`) — server
  warns this "can be slow for large device families i.e. 5 minutes" and
  suggests confirming with the user first
- **Returns:**
  | Field | Type | Notes |
  |---|---|---|
  | `targets` | array (required) | |
  | `targets[].device` / `.package` / `.variant` | string (required) | |
  | `targets[].properties` | object (optional) | e.g. price, package area, temp range, flash size — string or null values |
  | `targets[].peripherals` | object (optional) | peripheral counts, e.g. `{"Number of UARTs": 3}` |
  | `targets[].compatibility` | array (optional, only with `includeCompatibility: true`) | `status` (`compatible`\|`loading`\|`missing_functionality`\|`adjustments_needed`\|`no_pinmux`\|`error`\|`warning`\|`disabled`) + `issues` (string array) |
- **Description:** Valid migration targets for the currently open config.

#### `migrate`
*(19-tool list only)*
- **Params:** `device` (string, required), `package` (string, required),
  `variant` (string, optional — omit if the target has only one)
- **Returns:**
  | Field | Type | Notes |
  |---|---|---|
  | `migratedTo` | object `{device, package, variant}` (required) | |
  | `errorsAndWarnings` | array (required) | same shape as `getErrorsAndWarnings`, post-migration |
- **Description:** Performs the migration. Use `listMigrationTargets` first.

### Pin/connection graph

#### `getInstanceConnections`
*(23-tool list only — absent once this test file's `openFile` narrowed the list to 19)*
- **Params:**
  | Field | Type | Required | Notes |
  |---|---|---|---|
  | `instances` | array of `{moduleId, moduleInstanceId}` | no | omit for all instances with ≥1 port |
  | `includeConnectionTypes` | boolean, default `false` | no | fetch global type-compatibility rules once, omit on later calls |
- **Returns:**
  | Field | Type | Notes |
  |---|---|---|
  | `instances` | array (required) | |
  | `instances[].ports` | array | each port: `portId`, `portName`, `type`, `allowMultipleConnections`, `connections[]` (each with `moduleId`, `moduleInstanceId`, `portId`, `readOnly`) |
  | `connectionTypes` | array (optional, only with `includeConnectionTypes: true`) | `{start, end}` pairs — a connection from a `start`-type port to an `end`-type port is valid |
- **Description:** Ports and current connection state per instance.

#### `manageConnections`
*(23-tool list only)*
- **Params:**
  | Field | Type | Notes |
  |---|---|---|
  | `disconnect` | array of `{sourceModuleId, sourceInstanceId, sourcePortId, targetModuleId, targetInstanceId, targetPortId}` (optional) | connections to remove |
  | `connect` | array, same shape (optional) | connections to create; typically source=output port, target=input port |
- **Returns:** no output schema declared
- **Description:** Atomic connect/disconnect — cannot touch configurable
  values or add/remove instances.

#### `getInstanceLayout`
*(23-tool list only)*
- **Params:** `instances` (array of `{moduleId, moduleInstanceId}`, optional) — omit for all positioned instances
- **Returns:**
  | Field | Type | Notes |
  |---|---|---|
  | `instances[].position` | `[x, y]` (optional) | undefined if the instance has no position |
  | `instances[].size` | `[width, height]` (optional) | undefined if no size |
  | `instances[].memberContentsRestrictedTo` | `[width, height]` (optional) | groups only — bounds of the isolated coordinate space for group members |
- **Description:** Graphical-view positions/sizes. Groups have their own
  isolated coordinate space, separate from top-level instances.

#### `setInstancePositions`
*(23-tool list only)*
- **Params:** `positions` (array of `{moduleId, moduleInstanceId, position: [x,y]}`, required)
- **Returns:** no output schema declared
- **Description:** Sets graphical-view positions. Group members and
  top-level instances use separate coordinate spaces.

### Clock tree

#### `getClockTreeInstances`
*(19-tool list only)*
- **Params:**
  | Field | Type | Notes |
  |---|---|---|
  | `includeFilter` | string (regex, optional) | only names matching |
  | `excludeFilter` | string (regex, optional) | exclude names matching |
  | `instances` | array of strings (optional) | exact instance names only |
- **Returns:**
  | Field | Type | Notes |
  |---|---|---|
  | `instances[].name` / `.moduleId` / `.moduleInstanceId` | string (required) | |
  | `instances[].type` | string (required) | `Divider`, `Mux`, `Multiplier`, `Oscillator`, `Gate`, or `Adder` |
  | `instances[].inPins[]` | — | `name` + `connectedFrom {instanceName, pinName}` (all input pins are connected — required) |
  | `instances[].outPins[]` | — | `name` + `connectedTo[] {instanceName, pinName}` (non-empty — every output feeds at least one input) |
- **Description:** Clock tree topology. Instances/connections are fixed
  hardware — only configurables on them are adjustable (via
  `changeConfiguration`).

#### `traceClockSignal`
*(19-tool list only)*
- **Params:** `toInstance` (string, required), `toPin` (string, required — input or output pin)
- **Returns:**
  | Field | Type | Notes |
  |---|---|---|
  | `signalPath` | array (required) | ordered source-last; first element is the queried pin |
  | `signalPath[].instance` / `.pin` / `.type` / `.moduleId` / `.moduleInstanceId` / `.frequency` | — | all required; `type` is one of `Oscillator`/`Divider`/`Mux`/`Multiplier`/`Gate`/`Adder`; `frequency` is a display string like `"48 MHz"` or a range `"32 - 48 MHz"` |
- **Description:** Backward trace from a pin to its source, showing the
  frequency at each hop. Same instance appearing at different
  frequencies on different pins = a transformation happened inside it.

### Generated output

#### `listGeneratedArtifacts`
*(19-tool list only)*
- **Params:** none
- **Returns:**
  | Field | Type | Notes |
  |---|---|---|
  | `artifacts` | array (required) | |
  | `artifacts[].filePath` | string (required) | |
  | `artifacts[].blockedByErrors` | boolean (optional) | true if it would be generated but config errors block it |
  | `note` | string (optional) | present if generation wasn't possible, e.g. no `--output` specified |
- **Description:** What files the current configuration will produce.

#### `readGeneratedArtifact`
*(19-tool list only)*
- **Params:** `filePath` (string, required — from `listGeneratedArtifacts`)
- **Returns:** `content` (string, required) — the file's current contents
- **Description:** Reads one generated artifact.

---

## Verified live workflow (this session)

The following sequence was run end-to-end against a real CCS instance
and the `epwm_ex3_synchronization` project (`driverlib/f280013x/examples/
epwm/CCS/epwm_ex3_synchronization.projectspec`, imported and built via
the `project` MCP server first — see `ccs_headless_gui_and_mcp_walkthrough.md`):

1. `openFile` on `epwm_ex3_synchronization.syscfg` — hung on first-ever
   attempt (device-selection dialog, see above); after resolving that
   once via the GUI, subsequent `openFile` calls for this file completed
   in ~1.6–1.7 seconds from brand-new connections.
2. `getModuleInstances` (no filter) — returned the real instance graph:
   4 EPWM instances (168–171 configurables each), 1 SYNC static instance
   (5 configurables, parented to EPWM), 2 INPUTXBARINPUT instances (5
   configurables each).
3. `listModules` — returned all 47 module types available for this
   device (`F280013x`), spanning driverlib peripherals, board components
   (LED/SWITCH), utilities (DCSM, CMD), and libraries (FATFS, HRPWM SFO,
   DCL control blocks, IQmath, FFT/Filter/Vector, and BETA data-transfer
   tools).
4. `save` — persisted the (unmodified) configuration and regenerated all
   13 build artifacts successfully.

---

## How this was captured

Two things were queried directly against a live, running CCS instance
using a persistent Node.js child-process client (see
`ccs_headless_gui_and_mcp_walkthrough.md` §7.2 for the pattern):

1. `tools/list` **before** ever calling `openFile` — done twice,
   independently, both returning the same 23 tools with full
   `inputSchema`/`outputSchema` for each.
2. `tools/list` **after** a successful `openFile` for the
   `epwm_ex3_synchronization.syscfg` file — returning 19 tools (the same
   23 minus the 4 pin/layout tools listed above), again with full
   schemas.

Every `inputSchema`/`outputSchema` field documented above is copied
directly from those two captures, not inferred from example responses.
Live example payloads (marked "Live example" above) are additionally
copied verbatim from real `tools/call` responses made during this
session.
