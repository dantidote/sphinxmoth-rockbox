# Rockbox 4.0 for iPod 3G on a `v1.1-mg132-fix3b` bridge: `rockbox.ipod.fix3b-gateware`

**UNTESTED ON HARDWARE.** Built 2026-09-07 by applying, byte for byte, the
same five changes that were verified on the iPod 1G/2G build
(`../4.0-ipod1g2g/rockbox.ipod.fix3b-gateware`) to the official
Rockbox 4.0 `ipod3g` release binary. Every patch site was located by
disassembly and has the identical instruction shape in both builds, but no
3G with a fix3b bridge has booted it yet. Test before shipping.

## Why a 3G needs it

The 3G is the same PP5002 as the 1G/2G, and Rockbox uses the same ATA
driver for it, so it hits the same two problems on a fix3b bridge:
`ATA error: -32` at init (the bridge's synchroniser adds ~45 ns to every
register read, more than Rockbox's stock 0x10 IDE timing tolerates at
80 MHz) and, once slower timing is used, every write failing with -4
(Rockbox's PIO write loop outruns the slowed bus and the controller drops
words).

Unlike the 1G/2G, the 3G 4.0 release does **not** need the core-wake fix:
in this build the `core_semaphores` array landed on a word-aligned address
(0x400068a8), so the unaligned-store crash does not occur. Stock 4.0 boots
on a 3G; only the ATA changes below are applied.

## What changed (62 bytes differ from the release file)

| Site (file offset) | Change |
|---|---|
| `0x6db28`, `0x6db30` | `set_cpu_frequency(CPUFREQ_NORMAL)` programs the 80 MHz PLL (div 3, mult 10), so the IDE timing below is valid at all times. Battery life is worse than stock. |
| `0x6dd64`–`0x6dd98` | `ata_device_init`: IDE timing 0x11c1 / 0x8000ffff (retail 80 MHz values) instead of 0x10 / 0x80002150, config bits 2 and 3 set. |
| `0x68ad4`–`0x68ae8` | Aligned PIO write loop paced with one alternate-status read per data word. This is the change that makes writes work. |
| `0x68b28`–`0x68b44` | End-of-transfer check polls until the drive reports ready instead of sampling once (bounded). Harmless; not strictly needed. |
| `0x68044`–`0x68060` | SET FEATURES writes the sector count before the features byte. Harmless; not strictly needed. |

| | |
|---|---|
| Base | Rockbox 4.0 `rockbox-ipod3g-4.0.zip`, `rockbox.ipod` sha256 `3f4d088b52ce8fbe0cfa8a763d5620e7ff727c0ef452826f2399bcf0c44e3c9b` |
| This file | sha256 `8c9684d6252ef4ae9de7ad0c5be29675430fe52615b77df7e99b1404b604add0`, md5 `f2d1bf7aaa2fcbd95774bc7f8cf9ce9f`, 579340 bytes |

## Install

Install Rockbox 4.0 and its bootloader for the iPod 3G with Rockbox Utility,
then in disk mode copy this file over `<iPod>/.rockbox/rockbox.ipod`, eject,
reset (Menu + Play). It must sit on a 4.0 install: codecs and plugins are
version-locked to the binary, and a mismatch makes every track skip.

## Not needed on the raw-pad PIO bridge image

On bridges running the raw-pad passthrough image (fix3b plus
`fix/raw-pad-pio-passthrough`), stock Rockbox 4.0 for the 3G should work as
released. This file is only for boards already in the field on fix3b.
