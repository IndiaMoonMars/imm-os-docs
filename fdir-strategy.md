# FDIR strategy: fault detection, isolation and recovery

How IMM-OS notices that something is wrong, keeps the fault from spreading, and gets back to
normal. The rule throughout:
- **every layer protects itself;**
- **the layer above notices when it does not;**
- **a person is told whenever an automatic action was taken or did not work.**

Related: [operating-modes.md](operating-modes.md), [telemetry-quality.md](telemetry-quality.md),
[eva-contingency.md](eva-contingency.md), [backup-and-dr.md](backup-and-dr.md),
[vv-plan.md](vv-plan.md).

## Layers

```
 sensor board (ESP32 / STM32)   task watchdog / IWDG, reset reason, I²C bus recovery
        │ USB / UART
 edge node (Raspberry Pi 5)     systemd watchdog per service, hardware watchdog, degraded modes,
        │                       local broker: store-and-forward, blackbox (48 h)
        │ MQTT over TLS
 MCC broker ─► MQTT→Kafka bridge ─► validator ─► processor ─► InfluxDB
        │                                  └────► health monitor (FDIR engine) ─► alarms, modes
 MCC services                   Docker healthchecks + heartbeats, autoheal, bounded shutdown,
                                at-least-once processing, backups
 operators                      annunciator, Health tab, OpenMCT indicator, procedures
```

## Detection

| What fails | How it is detected | Where | Time to detect |
|---|---|---|---|
| Sensor reads impossible values | hard physical limits in the schema: dead-lettered with the reason | validator | immediate |
| Sensor reads implausible values | soft range, rate of change, stuck value, cross-check between sensors | validator, health monitor | 1 reading; stuck ≥ 10 min; cross-check 60 s on-delay |
| Sensor stops | stream stale (3 × period, ≥ 15 s), offline (10 × period, ≥ 120 s) | health monitor | 15 s / 120 s |
| Readings lost on the way | sequence numbers per stream (`seq`, `run`) | health monitor | next reading |
| Sensor chip resets (supply dip) | BME280 settings lost; ESP32 reset reason and boot count | ESP32 firmware → board stream | next reading / next boot |
| Board hangs | ESP32 task watchdog (10 s), STM32 IWDG (8 s) | firmware | ≤ 10 s |
| Edge service hangs | systemd watchdog (60 s daemons, 120 s pipelines) | Pi | ≤ 120 s |
| Edge node hangs | Pi hardware watchdog (15 s); node offline at the MCC | Pi / health monitor | 15 s / 120 s |
| Edge component in trouble | `habitat/health/<node>/<component>` status (NOMINAL … FAULT) | health monitor | on change; silent after 3 × interval |
| Link to the MCC down | local broker bridge state (`mcc_link`), queued backlog (`mqtt_backlog`) | Pi → sysmon | ≤ 10 s |
| MCC service down / hung | Docker healthcheck: HTTP `/health` or worker heartbeat file (60 s) | Docker | ≤ ~2 min |
| Processing stalled | Kafka consumer lag of validator and processor | health monitor | 10 s checks |
| No telemetry at all | nothing arrived for 60 s | health monitor | 60 s |
| Database down | Postgres / InfluxDB checks; writes queued | health monitor, processor | ≤ 10 s |
| EVA crew out of contact | no suit vitals or position | health monitor (EVA) | 10 s / 30 s / 120 s |
| Backups not happening | backup containers unhealthy when the last good one is > 2 intervals old | Docker, autoheal | ≤ 5 min after due |

## Isolation: keep a fault from spreading

- **Bad data never becomes a false alarm or a false all-clear.**
  - A reading outside physical limits is rejected, not stored.
  - A suspect reading is stored, but marked, and a limit alarm on it is marked UNVERIFIED and capped at WARNING.
  - Bad data is not used at all.
  - Fail-safe exception: data suspect only for being out of range still raises the full alarm (see telemetry-quality.md).
- **Redundant sources.**
  - Temperature and humidity fall back from the BME280 to the SCD40 (measurement DEGRADED, not lost).
  - CO₂ and O₂ have one source each, so losing either is loss of critical monitoring: a WARNING and NO-GO for that zone.
- **Components isolate themselves.** When its inputs are stale, the climate controller turns its
  relays off (SAFE). When humidity is missing it turns off only the dehumidifier (DEGRADED). A relay
  that can't be switched makes it command everything off (FAULT).
- **One node can't take the MCC down.** Every node's data goes through its own bridge queue.
  A node that floods is visible from its stream rates; the broker limits queues per session.
- **One MCC service can't take the others down.**
  - Every worker has its own healthcheck, restart policy and bounded shutdown.
  - Processing is at-least-once with Kafka as the buffer, so a stuck processor delays data but loses none.
- **Decommissioning.** A node that is removed or broken beyond repair is taken out with
  `POST /api/health/nodes/<node>/forget` (MCC operator or commander). This stops expecting its
  streams and closes its alarms, so it can't keep the habitat in DEGRADED.

## Recovery

| Fault | Automatic recovery | Reported as |
|---|---|---|
| Board hang | watchdog reset, sensors set up again | board reboot alarm (caution for watchdog, crash, brownout) |
| I²C bus stuck | 9 SCL clocks + STOP, sensors set up again | `# i2c: bus recovered` diagnostic, `i2c_err` count |
| Edge service hang / crash | systemd restarts it (5 s), forever; the stuck stack trace is logged | `svc_restarts` climbing → advisory / caution alarm |
| Edge node hang | hardware watchdog reboots the Pi; services start on boot | node offline, then back |
| MCC unreachable | the node's local broker queues on disk and delivers in order when the link is back | `mcc_link` 0, `mqtt_backlog`, late data marked delayed |
| Data still missing after that | blackbox replay of a time window (delayed, same seq/run) | gap counted as recovered |
| MCC worker crash | Docker restart policy | container restart count |
| MCC worker hang | heartbeat healthcheck → unhealthy → autoheal restart (3 per 15 min, then a person) | advisory alarm per restart; warning when it gives up |
| Kafka / InfluxDB / Postgres outage | producers buffer, processor retries without committing, health store queues writes | pipeline alarms while it lasts |
| Health monitor restart | open alarms, acknowledgements, armed EVA crew, component states and known streams restored from Postgres | none (by design) |
| Data loss / corruption | restore from the 6-hourly backups | backup-and-dr.md |

Automatic recovery never hides a fault. Every restart is counted and reported, and repeated
restarts become an alarm for a person.

## Who does what

- **The system:** detect, isolate, recover what is safe to recover automatically, and report.
- **MCC operator:** acknowledges alarms, follows the procedure the alarm names, decides on
  decommissioning, arms and disarms EVA monitoring, runs restores.
- **Crew:** act on local cautions (for example a portable monitor when CO₂ monitoring is lost). With a long
  communication delay the crew owns the response; the MCC supports.

## Verification

Every row above is exercised by the mission readiness test ([vv-plan.md](vv-plan.md)), which
injects the fault on the running system and checks the detection, the alarm, the degraded state,
the recovery and that no data was lost.
