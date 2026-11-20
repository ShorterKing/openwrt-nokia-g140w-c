# OpenWrt for the Nokia G-140W-C

By **Akshat Singh** ([ShorterKing](https://github.com/ShorterKing)).

An unofficial OpenWrt port for the **Nokia G-140W-C**, a fibre router (GPON ONT)
that ISPs in India, such as ACT Fibernet and Tata Play Fiber, supply. The
goal is a complete OpenWrt router: dual-band Wi-Fi, four gigabit ports with a
separate WAN port, LEDs and buttons. The fibre side and the phone port are out
of scope, because their drivers are closed.

> **Status: work in progress.** OpenWrt is installed and running on the
> reference unit, from flash, with settings kept and `sysupgrade` working.
> Wi-Fi, the LAN ports and the separate WAN port work, and routing keeps up
> with a 100 Mbit/s line. A second unit has also booted OpenWrt from RAM.
> The fixes are on their way into OpenWrt and Linux: see
> [Upstream status](#upstream-status). There are no downloadable images yet:
> build them as in [docs/building.md](docs/building.md).

## What works

| Feature | Status | Notes |
|---|---|---|
| Install to flash, `sysupgrade` | ✅ | OpenWrt in slot A, stock kept in slot B. See [docs/install.md](docs/install.md) |
| Boot from RAM for testing | ✅ | Linux 6.18, loaded over the serial console |
| LAN ports (LAN1–LAN3) | ✅ | Bridged, through the mainline `mt7530` DSA driver |
| Wi-Fi 2.4 GHz (MT7592) | ✅ | Access point with WPA2 |
| Wi-Fi 5 GHz (MT7612E) | ✅ | 802.11ac at 80 MHz; 433 Mbit/s link to a single-antenna phone |
| Wi-Fi calibration from flash | ✅ | Copied from the stock settings at install. See [the hardware notes](docs/hardware.md#wi-fi-calibration) |
| LEDs | ✅ | Power, PON, LOS, Internet, phone, USB, 2.4 GHz Wi-Fi |
| Buttons | ✅ | Reset and LED on/off |
| Separate WAN port (LAN4) | ✅ | Gets an address by DHCP and routes at the full speed of a 100 Mbit/s line. See [Performance](#performance) |
| USB | ✅ | Both ports (USB1, USB2), USB 2.0 speed; the USB LED follows them |
| USB storage | ✅ | FAT32 sticks on both ports, read at 15–23 MB/s |
| USB tethering | ✅ | Android (RNDIS): the router used the phone's connection. iPhone not tested |
| Windows file sharing (USB disk) | ⏳ | In the build (`ksmbd`), not yet tested. Testers welcome: please report how it works for you |
| WireGuard | ⏳ | In the build; the interface starts, a tunnel to a real peer is not yet tested |
| Switching back to the stock firmware | ✅ | `en75_chboot factory`, or automatically after nine failed boots |
| WLAN and WPS buttons | ❌ | Wired to a Wi-Fi chip, which the mt76 driver can't read |
| Fibre (GPON), phone port (VoIP) | ❌ | Closed vendor drivers |

The LEDs and buttons were found on hardware and are in the device tree. By
default, the PON LED shows the WAN cable link, the Internet LED shows WAN
link and traffic, the USB LED lights when a USB device is plugged in, and
the 2.4 GHz LED shows Wi-Fi activity. The WAN and USB LEDs were checked on
hardware.

## Performance

Measured on the reference unit as the router of a 100 Mbit/s fibre line,
with LAN4 connected to the ISP's router and a PC on LAN2:

| Test | Result |
|---|---|
| Download through the router | 108–113 Mbit/s |
| Upload through the router | 93–103 Mbit/s |
| Ping to the router | Under 1 ms |
| Ping to 1.1.1.1, idle and during a download | 8 ms average in both cases |
| Router CPU during a full-speed download | About 90% of one thread |

These were measured with the RAM-test image built from the patch series,
and downloads ran at the same speed (109 Mbit/s) on the installed system.
Getting there took several fixes, all in the patch series:

- **13 Mbit/s** with the Ethernet driver version that OpenWrt ships.
  Transmit completions arrived late, so the kernel held each transmit queue
  to about two packets.
- **59 Mbit/s** with the driver's 2026-10-14 version, which fixes that. The
  CPU was now fully busy.
- **101 Mbit/s** after aligning the IP header of received packets. Each
  packet had cost about 55 unaligned memory accesses, which MIPS traps and
  emulates.
- **108–111 Mbit/s** with 512 receive descriptors instead of 128, which
  ends the drops at the start of each download, and with GRO off on the
  switch ports. The driver has no checksum offload, so GRO cost more than
  it saved.
- **8 ms instead of 14 ms** to 1.1.1.1, and under 1 ms instead of 1–11 ms
  to the router itself, after fixing how Linux connects the interrupt
  controller. Received packets used to wait for the next timer tick; see
  [the hardware notes](docs/hardware.md#interrupt-controller). Handling
  each packet as it arrives costs more CPU at full speed than the old
  accidental batching: about 90% instead of 77%.

Two more driver fixes make the network come up on every boot. Before them,
some boots received nothing at all: the driver could lose its first
receive interrupt while starting, or the link between the LAN switch chip
and the CPU could come up corrupting every frame. Both are described in
[the hardware notes](docs/hardware.md).

The six Ethernet driver fixes are also kept as commits on top of the
driver's own repository, in
[ShorterKing/econet_eth](https://github.com/ShorterKing/econet_eth).

## Hardware

| Part | Details |
|---|---|
| SoC | EcoNet EN7526G (EN751221 family), MIPS 34Kc, 2 threads, 900 MHz |
| RAM | 512 MiB DDR3 (Linux can use 440 MiB) |
| Flash | 256 MiB SPI-NAND: Winbond W25M02GV on the reference unit, Toshiba TC58CVG1S3H on a second one |
| Ethernet | 4x gigabit on an MT7530 switch inside the SoC package; LAN4 is the WAN port |
| Wi-Fi | MT7592 (2.4 GHz, 802.11n 2x2) and MT7612E (5 GHz, 802.11ac 2x2) |
| USB | 2 ports (xHCI) |
| Serial | 115200 8N1, 3.3 V |
| Bootloader | tcboot ("free bootbase"), 1-second key window |

Some G-140W-C units are reported with 16 MiB NOR flash and 128 MiB RAM. This
port is for the 256 MiB NAND / 512 MiB RAM version, so check your boot log.

The full GPIO, LED, button, MAC and flash maps are in
[docs/hardware.md](docs/hardware.md).

## How it fits on the router

- The stock flash has two firmware slots. OpenWrt goes into **slot A** and the
  stock firmware stays in **slot B**, so you can switch back with
  `en75_chboot factory`.
- Slot A holds the kernel in the TRX format the bootloader expects,
  followed by UBI with the root filesystem and your settings. That gives
  normal `sysupgrade` and LuCI upgrades.
- The bootloader loads its own second stage from the slot it boots, so slot
  A needs a copy of it behind the kernel. It's Nokia's code and isn't in
  the images: `sysupgrade` copies it from the router each time.
- The bootloader switches slots after nine boots that didn't finish.
  OpenWrt resets that count after every boot, as stock does.
- The bootloader always decompresses kernels to `0x80002000`, so the kernel is
  prefixed with a jump to OpenWrt's loader at `0x80020000`.
- The board is a close sibling of the Nokia G-240G-E, which OpenWrt already
  supports. It has the same flash layout and bootloader.

## Try it

- **Boot from RAM:** [docs/ram-boot.md](docs/ram-boot.md). This uses a serial
  adapter and the bootloader's XMODEM command, and power-cycling returns to
  stock.
- **Install:** [docs/install.md](docs/install.md). Boot from RAM, back up,
  then run `sysupgrade`.
- **Build it yourself:** [docs/building.md](docs/building.md).

## Upstream status

The goal is to get this port into OpenWrt itself. Each fix goes to the
project that owns the code:

| Change | Submitted to | Status |
|---|---|---|
| Interrupt controller: deliver odd-numbered interrupts | OpenWrt ([PR #26744](https://github.com/openwrt/openwrt/pull/26744)) and the Linux kernel ([patch series](https://lore.kernel.org/all/20261115-en751221-intc-all-lines-v1-0-dc03a40f47f0@gmail.com/)) | In review |
| Ethernet driver: don't lose the first receive interrupt | The driver's repository ([econet_eth PR #19](https://github.com/cjdelisle/econet_eth/pull/19)) | In review |
| Ethernet driver: aligned receive buffers, more descriptors | Tested on this router for another contributor's [econet_eth PR #17](https://github.com/cjdelisle/econet_eth/pull/17) | In review |
| Device support for the G-140W-C | OpenWrt, once the switch support below is in | Planned |

This port drives the LAN switch with its own changes to the Ethernet driver
and the mainline `mt7530` driver. Caleb James DeLisle, the author of the
Ethernet driver, has DSA switch support for EcoNet in progress
([cjdelisle/openwrt, branch `econet-dsa-switch-sept21-2026`](https://github.com/cjdelisle/openwrt/tree/econet-dsa-switch-sept21-2026)).
It ran on this router at full line speed in a RAM test, so the OpenWrt
submission will build on it, and the switch patches here will be replaced.

## Known issues

- **Linux uses one of the CPU's two hardware threads.** The EN7526G's 34Kc
  core has two (VPEs), but the kernel's EN751221 platform code doesn't
  start the second one yet. Routing a 100 Mbit/s line keeps that one
  thread about 90% busy, so a faster line, or heavy Wi-Fi traffic at the
  same time, may hit the limit.
- The RAM-test image carries the reference unit's Wi-Fi calibration, since
  it doesn't touch slot A. Other units' radios work with it, but may
  perform a little worse. Installed systems use their own unit's
  calibration.
- Not tried yet: installing on a router that has never had OpenWrt.
  Installing from the RAM-booted image onto a router whose slot A already
  held OpenWrt works, as do upgrades, switching to stock with
  `en75_chboot factory` and back, and the bootloader's automatic fallback
  to stock.
- **Some ISP units have a locked bootloader.** On a unit from Tata Play
  Fiber, the bootloader's second stage is a different build from the
  reference unit's: a key press at boot asks for a login, and the RAM-boot
  steps in [docs/ram-boot.md](docs/ram-boot.md) don't work unchanged. Its
  stock firmware also sits in slot A, with slot B marked invalid: the
  reverse of the reference unit. The install steps here assume an unlocked
  bootloader and stock in slot B. I found a way to boot OpenWrt on such a
  unit anyway; it's documented in
  [ShorterKing/Hardware-Hacking](https://github.com/ShorterKing/Hardware-Hacking).
- 2.9 MiB of slot A is left unused. Test images had made the bootloader
  wipe some blocks there at every boot, so UBI starts after them; see
  [the flash notes](docs/hardware.md#flash-layout).

## Repository layout

| Path | Contents |
|---|---|
| `patches/` | The OpenWrt changes as a patch series (apply with `git am`) |
| `config/diffconfig` | Build configuration |
| `tools/g140wc-probe` | Bring-up helper for RAM-booted images |
| `docs/` | Hardware notes, RAM boot guide, build guide |

## Roadmap

1. Second unit: OpenWrt has booted on a Tata Play Fiber unit that never
   had it, with a different flash chip (Toshiba) and a locked bootloader.
   See [ShorterKing/hardware-testing](https://github.com/ShorterKing/Hardware-Hacking).
2. Windows file sharing from a USB disk, and WireGuard with a real peer:
   testers welcome.
3. Lower the CPU load: interrupt coalescing in the Ethernet driver, and the
   CPU's second hardware thread.
4. Get the fixes and the device support into OpenWrt (see
   [Upstream status](#upstream-status)), then publish images.

## Credits

- The OpenWrt developers behind the EcoNet target and the Nokia G-240G-E
  support that this port builds on.
- Caleb James DeLisle, for the econet_eth Ethernet driver and the EcoNet DSA
  switch work.
- Emre Yavuzalp, whose transmit fixes in the Ethernet driver took downloads
  from 13 to 59 Mbit/s.
- The mt76 Wi-Fi driver developers.

## Disclaimer

Installing third-party firmware can brick your device and voids any warranty.
The router may belong to your ISP, so check before modifying it. Do this at
your own risk.

## License

The changes in this repository are © 2026 Akshat Singh, licensed under
GPL-2.0 like OpenWrt. The device tree files carry their own SPDX headers.
