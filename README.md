# Waveshare USB-CAN-FD Linux driver (SocketCAN)

An open-source **Linux driver for the Waveshare USB-CAN-FD** adapter (the original, non-"B" model, SKU 25029),
which Waveshare only supports on Windows. It provides native **SocketCAN** interfaces (`can0`, `can1`) with
**classic CAN and CAN FD** (bit-rate switching up to 5 Mbit/s) on both isolated channels. So the adapter works
with `ip link`, `candump`/`cansend` (can-utils), python-can's `socketcan` interface, Wireshark,
[CANgaroo](https://github.com/Schildkroet/CANgaroo) and anything else that speaks SocketCAN.

![CANgaroo decoding traffic through the wscanfd driver](docs/cangaroo.png)

## Is this your adapter?

```
$ lsusb
Bus 003 Device 007: ID 04d8:0053 Microchip Technology, Inc. USB CANFD DEBUG
```

- **Waveshare USB-CAN-FD**: two isolated CAN/CAN FD channels, USB-B, NXP LPC54616 inside. This driver is for this model.
- **Waveshare USB-CAN-FD-B**: the newer model already has vendor Linux support, so you don't need this driver.

## Quick start (kernel driver)

```sh
sudo apt install build-essential linux-headers-$(uname -r) can-utils   # Debian/Ubuntu
git clone https://github.com/harvie256/waveshare-usb-can-fd-linux.git
cd waveshare-usb-can-fd-linux
make -C kernel
sudo modprobe can-dev
sudo insmod kernel/wscanfd.ko

# classic CAN, 250 kbit/s, on the connector labelled CAN1
sudo ip link set can0 type can bitrate 250000
sudo ip link set can0 up
candump can0

# CAN FD, 500 kbit/s nominal / 2 Mbit/s data, on CAN2
sudo ip link set can1 type can bitrate 500000 sample-point 0.8 dbitrate 2000000 fd on
sudo ip link set can1 up
cansend can1 123##1DEADBEEF
```

`dmesg` shows the adapter's serial number and firmware version when it binds. `can0` is the CAN1 connector and
`can1` is CAN2. The 120 Ω termination is set with the physical switches on the case.

### Install with DKMS (survives kernel updates)

```sh
sudo apt install dkms
sudo mkdir -p /usr/src/wscanfd-0.1.0
sudo cp kernel/wscanfd.c kernel/Makefile kernel/dkms.conf /usr/src/wscanfd-0.1.0/
sudo dkms install wscanfd/0.1.0
```

The module then loads automatically whenever the adapter is plugged in, and DKMS rebuilds it for each new kernel.
Check with `dkms status`. To update, `sudo dkms remove wscanfd/0.1.0 --all`, copy the new files and install
again. To uninstall, run `sudo dkms remove wscanfd/0.1.0 --all` and `sudo rm -r /usr/src/wscanfd-0.1.0`.

With Secure Boot on, DKMS signs the module with a local Machine Owner Key. The first time, `dkms` (or
`update-secureboot-policy --enroll-key`) asks for a password. Reboot and enrol the key in the blue MOK manager
screen, or the kernel will refuse to load the module.

### Recommended settings

- **CAN FD with bit-rate switching: use the same *nominal* sample point as the other nodes, normally
  `sample-point 0.8`** (CiA 601-3). The kernel's default (87.5 % at 250k/500k) against 80 % peers makes FD+BRS
  frames from those peers fail. Classic CAN is unaffected.
- Bursty senders are fine. The driver paces transmission itself (see below), so a large `txqueuelen` is safe.
- **Don't transmit on a channel with no other node on its bus.** See "Unacknowledged frames stall both channels"
  below.

### Module parameters

`/sys/module/wscanfd/parameters/`, all writable at runtime:

| Parameter | Default | Meaning |
|---|---|---|
| `tx_dev_queue` | 4 | Frames the driver lets queue inside the adapter, and USB transfers in flight, per channel (1-8) |
| `tx_min_frame_us` | 300 | Firmware per-frame cost floor, µs; the driver adds 2 µs per data byte |
| `idle_poll_us` | 1000 | RX poll interval while the bus is idle. Lower means less latency and more USB interrupts |

## How the adapter behaves (and how the driver copes)

The stock firmware has a few limitations that the driver works around:

- **No TX flow control.** The adapter buffers only about 10 frames, keeps accepting USB transfers when full and
  silently discards the excess. It also needs about 300 µs plus 2 µs per data byte for every frame it handles,
  received or transmitted, whatever the bit rate. The driver models the adapter's queue from the bit rate, frame
  length and received traffic, and stops the netdev queue before the adapter overflows. Result: zero loss on
  2000-frame back-to-back bursts and under a 1:1 echo load. TX tops out at about 3300 short frames/s.
- **No TX confirmation.** An echoed frame (IFF_ECHO) means the adapter accepted it. Frames nobody ACKs are
  retried for about 0.5 s and then dropped.
- **Unacknowledged frames stall both channels.** All transmissions for both channels share one USB endpoint, and
  the firmware stops reading it while it retries an unacknowledged frame. A channel sending into a bus with
  nobody to ACK (unconnected, wrong bit rate, peer off) slows the *other* channel to a few frames per second.
  After 3 s of such traffic, the other channel still waited about 12 s for its next frame. The driver keeps at
  most `tx_dev_queue` USB transfers in flight per channel to limit this (18 s without that limit), but it
  can't reach frames already buffered in the firmware.
- **No bus-error or bus-state reporting**, so interfaces always report ERROR-ACTIVE and error counters are
  unsupported.
- **RX endpoints never NAK.** Idle polls are paced with an hrtimer, which costs about 2 USB interrupts per ms per
  open channel.
- **Never send a second USB SET_CONFIGURATION.** The firmware stops responding until it is power-cycled.
  Neither driver does this.

## Userspace driver (Python, no kernel module)

`wscanfd/` is a pure-Python driver on pyusb/libusb with the same protocol, pacing and features, plus a small CLI and
a bridge to a `vcan` interface. It's useful for scripting and for machines where you can't build modules. It
detaches the kernel driver while it runs.

```sh
python3 -m venv .venv && .venv/bin/pip install pyusb pytest
sudo install -m 644 udev/70-wscanfd.rules /etc/udev/rules.d/ && sudo udevadm control --reload   # then replug
.venv/bin/python -m wscanfd info
.venv/bin/python -m wscanfd dump -c both -b 250000
.venv/bin/python -m wscanfd send -c 0 -b 500000 -d 2000000 123#11223344 456##1AABBCCDD
.venv/bin/python -m wscanfd bridge -c 0 -b 250000 -i vcan0      # needs a vcan0 interface
```

The udev rule is only for this userspace driver. The kernel driver doesn't need it, so skip it if you only use
`can0`/`can1`. It gives members of the `plugdev` group raw USB access to the adapter. Anyone with that access can
transmit arbitrary frames on both buses, detach the kernel driver and send the firmware-update command, so only
add users you trust with the bus (`sudo usermod -aG plugdev <user>`, then log in again).

## Testing

Everything was tested against a real bus with a LilyGo T-2CAN-FD (ESP32-S3 + MCP2518FD) running the echo
firmware in [`t2can-echo/`](t2can-echo). The firmware returns every frame with ID+1 and inverted data, and takes
serial commands to change the bit rate.

| Test | Result |
|---|---|
| Round-trip: classic 0-8 B, CAN FD 0-64 B with and without BRS, standard and extended IDs (82 cases) | 82/82 at 125k/1M, 250k/2M, 500k/2M, 500k/4M, 1M/5M, on both channels |
| Bit-rate sweep (userspace driver) | 50k to 1M nominal, 500k to 5M data, all pass |
| 2000-frame bursts (classic 8 B, FD 0 B BRS, FD 64 B BRS) | zero loss at 250k/2M, 500k/2M, 1M/5M |
| Sustained classic TX at 250 kbit/s | 1878 frames/s, about the bus limit |
| [CANgaroo](https://github.com/Schildkroet/CANgaroo) v0.7.1 via SocketCAN ([`tools/cangaroo/`](tools/cangaroo)) | 82/82 round-trip, 500/500 FD64+BRS burst |
| Listen-only mode | working |
| DKMS install, auto-load on plug-in, rebuild/reinstall (Ubuntu 26.04, kernel 7.0) | working |
| Unplugging with both interfaces up and transmitting | clean: senders get `ENXIO`, interfaces removed, no kernel warnings |

Test tools: [`tools/socketcan_roundtrip.py`](tools/socketcan_roundtrip.py) (kernel driver),
[`tools/fd_roundtrip.py`](tools/fd_roundtrip.py) (userspace driver), `tests/` (protocol unit tests, `pytest`).

Not tested yet: several adapters at once.

## Documentation

**[Online documentation](https://harvie256.github.io/waveshare-usb-can-fd-linux/)** ([`docs/index.html`](docs/index.html)) has the full illustrated reference: hardware, USB protocol and wire formats,
firmware behaviour, both drivers' internals, and the test rig and results.

## Repository layout

| Path | Contents |
|---|---|
| `kernel/` | SocketCAN kernel module `wscanfd.ko`, Makefile, `dkms.conf` |
| `wscanfd/` | Python userspace driver, CLI and vcan bridge |
| `tests/` | protocol unit tests |
| `tools/` | hardware test scripts, CANgaroo test workspace |
| `t2can-echo/` | ESP-IDF echo firmware for the LilyGo T-2CAN-FD test peer |
| `udev/` | udev rule for non-root access (userspace driver only) |
| `docs/` | HTML documentation |

## Disclaimer

This project is not affiliated with or endorsed by Waveshare. Use at your own risk. The adapter's USB ID
(`04d8:0053`) is the one the stock firmware reports.

## License

GPL-2.0. See [LICENSE](LICENSE).
