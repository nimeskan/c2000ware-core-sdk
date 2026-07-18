---
name: epwm-mcpwm-instance-setup
description: >
  Second phase of an EPWM-to-MCPWM migration: given an already-confirmed
  grouping/mapping report (from the epwm-mcpwm-sync-migration-report
  skill), creates the actual MCPWM module instances on a target device's
  .syscfg file and configures ONLY their time-base and synchronization
  settings (period, clock divider, counter mode, sync-in/sync-out, phase)
  to match the confirmed grouping. Does NOT touch action-qualifier,
  counter-compare, dead-band, trip-zone, or event-trigger configuration --
  that is a separate, later phase. Use this only after a user has reviewed
  and explicitly confirmed a migration grouping report; never run it from
  a report the user hasn't signed off on. Trigger when the user says
  something like "now set up the MCPWM modules", "apply the mapping we
  confirmed", "create the target instances", or otherwise asks to move
  from the migration report into actually building the target
  configuration's PWM mapping and synchronization.
---

# EPWM -> MCPWM Instance Setup (PWM Mapping + Synchronization)

## Where this fits

This is phase 2 of a three-phase workflow:

1. **Analysis** (`epwm-mcpwm-sync-migration-report` skill) -- read-only,
   produces a grouping proposal and a per-instance settings table. Ends
   with the user confirming or adjusting the grouping.
2. **This skill** -- creates the target MCPWM instances and sets up their
   PWM mapping (which source EPWM instances map to which target MCPWM
   instance, in what pair role) and synchronization (time-base match plus
   the sync-in/sync-out/phase chain between instances). Ends with the user
   confirming the applied configuration.
3. **Not built yet** -- migrating the rest of each EPWM instance's
   configuration (action-qualifier, counter-compare, dead-band,
   trip-zone, event-trigger) onto the corresponding MCPWM pairs. Out of
   scope here on purpose -- don't start on it, even opportunistically.

Do not run this skill against a grouping the user hasn't actually
confirmed. If you arrive here without a confirmed grouping in hand (e.g.
the user skipped straight to "set up MCPWM"), run the phase-1 skill first.

## Inputs

From the confirmed phase-1 report, you need:

1. **The groups** -- which source EPWM `moduleInstanceId`s are assigned to
   the same target MCPWM instance (at most 3 per group, one EPWM per pair
   slot), and in what order the user confirmed them.
