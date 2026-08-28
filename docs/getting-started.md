---
sidebar_position: 3
title: Getting Started
---

# Getting Started

This guide explains how to set up and run CarMate. The system supports three deployment modes:

| Mode | Requirements | Recommended for |
| --- | --- | --- |
| **Mock mode** (default) | Linux machine only | First-time setup, evaluation, CI |
| **CARLA mode** | Linux machine + Windows machine (Ethernet) | Realistic simulation |
| **Full hardware mode** | Both above + Azure IoT Dev Kit | Hardware integration demo |

## Prerequisites

### Hardware

| Component | Required | Notes |
| --- | --- | --- |
| Linux machine (x86_64 or ARM64) | **Yes** | Runs the full Docker Compose stack |
| Windows machine | Only for CARLA mode | Runs CARLA 0.9.16; dedicated GPU strongly recommended |
| Azure IoT Dev Kit (MXChip AZ3166) | No | Optional MCU node for real temperature sensing |
| Ethernet cable | Only for CARLA mode | Connects Linux and Windows machines |

### Software

| Software | Version | Purpose |
| --- | --- | --- |
| Docker Engine | ≥ 24.0 | Container runtime |
| Docker Compose | ≥ 2.20 | Multi-container orchestration |
| Ollama | Latest | Local LLM backend (if not using a cloud provider) |
| CARLA Simulator | 0.9.16 | Driving simulation (CARLA mode only) |

### Network

The Linux and Windows machines must be on the same local network, connected by an Ethernet cable with the following static IP configuration:

| Machine | Interface | IP Address |
| --- | --- | --- |
| Windows (CARLA) | Ethernet adapter | `192.168.43.249` |
| Linux (CarMate) | Ethernet interface | `192.168.43.245` |

---

## Quick Start (Mock Mode)

To run CarMate without a CARLA server or MCU hardware, use mock mode. The CARLA provider container automatically generates simulated telemetry data.

```bash
# 1. Clone the repository
git clone https://github.com/eclipse-sdv-blueprints/carmate.git
cd carmate

# 2. (Optional) Configure your LLM backend — default is Ollama with phi3
#    See "Step 1: Choose LLM Backend" below if you want to change this.

# 3. Start Ollama and pull the model (if using local LLM)
ollama serve
ollama pull phi3

# 4. Build and start the full Docker Compose stack
cd compute
sudo docker compose up -d --build

# 5. Open the CarMate Web UI
#    Navigate to http://localhost:5000 in your browser
```

The stack is ready when all containers are running. Use `sudo docker compose ps` to check their status.

---

## Step 1: Choose the LLM Backend

CarMate requires one AI backend for generating conversational responses. Two options are available.

### Option A: Local Ollama Model

Use this option to run the LLM entirely on the Linux machine without any cloud API keys.

Start Ollama and pull the model configured in CarMate:

```bash
ollama serve
ollama pull phi3
# Alternatively, pull a different model:
ollama pull mistral
```

In `compute/carmate_agents/carmate_agents.py`, keep the provider set to `ollama`:

```python
AI_PROVIDER = "ollama"

MODEL_MAP = {
    "ollama": "phi3",  # change to your preferred local model
    # keep the other provider entries unchanged
}
```

### Option B: Cloud LLM Provider

Use this option to call a cloud LLM provider. CarMate supports OpenAI, Gemini, Grok, and Groq.

Obtain a valid API key from the provider's platform, then edit the agent configuration:

```bash
nano compute/carmate_agents/carmate_agents.py
```

Locate the `HARDCODED AI CONFIGURATION` section at the top of the file and update it:

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

The Docker image embeds this configuration file, so the stack must be rebuilt (with `--build`) to apply any backend or model changes.

---

## Step 2: Configure Ethernet Connection (CARLA Mode Only)

