---
weight: 1
---

# Firmware Architecture

## Module layout

```
src/
├── config.h           compile-time constants (pins, rates, task params)
├── shared_state.h/cpp mutex-guarded state shared between both cores
├── provisioning.h/cpp WiFiManager + NVS credential storage
├── audio.h/cpp        I2S capture + WebSocket send  (Core 1)
├── display.h/cpp      OLED render + WiFi watchdog   (Core 0)
└── main.cpp           startup sequencing only
```

`config.h` is a pure header — no runtime behaviour, included by everything else. `shared_state` is the only module with no upstream dependencies and is initialised first in `setup()`.

---

## Dual-core task map

The ESP32-S3 has two Xtensa LX7 cores:

- **Core 0** (APP_CPU) — where the WiFi/LwIP stack runs by default.
- **Core 1** (PRO_CPU) — where Arduino's `setup()` and `loop()` run by default.

```
Core 0                              Core 1
──────────────────────────────      ──────────────────────────────────
WiFi / LwIP (system)                setup() → main.cpp sequencing
                                    ↓
displayTask  (priority 2)           audioTask  (priority 5)
  WiFi.status() watchdog              i2s_read() [~16 ms block]
  SSD1306 render every 100 ms         int32→int16 conversion + peak
  reads SharedState                   ws.loop() non-blocking
                                      ws.sendBIN()
                                      writes SharedState
```

**Why this split:**

- All `WiFi.*` API calls (`RSSI()`, `localIP()`, `status()`) must happen on Core 0 where LwIP runs. Calling them from Core 1 is technically legal in Arduino but can cause subtle race conditions under reconnect pressure.
- `i2s_read()` with `portMAX_DELAY` blocks until the DMA buffer is full (~16 ms at 16 kHz, 256 samples). Pinning the audio task to Core 1 co-locates it with the I2S DMA ISR, reducing cross-core cache invalidation.
- The display task runs at priority 2 — a missed render deadline is invisible; an audio drop is not.

### Task parameters

| Task | Core | Priority | Stack |
|---|---|---|---|
| `audioTask` | 1 | 5 | 8 192 B |
| `displayTask` | 0 | 2 | 4 096 B |

Priority 5 is intentionally below the WiFi task (priority 6) — exceeding it would starve TCP/IP and cause WebSocket stalls.

---

## Shared state

Both tasks communicate through a single mutex-protected struct defined in `shared_state.cpp`. All access goes through typed getters/setters:

```cpp
// Written by audioTask, read by displayTask
sharedSetPeak(peak);
sharedIncrPacketCount();

// Written by displayTask, read by audioTask indirectly
sharedSetState(DeviceState::WIFI_LOST);

// Written once in setup() before tasks start, read by audioTask
sharedSetWsConfig(host, port, path);
```

The mutex (`SemaphoreHandle_t`, created in `sharedStateInit()`) is taken and released inside each accessor, so callers never need to lock manually except when doing multi-field atomic updates via `sharedStateLock()` / `sharedStateUnlock()`.

---

## State machine

The `DeviceState` enum drives both the OLED display and the reconnection logic:

```
BOOTING
  │ setup() starts
  ▼
PROVISIONING          ← AP is open, waiting for credentials
  │ autoConnect() returns
  ▼
WIFI_CONNECTING
  │ WiFi.status() == WL_CONNECTED
  ▼
WIFI_CONNECTED
  │ ws.begin() called
  ▼
WS_CONNECTING
  │ WStype_CONNECTED event
  ▼
STREAMING  ◄─────────────────────────────────────┐
  │                                              │
  │ WiFi.status() != WL_CONNECTED               │ WiFi recovers
  ▼                                              │
WIFI_LOST ──── WiFi.reconnect() loop ────────────┘
  │
  │ WStype_DISCONNECTED (WS only, WiFi still up)
  ▼
WS_LOST ──── ws.setReconnectInterval(3000) ──────► WS_CONNECTING
```

**Authority rules** prevent both tasks from fighting over state transitions:

- `displayTask` is the sole authority for `WIFI_LOST` and `WIFI_LOST → WS_CONNECTING` (it is the only task calling `WiFi.status()`).
- `audioTask` is the sole authority for `WS_CONNECTING → STREAMING` and `STREAMING → WS_LOST` (it owns the WebSocket event callback).

---

## Startup sequence (`main.cpp`)

```cpp
sharedStateInit();       // mutex must exist before any task uses it
displayInit();           // hardware init — Wire.begin, display.begin
                         // must happen before provisioning needs OLED feedback

provisioningRun();       // blocks until WiFi connected (captive portal or fast reconnect)

provisioningLoadConfig(host, &port, path);
sharedSetWsConfig(host, port, path);

audioTaskStart();        // pinned to Core 1
displayTaskStart();      // pinned to Core 0

// loop() is now idle — all real work is in FreeRTOS tasks
```

`displayInit()` is called synchronously from `setup()` on Core 1, before tasks start. After that, all `display.*` calls must happen only from the display task on Core 0 — the SSD1306 driver is not thread-safe for concurrent access from two cores.
