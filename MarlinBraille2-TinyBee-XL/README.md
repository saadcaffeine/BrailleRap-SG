# MarlinBraille2 — MKS TinyBee v1.0 / BrailleRAP XL (Singapore build)

Firmware configuration for the [Singapore BrailleRAP XL](https://saadcaffeine.github.io/BrailleRap-SG/) (A3, 297 × 420 mm) running on an
**MKS TinyBee v1.0** (ESP32) with **TMC2209 standalone** drivers, with **ESP3D WiFi** enabled.

Tested working 2026-11-25: USB control at 250000 baud, homing, endstops, and motor control
from the ESP3D web interface over WiFi.

## Base

Firmware fork with these changes committed: [saadcaffeine/MarlinBraille2](https://github.com/saadcaffeine/MarlinBraille2/tree/xl-sg), branch `xl-sg`.
To build from it directly: `git clone -b xl-sg https://github.com/saadcaffeine/MarlinBraille2.git`.

- Firmware source: [braillerap/MarlinBraille2](https://github.com/braillerap/MarlinBraille2),
  branch `BrailleRAP`, commit `0673683ef0` (Marlin 2.1.1.2)
- Board config started from [braillerap/MarlinBraille2Configuration](https://github.com/braillerap/MarlinBraille2Configuration)
  `Firmware/configuration/MKSTinyBee/`

## Contents

| Path | Copy to (in MarlinBraille2) |
|---|---|
| `Marlin/Configuration.h` | `Marlin/Configuration.h` |
| `Marlin/Configuration_adv.h` | `Marlin/Configuration_adv.h` |
| `ini/features.ini` | `ini/features.ini` |
| `xl-sg.patch` | the same changes as a `git diff` against `0673683ef0` — apply with `git apply xl-sg.patch` |

## Changes vs upstream

> The configuration files and `xl-sg.patch` in this folder are from the first WiFi build. Later changes (SD support, paper sensor polarity, Y direction, paper-out and cancel handling, power-on self test) are in the firmware fork. For the current files, use [`xl-sg-v1.0.0`](https://github.com/saadcaffeine/MarlinBraille2/tree/xl-sg-v1.0.0).


| File | Change | Why |
|---|---|---|
| `Configuration.h` | `DEFAULT_AXIS_STEPS_PER_UNIT { 80, 43.5, 400, 500 }` | Upstream `87/2` is integer division in C → **43**, ~1.2 % short on Y (≈5 mm over an A3 sheet). |
| `Configuration.h` | `#define SERIAL_PORT_2 -1` | Gives ESP3D its own serial channel so the web UI receives Marlin's replies. Without it the WebUI shows "connection error". |
| `Configuration_adv.h` | `#define ESP3D_WIFISUPPORT` | Re-enables WiFi (disabled upstream on 2026-02-04 because of a library conflict). |
| `Configuration_adv.h` | `#define WIFI_CUSTOM_COMMAND` | Passes `[ESPxxx]` setup commands to ESP3D. |
| `ini/features.ini` | `me-no-dev/AsyncTCP@1.1.1, me-no-dev/ESP Async WebServer@1.2.3` | The unpinned libraries resolve to AsyncTCP 3.x / ESP Async WebServer 3.x, which need a newer ESP32 core than `espressif32@2.1.0` (Arduino-ESP32 1.0.4) and fail with `esp_idf_version.h: No such file`. This is the "lib conflict" behind the upstream WiFi disable. |

Note: `HOMING_FEEDRATE_MM_M` is `50*60` here (the configuration repo uses `20*60`).

## Build and flash (Windows, PlatformIO)

```powershell
git clone -b BrailleRAP https://github.com/braillerap/MarlinBraille2.git
cd MarlinBraille2
git checkout 0673683ef0
git apply ..\BrailleRap-SG\MarlinBraille2-TinyBee-XL\xl-sg.patch

$pio = "$env:USERPROFILE\.platformio\penv\Scripts\pio.exe"
& $pio run -e mks_tinybee -t upload --upload-port COM5
& $pio device monitor -p COM5 -b 250000 --echo
```

- The TinyBee enumerates as **USB-SERIAL CH340** (VID:PID `1A86:7523`).
- If the library pins were changed after a previous build, delete `.pio\libdeps\mks_tinybee` first.
- If the board boot-loops with `No MD5 found in partition table`, erase it and do a full upload:
  `python %USERPROFILE%\.platformio\packages\tool-esptoolpy\esptool.py -p COM5 erase_flash`
- Back up before flashing: `esptool.py -p COM5 -b 460800 read_flash 0 0x800000 backup.bin` (8 MB flash).
- After flashing: `M502`, `M500`, then check `M503` shows `M92 X80.00 Y43.50`.

## Wiring notes

- **Motors:** coil pairs must be on pins 1–2 and 3–4 of the TinyBee motor header. Our cables needed the
  two middle wires swapped (symptom: motor buzzes / jitters back and forth without turning).
- **Endstops** (internal pull-ups, use signal + GND only):
  - X → `X-MIN` (IO33), switch **C + NC** (`X_MIN_ENDSTOP_INVERTING false`)
  - Y → `Y-MIN` (IO32) is the **paper sensor**, switch **C + NO** with `Y_MIN_ENDSTOP_INVERTING false`. `M119` shows `y_min: TRIGGERED` with **no paper** and `open` with a sheet at the lever. That's what the paper-loading routine expects: it feeds forward while triggered, then backs up to find the paper edge. Fixed in `aad810f`; before that, loading drove the sheet backwards.
  - Z-MIN is unused and reads `TRIGGERED` — harmless.
- **Electromagnet:** heated-bed (HB) output. `M3 S255` = on, `M3 S0` = off.

## WiFi (ESP3D)

- Boots as an access point at **192.168.0.1** (default password `12345678`).
- Web UI: upload `index.html.gz` from **ESP3D-WEBUI 2.1**
  (`https://raw.githubusercontent.com/luc-github/ESP3D-WEBUI/2.1/index.html.gz`) via
  `http://192.168.0.1/?forcefallback=yes`. WebUI 3.x does not work with ESP3DLib 1.0.x.
- Join an existing network from the serial monitor:
  ```
  [ESP100]YourSSID
  [ESP101]YourPassword
  [ESP110]STA
  [ESP444]RESTART
  ```

- **Optional on-device interface:** upload `braille.html.gz` from [BrailleRapSGUI](https://github.com/saadcaffeine/BrailleRapSGUI) to the same ESP3D flash as `index.html.gz`, then open `http://192.168.0.1/braille.html` from any phone or laptop. You get UEB grade 1 printing over WiFi via the SD card, status lights, progress, cancel and emergency stop. It needs `SDSUPPORT` in the firmware.

## Host software

- [AccessBrailleRAP](https://github.com/braillerap/AccessBrailleRAP/releases) V0.9.0 — USB/COM port only. Set port COM5, paper A3 (297 × 420).
- [DesktopBrailleRAP](https://github.com/braillerap/DesktopBrailleRAP/releases) for tactile graphics.
