# User Guide: `epwm_mcpwm_syscfg_migration.json`

This is a guide for anyone using `epwm_mcpwm_syscfg_migration.json` (in this
same directory) to help migrate an EPWM-based SysConfig project to MCPWM.
Read this before treating the file as an authoritative, drop-in mapping --
it's a useful reference, but it has one significant blind spot that will
silently produce wrong results if you don't account for it.

## What this file is

It's a flat lookup table, keyed by **EPWM SysConfig configurable ID**
(e.g. `epwmTimebase_period`), describing whether that configurable has an
equivalent on the MCPWM module, and if so, what it's called there.

**The important thing to understand about its scope:** this mapping is
built from the SysConfig **module definitions** for `epwm.js` and
`mcpwm.js` -- it is a field-name-to-field-name dictionary, not something
generated from any specific project. It has no concept of "project
instance" at all -- it doesn't know what `myEPWM1` or `myEPWM2` are, how
many EPWM instances a real project has, or how those instances get grouped
onto MCPWM instances during a real migration. Keep that in mind for the
whole rest of this guide -- it's the source of the one real limitation
below.

## Schema

Each top-level key is an EPWM configurable ID. Its value is an object with:

| Field | Meaning |
|---|---|
| `from_type` | The EPWM configurable's SysConfig field type (`number`, `select`, `boolean`, etc.) |
| `from_displayName` | The EPWM configurable's UI label |
| `from_description` | The EPWM configurable's UI description |
| `from_default` | The EPWM configurable's default value |
| `devices` | Which EPWM device families expose this configurable (~12 families tracked) |
| `status` | `mapped`, `no_equivalent`, or `partial` -- see below |
| `to_config` | The MCPWM configurable ID this maps to, or `null` if `no_equivalent` |
| `to_type` / `to_displayName` | The MCPWM configurable's type/label, when `to_config` is set |
| `fixMsg` | Free-text note -- often a submodule-level summary shared across several related entries, not always specific to the one field |

`partial`-status entries (there is currently exactly one -- see below) add
two more fields:

| Field | Meaning |
|---|---|
| `mapping_type` | Always `"structural"` where present -- signals the relationship isn't a simple field rename |
| `option_map` | Per-enum-value breakdown: for each of the EPWM field's choice values, what it maps to on the MCPWM side, which devices support that specific choice, and both sides' display names |

## Status breakdown (551 entries total)

| Status | Count | What it means |
|---|---|---|
| `mapped` | 88 | Direct equivalent exists on MCPWM under `to_config` |
| `no_equivalent` | 462 | Nothing on MCPWM corresponds to this EPWM configurable -- the feature was removed or restructured away |
| `partial` | 1 | A structural change, not a rename -- see `option_map` instead of `to_config` |

Of the 462 `no_equivalent` entries, the largest groups by submodule are:

| Submodule | Count |
|---|---|
| `XCMP*` (extended compare -- a submodule some newer EPWM device families have, no MCPWM counterpart at all) | 48 |
| Digital Compare | 60 |
| ICL (input control logic) | 24 |
| Action-qualifier (the subset with no equivalent, e.g. T1/T2 events) | 22 |
| Trip-zone (the subset with no equivalent, e.g. digital-compare-based trips, advanced trip actions) | 22 |
| Diode emulation | 14 |
| Event-trigger (the subset with no equivalent, e.g. custom prescaler init) | 13 |
| Min dead-band | 12 |
| Counter-compare (the subset with no equivalent, e.g. global-load/link knobs) | 8 |
| Dead-band (the subset with no equivalent, e.g. half-cycle clocking) | 8 |
| Global load | 6 |
| Time-base (the subset with no equivalent, e.g. one-shot sync, period link) | 6 |
| Chopper | 4 |

These line up with the "removed entirely" list in
`.claude/skills/epwm-to-mcpwm-sysconfig-migration/references/phase3-overview.md`
(the SPRADL7 distillation) -- this JSON is the field-by-field enumeration
of what that document describes at a submodule level.

## How to use it

1. Take a configurable ID from a real project (e.g. from a live
   `getInstanceConfiguration` call against the SysConfig MCP server, or
   from one of the saved example JSON files in this repo).
2. Look it up in this file by that exact ID.
3. Check `status`:
   - `mapped` -- use `to_config` as the MCPWM field name, but read the
     **Limitation** section below before assuming the value carries over
     unchanged.
   - `no_equivalent` -- there is nothing to set on the MCPWM side; note it
     as a dropped setting rather than silently omitting it from your own
     migration notes.
   - `partial` -- don't use `to_config` at all (there usually isn't one
     useful for these); follow `option_map` and read `fixMsg` carefully,
     since these describe an actual restructuring, not a rename.

