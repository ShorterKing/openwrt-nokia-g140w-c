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
| USB | xHCI controller (USB 3.0); the reference unit has no USB connector |

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

The on-die switch's end of the link has a TRGMII block of the same layout
(base `0x1fb58000`), also set up by the bootloader. The `econet-eth` driver
resets the on-die switch at probe, which clears part of it:

| Register | Bootloader | After the reset |
|---|---|---|
| `0x7804` (trap) | `0x1608f` (manual override) | `0x60ef` (straps) |
| `0x7830` (interface mode) | `0x1` (TRGMII) | `0x0` |
| `0x7a54 + n × 8` (TD_ODT0–4) | `0xaa` | `0xff` |
| `0x7a7c` | `0x77` | `0xff` |

Everything else in `0x7a00`–`0x7a78` survives the reset. On reset defaults
the link usually works, but once it started corrupting every frame from
the MT7530: port 5 of the on-die switch counted only CRC errors (MIB offset
`0x78`), no receive delay setting gave a clean frame, and only a power
cycle helped. The patch series makes `econet-eth` save these registers
before the reset and restore them; with the bootloader's values the link
carries full-speed traffic with no CRC errors. From the bootloader prompt,
`dump bfb5fa00 80` and `dump bfb5f800 40` show the tuned values.

Even with those values in place, the on-die switch's receiver sometimes
locks on badly when the MT7530 at the other end is reset and set up again,
which the `mt7530` driver does after `econet-eth` has probed. Every frame
from the MT7530 then fails the CRC check, while the other direction keeps
working. On the reference unit that happened on 4 of 52 driver loads. A
pulse on `RX_RST` (bit 31 of `0x7a00`, normally `0x20000`) makes the
receiver lock on again at once:

```
v=$(devmem 0x1fb5fa00 32); devmem 0x1fb5fa00 32 $((v | 0x80000000)); devmem 0x1fb5fa00 32 $v
```

The patch series makes `econet-eth` do this whenever the port is opened.
With it, 40 driver loads in a row came up with no CRC errors.

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

Those `lan1`–`lan4` counts come from the MT7530's own counters, and
`eth0`'s from the frame engine (GDM 1). They show what arrived at the
ports, not what Linux received. The first column of
`/proc/net/softnet_stat` (hex) counts the packets the network stack
actually processed.

## Ethernet DMA (QDMA)

The frame engine hands packets to the CPU through QDMA 0 at `0x1fb54000`
(QDMA 1, at `0x1fb55000`, is unused). Findings that mattered for speed:

- **RX 2-byte offset.** Bit 31 of `qdma_cfg` (`+0x04`) makes the DMA store
  each received frame 2 bytes past the buffer address, so that the IP header
  is 4-byte aligned after the 14-byte Ethernet header and the 4-byte MTK
  tag. The driver didn't set it, and the IP header sat 2 bytes off. MIPS
  has no unaligned loads, so every packet cost about 55 trapped and
  emulated accesses, and routing topped out at about 60 Mbit/s with the CPU
  fully busy. The count is in `/sys/kernel/debug/mips/unaligned_instructions`.
- **RX ring size.** With 128 descriptors, a TCP sender's slow-start burst
  overflowed the ring at the start of each download, and the GDM 1 drop
  counter (`eth0` RX drops) rose by several hundred. With 512 it stays at 0.
- **Interrupts.** `int_status` is at `+0x50` and `int_enable` at `+0x54`.
  Bit 0 is TX done and bit 1 RX done. Bit 2 (TX "no descriptor") is always
  set; the driver leaves it disabled.
- **Starting the rings.** The interrupt handler masks a ring's interrupt
  and leaves unmasking it to the NAPI poll. The driver used to turn DMA on
  before enabling NAPI. Frames that had queued up in the frame engine
  arrived at once, and on a busy network their interrupt often came in
  between: it was masked, but the poll never ran, so the router received
  nothing for the whole boot. `eth0` then counts about 580 received frames
  and nothing more (the ring plus what the frame engine buffered), its
  drop count climbs, and `int_enable` bit 1 is clear. Setting the bit by
  hand doesn't help, because the driver keeps its own copy of the mask.
  The patch series enables NAPI first and polls every ring once on start.

The driver's debugfs, `/sys/kernel/debug/econet_eth/qdma0/`, shows the
registers, interrupt counts per source and TX counts per channel and
queue.

## Interrupt controller

The interrupt controller is at `0x1fb40000`. The vendor kernel names its
registers:

