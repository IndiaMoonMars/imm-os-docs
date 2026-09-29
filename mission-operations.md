# Mission record: sols, IST and the 7-sol archive

The **Mission** page (MSN-06) records a mission of N sols (default 7).
- A **sol is exactly 24 h** from the moment the mission starts (T0).
- **Every time is shown in IST.** The top bar of every page shows IST with the date, and next to it the mission clock (`SOL 3 · 14:22:05`).

## How the data is kept

| Layer | What | Where | Survives |
|---|---|---|---|
| Live database | Every reading at full rate, with its quality | InfluxDB on the MCC (`habitat_sensors`, kept indefinitely) | normal operation |
| Sol archive | 10 min after each sol ends: one CSV per sensor (every reading, IST and UTC, quality), `summary.json`, `rollup.json`, `README.txt`, `SHA256SUMS` | `MISSION_ARCHIVE_DIR/<mission>-<id>/sol-NN/`: **point it at an external drive** | a lost MCC disk |
| Backups | Postgres and InfluxDB every 6 h | `BACKUP_DIR` | see [backup-and-dr.md](backup-and-dr.md) |
| Kafka | Every reading, replayable | 10 days (`KAFKA_LOG_RETENTION_HOURS=240`) | a processing fault |
| Each Pi | Its own copy of every reading (blackbox) | `/var/lib/imm-os/blackbox`, 9 days (`IMM_BLACKBOX_RETENTION_H=216`, about 150 MB a day) | MCC down for days |

**Sols are not stored with the readings.** They are computed from each reading's time and T0, so correcting T0 on the Mission page rewrites nothing. (The telemetry processor's own `sol` tag is the Mars Sol Date and has nothing to do with mission sols.)

**The mission itself is stored in Postgres:**
- the mission (name, T0, sols, crew);
- the mission log (notes, archive entries, start, edits and end);
- which sols are archived.

Alarms come from the health monitor's alarm history.

## Numbers on the page

- **Per sol:**
  - mean, min and max of CO₂, O₂, temperature, humidity, pressure, radiation, methane and CO, with the IST time of each min and max;
  - a 10-minute trend.
  - **Warm-up is left out:** minutes in which a sensor reported `warming` (MQ-4, Geiger) are stored, flagged `suspect`, and don't count.
- **7-sol timeline:**
  - one measurement over the whole mission (10-minute mean, with a min–max band);
  - sol boundaries labelled in IST, the alarm limits, and a NOW marker.
- **Sol vs sol:** each sol's profile against hours into the sol, to spot drift from day to day.
- **Data health:**
  - the percentage of expected readings for each sensor in each sol (in slots of at least 1 min and at least 2 reporting periods). Anything under 95 % is outlined.
  - A sensor belongs to the mission once it reaches 10 % in some sol, so a test node that sends one reading doesn't count.
  - From its first sol, a silent sol counts 0 %: a dead sensor shows up as a gap and doesn't just vanish.
- **Mission dose:** µSv from the external Geiger's dose rate, per sol and cumulative. Minutes without data add nothing, so the total is a lower bound.

Statistics use real sensors only. Add `?sim=1` to an API call to include simulated streams.

## Downloads (Mission page buttons)

- **Sol N data (ZIP):**
  - the sol's per-sensor CSVs, summary and checksums;
  - served from the archive for a finished sol, and built on demand for the running one.
- **Sol N report (PDF):**
  - a printable page with the sol's statistics, coverage, dose, and the mission log and alarms;
  - use the browser's *Print → Save as PDF*.
- **Whole mission:** one CSV with 1-minute mean, min, max and count for every metric of every sensor, for every sol.

Download links carry a 2-minute token, because a browser link can't send the login header.

## Running a mission

| When | What |
|---|---|
| T−48 h | Power everything on and leave it on: MQ-4 / MQ-7 burn-in, GNSS almanac, stable temperatures. |
| T−2 h | Calibrate in fresh outdoor air (see [Calibrations](#calibrations-esp32-sensor-board)). Rotate the BNO055 until 3/3. |
| T−1 h | On the Pi, run `tools/bringup.py all`. On the MCC, `scripts/mission-readiness.ps1 --quick` must say **GO**. |
| T−10 min | Every sensor READY in *Sensor readiness* and the Health page GO. `MISSION_ARCHIVE_DIR` is on the external drive. |
| T0 | **Start mission** (commander or MCC operator): name, Sol 1 start in IST (default now), number of sols, crew. |
| During | Add notes to the mission log from the page. Check the data-health grid each sol. Each sol archives itself 10 min after it ends. |
| After | The mission ends by itself after the last sol, or use **Edit → End mission now**. Copy the archive folder somewhere safe. |

Edit (commander or MCC operator) can rename the mission, move T0 or change the number of sols. Every change goes into the mission log.

## Calibrations (ESP32 sensor board)

From the Pi, run `.venv/bin/python sensor_drivers/esp32_bridge.py --send "<command>"`:

| Command | Does |
|---|---|
| `CAL_O2` | After 5 min in fresh outdoor air: the O₂ cell reads 20.9 %. Stored in the sensor. |
| `CAL_CO2 [ppm]` | After 3 min in fresh outdoor air (default 420 ppm): SCD40 forced recalibration. **It also turns automatic self-calibration off.** That calibration needs about 7 days of regular fresh air, as long as the mission, and would drift instead. Stored in the sensor. |
| `ASC_ON` | Turns SCD40 self-calibration back on, after the mission. |
| `CAL_MQ4` | After 24–48 h burn-in, warm, in clean air: stores the MQ-4's R0 in flash. |
| `CAL_BNO_CLEAR` | Forgets the stored BNO055 calibration. |

## Warm-up, only where physics needs it

| Sensor | Warm-up | How it's handled |
|---|---|---|
| MQ-4 | 3 min heater warm-up | **Only after a real power-on or brownout.** After a watchdog, crash, software or EN-button reset the heater kept its 5 V, so there's no warm-up. The board reports `warm_left_s`, and the dashboard shows a countdown. |
| Geiger | 60 s counting window | A provisional CPM is shown while the window fills (IMM-OS firmware), flagged `warming`. |
| BNO055 | Calibration used to be lost at every restart | Once 3/3/3/3, its offsets are stored in flash and written back at every start (`cal_restored`). |
| SCD40 | Self-calibration drifts over 7 days | `CAL_CO2` before the mission turns it off (`asc: 0`). |
| O₂ | Needs calibrating once | `CAL_O2`; stored in the sensor. |
| GNSS | Minutes-long cold start | Keep the board powered. The page shows the satellites seen until the fix. |

## API

`/api/mission`:
- `GET ""` (the mission and its clock);
- `POST /start`, `PATCH ""`, `POST /end`;
- `GET /overview`, `/sol/{n}`, `/timeline?measurement=`, `/overlay?measurement=`, `/health`, `/dose`;
- `POST /events`;
- `POST /download-token`;
- `GET /download/sol/{n}`, `/download/mission`, `/report/{n}`.

Code: `imm-os-backend/services/mission.py` (pure logic) and `mission_api.py` (API, InfluxDB, archive).
