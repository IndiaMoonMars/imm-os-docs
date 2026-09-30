# Pi node not on the network: find it and bring it back

Use this when the MCC PC can't reach a Pi node, e.g. `ssh` or `provision-pi.ps1` says
`Could not resolve hostname node-rpi-01.local` or `can't find node-rpi-01.local`.

All commands run in **Windows PowerShell on the MCC PC**, in `C:\Users\PRATHAM\Documents\imm-os-edge`,
unless the step says "on the Pi".

## 1. Is the Pi on the network at all?

```powershell
ping -4 -n 2 node-rpi-01.local
```

If the name doesn't resolve, the Pi may still be on the network under a new IP address, or the
PC's name lookup (mDNS) may have failed. Look for it by address:

```powershell
# Ping every address once, so Windows learns which devices exist (1-2 min, prints nothing)
1..254 | ForEach-Object { ping.exe -n 1 -w 300 "192.168.1.$_" | Out-Null }

# Raspberry Pi network adapters (Pi 5: 2c-cf-67, d8-3a-dd, 88-a2-9e; older Pis: b8-27-eb, dc-a6-32, e4-5f-01)
arp -a | findstr /i "2c-cf-67 d8-3a-dd 88-a2-9e b8-27-eb dc-a6-32 e4-5f-01"

# Every device that accepts SSH (the router at .1 usually does as well)
1..254 | ForEach-Object { $ip = "192.168.1.$_"; if ((New-Object Net.Sockets.TcpClient).ConnectAsync($ip, 22).Wait(300)) { Write-Host "SSH open on $ip" } }
```

- **Found an IP:** check with `ssh pratham@<ip> hostname` (it prints `node-rpi-01`), then use the IP in
  place of the name, e.g. `-PiHost <ip>`.
- **Nothing found:** the router's device list (`http://192.168.1.1`, then DHCP clients or connected devices)
  is the last check. If `node-rpi-01` isn't there either, the Pi is **not on the network**: go to step 2.

Windows PowerShell 5.1 has no `Test-Connection -TimeoutSeconds`. Use `ping.exe -w` as above.

## 2. Why a Pi drops off the network

| Cause | Sign | Fix |
|---|---|---|
| **Weak power** (laptop USB port, small charger) | Green LED flickers endlessly; the Pi never shows up, or keeps dropping off | The official 27 W USB-C supply (5.1 V / 5 A), or at least a 5 V / 3 A USB-C charger. Never the laptop. |
| **Power pulled while running** | It worked before the unplug and hasn't come back since | Step 3 |
| **Still booting** | The first boot after a power cut checks the SD card | Wait a full 5 minutes before deciding |
| **Wi-Fi didn't reconnect** | A LAN cable works but Wi-Fi doesn't | Step 3, option B |

## 3. Bring it back without removing the SD card

**A. Clean restart.**
1. Unplug the ESP32 boards from the Pi.
2. Unplug the Pi's supply, wait 10 s, and plug the **wall supply** back in.
3. Wait 5 minutes, then repeat step 1.

**B. LAN cable.** Connect the Pi to a LAN port on the router, wait 2 minutes, and repeat step 1. If it
answers, rejoin the Wi-Fi (the SSID and password are the habitat network's):

```powershell
ssh -t pratham@node-rpi-01.local "sudo nmcli radio wifi on; sudo nmcli device wifi rescan; sleep 5; sudo nmcli device wifi connect '<SSID>' password '<Wi-Fi password>'; hostname -I"
```

Then unplug the cable.

**C. Monitor and keyboard.** Use the Pi 5's micro-HDMI port **next to the USB-C power port**, plus a USB
keyboard. Power-cycle the Pi and watch the screen:

- **Login prompt:** log in, then run
  `sudo nmcli device wifi connect '<SSID>' password '<Wi-Fi password>'` and `hostname -I`.
- **`emergency mode` / `root account is locked`:** the SD card's file system was damaged by the power cut
  and the automatic repair didn't finish. Raspberry Pi OS locks the root account, so you can't repair it
  from this screen. Take the card out and put it in the PC. In the `bootfs` drive, open `cmdline.txt` in
  Notepad and add ` fsck.mode=force fsck.repair=yes` to the end of its **single** line: keep it one line,
  with a space before each item. Save the file, put the card back and boot. The Pi repairs the card
  during the boot, then remove the two words again. If it still stops, go to step 4.
- **Nothing, or only the Raspberry Pi boot/diagnostic screen:** the Pi can't boot from the card. Go to step 4.

## 4. Re-install the SD card (the fix when nothing else works)

Nothing that matters lives only on the card: the code comes from the MCC PC, and the readings are on the MCC
in InfluxDB. Only the SD-card CSV copy (`/var/lib/imm-os/records`) and the blackbox are lost.

1. Take the microSD card out (with the power off) and put it in the PC.
2. **Raspberry Pi Imager:** choose **Raspberry Pi 5**, then **Raspberry Pi OS Lite (64-bit)**, then the card.
   Click **Next** and **Edit settings**:
   - hostname `node-rpi-01`, user `pratham`;
   - wireless LAN (the habitat network), country `IN`;
   - time zone `Asia/Kolkata`;
   - **Services:** enable SSH (password).

   Click **Save**, then **Yes**, then **Yes**. If you answer *No* to "apply OS customisation settings", the Pi
   boots with no Wi-Fi and no SSH.
3. Put the card back, use the wall supply, and wait 5 minutes.
4. Set the node up again:

   ```powershell
   ssh-keygen -R node-rpi-01.local
   .\scripts\provision-pi.ps1 -PiUser pratham -PiHost node-rpi-01.local -ExtBoard http://<external board ip>/data
   ```

   Then add the internal board as in [mission-operations.md](mission-operations.md) (flash, `WIFI_SSID` /
   `WIFI_PASS`, `-IntBoard http://<board ip>/json`).

## Prevention

- **Shut down before unplugging:** run `ssh -t pratham@node-rpi-01.local "sudo poweroff"`, and remove power
  once the green LED stops flickering.
- **Use a wall supply,** never a laptop USB port. `tools/bringup.py all` reports under-voltage, and so does
  the Sensors page (`sysmon`).
- **Reserve fixed IPs** in the router (DHCP reservation) for `node-rpi-01` and both ESP32 boards, so the
  addresses survive power cuts.
- **Before a mission, put the Pi on a UPS or power bank with pass-through,** so a mains cut doesn't become an
  unclean power-off.
