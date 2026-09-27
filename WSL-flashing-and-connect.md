# Flashing & connecting from Windows 11 + WSL2 (original Pi Zero W)

Companion to the main README. That doc explains *why* the Pi wouldn't enumerate
(32-bit image + `dtoverlay=dwc2,dr_mode=peripheral`). This doc is the **tested,
end-to-end procedure** for doing the whole thing from a Windows 11 box driving
WSL2 (Ubuntu), including the two gotchas that cost the most time:

1. `wsl --mount` **can't** attach a USB SD-card reader → flash from Windows instead.
2. Once the Pi boots, WSL binds the **wrong** USB-net driver → one-time rebind.

> No passwords, SSIDs, or tokens appear below — substitute your own where you see
> `YOUR_*` placeholders.

---

## 0. Hardware / environment used

| Part | Detail |
|------|--------|
| Board | Raspberry Pi Zero W (original, 32-bit armv6, BCM2835) |
| Image | evilsocket Pwnagotchi **v1.5.5** (Raspbian Buster, 32-bit) |
| Host | Windows 11 + WSL2 (Ubuntu), `usbipd-win` installed |
| Reader | USB SD-card reader (shows up as removable mass storage) |

---

## 1. Flashing — do it from Windows, not `wsl --mount`

`wsl --mount \\.\PHYSICALDRIVE<N> --bare` **fails on a USB SD reader**:

```
Error code: Wsl/Service/AttachDisk/MountDisk/0x8007000f
The system cannot find the drive specified.
```

That's because `wsl --mount` only accepts disks Windows treats as fixed; a USB
card reader is *removable media*, so it's refused. Don't fight it.

**Instead: flash from Windows with Raspberry Pi Imager**

1. Imager → **Use custom** → select `pwnagotchi-raspbian-lite-v1.5.5.img`.
2. Storage → the SD card (verify the size!).
3. **Skip** the "OS customisation" prompt (**No**) — those settings are for
   Raspberry Pi OS and can break a Pwnagotchi image.
4. Write. This also erases whatever was on the card.

> If Windows had the card **Offline** (e.g. from a failed `wsl --mount`), bring it
> back with `Set-Disk -Number <N> -IsOffline $false` in an admin PowerShell, or
> right-click the disk in Disk Management → **Online**.

---

## 2. Boot-partition config (on the FAT `boot` partition after flashing)

See `boot-config-example.txt`. The essentials:

- `config.txt` (under `[all]`): `dtoverlay=dwc2,dr_mode=peripheral`
- `cmdline.txt` (one line, after `rootwait`): `modules-load=dwc2,g_ether`
- empty file named `ssh`

You can edit these straight from WSL once the boot partition has a Windows drive
letter (e.g. `D:`), which it gets automatically after imaging:

```bash
# LF line endings must be preserved; edit in place, don't let an editor add CRLF.
```

Then eject, move the card to the Pi, and plug the PC into the Pi's **middle
(data) USB port** — not the PWR port.

---

## 3. Connecting after boot, from WSL2 (usbipd + a driver rebind)

Once booted, the Pi presents a **CDC-ECM USB-Ethernet gadget**:
`VID:PID 0525:a4a2`.

```powershell
usbipd list                              # find the busid of 0525:a4a2
usbipd attach --wsl --busid <PI_BUSID>   # 'Shared'->attach; no admin needed once bound
```

**The gotcha:** in WSL the `cdc_subset` driver claims the interface before
`cdc_ether` can, and you get a dead interface:

```
cdc_ether 1-1:1.0: probe with driver cdc_ether failed with error -16   # -16 = busy
```

Fix it by unloading `cdc_subset` and re-selecting the device's configuration so
`cdc_ether` binds, then give the host side the `10.0.0.1/24` address:

```bash
sudo modprobe -r cdc_subset
D=/sys/bus/usb/devices/1-1
echo 0 | sudo tee $D/bConfigurationValue; sleep 1
echo 1 | sudo tee $D/bConfigurationValue; sleep 2
IF=$(ls $D/1-1:1.0/net/)                 # e.g. enx8ad36561fe67
sudo ip addr replace 10.0.0.1/24 dev "$IF"
sudo ip link set "$IF" up
ping -c3 10.0.0.2
```

### One-shot reconnect helper

The attach + rebind must be repeated after **every Pi reboot or replug** (the
WSL interface name changes each time — it's derived from the MAC). Script it:

```sh
#!/bin/sh
# pi-connect.sh — reconnect the Pwnagotchi USB gadget to WSL and bring up 10.0.0.1
# Run as root:  wsl -u root sh ~/pi-connect.sh   (PowerShell)
#           or  sudo sh ~/pi-connect.sh          (Ubuntu)
set -e
PS='/mnt/c/Windows/System32/WindowsPowerShell/v1.0/powershell.exe'
BUSID=$("$PS" -NoProfile -Command "(usbipd list) -split \"\`n\" | Select-String '0525:a4a2' | ForEach-Object { (\$_ -split '\s+')[0] } | Select-Object -First 1" 2>/dev/null | tr -d '\r ')
[ -n "$BUSID" ] || { echo "Pi gadget (0525:a4a2) not found — booted and on the data port?"; exit 1; }
echo "Pi busid: $BUSID"
"$PS" -NoProfile -Command "usbipd attach --wsl --busid $BUSID" 2>&1 | tr -d '\r' | grep -iE 'error|already' || true
D=/sys/bus/usb/devices/1-1
for i in 1 2 3 4 5 6 7 8 9 10; do [ -d "$D" ] && break; sleep 1; done
modprobe -r cdc_subset 2>/dev/null || true
echo 0 > "$D/bConfigurationValue"; sleep 1
echo 1 > "$D/bConfigurationValue"; sleep 2
IF=$(ls "$D/1-1:1.0/net/" 2>/dev/null)
[ -n "$IF" ] || { echo "No net interface on the Pi's USB device"; exit 1; }
ip addr replace 10.0.0.1/24 dev "$IF"; ip link set "$IF" up; sleep 1
echo "iface=$IF"; ip -br addr show "$IF"
ping -c2 -W2 10.0.0.2 >/dev/null 2>&1 && echo "OK: Pi at 10.0.0.2" || echo "up but no ping yet — retry in a few seconds"
```

---

## 4. Verify

- `ping 10.0.0.2`
- `ssh pi@10.0.0.2`  (default image password is `raspberry` — **change it**)
- Web UI: `http://10.0.0.2:8080`  (default `changeme`/`changeme` — **change it**)
- Confirm the display without looking at the panel: the web UI renders the exact
  e-ink frame at `http://10.0.0.2:8080/ui` (a 250×122 PNG for the 2.13" V2).

---

## 5. Config choices worth noting (`/etc/pwnagotchi/config.toml`)

- **Whitelist your own SSID** so it won't attack your network:
  `main.whitelist = ["YOUR_SSID"]`
- **Solo / offline mode** — no peer discovery, no uploads to the online grid:
  ```toml
  personality.advertise = false
  main.plugins.grid.enabled = false
  main.plugins.grid.report  = false
  ```
- **Hidden networks need no special setting.** A cloaked AP still beacons its
  BSSID; pwnagotchi targets it by BSSID (`agent.py` handles the `<hidden>` case)
  and the SSID is revealed on a client re-association. `channels = []` (all
  channels) and a permissive `min_rssi` already maximise discovery. The only hard
  limit: a hidden AP with **no clients** yields no handshake.

---

*Tested end-to-end on Windows 11 + WSL2, Sept 2026.*