| Offset | Register |
|---|---|
| `0x00` | Interrupt type |
| `0x04`, `0x50` | Mask, for IRQs 0–31 and 32–39 |
| `0x08`, `0x54` | Pending, for IRQs 0–31 and 32–39 |
| `0x0c` | Interrupt set |
| `0x10`–`0x2c`, `0x58`–`0x5c` | Priority, one byte per IRQ, IRQ 0 in the top byte of `0x10` |
| `0x30`–`0x4c`, `0x60`–`0x64` | VPE and shadow register set for each IRQ |

Linux runs the CPU in the MIPS interrupt compatibility mode, not the
vectored (EIC) mode the controller was designed for. In that mode the
controller doesn't give each source its own CPU interrupt line. It presents
the number of the pending interrupt plus one, in binary, on the CPU lines
IP2–IP7. The vendor kernel reads it as `(read_c0_cause() & ST0_IM) >> 10`.

Mainline Linux chained the controller to IP2 alone, which only catches
even-numbered IRQs. The odd ones, including the Ethernet DMA, waited for
the next even-numbered interrupt, normally the 100 Hz timer tick. A quiet
router then answered pings in 1–11 ms, cycling through that range as
the timer tick drifted. Three checks pinned it down:

- the pending bit of IRQ 21 was set in about 80% of reads taken from a
  userspace loop, which can't happen if the interrupt reaches the CPU;
- giving IRQ 21 a different priority changed nothing;
- a 1 ms sleep loop on the router, which adds timer interrupts, made the
  latency flat.

This port chains the controller to IP2–IP6. IP7 is left out because the
CPU can also signal its own timer on it; only IRQ 31, which is unused,
would need it. With the fix, the router answers pings in under 1 ms, and
pings through it to 1.1.1.1 dropped from 14 to 8 ms. The `econet-eth`
driver's comments describe the same symptom for transmit completions
("in practice the timer tick") and work around it with a 150 µs poll
timer.

| IRQ | Device | Number on the CPU lines |
|---|---|---|
| 0 | UART | 1: IP2 |
| 17 | xHCI | 18: IP3, IP6 |
| 21 | QDMA 0 (Ethernet) | 22: IP3, IP4, IP6 |
| 22 | QDMA 1 | 23: IP2, IP3, IP4, IP6 |
| 23 | PCIe RC0 (2.4 GHz Wi-Fi) | 24: IP5, IP6 |
| 24 | PCIe RC1 (5 GHz Wi-Fi) | 25: IP2, IP5, IP6 |
| 30 | Timer | 31: IP2–IP6 |

## Flash layout

The stock partitions are kept as they are. OpenWrt only uses slot A.

