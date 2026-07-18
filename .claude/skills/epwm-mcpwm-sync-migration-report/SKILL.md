---
name: epwm-mcpwm-sync-migration-report
description: >
  Analyzes an EPWM-based SysConfig (.syscfg) file to determine how its EPWM
  instances should map onto a target device's MCPWM peripheral, and produces
  a settings report table -- it does NOT write or modify any .syscfg file.
  Use this whenever the user wants to migrate, port, or consolidate an
  existing EPWM-based project to a device with MCPWM, wants to know whether
  several EPWM instances can share one MCPWM instance, wants the sync chain
  (SyncIn/SyncOut, phase shift) or time-base settings of an EPWM project
  checked before a migration, or asks for a mapping/compatibility report
  between EPWM and MCPWM configuration. Trigger even if the user doesn't say
  "MCPWM" explicitly but describes moving a multi-EPWM design to a
  lower-pin-count or motor-control-oriented device. Always produce the
  analysis report as the deliverable -- do not proceed to actually create or
  edit any MCPWM .syscfg configuration as part of this skill.
---

# EPWM -> MCPWM Sync/Time-base Migration Report

## What this skill does, and doesn't, do

This skill inspects a source EPWM-based `.syscfg` file, works out how its
EPWM instances are wired together (the sync chain), and reports which EPWM
instances *could* be consolidated onto a single MCPWM instance on a target
device, and what their combined time-base/sync settings would need to be.

It stops at the report. It does not open, add, or edit any MCPWM module
instance, and it does not write any `.syscfg` file. That is deliberate --
treat "propose the equivalent MCPWM settings as configuration changes" as
out of scope for this skill even if it would be easy to continue further.

**Why the sync chain matters here:** a single MCPWM instance shares *one*
time-base counter (`TBCTR`) across all of its PWM output pairs. Two EPWM
instances can only become two pairs of the *same* MCPWM instance if they
already run off compatible time bases -- same period, same clock divider,
same counter mode -- and any relative phase between them has to be
achievable through the counter-compare submodule instead of two independent
counters. EPWM instances that are already synchronized to each other via
SyncIn/SyncOut are exactly the candidates worth checking first, since that
sync relationship is what a shared MCPWM counter is standing in for.

## Inputs

This skill needs three things, gathered from the user before starting:

1. **Source device** -- the device the source `.syscfg` file targets.
2. **Source `.syscfg` file** -- absolute path to the file to analyze.
3. **Target device** -- an MCPWM-capable device to migrate toward.

If any of these are missing, ask before proceeding -- don't guess a device
or file path.

## Tools used

All steps below use the SysConfig MCP server's tools. If you haven't
inspected this server's tools yet in the current session, expect roughly
this set: `openFile`, `getModuleInstances`, `getInstanceConfiguration`,
`addModuleInstances`, `removeModuleInstances`, `listFiles`, `closeFile`.
Call `tools/list` (or just try `openFile`) if you're unsure of exact
parameter names -- the ones below reflect a live-verified server, but
confirm before trusting field names blindly on a different server version.

## Step-by-step procedure

### Step 1 -- Open the source file

Call `openFile` with the source `.syscfg` file's absolute path.

Before doing this, `grep` the file for `@v2CliArgs` in its header comment:

```
grep "@v2CliArgs" <source .syscfg path>
```

If that header is **absent**, the *first* time this specific file is opened
via the MCP server, `openFile` can hang indefinitely on a GUI
device-selection dialog the MCP client has no way to answer (a known,
reproducible limitation -- not a timing fluke). If you hit this, it needs a
GUI-side click to resolve (see the project's CCS/MCP walkthrough doc for
the exact mechanism); don't just keep re-sending `openFile` with a longer
timeout expecting it to resolve itself.

### Step 2 -- Enumerate the EPWM instances

Call `getModuleInstances` filtered to the EPWM module (and the shared sync
module, if the server's driverlib metadata exposes device-level sync
routing as its own module rather than folding it into each EPWM instance).
Record every EPWM instance's `moduleInstanceId`, and note any
`parentInstances`/`childInstances` relationships reported -- a shared
static sync-routing instance parented under all the EPWM instances is a
sign the device does its cross-peripheral sync-source selection at the
device level, not per-EPWM-instance, and Step 4 needs to check that
separately from each EPWM instance's own settings.

### Step 3 -- Pull time-base and sync settings for every EPWM instance

Call `getInstanceConfiguration` **once**, passing all EPWM instances found
in Step 2 in the `instances` array, filtered with `ids` down to just the
time-base and sync-relevant configurables so the response stays small.
Based on a live-verified EPWM instance, that configurable set includes (id
prefixes will match, but confirm exact spelling against the live
`getInstanceConfiguration` response for the actual server/device rather
than assuming these are complete):