> **Note:** Skip this step if you are running in mock mode. Proceed directly to [Step 4](#step-4-start-the-carmate-stack).

Connect the Linux and Windows machines using an Ethernet cable.

### Windows Machine (CARLA)

Configure the Ethernet adapter with the following static IP settings:

```text
IP Address:   192.168.43.249
Subnet Mask:  255.255.255.0
Gateway:      (leave empty)
```

To configure this in Windows:

1. Open **Control Panel**
2. Go to **Network and Sharing Center**
3. Click **Change adapter settings**
4. Right-click the Ethernet adapter → **Properties**
5. Select **Internet Protocol Version 4 (TCP/IPv4)** → **Properties**
6. Enter the IP configuration above and click **OK**

### Linux Machine (CarMate)

Identify the Ethernet interface name:

```bash
ip a
```

Common interface names are `eth0`, `enp3s0`, or `ens33`. Assign a static IP address:

```bash
sudo ip addr add 192.168.43.245/24 dev eth0
sudo ip link set eth0 up
```

Verify the configuration and connectivity:

```bash
ip a
ping 192.168.43.249
```

---

## Step 3: Start CARLA (CARLA Mode Only)

> **Note:** Skip this step if you are running in mock mode. Proceed directly to [Step 4](#step-4-start-the-carmate-stack).

On the Windows machine, download CARLA 0.9.16 from:

https://github.com/carla-simulator/carla/releases/tag/0.9.16

Extract the archive and launch the simulator:

```bash
CarlaUE4.exe
```

Wait for the simulation to finish loading before proceeding to the next step.

---

## Step 4: Start the CarMate Stack

The Docker Compose stack is configured to run in **mock mode** by default. The `carla_provider` container generates simulated telemetry data without requiring a CARLA server.

### Mock Mode (Default)

No changes are needed. Run:

```bash
cd compute
sudo docker compose up -d --build
```

### CARLA Mode

To use the real CARLA server, modify the `carla_provider` Dockerfile before building.

Open `compute/Carla_Provider/Dockerfile` and locate the final `CMD` line:

```dockerfile
# Default: runs in mock mode
CMD ["python", "carla_provider.py", "--nocarla"]
```

Change it to run against the real CARLA server:

```dockerfile
CMD ["python", "carla_provider.py"]
```

Then start the stack:

```bash
cd compute
sudo docker compose up -d --build
```

### Verifying the Stack

Check that all containers are running:

```bash
sudo docker compose ps
```

You should see all eight services (`sdv_runtime`, `mqtt`, `zenoh`, `mqtt_provider`, `carla_provider`, `vehicle_data_accessor`, `carmate_agents`, `carmate_io`) with status `Up`.

---

## Step 5: Open the CarMate UI

Open a browser on the Linux machine and navigate to:

```text
http://localhost:5000
```

The Web UI shows the driver interaction panel and live vehicle telemetry.

### Available Telemetry Signals

The following signals are streamed in real time to the AI processing layer:

| Signal | Unit | Source |
| --- | --- | --- |
| Vehicle Speed | km/h | CARLA / Mock |
| Altitude | m | CARLA / Mock |
| Latitude | ° | CARLA / Mock |
| Longitude | ° | CARLA / Mock |
| Road Wetness / Humidity | % | CARLA / Mock |
| Ambient Temperature | °C | MCU (optional) |

---

## Step 6: MCU Node Setup (Optional)

This step is only required for hardware integration with the Azure IoT Dev Kit (MXChip AZ3166).

Ensure that:

- The Linux machine and the MCU are connected to the **same Wi-Fi network**
- You know the Wi-Fi IP address of the Linux machine (run `ip a` to find it)

### Configure `network.rs`

Open the MCU firmware source file:

```text
mcu_sw/threadx-app/cross/app/src/bin/network.rs
```

Update the MQTT broker IP address (line 391) to the Linux machine's Wi-Fi IP:

```rust
let broker_ip: core::net::Ipv4Addr =
    core::net::Ipv4Addr::new(192, 168, 43, 241); // replace with your Linux machine's Wi-Fi IP
```

Update the Wi-Fi credentials:

```rust
let ssid: &'static str = "YOUR_SSID";
let password = "YOUR_PASSWORD";
```

### Build and Flash the Firmware

The ThreadX Rust toolchain is available at:

https://github.com/Eclipse-SDV-Hackathon-Chapter-Three/threadx-rust

Build and flash the firmware with:

```bash
cd threadx-rust/threadx-app/cross/app

cargo run --release \
  --target thumbv7em-none-eabihf \
  --bin network
```

Once flashed, the MCU publishes temperature readings to the `mcu/temperature` MQTT topic on the Linux machine.

---

## Troubleshooting

### Docker containers fail to start

- Ensure Docker Engine is running: `sudo systemctl start docker`
- Check for port conflicts: `sudo docker compose logs`
- Rebuild from scratch: `sudo docker compose down && sudo docker compose up -d --build`

### CarMate Web UI is unreachable (`localhost:5000`)

- Confirm that the `carmate_io` container is running: `sudo docker compose ps`
- Check the container logs: `sudo docker compose logs carmate_io`
- Ensure no other service is bound to port 5000

### Kuksa Databroker connection errors

- The Kuksa Databroker runs as part of the `sdv_runtime` container
- Verify the container is healthy: `sudo docker compose logs sdv_runtime`
- Default gRPC port is `55555`; ensure it is not blocked by a firewall

### LLM returns no response or times out

- **Ollama:** Confirm Ollama is running (`ollama serve`) and the model is pulled (`ollama list`)
- **Cloud providers:** Verify the API key in `compute/carmate_agents/carmate_agents.py` is valid and has sufficient quota
- Rebuild the `carmate_agents` container after any configuration change: `sudo docker compose up -d --build carmate_agents`

### CARLA simulator not detected

- Confirm the Windows machine Ethernet IP is `192.168.43.249` and the Linux machine can ping it
- Confirm CARLA is fully loaded before starting the Docker stack
- Check `carla_provider` logs: `sudo docker compose logs carla_provider`

### MCU temperature data not appearing

- Confirm the MCU and Linux machine are on the same Wi-Fi network
- Verify the broker IP in `network.rs` matches the Linux machine's current Wi-Fi address
- Check that the MQTT broker is reachable: `mosquitto_sub -h localhost -t "mcu/#" -v`

---

## Configuration Reference

All user-facing configuration is located in `compute/carmate_agents/carmate_agents.py`.

| Parameter | Type | Default | Description |
| --- | --- | --- | --- |
| `AI_PROVIDER` | string | `"ollama"` | Active LLM backend. Options: `ollama`, `openai`, `gemini`, `grok`, `groq` |
| `API_KEYS["openai"]` | string | `"YOUR_OPENAI_API_KEY"` | OpenAI API key (required when `AI_PROVIDER = "openai"`) |
| `API_KEYS["gemini"]` | string | `"YOUR_GEMINI_API_KEY"` | Google Gemini API key (required when `AI_PROVIDER = "gemini"`) |
| `API_KEYS["grok"]` | string | `"YOUR_XAI_API_KEY"` | xAI Grok API key (required when `AI_PROVIDER = "grok"`) |
| `API_KEYS["groq"]` | string | `"YOUR_GROQ_API_KEY"` | Groq API key (required when `AI_PROVIDER = "groq"`) |
| `MODEL_MAP["ollama"]` | string | `"phi3"` | Ollama model name to use locally |
| `MODEL_MAP["openai"]` | string | `"gpt-4o-mini"` | OpenAI model name |
| `MODEL_MAP["gemini"]` | string | `"gemini-1.5-flash"` | Gemini model name |
| `MODEL_MAP["grok"]` | string | `"grok-beta"` | Grok model name |
| `MODEL_MAP["groq"]` | string | `"llama-3.3-70b-versatile"` | Groq model name |

Network configuration for CARLA and MCU modes is set in the respective provider source files:

| File | Parameter | Default | Description |
| --- | --- | --- | --- |
| `compute/Carla_Provider/carla_provider.py` | `CARLA_HOST` | `192.168.43.249` | IP address of the Windows machine running CARLA |
| `compute/Carla_Provider/carla_provider.py` | `CARLA_PORT` | `2000` | CARLA server port |
| `compute/MQTT_KUIKSA_Provider/mqtt_kuksa_provider.py` | `MQTT_BROKER` | `localhost` | MQTT broker hostname |
| `compute/MQTT_KUIKSA_Provider/mqtt_kuksa_provider.py` | `KUKSA_HOST` | `localhost` | Kuksa Databroker hostname |
| `compute/MQTT_KUIKSA_Provider/mqtt_kuksa_provider.py` | `KUKSA_PORT` | `55555` | Kuksa Databroker gRPC port |
| `mcu_sw/.../network.rs` | `broker_ip` | `192.168.43.241` | Linux machine Wi-Fi IP (update for your network) |
