# Getting Started: Headless Code Composer Studio for C2000Ware

This guide documents how to install Code Composer Studio (CCS) and the
SysConfig tool in a **headless (no GUI)** Linux environment, and how to use
their command-line interfaces to import and build a C2000Ware driverlib
example project — end to end, with no display required.

It was written from a real installation on a headless Linux container and
includes the two workarounds that were needed to get an unattended install
and project import working there.

---

## 1. Prerequisites

- 64-bit Linux
- ~6 GB free disk space (C2000-only component install)
- `curl`, `unzip`
- Outbound HTTPS access to `ti.com` / `dr-download.ti.com`

No GUI, X server, or display is required for anything in this guide.

---

## 2. Download the installers

TI's download page redirects (`302`) to a CDN host, so `curl` must follow
redirects (`-L`).

```bash
WORKDIR=~/ccs_install
mkdir -p "$WORKDIR" && cd "$WORKDIR"

# Code Composer Studio 21.0.0 — Linux offline installer (~1.5 GB)
curl -sSL -o CCS_21.0.0.00014_linux.zip \
  "https://dr-download.ti.com/software-development/ide-configuration-compiler-or-debugger/MD-J1VdearkvK/21.0.0/CCS_21.0.0.00014_linux.zip"

# SysConfig 1.28.0 standalone installer (~160 MB) — optional, CCS also bundles its own copy
curl -sSL -o sysconfig-1.28.0_4712-setup.run \
  "https://dr-download.ti.com/software-development/ide-configuration-compiler-or-debugger/MD-nsUM6f7Vvb/1.28.0.4712/sysconfig-1.28.0_4712-setup.run"
```

> Check `https://www.ti.com/tool/download/CCSTUDIO` and
> `https://www.ti.com/tool/SYSCONFIG` for current version numbers/URLs —
> the ones above will go stale as TI ships new releases.

Extract the CCS installer:

```bash
unzip -q CCS_21.0.0.00014_linux.zip
chmod +x CCS_21.0.0.00014_linux/ccs_setup_21.0.0.00014.run
```

---

## 3. Environment workarounds (containers without udev)

CCS's Linux installer bundles a Blackhawk emulator driver package that, as
a post-install step, tries to install a udev rule and restart the udev
service — even though we only need CLI build/import functionality and have
no physical hardware debug probe attached. In a minimal container without
udev, this step fails and aborts the whole installation.

Three things fix it:

```bash
# 1. udev has nowhere to put its rules file — give it a directory
mkdir -p /etc/udev/rules.d

# 2. There's no udev service to restart — stub it out so the "restart" call succeeds
mkdir -p /etc/init.d
cat > /etc/init.d/udev <<'EOF'
#!/bin/sh
exit 0
EOF
chmod +x /etc/init.d/udev

# 3. Belt-and-suspenders: shim the "service" command itself, in case anything
#    calls "service udev ..." directly instead of resolving via /etc/init.d.
#    Falls through to the real /usr/sbin/service for every other service name.
mkdir -p /root/.local/bin
cat > /root/.local/bin/service <<'EOF'
#!/bin/sh
if [ "$1" = "udev" ]; then
  exit 0
fi
exec /usr/sbin/service "$@"
EOF
chmod +x /root/.local/bin/service
# Make sure /root/.local/bin is early in PATH so this shim is found first:
export PATH="/root/.local/bin:$PATH"
```

Step 2 (the `/etc/init.d/udev` stub) is what actually made the install
succeed here; step 3 is a defensive fallback in case some other installer
path calls `service` directly rather than going through `/etc/init.d`. It's
harmless to include and costs nothing, so we keep both.

None of this affects compiling or building projects. If you have a real
debug probe and intend to flash/debug from this machine, install on a host
with a working udev instead.

---

## 4. Install CCS (unattended, C2000-only)

The installer supports a fully unattended, non-GUI mode with component
selection, so you can install just the C2000 (`PF_C28`) toolchain instead
of every device family (ARM, MSP430, Sitara, etc.):

```bash
cd "$WORKDIR/CCS_21.0.0.00014_linux"
./ccs_setup_21.0.0.00014.run \
  --mode unattended \
  --unattendedmodeui none \
  --enable-components PF_C28 \
  --prefix /root/ti/ccs2100
```

Useful flags (from `./ccs_setup_*.run --help`):

| Flag | Purpose |
|---|---|
| `--mode unattended` | No GUI/console prompts |
| `--enable-components <list>` | Comma-separated component IDs to install (e.g. `PF_C28` for C2000) |
| `--disable-components <list>` | Explicitly exclude components |
| `--prefix <dir>` | Install directory (default `~/ti/ccs2100`) |
| `--debuglevel <0-4>` | Verbosity for troubleshooting |

