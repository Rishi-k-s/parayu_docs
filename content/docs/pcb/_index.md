---
weight: 5
title: PCB
params:
  bookToc: true
---

# PCB — V2

The Parayu V2 PCB is a custom ESP32-S3 board designed in KiCad. It moves everything from the V1 breadboard prototype onto a single compact board — integrating dual MEMS microphones, a 128×32 OLED, LiPo charging, battery protection, USB-C, and an accelerometer.

> **V1 vs V2** — The V1 prototype uses an INMP441 breakout wired on a breadboard. The V2 PCB replaces it with a TDK MMICT5848-00-012 bottom-port I2S microphone soldered directly, and adds onboard power management, battery, and accelerometer. The firmware pin assignments and I2S config remain the same.

**KiCad project files:** [`parayu_PCB/`](https://github.com/Rishi-k-s/parayu/tree/main/parayu_PCB)

---

## Files

| File | Description |
|---|---|
| `esp32_parayu_prototype.kicad_pro` | KiCad project |
| `esp32_parayu_prototype.kicad_sch` | Schematic |
| `esp32_parayu_prototype.kicad_pcb` | PCB layout |
| `BOM parayu.csv` | Bill of materials |
| `ALL_ZIP/` | Component footprint libraries |

To generate Gerbers: open the `.kicad_pcb` file → **File → Fabrication Outputs → Gerbers**.

---

## Bill of Materials

**Total BOM cost: ₹ 2,550.05** &nbsp;(21 line items, quantities as specified)

### Core

| # | Component | MPN | Qty | Description | Package | Unit ₹ | Total ₹ |
|---|---|---|---|---|---|---|---|
| 1 | [ESP32-S3-WROOM-1-N16R8](https://www.digikey.in/en/products/detail/espressif-systems/ESP32-S3-WROOM-1-N16R8/16162642) | ESP32-S3-WROOM-1-N16R8 | 1 | 240 MHz dual-core, 16 MB flash, 8 MB PSRAM, WiFi + BT | SMD module | 638.89 | 638.89 |

### Audio & Sensing

| # | Component | MPN | Qty | Description | Package | Unit ₹ | Total ₹ |
|---|---|---|---|---|---|---|---|
| 2 | [MMICT5848-00-012](https://www.digikey.in/en/products/detail/tdk-invensense/MMICT5848-00-012/18634578) | MMICT5848-00-012 | 2 | Bottom-port I2S digital MEMS microphone | SMD | 223.99 | 447.98 |
| 3 | [0.91″ 128×32 OLED](https://robu.in/product/0-91-inch-128x32-i2c-iic-serial-blue-oled-lcd-display-module/) | — | 1 | 128×32 blue OLED, I2C interface | Module | 219.00 | 219.00 |
| 8 | [LIS2DH12TR](https://www.digikey.in/en/products/detail/stmicroelectronics/LIS2DH12TR/4899892) | LIS2DH12TR | 1 | 3-axis accelerometer, 2–16 g, I2C/SPI | 12-LGA | 169.17 | 169.17 |
| 14 | [NCU15XH103F60RC](https://www.digikey.in/en/products/detail/murata-electronics/NCU15XH103F60RC/9686714) | NCU15XH103F60RC | 1 | NTC thermistor, 10 kΩ, 3380 K | 0402 | 9.45 | 9.45 |

### Power Management

| # | Component | MPN | Qty | Description | Package | Unit ₹ | Total ₹ |
|---|---|---|---|---|---|---|---|
| 5 | [AP7365-33WG-7](https://www.digikey.in/en/products/detail/diodes-incorporated/AP7365-33WG-7/4249847) | AP7365-33WG-7 | 1 | LDO 3.3 V, 600 mA | SOT-25 | 26.46 | 26.46 |
| 4 | [MCP1700T-1802E/TT](https://www.digikey.in/en/products/detail/microchip-technology/MCP1700T-1802E-TT/652671) | MCP1700T-1802E/TT | 2 | LDO 1.8 V, 200 mA | SOT-23-3 | 48.20 | 96.40 |
| 6 | [BQ25185DLHR](https://www.digikey.in/en/products/detail/texas-instruments/BQ25185DLHR/21769368) | BQ25185DLHR | 1 | 1-cell, 1 A standalone LiPo charger | 10-WFDFN | 215.00 | 215.00 |
| 7 | [BQ29700DSER](https://www.digikey.in/en/products/detail/texas-instruments/BQ29700DSER/5973173) | BQ29700DSER | 1 | 1-cell Li-ion battery protection IC | 6-WSON | 53.87 | 53.87 |
| 18 | [DMG2305UX-13](https://www.digikey.in/en/products/detail/diodes-incorporated/DMG2305UX-13/4251560) | DMG2305UX-13 | 1 | P-channel MOSFET, 20 V, 4.2 A | SOT-23-3 | 33.08 | 33.08 |
| 21 | [WLY102535](https://robu.in/product/950mah-pcm-protected-micro-li-po-battery/) | WLY102535 | 1 | 3.7 V 950 mAh 1S LiPo battery | — | 387.00 | 387.00 |

### USB & Protection

| # | Component | MPN | Qty | Description | Package | Unit ₹ | Total ₹ |
|---|---|---|---|---|---|---|---|
| 11 | [UJ20-C-H-G-SMT-1A-P16-TR](https://www.digikey.in/en/products/detail/same-sky-formerly-cui-devices/UJ20-C-H-G-SMT-1A-P16-TR/28173556) | UJ20-C-H-G-SMT-1A-P16-TR | 1 | USB-C 2.0 SMT horizontal connector | SMT | 36.86 | 36.86 |
| 10 | [USBLC6-2SC6](https://www.digikey.in/en/products/detail/shenzhen-slkormicro-semicon-co-ltd/USBLC6-2SC6/28143029) | USBLC6-2SC6 | 1 | USB ESD protection, 5 V, 6 A | SOT-23-6 | 16.07 | 16.07 |

### Logic

| # | Component | MPN | Qty | Description | Package | Unit ₹ | Total ₹ |
|---|---|---|---|---|---|---|---|
| 9 | [TXS0104EPWRG4](https://www.digikey.in/en/products/detail/texas-instruments/TXS0104EPWRG4/1910179) | TXS0104EPWRG4 | 1 | Bidirectional voltage-level shifter, 4-bit | 14-TSSOP | 102.07 | 102.07 |

### UI

| # | Component | MPN | Qty | Description | Package | Unit ₹ | Total ₹ |
|---|---|---|---|---|---|---|---|
| 12 | [SWT0220-016016GSA](https://www.digikey.in/en/products/detail/gct/SWT0220-016016GSA/26753069) | SWT0220-016016GSA | 2 | Tactile switch, SPST-NO, 5.2×5 mm | SMD | 9.45 | 18.90 |
| 13 | [XL-1615RGBC-YG](https://robu.in/product/xl-1615rgbc-yg-xinglight-smd-4p1-6x1-5mm-rgb-leds-rohs/) | XL-1615RGBC-YG | 3 | RGB LED, SMD 1.6×1.5 mm | SMD | 2.46 | 7.38 |

### Passives

| # | Component | MPN | Qty | Value | Package | Unit ₹ | Total ₹ |
|---|---|---|---|---|---|---|---|
| 15 | [0402B104J250NT](https://robu.in/product/0402b104j250nt-fh-25v-100nf-x7r%C2%B15-0402-multilayer-ceramic-capacitors-mlcc-smd-smt-rohs/) | 0402B104J250NT | 15 | 100 nF, X7R, 25 V | 0402 | 0.35 | 5.25 |
| 17 | [CC0402MRX5R5BB106](https://robu.in/product/cc0402mrx5r5bb106-yageo-cap-smd-mlcc-10-%c2%b5f-6-3-v-0402-1005-metric-20-x5r-cc-series/) | CC0402MRX5R5BB106 | 8 | 10 µF, X5R, 6.3 V | 0402 | 8.00 | 64.00 |
| 19 | [RC0805FR-0710KL](https://robu.in/product/rc0805fr-0710kl-yageo-res-thick-film-0805-10k-ohm-1-0-125w1-8w-%c2%b1100ppm-c-pad-smd-t-r/) | RC0805FR-0710KL | 2 | 10 kΩ, 1%, 0.125 W | 0805 | 0.80 | 1.60 |
| 20 | [Yageo 100 kΩ](https://robu.in/product/yageo-100k-ohm-1-2w-0805-surface-mount-resistor-pack-of-50/) | — | 3 | 100 kΩ, 0.5 W | 0805 | 0.54 | 1.62 |

---

## Block diagram

```
USB-C ──► USBLC6 (ESD) ──► BQ25185 (charger) ──► LiPo 950mAh
                                │
                                ├──► AP7365 (3.3V) ──► ESP32-S3
                                │                      I2C/SPI bus
                                │                        ├── OLED (128×32)
                                │                        ├── LIS2DH12 (accel)
                                │                        └── NCU15 (thermistor)
                                │
                                └──► MCP1700 ×2 (1.8V) ──► MMICT5848 (mic ×2)
                                                            TXS0104 (level shift)

BQ29700 ── cell protection ──► MOSFET (DMG2305) ──► battery path
```
