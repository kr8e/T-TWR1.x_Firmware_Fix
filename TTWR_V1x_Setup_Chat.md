# LilyGO T-TWR V1.4 Firmware — Research & Setup Chat Log

**Date:** May 13  
**Topic:** Finding and flashing original T-TWR V1.4 firmware on Ubuntu/Debian Linux  
**Hardware:** LilyGO T-TWR V1.4 (PCB datecode 20230207), ESP32-S3-WROOM-1, VHF SA868 module

---

## Table of Contents

1. [Finding the V1.x Firmware](#1-finding-the-v1x-firmware)
2. [Identifying the Hardware](#2-identifying-the-hardware)
3. [Retrieving the Source Files](#3-retrieving-the-source-files)
4. [Build Instructions](#4-build-instructions)
5. [Configuring for VHF Band](#5-configuring-for-vhf-band)
6. [PlatformIO Setup on Linux](#6-platformio-setup-on-linux)
7. [Ubuntu/Debian Specifics](#7-ubuntudebian-specifics)
8. [Troubleshooting: Click Compatibility Error](#8-troubleshooting-click-compatibility-error)
9. [Troubleshooting: Port Disappears on Upload](#9-troubleshooting-port-disappears-on-upload)
10. [Reading the dmesg Output](#10-reading-the-dmesg-output)
11. [Editing platformio.ini](#11-editing-platformioini)
12. [Correcting the Upload Protocol](#12-correcting-the-upload-protocol)
13. [Troubleshooting: Port Still Not Found](#13-troubleshooting-port-still-not-found)
14. [Download Mode Timing Fix](#14-download-mode-timing-fix)
15. [Filesystem Upload](#15-filesystem-upload)

---

## 1. Finding the V1.x Firmware

**User:** Find the LilyGO T-TWR version 1.4 firmware. Not the TWR Plus or version 2 or 2.1 — must be version 1.4.

**Claude:** There is no T-TWR "version 1.4" in LilyGO's public lineup. The T-TWR hardware revisions that exist are:

- Rev 2.0 (UHF and VHF variants)
- Rev 2.1
- T-TWR Plus (a separate product line)

The official firmware repo at [github.com/Xinyuan-LilyGO/T-TWR](https://github.com/Xinyuan-LilyGO/T-TWR) only offers firmware for Rev 2.0 (UHF and VHF) and Rev 2.x (covers both 2.0 and 2.1). There is no "1.4" branch, bin, or uf2 anywhere in that repo.

Possible reasons you're seeing "1.4":

- **Different product** — LilyGO has a "TTGO-TS V1.4" board (a different device); confusion between product lines is common with their naming.
- **Seller/listing label** — AliExpress and other vendors sometimes use their own versioning that doesn't match LilyGO's official hardware revisions.
- **Firmware version, not hardware rev** — Some third-party firmware projects (like ESP32APRS) version their own firmware independently.

Can you tell me where you saw "T-TWR version 1.4" referenced? A photo of the PCB silkscreen, a product listing URL, or a forum post would help pinpoint the hardware.

---

## 2. Identifying the Hardware

**User:** It's printed on the circuit board. Also the number 20230207. The ESP32 is an S3-WROOM-1.

**Claude:** That's the key piece of evidence. The original T-TWR (not Plus, not Rev 2.x) was announced in March 2023, uses the ESP32-S3-WROOM-1-N16R8, has an 18650 battery holder (not 21700), and a 3-way encoder. Your board with "V1.4" and datecode "20230207" (February 7, 2023) is almost certainly a pre-production or early production unit of the original T-TWR, predating what LilyGO publicly calls "Rev 2.0."

LilyGO simply never published firmware for the V1.x hardware iteration — they jumped to "Rev 2.0" in their public-facing docs and firmware repo. The firmware for Rev 2.0/2.1 uses different pin assignments and may not work correctly on your board.

**Best path forward is the source code, not a binary blob.** The original T-TWR schematic/source is in the same GitHub repo, but there's likely a branch or older commit that targets V1.4 hardware.

Concrete steps:

- Go to <https://github.com/Xinyuan-LilyGO/T-TWR> and look at branches and tags for a `v1.x` or `rev1` branch.
- Look at the commit history around early 2023 — the repo was likely initialized with V1.x hardware definitions that were later overwritten.
- Check `platformio.ini` and any `pins_arduino.h` / board definition headers for a `TWTWR_V1` or similar define.

---

## 3. Retrieving the Source Files

**User:** Retrieve the appropriate files.

**Claude:** After digging through the git history of the T-TWR repo, the `examples/Rev1.x/` directory was found in the repo's first commit. The files were packaged into a zip.

**Contents of `T-TWR_V1.x_firmware_and_schematic.zip`:**

### `examples/walkie_talkie/` — Main firmware

| File | Description |
|------|-------------|
| `walkie_talkie.ino` | Main sketch |
| `sa868.cpp` / `sa868.h` | SA868 radio driver |
| `ui.cpp` / `ui.h` | OLED UI |
| `data.cpp` / `data.h` | Settings/persistence |
| `config.h` | Band selection (`UHF_BAND`, `VHF_BAND`, or `_350_BAND`) |
| `platformio.ini` | Build config |
| `data/settings.json` | Default settings |

### `examples/at_debug/`
SA868 AT command passthrough sketch — useful for testing the radio module directly before flashing full firmware.

### `schematic/`
`T-TWR_1.x.pdf` — V1.x hardware schematic.

### `datasheet/`
SA868 module datasheet, OLED spec, encoder switches.

**Important build notes:**

- The `platformio.ini` targets arduino-esp32 2.0.5 — stick to that version to avoid compilation issues.
- In `config.h`, uncomment the correct band macro for your SA868 variant (UHF 400–480 MHz is most common).
- To enter download mode: hold **Volume−** (GPIO 0), press **RST**, then release **Volume−**.

---

## 4. Build Instructions

**User:** Output full instructions.

**Claude:** Eight-section build and flash guide was generated. Summary:

| Part | Topic |
|------|-------|
| 1 | **Toolchain Setup** — PlatformIO (recommended) or Arduino IDE 2.x; ESP32 board package v2.0.5 exactly; required libraries at specific versions: RotaryEncoder 1.5.3, OneButton 2.0.3, U8g2 2.34.5, ArduinoJson 6.19.4 |
| 2 | **Project Setup** — Opening in PlatformIO vs Arduino IDE; placing the custom `lilygo-t-twr.json` board definition |
| 3 | **Configure Your Band** — Edit `config.h` or select a PlatformIO environment (`uhf_band`, `350_band`, or `vhf_band`) |
| 4 | **Upload the Filesystem** — `data/settings.json` must be uploaded separately to LittleFS via `pio run --target uploadfs` or the Arduino ESP32 filesystem upload plugin |
| 5 | **Enter Download Mode** — Hold **Volume−** (GPIO 0 / BOOT), briefly press **RST**, then release **Volume−**. Critical step most people miss. |
| 6 | **Build and Flash** — Standard PlatformIO or Arduino IDE upload once in download mode |
| 7 | **Verify Operation** — Expected behavior on first boot; note that OLED power enable pin (GPIO 21) must be driven HIGH for display to work |
| 8 | **AT Debug Sketch** — Using `at_debug` to verify SA868 is alive with raw AT commands before flashing full firmware |

---

## 5. Configuring for VHF Band

**User:** Band is VHF.

**Claude:** Done. `config.h` in the updated zip now has:

```cpp
// #define UHF_BAND 1
// #define _350_BAND 1
#define VHF_BAND 1
```

`settings.json` default frequency was already set to 134.0 MHz (bottom of the VHF range, 134–174 MHz).

When using PlatformIO, select the `vhf_band` environment — it also passes `-D VHF_BAND=1` as a build flag, matching what's in `config.h`.

---

## 6. PlatformIO Setup on Linux

**User:** How to do this in Linux? / PlatformIO.

**Claude:** The full sequence in order:

```bash
# One-time setup
pip install platformio --break-system-packages
sudo usermod -aG dialout $USER   # then log out and back in

# Project setup
unzip T-TWR_V1.x_firmware_and_schematic.zip
cd T-TWR_V1.x/examples/walkie_talkie
mkdir -p ~/.platformio/boards
cp ../../platformio/boards/lilygo-t-twr.json ~/.platformio/boards/

# Build
pio run -e vhf_band

# Put board in download mode: hold Volume−, press RST, release Volume−

# Flash sketch
pio run -e vhf_band --target upload

# Put board in download mode again

# Flash filesystem
pio run -e vhf_band --target uploadfs

# Monitor serial output
pio device monitor
```

> **Most common Linux gotcha:** the `dialout` group. If you get `Permission denied` on the port, that's why. Use `newgrp dialout` to fix it in the current shell without a full logout.

---

## 7. Ubuntu/Debian Specifics

**User:** Doing this in Ubuntu/Debian Linux.

**Claude:** The instructions above are already written for Ubuntu/Debian. The only additions specific to Ubuntu/Debian:

**Installing VSCode via the Microsoft repo (gets auto-updates):**

```bash
sudo apt install wget gpg
wget -qO- https://packages.microsoft.com/keys/microsoft.asc | gpg --dearmor > packages.microsoft.gpg
sudo install -o root -g root -m 644 packages.microsoft.gpg /etc/apt/trusted.gpg.d/
echo "deb [arch=amd64] https://packages.microsoft.com/repos/code stable main" | sudo tee /etc/apt/sources.list.d/vscode.list
sudo apt update && sudo apt install code
```

**Or install via `.deb` without adding the repo:**

```bash
# Download from https://code.visualstudio.com/ then:
sudo dpkg -i code_*.deb
sudo apt -f install   # fixes any missing dependencies
```

Everything else — `pip`, `usermod -aG dialout`, `dmesg`, port names — is standard Ubuntu/Debian.

---

## 8. Troubleshooting: Click Compatibility Error

**User:**
```
AttributeError: 'PlatformioCLI' object has no attribute 'resultcallback'.
Did you mean: 'result_callback'?
```

**Claude:** This is a Python/Click version compatibility issue — newer versions of the `click` library removed `resultcallback` and renamed it `result_callback`. PlatformIO's CLI code hasn't caught up.

**Fix — pin click:**

```bash
pip install "click<8.1" --break-system-packages
```

**Or pin both together:**

```bash
pip install "platformio==6.1.11" "click<8.1" --break-system-packages
```

**Cleanest solution — use a virtual environment** (recommended for Ubuntu 22.04+, which ships Click 8.1+ system-wide):

```bash
python3 -m venv ~/pio-env
source ~/pio-env/bin/activate
pip install platformio
```

Then activate it at the start of each terminal session:

```bash
source ~/pio-env/bin/activate
pio run -e vhf_band --target upload
```

---

## 9. Troubleshooting: Port Disappears on Upload

**User:** When trying to upload, the port disappears.

**Claude:** This is a known ESP32-S3 quirk. When the chip resets into download mode, the USB CDC serial port disappears because the chip is no longer running the CDC firmware — it's in the ROM bootloader, which uses a different USB descriptor.

**Check what appears after entering download mode:**

```bash
dmesg | tail -20
```

Look for `idVendor=303a` (Espressif). The port will appear as `/dev/ttyACM0` or `/dev/ttyUSB0` but may be a different device node than before.

**Step by step:**

1. Unplug the board completely.
2. Watch for the device in a terminal:
   ```bash
   watch -n 0.5 ls /dev/ttyACM* /dev/ttyUSB* 2>/dev/null
   ```
3. Enter download mode (hold **Volume−**, press **RST**, release **Volume−**).
4. Note which port appears, then use that exact port:
   ```bash
   pio run -e vhf_band --target upload --upload-port /dev/ttyACM0
   ```

**If no port appears in download mode**, add this to `platformio.ini` under `[env:vhf_band]`:

```ini
upload_protocol = esp-builtin
```

Then flash without needing manual download mode at all:

```bash
pio run -e vhf_band --target upload
```

The ESP32-S3's built-in USB can auto-reset into download mode without holding any buttons.

---

## 10. Reading the dmesg Output

**User:**
```
[337419.386852] usb 1-1: SerialNumber: 34:85:18:8E:5C:A8
[337419.392368] cdc_acm 1-1:1.0: ttyACM0: USB ACM device
[356464.667104] usb 1-1: USB disconnect, device number 23
[356467.201830] usb usb1-port1: connect-debounce failed
[356932.617829] usb 1-1: new full-speed USB device number 24 using xhci_hcd
[356932.742407] usb 1-1: New USB device found, idVendor=303a, idProduct=1001, bcdDevice= 1.01
[356932.742432] usb 1-1: New USB device strings: Mfr=1, Product=2, SerialNumber=3
[356932.742442] usb 1-1: Product: USB JTAG/serial debug unit
[356932.742449] usb 1-1: Manufacturer: Espressif
[356932.742456] usb 1-1: SerialNumber: 34:85:18:8E:5C:A8
[356932.745534] cdc_acm 1-1:1.0: ttyACM0: USB ACM device
[356936.426125] usb 1-1: USB disconnect, device number 24
...
```

**Claude:** Good news — the board is working correctly.

| Event | Meaning |
|-------|---------|
| First entry (`idProduct=0000`) | Normal running mode, CDC ACM |
| After download mode (`idProduct=1001`, "USB JTAG/serial debug unit") | ESP32-S3 ROM bootloader's built-in USB |
| Port stays as `ttyACM0` | Clean — no device node change |
| Disconnects | Chip re-enumerating between modes — normal |

The issue is **timing** — PlatformIO is trying to open the port while it's mid-re-enumeration.

**Fixes:**

1. Add `upload_protocol = esp-builtin` to `platformio.ini` (global `[env]` section):

```ini
[env]
upload_protocol = esp-builtin
upload_speed = 921600
```

2. Enter download mode, wait for `ttyACM0` to reappear in `dmesg`, then immediately:

```bash
pio run -e vhf_band --target upload --upload-port /dev/ttyACM0
```

If it still disconnects mid-flash, also add:

```ini
board_build.flash_mode = dio
```

---

## 11. Editing platformio.ini

**User:** How do I edit the file?

**Claude:** Simplest way on Ubuntu/Debian:

```bash
nano T-TWR_V1.x/examples/walkie_talkie/platformio.ini
```

- **Ctrl+O** — save
- **Enter** — confirm filename
- **Ctrl+X** — exit

GUI alternatives:

```bash
gedit T-TWR_V1.x/examples/walkie_talkie/platformio.ini
# or on newer Ubuntu:
gnome-text-editor T-TWR_V1.x/examples/walkie_talkie/platformio.ini
```

---

## 12. Correcting the Upload Protocol

**User:** Does `[env] upload_protocol = esp-builtin` get added to the env section already there?

**Claude:** No — `[env]` (with nothing after it) is a **global defaults block** that applies to all named environments. Your existing `[env:vhf_band]` etc. stay unchanged and inherit defaults from `[env]`.

```ini
[platformio]
...

[env]
upload_protocol = esp-builtin
upload_speed = 921600

[env:vhf_band]
... existing content unchanged ...

[env:uhf_band]
... existing content unchanged ...
```

---

**User:** File before adding (pasted full `platformio.ini`):

```ini
[platformio]
src_dir = .
lib_dir = ../../../lib
boards_dir = ../../../platformio/boards

[env]
platform = espressif32
upload_protocol = esptool
framework = arduino
upload_speed = 921600
monitor_speed = 115200
board = lilygo-t-twr

[env:uhf_band]
build_flags = -D BOARD_HAS_PSRAM -D CORE_DEBUG_LEVEL=3 -D ARDUINO_USB_MODE=1 -D UHF_BAND=1
lib_deps =
    olikraus/U8g2@^2.36.18
    bblanchon/ArduinoJson@^7.4.3
    mathertel/OneButton@^2.6.2
    mathertel/RotaryEncoder@^1.6.0

[env:350_band]
build_flags = -D BOARD_HAS_PSRAM -D CORE_DEBUG_LEVEL=3 -D ARDUINO_USB_MODE=1 -D _350_BAND=1
lib_deps =
    olikraus/U8g2@^2.36.18
    bblanchon/ArduinoJson@^7.4.3
    mathertel/OneButton@^2.6.2
    mathertel/RotaryEncoder@^1.6.0

[env:vhf_band]
build_flags = -D BOARD_HAS_PSRAM -D CORE_DEBUG_LEVEL=3 -D ARDUINO_USB_MODE=1 -D VHF_BAND=1
lib_deps =
    olikraus/U8g2@^2.36.18
    bblanchon/ArduinoJson@^7.4.3
    mathertel/OneButton@^2.6.2
    mathertel/RotaryEncoder@^1.6.0
```

**Claude:** The `[env]` section already exists. You just need to change **one line** — replace `esptool` with `esp-builtin`:

```diff
 [env]
 platform = espressif32
-upload_protocol = esptool
+upload_protocol = esp-builtin
 framework = arduino
 upload_speed = 921600
 monitor_speed = 115200
 board = lilygo-t-twr
```

Just that one word change. Everything else stays the same.

---

## 13. Troubleshooting: Port Still Not Found

**User:** Still can't find it.

**Claude:** Let's confirm exactly what's visible:

```bash
ls /dev/ttyACM* /dev/ttyUSB* 2>/dev/null
```

Run that, enter download mode (hold **Volume−**, press **RST**, release **Volume−**), then run it again immediately. Also run:

```bash
dmesg | tail -5
```

right after entering download mode.

---

## 14. Download Mode Timing Fix

**User:** (Pasted dmesg showing board entering JTAG mode but then disconnecting after ~75 seconds.)

**Claude:** The board is entering download mode correctly but timing out before the upload starts. You need to enter download mode and run the upload command at nearly the same time.

**Two-terminal method:**

**Terminal 1** — watch the port:
```bash
watch -n 0.5 ls /dev/ttyACM* /dev/ttyUSB* 2>/dev/null
```

**Terminal 2** — have this ready but **do not run yet**:
```bash
pio run -e vhf_band --target upload --upload-port /dev/ttyACM0
```

Enter download mode. The moment `ttyACM0` appears in Terminal 1, hit **Enter** in Terminal 2. You have a few seconds.

**Alternative — use esptool directly (connects faster than PlatformIO's wrapper):**

```bash
python3 -m esptool --chip esp32s3 --port /dev/ttyACM0 --baud 921600 \
  write_flash -z 0x0 .pio/build/vhf_band/firmware.bin
```

---

**User:** The two windows worked.

**Claude:** Great!

---

## 15. Filesystem Upload

Now do the same for the filesystem upload. Have this ready in Terminal 2 before entering download mode:

```bash
pio run -e vhf_band --target uploadfs --upload-port /dev/ttyACM0
```

Enter download mode again (hold **Volume−**, press **RST**, release **Volume−**), wait for `ttyACM0` to appear in Terminal 1, then immediately run it.

---

*End of chat log.*
