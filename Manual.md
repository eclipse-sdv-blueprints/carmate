---
sidebar_position: 5
title: Setup Manual
---

# CarMate Setup Manual

This guide explains how to set up and run CarMate. The system supports three deployment modes:

| Mode | Requirements | Recommended for |
| --- | --- | --- |
| **Mock mode** (default) | Linux machine only | First-time setup, evaluation |
| **CARLA mode** | Linux machine + Windows machine (Ethernet) | Realistic simulation |
| **Full hardware mode** | Both above + Azure IoT Dev Kit | Hardware integration demo |

For the complete documentation including architecture details and component reference, see the [`docs/`](docs/) folder.

---

## Prerequisites

### Hardware

| Component | Required | Notes |
| --- | --- | --- |
| Linux machine (x86_64 or ARM64) | **Yes** | Runs the full Docker Compose stack |
| Windows machine | Only for CARLA mode | Runs CARLA 0.9.16; dedicated GPU recommended |
| Azure IoT Dev Kit (MXChip AZ3166) | No | Optional MCU node for real temperature sensing |
| Ethernet cable | Only for CARLA mode | Connects Linux and Windows machines |

### Software

| Software | Version | Purpose |
| --- | --- | --- |
| Docker Engine | ≥ 24.0 | Container runtime |
| Docker Compose | ≥ 2.20 | Multi-container orchestration |
| Ollama | Latest | Local LLM backend (if not using a cloud provider) |
| CARLA Simulator | 0.9.16 | Driving simulation (CARLA mode only) |

---

## Step 1: Choose the LLM Backend

CarMate needs one AI backend for responses. Choose either the local Ollama backend or a public LLM provider.

### Option A: Local Ollama Model

Use this option if you want to run the LLM locally on the Linux machine.

Start Ollama and pull the model configured in CarMate:

```bash
ollama serve
ollama pull phi3
# or another local model, for example:
ollama pull mistral
```

In `compute/carmate_agents/carmate_agents.py`, keep the provider set to Ollama and set the local model name:

```python
AI_PROVIDER = "ollama"

MODEL_MAP = {
    "ollama": "phi3",
    # keep the other provider entries unchanged
}
```

### Option B: Public LLM Models

Use this option if you want CarMate to call a public LLM provider instead of running Ollama locally. CarMate currently supports OpenAI, Gemini, Grok, and Groq.

Get a valid API key from the provider platform, then open the agent configuration file:

```bash
nano compute/carmate_agents/carmate_agents.py
```

Locate the `HARDCODED AI CONFIGURATION` section at the top of the file:

1. Set `AI_PROVIDER` to `"openai"`, `"gemini"`, `"grok"`, or `"groq"`.
2. Put your API key into the matching entry in `API_KEYS`.
3. Optionally change the provider model string in `MODEL_MAP`.

```python
# Options: "ollama", "openai", "gemini", "grok", "groq"
AI_PROVIDER = "openai"

API_KEYS = {
    "openai": "YOUR_OPENAI_API_KEY",
    "gemini": "YOUR_GEMINI_API_KEY",
    "grok": "YOUR_XAI_API_KEY",
    "groq": "YOUR_GROQ_API_KEY",
}

MODEL_MAP = {
    "ollama": "phi3",
    "openai": "gpt-4o-mini",
    "gemini": "gemini-1.5-flash",
    "grok": "grok-beta",
    "groq": "llama-3.3-70b-versatile",
}
```

The Docker image includes this configuration file, so Step 4 uses `--build` to apply any backend or model changes.

---

