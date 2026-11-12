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
| Switch | MT7530-class at `0x1fb58000`; CPU port 6, user ports 1–4 |
| Ethernet PHYs | MT7530 GPHYs (ID `0x03a29412`) at MDIO addresses 1–4 |
| PCIe RC0 | MT7592 (`14c3:7603`), 2.4 GHz, `mt7603` driver |
| PCIe RC1 (Gen2) | MT7612E (`14c3:7662`), 5 GHz, `mt76x2e` driver |
| USB | xHCI host |

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
