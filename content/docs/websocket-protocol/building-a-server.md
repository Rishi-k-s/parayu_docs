---
weight: 3
---

# Building a Compatible Server

This page shows the minimum code needed to accept audio from a Parayu device in both **Node.js** and **Python**. These are self-contained examples — no Sarvam integration, just receive-and-log.

---

## What the server must do

1. Listen for WebSocket connections on the provisioned port (default `8765`) and path (default `/audio`).
2. Accept the connection — no subprotocol or authentication headers are sent by the device.
3. Read incoming **binary** frames. Each frame is 512 bytes of signed 16-bit PCM at 16 kHz mono.
4. Do something useful with them (buffer, forward to STT, write to file, etc.).

That's it. The device does not send a handshake message, does not expect any response, and does not use any custom WebSocket subprotocol.

---

## Node.js

Install the `ws` package:

```bash
npm install ws
```

Minimal receiver:

```javascript
const WebSocket = require('ws');

const wss = new WebSocket.Server({ port: 8765 });

wss.on('connection', (ws, req) => {
  console.log(`Device connected from ${req.socket.remoteAddress}`);

  ws.on('message', (data, isBinary) => {
    if (!isBinary) return;                 // device only sends binary
    const samples = data.length / 2;      // 512 bytes / 2 = 256 int16 samples
    console.log(`Received frame: ${data.length} bytes (${samples} samples)`);
  });

  ws.on('close', () => console.log('Device disconnected'));
  ws.on('error', err => console.error('WS error:', err.message));
});

console.log('Listening on ws://0.0.0.0:8765');
```

### Saving frames to a raw PCM file (Node.js)

```javascript
const fs = require('fs');
const WebSocket = require('ws');

const wss = new WebSocket.Server({ port: 8765 });

wss.on('connection', (ws) => {
  const out = fs.createWriteStream(`session-${Date.now()}.raw`);
  console.log('Recording started');

  ws.on('message', (data, isBinary) => {
    if (isBinary) out.write(data);
  });

  ws.on('close', () => {
    out.end();
    console.log('Recording saved');
  });
});
```

Open the resulting `.raw` file in Audacity: **File → Import → Raw Data**, Signed 16-bit PCM, Little-endian, 1 channel, 16000 Hz.

### Routing only the `/audio` path (Node.js + express)

If you want to serve an HTTP API on the same port alongside the WebSocket:

```javascript
const express = require('express');
const { createServer } = require('http');
const WebSocket = require('ws');

const app = express();
const server = createServer(app);
const wss = new WebSocket.Server({ noServer: true });

server.on('upgrade', (req, socket, head) => {
  if (req.url === '/audio') {
    wss.handleUpgrade(req, socket, head, ws => wss.emit('connection', ws, req));
  } else {
    socket.destroy();
  }
});

wss.on('connection', (ws) => {
  ws.on('message', (data, isBinary) => {
    if (isBinary) { /* handle audio */ }
  });
});

server.listen(8765);
```

---

## Python

Install `websockets`:

```bash
pip install websockets
```

Minimal receiver:

```python
import asyncio
import websockets

async def handler(ws):
    client = ws.remote_address
    print(f"Device connected: {client}")
    async for message in ws:
        if isinstance(message, bytes):
            samples = len(message) // 2          # 512 bytes / 2 = 256 int16
            print(f"Received frame: {len(message)} bytes ({samples} samples)")
    print(f"Device disconnected: {client}")

async def main():
    async with websockets.serve(handler, "0.0.0.0", 8765):
        print("Listening on ws://0.0.0.0:8765")
        await asyncio.Future()

asyncio.run(main())
```

### Saving frames to a raw PCM file (Python)

```python
import asyncio
import time
import websockets

async def handler(ws):
    filename = f"session-{int(time.time())}.raw"
    print(f"Recording to {filename}")
    with open(filename, "wb") as f:
        async for message in ws:
            if isinstance(message, bytes):
                f.write(message)
    print("Recording saved")

async def main():
    async with websockets.serve(handler, "0.0.0.0", 8765):
        await asyncio.Future()

asyncio.run(main())
```

### Decoding samples to numpy (Python)

If you want to process audio numerically:

```python
import numpy as np

async for message in ws:
    if isinstance(message, bytes):
        # Convert bytes → numpy int16 array
        pcm = np.frombuffer(message, dtype='<i2')   # '<i2' = signed int16, little-endian
        # pcm.shape == (256,)
        # pcm values range from -32768 to 32767

        # Normalise to float32 [-1.0, 1.0]
        audio = pcm.astype(np.float32) / 32768.0
```

### Handling the `/audio` path (Python + FastAPI)

The production Parayu server uses FastAPI. The path is handled with a route decorator:

```python
from fastapi import FastAPI, WebSocket, WebSocketDisconnect

app = FastAPI()

@app.websocket("/audio")
async def esp32_audio(ws: WebSocket):
    await ws.accept()
    try:
        while True:
            data = await ws.receive_bytes()
            # process data (512 bytes of int16 PCM)
    except WebSocketDisconnect:
        pass
```

---

## WAV wrapping for STT APIs

Most speech-to-text APIs (Sarvam, Google, Whisper, etc.) require a WAV header. Buffer frames until you have enough audio, then prepend the header:

```python
import struct

def make_wav(pcm: bytes, sample_rate=16000, channels=1, bits=16) -> bytes:
    data_size   = len(pcm)
    byte_rate   = sample_rate * channels * bits // 8
    block_align = channels * bits // 8
    header = struct.pack(
        "<4sI4s4sIHHIIHH4sI",
        b"RIFF", 36 + data_size, b"WAVE",
        b"fmt ", 16, 1, channels, sample_rate,
        byte_rate, block_align, bits,
        b"data", data_size,
    )
    return header + pcm
```

The Parayu server buffers 192 frames (~3 seconds) before calling this — empirically chosen to give the Sarvam model enough context for accurate transcription.

---

## Common pitfalls

**Receiving text frames instead of binary**

The device always calls `sendBIN()`. If you receive a text frame, another client (e.g. a browser debug tab) accidentally connected to `/audio`. Binary-only check:

```python
# Python
if isinstance(message, bytes): ...     # text arrives as str

// Node.js
ws.on('message', (data, isBinary) => {
  if (!isBinary) return;
});
```

**WAV header corruption**

Raw PCM buffers contain null bytes (`0x00`) and arbitrary byte patterns. If you accidentally pass PCM to a function that expects UTF-8 text (e.g. `ws.send(data)` in some JS contexts without the binary flag), the data will be corrupted. Always use the binary send path.

**Connection refused but device shows WS_LOST**

The device retries every 3 seconds. If you start the server after the device has been running, it will connect automatically within 3 seconds — no need to restart the device.

**Multiple devices connecting to the same endpoint**

The current server implementation handles one device connection at a time — a new connection from a second device will work independently, but the Sarvam session is per-connection. If you need multiple devices, each `@app.websocket("/audio")` handler spawns an independent `SarvamSession`.
