---
weight: 2
---

# Firmware Setup

The firmware is a PlatformIO + Arduino project targeting the ESP32-S3.

---

## Prerequisites

- Python 3.8+ with `pip`
- Git
- A USB-C cable connected to the ESP32-S3 dev board
- The `99-platformio-udev.rules` file installed (Linux only)

### Install udev rules (Linux)

Without these rules the device port won't get the right permissions and `esptool` will fail:

```bash
curl -fsSL https://raw.githubusercontent.com/platformio/platformio-core/develop/platformio/assets/system/99-platformio-udev.rules \
  | sudo tee /etc/udev/rules.d/99-platformio-udev.rules

sudo udevadm control --reload-rules
sudo udevadm trigger
```

Unplug and replug the board after running this.

---

## Project setup

```bash
git clone https://github.com/Rishi-k-s/parayu
cd parayu

# Create a virtual environment for PlatformIO
python3 -m venv .venv
source .venv/bin/activate   # Windows: .venv\Scripts\activate

# Install PlatformIO
pip install platformio

# Install the ESP32 platform + all libraries
pio pkg install
```

`pio pkg install` downloads:
- `espressif32` platform + xtensa toolchain
- `WiFiManager` (ESP32 captive-portal provisioning)
- `WebSockets` by links2004 (WebSocket client)
- `Adafruit SSD1306` + `Adafruit GFX Library` (OLED driver)

---

## Build

```bash
pio run
```

A successful build ends with:

```
RAM:   [==        ]  15.0% (used 49192 bytes from 327680 bytes)
Flash: [===       ]  30.8% (used 1029013 bytes from 3342336 bytes)
========================= [SUCCESS] Took 8.33 seconds =========================
```

---

## Flash

### Find the port

After connecting the board:

```bash
ls /dev/ttyACM*   # Linux
ls /dev/cu.*      # macOS
```

The port is set in `platformio.ini`:

```ini
upload_port  = /dev/ttyACM0
monitor_port = /dev/ttyACM0
```

Change `ttyACM0` to match your system if different.

### Bootloader mode

The **first flash** on a brand-new board requires manual bootloader entry:

1. Hold the **BOOT** button on the board.
2. Press and release **RESET** (or briefly connect EN to GND).
3. Release **BOOT**.

The board is now in ROM bootloader mode and `esptool` can write to it. Subsequent flashes via PlatformIO are automatic — no button dance needed.

### Upload

```bash
pio run --target upload
```

### Monitor serial output

```bash
pio device monitor
```

Output on first boot:

```
[prov] AP opened: Parayu-Setup
```

This means the device is waiting for WiFi credentials — proceed to [First Boot](first-boot/).