Verify the install:

```bash
CCS_ROOT=/root/ti/ccs2100/ccs
ls "$CCS_ROOT"                                   # ccs_base, eclipse, tools, utils, ...
"$CCS_ROOT"/tools/compiler/ti-cgt-c2000_*/bin/cl2000 -version
"$CCS_ROOT"/utils/sysconfig_*/sysconfig_cli.sh --version
```

### Convenience: add tools to `PATH`

```bash
cat >> ~/.bashrc <<'EOF'

# Code Composer Studio (C2000)
export CCS_ROOT=/root/ti/ccs2100/ccs
export PATH="$CCS_ROOT/tools/compiler/ti-cgt-c2000_25.11.1.LTS/bin:$CCS_ROOT/utils/sysconfig_1.28.0:$PATH"
alias ccs-cli="$CCS_ROOT/eclipse/ccs-server-cli.sh"
alias sysconfig_cli="$CCS_ROOT/utils/sysconfig_1.28.0/sysconfig_cli.sh"
EOF
source ~/.bashrc
```

(Adjust the `ti-cgt-c2000_*` and `sysconfig_*` version directory names to
whatever actually got installed — check with `ls "$CCS_ROOT/tools/compiler"`
and `ls "$CCS_ROOT/utils"`.)

---

## 5. The headless CCS CLI (`ccs-server-cli.sh`)

CCS 21's headless automation entry point is:

```bash
$CCS_ROOT/eclipse/ccs-server-cli.sh -workspace '<workspace-dir>' -application <app-id> [<args>]
```

Supported `<app-id>` values: `initialize`, `projectCreate`, `projectImport`,
`projectModify`, `projectBuild`, `inspect`. Get per-application help with:

```bash
$CCS_ROOT/eclipse/ccs-server-cli.sh -workspace /tmp/ws -application projectImport -ccs.help
```

---

## 6. Register this SDK checkout with CCS's product discovery

C2000Ware example `.projectspec` files declare `products="sysconfig;C2000WARE"`.
CCS validates that attribute against its own **product discovery** registry,
which looks for a manifest at `<product-root>/.metadata/.tirex/package.tirex.json`.

The official TI offline installer of C2000Ware includes this manifest
automatically. A plain git checkout of this repository does not, so CCS
reports:

```
!ERROR: Product C2000WARE v0.0 is not currently installed and no compatible version is available.
```

This repo now ships `.metadata/.tirex/package.tirex.json` (see below) so
that error no longer occurs. If you're working from a fork/checkout that's
missing it, recreate it like this:

```bash
mkdir -p /path/to/c2000ware-core-sdk/.metadata/.tirex
cat > /path/to/c2000ware-core-sdk/.metadata/.tirex/package.tirex.json <<'EOF'
[
    {
    "metadataVersion": "3.0.0",
    "id" : "C2000WARE",
    "name" : "C2000WARE",
    "version" : "26.01.00.00",
    "type" : "software",
    "description": "C2000Ware SDK",
    "allowPartialDownload": "false"
    }
]
EOF
```

Set the `version` to match `.metadata/sdk.json`'s `"version"` field in your
checkout.

