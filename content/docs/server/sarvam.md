---
weight: 2
---

# Sarvam AI Integration

## Model and mode

The server uses the `saaras:v3` model in `codemix` mode, targeting Malayalam + English (`ml-IN`):

```python
async with client.speech_to_text_streaming.connect(
    model                = "saaras:v3",
    mode                 = "codemix",
    language_code        = "ml-IN",
    sample_rate          = 16000,
    high_vad_sensitivity = False,
    vad_signals          = True,
    flush_signal         = True,
) as sarvam_ws:
```

- **`codemix`** — handles sentences that switch between Malayalam and English mid-utterance, which is common in the target use case.
- **`vad_signals=True`** — the API sends `START_SPEECH` and `END_SPEECH` events in addition to transcript fragments. These drive the sentence assembly logic.
- **`flush_signal=True`** — allows the client to send an explicit flush, triggering the API to emit any buffered partial transcript before the session ends.

---

## Audio buffering

Raw PCM frames (512 bytes each) arrive from the ESP32 continuously. The server buffers them and sends to Sarvam in larger chunks:

```
CHUNKS_PER_SEND = 192
```

192 frames × 256 samples × 16 kHz = **3.07 seconds** of audio per API call. This threshold was chosen empirically — too small (< 1 s) and the model lacks context for accurate transcription; too large (> 5 s) and latency increases noticeably.

Each chunk is wrapped in a WAV header before being sent:

```python
wav = make_wav_header(bytes(pcm_buffer))
b64 = base64.b64encode(wav).decode("utf-8")
await sarvam_ws.transcribe(audio=b64, encoding="audio/wav", sample_rate=16000)
await sarvam_ws.flush()
```

The Sarvam streaming API expects **base64-encoded WAV** passed to `.transcribe()`. The `.flush()` call tells the model to emit any pending partial output immediately rather than waiting for more audio.

---

## Message types from the API

The API emits two types of messages on the `sarvam_ws` async iterator:

### `events` — Voice activity detection signals

```python
msg_type = getattr(message, "type", None)      # "events"
signal   = getattr(message.data, "signal_type", None)
# signal == "START_SPEECH" or "END_SPEECH"
```

- **`START_SPEECH`** — voice activity detected; cancels any pending silence timer.
- **`END_SPEECH`** — speaker stopped; triggers immediate sentence flush and broadcasts `{"type": "status", "speech": "silent"}` to the browser.

### `data` — Transcript fragments

```python
fragment = getattr(message.data, "transcript", "").strip()
```

Fragments are partial transcriptions that arrive incrementally as the model processes audio. They are accumulated in a list and joined to form the live preview shown in the browser. When a sentence is complete (via `END_SPEECH`, silence timeout, or word count limit), the list is cleaned and emitted as a full transcript.

---

## Sentence assembly

```python
SENTENCE_TIMEOUT   = 2.5   # seconds of silence before auto-flush
MAX_SENTENCE_WORDS = 40    # word count limit before forced flush
```

The `clean_sentence()` function joins fragments, normalises whitespace, capitalises the first letter, and adds a trailing period if missing:

```python
def clean_sentence(fragments: list[str]) -> str:
    text = " ".join(fragments)
    text = re.sub(r"\s+", " ", text).strip()
    text = re.sub(r"\s([.,!?])", r"\1", text)   # fix space before punctuation
    if text:
        text = text[0].upper() + text[1:]
    if text and text[-1] not in ".!?,":
        text += "."
    return text
```

Three triggers cause a sentence to be emitted:

| Trigger | Source |
|---|---|
| `END_SPEECH` event from Sarvam | Voice activity detection |
| 2.5 second silence (no new fragments) | `asyncio` timer, reset on each fragment |
| Fragment word count ≥ 40 | Prevents very long run-on sentences |

---

## Session lifecycle

A `SarvamSession` object is created when the ESP32 connects and lives for the duration of that connection:

```
ESP32 connects → SarvamSession created → Sarvam WS opened
     │
     │  audio frames flowing
     ▼
     │  ESP32 disconnects
     ▼
None sentinel put into audio_queue
     │
     ▼
sender() drains remaining buffer → final .flush()
     │
     ▼
asyncio.wait(FIRST_COMPLETED) cancels receiver()
     │
     ▼
Sarvam WS closed
Final sentence emitted if any fragments remain
```

If the ESP32 disconnects mid-sentence, `_emit_sentence()` is called in the `run()` cleanup path to ensure no fragments are lost.
