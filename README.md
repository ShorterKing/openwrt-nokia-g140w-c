# OpenWrt for the Nokia G-140W-C

An unofficial OpenWrt port for the **Nokia G-140W-C**, the fibre router (GPON ONT)
that Airtel supplies in India. The goal is a complete OpenWrt router: dual-band
Wi-Fi, four gigabit ports with a separate WAN port, LEDs and buttons. The fibre
side and the phone port are out of scope, because their drivers are closed.

> **Status: work in progress.** OpenWrt boots from RAM on real hardware.
> Wi-Fi, the LAN ports and the separate WAN port work, and routing keeps up
> with a 100 Mbit/s line. Nothing here installs to flash yet, and there are
> no downloadable images yet.

## What works

| Feature | Status | Notes |
|---|---|---|
| Boot (loaded into RAM by the bootloader) | ✅ | Linux 6.18 |
| LAN ports (LAN1–LAN3) | ✅ | Bridged, through the mainline `mt7530` DSA driver |
| Wi-Fi 2.4 GHz (MT7592) | ✅ | Access point with WPA2 |
| Wi-Fi 5 GHz (MT7612E) | ✅ | 802.11ac at 80 MHz; 433 Mbit/s link to a single-antenna phone |
| LEDs | ✅ | Power, PON, LOS, Internet, phone, USB, 2.4 GHz Wi-Fi |
| Buttons | ✅ | Reset and LED on/off |
| Separate WAN port (LAN4) | ✅ | Gets an address by DHCP and routes at the full speed of a 100 Mbit/s line. See [Performance](#performance) |
| USB | ⏳ | Not yet tested |
| Extras: USB storage and file sharing, WireGuard, USB tethering | ⏳ | In the build, not yet tested |
| Install to flash, sysupgrade | ⏳ | Planned: OpenWrt in slot A, stock kept in slot B |
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
| Download through the router | 108–111 Mbit/s |
| Upload through the router | About 90 Mbit/s |
| Router CPU during a full-speed download | 60–80% of one thread |
| Ping to 1.1.1.1, idle and during a download | 14 ms average in both cases |

These were measured with the fixed Ethernet driver loaded into a running
RAM-booted build. Getting there took several fixes, all in the patch
series:

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
- Slot A holds a 4 MiB kernel in the TRX format the bootloader expects,
  followed by UBI with the root filesystem and your settings. That gives
  normal `sysupgrade` and LuCI upgrades.
- The bootloader always decompresses kernels to `0x80002000`, so the kernel is
  prefixed with a jump to OpenWrt's loader at `0x80020000`.
- The board is a close sibling of the Nokia G-240G-E, which OpenWrt already
  supports. It has the same flash layout and bootloader.

## Try it

- **Boot from RAM:** [docs/ram-boot.md](docs/ram-boot.md). This uses a serial
  adapter and the bootloader's XMODEM command, and power-cycling returns to
  stock.
- **Build it yourself:** [docs/building.md](docs/building.md).

## Known issues

- **Latency when the network is quiet (fix not yet re-tested).** The
  router answered pings in 1–11 ms, because its interrupt controller only
  passed the Ethernet interrupt on at the next timer tick. Packets through
  the router waited too, up to 10 ms each way. Under load the driver
  polls, so throughput wasn't affected. The fix is in the build; see
  [the hardware notes](docs/hardware.md#interrupt-controller). It also
  applies to USB and the 2.4 GHz Wi-Fi, whose interrupts have odd numbers
  too.
- **Linux uses one of the CPU's two hardware threads.** The EN7526G's 34Kc
  core has two (VPEs), but the kernel's EN751221 platform code doesn't
  start the second one yet.
- The device tree embeds Wi-Fi calibration data copied from the reference
  unit. Other units will work, but their radios may perform a little worse
  until the calibration is read from flash (planned).
- When the production image is RAM-booted, OpenWrt reads slot A while
  looking for UBI. On this NAND driver, reading a worn block makes it remap
  the block, which writes to flash. This happened once on the reference
  unit, inside slot A. The RAM-test image (`-bringup-` in the file name)
  avoids this: it turns remapping off for slot A and doesn't look for UBI
  there. Use it for testing. This was checked on hardware: over several RAM
  boots, the bad-block table kept the one remap from that first incident and
  recorded no new ones.

## Repository layout

| Path | Contents |
|---|---|
| `patches/` | The OpenWrt changes as a patch series (apply with `git am`) |
| `config/diffconfig` | Build configuration |
| `tools/g140wc-probe` | Bring-up helper for RAM-booted images |
| `docs/` | Hardware notes, RAM boot guide, build guide |

## Roadmap

1. Re-test the full image with the interrupt fix, and check the WAN LEDs.
2. Install to flash; test sysupgrade and switching back to stock.
3. Test the extras on hardware: USB storage and file sharing, WireGuard, USB
   tethering.
4. Read the Wi-Fi calibration from flash instead of embedding it.
5. Start the CPU's second hardware thread.
6. Send the Ethernet driver and interrupt fixes upstream, publish images
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
