---
weight: 3
---

# First Boot & Provisioning

On first boot (or after a credential reset) the device has no WiFi credentials stored. Instead of hanging, it opens its own WiFi access point and serves a configuration portal.

---

## What happens step by step

```
Power on
  │
  ▼
OLED: "Parayu / Booting..."
  │
  ▼
No stored WiFi credentials found
  │
  ▼
OLED: "Setup mode / Connect to: / Parayu-Setup"
ESP32 opens AP: "Parayu-Setup"  (open, no password)
  │
  ▼  [user connects phone/laptop to "Parayu-Setup"]
  │
  ▼
Captive portal auto-opens (or navigate to 192.168.4.1)
  │
  ▼
User fills in:
  • WiFi SSID
  • WiFi Password
  • WS Host  (IP of the machine running the server, e.g. 192.168.1.4)
  • WS Port  (default: 8765)
  • WS Path  (default: /audio)
  │
  ▼
Device connects to the entered WiFi network
Credentials written to NVS (non-volatile flash storage)
  │
  ▼
OLED: "WiFi OK / 192.168.x.x / Connecting WS..."
  │
  ▼
Device connects WebSocket to ws://[WS Host]:[WS Port][WS Path]
  │
  ▼
OLED: streaming screen (VU bar, packet counter, IP)
```

---

## The configuration portal

When your phone connects to **Parayu-Setup**, the captive portal should open automatically (iOS / Android intercept the DNS probe). If it doesn't, open a browser and navigate to:

```
http://192.168.4.1
```

You will see a form with five fields:

| Field | Example | Description |
|---|---|---|
| WiFi SSID | `MyHomeNetwork` | Name of your wireless network |
| WiFi Password | `hunter2` | Password (leave blank for open networks) |
| WS Host | `192.168.1.4` | LAN IP of the machine running the server |
| WS Port | `8765` | Server port (must match `uvicorn` startup) |
| WS Path | `/audio` | WebSocket endpoint path |

Click **Save**. The device will:
1. Connect to your WiFi.
2. Persist the WS config to NVS under the `parayu` namespace.
3. Begin streaming immediately.

---

## Subsequent boots

On every boot after provisioning, the device reads credentials from NVS and connects directly — the AP is never opened. Boot-to-streaming typically takes under 3 seconds.

---

## Re-provisioning

There is no physical button yet. To reset credentials and re-enter provisioning mode, open a serial monitor and restart the device while holding BOOT — this clears NVS on some boards. A proper reset button is planned for a future firmware update.

Alternatively, you can reflash with cleared NVS:

```bash
pio run --target erase   # wipes entire flash
pio run --target upload  # re-flash firmware
```

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| "Parayu-Setup" AP never appears | Board didn't boot into provisioning | Check serial monitor; try holding BOOT while pressing RESET |
| Portal doesn't open automatically | Captive portal DNS not intercepted | Navigate manually to `192.168.4.1` |
| OLED shows "WS Lost! Retrying..." | Server not running or wrong IP/port | Start the server; verify IP with `ip addr` |
| Device keeps re-entering AP mode | Wrong WiFi password stored | Erase flash and reflash, then re-provision |
