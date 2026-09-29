# Verification & validation plan and mission readiness test

## Levels

| Level | What | Where | When |
|---|---|---|---|
| Unit | logic of every module: quality flags, stream health, alarms, EVA LOS, rules, store, autoheal, watchdog, degraded modes | each repo's `tests/` (pytest), firmware host simulations (g++) | every push (CI) |
| Integration | real Postgres (health store), two real Mosquitto brokers (store-and-forward, replay), real firmware code on simulated hardware | CI (Postgres and Mosquitto services) | every push |
| System (fault injection) | the whole running MCC stack with a real test node; faults injected live | `imm-os-infra/vv/mission_readiness.py` | before each mission, after every deployment or hardware change |
| Hardware checkout | each real sensor and board on the bench | `tools/bringup.py`, `tools/verify_esp32.py` | when a node is built or changed |

## Mission readiness test

```powershell
.\scripts\mission-readiness.ps1           # everything, ~30 min
.\scripts\mission-readiness.ps1 --quick   # skip the slow tests
```

- **The test node.** A test node (`vv-node-01`, crew `vv-crew-01`) publishes with the edge credentials, so its
  data is treated exactly like hardware. Its alarms are real: **run it in a planned test window.**
- **Fault injection** is done through the Docker API (crash, SIGSTOP hang, stop/restart of the bridge,
  broker, Kafka, InfluxDB, Postgres and health monitor) and by what the test node publishes
  (limit breaches, rate glitches, cross-check faults, gaps, backfill, component SAFE, loss of suit telemetry).
- **Cleanup.** Every injected fault is undone in a `finally` block, and the test node is decommissioned at the end.
- **Report and verdict.** The report goes to `vv/reports/` (Markdown and JSON). **GO** means every critical requirement passed.

## Requirements

| ID | Requirement | Pass criteria |
|---|---|---|
| VV-PRE-01 | Stack healthy before the test | no container unhealthy or starting |
| VV-PRE-02 | No false alarms on nominal data | streams ok, atmosphere GO, 0 alarms after 15 s |
| VV-ALM-01 | Limit alarm raised, acknowledged, cleared | CO₂ > 5000 ppm → WARNING ≤ 15 s; ack works; closes after return + off-delay |
| VV-ALM-02 | Escalation | CO₂ > 20 000 ppm → same alarm EMERGENCY, ack reset, mode EMERGENCY, atmosphere NO-GO |
| VV-ALM-03 | No alarm flooding | one database row per episode; complete event trail |
| VV-ALM-04 | Suspect data | alarm UNVERIFIED, ≤ WARNING, mode not EMERGENCY |
| VV-ALM-05 | Alarm persistence | open alarm keeps its id and acknowledgement across a health-monitor restart |
| VV-DET-01 | Impossible values rejected | dead-lettered, counted, not stored |
| VV-DET-02 | Rate-of-change | stream flagged `rate` |
| VV-DET-03 | Cross-check | dew-point disagreement > 3 °C for 60 s → caution |
| VV-DET-04 | Lost readings (minor) | sequence gap counted |
| VV-DET-05 | Backfill | late readings stored at their own time, no live alarm |
| VV-RED-01 | Failover | temperature served by the SCD40 within 30 s, DEGRADED (not NO-GO), back to the BME280 after |
| VV-RED-02 | Loss of critical monitoring | CO₂ source lost → WARNING and NO-GO within 40 s; clears after |
| VV-EDGE-01 | Edge component SAFE | WARNING, node DEGRADED; clears on NOMINAL |
| VV-EVA-01 | Partial loss | vitals missing with position → caution at the partial-loss time |
| VV-EVA-02 | LOS sequence | LOS_WARN / LOS / CONTINGENCY at the configured times ± 5 s; last position and search radius; mode EMERGENCY |
| VV-EVA-03 | EVA recovery | NOMINAL when contact returns; alarm clears; suit backfill credited to the outage |
| VV-REC-01 | Crashed worker | restarted within 60 s, telemetry flowing within 90 s |
| VV-REC-02 | Hung worker | heartbeat → unhealthy → autoheal restart within 4 min, reported |
| VV-REC-03 | Bridge outage | readings queued by the broker, delivered after |
| VV-REC-04 | Broker restart | no acknowledged reading lost |
| VV-REC-05 | Kafka outage | pipeline resumes by itself, no loss |
| VV-REC-06 | InfluxDB outage | `/api/telemetry/latest` 503 (never mock data); no loss after |
| VV-REC-07 | Postgres outage | alarms still raised; records written when it returns |
| VV-INT-01 | End-to-end integrity | every broker-acknowledged reading of the whole test in InfluxDB exactly once, at its time, with its value |
| VV-INT-02 | Replay dedup | a replayed backlog leaves one copy of each reading |
| VV-DR-01 | Backups | newest backups younger than 2 intervals, checksums match |
| VV-DR-02 | Postgres restore | every table's rows restored in a scratch server; RTO recorded |
| VV-DR-03 | InfluxDB restore | every point of the recorded window restored; RTO recorded |

## What the test found (and was fixed)

The first runs of the test on the full stack found real faults, each now fixed and covered by a unit test:

1. **The CO₂ emergency could never fire.** The SCD40's soft range ended at 10 000 ppm, so any higher
   reading was "suspect" and its alarm was capped at WARNING. Fix: soft ranges are the
   measuring ranges, and range-only flags never downgrade a hazard alarm.
2. **Data loss during an InfluxDB outage longer than about 3 min, or a processor crash.** Offsets were
   auto-committed before the write. Fix: at-least-once processing, where offsets are committed only after
   InfluxDB accepts the batch.
3. **Store-and-forward backlog stored at the wrong time.** Readings older than 30 s were rewritten
   to "now". Fix: readings keep their own time; only a clearly wrong clock is corrected.
4. **Slow crash recovery.** A worker's graceful shutdown could hang for about 2 min on a Kafka
   connection. Fix: bounded shutdown.
5. **State lost on a health-monitor restart.** Edge component states and a just-given acknowledgement
   were lost. Fix: component states persisted; the write queue is flushed on shutdown.
6. **Backup point count recorded as 0** (a CSV line-ending bug in the backup script). Found by the DR
   drill.
7. **Other fixes along the way:**
   - the blackbox replay targeted Kafka, which a node cannot reach;
   - the MQTT bridge would not start as deployed (import path);
   - autoheal misread "health: starting";
   - the climate controller kept a dehumidifier running on stale humidity.

## Latest result (sandbox MCC stack, 2026-09-29)

All 31 checks passed: 30 in a full run, and VV-INT-01 in a re-check after correcting how the test counts readings still in flight.
- End-to-end integrity passed through every outage: 7692 values in the full run, 1410 in the re-check (Kafka and InfluxDB outages).
- EVA LOS at 10.6 s, 30.3 s and 120.7 s.
- A crashed worker was back in 7 s; a hung worker was restarted by autoheal in 120 s.
- Restore drills took about 3 s each (small database).

Run it on the real MCC with the real nodes before the mission; reports go to `vv/reports/`.
