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
git checkout 3d5a743f1f          # the commit the patches were made against
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

## Output

In `bin/targets/econet/en751221/`:

| File | Use |
|---|---|
| `openwrt-econet-en751221-nokia_g140w-c-initramfs-kernel.bin` | RAM boot, production image |
| `openwrt-econet-en751221-nokia_g140w-c-bringup-initramfs-kernel.bin` | RAM boot, bring-up image |
| `openwrt-econet-en751221-nokia_g140w-c-squashfs-tclinux.trx` | Full slot A image (kernel + UBI), for the first install |
| `openwrt-econet-en751221-nokia_g140w-c-squashfs-sysupgrade.bin` | Upgrades from a running OpenWrt |

The install procedure is not final yet. Until it is, only use the
`initramfs-kernel.bin` images (see [ram-boot.md](ram-boot.md)).

## Notes

- `config/diffconfig` builds both devices (production and bring-up) with
  LuCI, the GPIO tools and `/dev/mem` enabled for probing.
- The target is marked `source-only` in OpenWrt, so the official servers
  publish no kernel modules for it. Extra kernel modules have to come from
  your own build; userspace packages from the `mips_24kc` feeds work.
