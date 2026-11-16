# Crash logs (pstore)

The router keeps kernel crash logs in RAM, so a panic can be read after the
reboot without a serial console attached. This is on the `pstore` branch
until an image with it has been booted from flash.

## How it works

- Linux uses the RAM up to 0x1b800000 (see the memory node in the device
  tree). The 8 MiB from there up to the I/O window at 0x1c000000 are free.
  Their contents survive a reset: markers written there were still there
  after a reboot. A power cut clears them.
- The device tree puts a 1 MiB ramoops area at 0x1b800000: five 128 KiB
  slots for panic dumps and 256 KiB for a copy of the console, with ECC.
- `kmod-ramoops` is in the device packages, and OpenWrt mounts the records
  at `/sys/fs/pstore`.
- `/etc/init.d/pstore-save` copies them to `/root/pstore/<date>/` after an
  unexpected reboot and keeps the newest five. A clean reboot writes a
  marker to the console first, so those are not saved.

## Reading the logs

```sh
ls /root/pstore/
cat /root/pstore/*/dmesg-ramoops-0
```

- `dmesg-ramoops-N`: the kernel log at a panic or oops.
- `console-ramoops-0`: the console of the previous boot, up to the reset.
  It is there after any reset that kept the RAM, also without a panic.
- Nothing saved after an unexpected reboot: the RAM lost power (a power cut
  or the supply dipping), or the crash came in the first ~25 seconds of
  boot, before ramoops is loaded.

## Tested

- On build17, with the same area set through ramoops module parameters: a
  crash from `echo c > /proc/sysrq-trigger` was saved, with the panic line
  and `ECC: No errors detected`. A normal reboot saved nothing.
- The device tree version builds. It has not been booted yet.
