# IMM-OS operations runbook — start everything, and fix anything

One place for: how the pieces connect, how to switch the whole system on, and a fix (with
commands) for every problem seen in the field — Wi-Fi drops, changing IPs, reflashing, the SD
card, sensors reading wrong. Run every PowerShell command on the **MCC laptop**, in
`C:\Users\PRATHAM\Documents\imm-os-edge`, unless a step says "on the Pi".

Replace `<Wi-Fi name>` and `<Wi-Fi password>` with your real network's values when you type a
command (they are not written in this file on purpose).

## Contents
1. [The four pieces and how they talk](#1-the-four-pieces)
2. [Switch everything ON (the happy path)](#2-switch-everything-on)
3. [Switch everything OFF](#3-switch-everything-off)
4. [Fix anything — by symptom](#4-fix-anything--by-symptom)
5. [Make the problems stop for good](#5-make-the-problems-stop-for-good)
6. [When you change the Wi-Fi router](#6-when-you-change-the-wi-fi-router)
7. [Quick reference card](#7-quick-reference-card)

---

## 1. The four pieces

```
Internal board (BME280/SCD40/BNO055/O2/MQ-4)  --Wi-Fi-->  \
   name: imm-sensors.local   page: /json                   \
                                                             >  Raspberry Pi (node-rpi-01)  --> MCC laptop
External board (Geiger/GNSS)                  --Wi-Fi-->   /     reads both boards, records       dashboard
   its own firmware          page: /data                  /      to SD card, sends to MCC         http://imm.local
```

- **The Pi reads each board over Wi-Fi** and forwards to the MCC. The boards don't need a USB
  cable for data — only for flashing and calibration. Day-to-day they just need **power**.
- **The Pi finds the internal board by name** (`imm-sensors.local`), so its IP can change freely.
- **The external board has no name** (it runs your own firmware), so it's reached by IP or by a
  network scan (`find`).
- **Everything depends on the Wi-Fi being up and the addresses lining up.** Most problems below
  are one of those two things.

---

## 2. Switch everything ON

Do it in this order. The *why* matters — out of order, things fail to connect.

1. **Router** on. Wait ~1 min. *(Everything joins this network.)*
2. **Laptop** on → open **Docker Desktop**, wait for **"Engine running"**. *(The dashboard and
   database are Docker containers; they auto-start once the engine is up.)*
   - Make it automatic: Docker Desktop → Settings → General → **Start Docker Desktop when you sign in**.
3. **Raspberry Pi** on, from its **27 W wall charger** — never a laptop USB port. *(A Pi 5 needs
   5 V / 5 A; weak power makes it restart forever and never join Wi-Fi.)*
4. **Both boards** on (internal on a Pi USB port or a 5 V adapter; external on its charger).
5. Wait **2–3 min**, then:
   ```powershell
   ping -4 -n 2 node-rpi-01.local
   curl.exe -m 5 http://imm-sensors.local/json
   curl.exe -m 5 http://192.168.1.125/data
   ```
   All three should answer. (The external IP may differ — see [§4.5](#45-a-board-changed-its-ip).)
6. Point the Pi at both boards and go live:
   ```powershell
   Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass
   .\scripts\provision-pi.ps1 -PiUser pratham -PiHost node-rpi-01.local -Sensors "esp32_bridge.py external_board_bridge.py" -ExtBoard http://192.168.1.125/data -IntBoard http://imm-sensors.local/json
   ```
7. Open **http://imm.local → Sensors** (Ctrl+F5). Cards should say **"now"** and tick.

**Normal right after power-on:** MQ-4 shows **WARMING** for 3 min; Geiger shows a provisional
count for 60 s; GNSS has no fix until it sees open sky; BNO055 may be uncalibrated (see §4.11–4.12).

---

## 3. Switch everything OFF

1. **Shut the Pi down first — never just pull the plug** (that corrupts the SD card):
   ```powershell
   ssh -t pratham@node-rpi-01.local "sudo poweroff"
   ```
   Wait until the Pi's green LED stops flickering, then remove power.
2. Unplug the two boards (safe any time).
3. Shut down the laptop normally.

---

## 4. Fix anything — by symptom

### 4.1 "running scripts is disabled on this system"
PowerShell blocks scripts by default. Allow it for this window only:
```powershell
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass
```
Type `Y` if asked. It resets when you close the window. Or run the script in one shot:
```powershell
powershell -ExecutionPolicy Bypass -File .\scripts\provision-pi.ps1 -PiUser pratham -PiHost node-rpi-01.local -Sensors "esp32_bridge.py external_board_bridge.py" -ExtBoard http://192.168.1.125/data -IntBoard http://imm-sensors.local/json
```

### 4.2 Can't find `node-rpi-01.local` / the Pi doesn't answer
```powershell
ping -4 -n 2 node-rpi-01.local
```
If it fails:
1. **Wait** — a fresh boot takes 2–3 min (a first boot after re-flashing, up to 5).
2. **Check power:** the Pi must be on its **wall charger**, not laptop USB. Green LED flickering = OK.
3. **Find it by address** (the name lookup can lag):
   ```powershell
   1..254 | ForEach-Object { ping.exe -n 1 -w 300 "192.168.1.$_" | Out-Null }
   arp -a | findstr /i "88-a2-9e 2c-cf-67 d8-3a-dd"
   ```
   The line shown is the Pi; use that IP as `-PiHost <ip>` and `ssh pratham@<ip>`.
4. Still nothing after 5 min → [pi-recovery.md](pi-recovery.md) (monitor/keyboard, or SD re-flash).

### 4.3 The Wi-Fi dropped
Symptom: commands suddenly fail with "could not resolve host" / "connection timed out", often
several at once. The network blinked — nothing is broken.
1. Confirm the laptop is back on the Wi-Fi (taskbar icon).
2. Wait 1–2 min for the Pi and boards to rejoin on their own.
3. `ping -4 -n 2 node-rpi-01.local` until it replies, then re-run whatever failed.

A Wi-Fi drop often **changes the boards' IPs** — if a board is then unreachable, see §4.5.

### 4.4 `imm-sensors.local` won't resolve (Windows)
`curl: (6) Could not resolve host: imm-sensors.local`, usually right after a Wi-Fi blip. Windows
lost the name record; the board is fine. Two options:
- Wait ~1 min and retry; or
- Use the board's **IP** for now. Find it:
  ```powershell
  1..254 | ForEach-Object { ping.exe -n 1 -w 300 "192.168.1.$_" | Out-Null }
  arp -a | findstr /i "68-09-47"
  ```
  Then use `-IntBoard http://192.168.1.<ip>/json` in the provision command.

The **Pi** resolves the name with avahi (independently of Windows). Check the Pi can:
```powershell
ssh -t pratham@node-rpi-01.local "sudo systemctl enable --now avahi-daemon && sleep 3 && getent hosts imm-sensors.local"
```
It should print the board's current IP. If `avahi-daemon` isn't installed, run `setup-node.sh`
again (via `provision-pi.ps1`) — it now installs it.

### 4.5 A board changed its IP
The router handed it a new address (common after a power cut or Wi-Fi drop).
- **Internal board:** you don't care about its IP — use `http://imm-sensors.local/json`. If the
  name won't resolve from Windows, find the IP by its MAC `68-09-47-b4-79-b4` (see §4.4).
- **External board:** find it by probing every device for its `/data` page:
  ```powershell
  1..254 | ForEach-Object { ping.exe -n 1 -w 200 "192.168.1.$_" | Out-Null }
  (arp -a | Select-String '192\.168\.1\.\d+' | ForEach-Object { ($_ -split '\s+')[1] }) | ForEach-Object {
    $o = curl.exe -s -m 2 "http://$_/data" 2>$null; if ($o -match 'rad|cpm|gnss') { "FOUND $_ -> $o" } }
  ```
  The IP that prints with `rad`/`gnss` data is the external board. Its MAC is `90-15-06-95-09-bc`.
- Then re-run `provision-pi.ps1` with the correct `-IntBoard`/`-ExtBoard` addresses.

### 4.6 The external board isn't found at all (not just a new IP)
`external board not found` and no `/data` anywhere means it's **off or not on Wi-Fi**.
1. Check its **power LED**; try a known-good cable/adapter; wait 30 s.
2. If it still won't join, give it the Wi-Fi again over USB (plug it into the Pi):
   ```powershell
   $E = "cd ~/imm-os-edge && EXT_BOARD_PORT=/dev/ttyUSB0 .venv/bin/python sensor_drivers/external_board_bridge.py --send"
   ssh -t pratham@node-rpi-01.local "$E 'WIFI_SSID <Wi-Fi name>'"
   ssh -t pratham@node-rpi-01.local "$E 'WIFI_PASS <Wi-Fi password>'"
   ```
   (Only if it's the IMM-OS external firmware. If it's your own firmware, set its Wi-Fi however
   that firmware expects.)

### 4.7 One board fails bring-up and blocks the others ("only node health → LIVE")
`provision-pi.ps1` won't switch the real sensors live if **any** sensor fails its check — so a
single unreachable board leaves everything simulated. Work around it by setting up only the
board that's working, so the other can't block it:
- **Internal only:**
  ```powershell
  .\scripts\provision-pi.ps1 -PiUser pratham -PiHost node-rpi-01.local -Sensors "esp32_bridge.py" -IntBoard http://imm-sensors.local/json
  ```
- **External only:**
  ```powershell
  .\scripts\provision-pi.ps1 -PiUser pratham -PiHost node-rpi-01.local -Sensors "external_board_bridge.py" -ExtBoard http://192.168.1.125/data
  ```
If a dead address is still configured and keeps failing the check, clear it on the Pi first:
```powershell
ssh -t pratham@node-rpi-01.local "sudo sed -i '/^EXT_BOARD_URL=/d;/^EXT_BOARD_PORT=/d' /etc/imm-os/edge.env"
```
Then add that board back once it's reachable.

### 4.8 Flashing fails: "Unable to verify flash chip connection (No serial data received)"
The upload speed was too high for the cable/board. The firmware now flashes at 115200, which is
reliable, so just pull the latest and flash again:
```powershell
git pull
.\scripts\provision-pi.ps1 -PiUser pratham -PiHost node-rpi-01.local -CopyOnly
ssh -t pratham@node-rpi-01.local "cd ~/imm-os-edge && ESP32_PORT=/dev/ttyUSB0 ./scripts/flash-esp32.sh"
```
If it still stalls at `Connecting....`, **hold the board's BOOT button** until `Writing at…`
starts, and use a short, good-quality USB **data** cable in a Pi USB port.

### 4.9 The board keeps rebooting / "reset reason 4 (CRASH)" / web-server handler errors
This was a firmware bug (the web server started before Wi-Fi) and is fixed. Make sure the board
runs current firmware: `git pull`, then re-flash as in §4.8. After flashing, `STATUS` should show
`reset reason 1`, not `4`. ("Handler not found" also appears if you open the wrong page on a
board — internal serves **/json**, external serves **/data**.)

### 4.10 SD card corrupted (Pi won't boot after an unclean power-off)
If the Pi never comes back and a monitor shows boot errors, re-flash the card. Nothing important
is lost — code is on GitHub, readings are on the MCC.
1. Put the microSD card in the laptop.
2. **Raspberry Pi Imager** → **Raspberry Pi 5** → **Raspberry Pi OS Lite (64-bit)** → the card.
3. **Edit settings**: hostname `node-rpi-01`, user `pratham`, Wi-Fi `<Wi-Fi name>`/`<Wi-Fi password>`,
   country `IN`, time zone `Asia/Kolkata`, **Services → enable SSH (password)**. **Save → Yes → Yes**.
4. Card back in the Pi, wall charger, wait 5 min, then:
   ```powershell
   ssh-keygen -R node-rpi-01.local
   .\scripts\provision-pi.ps1 -PiUser pratham -PiHost node-rpi-01.local -ExtBoard find -IntBoard http://imm-sensors.local/json
   ```
Details and a repair-without-reflash option: [pi-recovery.md](pi-recovery.md).

### 4.11 CO₂ (SCD40) shows nothing
First minute after start, CO₂ reads 0 while the sensor settles — normal. If it **stays** blank
while temperature/humidity work, the sensor is faulty or its 3.3 V supply dips. Test it (board on
the Pi's USB):
```powershell
$B = "cd ~/imm-os-edge && ESP32_PORT=/dev/ttyUSB0 .venv/bin/python sensor_drivers/esp32_bridge.py --send"
ssh -t pratham@node-rpi-01.local "$B SCD_TEST"
```
- `passed` → power dip: give the SCD40 its own 3.3 V/5 V and short wires.
- `FAILED` → run `ssh -t pratham@node-rpi-01.local "$B SCD_RESET"`, wait 1 min, test again; if it
  still fails, replace the SCD40 (same wires, no code change).

### 4.12 Sensors show "no data · 10 min ago" (frozen numbers)
The values are old; new data stopped. Usually the last `provision-pi.ps1` failed partway (often on
a board that was unreachable), so the readers didn't restart. Re-run the provision (§2 step 6, or
the single-board version in §4.7). When it finishes cleanly, the cards go back to "now".

### 4.13 The internal board went offline and only RESET brings it back
All internal sensors **and** the board's own health stop together, the external board carries on,
and pressing the ESP32's RESET button fixes it. The board was running but unreachable. Firmware
`9bc3923` and later mends this itself (rejoin, restart, reboot as a last resort; see
[mission-operations.md](mission-operations.md#self-heal-if-the-board-drops-off-the-network)), so
update the board first (§4.8 for flashing). To see **which** fault it was, read the Pi's log
around the time it stopped:
```powershell
ssh pratham@node-rpi-01.local "journalctl -u imm-sensor-pipeline@esp32_bridge.py --since '2026-10-07 05:20' --until '2026-10-07 05:40' --no-pager | grep -i -E 'not answering|resolv|restarted|reachable'"
```
The **first** line of an outage is its cause (only failures 1, 10 and every 60th are logged); later
lines can show a name failure that is just a consequence: a board that left the Wi-Fi also stops
answering for its name once the Pi's cached answer runs out (~1 min).

| The first line says | What happened |
|---|---|
| `Name or service not known` / `name not resolving (mDNS)` | The board was fine; its name went quiet. The reader now keeps polling its last IP. |
| `timed out` / `no route to host` | The board was off the Wi-Fi, without power or frozen. Rejoin/reboot now handle it. If it keeps happening, check the supply (below). |
| `refused` | On the network, web server down: restarted after 5 min now. |

Example (7 Oct 2026, the night this was found): `05:28:26 … timed out`, then `05:29:28 … Name or
service not known` until RESET: the board dropped off the Wi-Fi, and its own reconnect never brought
it back.

Then check the board itself: `ssh -t pratham@node-rpi-01.local "$B STATUS"` (on the Pi's USB) shows
`network: N Wi-Fi drop(s) (reason R …), … self-heal reboot(s)`. Many drops with reason **200** mean
weak signal; repeated `reset reason 9 (BROWNOUT)` in `board.csv` means the supply sags: give the
sensors their own 3.3 V regulator rather than the ESP32 board's, and the MQ-4 heater the 5 V input.

### 4.14 Pi health check: "MQTT TLS port / MQTT login / Keycloak unreachable", "MCC link DOWN"
The Pi can't reach the laptop. **No data is lost:** the Pi stores everything ("N message(s) stored
in the local broker") and sends it when the link is back. The Pi also re-finds the laptop by itself
every minute if its IP changed, but only if the laptop's port 8883 answers. So the cause is on the
laptop. The error tells which:

| Error | Meaning |
|---|---|
| `No route to host` | Nothing at that IP: the laptop got a new IP, or it is asleep / off the Wi-Fi |
| `timed out` | Laptop there, Windows firewall blocking (network set to Public) |
| `Connection refused` | Laptop there, IMM-OS containers not running |

Fix, in **PowerShell opened with "Run as administrator"** on the laptop:
```powershell
# 1. What is the laptop's IP now? (the Pi's error shows the one it is trying)
Get-NetIPAddress -AddressFamily IPv4 | Where-Object PrefixOrigin -eq Dhcp | Select-Object InterfaceAlias, IPAddress
# 2. Containers running? (Docker Desktop must say "Engine running"; every service should be Up/healthy)
cd C:\Users\PRATHAM\Documents\imm-os-infra
docker compose up -d
docker compose ps
# 3. Network Private (Windows sometimes puts it back to Public after a reconnect)
Get-NetConnectionProfile | Where-Object NetworkCategory -eq 'Public' | Set-NetConnectionProfile -NetworkCategory Private
# 4. The ports answer on the laptop itself
Test-NetConnection localhost -Port 8883 | Select-Object TcpTestSucceeded
Test-NetConnection localhost -Port 80   | Select-Object TcpTestSucceeded
# 5. Don't let the laptop sleep on mains (an overnight run stops when it sleeps)
powercfg /change standby-timeout-ac 0
powercfg /change hibernate-timeout-ac 0
```
Then make the Pi look for the laptop now instead of within the minute, and check again:
```powershell
ssh -t pratham@node-rpi-01.local "cd ~/imm-os-edge && sudo .venv/bin/python tools/find_mcc.py --scan"
```
It prints the laptop's address when found. The stored messages then drain by themselves (watch
the number fall in the health check). If it still says not found, step 2 or 3 is the problem:
re-run `provision-pi.ps1` as Administrator; it adds the firewall rule and warns about a Public network.

### 4.15 Readings look wrong (not missing)
- **MQ-4 "NOT CALIBRATED" / methane blank:** after 24–48 h powered burn-in, in clean air:
  `ssh -t pratham@node-rpi-01.local "$B CAL_MQ4"` (board on the Pi's USB).
- **O₂ not ~20.9 %:** in fresh outdoor air: `... "$B CAL_O2"`.
- **Orientation "SUSPECT: uncalibrated":** rotate the board slowly in a figure-8 until calibration
  reads 3/3; it's then saved and restored at every start.
- **GNSS no fix:** the antenna needs a view of open sky; the first fix takes minutes.

---

## 5. Make the problems stop for good

Most of today's pain is **unstable Wi-Fi** + **changing IPs**. Fix those two and startup becomes
"switch it on".

1. **Reserve the addresses in the router** (DHCP reservation / Address reservation), so they never
   change — even across the Wi-Fi drops:

   | Device | MAC | Reserve |
   |---|---|---|
   | Pi `node-rpi-01` | `88-a2-9e-54-c3-bb` | e.g. `192.168.1.135` |
   | Internal board | `68-09-47-b4-79-b4` | e.g. `192.168.1.144` |
   | External board | `90-15-06-95-09-bc` | e.g. `192.168.1.140` |

   The internal board is already found by name (`imm-sensors.local`), so reserving it is a bonus;
   reserving the **external** board is the one that actually saves you re-running the setup.

2. **Give the Pi and boards a stronger, steadier Wi-Fi** (move the router closer, or add a mesh
   point). A drop mid-mission also punches a gap in your recorded data.

3. **Put the Pi on a UPS or a power bank with pass-through** during a mission, so a mains blip
   doesn't become an unclean power-off (which is what risks the SD card).

4. **Always shut the Pi down with `sudo poweroff`** before removing power (§3).

5. **Reserve IPs again after changing routers** — reservations live inside the router, so a new
   router needs them set once. The names (`node-rpi-01.local`, `imm-sensors.local`) keep working
   on any router with no setup.

6. **Read both sensor boards over two links at once: USB cable *and* Wi-Fi.** Each board sends
   every reading over its USB cable to the Pi and also serves it over Wi-Fi. The Pi reads both and
   keeps each reading once (the board's own counter tells the two copies apart), so losing either
   link loses nothing:

   | What fails | Readings still reach the Pi over | SD card | Dashboard |
   |---|---|---|---|
   | Router / Wi-Fi | USB | ✅ complete | gap while the laptop can't be reached, then filled in from the Pi's queue |
   | A board's Wi-Fi only | USB | ✅ | ✅ live |
   | A USB cable | Wi-Fi | ✅ | ✅ live |

   Every reading is saved on the Pi's SD card as CSV (`/var/lib/imm-os/records/<mission>/sol-NN/`,
   7 sols and more, nothing deleted) with a `via` column (`usb` or `wifi`: the link it came by), and
   the dashboard's ESP32 board cards show **USB link** and **Wi-Fi link** (1 = delivering). An
   advisory alarm says when one link goes silent. While the Pi reads its USB, the internal board
   never reboots itself over a Wi-Fi outage (it keeps rejoining instead), so its USB stream isn't
   interrupted.

   **Which board is where (pinned, so nothing can be mixed up):**

   | | Internal board | External board |
   |---|---|---|
   | USB chip → Pi port (pinned in `/etc/imm-os/edge.env`) | CP2102 → `ESP32_PORT=/dev/serial/by-id/usb-Silicon_Labs_CP2102_USB_to_UART_Bridge_Controller_0001-if00-port0` | CH340 → `EXT_BOARD_PORT=/dev/serial/by-id/usb-1a86_USB_Serial-if00-port0` (`EXT_BOARD_BAUD=115200`) |
   | Wi-Fi | `ESP32_URL=http://imm-sensors.local/json` | `EXT_BOARD_URL=http://192.168.1.125/data` |
   | Zone | `zone_a` | `exterior` |
   | Pins | I²C GPIO21 SDA / GPIO22 SCL: BME280 0x76, BNO055 0x28, SEN0322 O₂ 0x70–0x73, (SCD40 0x62, off); MQ-4 AO → divider → GPIO32 | SEN0463 Geiger pulses → GPIO4; TEL0157 GNSS I²C 0x20 on GPIO21/22 |
   | SD files | `bme280_zone_a.csv`, `o2_zone_a.csv`, `bno055_zone_a.csv`, `mq4_zone_a.csv`, `board_zone_a.csv` | `geiger_exterior.csv`, `gnss_exterior.csv`, `board_exterior.csv` |

   The ports are named by the board's USB chip, so either board can go in any Pi USB socket. Setup
   finds each one by what it prints and pins it; the readers and `flash-esp32.sh` use those pins
   and refuse to touch the other board's port. Use **USB data cables** (charge-only ones show nothing).

   **Set it up:**
   ```powershell
   cd C:\Users\PRATHAM\Documents\imm-os-edge
   git pull
   .\scripts\provision-pi.ps1 -PiUser pratham -PiHost node-rpi-01.local -CopyOnly
   # internal board: current firmware (knows the Pi reads its USB); flashed on its own pinned port
   ssh -t pratham@node-rpi-01.local "cd ~/imm-os-edge && ESP32_PORT=/dev/serial/by-id/usb-Silicon_Labs_CP2102_USB_to_UART_Bridge_Controller_0001-if00-port0 ./scripts/flash-esp32.sh internal"
   # both boards, both links
   .\scripts\provision-pi.ps1 -PiUser pratham -PiHost node-rpi-01.local -Sensors "esp32_bridge.py external_board_bridge.py" -IntBoard both -ExtBoard both
   ```
   Setup should print `internal sensor board: both links, USB on … and Wi-Fi at …` and `external
   board: both links, USB on … and Wi-Fi at …`.

   **The external board's sketch** must print its readings on USB for its USB link to carry them;
   until then setup says so and its readings come over Wi-Fi only. Add to its `loop()` (the same JSON
   its `/data` page sends, on one line, once a second; it has `"uptime"`, which tells the two copies apart):
   ```cpp
   static unsigned long lastUsb = 0;
   if (millis() - lastUsb >= 1000) {
     lastUsb = millis();
     Serial.println(dataJson);        // the String your /data handler sends
   }
   ```
   Flash it as you normally do, then run the last provision command again: nothing else changes.

   **Test it:** with a test run going, switch the router off for 5 minutes, then on. The SD files
   have no gap (rows with `via` = `usb` cover the outage):
   ```powershell
   ssh pratham@node-rpi-01.local "tail -3 /var/lib/imm-os/records/*/sol-*/bme280_zone_a.csv"
   ```
   and the dashboard fills the gap in once the laptop is reachable again.

---

## 6. When you change the Wi-Fi router

Every device (Pi + both boards) stores the Wi-Fi name and password, and the router hands out the
IPs — so a new router affects all of them. How much work it is depends on **one decision you make
when setting up the new router**.

### The one trick that makes it almost free
**Set the new router's Wi-Fi name (SSID) and password to be *exactly the same* as the old one.**
Then every device rejoins automatically — nothing to reconfigure. You only redo the IP
reservations (step C) and re-point the external board (step D). Do this if you possibly can.

If the new network must have a different name/password, you have to re-give the Wi-Fi to each
device (step B).

### A. Put the laptop on the new Wi-Fi
Connect the MCC laptop to the new network first, and start Docker Desktop as usual.

### B. Give each device the new Wi-Fi — ONLY if the name/password changed
Skip this entire step if you kept the same SSID and password.

- **Internal board** — plug it into the Pi's USB, then:
  ```powershell
  $B = "cd ~/imm-os-edge && ESP32_PORT=/dev/ttyUSB0 .venv/bin/python sensor_drivers/esp32_bridge.py --send"
  ssh -t pratham@node-rpi-01.local "$B 'WIFI_SSID <new Wi-Fi name>'"
  ssh -t pratham@node-rpi-01.local "$B 'WIFI_PASS <new Wi-Fi password>'"
  ssh -t pratham@node-rpi-01.local "$B STATUS"
  ```
  (This needs the Pi reachable — so do the Pi first, or send it with the board on a laptop using the
  board's own USB tools.)

- **Raspberry Pi** — it can't reach Wi-Fi to be told the new Wi-Fi, so use one of:
  1. **Ethernet cable** from the Pi to the new router, then:
     ```powershell
     ssh -t pratham@node-rpi-01.local "sudo nmcli device wifi connect '<new Wi-Fi name>' password '<new Wi-Fi password>'; hostname -I"
     ```
     Unplug the cable after.
  2. **Monitor + keyboard** on the Pi: log in and run the same `nmcli … connect …` line.
  3. **Re-flash the SD card** with Raspberry Pi Imager, putting the new Wi-Fi in its settings
     (§4.10) — the simplest if you have no cable or monitor.

- **External board** — plug it into the Pi's USB and set its Wi-Fi the way *its own firmware*
  expects (for the IMM-OS external firmware it's the same `WIFI_SSID`/`WIFI_PASS` commands as the
  internal board, with `EXT_BOARD_PORT=/dev/ttyUSB0` and `external_board_bridge.py`).

### C. Redo the IP reservations in the new router
Reservations live **inside the router**, so the new one starts with none. In the new router's
DHCP-reservation page, reserve the three MACs again (see §5):

| Device | MAC |
|---|---|
| Pi | `88-a2-9e-54-c3-bb` |
| Internal board | `68-09-47-b4-79-b4` |
| External board | `90-15-06-95-09-bc` |

### D. Re-point the Pi at the boards and go live
The external board will have a new IP on the new router — find it (§4.5) and provision:
```powershell
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass
.\scripts\provision-pi.ps1 -PiUser pratham -PiHost node-rpi-01.local -Sensors "esp32_bridge.py external_board_bridge.py" -ExtBoard http://192.168.1.<new-ext-ip>/data -IntBoard http://imm-sensors.local/json
```
The internal board needs no IP — `imm-sensors.local` works on the new router automatically.

### What you do NOT need to worry about on a new router
- **The names** `node-rpi-01.local` and `imm-sensors.local` work on any router — no change.
- **The MCC containers / dashboard** — unaffected; they run on the laptop.
- **Your recorded data** — untouched; it's in the database on the MCC and on the Pi's SD card.

> Checklist: laptop on new Wi-Fi → (if SSID/password changed) re-give Wi-Fi to Pi + both boards →
> reserve the 3 MACs → find the external board's new IP → run `provision-pi.ps1` → check
> http://imm.local.

---

## 7. Quick reference card

| Thing | Value |
|---|---|
| Pi | `node-rpi-01` · user `pratham` · `node-rpi-01.local` · MAC `88-a2-9e-54-c3-bb` |
| Internal board | `imm-sensors.local` · page `/json` · MAC `68-09-47-b4-79-b4` · USB `/dev/ttyUSB0` |
| External board | page `/data` · MAC `90-15-06-95-09-bc` · own firmware (no name) |
| Dashboard | `http://imm.local` → Sensors |
| Code on the laptop | `C:\Users\PRATHAM\Documents\imm-os-edge` (and `imm-os-infra`) |

**Set up both boards (the one command you run most):**
```powershell
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass
.\scripts\provision-pi.ps1 -PiUser pratham -PiHost node-rpi-01.local -Sensors "esp32_bridge.py external_board_bridge.py" -ExtBoard http://192.168.1.<ext-ip>/data -IntBoard http://imm-sensors.local/json
```

**Check the boards answer:**
```powershell
curl.exe -m 5 http://imm-sensors.local/json
curl.exe -m 5 http://192.168.1.<ext-ip>/data
```

**Talk to the internal board over USB (STATUS, calibrations):**
```powershell
$B = "cd ~/imm-os-edge && ESP32_PORT=/dev/ttyUSB0 .venv/bin/python sensor_drivers/esp32_bridge.py --send"
ssh -t pratham@node-rpi-01.local "$B STATUS"
```

**Shut the Pi down safely:**
```powershell
ssh -t pratham@node-rpi-01.local "sudo poweroff"
```

See also: [power-off-on.md](power-off-on.md) (shutdown/startup detail) and
[pi-recovery.md](pi-recovery.md) (Pi not on the network / SD-card repair).
