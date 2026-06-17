---
weight: 2
---

# Audio Capture

## I2S driver configuration

The firmware uses the legacy ESP-IDF `driver/i2s.h` API (v1). Configured in `src/audio.cpp`:

```cpp
i2s_config_t cfg = {
    .mode                 = I2S_MODE_MASTER | I2S_MODE_RX,
    .sample_rate          = 16000,
    .bits_per_sample      = I2S_BITS_PER_SAMPLE_32BIT,
    .channel_format       = I2S_CHANNEL_FMT_ONLY_LEFT,
    .communication_format = I2S_COMM_FORMAT_STAND_I2S,
    .intr_alloc_flags     = ESP_INTR_FLAG_LEVEL1,
    .dma_buf_count        = 8,
    .dma_buf_len          = 256,
    .use_apll             = false,
};
```

Key decisions:

- **`I2S_BITS_PER_SAMPLE_32BIT`** — The INMP441 is a 24-bit microphone but the I2S peripheral's DMA only supports 8, 16, or 32-bit words. We read 32-bit words and shift down (see below).
- **`I2S_CHANNEL_FMT_ONLY_LEFT`** — Reads only the left channel, matching the INMP441's `L/R = GND` wiring.
- **8 DMA buffers × 256 samples** — 2 048 samples of DMA FIFO ≈ 128 ms. This is the headroom before audio is lost if the task is preempted.

---

## 32-bit to 16-bit conversion

The INMP441 left-justifies its 24-bit output in a 32-bit I2S word. The MSB of the sample sits at bit 31 of the `int32_t` returned by the DMA:

```
Bit 31                    Bit 8   Bit 7   Bit 0
┌────────────────────────────────┬───────────────┐
│   24-bit signed sample         │   0x00        │
└────────────────────────────────┴───────────────┘
```

The firmware shifts right by 14:

```cpp
pcm_buf[i] = (int16_t)(i2s_buf[i] >> 14);
```

Why 14 and not 16?

- Shifting by **16** would give the top 16 bits of the 24-bit sample — true 16-bit precision.
- Shifting by **14** gives 18-bit precision packed into a 16-bit integer (the bottom 2 bits are always zero). This matches the original prototype's behaviour and leaves a small amount of headroom before clipping.

To get the full 16-bit dynamic range, change `>> 14` to `>> 16` in `src/audio.cpp`.

---

## VU peak calculation

The audio task computes the peak absolute sample value for each DMA buffer and writes it to shared state. The display task reads it for the OLED VU bar:

```cpp
int32_t peak = 0;
for (int i = 0; i < samples; i++) {
    pcm_buf[i] = (int16_t)(i2s_buf[i] >> 14);
    int32_t a = abs((int32_t)pcm_buf[i]);
    if (a > peak) peak = a;
}
sharedSetPeak(peak);
```

The peak is in the range `0..32767` (max value of a signed 16-bit integer). The display maps it to a pixel bar width:

```cpp
int barW = constrain((int)(peak * 98 / 32767), 0, 98);
```

---

## Audio task loop

```cpp
for (;;) {
    s_ws.loop();                                        // (1)

    size_t bytesRead = 0;
    i2s_read(I2S_PORT, s_i2sBuf, sizeof(s_i2sBuf),
             &bytesRead, portMAX_DELAY);               // (2) blocks ~16 ms

    int samples = bytesRead / sizeof(int32_t);
    // convert + compute peak                           // (3)

    if (s_wsConnected) {
        s_ws.sendBIN(pcmBuf, samples * sizeof(int16_t)); // (4)
    }
}
```

1. `ws.loop()` is called **before** `i2s_read`. This is important: the WebSocket library drives its reconnect timer and processes incoming frames inside `loop()`. Since `i2s_read` blocks for ~16 ms, calling `loop()` first ensures the timer fires at least once per buffer period.
2. `i2s_read` blocks until one full DMA buffer (256 samples) is ready. `portMAX_DELAY` means "wait forever" — the task never busy-waits.
3. Conversion and peak are computed in-place on the local buffers, which are never shared between tasks.
4. `sendBIN` transmits the raw PCM as a WebSocket binary frame. If the connection is down, the frame is silently dropped and audio continues — no backpressure, no blocking.
