# # Carla Provider Component

The **Carla Provider** component acts as a bridge between the CARLA simulator (or simulated mock telemetry data) and an MQTT broker. It streams real-time vehicle metrics such as speed, GPS coordinates (latitude, longitude, altitude), and road wetness.

---

## Features

* **CARLA Integration:** Connects to a live CARLA simulator instance, spawns an autonomous vehicle actor with GNSS sensors, and streams real telemetry.
* **Mock Mode (`--nocarla`):** Default fallback mode. Runs standalone without a connected CARLA server by generating realistic mock vehicle data.
* **MQTT Publishing:** Transmits telemetry variables continuously over discrete MQTT topics.
* **Externalized Config:** Reads server endpoints and topic definitions dynamically from `config.json` (with environment variable overrides supported).

---

## Directory Layout

```text
Carla_Provider/
├── carla_provider.py  # Main entrypoint script and telemetry loop
├── config.json         # External configuration parameters
├── Dockerfile          # Container specification
└── requirements.txt    # Python dependencies
```

---

## Container Dependencies

This component requires an **MQTT Broker** (e.g., Mosquitto) to publish its vehicle telemetry stream.

* **MQTT Broker Endpoint:** `localhost:1884` (or overridden via `MQTT_BROKER` environment variable).
* **CARLA Server (CARLA Mode):** Requires a running CARLA Simulator instance at the host/port configured in `config.json` (or `CARLA_HOST` / `CARLA_PORT` environment variables).

---

## Building & Running with Docker

### 1. Build the Docker Image
From the `Carla_Provider` directory:

```bash
docker build -t carla-provider .
```

### 2. How to Switch Between MOCK Mode and CARLA Mode

By default, the container is set to run in **MOCK mode** using the default command in `Dockerfile`:
```dockerfile
CMD ["python", "carla_provider.py", "--nocarla"]
```

To switch between modes when running Docker:

#### 🟡 Option A: Run in Default MOCK Mode (No CARLA Required)
Simply run the container normally. It executes using the default `CMD` in `Dockerfile`:

```bash
docker run --rm --network host carla-provider
```

#### 🟢 Option B: Run in Live CARLA Mode
Override the default command by passing `python carla_provider.py` at the end of `docker run` (which omits the `--nocarla` flag):

```bash
docker run --rm --network host carla-provider python carla_provider.py
```

> **Note:** Ensure your CARLA simulator server is running before executing CARLA mode. You can pass custom host/port settings via environment variables:
> ```bash
> docker run --rm --network host -e CARLA_HOST=127.0.0.1 -e CARLA_PORT=2000 carla-provider python carla_provider.py
> ```

#### Option C: Permanently Change Default in `Dockerfile`
If you want the container to run in **CARLA mode by default**, edit `Dockerfile` and remove `"--nocarla"`:
```dockerfile
# Before (MOCK Mode default):
CMD ["python", "carla_provider.py", "--nocarla"]

# After (CARLA Mode default):
CMD ["python", "carla_provider.py"]
```
*Then rebuild the image:* `docker build -t carla-provider .`

---

## 📡 Published MQTT Topics

| Topic | Data Type | Description |
| :--- | :--- | :--- |
| `carla/vehicle/speed` | Float | Vehicle speed in km/h |
| `carla/vehicle/lat` | Float | Vehicle GNSS latitude |
| `carla/vehicle/lon` | Float | Vehicle GNSS longitude |
| `carla/vehicle/alt` | Float | Vehicle GNSS altitude (meters) |
| `carla/vehicle/wetness` | Integer | Animated road wetness percentage (0–20%) |

---

## ⚙️ Configuration

Connection parameters are defined in `config.json` and can be overridden with environment variables (`CARLA_HOST`, `CARLA_PORT`, `MQTT_BROKER`, `MQTT_PORT`):

```json
{
  "carla": {
    "host": "127.0.0.1",
    "port": 2000,
    "tm_port": 8000
  },
  "mqtt": {
    "broker": "localhost",
    "port": 1884,
    "topics": {
      "speed": "carla/vehicle/speed",
      "lat": "carla/vehicle/lat",
      "lon": "carla/vehicle/lon",
      "alt": "carla/vehicle/alt",
      "wetness": "carla/vehicle/wetness"
    }
  }
}
```