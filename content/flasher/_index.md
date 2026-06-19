---
title: Web Flasher
weight: 10
---

# Web Flasher

Flash the latest Parayu firmware to your ESP32-S3 directly from this page — no software to install.

<script type="module" src="https://unpkg.com/esp-web-tools@10/dist/web/install-button.js?module"></script>

<div style="margin: 2rem 0; padding: 2rem; border: 1px solid #e0e0e0; border-radius: 8px; text-align: center; background: #fafafa;">
  <p style="margin: 0 0 1.5rem; font-size: 0.9rem; color: #555;">
    Connect your ESP32-S3 via USB, then click the button below.
  </p>
  <esp-web-install-button manifest="/flasher/manifest.json">
    <button slot="activate" style="
      background: #1a73e8;
      color: #fff;
      border: none;
      border-radius: 6px;
      padding: 0.75rem 2rem;
      font-size: 1rem;
      font-family: inherit;
      cursor: pointer;
      letter-spacing: 0.02em;
    ">
      ⚡ Install Parayu Firmware
    </button>
    <span slot="unsupported" style="color: #c0392b; font-size: 0.9rem;">
      Your browser does not support Web Serial.<br>
      Use Chrome or Edge on desktop.
    </span>
  </esp-web-install-button>
</div>

---

## Requirements

| Requirement | Details |
|---|---|
| **Browser** | Chrome 89+ or Edge 89+ on desktop (Web Serial API required) |
| **OS** | Windows, macOS, or Linux |
| **Cable** | USB-C data cable (not a charge-only cable) |
| **Driver** | Linux: [udev rules](../docs/getting-started/firmware-setup/#install-udev-rules-linux) must be installed |

Firefox and Safari do not support Web Serial — the button will show an "unsupported" message on those browsers.

---

## First-time flash vs update

**First-time flash** — the device has never been flashed before or the flash has been erased:

1. Hold **BOOT**, press **RESET**, release **BOOT** to enter bootloader mode.
2. Click **Install Parayu Firmware** and select the correct serial port.
3. The flasher writes the bootloader, partition table, and firmware.
4. The device reboots automatically into provisioning mode (see [First Boot](../docs/getting-started/first-boot/)).

**Update** — the device is already running Parayu firmware:

1. Connect via USB (no button dance needed).
2. Click **Install Parayu Firmware**.
3. The flasher updates only the firmware partition — WiFi credentials and provisioned config are preserved.

> To wipe everything (credentials included), check **Erase device** in the confirmation dialog before flashing.

---

## How it works

The flasher uses [ESP Web Tools](https://esphome.github.io/esp-web-tools/) by the ESPHome project. It reads the firmware manifest at `/flasher/manifest.json`, downloads the binary files from the latest GitHub release, and writes them to the ESP32-S3 over Web Serial — all inside the browser with no local install.

The three binary files written are:

| File | Flash offset | Description |
|---|---|---|
| `bootloader.bin` | `0x0000` | ESP-IDF second-stage bootloader |
| `partitions.bin` | `0x8000` | Partition table (default_8MB layout) |
| `firmware.bin` | `0x10000` | Parayu application firmware |
