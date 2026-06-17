---
weight: 3
---

# Web UI

## Accessing the UI

Open a browser on any device on the same network as the server:

```
http://[server-ip]:8765
```

The UI is a single static HTML file served by FastAPI's `StaticFiles` mount. No build step, no framework — vanilla JavaScript with a WebSocket connection.

---

## Layout

```
┌─────────────────────────────────────────────────────────┐
│  PARAYU          ● mic connected  ○ stt idle  ● server  │  ← header
├─────────────────────────────────────────────────────────┤
│  live                                                   │
│  partial transcript text here ▌                         │  ← live panel
├─────────────────────────────────────────────────────────┤
│                                                         │
│  11:09:02   ആ എൻ്റെ പേര് Rishi Krishnan.              │
│  11:09:11   എനിക്ക് വയറ് വേദനിക്കുന്നു.               │  ← transcript history
│  11:09:18   I am feeling better now.                    │
│                                                         │
├─────────────────────────────────────────────────────────┤
│  session 11:09:00                         3 lines       │  ← footer
└─────────────────────────────────────────────────────────┘
```

---

## Browser WebSocket

The browser connects to `ws://[server-host]/ws` and receives JSON messages. The connection URL is derived automatically from `location.host`, so the UI always connects back to whichever server served the page.

Auto-reconnect: if the server restarts, the browser retries every 2.5 seconds.

---

## JSON event reference

All events from server → browser are JSON objects with a `type` field.

### `fragment` — live partial transcript

Sent whenever the Sarvam API returns a new transcript fragment. Replaces the current live panel text.

```json
{
  "type": "fragment",
  "text": "ആ എൻ്റെ പേര്"
}
```

The `text` field is the **accumulated** partial sentence (all fragments joined so far), not just the latest fragment. The live panel always shows the full current sentence-in-progress.

### `transcript` — completed sentence

Sent when a sentence is finalised (END_SPEECH, silence timeout, or word limit). Clears the live panel and adds a new entry to the history.

```json
{
  "type": "transcript",
  "text": "ആ എൻ്റെ പേര് Rishi Krishnan.",
  "timestamp": "11:09:02"
}
```

### `status` — device and speech state changes

Sent when the mic connects/disconnects or when speech activity changes.

```json
{ "type": "status", "mic": "connected" }
{ "type": "status", "mic": "disconnected" }
{ "type": "status", "speech": "active" }
{ "type": "status", "speech": "silent" }
```

`mic` and `speech` are independent fields — a single `status` event carries at most one of them.

---

## Status indicator states

| Indicator | State | Display |
|---|---|---|
| **mic** | `connected` | Green dot, "mic connected" |
| **mic** | `disconnected` | Red dot, "mic offline" |
| **stt** | `active` (speech detected) | Pulsing amber dot, "stt live" |
| **stt** | `silent` | Green dot, "stt ready" |
| **stt** | idle (mic not connected) | Grey dot, "stt idle" |
| **server** | WebSocket open | Green dot, "server ok" |
| **server** | WebSocket closed | Red dot, "reconnecting" |

---

## Customisation

The entire UI is in `server/static/index.html`. CSS variables at the top of the `<style>` block control all colours:

```css
:root {
  --bg:        #080808;   /* page background */
  --surface:   #0f0f0f;   /* header, live panel, footer */
  --border:    #1c1c1c;   /* dividers */
  --text:      #e2e2e2;   /* transcript text */
  --muted:     #3a3a3a;   /* border colour */
  --dim:       #555;      /* timestamps, labels */
  --live:      #f59e0b;   /* amber — live transcript, cursor */
  --green:     #22c55e;   /* connected, ready */
  --red:       #ef4444;   /* disconnected, error */
}
```

To add a feature — for example saving the session transcript as a `.txt` download — add a button to the footer and hook it to `document.getElementById('transcript-wrap').innerText`.
