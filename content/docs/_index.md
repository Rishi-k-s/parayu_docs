# Parayu

**Parayu** is an ESP32-S3 based wireless audio streaming device that captures microphone input, streams it over WebSocket to a server, and transcribes it in real time using the Sarvam AI speech-to-text API.

The name *Parayu* (പറയൂ) is Malayalam for **"speak"**.

---

## What it does

1. An ESP32-S3-WROOM-1-N16R8 captures audio from an INMP441 MEMS microphone via I2S.
2. The device connects to your WiFi network (configured via a captive portal on first boot).
3. Raw 16-bit PCM audio is streamed over WebSocket to a Python/FastAPI server on your LAN.
4. The server buffers the audio, wraps it in WAV format, and sends it to the Sarvam `saaras:v3` streaming STT model.
5. Transcripts appear live in a minimal web UI accessible from any browser on the network.

---

## Sections

| Section | What's inside |
|---|---|
| [Getting Started](getting-started/) | Hardware wiring, firmware build & flash, first boot provisioning |
| [Firmware Reference](firmware/) | Dual-core architecture, FreeRTOS tasks, I2S driver, state machine |
| [WebSocket Protocol](websocket-protocol/) | Binary audio format, connection flow, building a compatible receiver |
| [Server](server/) | Python server setup, Sarvam integration, web UI events |

---

## Architecture overview

```
┌─────────────────────────────────────┐
│  ESP32-S3  (Core 1)                 │
│  I2S → PCM convert → ws.sendBIN()  │
└────────────────┬────────────────────┘
                 │  WebSocket /audio
                 │  binary frames (512 B each)
                 ▼
┌─────────────────────────────────────┐
│  FastAPI server  (192.168.x.x:8765) │
│  buffer 192 frames → WAV → Sarvam  │
└───────────┬─────────────────────────┘
            │  WebSocket /ws  (JSON events)
            ▼
┌─────────────────────────────────────┐
│  Browser UI                         │
│  live fragment + completed sentence │
└─────────────────────────────────────┘
```
