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
| Each Pi's SD card | Every reading as **readable CSV**, filed by sol: `/var/lib/imm-os/records/<mission>-<id>/sol-NN/<sensor>_<zone>.csv` (IST and UTC times) | kept (nothing deleted unless `IMM_RECORD_KEEP_DAYS` is set; pauses below 1 GB free) | a lost MCC PC |

**Sols are not stored with the readings.** They are computed from each reading's time and T0, so correcting T0 on the Mission page rewrites nothing. (The telemetry processor's own `sol` tag is the Mars Sol Date and has nothing to do with mission sols.)

**The mission itself is stored in Postgres:**
- the mission (name, T0, sols, crew);
- the mission log (notes, archive entries, start, edits and end);
- which sols are archived.

Alarms come from the health monitor's alarm history.

### The Pi's SD-card copy (`imm-sd-recorder` service)

Every node writes what it publishes to CSV on its own SD card:
- `<mission>-<id>/sol-01/`, `sol-02/`, … during the mission;
- `pre-mission/<IST date>/` before Sol 1;
- `no-mission/<IST date>/` otherwise.

The sol comes from the MCC's mission start. The Pi asks every 5 min and remembers the answer, so an MCC outage doesn't mix up the folders. This is about 60–100 MB a day for both sensor boards.

Copy it to the MCC PC (PowerShell):

```powershell
scp -r pratham@node-rpi-01.local:/var/lib/imm-os/records C:\Users\PRATHAM\Documents\pi-records
```

With no second drive on the MCC PC, this is the mission's second copy.

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

## Mission control: restart, test runs, abort, History

**Nothing here deletes readings.** A mission is a time window, and these controls only end or relabel it. Every action is written to the mission log. Only a commander or MCC operator can use them, and **Restart** and **Abort** ask you to type the mission name to confirm.

| Mission control ▾ | What happens |
|---|---|
| **Restart mission** | Ends the current mission now and starts a new one with the same name, sols and crew. Sol 1 starts now, or at a time you pick. The old mission is kept as *restarted* (or as a *test run*, your choice), with its sols, archive, downloads and SD-card folder. |
| **Mark as test run** | A dry run: kept, but left out of History's list and labelled TEST RUN. Undo it the same way. |
| **End mission now** | Finishes it early. The sols so far are archived; **New mission** appears. |
| **Abort (false start)** | Only until **1 hour into Sol 1**. The mission is labelled *aborted* and leaves the page; the previous mission (if any) is the current one again. |

**History** lists every mission, newest first, with its status and archived readings. Tick *show test runs and aborted* to see those too. **Open** shows an earlier mission's page read-only (address `?mission=<id>`): its sols, statistics, timeline (marked where it ended), data health, events and all downloads.

**The Pi's SD card** files by mission id. After a restart, new readings go into a new folder `<mission>-<new id>/sol-01/`, and the old folder stays. After an abort, readings go back to the previous mission's folders, or to `no-mission/`.

**The archiver** also finishes a restarted mission: its cut-short last sol is archived 10 minutes after the restart.

## Calibrations (ESP32 sensor board)

From the Pi, run `.venv/bin/python sensor_drivers/esp32_bridge.py --send "<command>"`:

| Command | Does |
|---|---|
| `CAL_O2` | After 5 min in fresh outdoor air: the O₂ cell reads 20.9 %. Stored in the sensor. |
| `CAL_CO2 [ppm]` | After 3 min in fresh outdoor air (default 420 ppm): SCD40 forced recalibration. **It also turns automatic self-calibration off.** That calibration needs about 7 days of regular fresh air, as long as the mission, and would drift instead. Stored in the sensor. |
| `ASC_ON` | Turns SCD40 self-calibration back on, after the mission. |
| `CAL_MQ4` | After 24–48 h burn-in, warm, in clean air: stores the MQ-4's R0 in flash. |
| `CAL_BNO_CLEAR` | Forgets the stored BNO055 calibration. |
| `SCD_TEST` | SCD40 self-test (10 s). Use it when CO₂ reads 0 but temperature and humidity don't: *passed* means the 3.3 V supply sags during the sensor's lamp pulses (give it its own supply and short wires); *FAILED* means the sensor itself. |
| `SCD_RESET` | SCD40 factory reset: forgets a forced recalibration and stored settings. Run `CAL_CO2` again afterwards. |
| `SCD_OFF` / `SCD_ON` | Stop / resume using the SCD40 (remembered across restarts). Use `SCD_OFF` for a **faulty SCD40 that hangs the I2C bus and takes the other sensors down with it** — the firmware then never touches it and BME280, O₂ and BNO055 keep working. |

