---
weight: 1
---

# Server Setup

## Requirements

- Python 3.10+
- A Sarvam AI API key ([console.sarvam.ai](https://console.sarvam.ai))
- The ESP32 and the server machine on the same LAN

---

## Directory layout

```
server/
├── main.py            FastAPI app — WebSocket endpoints, static serving
├── sarvam_client.py   Sarvam streaming STT session wrapper
├── static/
│   └── index.html     Web UI (single file, no build step)
├── requirements.txt
├── .env               API key (not committed)
└── .env.example       Template
```

---

## Install

```bash
cd server

python3 -m venv .venv
source .venv/bin/activate       # Windows: .venv\Scripts\activate

pip install -r requirements.txt
```

`requirements.txt`:

```
fastapi
uvicorn[standard]
sarvamai
python-dotenv
```

---

## Configuration

Copy `.env.example` to `.env` and add your key:

```bash
cp .env.example .env
```

`.env`:

```
SARVAM_API_KEY=sk_your_key_here
```

The key is loaded at startup via `python-dotenv`:

```python
from dotenv import load_dotenv
load_dotenv()
SARVAM_API_KEY = os.environ.get("SARVAM_API_KEY", "")
```

---

## Run

```bash
.venv/bin/uvicorn main:app --host 0.0.0.0 --port 8765
```

- `--host 0.0.0.0` makes the server reachable from the ESP32 and from browser clients on the LAN. Do not use `127.0.0.1` — the device cannot reach a loopback address.
- `--port 8765` must match what you entered in the provisioning portal.

For development with auto-reload:

```bash
.venv/bin/uvicorn main:app --host 0.0.0.0 --port 8765 --reload
```

---

## Verify it's running

Open a browser on any machine on the LAN:

```
http://[your-machine-ip]:8765
```

You should see the Parayu web UI with three status dots in the header. The **server** dot should be green within a second. The **mic** dot turns green when the ESP32 connects.

To find your machine's IP:

```bash
ip addr show        # Linux
ipconfig            # Windows
ifconfig            # macOS
```

---

## Endpoints

| Method | Path | Description |
|---|---|---|
| `GET` | `/` | Serves `static/index.html` |
| `WebSocket` | `/audio` | ESP32 audio input (binary PCM frames) |
| `WebSocket` | `/ws` | Browser output (JSON transcript events) |

---

## Process management (optional)

For a persistent deployment, run the server under `systemd` or `screen`:

```bash
# Using screen
screen -S parayu-server
.venv/bin/uvicorn main:app --host 0.0.0.0 --port 8765
# Ctrl+A, D to detach
```
