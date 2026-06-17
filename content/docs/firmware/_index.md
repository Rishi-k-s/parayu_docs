---
weight: 2
params:
  bookCollapseSection: true
---

# Firmware Reference

Deep reference for the ESP32-S3 firmware — how it's structured, how the two cores divide work, and how each module fits together.

| Page | Topic |
|---|---|
| [Architecture](architecture/) | FreeRTOS tasks, dual-core split, state machine, shared state |
| [Audio](audio/) | I2S driver config, PCM conversion, VU peak calculation |
| [Provisioning](provisioning/) | WiFiManager lifecycle, NVS storage, warm vs cold boot |
