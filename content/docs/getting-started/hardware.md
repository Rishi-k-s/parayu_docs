---
weight: 1
---

# Hardware

## Components

| Component | Part | Notes |
|---|---|---|
| Microcontroller | ESP32-S3-WROOM-1-N16R8 | 16 MB flash, 8 MB PSRAM |
| Microphone | INMP441 MEMS I2S mic | 24-bit, left-channel |
| Display | SSD1306 128×64 OLED | I2C interface |
| Power | USB-C (5 V) | Powers ESP32 dev board |

---

## Pin Connections

### I2S Microphone (INMP441)

| INMP441 pin | ESP32-S3 GPIO | Notes |
|---|---|---|
| `WS` (word select / LRCK) | GPIO 3 | Left/right clock |
| `SCK` (bit clock) | GPIO 46 | Serial clock |
| `SD` (serial data) | GPIO 15 | Audio data out |
| `VDD` | 3.3 V | |
| `GND` | GND | |
| `L/R` | GND | Selects left channel |

> **Important:** The `L/R` pin must be tied to GND to use the left channel. The firmware configures `I2S_CHANNEL_FMT_ONLY_LEFT` — if `L/R` is left floating the mic output will be silence or noise.

### OLED Display (SSD1306, I2C)

| SSD1306 pin | ESP32-S3 GPIO |
|---|---|
| `SDA` | GPIO 16 |
| `SCL` | GPIO 17 |
| `VCC` | 3.3 V |
| `GND` | GND |

The firmware initialises Wire with `Wire.begin(16, 17)` and looks for the display at I2C address `0x3C`. Most SSD1306 breakout boards default to this address. If yours uses `0x3D`, update `OLED_ADDR` in `src/config.h`.

### Miscellaneous

| Signal | ESP32-S3 GPIO | State |
|---|---|---|
| Onboard LED / power rail | GPIO 8 | Driven LOW at boot |

---

## Wiring Diagram

```
           ESP32-S3-WROOM-1
          ┌──────────────────┐
          │                  │
  3.3V ───┤ 3V3          GND ├─── GND
          │                  │
          │  GPIO3  ─────────┼──── INMP441 WS
          │  GPIO15 ─────────┼──── INMP441 SD
          │  GPIO46 ─────────┼──── INMP441 SCK
          │                  │
          │  GPIO16 ─────────┼──── SSD1306 SDA
          │  GPIO17 ─────────┼──── SSD1306 SCL
          │                  │
          └──────────────────┘
```

---

## Notes on the ESP32-S3-N16R8

The **N16R8** variant has 16 MB of quad-die flash and 8 MB of Octal-SPI PSRAM. The firmware enables PSRAM via build flags:

```ini
build_flags =
    -DBOARD_HAS_PSRAM
    -mfix-esp32-psram-cache-issue
```

The `-mfix-esp32-psram-cache-issue` flag is required for the ESP32-S3 silicon errata around the PSRAM cache bus — omitting it causes random crashes under load. It is already included in `platformio.ini`.