2. **Each group's shared time-base values** -- period, clock divider,
   counter mode (these must already be identical across a group's members,
   or the grouping wasn't valid).
3. **The sync chain** -- which EPWM instance was the original sync root,
   and for every other instance, which upstream instance's sync-out fed it
   and with what phase-shift value.
4. **Target device** and **target `.syscfg` file** (absolute path) -- the
   file phase 1's target-capacity check (Step 6) was run against, or a
   fresh one the user points you to.

If any of these weren't part of the confirmed report, ask before
proceeding rather than re-deriving or guessing them.

## Step-by-step procedure

### Step 1 -- Open the target file

Call `openFile` with the target `.syscfg` file's absolute path. Same
`@v2CliArgs` header caveat as phase 1 applies -- `grep` for it first; its
absence means the first `openFile` on this specific file can hang on a
GUI device-selection dialog.

### Step 2 -- Create one MCPWM instance per confirmed group

Call `addModuleInstances` with `moduleIds` containing `"/driverlib/mcpwm.js"`
repeated once per group (e.g. two groups -> pass it twice in the array),
and `maxConfigurables: 0` -- you don't need the automatic representative
config, you're about to set specific fields yourself in Step 4.

Record the returned `addedInstances[].moduleInstanceId` values **in the
order returned**, and map them to the confirmed groups **in the same order
the user confirmed the groups** (group 1 -> first returned instance id,
group 2 -> second, ...). `addModuleInstances` auto-assigns names
(`myMCPWM0`, `myMCPWM1`, ...) -- you don't get to choose them up front.
State this group -> instance-name mapping explicitly in the Step 6 report;
it's the concrete answer to "which EPWM instances map to which MCPWM
instance."

### Step 3 -- Set each instance's shared time-base fields

For each group's MCPWM instance, call `changeConfiguration` **once**
(batch that instance's fields into one call so they apply atomically; use
a separate call per instance so one instance's failure doesn't revert
another's). Set, from that group's confirmed shared values:

| Target field (`/driverlib/mcpwm.js`) | Source field (`/driverlib/epwm.js`) | Notes |
|---|---|---|
| `mcpwmTimebase_period` | `epwmTimebase_period` | direct copy |
| `mcpwmTimebase_clockDiv` | `epwmTimebase_clockDiv` | direct copy -- MCPWM has no separate high-speed divider stage, so if the source used a non-default `epwmTimebase_hsClockDiv` alongside `epwmTimebase_clockDiv`, note that the combined effective divide ratio may not be exactly reproducible by `mcpwmTimebase_clockDiv` alone (flag it, don't silently drop it) |
| `mcpwmTimebase_counterMode` | `epwmTimebase_counterMode` | direct copy -- MCPWM has no down-count-only mode; if the source was down-count-only, this is a real gap to flag, not a silent substitution |
| `mcpwmTimebase_periodLoadMode` | `epwmTimebase_periodLoadMode` | MCPWM only has two choices (shadow-load-at-TBCTR=0, or disabled) versus EPWM's richer `periodLoadMode` + `periodLoadEvent` pair -- if the source's `epwmTimebase_periodLoadEvent` was anything other than "reaches 0" (e.g. "only on SYNC"), that nuance has **no MCPWM equivalent** (confirmed in `epwm_mcpwm_syscfg_migration.json`) and is lost; flag it |
| `mcpwmTimebase_forceSyncPulse` | `epwmTimebase_forceSyncPulse` | direct copy |

If `mcpwmTimebase_counterMode` is being set to `"Up - down - count mode"`,
expect `mcpwmTimebase_counterModeAfterSync` to become newly visible
(SysConfig conditional-visibility, same mechanism as
`epwmTimebase_phaseShift` in phase 1 -- confirm via the `available` array
in this call's response rather than assuming). This field has **no EPWM
source** -- EPWM never had it -- so don't default it silently; ask the
user which direction (count up or down after sync) is intended if it
wasn't already covered by the confirmed report, and set it in a follow-up
`changeConfiguration` call once you know.

`epwmTimebase_counterValue`, `epwmTimebase_periodLink`, and
`epwmTimebase_periodGld` have no MCPWM counterpart at all (confirmed in
`epwm_mcpwm_syscfg_migration.json`) -- don't attempt to set anything for
them, just note in the Step 6 report that they were dropped if the source
used non-default values for any of them.

### Step 4 -- Translate the sync chain onto the new instances

This is the actual "synchronization setup," and it is a translation, not a
copy -- the sync relationships in the source file are named in terms of
EPWM instances, but the target's `mcpwmTimebase_syncInPulseSource` choices
are named in terms of MCPWM instances (and other peripherals), so every
reference has to be re-resolved through the group -> instance mapping from
Step 2:

1. **Find the new root.** The group containing the original sync root EPWM
   instance is the new root group; its MCPWM instance doesn't need a
   `syncInPulseSource` change (leave it be). Set its
   `mcpwmTimebase_syncOutPulseMode` to match the original root's
   `epwmTimebase_syncOutPulseMode`. **Cardinality note:** EPWM's
   `syncOutPulseMode` is multi-select (an array -- e.g. it could combine
   "counter zero" and "compare C" simultaneously); MCPWM's is single-select
   (confirmed live -- one string value, no "enable all of the above"
   choice). If the source array has more than one entry, only one can
   survive -- pick the one that's actually load-bearing for the
   cross-group sync relationships found in step 2 below, and explicitly
   name whichever entries got dropped rather than silently discarding them.

2. **For every other group**, check whether any of its member EPWM
   instances had a `syncInPulseSource` pointing at an instance that ended
   up in a *different* group. If so:
   - Call `getInstanceConfiguration` on that group's new MCPWM instance
     with `ids: ["mcpwmTimebase_syncInPulseSource"]` and
     `includeChoices: true` first, to read the actual available choice
     strings (they're phrased like `"Sync-in source is MCPWM1 sync-out
     signal"` -- confirm the exact instance number/wording rather than
     constructing the string yourself, since the choice list is generated
     per-device and per-instance).
   - Set that group's `mcpwmTimebase_syncInPulseSource` to the choice
     naming the upstream group's new MCPWM instance.
   - Set that group's `mcpwmTimebase_phaseEnable` / `mcpwmTimebase_phaseShift`
     to the *specific member EPWM instance's* original
     `epwmTimebase_phaseEnable` / `epwmTimebase_phaseShift` values -- this
     is the one EPWM instance in the group whose original phase relationship
     is being carried forward at the instance level.

3. **For every EPWM instance whose sync relationship was to another
   instance in the *same* group**, do not set anything at the instance
   level for it. Name it explicitly in the Step 6 report as deferred to
   counter-compare configuration in phase 3 -- a shared counter can carry
   exactly one instance-level phase relationship (to whatever's upstream
   of the whole group), not a separate one per original EPWM instance, so
   intra-group phase differences have to be reproduced some other way
   later, not here.

4. **If a single group would need more than one upstream sync-in
   relationship** (e.g. two of its member EPWM instances each synced from
   a different EPWM that's now in two different other groups), stop and
   surface this to the user rather than picking one -- `mcpwmTimebase_syncInPulseSource`
   only has one slot, and choosing which relationship wins is a design
   decision, not something to resolve unilaterally.

### Step 5 -- Verify

Call `getErrorsAndWarnings`. Confirm it comes back clean before presenting
anything as done -- if the new instances or changes introduced any error
or warning, resolve or report it before moving to Step 6.

### Step 6 -- Save and present the result for confirmation

Call `save`. Then present a report with this structure:

1. **Group -> MCPWM instance mapping** -- a table: confirmed group number,
   its member source EPWM instances, the new MCPWM `moduleInstanceId`
   SysConfig assigned it.
2. **Applied time-base/sync settings per instance** -- pull this via
   `getInstanceConfiguration` with `changesOnly: true` on the new MCPWM
   instances so the report reflects what's actually in the file, not just
   what you intended to set.
3. **Explicitly flagged gaps and decisions**, gathered from Steps 3-4:
   dropped EPWM-only fields (`hsClockDiv` nuance, `periodLoadEvent`,
   `periodLink`, `periodGld`, `counterValue`, `oneShotSyncOutTrigger`),
   any `syncOutPulseMode` entries that didn't survive the array->single
   narrowing, the new-to-MCPWM `counterModeAfterSync` decision if
   applicable, and the full list of intra-group phase relationships
   deferred to phase 3 (name each EPWM instance and its original phase
   value).
4. **Verification result** -- confirm `getErrorsAndWarnings` was clean and
   the file was saved.

### Step 7 -- Stop and confirm before phase 3

**End your turn after presenting the Step 6 report.** Do not start on
action-qualifier, counter-compare, dead-band, trip-zone, or event-trigger
configuration in the same turn, even though it's the obvious next
question -- that phase hasn't been designed yet and needs its own pass.
Ask the user to review the mapping and, in particular, the flagged
intra-group phase deferrals and any dropped fields from section 3, since
those are exactly the places a judgment call was made that the user may
want to revisit before the target instances get any further configuration
built on top of them.

## Reference material in this repository

- `syscfg_mcp.md` -- full tool schemas, including `changeConfiguration`'s
  atomic-per-call semantics and its acceptance of driverlib constant names
  for choice-type configurables.
- `epwm_to_mcpwm_migration.md` -- submodule-by-submodule EPWM/MCPWM
  differences distilled from TI's SPRADL7 application note.
- `epwm_mcpwm_syscfg_migration.json` -- the authoritative per-configurable
  mapped/no_equivalent/partial status data referenced throughout Step 3.
