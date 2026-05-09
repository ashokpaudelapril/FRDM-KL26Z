# OpenSDA Firmware Update Guide — P&E Micro FRDM Boards on Linux

## Overview

FRDM boards (FRDM-KL26Z and others) ship with P&E Micro OpenSDA v1 firmware on their
debug chip (MK20DX128). The factory bootloader (v1.09) is incompatible with Mac and
requires a special procedure on Linux to update. This guide documents the complete
working procedure verified on Linux on 2026-05-08.

**Target board:** FRDM-KL26Z  
**Debug chip:** MK20DX128 (OpenSDA v1)  
**OS:** Ubuntu Linux  

---

## Prerequisites

```bash
sudo apt-get install binutils-arm-none-eabi
```

No other tools required for flashing. pyOCD and LinkServer are NOT needed for this workflow.

---

## Files Required

| File | Purpose |
|---|---|
| `BOOTUPDATEAPP_Pemicro_v111.SDA` | Updates bootloader from v1.09 → v1.11. Universal for all P&E Micro OpenSDA v1 boards. |
| `Pemicro_OpenSDA_Debug_MSD_Update_Apps_2023_12_12/MSD-DEBUG-FRDM-KL26Z_Pemicro_v118.SDA` | MSD+Debug application firmware. Board-specific — match filename to your board. |

Both files are in this repository.

---

## Key Rule: Always Mount with `sync`

The Linux kernel FAT driver buffers writes by default. The OpenSDA bootloader reads
directly from the USB mass storage stream and never sees buffered data. **Always remount
with the `sync` option before copying any `.SDA` file.**

```bash
udisksctl unmount -b /dev/sdX
udisksctl mount -b /dev/sdX --options sync
```

This is the fix that makes Linux work. Without it, the copy appears to succeed but the
bootloader ignores the file.

---

## Step 1 — Enter Bootloader Mode

1. Hold the **Reset** button on the board.
2. Plug the USB cable into the **SDA port** (labeled on silkscreen — NOT the port labeled USB).
3. Release the Reset button.

Expected result:

```
$ lsusb | grep -i freescale
Bus 001 Device 006: ID 2504:0200 FREESCALE SEMICONDUCTOR INC. OpenSDA MSD APP

$ lsblk | grep BOOT
sda    BOOTLOADER   /run/media/$USER/BOOTLOADER   vfat
```

Check current bootloader version:

```bash
grep BOOTVER /run/media/$USER/BOOTLOADER/SDA_INFO.HTM
# Expected: value="1.09"
```

