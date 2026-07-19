---
name: epwm-to-mcpwm-sysconfig-migration
description: >
  Orchestrates migrating an EPWM-based SysConfig (.syscfg) project to a
  target device's MCPWM peripheral via the live SysConfig MCP server, in
  confirmed phases: (1) read-only analysis of the source EPWM sync chain
  and time-base settings, producing a proposed EPWM->MCPWM instance
  grouping report; (2) creating the target MCPWM instances and setting up
  their PWM mapping and synchronization (time-base, sync-in/sync-out,
  phase) to match the confirmed grouping; (3) migrating the rest of each
  instance's configuration across seven confirmed sub-phases -- counter-
  compare, action-qualifier, dead-band, trip-zone, event-trigger,
  global-load, and a removed-submodule audit (chopper, digital compare,
  HRPWM, extended compare, ICL, diode emulation, min dead-band). Use this
  whenever the user wants to migrate, port, or consolidate an existing
  EPWM-based project to
  a device with MCPWM, wants to know whether several EPWM instances can
  share one MCPWM instance, wants the sync chain or time-base settings of
  an EPWM project checked before a migration, or asks to set up / create /
  apply the MCPWM modules for a migration. Trigger even if the user doesn't
  say "MCPWM" explicitly but describes moving a multi-EPWM design to a
  lower-pin-count or motor-control-oriented device. This is the top-level
  entry point -- always consult it first rather than a phase file directly,
  so the phase gating and confirmation gates are respected.
---

# EPWM -> MCPWM SysConfig Migration (Orchestrator)

This skill coordinates a multi-phase migration of an EPWM-based `.syscfg`
project onto a target device's MCPWM peripheral, using the live SysConfig
MCP server. It is the entry point -- the actual step-by-step procedures
live in the `references/` files below, loaded one phase at a time.

## The phases

1. **Analysis & migration report** -- read-only. Reads the source EPWM
   `.syscfg` file, reconstructs its sync chain and time-base settings, and
   produces a report proposing which EPWM instances group onto which
   target MCPWM instance. Never touches the target file.
   -> `references/phase1-migration-report.md`

2. **Instance setup: PWM mapping + synchronization** -- writes the target
   file. Only runs once the user has confirmed phase 1's grouping. Creates
   one MCPWM instance per confirmed group, sets each instance's shared
   time-base fields, and translates the EPWM sync chain onto the new
   instances. Does **not** touch action-qualifier, counter-compare,
   dead-band, trip-zone, or event-trigger settings.
   -> `references/phase2-instance-setup.md`

3. **Full configuration migration** -- split into seven independent
   sub-phases, each its own reference file with its own stop-and-confirm
   gate, run in this order:

   | Sub-phase | Covers | Writes? |
   |---|---|---|
   | 3a | Counter-Compare | yes |
   | 3b | Action-Qualifier | yes |
   | 3c | Dead-Band | yes |
   | 3d | Trip-Zone | yes |
   | 3e | Event-Trigger | yes |
   | 3f | Global-Load | yes |
   | 3g | Removed-submodule audit (Chopper, Digital Compare, HRPWM, Extended Compare, ICL, Diode Emulation, Min Dead-Band) | no -- read-only report |

   3a-3f each rely on an MCP tool, `get_syscfg_module_migration_guide`
   (args: source device, target device, `module_to_module`, and an array of
   SysConfig configurable ids -- returns markdown migration guidance per
   id), to get the field-level mapping instead of any locally-cached data.
   That tool is not guaranteed to be present in every environment -- if
   it's missing, the sub-phase says so and stops rather than improvising.
   Every sub-phase also has to translate the tool's naive one-EPWM-to-one-
   MCPWM-pair assumption onto the real confirmed grouping (pair
   substitution for per-pair fields, explicit reconciliation for
   instance-wide-shared fields) -- see each sub-phase file for which kind
   applies and why.

