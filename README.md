# OpenWrt for the Nokia G-140W-C

An unofficial OpenWrt port for the **Nokia G-140W-C**, the fibre router (GPON ONT)
that Airtel supplies in India. The goal is a complete OpenWrt router: dual-band
Wi-Fi, four gigabit ports with a separate WAN port, LEDs and buttons. The fibre
side and the phone port are out of scope, because their drivers are closed.

> **Status: work in progress.** OpenWrt is installed and running on the
> reference unit, from flash, with settings kept and `sysupgrade` working. Wi-Fi, the LAN ports and the separate WAN port work, and routing
> keeps up with a 100 Mbit/s line. There are no downloadable images yet:
> build them as in [docs/building.md](docs/building.md).

## What works

| Feature | Status | Notes |
|---|---|---|
| Install to flash, `sysupgrade` | ✅ | OpenWrt in slot A, stock kept in slot B. See [docs/install.md](docs/install.md) |
| Boot from RAM for testing | ✅ | Linux 6.18, loaded over the serial console |
| LAN ports (LAN1–LAN3) | ✅ | Bridged, through the mainline `mt7530` DSA driver |
| Wi-Fi 2.4 GHz (MT7592) | ✅ | Access point with WPA2 |
| Wi-Fi 5 GHz (MT7612E) | ✅ | 802.11ac at 80 MHz; 433 Mbit/s link to a single-antenna phone |
| LEDs | ✅ | Power, PON, LOS, Internet, phone, USB, 2.4 GHz Wi-Fi |
| Buttons | ✅ | Reset and LED on/off |
| Separate WAN port (LAN4) | ✅ | Gets an address by DHCP and routes at the full speed of a 100 Mbit/s line. See [Performance](#performance) |
| USB | ⏳ | Not yet tested |
| Extras: USB storage and file sharing, WireGuard, USB tethering | ⏳ | In the build, not yet tested |
| Switching back to the stock firmware | ⏳ | `en75_chboot factory`; not yet tried |
| WLAN and WPS buttons | ❌ | Wired to a Wi-Fi chip, which the mt76 driver can't read |
| Fibre (GPON), phone port (VoIP) | ❌ | Closed vendor drivers |

The LEDs and buttons were found on hardware and are in the device tree. By
default, the PON LED shows the WAN cable link, the Internet LED shows WAN
link and traffic, the USB LED lights when a USB device is plugged in, and
the 2.4 GHz LED shows Wi-Fi activity. These defaults are in the build but not
yet checked on hardware.

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

## Hardware

| Part | Details |
|---|---|
| SoC | EcoNet EN7526G (EN751221 family), MIPS 34Kc, 2 threads, 900 MHz |
| RAM | 512 MiB DDR3 (Linux can use 440 MiB) |
| Flash | 256 MiB SPI-NAND (Winbond W25M02GV) |
| Ethernet | 4x gigabit on an MT7530 switch inside the SoC package; LAN4 is the WAN port |
| Wi-Fi | MT7592 (2.4 GHz, 802.11n 2x2) and MT7612E (5 GHz, 802.11ac 2x2) |
| USB | xHCI host |
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

## Known issues

- **Linux uses one of the CPU's two hardware threads.** The EN7526G's 34Kc
  core has two (VPEs), but the kernel's EN751221 platform code doesn't
  start the second one yet. Routing a 100 Mbit/s line keeps that one
  thread about 90% busy, so a faster line, or heavy Wi-Fi traffic at the
  same time, may hit the limit.
- The device tree embeds Wi-Fi calibration data copied from the reference
  unit. Other units will work, but their radios may perform a little worse
  until the calibration is read from flash (planned).
- Installing with `sysupgrade -n` from the RAM-booted image and switching
  back to stock haven't been tried yet. The first install on the reference
  unit was done by hand, with the steps `sysupgrade` now automates, and
  upgrades of the installed system work.
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

1. Test a fresh install with `sysupgrade -n` and switching back to stock.
   Check the WAN LEDs.
2. Test the extras on hardware: USB storage and file sharing, WireGuard, USB
   tethering.
3. Read the Wi-Fi calibration from flash instead of embedding it.
4. Lower the CPU load: interrupt coalescing in the Ethernet driver, and the
   CPU's second hardware thread.
5. Send the Ethernet driver and interrupt fixes upstream, publish images
   and submit the port to OpenWrt.

## Credits

- The OpenWrt developers behind the EcoNet target and the Nokia G-240G-E
  support that this port builds on.
- The mt76 Wi-Fi driver developers.

## Disclaimer

Installing third-party firmware can brick your device and voids any warranty.
The router may belong to your ISP, so check before modifying it. Do this at
your own risk.

## License

GPL-2.0, like OpenWrt. The device tree files carry their own SPDX headers.
