# Building the images

The port is a patch series on top of OpenWrt `main`. These steps build the
same images that were tested on the reference unit.

## Requirements

A Linux system (or WSL2) with the usual OpenWrt build dependencies; see the
[OpenWrt build system guide](https://openwrt.org/docs/guide-developer/toolchain/install-buildsystem).
The first build takes about an hour on 16 threads and needs about 20 GB of
disk.

## Steps

```sh
git clone https://github.com/openwrt/openwrt.git
git clone https://github.com/ShorterKing/openwrt-nokia-g140w-c.git
cd openwrt
git checkout 7b46f9593e          # the commit the patches were made against
git am ../openwrt-nokia-g140w-c/patches/*.patch
./scripts/feeds update -a
./scripts/feeds install -a
cp ../openwrt-nokia-g140w-c/config/diffconfig .config
make defconfig
mkdir -p files/usr/sbin && cp ../openwrt-nokia-g140w-c/tools/g140wc-probe files/usr/sbin/
make download -j8
make -j"$(nproc)"
```

On **WSL**, remove the Windows entries from `PATH` before running `make`.
Otherwise the last step fails with `find: The relative path ... is included
in the PATH environment variable`:

```sh
env PATH="$(echo "$PATH" | tr ':' '\n' | grep -vE '^/mnt/' | grep -E '^/' | paste -sd:)" make -j"$(nproc)"
```

The last patch adds the RAM-test image, which is only meant for this
repository's [RAM boot guide](ram-boot.md); the others are the port itself.

## Output

In `bin/targets/econet/en751221/`:

| File | Use |
|---|---|
| `openwrt-econet-en751221-nokia_g140w-c-bringup-initramfs-kernel.bin` | RAM boot for testing (never writes slot A) |
| `openwrt-econet-en751221-nokia_g140w-c-initramfs-kernel.bin` | RAM boot, production image (for installing) |
| `openwrt-econet-en751221-nokia_g140w-c-squashfs-sysupgrade.bin` | Installing from the RAM-booted production image, and upgrading |

There is no factory image to flash from the bootloader or the stock web
interface: slot A also needs the bootloader's second stage, which only
the router has, so installing goes through `sysupgrade`. See
[install.md](install.md).

## Notes

- `config/diffconfig` builds both devices (production and RAM test) with
  LuCI, WPA3-capable Wi-Fi (`wpad-basic-mbedtls`), the GPIO tools, and
  `/dev/mem` for probing, plus WireGuard. There are no USB packages: the
  board has no USB connector.
- RAM-boot images must stay under 11,394,032 bytes. The bootloader loads
  them at `0x80020000` and the kernel decompresses to `0x81000000`, so a
  bigger loader would overlap it. The image recipe checks this and drops
  an oversized image with a warning.
- The target is marked `source-only` in OpenWrt, so the official servers
  publish no kernel modules for it. Extra kernel modules have to come from
  your own build; userspace packages from the `mips_24kc` feeds work.
