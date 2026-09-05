# BIOS Password Unlock

## Comprehensive Guide: Bypassing Protected Range Registers (FPRR) & BIOS Lock on Modern Acer Predator Laptops

This document outlines the end-to-end technical workflow for dumping, extracting, unlocking variables via UEFI Shell, and flashing a modified BIOS region using Intel CSME System Tools.

---

## Prerequisites & Required Tools

Ensure you have downloaded and organized the following tools in a working directory on your PC:

- **Intel CSME System Tools 16.1** (matching your firmware's ME version) — specifically the `Flash Programming Tool (FPT)` directory (`fptw64.exe`) and `MEInfo` (`MEInfoWin64.exe`).
- **UEFITool (latest)** and **UEFIExtract (latest)** — for navigating the firmware structure and extracting specific raw modules.
- **IFRExtractor-RS (latest)** — to parse extracted UEFI PE32 binary blobs into readable language configuration text files.
- **setup_var.efi** — a specialized UEFI shell utility used to read and write NVRAM variables from a command line environment. This guide assumes the 0.3.0+ syntax (`VAR_NAME[(VAR_ID)]:OFFSET[(SIZE)][=VALUE]`, one whitespace-free token per value); older 0.2.x builds use a different space-separated syntax.
- **shellx64.efi** — the primary native UEFI shell environment file.
- **HxD** — for manual hex-level verification of dumped/modified firmware images, used as a sanity check alongside FPT's own verify pass.

Tool versions actually used and confirmed working for this process:
- Intel CSME System Tools: **16.1** (FPT reported `16.1.25.2049`, MEInfo reported `16.1.25.1932`)
- IFRExtractor-RS, UEFITool, UEFIExtract: latest releases at time of writing

---

## Phase 0: Verify Boot Guard Status (Mandatory Pre-Check)

**Do this before investing time in Phases 1–4.** Everything downstream in this guide only works because Intel Boot Guard's Verified/Measured Boot is *not* fused active on the target hardware. If it is, disabling `BIOS Lock`/`FPRR` via `setup_var.efi` will have no effect on Error 167 — Boot Guard enforces flash protection from a fused, unmodifiable state in the PCH itself, independent of any Setup/NVRAM variable, and no in-band software tool (Windows, Linux, or UEFI Shell) can override it.

1. From an elevated Windows command prompt, in the CSME System Tools `WIN64` directory, run:
    ```bash
    MEInfoWin64.exe -verbose
    ```
2. Scroll to the bottom section of the report and check these specific fields (each has an FPF/fuse column and a runtime column):
    ```
    Measured Boot            Disabled   Disabled
    Verified Boot            Disabled   Disabled
    Force Boot Guard ACM     Disabled   Disabled
    Protect BIOS Environment Disabled   Disabled
    ```
3. **If all four read `Disabled`/`Disabled`:** proceed with this guide — the PRR block you'll hit later is coming from the platform's Setup-controlled PRx programming, which the rest of this document successfully bypasses.
4. **If any of these read `Enabled` on the FPF (left) column:** stop. This method will not get you past Error 167. Boot Guard's public key hash and policy are burned into one-time-programmable fuses at the factory; software running after that fuse is blown cannot disable it, regardless of what any Setup checkbox says. The only remaining path in that scenario is external hardware SPI programming (e.g., CH341A + SOIC-8 clip directly on the flash chip), which bypasses the PCH's SPI controller — and its PRx enforcement — entirely by talking to the chip's own pins.

---

## Phase 1: Dumping the Factory BIOS Firmware

Before making any modifications, you must read the existing firmware partition currently active on the motherboard.

1. Open your Windows **Command Prompt** as an **Administrator**.
2. Navigate into the folder containing your specific FPT version (e.g., the `WIN64` subfolder).
3. Execute the target extraction command to read only the unprotected consumer partition:
    
    ```bash
    fptw64.exe -bios -d backup_bios.bin
    ```
    
4. Verify that the terminal returns a green `FPT Operation Passed` text string. Your raw firmware dump is now saved as `backup_bios.bin`.

*(Note: At this stage, modify the dump file to address your required firmware configurations, such as structural modifications to remove administrative constraints. Save your final modified output file as `mod_bios.bin`)*.

---

## Phase 2: Locating the Hardware Lock Offsets

Modern platforms implement hardware security bits preventing software-level flashing tools from overwriting active firmware memory ranges. You must map out exactly where these switches reside in your motherboard's internal NVRAM registers.

### 1. Extracting the Setup Module Body

1. Launch **UEFITool** and load your `mod_bios.bin` firmware file.
2. Press `Ctrl + F` and navigate to the **GUID** search tab.
3. Search for the standard setup package identifier: `FE3542FE-C1D3-4EF8-657C-8048606FF670`.
4. Double-click the search result at the bottom panel to auto-scroll directly to the true system **`SetupUtility`** module wrapper in the structural tree.
5. Click the dropdown arrow to expand this module.
6. Right-click specifically on the inner **`PE32 image section`** layer.
7. Select **`Extract body...`** from the context menu. Save this file into your directory precisely as `setup_body.bin`.

### 2. Decompiling to a Readable Map

1. Open a command prompt inside your working directory and execute **IFRExtractor-RS** against your extracted binary layer:
    
    ```bash
    ifrextractor.exe setup_body.bin setup_ifr.txt
    ```
    
2. The decompiler will break down the asset package into text fragments mapping out your internal language blocks. Look for the file reflecting your primary configuration workspace, typically built as:
`setup_body.bin.0.0.en-US.uefi.ifr.txt`

### 3. Pinpointing Variable Names and Offsets

Open the translated `.txt` file in an editor (like Notepad) and use `Ctrl + F` to write down your explicit memory mappings:

- **BIOS Lock:** Look for the offset array specifying the BIOS Lock status.
    
    ```
    OneOf Prompt: "BIOS Lock", ..., VarStoreId: 0x5, VarOffset: 0x1C
    ```
    
- **FPRR (Flash Protection Range Registers):** Look for the protection register configurations.
    
    ```
    OneOf Prompt: "Flash Protection Range Registers (FPRR)", ..., VarStoreId: 0x5, VarOffset: 0x73D
    ```
    
- **The VarStore Map:** Scroll to the absolute header ceiling of the document to inspect what text string labels map directly to `VarStoreId: 0x5`. For your modern Predator motherboard, this string is mapped under the name: **`PchSetup`**.

**Important constraint:** `VarStoreId` numbers are not globally unique — the same ID can be reused by different named variables in different form packages within the same dump. Before trusting a match:

1. Search the *entire* decompiled text file for every occurrence of `VarStoreId: 0x5,` (not just the one near your target `OneOf` block) and confirm there's exactly one `VarStoreEfi`/`VarStore` declaration line using that ID.
2. Confirm the declared variable's `Size` is large enough to contain your target offset (e.g., `PchSetup` here is declared `Size: 0x80D`, comfortably covering offset `0x73D`). If the size is smaller than the offset, you have the wrong variable — `setup_var.efi` will also throw an explicit "offset exceeds variable size" error confirming this if you try to use it anyway.
3. Confirm the variable name isn't duplicated elsewhere under a *different* GUID (some firmwares reuse names like `Setup` across multiple unrelated variables). If it is, `setup_var.efi` will report ambiguity and require a `VAR_ID` in parentheses — e.g. `PchSetup(1):0x73D=0x00` — to disambiguate.

---

## Phase 3: Building the Bootable UEFI Shell

To modify hardware variables at a level deeper than the active operating system controls, you must boot into a native UEFI configuration console environment.

### 1. Formatting and File Mapping

1. Connect a USB storage drive to your PC.
2. Format the disk specifically to the **FAT32** file system format.
3. Open the root directory of your USB device and construct this folder layout structure:
    
    ```
    USB_ROOT:\EFI\BOOT\
    ```
    
4. Place your native UEFI environment console tool (`shellx64.efi`) inside that `BOOT` folder and rename it exactly to: **`BOOTX64.EFI`**.
5. Copy your pre-downloaded **`setup_var.efi`** tool and drop it directly onto the **root** baseline of the USB storage drive layout.

### 2. Accessing the Environment via Windows Recovery

Because modern motherboards often restrict raw hardware boot keys during traditional cold startups or due to configuration locks, utilize Windows Advanced Boot controls to comfortably steer the launch sequence:

1. Within your active Windows environment, navigate to:
`Settings > System > Recovery`.
2. Find the **Advanced startup** row and click **Restart now**.
3. Upon system logo reload, choose **Use a device** from the blue menu interface screens.
4. Select your **USB Flash Drive**. The laptop will recycle power and execute your custom terminal environment directly into the command-line UEFI shell interface.

---

## Phase 4: Overriding Security Bits inside the UEFI Shell

Once your console interface finishes initialising and drops you to a blinking prompt, proceed to toggle off the motherboard's built-in firmware write protections.

1. Check current assignments by reading the exact state variables using the parameters discovered during Phase 2:
    
    ```
    setup_var.efi PchSetup:0x1C
    setup_var.efi PchSetup:0x73D
    ```
    
    *(The screen output should display `0x01` for both items, proving that BIOS Lock and Flash Protection Range Registers are actively locked down by default)*.
    
2. Execute the clear sequences by feeding the terminal explicit target byte variables:
    
    ```
    setup_var.efi PchSetup:0x1C=0x00
    setup_var.efi PchSetup:0x73D=0x00
    ```
    
3. Optionally, re-read both offsets immediately to confirm the write landed in NVRAM:
    
    ```
    setup_var.efi PchSetup:0x1C
    setup_var.efi PchSetup:0x73D
    ```
    Both should now report `0x00`. Note that this confirms the NVRAM variable was updated — it does **not** yet confirm the PRx hardware registers reflect the change (see step 4 below for why that distinction matters).

4. **Critical: you must trigger a genuine platform reset here, not just exit the shell.** `setup_var.efi` writing to NVRAM via `SetVariable()` takes effect immediately in storage, but the actual Protected Range registers in the PCH's SPI controller are only *reprogrammed* during the PEI phase of the next POST, based on whatever the `PchSetup` variable says at that boot. Typing `exit` returns you to the boot manager within the *same* power-on session — POST does not run again, so PEI never re-reads the updated variable or reprograms PRx. The hardware registers will still reflect whatever was set at the start of the session you're currently in, and Error 167 will persist in Phase 5 even though the NVRAM value is correctly `0x00`.

    Use one of:
    ```
    reset
    ```
    or a full physical power-off/power-on cycle (more reliable if `reset` behaves inconsistently on your firmware, and the recommended way to confirm persistence rather than just a warm reset).

5. After the reset, boot back into the UEFI Shell **before** going to Windows and re-run the two read commands one more time. If both still show `0x00` after a genuine reset, the values are confirmed both stored and applied — proceed to Phase 5. If either reverts to `0x01` on its own, the platform is re-enforcing the value at every POST regardless of the stored setting, and this method will not work as-is.

---

## Phase 5: Executing the Final BIOS Flash

Your motherboard registers have now been opened up to receive payloads. FPT can now communicate directly with the SPI chips without getting intercepted by FPRR or BIOS Lock.

1. Let your system complete its standard boot path cleanly straight into **Windows**.
2. Launch your command prompt environment as an **Administrator**.
3. Pivot your operational pointer directly back inside your working FPT package directory:
    
    ```bash
    cd "C:\Path\To\Intel CSME System Tools\Flash Programming Tool\WIN64"
    ```
    
4. Copy your edited, prepared file (`mod_bios.bin`) and make sure it sits directly inside that `WIN64` folder execution target window.
5. Issue the write instruction string targeting the newly unlocked partition:
    
    ```bash
    fptw64.exe -bios -f mod_bios.bin
    ```
    
6. With Boot Guard confirmed off (Phase 0) and PRx confirmed cleared via a genuine reset (Phase 4, step 5), FPT should now step past Error 167, displaying a sequential loading meter as it erases and reprograms only the blocks that actually differ from what's on the chip. Wait for the success line — observed output ends with `RESULT: The data is identical.` followed by `FPT Operation Successful.`
    
    *(Note: `Error 280` is referenced in earlier drafts of this guide but was not something encountered or diagnosed during this specific run — if you hit it, treat it as a separate issue to research rather than assuming it's covered by the fixes above.)*

7. **Before rebooting into Windows**, optionally re-dump the just-flashed region (`fptw64.exe -bios -d verify_dump.bin`) and open both `mod_bios.bin` and `verify_dump.bin` side-by-side in **HxD**. Use HxD's compare feature (or a manual byte-range spot check around the offsets you intended to change) to confirm the written image matches what you intended to flash, independent of FPT's own internal verify pass. This catches cases where FPT reports success but a downstream tool (UEFITool, IFRExtractor) misread an offset earlier in the process.
8. Restart your device immediately and verify functionality.

*(Note: The very first restart after completing this phase can prompt a black screen delay lasting roughly 60–120 seconds while memory maps retrain. Allow this sequence to wrap up entirely undisturbed until the Acer Predator splash screen confirms a successful firmware boot).*

---

## Hardware & Software Constraints

This method is not universal. It worked on this specific Predator unit under the following conditions, all of which must independently hold true:

| Constraint | How to verify | What happens if it doesn't hold |
|---|---|---|
| Boot Guard Verified/Measured Boot disabled (fused off) | `MEInfoWin64.exe -verbose` → `Verified Boot`, `Measured Boot`, `Force Boot Guard ACM` all `Disabled` on the FPF column | Software-level PRx bypass is impossible regardless of Setup variables; requires external SPI programmer |
| `BIOS Lock` and `FPRR` are real, wired Setup options (not cosmetic/vestigial) on this firmware | Confirmed by successful flash after toggling them | Some OEM builds expose the checkbox but a separate PEI/DXE module ignores it or has a hardcoded fallback; would require finding an alternate offset or accepting external programming |
| `VarStoreId` for the target offsets resolves to exactly one variable, with enough declared size to contain the offset | Full-document grep for `VarStoreId: 0x5,`, cross-checked against `Size:` field | Ambiguous or undersized matches produce "offset exceeds variable size" or "no variable found" errors from `setup_var.efi` |
| A genuine platform reset (not shell `exit`) occurs between the `setup_var.efi` write and the FPT flash attempt | Re-read both offsets from the Shell after reset; both should read `0x00` | PRx hardware registers retain their pre-write state for the remainder of the current boot session; Error 167 persists even though NVRAM is correctly updated |
| Flash Descriptor region and ME region are not independently write-locked for the specific blocks being modified | FPT's own erase/program/verify output completing without new errors | Descriptor- or ME-region-specific protections are a separate mechanism from PRx/BIOS Lock and are out of scope for this guide |

If any row fails, the fix is not "try harder with `setup_var.efi`" — it's either finding the platform-specific reason the toggle isn't wired through, or moving to external hardware programming (CH341A + SOIC-8 clip) as the fallback method, which bypasses the PCH's SPI controller and its PRx enforcement entirely by talking directly to the flash chip's pins.