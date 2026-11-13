# Nokia G-140W-C hardware notes

Everything here was checked on a real unit unless marked otherwise. Sources:
the stock UART boot log, the stock firmware (kernel modules and tools), and
tests from the bootloader and from RAM-booted OpenWrt.

## Overview

| Part | Details |
|---|---|
| SoC | EcoNet EN7526G (EN751221 family), MIPS 34Kc, 2 threads (VPEs), 900 MHz, big-endian |
| RAM | 512 MiB DDR3-1066. Linux can use `0x0`–`0x1b800000` (440 MiB) |
| Flash | 256 MiB Winbond W25M02GV SPI-NAND (two dies), 128 KiB blocks, 2 KiB pages |
| Switch | MT7530 in the SoC package (LAN1–LAN4), linked over TRGMII to the SoC's on-die switch |
| Ethernet PHYs | MT7530 GPHYs (ID `0x03a29412`) at MDIO addresses 1–4 |
| PCIe RC0 | MT7592 (`14c3:7603`), 2.4 GHz, `mt7603` driver |
| PCIe RC1 (Gen2) | MT7612E (`14c3:7662`), 5 GHz, `mt76x2e` driver |
| USB | xHCI host |

## Switch

The Ethernet ports are on an **MT7530** in the SoC package (a multi-chip
module, CREV `0x75300001`), not on the SoC's own on-die switch. The two
switches are chained:

```
frame engine -- port 6 [on-die switch] port 5 == TRGMII == port 6 [MT7530] ports 1-4 -- LAN1-LAN4
```

| Item | Details |
|---|---|
| MT7530 | MDIO address 31. Ports 1–4 = LAN1–LAN4, with their PHYs at MDIO 1–4 (ID `0x03a29412`). Port 0 has a PHY (MDIO 0) but no socket |
| On-die switch | Registers at `0x1fb58000`, reset at SCU register `0x1fb00834` bit 23. Port 6 goes to the frame engine, port 5 to the MT7530. Its own PHYs (MDIO 8–11) have no sockets |
| MDIO master | The on-die switch's PHY indirect access controller at `0x1fb5f01c`, with the same command layout as the MT7531's |
| In OpenWrt | `econet-eth` keeps the on-die switch as a dumb switch and provides the MDIO bus. The mainline `mt7530` DSA driver runs the MT7530 as `lan1`–`lan4` |

The stock port map agrees: LAN1–LAN4 are ports 1–4, with no reversal.

### TRGMII link

The bootloader sets up the link between the two switches, and the MT7530
has no reset line, so Linux starts with that set-up in place. On the
reference unit (25 MHz crystal) it is:

| Register | Value | Meaning |
|---|---|---|
| Core `0x404` | `0x1000` | PLL: 200 MHz TRGMII clock |
| `0x7804` (MTRAP) | `0x17e8f` | |
| `0x7810` (IO_DRV_CR) | `0x10022` | |
| `0x7830` (P6ECR) | `0x1` | Port 6 in TRGMII mode |
| `0x7a00`, `0x7a04` (RCK_CTRL, RCK_RTT) | `0x20000`, `0xc0800000` | |
| `0x7a10 + n × 8` (RD0–RD4) | `0x0` | |
| `0x7a40` (TXCTRL) | `0x40020000` | |
| `0x7a54 + n × 8` (TD_ODT0–TD_ODT4) | `0xaa` | |
| `0x7a78` (TCK_CTRL) | `0x55` | |
| `0x3600` (port 6 PMCR) | `0x9e30b` | Forced link, 1 Gbit/s full duplex |

The mainline driver resets the switch at probe (`SYS_CTRL`, `0x7000` = 7).
On this unit, that reset overwrites MTRAP, IO_DRV_CR, P6ECR, the TD_ODT
registers and port 6's PMCR, but leaves the PLL alone, and the link stays
down until they are set again. The driver would also program the PLL and pad
tuning for an MT7621 or a standalone MT7530. With the
`econet,en751221-mt7530` compatible that this port adds, the driver saves
the trap and TRGMII registers before the reset and restores them, keeps the
PLL, and sets port 6's PMCR itself as usual.

### Counters

The on-die switch's counters are memory-mapped, which helps when checking
the link between the switches. Each port's MIB block is at
`0x1fb5c000 + port × 0x100`. Counter offsets: TX drop `0x00`, TX unicast
`0x08`, multicast `0x0c`, broadcast `0x10`; RX drop `0x60`, filtered `0x64`,
unicast `0x68`, multicast `0x6c`, broadcast `0x70`, alignment errors `0x74`,
CRC errors `0x78`. With `/dev/mem` in the build, this prints the received
unicast, multicast and broadcast counts for port 5 (from the MT7530) and
port 6 (from the CPU):

```
for p in 5 6; do printf "p$p rx:"; for o in 104 108 112; do printf " %d" $(devmem $((0x1fb5c000+p*0x100+o)) 32); done; echo; done
```

The MT7530's own registers are only reachable over MDIO. For the LAN
ports, `ip -s link show lan1` (and so on) gives the counts.

## Flash layout

The stock partitions are kept as they are. OpenWrt only uses slot A.

