# # Vehicle Data Accessor Component

The **Vehicle Data Accessor** component bridges MQTT telemetry sources with the uProtocol/Zenoh pub/sub network. It consumes raw vehicle metrics (speed, position, wetness, MCU temperature) over MQTT and republishes them as structured uProtocol messages across the Zenoh bus.

---

## Features

* **MQTT Consumer:** Subscribes to telemetry topics published by vehicle providers or hardware components over MQTT.
* **uProtocol Telemetry Bridge:** Packs incoming MQTT text streams into Protobuf `StringValue` payloads wrapped inside uProtocol `UMessage` structures.
* **Resource ID Mapping:** Maps distinct MQTT topics directly to standardized uProtocol resource IDs (e.g., `0x1001` for speed, `0x2001` for temperature).
* **Async Queue Integration:** Uses an `asyncio.Queue` combined with a daemonized MQTT loop thread to safely cross execution contexts without blocking network loops.

---

## Directory Layout

```text
Vehicle_data_accessor/
├── main.py          # Main entrypoint, MQTT callbacks, and Zenoh publisher loop
├── Dockerfile        # Container specification
└── requirements.txt  # Python dependencies
```

---

## Container Dependencies

This component acts as a translator and requires both an MQTT Broker and a Zenoh Router to function:

* **MQTT Broker Endpoint:** `localhost:1883` (Source of raw vehicle telemetry)
* **Zenoh Router Endpoint:** `tcp/zenoh:7447` (Target bus for uProtocol publishing)

---

## Building & Running with Docker

### 1. Build the Docker Image
From the `Vehicle_data_accessor` directory:

```bash
docker build -t vehicle-data-accessor .
```

### 2. Run with Docker Compose (Recommended)
Run the accessor alongside its required dependencies using Docker Compose from the `compute/` root directory:

* **Run `Vehicle_data_accessor` stack:**
  ```bash
  docker compose up Vehicle_data_accessor
  ```

* **Run stack in detached mode:**
  ```bash
  docker compose up -d
  ```

---

## Topic to uProtocol Resource Mappings

| Subscribed MQTT Topic | uProtocol Resource ID | Data Description |
| :--- | :--- | :--- |
| `carla/vehicle/speed` | `0x1001` | Vehicle speed |
| `carla/vehicle/lat` | `0x1002` | GNSS Latitude |
| `carla/vehicle/lon` | `0x1003` | GNSS Longitude |
| `carla/vehicle/alt` | `0x1004` | GNSS Altitude |
| `carla/vehicle/wetness` | `0x1005` | Road wetness |
| `mcu/temperature` | `0x2001` | MCU / Ambient temperature |