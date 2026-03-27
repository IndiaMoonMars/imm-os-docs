# Sensor Integration Guide — IMM-OS

> **Status:** Development mode — using simulator. Hardware not yet available.

This guide explains exactly what to do when physical sensors arrive.  
The system is designed so sensor integration requires **zero backend changes**.

---

## How It Works

```
[Simulator / Real Sensor]
         ↓  MQTT publish
    [Mosquitto broker]
         ↓  subscribe
  [telemetry_worker.py]
         ↓  write
      [InfluxDB]
         ↓  query via API
   [OpenMCT Dashboard]
```

Both the simulator and real sensors publish identical JSON to identical MQTT topics.  
Everything downstream is already built and working.

---

## When Hardware Arrives

### Step 1 — Flash the RPi nodes

```bash
# Ubuntu 22.04 on RPi 4
sudo rpi-imager  # or flash with Raspberry Pi Imager
```

### Step 2 — Install sensor drivers on each RPi

```bash
# Enable I2C
sudo raspi-config  # Interface Options → I2C → Enable

# Install Python sensor libraries
pip install adafruit-circuitpython-bme280    # temp/humidity/pressure
pip install adafruit-circuitpython-scd30     # CO2 sensor
pip install RPi.GPIO paho-mqtt
```

### Step 3 — Deploy the edge agent

Replace `sensor_sim.py` with a real sensor read loop that publishes the **same payload schema**:

```python
import paho.mqtt.client as mqtt
import board, adafruit_bme280, time, json

client = mqtt.Client()
client.connect("imm.local", 1883)

i2c = board.I2C()
bme280 = adafruit_bme280.Adafruit_BME280_I2C(i2c)

NODE_ID = "node-rpi-01"  # match config.py

while True:
    payload = {
        "node_id":    NODE_ID,
        "node_type":  "rpi",
        "measurement": "temperature",
        "value":      round(bme280.temperature, 3),
        "unit":       "celsius",
        "timestamp":  int(time.time()),
        "simulated":  False,   # ← key change
    }
    client.publish(f"imm/habitat/{NODE_ID}/telemetry/temperature",
                   json.dumps(payload), qos=1)
    time.sleep(5)
```

### Step 4 — Stop the simulator

```bash
# In imm-os-infra/
docker compose stop sensor-sim
```

### Step 5 — Verify data in InfluxDB

```
http://imm.local:8086
Org: imm_org  |  Bucket: telemetry
Filter: simulated = "False"
```

---

## Sensor Procurement List (Phase 1)

| Sensor | Part | Measures | Interface | Node |
|---|---|---|---|---|
| BME280 | Adafruit #2652 | Temp / Humidity / Pressure | I2C | RPi-01, RPi-02 |
| SCD30 | Adafruit #4867 | CO2 (NDIR) | I2C | RPi-01, RPi-02 |
| Grove O2 | Seeed #101020002 | O2 % | Analog | RPi-01, RPi-02 |
| INA3221 | Adafruit #4226 | Power / Current | I2C | Jetson |
| Solar charger | Waveshare Solar Power | Solar input | GPIO / I2C | Jetson |

---

## Checklist: Sensor Integration Complete When

- [ ] RPi-01 publishing `temperature`, `humidity`, `pressure`, `co2`, `o2` with `simulated=false`
- [ ] RPi-02 publishing same measurements with `simulated=false`
- [ ] Jetson publishing `cpu_temp`, `gpu_temp`, `power_draw`, `battery_level`, `solar_input`
- [ ] InfluxDB shows real data in `telemetry` bucket
- [ ] OpenMCT dashboard shows live readings
- [ ] Simulator container stopped/removed
