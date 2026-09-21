# C2000Ware 26.01.00.00 STS — Full Installer

This repo's checkout (`c2000ware-core-sdk`, root of this repository) is a
**partial** C2000Ware tree — it's missing `libraries/communications/Ethernet`
entirely, which means no lwIP, no EMAC networking library, and no Ethernet
examples (CM-core or otherwise). The full installer restores that content.

## Source

Downloaded from a Dropbox share link provided directly by the team:

```
https://www.dropbox.com/scl/fi/39hlbiup2uzauo4fbwya4/C2000Ware_26_01_00_00_STS_setup.run?rlkey=gss9n2quqguovkyf7loukax61&st=7duzx8s7&dl=0
```

Filename: `C2000Ware_26_01_00_00_STS_setup.run`
Size: 332,093,771 bytes (~332 MB)
Type: Linux x86-64 ELF self-extracting installer (TI/BitRock InstallBuilder)

**Not stored in this repo** — at ~332 MB it exceeds GitHub's 100 MB
plain-push limit, and this repo has no Git LFS set up. If you need the
`.run` file itself, re-download it from the Dropbox link above, or from
wherever the team keeps it internally.

## Install command used

```bash
chmod +x C2000Ware_26_01_00_00_STS_setup.run
./C2000Ware_26_01_00_00_STS_setup.run \
    --mode unattended \
    --unattendedmodeui none \
    --prefix /root/ti/C2000Ware_26_01_00_00
```

Installed alongside the CCS install (`/root/ti/ccs2100`), **not** inside
this git checkout — keeps the ~1.7 GB payload out of the repo entirely.

## Resulting path

The installer nests its own version folder under the prefix:

```
/root/ti/C2000Ware_26_01_00_00/C2000Ware_26_01_00_00/
```

Confirmed present after install (absent from this repo's checkout):

```
/root/ti/C2000Ware_26_01_00_00/C2000Ware_26_01_00_00/libraries/communications/Ethernet/third_party/lwip/
├── lwip-2.1.2/                  # upstream lwIP source
├── ports/C2000/                 # C2000 lwIP port
├── ports/FreeRTOS/
├── driver/
├── board_drivers/
└── examples/
    ├── enet_lwip/
    ├── enet_lwip_udp/            # <- CM-core lwIP + UDP example
    ├── enet_lwip_freertos/
    ├── enet_lwip_udp_freertos/
    └── enet_lwip_iperf_freertos/
```

`enet_lwip_udp` is the closest example for lwIP-based UDP work on the CM
core.

## For future sessions

If this repo's checkout still lacks `libraries/communications/Ethernet`
(check first — it may get folded into a future sync), re-run the install
command above with the Dropbox-downloaded `.run` file to regenerate the
full SDK tree at `/root/ti/C2000Ware_26_01_00_00/C2000Ware_26_01_00_00/`.