Each phase (and each phase-3 sub-phase) ends with an explicit stop-and-
confirm step. Honor it: end your turn, present the report, and wait for
the user before loading the next reference file. Don't cascade through
phases or sub-phases in one turn even when the next step looks obvious --
the confirmation gates exist because each one makes judgment calls (how to
group instances, which sync relationship wins a single slot, which source
instance's value wins a shared field, etc.) that are the user's to approve
or override.

## How to use this orchestrator

- Figure out which phase applies from the conversation so far: has a
  phase-1 report already been produced and confirmed? Have MCPWM instances
  already been created? If you're not sure, ask rather than assume.
- **Read the relevant `references/*.md` file with the Read tool before
  doing any of that phase's work.** This orchestrator file only summarizes
  what each phase does -- the concrete tool schemas, exact configurable
  ids, live-verified quirks, and table/report formats live in the
  reference files, not here.
- Never skip straight to phase 2 (or a hypothetical phase 3) without the
  confirmed output of the phase before it in hand. If the user asks to
  jump ahead, either load the earlier phase first or explicitly ask them
  for the specific confirmed inputs that phase would have produced --
  don't reconstruct or guess a grouping/mapping that was never confirmed.

## Inputs (gathered once, used across phases)

1. **Source device** -- the device the source `.syscfg` file targets.
2. **Source `.syscfg` file** -- absolute path.
3. **Target device** -- an MCPWM-capable device to migrate toward.
4. **Target `.syscfg` file** -- absolute path; needed starting phase 2 (may
   be the same file phase 1's capacity check probed, or a fresh one).

If any of these are missing when a phase needs them, ask -- don't guess a
device or file path.

## Reference files

- `references/phase1-migration-report.md` -- full phase-1 procedure (8
  steps): open source file, enumerate EPWM instances, pull time-base/sync
  settings, check device-level sync routing, reconstruct the sync chain,
  check target MCPWM capacity, produce the 5-section report, stop and
  confirm.
- `references/phase2-instance-setup.md` -- full phase-2 procedure (7
  steps): open target file, create MCPWM instances per confirmed group,
  set shared time-base fields, translate the sync chain onto the new
  instances, verify, save and report, stop and confirm before phase 3.
- `references/phase3a-counter-compare.md` -- CMPA-CMPD migration; pair
  substitution for CMPA/CMPB, reconciliation for shared CMPC/CMPD.
- `references/phase3b-action-qualifier.md` -- per-output action-table
  migration; pair substitution, T1/T2 fields always `no_equivalent`.
- `references/phase3c-dead-band.md` -- instance-wide, reconciliation only.
- `references/phase3d-trip-zone.md` -- instance-wide reconciliation, plus
  re-routing trip sources through PWM X-BAR (not a plain rename).
- `references/phase3e-event-trigger.md` -- instance-wide reconciliation,
  plus the one structural (`partial`) field, `epwmEventTrigger_interruptSource`.
- `references/phase3f-global-load.md` -- instance-wide reconciliation,
  smallest and simplest field set.
- `references/phase3g-removed-submodules-audit.md` -- read-only; reports
  whether the source project actually used any of the seven fully-removed
  feature areas, rather than silently dropping them.

Each phase-3 sub-phase file starts with its own complete, exact SysConfig
id checklist (551 total ids across the whole migration, verified to
partition with no gaps and no overlaps across phases 1-3g) before its
step-by-step procedure.

## Reference material in this repository (shared across phases 1-2)

- `syscfg_mcp.md` -- full tool schemas and known limitations for the
  SysConfig MCP server, including the `openFile` device-dialog hang and
  `changeConfiguration`'s atomic-per-call semantics.
- `epwm_to_mcpwm_migration.md` -- submodule-by-submodule EPWM/MCPWM
  differences and driverlib renames, distilled from TI's SPRADL7
  application note.

Phase 3's sub-phases get their field-level migration guidance from the
live `get_syscfg_module_migration_guide` MCP tool at runtime instead of any
file in this repository -- see each `phase3*.md` file for how it's called.
