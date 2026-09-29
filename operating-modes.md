# Operating modes and degraded operation

## Mission mode

Computed by the health monitor every second from the subsystems and the open alarms.
It is shown in the IMM-OS top bar, on the Health tab and in OpenMCT's status bar.

| Mode | When | What it asks of people |
|---|---|---|
| **NOMINAL** | every subsystem GO, no active warning or emergency | normal operations |
| **DEGRADED** | a subsystem DEGRADED or NO-GO, or a warning is active | read the alarms; work around the degraded function; plan the fix |
| **EMERGENCY** | an emergency alarm is active (real data) | follow the emergency procedure the alarm names, now |

Simulated data and simulator alarms never change the mode.

## Subsystems: GO / DEGRADED / NO-GO

| Subsystem | NO-GO when | DEGRADED when |
|---|---|---|
| Atmosphere, per node and zone | CO₂ or O₂ monitoring lost; atmosphere emergency | a measurement on a backup or suspect source; caution or warning active |
| Node, per edge node | node not reporting | weak power, heat, failed services, a component in SAFE, FAULT, DEGRADED or ISOLATED |
| EVA | a crew member in LOS or CONTINGENCY, or a vitals emergency | LOS warning, partial loss (vitals or position) |
| MCC data pipeline | a critical service down, or processing stalled | a non-critical service down |

## Alarm lifecycle

Based on ISA-18.2.

```
 condition true for on-delay ──► ACTIVE (unacknowledged) ──ack──► ACTIVE (acknowledged)
        │                           │                                │
        │                  condition false for off-delay     condition false for off-delay
        │                           ▼                                ▼
        │                    RTN (unacknowledged) ──ack──►        CLOSED
        │                           │
        └──────── condition true again ◄─┘   (the same alarm becomes ACTIVE again)
```

- There is **one open alarm per condition**. A condition that stays true is one alarm, not one per reading.
- **Escalation** (caution → warning → emergency) removes the acknowledgement, because the new
  severity needs its own. De-escalation keeps it.
- **Severities:**
  - advisory: information;
  - caution: attention soon;
  - warning: act now;
  - emergency: life or mission at risk.
- Alarm history (every raise, escalation, acknowledgement, clear) is kept in Postgres and shown on the Health tab.

## Degraded operation by fault

| Fault | What keeps working | What the crew or MCC does |
|---|---|---|
| BME280 lost | temperature and humidity from the SCD40 | replace or reseat the BME280 when convenient |
| SCD40 lost | temperature and humidity from the BME280; **no CO₂** (NO-GO) | portable CO₂ monitor; ventilate by procedure; fix the sensor |
| O₂ cell lost or uncalibrated | readings marked suspect or lost | portable O₂ monitor; `CAL_O2` in fresh air |
| No humidity for the climate controller | temperature control; dehumidifier held off (DEGRADED) | check the humidity sensor |
| No temperature for the climate controller | relays off (SAFE) until readings return | manual climate control by procedure |
| Relay fault | outputs commanded off (FAULT) | inspect the relay board |
| Link to the MCC lost | everything on the node keeps running and logging; readings queue and arrive later in order | nothing for data; check the network |
| MCC outage | the nodes keep running (local control, local queues) | restore the MCC; the data catches up |
| InfluxDB down | alarms and live views keep working; history waits in Kafka | restart InfluxDB (autoheal tries first) |
| Postgres down | alarms keep working from memory; records written when it returns | restart Postgres |
| Health monitor down | data keeps flowing and being stored; no alarms | autoheal restarts it; alarms are restored |

## Component states (edge)

Components report `habitat/health/<node>/<component>` (see imm-os-edge core/health.py):

| State | Meaning |
|---|---|
| NOMINAL | working normally |
| DEGRADED | working with less |
| SAFE | outputs held safe until inputs return |
| ISOLATED | a faulty part switched out |
| FAULT | not working |
| STARTING / STOPPED | lifecycle |
