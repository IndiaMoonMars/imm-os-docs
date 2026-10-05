# Switching everything off and on again

This is for the whole IMM-OS setup: the MCC laptop, the Raspberry Pi (`node-rpi-01`), the internal ESP32 board and the external ESP32 board.

**You don't need to flash anything after a power cycle.** Each device keeps what it needs and starts by itself:

| Device | What it keeps | At power-on |
|---|---|---|
| Internal board (BME280, SCD40, BNO055, O₂, MQ-4) | Firmware and Wi-Fi in its flash; BNO055 calibration | Joins the Wi-Fi and serves readings at `http://imm-sensors.local/json` |
| External board (Geiger, GNSS) | Its own firmware and Wi-Fi | Serves readings at `http://192.168.1.139/data` |
| Raspberry Pi | IMM-OS services (started at boot), board addresses in `/etc/imm-os/edge.env` | Reads both boards over Wi-Fi, finds the MCC, sends the data, records it to the SD card |
| MCC laptop | IMM-OS containers (`restart: unless-stopped`) | The dashboard comes back once Docker Desktop is running |

Flash again only when the firmware changes or a board is replaced.

**Addresses are handled so you don't chase IPs** (firmware from Oct 2026 onward):

| Device | Reached by | Survives a router change? |
|---|---|---|
| Pi `node-rpi-01` | the name `node-rpi-01.local` | ✅ yes |
| Internal board | the name `imm-sensors.local` (it announces itself; the Pi uses `http://imm-sensors.local/json`) | ✅ yes |
| External board | the Pi scans and finds it (`-ExtBoard find`) | ✅ yes |

So with the current firmware **nothing is tied to an IP**: power on and it reconnects, on this router or any future one. (If a board still runs older firmware, reserve its IP in the router's DHCP settings instead — internal board MAC `68:09:47:b4:79:b4`.)

---

## Switching OFF

1. **Shut the Pi down properly. Never just pull its plug**: that can corrupt the SD card, which holds its copy of the mission data. From PowerShell on the laptop:

   ```powershell
   ssh -t pratham@node-rpi-01.local "sudo poweroff"
   ```

   Wait until the green LED next to the SD slot stops flickering (10–20 s), then unplug the Pi.

2. **Unplug the two ESP32 boards.** They're safe to unplug at any time.

3. **Shut down the laptop** normally.

During a mission, switching off leaves a gap in the data. It shows on the Mission page's data-health grid. The sols carry on counting: they are 24 h from T0, whatever is switched on.

---

## Switching ON

Go in this order: network first, then the MCC, then the Pi and the boards.

1. **Wi-Fi router** on. Wait about 1 minute.
2. **Laptop** on. Start **Docker Desktop** and wait for *Engine running*. The IMM-OS containers start by themselves.
   - To make this automatic: Docker Desktop → Settings → General → **Start Docker Desktop when you sign in**.
3. **Pi** on, from its **wall charger** (27 W, 5 V / 5 A). **Never power it from the laptop's USB port.** That's too weak for a Pi 5, and it stalls or keeps restarting.
4. **Both boards** on:
   - the internal board on a Pi USB port or a 5 V USB adapter;
   - the external board on its charger.
5. Wait **2–3 minutes**, then open **http://imm.local → Sensors**. Everything should show **LIVE**.

### Normal right after power-on

| Sensor | What you see |
|---|---|
| MQ-4 methane | **WARMING** for 3 minutes (heater warm-up after a real power cut) |
| Geiger | A provisional count for the first 60 s |
| GNSS | Needs open sky; the first position fix takes a few minutes |
| BNO055 orientation | Stored calibration restored; move the board gently if it isn't 3/3 yet |

---

## If something is missing

Run these in PowerShell on the laptop:

```powershell
ping -4 -n 1 node-rpi-01.local
curl.exe -m 5 http://imm-sensors.local/json
curl.exe -m 5 http://192.168.1.139/data
ssh pratham@node-rpi-01.local "systemctl is-active imm-sensor-pipeline@esp32_bridge.py imm-sensor-pipeline@external_board_bridge.py imm-sd-recorder"
```

| Problem | Fix |
|---|---|
| A board doesn't answer | Power-cycle it and wait 30 s. |
| A board has a new IP | Re-run the setup with the new IP (below), then reserve it in the router. |
| The Pi doesn't answer | Check it's on the wall charger and the green LED flickers; wait 3 minutes. Then see [pi-recovery.md](pi-recovery.md). |
| Dashboard doesn't load | Check that Docker Desktop is running, then run `docker compose up -d` in `imm-os-infra`. |

Re-run the setup after a board's IP changed (replace the IPs as needed):

```powershell
cd C:\Users\PRATHAM\Documents\imm-os-edge
.\scripts\provision-pi.ps1 -PiUser pratham -PiHost node-rpi-01.local -Sensors "esp32_bridge.py external_board_bridge.py" -ExtBoard find -IntBoard http://imm-sensors.local/json
```

Calibrations (`CAL_O2`, `CAL_CO2`, `CAL_MQ4`) need the internal board on the Pi's USB for a moment; see [mission-operations.md](mission-operations.md#calibrations-esp32-sensor-board). They don't need a flash.
