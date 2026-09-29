# Data backup and disaster recovery

## What is protected, and how

| Data | Where it lives | Protection |
|---|---|---|
| Postgres (alarms and history, plans, crew, inventory, Keycloak realm data) | `postgres_data` volume | `pg_dump` every 6 h, verified (`pg_restore --list`), row counts recorded |
| InfluxDB (all telemetry) | `influxdb_data` volume | `influx backup` every 6 h, manifest verified, point count of a closed 24 h window recorded |
| Telemetry in transit | Kafka (10-day retention), each node's local broker queue, each node's blackbox (9 days) | nothing is committed until stored; replay tools |
| Mission record | `MISSION_ARCHIVE_DIR`: every finished sol, per-sensor CSVs + summary + SHA-256 ([mission-operations.md](mission-operations.md)) | put it on an external drive |
| Configuration and secrets | `imm-os-infra/.env`, `mosquitto/certs/`, the nodes' `/etc/imm-os/` | **keep an offline copy in a password manager or safe**: not in the automatic backups |
| Code | GitHub | – |

## Backups

The `backup-postgres` and `backup-influx` containers run the backups into `BACKUP_DIR`
(default `imm-os-infra/backups/`):

- interval `BACKUP_INTERVAL_H` (6); kept `BACKUP_KEEP_DAYS` (14), and always at least the 4 newest;
- each backup gets a `.sha256` and a counts file;
- a failed backup raises a caution alarm and is retried in 15 min;
- the container goes unhealthy if the last good backup is older than two intervals, so autoheal
  restarts it (which takes a backup at once). If that also fails, the "gave up" alarm follows.

**Keep a copy off the MCC PC.** Point `BACKUP_DIR` at an external drive (for example
`BACKUP_DIR=D:/imm-backups` in `.env`), or copy `backups/` to another machine daily. A backup on
the same disk does not survive the disk.

**Targets.** RPO ≤ 6 h for databases (telemetry also sits in Kafka for 7 days). RTO ≤ 30 min to a
working MCC on replacement hardware.

## Practise: the DR drill (safe, anytime)

```powershell
.\scripts\dr-drill.ps1
```

1. Takes fresh backups.
2. Restores them into **scratch containers** (live data is never touched).
3. Checks every table's row count, and the InfluxDB points of the recorded window, against the counts recorded at backup time.
4. Reports the restore time (RTO) and removes the scratch containers.

It is also part of the mission readiness test (VV-DR-01..03). Measured in the sandbox, with a
small database: about 3 s per restore.

## Restore live data

```powershell
.\scripts\restore.ps1 postgres            # newest backup; or -File backups\postgres\imm_db-...dump
.\scripts\restore.ps1 influx
```

The script:
1. checks the backup's checksum;
2. asks you to type RESTORE;
3. takes a safety backup of the current data (if the database is up);
4. stops the services that write to that database;
5. restores (`pg_restore --clean` / `influx restore --full`);
6. compares row counts (Postgres);
7. starts everything again.

## Disaster: the MCC PC is lost

1. Install Docker Desktop on the replacement PC. Clone `imm-os-infra`, `imm-os-backend`,
   `imm-os-frontend` and `imm-os-openmct` side by side (same branch as before).
2. Put back `.env` and `mosquitto/certs/` from your offline copy. **Use the same CA**, or every
   node needs the new `ca.crt`.
3. `docker compose up -d postgres influxdb`, then restore both from the newest off-PC backup.
4. `docker compose up -d`.
5. Give the new PC the old IP, or let the nodes find it: `imm-mcc-discovery` re-finds the MCC by its
   certificate within a minute.
6. The nodes' queued readings arrive on their own. Check the Health tab for gaps. Replay a
   window from a node's blackbox if needed:
   `core/blackbox_replay.py --since … --until …`.
7. Run `.\scripts\mission-readiness.ps1 --quick` before going back to operations.

## Disaster: an edge node is lost

Flash a new SD card, run `scripts/setup-node.sh` with the same `--node-id` and `--zone`, and put back
`/etc/imm-os/calibration.yaml` if you kept it. The MCC treats it as the same node. Readings still in the
old node's queue or blackbox are lost unless its SD card can be read.
