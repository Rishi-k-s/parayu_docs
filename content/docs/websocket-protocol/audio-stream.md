---
weight: 2
---

# Audio Stream Format

## Summary

| Property | Value |
|---|---|
| Encoding | Raw signed 16-bit PCM, little-endian |
| Sample rate | 16 000 Hz |
| Channels | 1 (mono) |
| Bit depth | 16 bit |
| Samples per frame | 256 |
| Bytes per frame | 512 |
| Frame rate | ~62.5 frames / second |
| Bandwidth | ~32 KB/s payload |

---

## Frame format

Each WebSocket binary frame is exactly **512 bytes** of contiguous raw PCM samples.

```
Byte offset:  0       1       2       3   ...  510     511
              ┌───────────────┬───────────────┬───────────────┐
              │   sample[0]   │   sample[1]   │  sample[255]  │
              │  (int16 LE)   │  (int16 LE)   │  (int16 LE)   │
              └───────────────┴───────────────┴───────────────┘
              ◄────────────── 512 bytes ─────────────────────►
```

- Each sample is a **signed 16-bit integer, little-endian** (LSB first).
- Samples represent instantaneous amplitude. Full scale is `−32768` to `+32767`.
- There is **no header, timestamp, sequence number, or metadata** in the frame — it is raw PCM only.
- Frames are always 512 bytes. The DMA buffer count is fixed at 256 samples; partial frames are not sent.

---

## Reconstructing the audio stream

To reconstruct a continuous audio stream from incoming frames, simply concatenate the payloads in order:

```
frame_0_payload (512 B) + frame_1_payload (512 B) + ...
= continuous PCM stream
```

The resulting byte stream is a valid headerless raw PCM file. To open it in **Audacity**:

1. File → Import → Raw Data
2. Encoding: Signed 16-bit PCM
3. Byte order: Little-endian
4. Channels: 1
5. Sample rate: 16000

---

## Timing and jitter

At 16 kHz with 256-sample buffers, one frame represents **16 ms** of audio. The I2S DMA delivers frames at this rate with hardware precision — the frame rate is determined by the I2S clock, not by the WiFi stack or the application.

However, **network jitter** means frames may arrive at the server in bursts rather than evenly spaced. A server that processes frames synchronously (one at a time) will be fine. A server doing real-time playback needs a jitter buffer of at least 3–5 frames (~50–80 ms).

---

## Silence during WebSocket reconnection

If the WebSocket connection drops and the device is in `WS_LOST` state, the audio task continues capturing I2S data but **drops all frames** (the `sendBIN` call is skipped). Audio captured during the outage is not buffered and cannot be recovered. When the connection re-establishes, streaming resumes from the next captured frame.

---

## Effective bit depth

The INMP441 is a 24-bit microphone. The firmware reads it as a 32-bit I2S word (the DMA requires a power-of-two word size) and shifts right by 14:

```
32-bit I2S word:  [23-bit sample | 8-bit zero padding | 1 extra bit]
                                   >> 14
16-bit PCM word:  [top 18 bits of the 24-bit sample, bottom 2 bits zero]
```

The effective dynamic range is approximately **18 bits** (~108 dB), although only 16 bits are transmitted. In practice this is well above the noise floor of any MEMS microphone.

To extract full 16-bit precision, change `>> 14` to `>> 16` in `src/audio.cpp` — this discards the 8 least significant bits of the 24-bit sample rather than 6.

---

## Building a WAV file from frames

The Sarvam API (and most STT APIs) require a WAV header. To wrap accumulated PCM frames into a valid WAV:

```
RIFF header (44 bytes) + raw PCM payload
```

The exact structure (matching the server implementation):

```
Offset  Size  Value
0       4     "RIFF"
4       4     36 + data_size  (little-endian uint32)
8       4     "WAVE"
12      4     "fmt "
16      4     16              (PCM chunk size)
20      2     1               (PCM format)
22      2     1               (channels)
24      4     16000           (sample rate)
28      4     32000           (byte rate = sample_rate × channels × bits/8)
32      2     2               (block align = channels × bits/8)
34      2     16              (bits per sample)
36      4     "data"
40      4     data_size       (bytes of raw PCM)
44      …     raw PCM
```

The server accumulates **192 frames** (192 × 512 = 98 304 bytes ≈ 3.07 seconds) before wrapping them in a WAV header and sending to the Sarvam API. This threshold is `CHUNKS_PER_SEND = 192` in `server/sarvam_client.py`.