If `BOOTVER` is already `1.11` or higher, skip to [Step 4](#step-4--flash-msd-debug-application).

---

## Step 2 — Remount with `sync`

```bash
udisksctl unmount -b /dev/sda
udisksctl mount -b /dev/sda --options sync
```

Verify:

```bash
mount | grep sda
# Must contain: sync
```

---

## Step 3 — Update Bootloader (v1.09 → v1.11)

```bash
cp BOOTUPDATEAPP_Pemicro_v111.SDA /run/media/$USER/BOOTLOADER/
sync
ls /run/media/$USER/BOOTLOADER/
# File must appear in listing before continuing
```

Then:

1. **Unplug the board.** Do NOT run `udisksctl unmount` — just pull the cable.
2. Wait 5 seconds.
3. Plug back in normally (no reset button).
4. Wait up to 30 seconds. The `BOOTLOADER` drive reappears automatically.

Verify the update succeeded:

```bash
grep BOOTVER /run/media/$USER/BOOTLOADER/SDA_INFO.HTM
# Expected: value="1.11"
```

> **Device node change:** The v1.11 bootloader presents as `/dev/sda1` (partitioned device)
> instead of `/dev/sda` (whole device). Use `/dev/sda1` for all subsequent commands.

---

## Step 4 — Flash MSD+Debug Application

The `BOOTLOADER` drive is now at `/dev/sda1`. The previous `sync` mount is still active
if the drive has not been remounted. If it has been remounted by the system, remount it:

```bash
udisksctl unmount -b /dev/sda1
udisksctl mount -b /dev/sda1 --options sync
```

Copy the application firmware for your specific board:

```bash
# For FRDM-KL26Z:
cp Pemicro_OpenSDA_Debug_MSD_Update_Apps_2023_12_12/MSD-DEBUG-FRDM-KL26Z_Pemicro_v118.SDA \
   /run/media/$USER/BOOTLOADER/
sync
ls /run/media/$USER/BOOTLOADER/
# File must appear in listing before continuing
```

Then:

1. **Unplug the board.**
2. Wait 5 seconds.
3. Plug back in normally (no reset button).

Expected result — a drive named after the board appears:

```
$ lsusb | grep -i "p&e\|pemicro\|1357"
Bus 001 Device 009: ID 1357:0089 P&E Microcomputer Systems OpenSDA - CDC Serial Port

$ lsblk | grep FRDM
sda1   FRDM-KL26Z   /run/media/$USER/FRDM-KL26Z   vfat
```

> The drive is named `FRDM-KL26Z`, not `MBED`. P&E Micro firmware uses the board name.
> This is correct.

---

## Step 5 — Flash Your Application

The P&E Micro MSD firmware accepts **Motorola S-Record** (`.srec`) files.

Convert your ELF:

```bash
arm-none-eabi-objcopy -O srec your_program.elf your_program.srec
```

Flash:

```bash
cp your_program.srec /run/media/$USER/FRDM-KL26Z/
sync
```

Then unplug and replug normally. Firmware runs immediately — no reset needed.

### Example: LED blink (FRDM-KL26Z SDK)

```bash
# Build
cd SDK_2_2_0_FRDM-KL26Z/boards/frdmkl26z/driver_examples/gpio/led_output/armgcc
export ARMGCC_DIR=$(dirname $(dirname $(which arm-none-eabi-gcc)))
cmake -DCMAKE_TOOLCHAIN_FILE="../../../../../../tools/cmake_toolchain_files/armgcc.cmake" \
      -DCMAKE_POLICY_VERSION_MINIMUM=3.5 \
      -G "Unix Makefiles" \
      -DCMAKE_BUILD_TYPE=Debug .
make -j4

# Convert and flash
arm-none-eabi-objcopy -O srec debug/gpio_led_output.elf gpio_led_output.srec
cp gpio_led_output.srec /run/media/$USER/FRDM-KL26Z/
sync
```

---

## Applying This to Other FRDM Boards

The procedure is identical for any FRDM board with P&E Micro OpenSDA v1.

1. `BOOTUPDATEAPP_Pemicro_v111.SDA` — same file, works for all boards.
2. Choose the correct MSD+Debug firmware from `Pemicro_OpenSDA_Debug_MSD_Update_Apps_2023_12_12/`:

```
MSD-DEBUG-FRDM-K22F_Pemicro_v114.SDA
MSD-DEBUG-FRDM-K64F_Pemicro_v114.SDA
MSD-DEBUG-FRDM-KL25Z_Pemicro_v118.SDA
MSD-DEBUG-FRDM-KL26Z_Pemicro_v118.SDA
MSD-DEBUG-FRDM-KL46Z48M_Pemicro_v118.SDA
... (see full list in the directory)
```

3. After Step 4, the drive will be named after your board (e.g., `FRDM-K22F`).

---

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| File copied but bootloader ignores it | Drive not mounted with `sync` | Remount: `udisksctl unmount -b /dev/sdX && udisksctl mount -b /dev/sdX --options sync` |
| `LASTSTAT.TXT` still says `Ready.` after update attempt | Same as above | Same fix |
| Drive appears but version unchanged | Board did not run update app | Make sure you unplugged and replugged WITHOUT holding Reset |
| Drive does not appear at all after normal replug | Board has no application (APPVER 0.00) | This is normal — it only shows the drive in bootloader mode until an app is flashed |
| `udisksctl` says "not a mountable filesystem" | Bootloader v1.11 changed from `/dev/sda` to `/dev/sda1` | Use `/dev/sda1` instead of `/dev/sda` |
| `arm-none-eabi-objcopy` not found | Toolchain not installed | `sudo apt-get install binutils-arm-none-eabi` |

---

## USB ID Reference

| VID:PID | Description | Board state |
|---|---|---|
| `15a2:0038` | Freescale (original, no app) | Plugged in normally, APPVER 0.00 |
| `2504:0200` | Freescale OpenSDA MSD APP | Bootloader mode (any version) |
| `1357:0089` | P&E Micro OpenSDA CDC Serial | MSD+Debug firmware running — ready to flash |

---

## Hardware Notes (FRDM-KL26Z)

- **Target chip:** MKL26Z128VLH4 — Cortex-M0+, 128 KB Flash, 16 KB RAM
- **Debug chip:** MK20DX128 — OpenSDA v1
- **SDA port:** Use for all programming. Labeled `SDA` on silkscreen.
- **USB port:** Target USB only (for applications that use USB). Do not use for programming.
- The two ports are physically identical — check the silkscreen label carefully.
