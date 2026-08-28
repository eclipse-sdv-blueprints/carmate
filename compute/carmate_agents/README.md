# # CarMate Agents Component

The **CarMate Agents** component provides AI-driven assistant capabilities for the Software-Defined Vehicle stack. It receives user query RPC requests via uProtocol/Zenoh, queries real-time vehicle data streams, and routes requests to configurable LLM providers (Ollama, OpenAI, Gemini, Grok, or Groq) to generate conversational responses.

---

## Features

* **Multi-LLM Provider Router:** Supports multiple LLM backends including Ollama (local), OpenAI, Google Gemini, xAI Grok, and Groq.
* **uProtocol RPC Server:** Listens on Zenoh transport for voice command RPC queries.
* **Real-Time Data Sync:** Subscribes to Zenoh pub/sub topics (`up/publisher/**`) to maintain an in-memory state of live vehicle telemetry (speed, location, temperature, wetness).
* **Thread-Safe Processing:** Offloads LLM queries to background worker threads to prevent blocking event loop execution.

---

## Directory Layout

```text
carmate_agents/
├── main.py          # Main entrypoint, RPC handler, and AI router logic
├── common_uuri.py   # Zenoh configuration & uProtocol URI generation helper
├── zenoh.json       # Zenoh network endpoints and scouting settings
├── Dockerfile       # Container specification
└── requirements.txt # Python dependencies
```

---

## Container Dependencies

This component depends on the **Zenoh Router** container to handle RPC calls and telemetry data streams.

* **Zenoh Router Endpoint:** `tcp/zenoh:7447`
* **Ollama Service (Optional):** Requires a running Ollama container/host if local model inference (`AI_PROVIDER="ollama"`) is selected.

---

## Building & Running with Docker

### 1. Build the Docker Image
From the `carmate_agents` directory:

```bash
docker build -t carmate-agents .
```

### 2. Run with Docker Compose (Recommended)
Run the agent service along with the required Zenoh dependencies using Docker Compose from the `compute/` root directory:

* **Run `carmate_agents` and `zenoh`:**
  ```bash
  docker compose up carmate_agents zenoh
  ```

* **Run stack in detached mode:**
  ```bash
  docker compose up -d
  ```

---

## Configuration

### LLM Provider & API Keys
Managed directly in `main.py` (or through environment variables depending on deployment setup):

```python
AI_PROVIDER = "ollama"  # Options: "ollama", "openai", "gemini", "grok", "groq"
```

### Zenoh Connection
Configured in `zenoh.json`:

```json
{
  "mode": "peer",
  "connect": {
    "endpoints": ["tcp/zenoh:7447"]
  }
}
```