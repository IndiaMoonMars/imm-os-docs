# IMM-OS MQTT Topic Schema

## Overview

All IMM-OS telemetry flows through Mosquitto MQTT broker using a structured topic hierarchy.  
This schema is the **integration contract** between edge nodes and the backend.

> Real sensors and the simulator both use identical topics and payloads.

---

## Topic Hierarchy

```
imm/
├── habitat/
│   ├── {node_id}/
│   │   ├── telemetry/
│   │   │   ├── temperature
│   │   │   ├── humidity
│   │   │   ├── pressure
│   │   │   ├── co2
│   │   │   ├── o2
│   │   │   ├── cpu_temp
│   │   │   ├── gpu_temp
│   │   │   ├── power_draw
│   │   │   ├── battery_level
│   │   │   └── solar_input
│   │   └── status           ← heartbeat (retained)
└── mcc/
    ├── command/{subsystem}  ← MCC → Edge commands
    └── ack/{command_id}     ← Edge → MCC acknowledgements
```

---

## Node IDs

| Node ID | Hardware | Location |
|---|---|---|
| `node-rpi-01` | Raspberry Pi 4 (4GB) | Habitat Zone A |
| `node-rpi-02` | Raspberry Pi 4 (4GB) | Habitat Zone B |
| `node-jetson` | NVIDIA Jetson Orin Nano | Edge AI / Compute |

---

## Telemetry Payload Schema

**Topic:** `imm/habitat/{node_id}/telemetry/{measurement}`  
**QoS:** 1  
**Retain:** false

```json
{
  "node_id":    "node-rpi-01",
  "node_type":  "rpi",
  "measurement":"temperature",
  "value":      22.4,
  "unit":       "celsius",
  "timestamp":  1711234567,
  "simulated":  true
}
```

| Field | Type | Description |
|---|---|---|
| `node_id` | string | Unique node identifier |
| `node_type` | string | `rpi` or `jetson` |
| `measurement` | string | Sensor type (see table below) |
| `value` | float | Sensor reading |
| `unit` | string | SI unit string |
| `timestamp` | int | Unix epoch (seconds) |
| `simulated` | bool | `true` = simulator, `false` = real sensor |

---

## Measurements & Units

### RPi Environmental Nodes

| Measurement | Unit | Range | Sensor (when available) |
|---|---|---|---|
| `temperature` | celsius | 15–35 | BME280 / DHT22 |
| `humidity` | percent | 20–80 | BME280 / DHT22 |
| `pressure` | hPa | 950–1050 | BME280 |
| `co2` | ppm | 380–5000 | SCD30 / MH-Z19B |
| `o2` | percent | 15–22 | Grove O2 sensor |

### Jetson Compute Node

| Measurement | Unit | Range | Source |
|---|---|---|---|
| `cpu_temp` | celsius | 30–85 | Jetson thermal zone |
| `gpu_temp` | celsius | 30–90 | Jetson thermal zone |
| `power_draw` | watts | 5–25 | INA3221 power monitor |
| `battery_level` | percent | 0–100 | Battery management IC |
| `solar_input` | watts | 0–20 | Solar charge controller |

---

## Status / Heartbeat Payload

**Topic:** `imm/habitat/{node_id}/status`  
**QoS:** 0  **Retain:** true

```json
{
  "node_id":   "node-rpi-01",
  "status":    "online",
  "timestamp": 1711234567
}
```

---

## MCC Command Payload

**Topic:** `imm/mcc/command/{subsystem}`  
**QoS:** 2 (exactly once)

```json
{
  "command_id": "cmd-uuid-0001",
  "subsystem":  "life_support",
  "action":     "set_co2_alert_threshold",
  "params":     { "threshold_ppm": 1000 },
  "issued_by":  "operator@imm.local",
  "timestamp":  1711234567
}
```

---

## InfluxDB Mapping

The `telemetry_worker.py` service writes each message as:

```
measurement = {measurement}   e.g. "temperature"
tags:
  node_id   = "node-rpi-01"
  node_type = "rpi"
  unit      = "celsius"
  simulated = "True"/"False"
fields:
  value     = 22.4
```
