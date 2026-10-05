# IMM OS Documentation
All documentation, ADRs, API specs.

## Operations and safety

- **[Operations runbook](operations-runbook.md): start the whole system, and a fix with commands for every field problem (Wi-Fi drops, changing IPs, reflashing, SD card, sensor calibration) — start here.**

- [Mission record](mission-operations.md): sols (24 h from the start), IST, the 7-sol archive, downloads, warm-up and calibrations
- [FDIR strategy](fdir-strategy.md): fault detection, isolation and recovery, layer by layer
- [Operating modes](operating-modes.md): mission mode, subsystem GO / NO-GO, alarm lifecycle, degraded operation
- [Telemetry quality](telemetry-quality.md): quality and health flags on every reading
- [EVA contingency](eva-contingency.md): loss of signal handling
- [Backup and disaster recovery](backup-and-dr.md)
- [Switching everything off and on again](power-off-on.md): shutdown order, start-up order, what's normal after power-on
- [Pi node not on the network](pi-recovery.md): find it, bring it back, and keep it from happening again
- [V&V plan and mission readiness test](vv-plan.md)
