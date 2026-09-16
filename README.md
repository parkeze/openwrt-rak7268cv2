# openwrt-devices

OpenWrt device support for hardware the release does not cover yet, and the
ImageBuilders built with it.

| Device | OpenWrt | Target | Profile |
|---|---|---|---|
| RAKwireless RAK7268CV2 (WisGate Edge Lite 2 V2) | 24.10.0 | ramips/mt76x8 | `rakwireless_rak7268cv2` |

## Using a release

Each release is tagged `<openwrt-version>-<n>` and its assets are laid out like
a target directory on downloads.openwrt.org:

```
openwrt-imagebuilder-24.10.0-ramips-mt76x8.Linux-x86_64.tar.zst
openwrt-24.10.0-ramips-mt76x8-rakwireless_rak7268cv2-initramfs-kernel.bin
openwrt-24.10.0-ramips-mt76x8-rakwireless_rak7268cv2-squashfs-sysupgrade.bin
sha256sums
```

So anything that fetches an official ImageBuilder and checks it against
`sha256sums` only needs a different base URL:

```
https://github.com/parkeze/openwrt-devices/releases/download/24.10.0-1/
```

The ImageBuilder is the release's own tree with these devices added. Its
kernel is built from the release's `config.buildinfo`, and the build fails
unless its vermagic matches the release's kmods, so every upstream kmod and
package still installs.

## Building

```bash
./build.sh 24.10.0 ramips mt76x8 out
```

Linux, on a case-sensitive filesystem, with
[OpenWrt's build dependencies](https://openwrt.org/docs/guide-developer/toolchain/install-buildsystem).
The first build compiles the toolchain and takes a couple of hours; CI does it
on every push and publishes on a tag.

## Adding a device

`devices/<board>/` holds the DTS and a patch against the OpenWrt tree for
everything else: the image recipe in `target/linux/<target>/image/`, and
whatever `board.d` network, LED and `uboot-envtools` entries the board needs.
`build.sh` picks the device name out of the patch's `define Device/...`.

Write the DTS from the vendor's running firmware, not from a datasheet: most
vendor firmware is OpenWrt underneath, and `/sys/firmware/fdt`, `/proc/mtd` and
`/sys/kernel/debug/gpio` give the flash map and the wiring directly.

## RAK7268CV2

A RAK636 module: MT7628AN, 128MB RAM, 32MB W25Q256 SPI NOR, SX1302 LoRa
concentrator on SPI CS1, Quectel EG95 LTE on USB, SD card slot on the ethernet
PHY's P1-P4 pads (so one ethernet port).

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

From a running OpenWrt, `sysupgrade` takes the same `squashfs-sysupgrade.bin`.

## Licence

GPL-2.0, as OpenWrt. The device trees are `GPL-2.0-or-later OR MIT`.
