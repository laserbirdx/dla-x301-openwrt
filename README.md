![OpenWrt on D-Link DLA-X301](assets/banner.jpg)

# OpenWrt for D-Link DLA-X301

OpenWrt port for the D-Link DLA-X301 access point (Qualcomm IPQ5018).
Both radios, LuCI, LAN/WAN, correct MAC addresses and persistent installation to
NAND via sysupgrade all work.

Prebuilt images are on the [Releases](../../releases/latest) page.

> **Unofficial.** This board is not in upstream OpenWrt yet. Flashing replaces the
> vendor firmware. Take a full flash backup first (see below). You do this at your
> own risk.

## Hardware

| Component | Details |
|---|---|
| SoC | Qualcomm IPQ5018, dual Cortex-A53 @ 800 MHz (AP-MP03.3 reference design) |
| RAM | 512 MiB |
| Flash | 128 MiB SPI-NAND, ESMT F50D1G41LB (GigaDevice-branded), A/B layout |
| Wi-Fi | IPQ5018 2.4 GHz + QCN6122 5 GHz (802.11ax) |
| Ethernet | 2x GbE. Port A: `gmac0` -> internal GE PHY. Port B: `gmac1` -> SGMII -> QCA8337 |
| LED | one green LED, driven by GPIO 22 / 25 / 26 |
| Buttons | reset (GPIO 32), WPS (GPIO 38) |
| Serial | 115200 8N1, `ttyMSM0` |
| ODM | Changjin |
| Stock firmware | QSDK, Linux 4.4.60, U-Boot 2016.01 |

Calibration data lives in the `0:ART` partition (IPQ5018 at `0x1000`, QCN6122 at
`0x26800`). MAC addresses are not in ART: they are read from the U-Boot environment
(`0:APPSBLENV`, `ethaddr` / `eth1addr`).

## Default configuration of the release image

| Setting | Value |
|---|---|
| Port A | LAN, `192.168.1.1`, DHCP server |
| Port B | WAN, DHCP client |
| Login | `root`, **no password**. Set one on first login (`passwd` or LuCI) |
| Wi-Fi | **disabled** by default, country set to `TR`. Enable it in LuCI after setting a password |
| Web UI | LuCI |

## Serial console (UART)

The stock firmware and U-Boot both give a console on the on-board serial header.

- 115200 8N1, 3.3 V TTL. **Do not** use an RS-232 level adapter.
- Connect GND, TX and RX only (TX to RX, RX to TX). Do not connect VCC.
- Stock firmware drops you into a passwordless root shell after boot.
- U-Boot: press any key during the 3 s autoboot countdown. The prompt is `IPQ5018#`.

## Installation

You need: a USB-TTL serial adapter, an Ethernet cable, and a TFTP server on your PC.

U-Boot can only use **Port A** (`eth0`) for networking. Plug the cable into Port A.

### 1. Boot the initramfs image from RAM

Put `openwrt-qualcommax-ipq50xx-dlink_dla-x301-initramfs-uImage.itb` in the TFTP
server root. Give the PC a static IP, e.g. `192.168.1.2/24`.

Interrupt autoboot and run:

```
setenv ipaddr 192.168.1.10
setenv serverip 192.168.1.2
tftpboot 0x44000000 openwrt-qualcommax-ipq50xx-dlink_dla-x301-initramfs-uImage.itb
bootm
```

Do **not** run `saveenv`. Nothing is written to flash at this step: if anything goes
wrong, power-cycle and the stock firmware boots again.

### 2. Back up the flash

OpenWrt is now running from RAM at `192.168.1.1`. Before writing anything, copy every
MTD partition to your PC. `0:ART` (Wi-Fi calibration) and `0:APPSBLENV` (MAC
addresses) are unique to your unit and cannot be recovered from anywhere else.

```sh
# on the PC
for i in $(seq 0 16); do
  ssh root@192.168.1.1 "cat /dev/mtd$i" > mtd$i.bin
done
ssh root@192.168.1.1 cat /proc/mtd > proc_mtd.txt
```

### 3. Install to flash

Copy the sysupgrade image to the device and flash it:

```sh
scp -O openwrt-qualcommax-ipq50xx-dlink_dla-x301-squashfs-sysupgrade.bin root@192.168.1.1:/tmp/
ssh root@192.168.1.1 sysupgrade -n /tmp/openwrt-qualcommax-ipq50xx-dlink_dla-x301-squashfs-sysupgrade.bin
```

The upgrade removes the OEM UBI volumes (`ubi_rootfs`, `wifi_fw`, `bt_fw`) and installs
OpenWrt into the `rootfs` UBI partition. The device reboots into OpenWrt from flash.

Later upgrades are normal sysupgrades from LuCI or the command line.

### Going back to stock

Restore the `rootfs` partition from your backup in U-Boot (TFTP the backup into RAM, then
`nand erase` / `nand write` the partition). Without a backup there is no easy way back.

## Building from source

The port targets OpenWrt `main` (snapshot). The QCN6122 support it relies on is not in
the 24.10 or 25.12 branches. The release was built from `main` at `r36880-bb6df54d9d`,
kernel 6.18.55.

1. Copy [`ipq5018-dla-x301.dts`](ipq5018-dla-x301.dts) to `target/linux/qualcommax/dts/`.
2. Apply [`openwrt/qualcommax-ipq50xx.diff`](openwrt/qualcommax-ipq50xx.diff). It adds the
   image recipe (`ipq50xx.mk`), network and LED setup, caldata extraction, the upgrade
   handler, and patch `0191` (CMN PLL bus clock fix, OpenWrt PR #24653).
3. Copy [`0413-spi-spi-qpic-snand-write-the-feature-value-before-exe.patch`](0413-spi-spi-qpic-snand-write-the-feature-value-before-exe.patch)
   to `target/linux/qualcommax/patches-6.18/`. **This one is required.** Without it, every
   NAND read returns the chip's OTP area and the flash looks empty. It is a backport of
   upstream Linux commit `8fd62901d6bf`.
4. Wi-Fi board data: put the board files in `files/lib/firmware/ath11k/IPQ5018/hw1.0/board.bin`
   and `files/lib/firmware/ath11k/QCN6122/hw1.0/board.bin`, taken from your own unit's
   stock firmware. Do **not** put `cal-*.bin` files there: calibration is read from each
   unit's own ART partition at boot.
5. `make menuconfig`: Target `Qualcomm Atheros 802.11ax WiSoC-s`, subtarget `ipq50xx`,
   profile `D-Link DLA-X301`. Add `luci`. Then `make -j$(nproc)`.

## Repository layout

| Path | Contents |
|---|---|
| `ipq5018-dla-x301.dts` | device tree |
| `openwrt/qualcommax-ipq50xx.diff` | changes to the OpenWrt tree |
| `0413-*.patch` | SPI-NAND SET_FEATURE fix (required) |

## License

[GPL-2.0](LICENSE), the same as OpenWrt and the Linux kernel.

- `ipq5018-dla-x301.dts` is dual-licensed `GPL-2.0-or-later OR MIT`, as stated in its header.
- `0413-*.patch` is a backport of an upstream Linux commit and stays under the kernel's GPL-2.0.
