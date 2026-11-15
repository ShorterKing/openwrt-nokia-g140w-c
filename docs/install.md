# Install OpenWrt to flash

OpenWrt goes into firmware slot A. The stock firmware stays in slot B, with
its settings, so you can switch back to it at any time.

> **Read this first.** This writes to the router's flash. A mistake can
> leave it unable to boot until you recover it over the serial console.
> The router may belong to your ISP. Do this at your own risk.

## You need

- Everything from the [RAM boot guide](ram-boot.md): a 3.3 V serial adapter
  and a terminal that can send XMODEM.
- Two images built from this repository (see [building.md](building.md)):
  - `...-nokia_g140w-c-initramfs-kernel.bin`: the **production** RAM image.
    Not the `-bringup-` one, which doesn't touch slot A and refuses to
    install.
  - `...-nokia_g140w-c-squashfs-sysupgrade.bin`.

## 1. Boot OpenWrt from RAM

Follow the [RAM boot guide](ram-boot.md) with the production initramfs
image. Connect only your computer, to LAN1–LAN3, and leave the WAN port
(LAN4) unplugged: the RAM image uses `192.168.1.1`, which many home
networks use too.

## 2. Back up the flash

Do this once, and keep the files private: they hold your router's MAC
address, serial number and ISP settings. From a Linux or macOS terminal,
or Windows `cmd` (PowerShell mangles binary output):

```
for i in 0 1 5 6 7 8 9 10 11 12 13 14 15 16; do ssh root@192.168.1.1 "cat /dev/mtd$i" > mtd$i.bin; done
```

`cat /proc/mtd` on the router lists the partitions. Check each file
against `sha256sum /dev/mtdN` on the router. Slot A (`mtd2`–`mtd4`) is
left out: it usually holds an old stock copy that the install overwrites,
and on the reference unit it had read errors.

## 3. Install

Copy the sysupgrade image to the router and run it, without keeping
settings (the RAM image can't save any):

```
ssh root@192.168.1.1 "cat > /tmp/sysupgrade.bin" < openwrt-econet-en751221-nokia_g140w-c-squashfs-sysupgrade.bin
ssh root@192.168.1.1 "sysupgrade -n /tmp/sysupgrade.bin"
```

The LuCI page *System → Backup / Flash Firmware* works too: untick *Keep
settings*.

The upgrade then:

1. Copies the bootloader's second stage (the "bootram") from the router.
   The bootloader needs it in the slot it boots, and it's Nokia's code, so
   it isn't in the image. See [the hardware notes](hardware.md#bootloader-stages).
   If it can't find one, the image check fails and nothing is written.
2. Copies the Wi-Fi calibration from the stock settings, read-only, unless
   it already has a copy. See [the hardware notes](hardware.md#wi-fi-calibration).
   If it can't find it, the install goes ahead and Wi-Fi runs uncalibrated.
3. Selects slot A for the next boot.
4. Writes the kernel with the bootram, the Wi-Fi calibration and the root
   filesystem, and reboots.

The router comes up as OpenWrt from flash, at `192.168.1.1` again.

## After installing

- Set a root password, with `passwd` on the console or in LuCI.
- Wi-Fi is off by default; turn it on as in the
  [RAM boot guide](ram-boot.md#after-booting).
- Settings now survive reboots.

## Upgrading

Use `sysupgrade` or LuCI as on any OpenWrt router; keeping settings works.
The bootram is copied over again each time, and the Wi-Fi calibration
volume is kept.

`sysupgrade` runs the upgrade scripts of the installed system, not of
the new image. So when upgrading from a build without the calibration
volume (before 2026-11-14), the first upgrade doesn't create it, and
Wi-Fi stays off afterwards. Run the same upgrade a second time: the
scripts that are installed now create the volume.

## Going back to stock

```
en75_chboot factory; reboot
```

The stock firmware should then boot from slot B with its old settings,
with OpenWrt still in slot A. No way is known to switch slots from the
stock firmware, so to return, boot OpenWrt from RAM again, run
`en75_chboot openwrt`, and power-cycle.

## If it doesn't boot

The bootloader counts boots that don't finish. After nine, it switches to
the other slot by itself: power-cycle the router nine times and the stock
firmware comes up. OpenWrt resets the count after every boot, so this
doesn't happen during normal use.

To get back to OpenWrt from there, boot the production image from RAM as
in step 1. Its `sysupgrade` reinstalls OpenWrt to slot A and selects it
again. To keep your settings, use the backup that the failed upgrade left
in slot A, if there is one: mount `rootfs_data` read-only, copy
`upper/sysupgrade.tgz` out, unmount, and pass it with `-f`:

```
mkdir /tmp/d; mount -t ubifs -o ro ubi0:rootfs_data /tmp/d; cp /tmp/d/upper/sysupgrade.tgz /tmp/cfg.tgz; umount /tmp/d
sysupgrade -f /tmp/cfg.tgz /tmp/sysupgrade.bin
```

Otherwise use `sysupgrade -n`, or `-f` with a backup from LuCI.

## What was tested

On the reference unit:

- Upgrading an installed OpenWrt with `sysupgrade`, keeping settings.
- Recovering from the stock firmware: a RAM boot, then `sysupgrade -f`
  with the settings backup from slot A. That upgrade also created the
  Wi-Fi calibration volume, and both radios then used exactly the
  calibration the stock settings hold.
- Finding the bootram in slot A and in slot B, and building the slot A
  image from it, in a dry run that matched a working image byte for byte.
- Reboots, with the boot count cleared each time.
- The fallback to stock: an image that froze at boot made the bootloader
  start the stock firmware from slot B after nine failed boots, with
  OpenWrt and its settings intact in slot A.

The first install there was done by hand, with the steps the upgrade now
automates. Not tried yet: installing onto a router that never had
OpenWrt (slot A still holds a stock image, so the upgrade formats it as
UBI first), and going back to stock with `en75_chboot factory`.
