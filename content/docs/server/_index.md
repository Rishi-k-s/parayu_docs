---
weight: 4
params:
  bookCollapseSection: true
---

# Server

The Parayu server is a Python/FastAPI application that bridges the ESP32 WebSocket audio stream to the Sarvam AI STT API and pushes transcripts to a browser UI.

| Page | Topic |
|---|---|
| [Setup](setup/) | Requirements, install, running the server |
| [Sarvam Integration](sarvam/) | Streaming STT session lifecycle, buffering, sentence assembly |
| [Web UI](web-ui/) | Browser WebSocket events, UI states, customisation |
