# Telemetry quality and health flags

Every validated reading carries its quality, so nobody has to guess whether a number can be
trusted.

## Fields on every reading

| Field | Set by | Meaning |
|---|---|---|
| `q` | validator (edge may start it) | `good`, `suspect` or `bad`: the worst of its flags. An edge-declared quality is never improved. |
| `qf` | validator, edge | reason codes (below) |
| `delayed` | edge (replay), validator | arrived more than 60 s after its timestamp: history, not live |
| `seq`, `run` | edge publisher | sequence number per topic and an ID per driver run: gaps are lost readings |
| `simulated` | simulator / edge | simulator data: never changes the mission mode |

InfluxDB stores `q` and `delayed` as fields, not tags. A replayed reading therefore overwrites
the original instead of creating a second series.

## Reason codes

| Code | Quality | Where | Meaning |
|---|---|---|---|
| `uncalibrated` | suspect | edge | calibration never done (MQ-4 R0, O₂ `CAL_O2`, BNO055) |
| `warming` | suspect | edge | heated sensor not at temperature (MQ-4 first 3 min) |
| `sensor_reset` | suspect | edge | the chip reset since its previous reading |
| `saturated` | suspect | edge | at the end of its measuring range |
| `source_fault` | bad | edge | the driver or board says the value is wrong |
| `soft_range` | suspect | validator | outside the sensor's normal or measuring range |
| `clock` | suspect | validator | timestamp in the future: the node's clock is off |
| `delayed` | good | validator | late; says nothing about the value |
| `stuck` | suspect | health monitor | same value for ≥ 20 readings and ≥ 10 min |
| `rate` | suspect | health monitor | changed faster than physics allows (held 60 s) |
| `cross_check` | suspect | health monitor | disagrees with another sensor on the same air: dew point BME280 vs SCD40 differs by > 3 °C, or O₂ is > 1 % from what the CO₂ level implies |
| `seq_gap` | good | health monitor | readings were lost on the way (integrity, not value) |

## Limits

- **Hard limits** (physically impossible, for example BME280 temperature > 150 °C) reject the
  reading. It is sent to `telemetry.deadletter` with the reason and counted against the sensor.
  After 5 rejections in 5 min the stream is `bad`.
- **Soft ranges** only flag. For gas sensors they are the sensor's measuring range: SCD40
  0–40 000 ppm, O₂ 0–25 %, MQ-7 0–2000 ppm, MQ-4 0–10 000 ppm.

## Quality and alarms (fail-safe)

- **Good data** raises alarms normally.
- **Suspect data** still raises the alarm, marked **UNVERIFIED** and capped at WARNING.
- **Bad data** raises no limit alarm. The sensor-fault alarm covers it instead.
- **Delayed data** raises no live alarm (it is history).
- **Exception:** data suspect *only* because it is out of range (`soft_range`, `saturated`) counts
  fully. A CO₂ sensor pinned high means "at least this much". This was found by the mission
  readiness test: before the fix, the CO₂ emergency at 20 000 ppm could never fire.

## Where you see it

- **Sensors tab:** a SUSPECT or BAD DATA badge on the card, with the reasons.
- **Health tab:** each stream's status, reasons, and lost / filled-in counts.
- **`/api/telemetry/latest`:** each reading's `q`. A bad reading loses to a usable one from another sensor.
- **Alarm text:** UNVERIFIED and the reason.
