# # MQTT Kuksa Provider Component

The **MQTT Kuksa Provider** component serves as a bi-directional data bridge between an MQTT Broker and the KUKSA.val Databroker (COVESA Vehicle Signal Specification / VSS data model). It translates incoming MQTT vehicle telemetry into standard VSS signals and streams KUKSA state updates back out to MQTT.

---

## Features

* **MQTT to KUKSA Data Mapping:** Subscribes to raw vehicle telemetry topics (speed, GPS, wetness, MCU temperature) and updates their corresponding VSS paths in KUKSA.val.
* **KUKSA to MQTT Sync:** Subscribes to KUKSA signal updates (such as ambient light color) and publishes changed states back onto MQTT topics for hardware actuators or downstream listeners.
* **Thread-Safe Architecture:** Uses a thread-safe Queue alongside `asyncio.to_thread` to ensure non-blocking MQTT message handling during gRPC writes.

---

## Directory Layout

```text
mqtt_kuksa_provider/
├── main.py          # Main entrypoint, MQTT callbacks, and VSS mapping logic
├── Dockerfile        # Container specification
└── requirements.txt  # Python dependencies
```

---

## Container Dependencies

This component relies on both an MQTT Broker and a KUKSA Databroker instance to operate:

* **MQTT Broker Endpoint:** `localhost:1884`
* **KUKSA.val Databroker Endpoint:** `localhost:55555`

---

## Building & Running with Docker

### 1. Build the Docker Image
From the `mqtt_kuksa_provider` directory:

```bash
docker build -t mqtt-kuksa-provider .
```

### 2. Run with Docker Compose (Recommended)
Run the provider along with its required dependencies using Docker Compose from the `compute/` root directory:

* **Run `mqtt_kuksa_provider` and dependencies:**
  ```bash
  docker compose up mqtt_kuksa_provider
  ```

* **Run stack in detached mode:**
  ```bash
  docker compose up -d
  ```

---

## VSS Signal Mappings

### Inbound (MQTT → KUKSA VSS)

| Incoming MQTT Topic | Target KUKSA VSS Path |
| :--- | :--- |
| `carla/vehicle/speed` | `Vehicle.Speed` |
| `carla/vehicle/lat` | `Vehicle.CurrentLocation.Latitude` |
| `carla/vehicle/lon` | `Vehicle.CurrentLocation.Longitude` |
| `carla/vehicle/alt` | `Vehicle.CurrentLocation.Altitude` |
| `carla/vehicle/wetness` | `Vehicle.Exterior.Humidity` |
| `mcu/temperature` | `Vehicle.Cabin.HVAC.AmbientAirTemperature` |

### Outbound (KUKSA VSS → MQTT)

| Source KUKSA VSS Path | Outbound MQTT Topic |
| :--- | :--- |
| `Vehicle.Cabin.Light.AmbientLight.Row1.DriverSide.Color` | `compute/color` |