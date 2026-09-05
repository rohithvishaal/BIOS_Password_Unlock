# Acer Predator BIOS Lock / FPRR Bypass Guide

![Platform](https://img.shields.io/badge/platform-Intel%20PCH-blue)
![Firmware](https://img.shields.io/badge/firmware-AMI%20UEFI-lightgrey)
![Status](https://img.shields.io/badge/status-verified%20working-brightgreen)
![License](https://img.shields.io/badge/license-MIT-informational)

A step-by-step, verified workflow for disabling **BIOS Lock Enable (BLE)** and **Flash Protection Range Registers (FPRR)** on Intel PCH-based Acer Predator laptops, in order to flash a custom/modified BIOS image via Intel's Flash Programming Tool (FPT) — without needing an external SPI programmer.

This isn't a generic copy-paste guide. It walks through the actual diagnostic path used to get from `Error 167: Protected Range Registers are currently set by BIOS` to a successful in-band flash, including the platform checks that determine whether this approach will work on your hardware at all.

---

## ⚠️ Before You Start

This process edits low-level firmware security registers that exist specifically to prevent bad or malicious BIOS writes. Used correctly, it's how you regain the flash access those registers are designed to block. Used carelessly, it removes that safety net right before the operation where you'd want it most.

- **Read the whole guide once before running anything.** Steps are order-dependent — running Phase 4 before confirming Phase 0 will waste your time at best.
- **Back up your factory BIOS dump** before modifying anything, and store the backup somewhere other than the machine you're flashing.
- **Have a hardware recovery fallback ready** (CH341A programmer + SOIC-8 clip) before you flash. If this guide's software path fails for your specific board (see [Constraints](#hardware--software-constraints)), external SPI programming is the fallback — not another software workaround.
- **Do not interrupt power during any flash operation.** With FPRR/BLE disabled, SMM-level flash protection is off; there is no safety net for an interrupted write.

---

## What This Guide Covers

| Phase | What happens |
|---|---|
| 0 | Verify Intel Boot Guard status via MEInfo — determines if this method can work *at all* on your unit |
| 1 | Dump the factory BIOS with FPT |
| 2 | Extract the Setup module and decompile it to find variable names/offsets |
| 3 | Build a bootable UEFI Shell USB |
| 4 | Disable `BIOS Lock` and `FPRR` via `setup_var.efi`, then force a real platform reset |
| 5 | Flash the modified BIOS with FPT and verify the result independently with HxD |

Full walkthrough: [`BIOS_Password_Unlock.md`](./BIOS_Password_Unlock.md)

---

## Tools Used

| Tool | Purpose | Version tested |
|---|---|---|
| [Intel CSME System Tools](https://www.intel.com/content/www/us/en/download/) (FPT + MEInfo) | Dump/flash BIOS region, read Boot Guard fuse status | 16.1 (FPT `16.1.25.2049`, MEInfo `16.1.25.1932`) |
| [UEFITool](https://github.com/LongSoft/UEFITool) | Navigate firmware structure, extract Setup module | Latest (NE build) |
| [UEFIExtract](https://github.com/LongSoft/UEFITool) | Bulk-extract firmware sections | Latest |
| [IFRExtractor-RS](https://github.com/LongSoft/Universal-IFR-Extractor) | Decompile Setup module into readable variable/offset map | Latest |
| [setup_var.efi](https://github.com/datasone/setup_var.efi) | Read/write UEFI NVRAM variables from the Shell | 0.3.0+ syntax |
| [HxD](https://mh-nexus.de/en/hxd/) | Manual hex verification of dumped/flashed images | Latest |

---

## Hardware & Software Constraints

This is a conditional recipe, not a universal one. Every row below must hold for it to work:

| Constraint | How to verify | If it fails |
|---|---|---|
| Boot Guard Verified/Measured Boot disabled | `MEInfoWin64.exe -verbose` → `Verified Boot`, `Measured Boot`, `Force Boot Guard ACM` all `Disabled` | Software bypass is impossible; requires external SPI programmer |
| `BIOS Lock`/`FPRR` are genuinely wired, not cosmetic | Confirmed by a successful flash after toggling | Toggle may be vestigial on some OEM builds; find an alternate offset or use external programming |
| `VarStoreId` resolves to exactly one variable with sufficient size | Grep the decompiled dump for all matching `VarStoreId` lines, check `Size:` | Ambiguous/undersized matches throw explicit errors from `setup_var.efi` |
| A genuine platform reset occurs between the variable write and the flash attempt | Re-read both offsets after reset — should show `0x00` | PRx hardware registers won't reflect the change until a real POST reprograms them; `exit` alone is not enough |
| Flash Descriptor / ME region aren't separately locked | FPT completes erase/program/verify without new errors | Descriptor/ME-region locks are a separate mechanism, out of scope here |

---

## Contributing

Found a board where a step in this guide behaved differently, or hit an error not covered here (e.g. FPT Error 280)? Open an issue with your `MEInfoWin64.exe -verbose` Boot Guard section and the exact error text — that's the fastest way to figure out which constraint above is the actual blocker.

## License

MIT — use at your own risk. Flashing firmware can render a device unbootable; this guide documents one specific successful process and is not a guarantee of results on other hardware.
