# Pwnagotchi on an original Pi Zero W — "my PC can't see it" fix

A complete, tested troubleshooting guide for the most common way a Pwnagotchi
(or any Pi USB-gadget project) fails to appear on a Windows PC over USB.

It documents a real debugging session: **the two things that were actually
wrong**, everything that was **ruled out**, and the **exact fix** — so if it
happens again, you can jump straight to the answer.

> **TL;DR** — On an *original* (32-bit) Raspberry Pi Zero W you need a **32-bit
> image**, and the USB gadget only enumerates if `config.txt` has
> **`dtoverlay=dwc2,dr_mode=peripheral`** (not plain `dtoverlay=dwc2`).

---

## The hardware this applies to

| Part | Detail |
|------|--------|
| Board | **Raspberry Pi Zero W (original)** — 32-bit / armv6, BCM2835 |
| Battery HAT | PiSugar 2 (works with or without a battery installed) |
| Display | Waveshare 2.13" e-Paper HAT **Rev 2.1 = "V2"** → `waveshare_2` |
| Host PC | Windows 11 |

### Is it a Zero W or a Zero 2 W? (this matters — it decides 32 vs 64-bit)
Remove the SD card, plug the Pi's **data** USB port into the PC; with no
bootable card it enters **USB boot mode** and reports a chip ID:

| USB PID (VID `0A5C`) | Chip | Board | Architecture |
|---|---|---|---|
| `2763` | BCM2835 | **Pi Zero / Zero W** | 32-bit (armv6) |
| `2764` | BCM2837 | Pi Zero 2 W | 64-bit |
| `2711` | BCM2711 | Pi 4 | 64-bit |
| `2712` | BCM2712 | Pi 5 | 64-bit |

> ⚠️ "**Pi 2 Sugar**" printed on the board is the **PiSugar 2** battery HAT,
> **not** "Pi Zero 2". Don't identify the board from the HAT.

---

## Symptoms

- Blank e-ink screen.
- PC sees **nothing** on USB — no drive, no network adapter, not even an
  "unknown device".
- With a 64-bit image, the green LED blinks **7 times** (repeating).

---

## Root causes (there were two, independent)

### 1. Wrong-architecture image → 7 flashes
The current Pwnagotchi build (jayofelony) ships **64-bit only**. An original
Pi Zero W is 32-bit, so firmware looks for a 32-bit `kernel.img` that a
64-bit image doesn't contain → **"kernel not found" = 7 flashes.**

**Fix:** use a **32-bit** image — the classic **evilsocket Pwnagotchi v1.5.5**
(`pwnagotchi-raspbian-lite-v1.5.5.zip`, contains an `.img`).

*(Raspberry Pi LED error codes: 3 = generic, 4 = start.elf, **7 = kernel not
found**, 8 = SDRAM, 4long+4short = unsupported/wrong-arch.)*

### 2. USB gadget never enters "peripheral" mode → PC sees nothing
The default `dtoverlay=dwc2` is OTG **auto-detect**. On this Pi it did **not**
switch into gadget/peripheral mode, so no USB device was ever presented.

**Fix (the key line)** — in `config.txt` on the FAT boot partition:

```ini
dtoverlay=dwc2,dr_mode=peripheral
```

The moment this was forced, the Pi enumerated on the PC.

---

## What was NOT the problem (all tested, all ruled out)

- ❌ The **SD card** — written *and read back byte-for-byte verified*.
- ❌ The **flashing process** — byte-verified every write.
- ❌ The **USB cable** — proven to carry data (enumerated in USB boot mode).
- ❌ **Power / the PiSugar** — a stock Raspberry Pi OS booted fine on it.
- ❌ **The Pi itself** — alive (USB boot mode) and fully boots a stock OS.
- ❌ **Windows settings** — Windows shows *any* USB device (even driverless);
  seeing *nothing* means nothing was sent, which is upstream of drivers.

**How the Pi was proven healthy:** flash a stock **Raspberry Pi OS Lite
(32-bit)**, add the boot-partition config below, boot it. If the first-boot
files (`ssh`, `userconf.txt`, `firstrun.sh`) get **consumed** and `resize` is
stripped from `cmdline.txt`, the Pi booted fully — so any remaining failure is
the USB-gadget config, not the board.

---

## The working recipe (original 32-bit Pi Zero W)

1. **Flash** a 32-bit image (evilsocket Pwnagotchi v1.5.5).
2. On the FAT **boot** partition, edit **`config.txt`** — under `[all]`:
   ```ini
   dtoverlay=dwc2,dr_mode=peripheral
   ```
3. In **`cmdline.txt`** (single line, e.g. after `rootwait`):
   ```
   modules-load=dwc2,g_ether
   ```
4. (Optional) create an empty file named **`ssh`** on the boot partition.
5. Boot the Pi (first boot takes several minutes; be patient).

> **Note on Raspberry Pi OS Bookworm:** the legacy `g_ether` may come up as a
> USB **serial** gadget instead of Ethernet. Pwnagotchi's Buster-based v1.5.5
> presents a proper **network** gadget.

---

## Connecting from Windows

- Pi address over USB: **`10.0.0.2`**
- Give the new USB/RNDIS adapter a **static IP**:
  - IP `10.0.0.1`  •  Mask `255.255.255.0`  •  Gateway *(blank)*
- If it shows as a **COM port** or **unknown device** (known issue,
  [evilsocket #1228](https://github.com/evilsocket/pwnagotchi/issues/1228)):
  Device Manager → the device → **Update driver** → *Let me pick* →
  **Network adapters → Microsoft → "Remote NDIS Compatible Device"**.
- Web UI: **http://10.0.0.2:8080** (default `changeme` / `changeme`)
- SSH: `ssh pi@10.0.0.2` (default password `raspberry`)

---

## Enable the e-ink display

Over SSH, edit `/etc/pwnagotchi/config.toml`:

```toml
ui.display.enabled = true
ui.display.type = "waveshare_2"   # 2.13" Rev 2.1 = V2
```

Reboot. If it stays blank, try `waveshare_3` or `waveshare_4` (Waveshare's
Rev numbers aren't always exact).

---

## Flashing safely on Windows (notes)

- **`rpi-imager --cli` (v2.0.x) can silently no-op** (returns success without
  writing). Verify afterward, or use a tool that reads back and compares.
- Windows **auto-mounts** removable media and fights a raw write. Disable it
  first: `diskpart` → `automount disable`, then `select disk N`, `clean`.
- **Always verify**: read the whole card back and compare to the source image
  (or check the image's published SHA-256 before flashing).

---

## One-line summary

> Original (32-bit) Pi Zero W → needs a **32-bit image**, and the USB gadget
> only works with **`dtoverlay=dwc2,dr_mode=peripheral`** in `config.txt`.

---

*Documented from a real end-to-end debugging session, Sept 2026.*
