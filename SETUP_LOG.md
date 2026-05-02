# FRDM-KL26Z Setup Log
**Date:** 2026-05-02  
**Mac User:** ashokpaudelapril  
**Goal:** Get FRDM-KL26Z working with VS Code on Mac (build, flash, debug)

---

## What Is Installed (Mac)

| Tool | Version | Status |
|---|---|---|
| arm-none-eabi-gcc | 15.2.1 (Arm GNU Toolchain 15.2.Rel1) | ✅ Installed |
| CMake | 4.3.0 | ✅ Installed |
| Make | 3.81 | ✅ Installed |
| pyOCD | 0.44.0 | ✅ Installed via pipx |
| pyOCD device pack | MKL26Z128xxx4 | ✅ Installed |
| LinkServer | 26.3.123 | ✅ Installed at /Applications/LinkServer_26.3.123/ |
| MCUXpresso for VS Code | latest | ✅ Installed (VS Code extension by NXP) |
| NXP SDK | SDK_2_2_0_FRDM-KL26Z | ✅ Located at ~/FRDM-KL26Z/SDK_2_2_0_FRDM-KL26Z/ |

---

## SDK Location
```
~/FRDM-KL26Z/SDK_2_2_0_FRDM-KL26Z/
```

## LED Example (Already Built Successfully)
```
~/FRDM-KL26Z/SDK_2_2_0_FRDM-KL26Z/boards/frdmkl26z/driver_examples/gpio/led_output/armgcc/
```
Built ELF file is at:
```
~/FRDM-KL26Z/SDK_2_2_0_FRDM-KL26Z/boards/frdmkl26z/driver_examples/gpio/led_output/armgcc/debug/gpio_led_output.elf
```
Build command used:
```bash
export ARMGCC_DIR=$(dirname $(dirname $(which arm-none-eabi-gcc)))
cmake -DCMAKE_TOOLCHAIN_FILE="../../../../../../tools/cmake_toolchain_files/armgcc.cmake" \
      -DCMAKE_POLICY_VERSION_MINIMUM=3.5 \
      -G "Unix Makefiles" \
      -DCMAKE_BUILD_TYPE=Debug .
make -j4
```

---

## The Problem — OpenSDA Firmware

### Board State
- **OpenSDA chip:** MK20DX128 (on the FRDM-KL26Z debug side)
- **Bootloader version:** 1.09 (P&E Micro)
- **Application version:** 0.00 (NO application installed)
- **Root cause:** OpenSDA bootloader v1.09 does NOT support Mac file transfers (Mac support added in v1.10)

### Why Nothing Works on Mac
- pyOCD cannot detect the board (no CMSIS-DAP application on OpenSDA)
- LinkServer cannot detect the board (same reason)
- Dragging .SDA files on Mac fails silently due to bootloader v1.09 Mac incompatibility

---

## What Needs to Be Done on Windows

### Files Ready (already on Mac, transfer to Windows)
```
~/FRDM-KL26Z/BOOTUPDATEAPP_Pemicro_v111.SDA         ← Update bootloader first
~/FRDM-KL26Z/Pemicro_OpenSDA_Debug_MSD_Update_Apps_2023_12_12/MSD-DEBUG-FRDM-KL26Z_Pemicro_v118.SDA  ← Then flash this
```

### Step 1 — Update Bootloader to v1.11
1. Hold Reset button → plug board into OpenSDA port → release Reset
2. BOOTLOADER drive appears in Windows Explorer
3. Drag `BOOTUPDATEAPP_Pemicro_v111.SDA` onto BOOTLOADER drive
4. Wait for copy to complete fully
5. Unplug board, wait 3 seconds, plug back in normally (no reset)
6. Wait up to 15 seconds — board updates itself
7. BOOTLOADER drive reappears automatically
8. Open `SDA_INFO.HTM` — confirm BOOTVER = **1.11**

### Step 2 — Flash MSD+Debug Firmware
1. Enter bootloader mode again (Hold Reset → plug in → release)
2. Drag `MSD-DEBUG-FRDM-KL26Z_Pemicro_v118.SDA` onto BOOTLOADER drive
3. Wait for copy to complete
4. Unplug, wait 3 seconds, plug back in normally
5. **MBED drive should appear in Windows Explorer** ✅
6. LED should be solid green or slow blink

### Step 3 — Verify on Mac
Plug into Mac (OpenSDA port), run:
```bash
system_profiler SPUSBDataType | grep -A 5 "Manufacturer"
```
Should now show a different Vendor ID (0x0d28 for mbed/DAPLink) instead of 0x15a2

---

## After Windows Fix — Back on Mac

Once MBED drive appears, continue setup:

### 1. Test pyOCD detects the board
```bash
pyocd list
```
Should show the board as a CMSIS-DAP probe.

### 2. Flash the LED example
```bash
cd ~/FRDM-KL26Z/SDK_2_2_0_FRDM-KL26Z/boards/frdmkl26z/driver_examples/gpio/led_output/armgcc
pyocd flash --target MKL26Z128xxx4 debug/gpio_led_output.elf
```

### 3. Set up VS Code tasks.json + launch.json
Still to be done — Claude Code can help with this after flashing works.

---

## Board Hardware Notes
- **Target chip:** MKL26Z128VLH4 (Cortex-M0+, 128KB Flash, 16KB RAM)
- **Debug chip:** MK20DX128 (OpenSDA v1)
- **OpenSDA USB port:** labeled `SDA` on board silkscreen — use THIS for programming
- **KL26Z USB port:** labeled `USB` — for target USB applications only
- **Two USB ports look identical** — make sure to use the correct one

---

## USB Port Detection (Mac)
To verify correct port is connected:
```bash
system_profiler SPUSBDataType | grep -A 5 "Manufacturer"
```
- Shows `Freescale` → correct OpenSDA port, but no app installed
- Shows nothing → wrong port or wrong cable (use data cable, not charge-only)

---

## Notes
- Factory demo (tilt/color LED) is still running on the KL26Z target — board is fine
- The OpenSDA chip issue is purely firmware, not hardware damage
- Once MSD firmware is flashed, drag-and-drop flashing will work on both Mac and Windows