- Time-base: period, clock divider(s), counter mode, counter initial value,
  and how the period register loads (direct vs. shadow, and on what event)
- Sync: sync-in pulse source, sync-out pulse mode, phase-shift enable,
  force-sync-pulse, one-shot-sync-out trigger

Do this as a single call across all instances rather than one call per
instance -- it's cheaper and makes the values directly comparable.

### Step 4 -- Check device-level sync routing, if applicable

If Step 2 revealed a separate shared sync-routing instance (rather than
each EPWM instance selecting its own sync-in source independently), call
`getInstanceConfiguration` on that instance with `searchText: "sync"`
instead of a hardcoded `ids` list -- the exact configurable names for
device-level sync crossbar routing vary by device family (how many
peripherals can be a sync source/destination differs device to device), so
searching is more robust than assuming fixed IDs.

### Step 5 -- Reconstruct the sync chain

From the values gathered in Steps 3-4, build the sync topology by hand:

- Which EPWM instance is the **sync root** -- i.e., its sync-in source is
  external or otherwise not another EPWM's sync-out (nothing upstream of
  it in the chain)?
- For every other instance, which upstream instance's sync-out feeds its
  sync-in, and in what mode (pulse-on-zero, pulse-on-period, one-shot)?
- For each instance with phase-shift load enabled, is the effective phase
  offset zero (meaning it's really just following the root's counter with
  no offset) or non-zero (a genuine phase relationship that would need to
  be re-expressed via counter-compare on a shared MCPWM counter, since
  MCPWM has one counter per instance, not one per pair)?
- Are there EPWM instances with **no** sync relationship to the others at
  all (independent period/clock, no sync-in from a sibling)? These are not
  consolidation candidates with the rest and should be flagged separately.

### Step 6 -- Determine target-device MCPWM capacity

Each MCPWM instance always exposes exactly three PWM output pairs
(confirmed via the fixed `1A/1B`, `2A/2B`, `3A/3B` naming baked into every
MCPWM instance's action-qualifier configurables on a live-inspected
instance) -- there's no separate "number of pairs" setting to query. What
does vary by device is how many separate MCPWM instances the target device
has. To determine that for the target device:

- If a `.syscfg` file already targeting the target device is open (or can
  be opened) in the workspace, use `getModuleInstances` filtered to the
  MCPWM module to see what's already present, and `addModuleInstances`
  (repeating the MCPWM module ID) to discover how many instances the
  device supports -- it will fail once you exceed the hardware count.
- If no such file exists yet, note this as a gap in the report rather than
  guessing a pair/instance count -- the target device's data sheet or TRM
  is the authoritative source, and this skill's job is the EPWM-side
  analysis, not confirming target hardware limits from scratch.

### Step 7 -- Produce the report

Output a single settings table, one row per **source EPWM instance**, plus
a short prose summary above it. Do not include any guidance on the
mechanics of creating the target `.syscfg` file, calling
`addModuleInstances`/`changeConfiguration` for MCPWM, or naming target
instances -- that is explicitly out of scope for this skill's output.

**Table columns:**

| Column | Content |
|---|---|
| EPWM instance | `moduleInstanceId` from Step 2 |
| Period | value from Step 3 |
| Clock divider(s) | value(s) from Step 3 |
| Counter mode | value from Step 3 |
| Period load mode | value from Step 3 |
| Sync-in source | value from Step 3 (resolved to which instance/external signal it names) |
| Sync-out mode | value from Step 3 |
| Phase-shift enable / value | value from Step 3, and whether the effective offset is zero or non-zero |
| One-shot sync-out trigger | value from Step 3 |
| Sync role | Root / Synced (from `<X>`) / Independent, per Step 5 |
| Consolidation candidate? | Yes, with which sibling instances, if period + clock divider + counter mode all match and it's part of the same sync chain; No, with the specific mismatching field(s) named, otherwise |

**Prose summary above the table** should state in plain terms: how many
independent time-base groups exist among the source EPWM instances, how
many MCPWM instances (at 3 pairs each) that would require on the target
device, whether Step 6 confirmed the target device actually has that many,
and which EPWM instances (if any) cannot be consolidated with any other and
why.

## Reference material in this repository

- `syscfg_mcp.md` -- full tool schemas and known limitations for the
  SysConfig MCP server, including the `openFile` device-dialog hang
  referenced in Step 1.
- `epwm_to_mcpwm_migration.md` -- submodule-by-submodule EPWM/MCPWM
  differences and driverlib renames, distilled from TI's SPRADL7
  application note.
- `epwm_mcpwm_syscfg_migration.json` -- per-configurable EPWM -> MCPWM
  mapping data (status: mapped / no_equivalent / partial) for every
  SysConfig configurable, useful context once someone moves past this
  report into actually building the target configuration.