Then point CCS's product discovery at the **parent directory** of the SDK
checkout (discovery scans a given path's immediate children):

```bash
WORKSPACE=~/ccs_workspace
mkdir -p "$WORKSPACE"

$CCS_ROOT/eclipse/ccs-server-cli.sh \
  -workspace "$WORKSPACE" \
  -application initialize \
  -ccs.productDiscoveryPath "$(dirname /path/to/c2000ware-core-sdk)"
```

Confirm it was found:

```bash
cat "$CCS_ROOT/eclipse/configuration/com.ti.ccs.project/productDescriptor.cache.log"
# Discovered a total of 2 products:
#     .../ccs/utils/sysconfig_1.28.0 : 1.28.0
#     /path/to/c2000ware-core-sdk : 26.1.0.00
```

---

## 7. Import a driverlib example project

Every driverlib example ships a `.projectspec` file under
`driverlib/<device>/examples/<peripheral>/CCS/`. For example, ePWM Example 1
(trip zone) on F28P551x:

```bash
driverlib/f28p551x/examples/epwm/CCS/epwm_ex1_trip_zone.projectspec
```

Import it into a workspace:

```bash
WORKSPACE=~/ccs_workspace
PROJSPEC=/path/to/c2000ware-core-sdk/driverlib/f28p551x/examples/epwm/CCS/epwm_ex1_trip_zone.projectspec

$CCS_ROOT/eclipse/ccs-server-cli.sh \
  -workspace "$WORKSPACE" \
  -application projectImport \
  -ccs.location "$PROJSPEC"
```

Expected output ends with something like:

```
Project details:
    Name: epwm_ex1_trip_zone
    Location: <workspace>/epwm_ex1_trip_zone
    Target configuration: <workspace>/epwm_ex1_trip_zone/targetConfigs/TMS320F28P551SG5.ccxml
```

This copies in the device headers, linker command files, the target
`.ccxml`, the example's `.c`/`.syscfg` sources, and links the project
against the prebuilt `driverlib.lib` for that device
(`driverlib/<device>/driverlib/ccs/Debug/driverlib.lib`) — no need to build
driverlib separately for these examples.

Other useful `projectImport` flags:

| Flag | Purpose |
|---|---|
| `-ccs.device <id>` | Explicit device variant (only needed if `.projectspec` uses a generic device ID and doesn't already pin one via `sysConfigBuildOptions`) |
| `-ccs.renameTo <name>` | Import under a different project name |
| `-ccs.overwrite` | Overwrite existing files at the destination |
| `-ccs.autoBuild` | Build immediately after import |

---

## 8. Build the project

Each `.projectspec` typically defines two build configurations —
`CPU1_RAM` (loads to RAM for debug) and `CPU1_FLASH` (programs flash).
Build one from the CLI:

```bash
$CCS_ROOT/eclipse/ccs-server-cli.sh \
  -workspace "$WORKSPACE" \
  -application projectBuild \
  -ccs.projects epwm_ex1_trip_zone \
  -ccs.configuration CPU1_RAM \
  -ccs.buildType full \
  -ccs.listErrors -ccs.listProblems
```

A successful build runs SysConfig code generation, compiles all sources
with `cl2000`, links against `driverlib.lib`, and ends with:

```
**** Build finished ****
# --------------------------------------------------------------------------------
# Problem summary for project 'epwm_ex1_trip_zone': 0 errors, N warnings
# [<timestamp>]: CCS headless build complete - 0 out of 1 projects have errors
```

The output binary is written to:

```
<workspace>/epwm_ex1_trip_zone/CPU1_RAM/epwm_ex1_trip_zone.out
```

Verify it's a real C2000 ELF executable:

```bash
file "$WORKSPACE/epwm_ex1_trip_zone/CPU1_RAM/epwm_ex1_trip_zone.out"
# ELF 32-bit LSB executable, TI TMS320C2000 DSP family, ...
```

`projectBuild` flags of note:

| Flag | Purpose |
|---|---|
| `-ccs.projects <name...>` | Build specific project(s) by name |
| `-ccs.locations <path...>` | Or build by project path instead of name |
| `-ccs.workspace` | Build every project in the workspace |
| `-ccs.buildType (incremental\|full\|clean)` | Defaults to `incremental` |
| `-ccs.configuration <name>` | Which build configuration (e.g. `CPU1_RAM`, `CPU1_FLASH`) |
| `-ccs.listErrors` / `-ccs.listProblems` | Print errors / errors+warnings after the build |

---

## 9. Known limitations in a headless/no-hardware environment

- **No flashing or debugging**: without udev and a real USB/JTAG debug
  probe attached, you cannot load/run/debug on actual silicon from this
  environment — only build. Use CCS on a machine with the hardware
  attached for that step.
- **`GENERATE_DIAGRAM` post-build step**: several `.projectspec` files run
  an optional CLB diagram-generation post-build step guarded by
  `if ${GENERATE_DIAGRAM} == 1 ...`. In a headless build this can print a
  harmless `Error 2 (ignored)` from `/bin/sh` — it's gated off by default
  (`GENERATE_DIAGRAM=0`) and does not affect the 0-errors build result.

---

## 10. Quick reference: full workflow

```bash
CCS_ROOT=/root/ti/ccs2100/ccs
REPO=/path/to/c2000ware-core-sdk
WORKSPACE=~/ccs_workspace
PROJSPEC="$REPO/driverlib/f28p551x/examples/epwm/CCS/epwm_ex1_trip_zone.projectspec"

# one-time per workspace
"$CCS_ROOT/eclipse/ccs-server-cli.sh" -workspace "$WORKSPACE" -application initialize \
  -ccs.productDiscoveryPath "$(dirname "$REPO")"

# import
"$CCS_ROOT/eclipse/ccs-server-cli.sh" -workspace "$WORKSPACE" -application projectImport \
  -ccs.location "$PROJSPEC"

# build
"$CCS_ROOT/eclipse/ccs-server-cli.sh" -workspace "$WORKSPACE" -application projectBuild \
  -ccs.projects epwm_ex1_trip_zone -ccs.configuration CPU1_RAM \
  -ccs.buildType full -ccs.listErrors -ccs.listProblems
```
