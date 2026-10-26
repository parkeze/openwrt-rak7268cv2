# openwrt-rak7268cv2

OpenWrt device support for the RAKwireless RAK7268CV2 (WisGate Edge Lite 2 V2),
and the ImageBuilders built with it. OpenWrt 25.12.6, `ramips/mt76x8`, profile
`rakwireless_rak7268cv2`.

The device tree and recipe are the ones prepared for upstream OpenWrt (the
shared `mt7628an_rakwireless_rak636.dtsi` is common to the RAK7289CV2). Once a
release carries them, this repo is no longer needed.

Built from [openwrt-device-template](https://github.com/parkeze/openwrt-device-template).

## Using a release

Each release is tagged `<openwrt-version>-<n>` and its assets are laid out like
a target directory on downloads.openwrt.org:

```
openwrt-imagebuilder-25.12.6-ramips-mt76x8.Linux-x86_64.tar.zst
openwrt-25.12.6-ramips-mt76x8-rakwireless_rak7268cv2-initramfs-kernel.bin
openwrt-25.12.6-ramips-mt76x8-rakwireless_rak7268cv2-squashfs-sysupgrade.bin
sha256sums
```

So anything that fetches an official ImageBuilder and checks it against
`sha256sums` only needs a different base URL:

```
https://github.com/parkeze/openwrt-rak7268cv2/releases/download/25.12.6-1/
```

The ImageBuilder is the release's own tree with these devices added. Its
kernel is built from the release's `config.buildinfo`, and the build fails
unless its vermagic matches the release's kmods, so every upstream kmod and
package still installs.

The one package the release does not have, `kmod-leds-aw2013`, is built here
(see below) and added to the ImageBuilder's local repository, beside the
kernel it was built against.

## Building

```bash
./build.sh 25.12.6 ramips mt76x8 out
```

Linux, on a case-sensitive filesystem, with
[OpenWrt's build dependencies](https://openwrt.org/docs/guide-developer/toolchain/install-buildsystem).
The first build compiles the toolchain and takes a couple of hours; CI does it
on every push and publishes on a tag.

## RAK7268CV2

A RAK636 module: MT7628AN, 128MB RAM, 32MB W25Q256 SPI NOR, SX1302 LoRa
concentrator on SPI CS1 (`/dev/spidev0.1`, reset on GPIO 11, active high;
the MT7628 SPI controller is half-duplex only), Quectel EG95 LTE on USB, SD card slot on the ethernet
PHY's P1-P4 pads (so one ethernet port), and an RGB status light on top.

### The RGB status light

An Awinic AW2013 three-channel driver at I2C 0x45. The kernel's `leds-aw2013`
driver exposes it as `red:status`, `green:status` and `blue:status`. Release
kernels do not build that driver, and building it in would change the
vermagic, so `devices/rak7268cv2/package/leds-aw2013` builds it out of tree as
`kmod-leds-aw2013`. Its source is Linux 6.12's driver plus hardware breathing,
the patch prepared for upstream: the pattern trigger's `hw_pattern` takes four
stages, `0 rise high hold high fall 0 off`, and the chip breathes on its own.
The same string in `pattern` is the software fallback on a kernel without the
patch:

```sh
echo pattern > /sys/class/leds/blue:status/trigger           # kmod-ledtrig-pattern
echo "0 2080 255 130 255 2080 0 260" > /sys/class/leds/blue:status/hw_pattern
```

Durations round up to 130 ms times a power of two.

Not yet run on a unit: the AW2013 node, the module and its breathing were
written without a RAK7268CV2 on the bench. The register use matches the
vendor firmware's `breathing_led`, which is known to drive this light.

Flash map, identical to the vendor's so the bootloader, calibration and RAK's
unit data are untouched:

| Offset | Size | Partition |
|---|---|---|
| 0x000000 | 0x30000 | u-boot (read-only) |
| 0x030000 | 0x10000 | u-boot-env |
| 0x040000 | 0xe000 | factory: WiFi calibration and MACs (read-only) |
| 0x04e000 | 0x2000 | pst-data: RAK's unit identity (read-only) |
| 0x050000 | 0x1fb0000 | firmware |

### Installing

The stock Ralink U-Boot has a TFTP menu on the serial console (57600 8N1,
`bootdelay` 5s). **Back up the whole flash from the stock firmware first.**

- **1: Load system code to SDRAM via TFTP** boots the `initramfs-kernel.bin`
  without writing anything. Use it to try an image.
- **2: Load system code then write to Flash via TFTP** writes a
  `squashfs-sysupgrade.bin` to the firmware partition.
- Never 7 or 9: those rewrite the bootloader.

U-Boot's network needs a host that accepts its MAC: the stock environment has
`ethaddr=03:17:73:AB:CD:EF`, a multicast address that switches, Wi-Fi and
macOS all drop, and `ethaddr` cannot be changed from the prompt. A Linux or
Windows machine on a direct cable works. Otherwise `loadb` takes the image over
the serial console (Kermit, ~3.4 KB/s at 57600 with long packets), and
installing from the running stock firmware over the network needs neither:

```sh
wget -O /tmp/fw.bin http://<host>/openwrt-...-squashfs-sysupgrade.bin
sha256sum /tmp/fw.bin
mtd -r write /tmp/fw.bin firmware
```

From a running OpenWrt, `sysupgrade` takes the same `squashfs-sysupgrade.bin`.

## Licence

GPL-2.0, as OpenWrt. The device trees are `GPL-2.0-or-later OR MIT`.
