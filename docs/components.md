---
sidebar_position: 4
title: Components
---

# Components

CarMate runs as a Docker Compose stack on the Linux Compute Node. This page describes each containerized service: its purpose, inputs and outputs, configuration, and source location.

For the overall data flow between components, see the [Architecture](./architecture) page.

---

## Eclipse AutoWRX SDV Runtime

| Property | Value |
| --- | --- |
| **Service name** | `sdv_runtime` |
| **Image** | `ghcr.io/eclipse-autowrx/sdv-runtime:latest` |
| **Source** | [eclipse-autowrx](https://github.com/eclipse-autowrx) |
| **Network** | Host |

### Purpose

The Eclipse AutoWRX SDV Runtime is the primary middleware layer that manages the vehicle software stack. It hosts the Eclipse Kuksa Databroker and provides the core SDV service orchestration for the CarMate blueprint.

### Configuration

| Variable | Default | Description |
| --- | --- | --- |
| `RUNTIME_NAME` | `CarMate` | Display name for the runtime instance |

---

## Eclipse Mosquitto MQTT Broker

| Property | Value |
| --- | --- |
| **Service name** | `mqtt` |
| **Image** | `eclipse-mosquitto:2.0` |
| **Source** | [eclipse/mosquitto](https://github.com/eclipse/mosquitto) |
| **Config volume** | `./mosquitto/config:/mosquitto/config` |

### Purpose

The MQTT broker is the central message bus for all sensor data entering the Compute Node. Both the CARLA simulator bridge and the MCU node publish raw telemetry to MQTT topics, which downstream providers then consume.

### Topics

| Topic | Publisher | Consumer | Signal |
| --- | --- | --- | --- |
| `carla/vehicle/speed` | CARLA-Kuksa Provider | MQTT-Kuksa Provider | Vehicle speed (km/h) |
| `carla/vehicle/lat` | CARLA-Kuksa Provider | MQTT-Kuksa Provider | GPS latitude |
| `carla/vehicle/lon` | CARLA-Kuksa Provider | MQTT-Kuksa Provider | GPS longitude |
| `carla/vehicle/alt` | CARLA-Kuksa Provider | MQTT-Kuksa Provider | Altitude (m) |
| `carla/vehicle/wetness` | CARLA-Kuksa Provider | MQTT-Kuksa Provider | Road wetness / humidity (%) |
| `mcu/temperature` | MCU Node (ThreadX) | MQTT-Kuksa Provider | Ambient temperature (°C) |
| `compute/color` | CarMate Agent | MQTT-Kuksa Provider | Ambient lighting color command |

---

## Eclipse Zenoh Router

| Property | Value |
| --- | --- |
| **Service name** | `zenoh` |
| **Image** | `eclipse/zenoh` |
| **Source** | [eclipse-zenoh/zenoh](https://github.com/eclipse-zenoh/zenoh) |
| **Network** | Host |

### Purpose

The Zenoh router is the uProtocol transport layer used for internal service-to-service communication on the Compute Node. All uProtocol RPC calls between CarMate I/O, the CarMate Agent, and the Vehicle Data Accessor are routed through Zenoh.

---

## MQTT-Kuksa Provider

| Property | Value |
| --- | --- |
| **Service name** | `mqtt_provider` |
| **Build context** | `compute/MQTT_KUIKSA_Provider/` |
| **Source** | `compute/MQTT_KUIKSA_Provider/mqtt_kuksa_provider.py` |
| **Network** | Host |

### Purpose

The MQTT-Kuksa Provider bridges the MQTT message bus to the Eclipse Kuksa Databroker. It subscribes to all vehicle and sensor MQTT topics, translates the raw values to VSS signal paths, and writes them into the Kuksa Databroker via gRPC.

### Signal Mapping

| MQTT Topic | VSS Signal Path |
| --- | --- |
| `carla/vehicle/speed` | `Vehicle.Speed` |
| `carla/vehicle/lat` | `Vehicle.CurrentLocation.Latitude` |
| `carla/vehicle/lon` | `Vehicle.CurrentLocation.Longitude` |
| `carla/vehicle/alt` | `Vehicle.CurrentLocation.Altitude` |
| `carla/vehicle/wetness` | `Vehicle.Exterior.Humidity` |
| `mcu/temperature` | `Vehicle.Cabin.HVAC.AmbientAirTemperature` |

### Configuration

Configured directly in `mqtt_kuksa_provider.py`:

| Parameter | Default | Description |
| --- | --- | --- |
| `MQTT_BROKER` | `localhost` | MQTT broker hostname |
| `MQTT_PORT` | `1883` | MQTT broker port |
| `KUKSA_HOST` | `localhost` | Kuksa Databroker gRPC hostname |
| `KUKSA_PORT` | `55555` | Kuksa Databroker gRPC port |

---

## CARLA-Kuksa Provider

| Property | Value |
| --- | --- |
| **Service name** | `carla_provider` |
| **Build context** | `compute/Carla_Provider/` |
| **Source** | `compute/Carla_Provider/carla_provider.py` |
| **Network** | Host |

### Purpose

The CARLA-Kuksa Provider connects the CARLA driving simulator to the MQTT message bus. It reads vehicle dynamics data from a CARLA actor over the CARLA Python API and publishes each value to the corresponding MQTT topic in real time.

When started with the `--nocarla` flag (the default in the Dockerfile), it generates mock telemetry data instead of connecting to a real CARLA server. This allows the full CarMate stack to run and be evaluated without simulation hardware.

### Modes

| Mode | Start Command | Use Case |
| --- | --- | --- |
| **Mock** (default) | `python carla_provider.py --nocarla` | Evaluation without CARLA |
| **CARLA** | `python carla_provider.py` | Real simulation |

### Configuration

Configured directly in `carla_provider.py`:

| Parameter | Default | Description |
| --- | --- | --- |
| `CARLA_HOST` | `192.168.43.249` | IP address of the Windows machine running CARLA |
| `CARLA_PORT` | `2000` | CARLA server port |
| `MQTT_BROKER` | `localhost` | MQTT broker hostname |
| `MQTT_PORT` | `1883` | MQTT broker port |

---

## Eclipse Kuksa Databroker

| Property | Value |
| --- | --- |
| **Hosted by** | `sdv_runtime` container |
| **Protocol** | gRPC |
| **Default port** | `55555` |
| **Source** | [eclipse-kuksa/kuksa-databroker](https://github.com/eclipse-kuksa/kuksa-databroker) |

### Purpose

The Eclipse Kuksa Databroker is the central vehicle signal store. It provides a gRPC API that other services use to read, write, and subscribe to VSS-aligned vehicle signals. All sensor data ingested through the MQTT-Kuksa Provider is stored here as typed VSS values.

The Databroker acts as the single source of truth for vehicle state in the CarMate blueprint.

---

## Vehicle Data Accessor

| Property | Value |
| --- | --- |
| **Service name** | `vehicle_data_accessor` |
| **Build context** | `compute/Vehicle_data_accessor/` |
| **Source** | `compute/Vehicle_data_accessor/vehicle_data_accessor.py` |
| **Network** | Host |

### Purpose

The Vehicle Data Accessor is a bridge service between the Kuksa Databroker and the CarMate Agent. It reads the current values of all relevant VSS signals from the Databroker via gRPC and exposes them in a structured format that the CarMate Agent can consume when building LLM prompts.

### Signals Read

All signals listed in the [Signal Mapping](#signal-mapping) table in the MQTT-Kuksa Provider section above.

---

## CarMate Agent

| Property | Value |
| --- | --- |
| **Service name** | `carmate_agents` |
| **Build context** | `compute/carmate_agents/` |
| **Source** | `compute/carmate_agents/carmate_agents.py` |
| **Network** | Host |

### Purpose

The CarMate Agent is the AI reasoning core of the blueprint. It acts as a uProtocol RPC server, receiving text queries from CarMate I/O, enriching them with real-time vehicle context from the Vehicle Data Accessor, calling the configured LLM, and returning the AI-generated response.

### Supported LLM Backends

| Provider | `AI_PROVIDER` value | Model (default) |
| --- | --- | --- |
| Ollama (local) | `ollama` | `phi3` |
| OpenAI | `openai` | `gpt-4o-mini` |
| Google Gemini | `gemini` | `gemini-1.5-flash` |
| xAI Grok | `grok` | `grok-beta` |
| Groq | `groq` | `llama-3.3-70b-versatile` |

### VSS Resource Map

The agent maps numeric uProtocol resource IDs to VSS signal names:

| Resource ID | Signal |
| --- | --- |
| `1001` | Speed |
| `1002` | Latitude |
| `1003` | Longitude |
| `1004` | Altitude |

### Configuration

All AI configuration is in `compute/carmate_agents/carmate_agents.py` under the `HARDCODED AI CONFIGURATION` section. See the [Configuration Reference](./getting-started#configuration-reference) in the Getting Started guide for full details.

---

## CarMate I/O

| Property | Value |
| --- | --- |
| **Service name** | `carmate_io` |
| **Build context** | `compute/car_mate_io/` |
| **Source** | `compute/car_mate_io/car_mate_io.py` |
| **Network** | Host |
| **Web UI port** | `5000` |

### Purpose

CarMate I/O is the driver-facing component. It is a Flask web application that provides:

- **Speech-to-Text (STT)** — Records the driver's spoken input and converts it to text using the `speech_recognition` library
- **Text-to-Speech (TTS)** — Converts the AI-generated text response back to speech for the driver
- **Web UI** — The driver interaction panel accessible at `http://localhost:5000`
- **Cluster display** — Serves the instrument cluster visualization (static assets in `compute/car_mate_io/static/`)
- **uProtocol RPC client** — Sends the transcribed driver query to the CarMate Agent and receives the response

### Endpoints

| Endpoint | Method | Description |
| --- | --- | --- |
| `/` | GET | Main driver Web UI |
| `/listen` | POST | Triggers STT recording and returns transcribed text |
| `/speak` | POST | Accepts text and returns synthesized audio |
| `/query` | POST | Sends a text query to the CarMate Agent via uProtocol and returns the response |
