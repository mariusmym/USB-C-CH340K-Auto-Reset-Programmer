# USB-C CH340K Auto-Reset Programmer

A compact USB-to-serial programmer for ESP-based boards, featuring a **CH340K** chip, **USB-C** connector, and a built-in **auto-reset circuit**.

![License: CC BY-SA 4.0](https://img.shields.io/badge/License-CC%20BY--SA%204.0-lightgrey.svg)
![Status: Active](https://img.shields.io/badge/Status-Active-brightgreen.svg)
![EDA: EasyEDA](https://img.shields.io/badge/EDA-EasyEDA-blue.svg)

<img width="500" height="375" alt="527034675_17967334817932042_1660775298333000321_n" src="https://github.com/user-attachments/assets/edaca671-a6d6-4dcf-949c-a02818f5f4d1" /> 

---

## Why This Exists

Many embedded or custom ESP-based projects don't need a dedicated USB-to-serial chip on the main PCB — especially when firmware uploads are infrequent. An external programmer is the practical solution.

The problem? Getting ESP chips into boot mode typically requires holding `IO0` low while toggling `EN` (reset). Without automation, you're left pressing buttons, shorting pads with tweezers, or other tedious workarounds.

This programmer solves that. Inspired by official Espressif development boards, it uses two NPN transistors to automatically control the reset (`EN`) and boot (`IO0`) pins via the serial adapter's `DTR` and `RTS` signals. Just plug in, hit upload, and the auto-reset circuit handles the rest.

## Features

- **CH340K** — USB-to-UART bridge IC with integrated clock (no external crystal needed), built-in anti-backflow protection
- **USB-C** connector — modern, reversible, robust
- **Auto-reset circuit** — two NPN transistors cross-wired to DTR/RTS, following the Espressif reference design
- **Compact form factor** — minimal footprint, easy to store and use
- **1×6 pin header** — simple connection to your ESP target board
- **3.3V compatible** — safe for ESP8266, ESP32, ESP32-S2, ESP32-S3, ESP32-C3, and similar chips

## How the Auto-Reset Works

The auto-reset circuit uses two NPN transistors in a cross-coupled configuration between the CH340K's `DTR#`/`RTS#` signals and the ESP's `EN`/`IO0` pins:

| DTR | RTS | EN (Reset) | IO0 (Boot) | Mode             |
|:---:|:---:|:----------:|:----------:|:-----------------|
|  1  |  1  |     1      |     1      | Normal operation |
|  0  |  0  |     1      |     1      | Normal operation |
|  1  |  0  |     0      |     1      | Chip in reset    |
|  0  |  1  |     1      |     0      | Boot mode        |

When `esptool.py` (or Arduino/PlatformIO) initiates an upload, it toggles DTR and RTS in sequence:
1. **Assert DTR=1, RTS=0** → pulls `EN` low (chip held in reset), `IO0` stays high
2. **Switch to DTR=0, RTS=1** → releases `EN` (chip starts), pulls `IO0` low (boot mode entered)
3. Firmware upload proceeds over UART
4. After upload, chip is reset into normal run mode

A small capacitor on the `EN` line ensures proper timing between the signal transitions.

## Pin Header Mapping

The 1×6 header connects to your ESP board as follows:

| Pin | Signal | Description                        |
|:---:|:------:|:-----------------------------------|
|  1  |  3V3   | 3.3V power output                  |
|  2  |  GND   | Ground                             |
|  3  |  TX    | Programmer TX → ESP RX             |
|  4  |  RX    | Programmer RX → ESP TX             |
|  5  |  DTR   | Auto-reset: connected to EN circuit |
|  6  |  RTS   | Auto-reset: connected to IO0 circuit|

> **Important:** TX and RX are labeled from the programmer's perspective. Connect the programmer's **TX → ESP's RX** and **RX → ESP's TX** (crossover).

## Compatible Target Boards

This programmer works with any ESP chip that uses the standard EN + IO0 boot mode selection:

- ESP8266
- ESP32
- ESP32-S2
- ESP32-S3
- ESP32-C3
- ESP32-C6
- ESP32-H2
- Any other board exposing a UART header with EN and IO0/BOOT pins

## Getting Started

### 1. Install the CH340 Driver

Most modern operating systems include CH340 drivers. If your system doesn't recognize the programmer:

- **Windows:** Download from [WCH's official site](http://www.wch.cn/download/CH341SER_EXE.html)
- **macOS:** Download from [WCH's official site](http://www.wch.cn/download/CH341SER_MAC_ZIP.html) or install via Homebrew: `brew install --cask wch-ch34x-usb-serial-driver`
- **Linux:** Driver is typically included in the kernel (`ch341` module). Verify with `lsmod | grep ch341`

### 2. Connect to Your ESP Board

Wire the 1×6 header to your target board's programming header. Double-check TX/RX crossover and ensure the target board has a capacitor (1–10 µF) between EN and GND for reliable auto-reset timing.

### 3. Upload Firmware

Use your preferred tool:

**Arduino IDE:**
- Select the correct board and port
- Click Upload — auto-reset handles boot mode entry

**PlatformIO:**
- Configure `upload_port` in `platformio.ini` if needed
- Run `pio run --target upload`

**esptool.py (direct):**
```bash
esptool.py --chip esp32 --port /dev/ttyUSB0 --baud 460800 write_flash -z 0x1000 firmware.bin
```

## Schematic & PCB

The full schematic and PCB layout are designed in **EasyEDA** and published on OSHWLab:

**→ [Open project on OSHWLab](https://oshwlab.com/mariusmym/usb-c-ch340k_auto-reset_programmer)**

From there you can:
- View and clone the schematic and PCB
- Export Gerber files for fabrication
- Download the BOM for component sourcing
- Order PCBs directly via JLCPCB


## 3D Printable Accessories

To keep the pin header stable during programming, you can 3D print a clip or holder:

**→ [Modular Programming Clip on Printables](https://www.printables.com/model/378005-modular-programming-clip)**

## Troubleshooting

| Problem | Solution |
|:--------|:---------|
| Device not recognized | Install CH340 driver. On Linux, check `dmesg` for USB errors. |
| Upload fails with "Timed out waiting for packet header" | Ensure EN has a 1–10 µF cap to GND on target board. Check TX/RX crossover. |
| Upload works only when holding BOOT button | Verify DTR/RTS wiring to the transistor circuit. Check solder joints on Q1/Q2. |
| Erratic resets during serial monitoring | Some serial monitors assert DTR/RTS on connect. Disable hardware flow control in your terminal. On Linux: `stty -F /dev/ttyUSB0 -hupcl` |
| Port disappears intermittently | Try a different USB-C cable (some cables are charge-only with no data lines). |

## Design Notes

- The **CH340K** variant was chosen specifically because it has an integrated oscillator (no external crystal), built-in anti-backflow protection, and a compact ESSOP-10 footprint.
- The auto-reset circuit follows the same topology used on official Espressif DevKitC boards, ensuring broad compatibility with `esptool.py` and all major IDEs.
- USB-C was chosen over Micro-USB for durability, reversibility, and future-proofing.

## Related Projects

- [WLED 2024 Mini P](https://oshwlab.com/mariusmym/wled2024_mini_p) — an ESP-based LED controller board designed to work with this programmer

## License

This project is licensed under **[CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/)**.

You are free to share and adapt this design, provided you give appropriate credit and distribute any derivative works under the same license.