> **Note:** If you do not want to use the CARLA server, skip Steps 2 and 3 and proceed directly to [Step 4](#step-4-start-the-carmate-stack).

## Step 2: Configure Ethernet Connection

Connect the Linux and Windows machines using an Ethernet cable.

### Windows Machine (CARLA)

Configure the Ethernet adapter with the following static IP settings:

```text
IP Address: 192.168.43.249
Subnet Mask: 255.255.255.0
Gateway: Leave empty
```

To configure this:

1. Open Control Panel
2. Go to Network and Sharing Center
3. Click "Change adapter settings"
4. Right-click the Ethernet adapter → Properties
5. Select "Internet Protocol Version 4 (TCP/IPv4)"
6. Click Properties
7. Enter the IP configuration above

### Linux Machine (CarMate UI)

First identify the Ethernet interface:

```bash
ip a
```

The interface name is usually something like:

```text
eth0
enp3s0
ens33
```

Assign a static IP address:

```bash
sudo ip addr add 192.168.43.245/24 dev eth0
sudo ip link set eth0 up
```

Verify the configuration:

```bash
ip a
```

Verify the connection with ping from the Linux machine:

```bash
ping 192.168.43.249
```

---

## Step 3: Start CARLA

> **Note:** If you do not want to use CARLA, skip this step and proceed to [Step 4](#step-4-start-the-carmate-stack).

On the Windows machine, download CARLA 0.9.16:

https://github.com/carla-simulator/carla

Extract the folder and launch the simulator:

```bash
CarlaUE4.exe
```

Wait until the simulation finishes loading.

---

## Step 4: Start the CarMate Stack

By default, the container is configured to spin up the **Mock Server** automatically. If you want to use the **Real CARLA Server** as your permanent default, you need to modify the Dockerfile directly before building the image.

Open `compute/Carla_Provider/Dockerfile` and locate the final line:

```dockerfile
# Default: Runs in MOCK mode
CMD ["python", "carla_provider.py", "--nocarla"]
```

Change it to connect to the real CARLA server:

```dockerfile
CMD ["python", "carla_provider.py"]
```

Then run the following command to build and start the stack:

```bash
cd compute
sudo docker compose up -d --build
```

> **Note:** If you are running in mock mode, no Dockerfile changes are needed. Just run the command above.

---

## Step 5: Open the CarMate UI

Open a browser on the Linux machine and go to:

```text
http://localhost:5000
```

### Available Telemetry Signals

The current streaming architecture exposes the following dynamic signals to the AI processing layer:

| Signal | Unit | Source |
| --- | --- | --- |
| Vehicle Speed | km/h | CARLA / Mock |
| Altitude | m | CARLA / Mock |
| Latitude | ° | CARLA / Mock |
| Longitude | ° | CARLA / Mock |
| Road Wetness / Humidity | % | CARLA / Mock |
| Ambient Temperature | °C | MCU (optional) |

To expand the dataset with environmental metrics, proceed to **Step 6** to integrate external hardware for real-time temperature sensing.

---

## (Optional) Step 6: Azure IoT Dev Kit (MCU Node) Setup

This section is only required if you want hardware integration with CarMate.

Make sure:

- The Linux machine and MCU are connected to the same Wi-Fi network
- You know the Wi-Fi IP address of the Linux machine

Find the Linux Wi-Fi IP address using:

```bash
ip a
```

---

### Configure `network.rs`

Open:

```text
mcu_sw/threadx-app/cross/app/src/bin/network.rs
```

Update the broker IP at line 391 to the Linux machine's Wi-Fi IP:

```rust
let broker_ip: core::net::Ipv4Addr =
    core::net::Ipv4Addr::new(192, 168, 43, 241);
```

Update the Wi-Fi credentials:

```rust
let ssid: &'static str = "YOUR_SSID";
let password = "YOUR_PASSWORD";
```

---

### Build and Flash the Firmware

ThreadX Rust repository:

https://github.com/Eclipse-SDV-Hackathon-Chapter-Three/threadx-rust

Build command:

```bash
cd threadx-rust/threadx-app/cross/app

cargo run --release \
  --target thumbv7em-none-eabihf \
  --bin network
```

---

## Troubleshooting

### Docker containers fail to start

- Ensure Docker Engine is running: `sudo systemctl start docker`
- Check for port conflicts: `sudo docker compose logs`
- Rebuild from scratch: `sudo docker compose down && sudo docker compose up -d --build`

### CarMate Web UI is unreachable (`localhost:5000`)

- Confirm the `carmate_io` container is running: `sudo docker compose ps`
- Check the container logs: `sudo docker compose logs carmate_io`
- Ensure no other service is bound to port 5000

### Kuksa Databroker connection errors

- Verify the `sdv_runtime` container is healthy: `sudo docker compose logs sdv_runtime`
- Default gRPC port is `55555`; ensure it is not blocked by a firewall

### LLM returns no response or times out

- **Ollama:** Confirm Ollama is running (`ollama serve`) and the model is pulled (`ollama list`)
- **Cloud providers:** Verify the API key in `compute/carmate_agents/carmate_agents.py` is valid
- Rebuild the agent container: `sudo docker compose up -d --build carmate_agents`

### CARLA simulator not detected

- Confirm the Windows machine Ethernet IP is `192.168.43.249` and the Linux machine can ping it
- Confirm CARLA is fully loaded before starting the Docker stack
- Check `carla_provider` logs: `sudo docker compose logs carla_provider`

### MCU temperature data not appearing

- Confirm the MCU and Linux machine are on the same Wi-Fi network
- Verify the broker IP in `network.rs` matches the Linux machine's current Wi-Fi address
- Check the MQTT broker: `mosquitto_sub -h localhost -t "mcu/#" -v`

---

## Configuration Reference

### AI Configuration (`compute/carmate_agents/carmate_agents.py`)

| Parameter | Default | Description |
| --- | --- | --- |
| `AI_PROVIDER` | `"ollama"` | Active LLM backend: `ollama`, `openai`, `gemini`, `grok`, or `groq` |
| `API_KEYS["openai"]` | `"YOUR_OPENAI_API_KEY"` | OpenAI API key |
| `API_KEYS["gemini"]` | `"YOUR_GEMINI_API_KEY"` | Google Gemini API key |
| `API_KEYS["grok"]` | `"YOUR_XAI_API_KEY"` | xAI Grok API key |
| `API_KEYS["groq"]` | `"YOUR_GROQ_API_KEY"` | Groq API key |
| `MODEL_MAP["ollama"]` | `"phi3"` | Ollama model name |
| `MODEL_MAP["openai"]` | `"gpt-4o-mini"` | OpenAI model name |
| `MODEL_MAP["gemini"]` | `"gemini-1.5-flash"` | Gemini model name |
| `MODEL_MAP["grok"]` | `"grok-beta"` | Grok model name |
| `MODEL_MAP["groq"]` | `"llama-3.3-70b-versatile"` | Groq model name |

### Network Configuration

| File | Parameter | Default | Description |
| --- | --- | --- | --- |
| `compute/Carla_Provider/carla_provider.py` | `CARLA_HOST` | `192.168.43.249` | Windows machine IP (CARLA) |
| `compute/Carla_Provider/carla_provider.py` | `CARLA_PORT` | `2000` | CARLA server port |
| `compute/MQTT_KUIKSA_Provider/mqtt_kuksa_provider.py` | `MQTT_BROKER` | `localhost` | MQTT broker hostname |
| `compute/MQTT_KUIKSA_Provider/mqtt_kuksa_provider.py` | `KUKSA_HOST` | `localhost` | Kuksa Databroker hostname |
| `compute/MQTT_KUIKSA_Provider/mqtt_kuksa_provider.py` | `KUKSA_PORT` | `55555` | Kuksa Databroker gRPC port |
| `mcu_sw/.../network.rs` | `broker_ip` | `192.168.43.241` | Linux machine Wi-Fi IP (update for your network) |

