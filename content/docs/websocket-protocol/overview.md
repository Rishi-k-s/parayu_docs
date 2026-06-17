---
weight: 1
---

# WebSocket Overview

## Endpoint

The device connects to a single WebSocket endpoint configured during provisioning:

```
ws://[WS_HOST]:[WS_PORT][WS_PATH]
```

Default values: `ws://192.168.1.4:8765/audio`

There is no TLS (`wss://`) in the current firmware. All communication is plaintext WebSocket over TCP on the local network.

---

## Connection lifecycle

```
Device boots
  │
  ▼
WiFi connected
  │
  ▼
ws.begin(host, port, path)           ← non-blocking, starts connect attempt
  │
  ▼ (async, driven by ws.loop())
  │
  ├─ Server reachable ─────────────► WStype_CONNECTED
  │                                       │
  │                                       ▼
  │                                  Device streams audio frames
  │                                       │
  │         Server closes connection ◄────┤
  │         or network drops              │
  │                                       ▼
  │                                  WStype_DISCONNECTED
  │                                       │
  └─ Server unreachable ───────────► reconnect after 3 s (automatic)
                                          │
                                          └─► retry indefinitely
```

The reconnect interval is set in `src/audio.cpp`:

```cpp
s_ws.setReconnectInterval(3000);  // ms
```

The WebSockets library handles all retries internally — the application does not need to call `ws.begin()` again after a disconnect.

---

## Driving the connection

The WebSocket library requires `ws.loop()` to be called regularly. In Parayu, this happens at the top of every audio task iteration:

```cpp
for (;;) {
    s_ws.loop();           // processes events, drives reconnect timer
    i2s_read(...);         // blocks ~16 ms
    // ...
}
```

This means `ws.loop()` is called approximately **62 times per second** (once per 256-sample / 16 kHz buffer period). The reconnect timer resolution is therefore ~16 ms, and WebSocket PING/PONG latency is at most ~16 ms.

> **Server side:** If your server sends WebSocket PINGs, set the ping timeout to at least 60 seconds. The ~16 ms polling interval is fine for keepalives, but if the server sends a PING during the `i2s_read` block the PONG may be delayed by up to 16 ms.

---

## WebSocket frame types used

| Direction | Type | Content |
|---|---|---|
| Device → Server | **Binary** | Raw PCM audio (see [Audio Stream](audio-stream/)) |
| Server → Device | none | The device does not process any incoming frames |

The device uses `sendBIN()` exclusively. `sendTXT()` must not be used — raw PCM buffers contain null bytes and values that are not valid UTF-8, which corrupts a text frame.

---

## Observing the connection state on the OLED

| OLED text | `DeviceState` | Meaning |
|---|---|---|
| `Connecting WS...` | `WS_CONNECTING` | `ws.begin()` called, waiting for handshake |
| streaming screen | `STREAMING` | `WStype_CONNECTED` received, audio flowing |
| `WS Lost! Retrying...` | `WS_LOST` | `WStype_DISCONNECTED`, auto-retry pending |
| `WiFi Lost!` | `WIFI_LOST` | Network dropped; WebSocket will retry after WiFi recovers |
