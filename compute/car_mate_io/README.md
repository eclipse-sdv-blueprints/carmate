## CarMate I/O Component

The **CarMate I/O** component serves as the voice interaction and web interface bridge for the CarMate system. It provides Speech-to-Text (STT), Text-to-Speech (TTS), and RPC communication capabilities over the Zenoh transport protocol using the uProtocol standard.

---

## Features

* **Web UI Server:** Flask-based application serving the frontend interface.
* **Speech-to-Text (STT):** Converts uploaded user audio into text using Google Speech Recognition.
* **Text-to-Speech (TTS):** Generates voice audio responses from text using `espeak` with LRU caching.
* **uProtocol / Zenoh RPC:** Dispatches recognized voice commands over Zenoh RPC to downstream agent services.

---

## Directory Layout

```text
car_mate_io/
├── car_mate_io.py    # Main Flask application and API route endpoints
├── common_uuri.py    # Helper utilities for Zenoh configuration and uProtocol URIs
├── zenoh.json        # Zenoh connection and scouting parameters
├── Dockerfile        # Container specification
├── requirements.txt  # Python dependencies
└── static/           # Frontend assets (index.html, JS, CSS)
```

---

## Container Dependencies

This container depends on the **Zenoh Router** container to send and receive uProtocol RPC requests.

* **Zenoh Router Endpoint:** `tcp/zenoh:7447`
* **Target Services:** Requires downstream agent services (e.g., `carmate_agents`) active on the Zenoh network bus to handle voice/RPC execution requests.

---

## Building & Running with Docker

### 1. Build the Docker Image Standalone
From the `car_mate_io` directory:

```bash
docker build -t carmate-io .
```

### 2. Run with Docker Compose (Recommended)
Since this service relies on the `zenoh` router container, execute it using Docker Compose from the `compute/` root directory:

* **Run `car_mate_io` along with its required Zenoh dependency:**
  ```bash
  docker compose up car_mate_io zenoh
  ```

* **Run the entire stack in detached mode:**
  ```bash
  docker compose up -d
  ```

---

## 📡 API Endpoints

| Endpoint | Method | Description |
| :--- | :--- | :--- |
| `/` | `GET` | Serves the web UI (`index.html`). |
| `/stt` | `POST` | Accepts an audio file input (`multipart/form-data`) and returns recognized text. |
| `/rpc` | `POST` | Sends text payloads as uProtocol RPC requests over Zenoh. |
| `/tts` | `POST` | Accepts a text payload and returns a generated `.wav` audio stream. |

---

## ⚙️ Configuration

Zenoh endpoints and network discovery modes are configured in `zenoh.json`:

```json
{
  "mode": "peer",
  "connect": {
    "endpoints": ["tcp/zenoh:7447"]
  }
}
```