## Self-heal if the sensors freeze

If every I2C sensor on the internal board goes silent while the board keeps running (a dead
device holding the shared bus — seen with a failed SCD40), the firmware recovers the bus after
45 s and, if that doesn't bring the sensors back, reboots the board after 2 min to clear it. A
reboot skips the MQ-4 warm-up, so it costs only a few seconds of data. This is a safety net —
the real fix for a faulty device is `SCD_OFF` (above) or removing it.

## Self-heal if the board drops off the network

The task watchdog only resets a board whose loop hangs. A board that is running but
**unreachable** (Wi-Fi lost and not coming back, its web server or `imm-sensors.local` name gone
quiet) used to stay that way until someone pressed RESET, with nothing recorded meanwhile. Now the
firmware mends it, without a reboot where it can, so the gap is only the outage itself:

| What the board sees | What it does | Data gap |
|---|---|---|
| Wi-Fi down 20 s | Rejoins the network, every 20 s | The outage only |
| Wi-Fi still down after 3 min | Reboots — **once per outage** (a router that is switched off is waited for, not rebooted against) | ~10 s on top |
| Connected, but the Pi hasn't read it for 5 min | Restarts its web server and mDNS name | none |
| Still not read after 15 min | Reboots — once, until the Pi reads it again (a Pi that is off doesn't cause a reboot loop) | ~10 s |
| Free memory below 16 KB (after 10 min up) | Reboots before the network stack runs out | ~10 s |

On the Pi, the reader polls the IP that `imm-sensors.local` resolves to and **keeps that IP if the
name stops resolving**, so a quiet mDNS answer costs no data. It looks the name up again after a
failed poll, so a new DHCP address is picked up by itself.

A self-heal reboot keeps the MQ-4 hot (no warm-up) and the BNO055 calibration.

### Every restart is labelled in the data

Each start of the board is in the record with **when** and **why**:

- **Mission page → sol events:** `RESTART ESP32 board on node-rpi-01 (zone_a) restarted:
  self-heal reboot: Wi-Fi lost (boot 25293, no data for ~200 s)`. The same line is in the printed
  sol report.
- **Sol download → `summary.json` → `"reboots"`:** one entry per restart: `at_ist`, `boot`, `why`
  (power-on, RESET button, brownout, watchdog, crash, or `self-heal reboot: <cause>`),
  `reset_reason`, `heal_cause`, `last_heard` and `gap_s` (the gap, to within the 10 s health
  interval).
- **Sol download → the board's CSVs** (`bme280`, `o2`, `bno055`, `mq4`, `scd40`, `board`): two
  more columns, `board_boot` (changes at every restart) and `since_boot_s`. Filter or group by
  `board_boot` to separate the data before and after a restart.
- **`board.csv`** carries the health counters every 10 s: `heal_cause` (0 = not a self-heal boot,
  1 I2C bus stall, 2 Wi-Fi lost, 3 not polled by the Pi, 4 memory low), `heal_reboots` (total since
  flashing), `wifi_drops`, `wifi_reason`, `net_restarts`, `heap_free`, `heap_min`.

`wifi_reason` is why the Wi-Fi link last dropped (ESP-IDF codes): **8** the board left, **2/15**
authentication or handshake timeout (password or router trouble), **200** beacon timeout (lost the
router's signal), **201** no access point found (router off or out of range).

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
- `POST /restart` (`confirm`, `keep_as`: restarted | test, `start_ist`), `POST /abort` (`confirm`), `POST /label` (`test`), `GET /history` (`?all=1`);
- every read and download takes `?mission=<id>` for an earlier mission;
- `GET /overview`, `/sol/{n}`, `/timeline?measurement=`, `/overlay?measurement=`, `/health`, `/dose`;
- `POST /events`;
- `POST /download-token`;
- `GET /download/sol/{n}`, `/download/mission`, `/report/{n}`.

Code: `imm-os-backend/services/mission.py` (pure logic) and `mission_api.py` (API, InfluxDB, archive).
