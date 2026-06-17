---
layout: landing
---

<div class="book-hero">

# Parayu {anchor=false}
An ESP32-S3 wireless audio streaming device that transcribes speech in real time — Malayalam, English, or both.

{{<button href="/docs/">}}Get Started{{</button>}}
{{<button href="/docs/websocket-protocol/" relref="/docs/websocket-protocol/">}}Protocol Reference{{</button>}}

</div>

{{% columns %}}
- ## Speak, it listens
  Captures audio from an INMP441 MEMS microphone over I2S, streams raw PCM to your server over WebSocket, and shows live transcriptions in the browser — all on a device smaller than your palm.

- ## Zero-friction setup
  On first boot the device opens a WiFi access point. Connect your phone, open the captive portal, enter your network credentials and server address. Done — no reflashing, no config files.

- ## Built for real speech
  Powered by the Sarvam `saaras:v3` model in codemix mode — handles natural Malayalam + English code-switching without needing to pick a language upfront.
{{% /columns %}}