| Partition | Offset | Size | Contents |
|---|---|---|---|
| `bootloader` | `0x0` | 256 KiB | tcboot |
| `romfile` | `0x40000` | 256 KiB | Stock settings |
| `tclinux` (slot A) | `0x80000` | 34 MiB | **OpenWrt**: `tclinux_kernel` (4 MiB TRX) + `ubi` (30 MiB) |
| `kernel_slave` + `rootfs_slave` (slot B) | `0x2280000` | 3 + 31 MiB | Stock firmware |
| `kernel_oflt` + `rootfs_oflt` | `0x4480000` | 3 + 12 MiB | Stock OFLT image |
| `config` | `0x5380000` | 8 MiB | Stock settings (UBIFS); holds the Wi-Fi calibration |
| `log` | `0x5b80000` | 12 MiB | Stock logs |
| `extfs` | `0x6780000` | 6 MiB | Stock |
| `bosa` | `0x6d80000` | 256 KiB | Optical module calibration: never erase |
| `flag`, `flagback` | `0x6dc0000`, `0x6e00000` | 256 KiB each | Boot slot flags |
| `ri`, `riback` | `0x6e40000`, `0x6e80000` | 256 KiB each | Factory data (MAC address, serial) |

About 125 MiB between `0x6ec0000` and the bad-block pool is unused by stock.

**Bad blocks:** the flash uses the vendor's bad-block table (BMT) with a pool
of 163 blocks at the end of the chip. OpenWrt's `en75_bmt` driver understands
it. Be aware that it remaps a block when a *read* hits an uncorrectable error
or too many bit flips, so reading worn blocks can write to flash.

**Boot flags:** `flag` holds `commit`, `next_active`, `image0_is_valid`,
`image1_is_valid` and `boot_fail_num`. `en75_chboot openwrt` and
`en75_chboot factory` switch the active slot.

## MAC addresses

The base MAC is the 6 bytes at offset `0x3e` of `ri` (the value `ritool`
reports as `MACAddress`). The stock firmware uses:

| Interface | MAC |
|---|---|
| LAN | base |
| WAN | base + 1 |
| Wi-Fi 2.4 GHz | base + 9 |
| Wi-Fi 5 GHz | base + 0xd |

## GPIOs

All on the first GPIO bank (`gpio0`, data register `0x1fbf0204`). Every LED and
button is active-low.

| GPIO | Function | PCB |
|---|---|---|
| 0 | Reset button | |
| 2 | PON LED (green) | D501 |
| 5 | LED on/off button | |
| 6 | Power LED (green) | D508 |
| 11 | USB LED | D514 |
| 24 | LOS LED (red) | D504 |
| 25 | Phone LED | D506 |
| 27 | **PON laser TX disable: never drive it** | |
| 28 | Internet LED | D503 |

The stock firmware also configures GPIOs 3, 8, 16 and 26 on this board; their
functions are unknown.

On the Wi-Fi chips:

- **2.4 GHz WLAN LED (D505):** LED pin 0 of the MT7592, active-low. OpenWrt
  drives it through mt76 (`led-active-low` in the device tree).
- **WLAN and WPS buttons:** on Wi-Fi chip GPIOs, read in stock through the
  vendor Wi-Fi driver. mt76 can't read them, so they don't work in OpenWrt.
- **5 GHz:** the MT7612E's LED pin 0 lights nothing.

The LAN port LEDs are driven by the switch PHYs and light with link.

Not identified yet: D510, D511, D512 and D502 (probably LAN1–LAN4), D513 and
D515. Their order along the board edge, starting at the power LED, is: D508
D501 D504 D510 D511 D512 D502 D506 D513 D505 D515 D514 D503.

### How the GPIOs were found

1. Flipping bits of the GPIO data register from the bootloader prompt
   (`memrl bfbf0204`, `memwl bfbf0204 <value>`).
2. Decoding the stock firmware's tables for this board, which it calls
   `G140WC` (board type `0x2c`):
   - `led_drv.ko`: `led_g140wc`, which maps LED IDs to abstract pins.
   - `gpio_mgr.ko`: the pin map built for board type `0x2c`, which maps
     abstract pins to SoC GPIOs, switch pins or Wi-Fi chip pins.
   - `button_driver.ko`: `g140wc_btntbl`.
   - `evtmgr` and `libsal_led.so` give the meaning of the button and LED
     IDs. For example, the USB notifier uses LED ID 15 and the SIP app uses
     LED ID 9.
3. Lighting each LED from OpenWrt and reading its D-number on the PCB, then
   polling the buttons with `gpioget` while pressing them.

## Serial console and bootloader

- 115200 baud, 8N1, 3.3 V levels.
- tcboot shows a 1-second window: press any key to get the `bldr>` prompt.
- Useful commands:
  - `xmdm <addr> <len>`: receive a file over XMODEM-CRC (lengths in hex).
  - `jump <addr>`: run code.
  - `memrl <addr>` / `memwl <addr> <value>`: read or write a 32-bit word.
- `httpd` and `flash` write to flash; avoid them unless you mean it.
- There is no TFTP.
