# Rockbox 4.0 for iPod 1G/2G, boot-crash fix

`rockbox.ipod` here is the official Rockbox 4.0 release binary for
`ipod1g2g` with one seven-byte change, and nothing else.

## Why

Rockbox 4.0 as released crashes at boot on every iPod 1G/2G, flash mod or
not, with:

    Data abort at 0006e8ec (0)

The cause is an unaligned halfword store in the PP5002 core-wake assembly
(`firmware/target/arm/pp/thread-pp.c`); the 4.0 link happened to place the
`core_semaphores` array at an odd address. Rockbox fixed it upstream on
2025-08-04 (commit `c33602375`, "pp500x: Switch to plain C sleep/wake code"),
after the 4.0 branch point, so it is in every development build but not in
the 4.0 release that Rockbox Utility installs by default.

## What changed

Two instructions in `core_wake()`: the halfword store of the two wake flags
is replaced by two byte stores, and the file's checksum header is
recomputed. Seven bytes differ from the release file.

| | |
|---|---|
| Base | Rockbox 4.0 release, `rockbox-ipod1g2g-4.0.zip`, `rockbox.ipod` sha256 `15e24f3bf620e36800d158f07cbb6e58df14063248964ec023f45529cf7b43d8` |
| This file | sha256 `c4e0dcfaeb929f394fc2feb81e0e026f8aaf88af5b29202950d3f1cfb222071c`, md5 `82623209da5c5d5cfa1044d37ee7f286`, 572456 bytes |
| Patch site | file offset `0x6e8f0`: `orr r1,r1,r1,lsl #8` / `strh r1,[ip]` becomes `strb r1,[ip,#1]` / `strb r1,[ip]` |

No ATA, timing, or storage code is touched. It behaves exactly like 4.0
with the upstream core-wake fix applied.

## Install

1. Install Rockbox 4.0 and the bootloader normally with Rockbox Utility
   (target: iPod 1st/2nd gen).
2. With the iPod in disk mode, copy this `rockbox.ipod` over
   `<iPod>/.rockbox/rockbox.ipod`.
3. Eject and reset (Menu + Play).

Alternatively install a Rockbox development build from Rockbox Utility; those
already contain the upstream fix and need no patched file. Once a Rockbox
release newer than 4.0 ships, this file is obsolete.

## Sphinxmoth gateware requirement

Rockbox drives the card directly in PIO. Bridge images up to `v1.1-mg132-fix3b`
add about 45 ns to every register read through the passthrough synchroniser,
which is more than the PP5002 tolerates at Rockbox's stock IDE timing:
init fails with `ATA error: -32`. Flash the raw-pad PIO passthrough image
(fix3b + `fix/raw-pad-pio-passthrough`) or newer. With that image, unmodified
Rockbox boots, reads, and writes at stock timing; verified 2026-09-07 with the
2026-09-05 development build (commit `2ec4760117`) on an iPod 2G, genuine CF
card and FC1307A SD adapter.

## For bridges still on `v1.1-mg132-fix3b`: `rockbox.ipod.fix3b-gateware`

If the bridge has not been updated, `rockbox.ipod` above stops at
`ATA error: -32`. `rockbox.ipod.fix3b-gateware` is the 4.0 binary with the
core-wake fix plus three more changes that work around the fix3b read
latency from the Rockbox side (67 bytes differ from the release file):

- CPU pinned at 80 MHz (the 30 MHz "normal" clock is set to the same PLL
  values), and the IDE timing registers set to the retail OS's 80 MHz values
  (0xc0003000 = 0x11c1, 0xc0003004 = 0x8000ffff, config bits 2 and 3 set)
  instead of Rockbox's 0x10 / 0x80002150. Battery life is worse than stock.
- PIO write loop paced with a status read after every data word, so the CPU
  cannot outrun the slower bus (otherwise every write fails with -4).
- End-of-transfer check polls until the drive is ready instead of sampling
  once, and SET FEATURES writes the sector count before the features byte.
  Neither turned out to be necessary; they are harmless and were in the
  tested build.

sha256 `d17588e6d17fc01cdffd00dd4e2a5caadec013a216654523549aa221e23a8426`.
Verified 2026-09-07 on fix3b (0x003B) with an FC1307A SD adapter (boot,
reads, disk-cache flush writes, playback) and, on a second board and iPod,
with an iFlash CF/SD adapter (boot to menu; the stock-timing file gave
`ATA error: -11` on that same setup). Install the same way, over a clean
4.0 install. Prefer updating the bridge instead; this file exists for iPods
that cannot be reflashed.

## Behaviour to expect

- Plugging in FireWire makes Rockbox flush the disk and reboot into Apple
  disk mode. That is normal Rockbox behaviour on 1G to 3G iPods, not a fault.
- Rockbox cuts drive power a few seconds after the disk sleeps; the bridge
  re-initialises the card on wake, as it does for the retail OS.
