<div align="center">

<img src="docs/assets/logo.svg" width="120" alt="tronsmart-elite-linux-rgb logo" />

# tronsmart-elite-linux-rgb

**Set the Tronsmart Elite keyboard colour from Linux.**

A small Python script that sets a static backlight colour and brightness on the Tronsmart Elite
mechanical keyboard (SONiX chip, USB id `0c45:7044`). Tronsmart's driver is Windows only and OpenRGB
does not detect this keyboard; the script speaks the driver's own protocol over USB with nothing but
the Python standard library.

[![License: PolyForm Noncommercial](https://img.shields.io/badge/License-PolyForm%20Noncommercial-D97757.svg)](LICENSE)
![Platform](https://img.shields.io/badge/platform-Linux%2064--bit-D97757)
![Stack](https://img.shields.io/badge/Python%203-standard%20library%20only-D97757)
![Status](https://img.shields.io/badge/status-tested%20on%20Tronsmart%20Elite-D97757)

</div>

---

## Why this exists

The keyboard's Fn shortcuts switch between effects, speeds and brightness levels, but they do not let
you pick the colour of the static mode. Only Tronsmart's "Tronsmart Elite Driver" can, and it runs on
Windows only. OpenRGB has no support for this model, and running the driver under Wine does not help
either (see [Things worth knowing](#things-worth-knowing) for why).

The colour setting is a single 64-byte USB message. This repo sends it from Linux.

---

## What it does

- **Sets a static colour**: any RGB value, written as six hex digits (`ff0000`, `#00ff88`).
- **Sets the brightness** from 0 to 6 (6 is the default and the keyboard's maximum).
- **Reads the current light setting** (effect, speed, brightness, colour) without changing anything.

It does not set animated effects or per-key colours. The keyboard's Fn shortcuts keep working
alongside it.

---

## Requirements

- Linux on a 64-bit machine, with Python 3.6 or newer.
- A Tronsmart Elite connected by USB (`lsusb` shows `0c45:7044 Microdia Tronsmart Elite`).
- `sudo`, once, to install the udev rule that lets your user talk to the keyboard.

---

## Install

There are no releases: the script is a single file, installed from `main`.

```bash
git clone https://github.com/JoseJimenez-M/tronsmart-elite-linux-rgb
cd tronsmart-elite-linux-rgb
sudo cp 60-tronsmart-elite.rules /etc/udev/rules.d/
sudo udevadm control --reload-rules && sudo udevadm trigger
install -m 755 tronsmart-color ~/.local/bin/
```

`~/.local/bin` must be on your `PATH` (it is by default on Ubuntu and Mint once the folder exists).

---

## First run

```bash
tronsmart-color              # read the current setting, changes nothing
tronsmart-color ff0000       # static red, brightness 6
tronsmart-color 00ff88 3     # static green, brightness 3
```

Reading first is a safe way to check that the udev rule works. A permission error means the rule is
not active yet: re-run the two `udevadm` commands, or unplug and replug the keyboard.

---

## How it fits together

| File | What it does |
|------|--------------|
| `tronsmart-color` | The script: finds the keyboard in sysfs, claims USB interface 2 and sends the HID feature reports. |
| `60-tronsmart-elite.rules` | udev rule that gives the logged-in user access to the keyboard's USB device node. |
| `docs/assets/` | Logo and social card. |

### The protocol

Every command is a 64-byte HID feature report with report id 0, sent as a USB control transfer to
**interface 2**: `SET_REPORT` is `bmRequestType 0x21, bRequest 0x09, wValue 0x0300, wIndex 2`, and
`GET_REPORT` is `0xA1, 0x01, 0x0300, 2`. Unused bytes are zero.

| Command | Bytes | Notes |
|---------|-------|-------|
| Set light | `04 effect 5-speed brightness option R G B` | Static colour is `04 01 05 <0-6> 07 R G B`. Option `07` means "use the RGB bytes". |
| Get light | `84`, then `GET_REPORT` | The reply has the same layout as "set light". |
| Who am I | `8F`, then `GET_REPORT` | Bytes 1-4 are a little-endian model id. |

Effect ids, from the driver: 0 off, 1 static, 2 breathing, 3 neon, 4 wave, 5 ripple, 6 raindrop,
7 snake, 8 reactive, 9 converge, 10 per-key picture. Only effect 1 has been tried with this script.

---

## Things worth knowing

- **Why OpenRGB and Wine cannot see it.** The keyboard exposes three USB interfaces. Interface 2 is
  the lighting channel, and it has only an interrupt OUT endpoint. Linux's `usbhid` refuses HID
  interfaces without an interrupt IN endpoint (`dmesg` shows "couldn't find an input interrupt
  endpoint"), so no `/dev/hidraw` node is created for it, and tools built on hidraw never see it.
  The script goes around that through usbfs, sending control transfers to interface 2 directly.
- **No root at run time.** The udev rule tags the USB device with `uaccess`, so the user logged in at
  the seat gets access to it and the script runs as you.
- **Pure white is sent as `#fafffa`.** The official driver does the same; the script copies it.
- **64-bit only.** The usbfs ioctl structure is packed by hand for the 64-bit layout. On a 32-bit
  system the ioctl would fail; `pyusb` would be the fix if that ever matters.
- **Where the protocol comes from.** The Windows driver (Tronsmart Elite Driver 0.9.17) is an
  Electron app that ships its TypeScript source maps. The commands above were read from them
  (`sdk/DriverIO/DeviceAPI/k68.ts`, `sdk/DriverIO/usb/MacUSB.ts`). The driver was never run, and no
  part of it is included in this repo.
- **Not tested yet:** the Elite Pro (the driver uses the same code path for it), the keyboard over
  its wireless receiver or Bluetooth, and whether the colour survives unplugging the keyboard.

---

## Development

There is no build and there are no dependencies. Before a change, compile the script and run the
read-only command against a connected keyboard:

```bash
python3 -m py_compile tronsmart-color
./tronsmart-color
```

---

## Bugs and contributions

Bug reports, and reports from other keyboards that use the same driver, are welcome as issues:
<https://github.com/JoseJimenez-M/tronsmart-elite-linux-rgb/issues>.

## License

Source-available under PolyForm Noncommercial 1.0.0. Commercial use requires a license: contact
jimenez331375@gmail.com.

Personal, study, research, hobby, and noncommercial organizational use is permitted. Commercial use
(companies, paid services, resale, including modified versions) requires a separate license. This is
not an OSI open source license. See [LICENSE](LICENSE).

Required Notice: Copyright 2026 Jose Jimenez (https://github.com/JoseJimenez-M)