| Partition | Offset | Size | Contents |
|---|---|---|---|
| `bootloader` | `0x0` | 256 KiB | tcboot |
| `romfile` | `0x40000` | 256 KiB | Stock settings |
| `tclinux` (slot A) | `0x80000` | 34 MiB | **OpenWrt**: `tclinux_kernel` (4 MiB) + `ubi` (27 MiB), see below |
| `kernel_slave` + `rootfs_slave` (slot B) | `0x2280000` | 3 + 31 MiB | Stock firmware |
| `kernel_oflt` + `rootfs_oflt` | `0x4480000` | 3 + 12 MiB | Stock OFLT image |
| `config` | `0x5380000` | 8 MiB | Stock settings (UBIFS); holds the [Wi-Fi calibration](#wi-fi-calibration) |
| `log` | `0x5b80000` | 12 MiB | Stock logs |
| `extfs` | `0x6780000` | 6 MiB | Stock |
| `bosa` | `0x6d80000` | 256 KiB | Optical module calibration: never erase |
| `flag`, `flagback` | `0x6dc0000`, `0x6e00000` | 256 KiB each | Boot slot flags |
| `ri`, `riback` | `0x6e40000`, `0x6e80000` | 256 KiB each | Factory data (MAC address, serial) |

About 125 MiB between `0x6ec0000` and the bad-block pool is unused by stock.

**Slot A** as OpenWrt uses it (offsets from the start of the slot):

| Offset | Contents |
|---|---|
| `0x0` | TRX header, 256 bytes |
| `0x100` | OpenWrt kernel (LZMA), padded with `0xff` |
| `0x3e0000` | The bootloader's second stage ("bootram"), copied from the router |
| `0x6e0000` | UBI (`mtd` partition `ubi`): `rootfs` (squashfs), `rootfs_data` (settings) and `calibration` (Wi-Fi) |

The space between the bootram and UBI is unused, for the reason below.

**Bad blocks:** the flash uses the vendor's bad-block table (BMT) with a pool
of 163 blocks at the end of the chip. OpenWrt's `en75_bmt` driver understands
it. On partitions marked `econet,enable-remap`, it remaps a block when a
*read* hits an uncorrectable error or too many bit flips, which writes to
flash. That goes wrong in slot A. The old stock data there has such errors,
and the bootloader, which checks every remapped block at power-up, redoes
any remap whose replacement block has lost its mapping mark. UBI erases
blocks all the time, which removes that mark, so the bootloader then wipes
UBI data at every boot. Test images had remapped blocks 36–58 of the
reference unit like this. So slot A has no remapping, and UBI starts at
block 59 (`0x760000` on the chip), past them.

**Boot flags:** `flag` (copied to `flagback`) starts with six 32-bit
words: `commit`, `next_active` (0 = slot A), `image0_is_valid`,
`image1_is_valid`, `boot_fail_num` and the current image. At every
power-up the bootloader adds 1 to `boot_fail_num` and writes both
partitions. When the count reaches 9 it switches to the other slot. The
firmware is expected to clear the count once it has booted: stock does,
and OpenWrt runs `en75_chboot openwrt` at the end of each boot for the same
purpose (`/etc/init.d/bootcount`). `en75_chboot openwrt` and
`en75_chboot factory` switch the active slot. To read the flag:
`dd if=/dev/mtd13 bs=24 count=1 | hexdump -C` (check the partition number
in `/proc/mtd`).

## Wi-Fi calibration

Each radio needs its unit's RF calibration, a 512-byte EEPROM table. The
stock firmware keeps them as files in its settings, the UBIFS volume
`configfs` on `config`, mounted at `/configs`:

| Radio | File |
|---|---|
| MT7592, 2.4 GHz | `/configs/RT30xxEEPROM.bin` |
| MT7612E, 5 GHz | `/configs/bin_ac/RT30xxEEPROM.bin` |

There is no other copy in flash, and the chips' efuses hold almost nothing
(the driver shows them as `otp` in debugfs), so they can't stand in. The
mt76 driver can't read files: it takes calibration from the device tree,
a raw flash offset or an NVMEM cell.

So `sysupgrade` copies the two tables into a static UBI volume,
`calibration`, next to OpenWrt's root. Static volumes carry a CRC, and
upgrades leave the volume alone. The kernel's UBI NVMEM support
(`CONFIG_MTD_UBI_NVMEM`) turns it into NVMEM cells that the device tree
gives the radios:

| Offset | Contents |
|---|---|
| `0x0` | 2.4 GHz table, then 512 zero bytes (the mt7603 driver reads 1 KiB) |
| `0x400` | 5 GHz table |

To read them, `sysupgrade` attaches the stock UBI and mounts it read-only.
`config` is read-only in the device tree, so UBI can't write to it, and the
upgrade refuses to attach it otherwise. The volume has to exist before the
upgrade creates `rootfs_data` from all the free space. So if it's missing,
the upgrade removes `rootfs_data` and creates the volume first;
`rootfs_data` is recreated right after, and the settings to keep are saved
in RAM by then.

The RAM-test image doesn't attach slot A's UBI, so it carries the reference
unit's tables instead. Other units' radios work with them, but may perform
a little worse.

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

### Bootloader stages

tcboot runs in two stages. The first, in `bootloader`, reads `flag`, picks
the slot and loads the second stage, the **bootram** (the one with the
`bldr>` prompt), *from that slot*. In the slot's TRX it follows the kernel
and rootfs, at `0x100 + kernel length + rootfs length`, and the TRX header
holds its length:

| Header offset | Field |
|---|---|
| `0x00` | Magic `2RDH` |
| `0x08` | Total length, header included |
| `0x0c` | CRC-32/JAMCRC of everything after the header |
| `0x50` | Kernel length |
| `0x54` | Rootfs length |
| `0x94` | Bootram length (LZMA) |

A slot without a bootram crashes the bootloader, and the router only falls
back to the other slot after nine failed boots. The bootram is Nokia's
code, so OpenWrt's images don't include it. Instead, `sysupgrade` copies it
from the router (from slot A, or else from the stock firmware in slot B),
puts it at `0x3e0000` behind the kernel and updates the header to match.
The kernel length stays padded to `0x3dff00`: when booting slot B, the
bootloader reads the kernel length from slot A, so it must not be shorter
than the stock kernel.
