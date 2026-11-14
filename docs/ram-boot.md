# Boot OpenWrt from RAM

This loads an OpenWrt kernel into RAM through the bootloader and runs it. It
doesn't install anything: power-cycle the router and it boots stock again.

> **Use the RAM-test image for testing.** The production image attaches
> and mounts the UBI area in slot A if it finds one, so on a router with
> OpenWrt installed, a RAM boot would use (and write to) its settings. The
> RAM-test image leaves slot A alone. Use the production image only to
> install, as in [install.md](install.md).

## You need

- A USB-to-serial adapter with **3.3 V** levels, wired to the board's UART
  (GND, TX, RX; don't connect VCC).
- A terminal that can send XMODEM files: Tera Term on Windows, or
  `minicom` with `lrzsz` on Linux.
- An image ending in `-initramfs-kernel.bin` (see [building.md](building.md)):
  - `...-nokia_g140w-c-bringup-initramfs-kernel.bin`: **RAM-test image**.
    It's the same as production (switch under DSA, LAN4 = WAN), but it
    never writes slot A and checks the bad-block pool size. Use this one.
  - `...-nokia_g140w-c-initramfs-kernel.bin`: production image, used for
    [installing](install.md).

## Steps

1. Open the serial port at **115200 baud, 8N1, no flow control**.
2. Power on the router and press any key within **1 second**. You get the
   `bldr>` prompt.
3. Work out the file size in hex:
   - Linux: `printf '%x\n' $(stat -c %s file.bin)`
   - PowerShell: `'{0:x}' -f (Get-Item file.bin).Length`
4. Tell the bootloader to receive the file at `0x80020000`. For example, for
   a file of 7571225 bytes (`0x738719`):

   ```
   xmdm 80020000 738719
   ```

5. Start sending **right away**: the bootloader gives up after about
   30 seconds and prints `received len=fffffffd`. In Tera Term, use
   **File → Transfer → XMODEM → Send**, pick the file and choose **CRC**. The
   transfer runs at about 10 KiB/s, so a 10.6 MiB image takes about
   18 minutes.
6. When it finishes, check the image landed and start it:

   ```
   memrl 80020000
   jump 80020000
   ```

   `memrl` should print `808025`, the first instruction of the kernel loader.

OpenWrt boots on the serial console. Press Enter to get a root shell.

## After booting

- Connect a computer to **LAN1, LAN2 or LAN3**. **LAN4 is the WAN port**:
  plug it into your main router or modem, and OpenWrt gets an address there
  by DHCP.
- LuCI is at `http://192.168.1.1`. Log in as `root` with an empty password.
- If your main network also uses `192.168.1.x`, move the router's LAN first,
  from the serial console:

  ```
  uci set network.lan.ipaddr='192.168.50.1'; uci set network.lan.netmask='255.255.255.0'; uci commit network; service network restart
  ```

  Set the netmask together with the address: without it the LAN ends up as a
  /32 and DHCP refuses to start. LuCI then moves to `http://192.168.50.1`.
- Wi-Fi is off by default. To turn it on from the serial console, with your
  own country code and password:

  ```
  for r in 0 1; do uci set wireless.radio$r.country=IN; uci set wireless.default_radio$r.encryption=psk2; uci set wireless.default_radio$r.key='your-password'; uci set wireless.default_radio$r.disabled=0; done; uci commit wireless; wifi reload
  ```

  Both bands then broadcast `OpenWrt`.
- When measuring speed from a PC that is also on another network over
  Wi-Fi, turn its Wi-Fi off first. Windows can send IPv6 traffic over the
  Wi-Fi and bypass the router.
- Settings are lost on power-off; this is a RAM session only. To keep
  OpenWrt, [install it](install.md).