## Limitation: this file assumes one EPWM instance maps to pair 1 of one MCPWM instance

This is the one thing to watch for. A single MCPWM instance has **three**
PWM output pairs (1A/1B, 2A/2B, 3A/3B) sharing one time-base counter, and a
real migration very often consolidates **multiple** EPWM instances onto
the pairs of a **single** MCPWM instance (that's the whole reason MCPWM
exists -- more outputs per instance). This file has no awareness of that
at all, because -- as noted above -- it's generated from the module
schemas, not from any project with actual instances and groupings.

Concretely, verified directly against the data:

- Of the 88 `mapped` entries, 18 are action-qualifier fields (e.g.
  `epwmActionQualifier_EPWM_AQ_OUTPUT_A_ON_TIMEBASE_ZERO`). **Every one of
  them** maps to the **pair-1** MCPWM field
  (`mcpwmActionQualifier_MCPWM_AQ_OUTPUT_1A_ON_TIMEBASE_ZERO`). None of
  them ever mention the pair-2 or pair-3 variants
  (`..._2A_.../..._3A_...`), even though a live MCPWM instance genuinely
  exposes full pair-2 and pair-3 action-qualifier field sets.
- Same story for counter-compare: a live MCPWM instance exposes three full
  field sets -- `mcpwmCounterCompare_cmpA`, `mcpwmCounterCompare_cmpA_pwm2`,
  `mcpwmCounterCompare_cmpA_pwm3` (and the same pattern for `cmpB` and
  their shadow-enable/shadow-mode counterparts) -- but this file's
  `to_config` for `epwmCounterCompare_cmpA` only ever points at the
  un-suffixed, pair-1 field.

So the file's implicit assumption is: **exactly one source EPWM instance,
landing on pair 1 of exactly one target MCPWM instance.** If your
migration only ever has one EPWM instance in play, that's fine as-is. If
you're consolidating two or three EPWM instances onto one MCPWM instance
(the common case), you have to substitute the pair number yourself for
every per-pair field this file gives you -- the second and third EPWM
instances in a group need `_pwm2`/`_pwm3` (counter-compare) or `2A/2B`,
`3A/3B` (action-qualifier) appended/substituted into the field name
according to which pair slot that instance actually landed in, based on
your own confirmed grouping.

This doesn't affect every field, though -- some MCPWM fields are
genuinely **instance-wide** regardless of how many pairs are populated
(time-base period/clock-divider/counter-mode/sync settings, dead-band,
and the single MCPWM interrupt-source configurable covered by the one
`partial` entry). For those, the flat mapping in this file is accurate as
given. But it raises a different question this file can't answer either:
if you're merging three EPWM instances that each had *different*
dead-band or interrupt settings onto one MCPWM instance with only *one*
shared field for that setting, which source instance's value should win?
This file has no way to flag that conflict -- it just describes one field
mapping to another in isolation, assuming there's only ever one source
value to consider.

## Bottom line

Treat this file as a **field-existence reference** -- does *any* MCPWM
equivalent exist for a given EPWM configurable, and what's it generically
called -- not as something you can mechanically replay field-by-field
across a multi-instance consolidation. Every per-pair `to_config` needs
its pair number substituted according to your actual grouping, and every
genuinely shared field needs an explicit decision about which source
instance's value to keep whenever more than one EPWM instance lands in the
same group.

This is exactly the kind of translation (not copy) that
`.claude/skills/epwm-to-mcpwm-sysconfig-migration/references/phase2-instance-setup.md`
already does by hand for time-base and synchronization settings, resolving
everything through the confirmed group -> instance mapping rather than
trusting a flat file. The not-yet-designed phase 3 (action-qualifier,
counter-compare, dead-band, trip-zone, event-trigger) will need to do the
same thing this file can't: resolve each field through the real pair
assignment, and make an explicit, visible call on every shared-field
conflict instead of silently picking one source instance's value.

## Related files in this repository

- `epwm_mcpwm_syscfg_migration.json` -- the data this guide describes.
- `.claude/skills/epwm-to-mcpwm-sysconfig-migration/references/phase3-overview.md` --
  the submodule-level distillation of TI's SPRADL7 application note (now
  phase 3's hub file); explains *why* the `no_equivalent` categories above
  were removed, not just that they were.
- `syscfg_mcp.md` -- SysConfig MCP server tool reference, for actually
  reading/writing configurable values on a live project.
- `.claude/skills/epwm-to-mcpwm-sysconfig-migration/` -- the migration
  skill itself, which handles the per-instance grouping problem this file
  doesn't.
