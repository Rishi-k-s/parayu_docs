---
weight: 3
---

# WiFi Provisioning

## Library

Provisioning uses [tzapu/WiFiManager](https://github.com/tzapu/WiFiManager) (master branch, 2.0.17+). The versioned release `0.16.0` is ESP8266-only — always use the git master branch for ESP32:

```ini
lib_deps =
    https://github.com/tzapu/WiFiManager.git
```

---

## Cold boot vs warm boot

**Cold boot** — no WiFi credentials in NVS flash:

1. `wm.autoConnect("Parayu-Setup")` opens a soft AP.
2. The display task shows "Setup mode / Connect to: / Parayu-Setup".
3. `autoConnect` blocks until the user submits the captive portal form and WiFi connects.
4. The `saveConfigCallback` fires and persists WS config to NVS.
5. `autoConnect` returns `true`.

**Warm boot** — credentials already stored:

1. `wm.autoConnect` attempts WiFi with stored credentials.
2. If it connects within 30 seconds, returns `true` immediately.
3. The AP is never opened. Boot-to-streaming is typically under 3 seconds.

The 30-second connect timeout is set with:
```cpp
wm.setConnectTimeout(30);
```

---

## Custom parameters

Three extra form fields are added to the portal beyond SSID/password:

```cpp
WiFiManagerParameter s_paramHost("ws_host", "WS Host", "", 63);
WiFiManagerParameter s_paramPort("ws_port", "WS Port", "8765", 5);
WiFiManagerParameter s_paramPath("ws_path", "WS Path", "/audio", 63);

wm.addParameter(&s_paramHost);
wm.addParameter(&s_paramPort);
wm.addParameter(&s_paramPath);
```

These appear as standard HTML `<input>` elements in the portal page between the SSID and the Save button.

---

## NVS storage

WiFiManager stores WiFi credentials in its own NVS partition. The WS config is stored separately using Arduino's `Preferences` API under the `parayu` namespace (max 15 chars):

```cpp
// Write (inside saveConfigCallback)
Preferences prefs;
prefs.begin("parayu", false);                       // false = read-write
prefs.putString("ws_host", s_paramHost.getValue());
prefs.putUInt  ("ws_port", atoi(s_paramPort.getValue()));
prefs.putString("ws_path", s_paramPath.getValue());
prefs.end();

// Read (provisioningLoadConfig)
prefs.begin("parayu", true);                        // true = read-only
prefs.getString("ws_host", wsHost, 64);
*wsPort = prefs.getUInt("ws_port", 8765);
prefs.getString("ws_path", wsPath, 64);
prefs.end();
```

NVS survives power cycles and OTA updates. It is wiped by `pio run --target erase`.

---

## Captive portal timeout

`setConfigPortalTimeout(0)` means the portal never times out — the device stays in AP mode until a user submits credentials. Remove the `0` argument or pass a value in seconds if you want the device to restart itself after a timeout period.
