# Gemtek W1700K boot chain and flash layout

This document states how the W1700K boots and how its 512 MiB SPI NAND is laid out, under the
stock firmware and under OpenWrt. It also states which parts of the flash each stage owns. The reader is
porting a bootloader or an operating system to the board, or recovering one. The
[hardware reference](hardware-reference.md) states the board's parts. The
[AN7581 system reference](../system-reference.md#boot-security) states the SoC's boot ROM.

No reference unit has been probed yet. Every claim here is `[source]`, `[consensus]` or
`[reported]`.

## The boot chain

| Stage | Image | Where it lives | Owner |
|---|---|---|---|
| Boot ROM | Mask ROM | The SoC | Airoha |
| BL2 (preloader) | Signed, with DRAM training | `tcboot.bin`, offset 0 of flash | Vendor |
| BL31 | TF-A v2.3 | `tcboot.bin` | Vendor |
| Vendor U-Boot | U-Boot 2014.04-rc1 "AXON", LZMA-packed | `tcboot.bin` | Vendor |
| Stock firmware | FIT: Linux 5.4.55 and a rootfs | `tclinux` or `tclinux_slave` | Vendor |
| Chainloader (OpenWrt) | U-Boot 2026.x, FIT | Flash `0x60_0000`, 1 MiB | OpenWrt |
| OpenWrt | FIT: Linux 6.18 and squashfs | UBI volume `fit` | OpenWrt |

`tcboot.bin` is 540,672 bytes. It holds BL2 at offset 0 with its DRAM initialisation at
`+0x4420`, BL31 at `+0x2_0000`, and the LZMA-packed vendor U-Boot at `+0x2_a400`
[source][SDR]. The vendor U-Boot runs from `0x81e0_0000` [source][SDR].

BL2 prints `Secure key exist`, then trains the DRAM. It prints `AN7581DRAMC V0.7` under AXON
1.6 and `V0.8` under AXON 2.0 [consensus][T153][T323][T3498]. The secure-boot key is fused,
so the boot ROM runs only a BL2 signed with it [reported][T472][T565].

Under OpenWrt the vendor U-Boot still runs first. Its boot command reads the chainloader and
boots it, and the chainloader boots OpenWrt from UBI [source][CHAINLOADER].

## The vendor U-Boot

### Versions

| Banner | Shipped in |
|---|---|
| `U-Boot 2014.04-rc1 (Dec 01 2023 - 14:34:20) AXON 1.6` | Early units |
| `(Apr 01 2024) AXON 1.8` | `WXK001-05.00.30.62` |
| `(Jun 12 2024 - 08:14:34) AXON 2.0` | `WXK001-05.00.30.64` to `05.00.31.06` |
| `(Jun 26 2024 - 16:03:23) AXON 2.0`, `(Nov 26 2024 - 11:47:20) AXON 2.0` | Factory flash only |
| `(Dec 03 2025) AXON 2.3` | `WXK002-05.03.41.02`, HW2.1 |
| `(Apr 21 2026 - 07:22:08) AXON 2.5` | `WXK001-05.00.31.10` |

Sources: [consensus][T143][T156][T3244][T3274][T3450]. The firmware images update the
bootloader along with the system [reported][T3274].

### Console and commands

The vendor U-Boot prints `Hit any key to stop autoboot`, waits `bootdelay=4` seconds, and then
gives the prompt `ECNT>` [consensus][T143][TEARDOWN]. Its commands include `flash`,
`tftpboot`, `bootm`, `setenv`, `saveenv`, `printenv`, `md`, `mw`, `mdio`, `efuse`, `gpio`,
`imginfo`, `iminfo`, `loadb`, `loadx`, `loady` and `reset` [source][TEARDOWN][T98]. It has
no `reboot` command [reported][T2298].

`flash` takes `init`, `erase`, `read`, `write`, `dumpbmt` and `imgread` [reported][T98]. Its
offsets are logical: the vendor U-Boot remaps each through its bad-block table (see
[the vendor bad-block area](#the-vendor-bad-block-area)) [inferred].

### Image verification

Two load paths exist, and only one verifies a signature [source][SDR]:

| Command | Check |
|---|---|
| `flash imgread 2048; bootm` | BL31 verifies the image against a SHA-512 root-key digest in eFuse |
| `flash read <offset> <length> $loadaddr; bootm` | `bootm` checks the FIT's own hashes only |

The stock `bootcmd` is `flash imgread 2048;bootm` [source][TEARDOWN]. A failed check prints
`verify kernel:0x600000 error` and `Verify image fail`, or
`ERROR: BL31: Failed to verify image id 5 (-80)` [consensus][T100][T153][T3886]. The
chainloader boots by the second path [source][CHAINLOADER].

The vendor U-Boot rejects a FIT hash algorithm it does not know, so a FIT for it carries SHA-1
hashes only [source][SDR]. It prints `Unexpected image` and `Verify image!!!` over a TFTP load,
which are harmless [reported][T3621].

AXON 1.6 passes its environment to the kernel as boot arguments, including
`root=/dev/mtdblock4`. AXON 2.0 passes none [consensus][T189][T201].

### Environment

The environment is 16 KiB, in the `uenv` partition [source][TEARDOWN]. These variables bear on
a port [consensus][TEARDOWN][T334][T3941]:

| Variable | Stock value | Meaning |
|---|---|---|
| `bootcmd` | `flash imgread 2048;bootm` | Verified boot of the slot `bootflag` selects |
| `bootflag` | 0 or 1 | Firmware slot: 0 is `tclinux`, 1 is `tclinux_slave` |
| `bootdelay` | 4 | Seconds before autoboot |
| `loadaddr` | `0x8180_0000` | Default load address |
| `fdt_high` | `0xac00_0000` | Ceiling for the relocated tree |
| `bootfile` | `tclinux.bin` | TFTP file name |
| `fw_port` | 0 | The Ethernet port U-Boot uses |
| `serdes_pon`, `serdes_ethernet` | `23`, `11` | SerDes modes for the `wan` and `lan2` paths |
| `serdes_usb1`, `serdes_usb2` | `00`, `01` | SerDes modes for the USB PHYs. `01` gives USB1's lane to PCIe |
| `board`, `product_name`, `vendor_name` | `an7581_evb`, `xPON ONU`, `ECONET Technologies Corp.` | Reference-board names |

`serdes_ethernet=11` puts `lan2`'s path in USXGMII mode, and `12` in HSGMII mode
[reported][T3276].

### Networking

The vendor U-Boot's Ethernet driver is `ecnt_eth`. It reaches the network through the internal
switch's ports, which are `lan3` and `lan4` [consensus][T2736][T3365][B4]. It does not
bring up either 10G port on HW2.1 [reported][T3364].

It offers two TFTP modes:

- **Client:** `tftpboot <address> <file>` fetches from the host the environment's `serverip`
  names.
- **Server:** with no file named, it listens for a client to send `tclinux.bin` to the
  environment's `ipaddr` [source][T98][T174].

The client gives up after ten timeouts, and its CPU port inherits a 5 Mbit/s egress rate
limit from the switch [reported][SDR].

> [!CAUTION]
> **Start a `tftpboot` within a few seconds of interrupting autoboot.** Started later, or after
> other commands, the transfer stalls at `Loading: T T T` [consensus][T345][T699][T2298].
> Starting over from a reset clears it.

## The vendor bad-block area

The vendor U-Boot keeps a bad-block table and a pool of replacement blocks at the end of the
flash, which Linux knows nothing of [source][CHAINLOADER][T3039]. The area has three parts:

- **Pool:** 250 erase blocks, `0x1e0c_0000` to the end of flash. The vendor U-Boot prints
  `bmt pool size: 250` and `BMT & BBT Init Success` [consensus][T143][T2743][SDR].
- **Table:** a block that starts `RAWB` with a checksum holds the bad-block table. The
  last block holds the `BMT` header [reported][T3039][T3514].
- **Rewrite:** if the vendor U-Boot finds no valid `RAWB` table in the pool at boot, it
  erases the pool and writes a new table [reported][T3039][T3086].

The stock partitions end at `0x1be0_0000`, and nothing stock stores lies in the 66 MiB above it
[consensus][T573][T3332][SDR]. The table's code in an Airoha vendor kernel caps the pool at
250 blocks, `MAX_BMT_SIZE` [reported][T3086].

> [!CAUTION]
> **Keep every partition the vendor U-Boot does not own below `0x1be0_0000`.** Data in the last
> 250 blocks is erased by the next boot that finds no valid table there [reported][T3039].
> OpenWrt reserves all 66 MiB from `0x1be0_0000`, 528 blocks [source][DTS].

OpenWrt's tree marks `reserved_bmt` read-only and binds no bad-block driver to the NAND
[source][DTS][DTSI]. Linux therefore neither reads nor writes the vendor table.

## Stock flash layout

The stock kernel's partition string is
`2m[bootloader],2m[uenv],2m[dsd],64m[tclinux],64m[tclinux_slave],312m[system]`
[source][TEARDOWN][SDR]:

| Partition | Start | End | Size | Contents |
|---|---|---|---|---|
| `bootloader` | `0x0` | `0x20_0000` | 2 MiB | `tcboot.bin`: BL2, BL31, vendor U-Boot |
| `uenv` | `0x20_0000` | `0x40_0000` | 2 MiB | The vendor U-Boot's environment |
| `dsd` | `0x40_0000` | `0x60_0000` | 2 MiB | Factory data: see below |
| `tclinux` | `0x60_0000` | `0x460_0000` | 64 MiB | Firmware slot 0, a FIT |
| `tclinux_slave` | `0x460_0000` | `0x860_0000` | 64 MiB | Firmware slot 1, a FIT |
| `system` | `0x860_0000` | `0x1be0_0000` | 312 MiB | UBI: configuration, logs, overlay |
| (none) | `0x1be0_0000` | `0x2000_0000` | 66 MiB | The vendor bad-block area |

Each firmware slot holds a FIT of an LZMA kernel loaded at `0x8008_8000`, its tree and a
rootfs, each hashed with SHA-1 [consensus][TEARDOWN][T159]. The HW2.1 board uses the same
partitions [consensus][T3332][T3334][GH10].

### Factory data

`dsd` holds the unit's identity and calibration [source][INSTALLER][SDR]:

| Offset | Length | Contents |
|---|---|---|
| `0x40_0000` | `0x1000` | Key-value strings: `wan_mac`, `lan_mac`, `fan_id`, `serial_number`, and the stock credentials |
| `0x40_1000` | | Certificates (DER) |
| `0x40_5000` | `0x1e00` | The MT7996 calibration EEPROM. It starts with the chip ID, `90 79` |

The installer searches for the EEPROM one erase block at a time, up to four blocks from
`0x40_5000`, by its first two bytes [source][INSTALLER].

> [!CAUTION]
> **Keep a copy of the first 6 MiB of every unit.** `dsd` is unique to the unit: its MACs,
> serial and Wi-Fi calibration exist nowhere else. OpenWrt maps it read-only as `vendor`,
> so a dump from OpenWrt is a complete copy [source][DTS].

## OpenWrt flash layout

OpenWrt's layout keeps the vendor area and the bad-block area, and holds everything of its
own in one UBI device [source][DTS]:

| Partition | Start | End | Size | Linux access |
|---|---|---|---|---|
| `vendor` | `0x0` | `0x60_0000` | 6 MiB | Read-only |
| `chainloader` | `0x60_0000` | `0x70_0000` | 1 MiB | Read-only |
| `ubi` | `0x70_0000` | `0x1be0_0000` | 439 MiB, 3512 erase blocks | UBI |
| `reserved_bmt` | `0x1be0_0000` | `0x2000_0000` | 66 MiB, 528 erase blocks | Read-only |

The chainloader overwrites the first 1 MiB of stock slot 0. UBI covers the rest of both stock
slots and the stock `system` partition.

The installer creates these UBI volumes [source][INSTALLER], [consensus][T4037][T3311]:

| Index | Volume | Contents |
|---|---|---|
| 0 | `ubootenv` | The chainloader's environment, one LEB of 126,976 bytes |
| 1 | `ubootenv2` | Its redundant copy |
| 2 | `factory` | Static, 64 KiB: the calibration and identity, rebuilt from `dsd` |
| 3 | `recovery` | An OpenWrt initramfs FIT |
| 4 | `fit` | The OpenWrt FIT: kernel, tree and squashfs |
| 5 | `rootfs_data` | The overlay, created on first boot: about 380 MiB |

The `factory` volume's layout [source][INSTALLER][DTS]:

| Offset | Contents |
|---|---|
| `0x0` | The MT7996 EEPROM, `0x1e00` bytes |
| `0x5000` | `wan_mac`, 6 bytes |
| `0x6000` | `lan_mac`, 6 bytes, the base for the other addresses |
| `0x7000` | `fan_id`, ASCII |
| `0x8000` | `serial_number`, ASCII |

Linux mounts the root from the `fit` volume: `ubi.block=0,fit root=/dev/fit0`, with the console
on ttyS0 [source][DTS]. Sysupgrade writes the new FIT into `fit` [source][BASEFILES].

### Layout versions

OpenWrt's image carries `DEVICE_COMPAT_VERSION := 2.0` for this layout [source][IMAGE]. An
earlier layout, 1.0, ran UBI to `0x1fe0_0000` and reserved only the last 2 MiB
[consensus][T3014][T3696][B77]. Under it the vendor U-Boot's table rewrite landed inside UBI,
and about 3% of sysupgrades left a corrupt FIT [reported][T2754][T3039]. The two layouts
interact as follows:

- **Moving between layouts:** sysupgrade refuses an image whose compat version differs. Its
  message begins "Partition table has been changed to cooperate with the vendor bootloader"
  [source][IMAGE][T3907]. A unit moves by rewriting the chainloader and running the installer.
- **Telling them apart:** the chainloader's tree carries its `reserved_bmt` node.
  `strings /dev/mtd1 | grep reserved_bmt` prints `reserved_bmt@1be00000` on 2.0 and
  `reserved_bmt@1fe00000` on 1.0 [consensus][T3846][T3696].
- **Configuration backups:** a 1.0 backup restored on 2.0 needs
  `uci set system.@system[0].compat_version="2.0"` [consensus][T3145][T3147][T3381].

## The chainloader

The chainloader is mainline U-Boot with an `an7581_w1700k` board, built by OpenWrt as
`openwrt-airoha-an7581-gemtek_w1700k-ubi-chainload-uboot.itb` [source][CHAINLOADER][IMAGE].
It is a FIT whose configuration `conf-uboot` holds an LZMA U-Boot loaded at `0x8020_0000`
[consensus][T3498][T2743]. The vendor U-Boot runs it from this boot command:

```
bootcmd=flash read 0x600000 0x100000 $loadaddr; bootm
```

The vendor U-Boot prints `read: src=0x600000, len=0x100000, dst=0x81800000` and boots the
FIT [consensus][T3498][T2743].

| Property | Value |
|---|---|
| Versions seen | 2026.01-OpenWrt (r33314 to r34770), 2026.07-OpenWrt (r35591) |
| Environment | Redundant, in UBI volumes `ubootenv` and `ubootenv2` |
| Network | GDM1 only: the switch ports `lan3` and `lan4`, with a random MAC |
| Load addresses | `loadaddr=0x8180_0000`, `initaddr=0x8900_0000` |

Sources: [source][CHAINLOADER], [consensus][T3014][T3498][B372][B373].

Its `bootcmd` attaches `ubi`, then boots `fit` with the red LED lit. If that fails it boots
`recovery` with the blue LED lit [source][CHAINLOADER]. Its boot menu offers the same two,
plus a TFTP boot of a recovery image, a TFTP boot of the installer, and a reset. The
2026.04.25 build adds a reset-button trigger for recovery [reported][T2910].

The chainloader has no `tftpput` [consensus][T3014][T3039]. Its `saveenv` fails until the
installer has created the environment volumes [reported][T3515].

Its tree declares 8 GiB of memory with a usable range of 2 GiB from `0x8020_0000`, and keeps
`0x8000_0000` to `0x8020_0000` for BL31 [source][CHAINLOADER]. It prints `DRAM: 8 GiB (total
2 GiB)` [reported][T3498].

## The installer

The community installer is an OpenWrt initramfs that builds the OpenWrt layout from a stock or
OpenWrt unit [source][INSTALLER]. The chainloader's menu boots it over TFTP. It runs:

1. Reads the EEPROM and the `dsd` strings from raw flash, and identifies the model by the
   serial number's prefix.
2. If a UBI device already attaches, asks `Existing UBI layout detected. Proceed and
   overwrite? (yes/no)`. Only `yes` proceeds.
3. Builds the 64 KiB `factory` blob.
4. Runs `ubiformat` on the `ubi` partition, and creates the volumes in the table above.
5. Writes its bundled recovery image to `recovery`, and its bundled sysupgrade image to `fit`
   through `nand_upgrade_fit`. Then it reboots.

| Release | Flash parts | Layout |
|---|---|---|
| hurrian/w1700k-ubi-installer `2026.04.07` and older | W25N04KV | 1.0 |
| hurrian/w1700k-ubi-installer `2026.04.25` | W25N04KV | 2.0 |
| w1700k/ubi2-installer `ubi2-installer-beta` (2026-10-18) | W25N04KV, W25N04LV | 2.0 |

Sources: [consensus][T3153][T3305][B512].

The installer fails intermittently with `ubidetach: error!: cannot detach "/dev/mtd2" error 16`
or `Volume fit not found!`. A second run passes [consensus][T2447][T2451][T2399].

## Recovery paths

Every path below starts at the console. Each works while `vendor` is intact, and nothing in
OpenWrt writes it.

| Damage | Path |
|---|---|
| `fit` | The chainloader falls back to `recovery`, or its menu boots `recovery`. Sysupgrade from there |
| `ubi` | The chainloader's menu boots the installer over TFTP |
| `chainloader` | Interrupt the vendor U-Boot and rewrite `0x60_0000` |
| `vendor` | The boot ROM's XMODEM loader. It accepts a BL2 signed with the fused key only |

Holding GPIO0, the reset button, low at power-on starts the boot ROM's loader. It prints
`Press x` and then takes XMODEM [reported][T511][T565]. With the key fused, the boot ROM
accepts only a FIP whose BL2 carries the key's signature. The proprietary BL2 then offers to
write the upload over the first 512 KiB of flash [reported][T565].

A unit with no console output at all was brought to the chainloader menu in RAM through that
loader. It did not survive a power cycle [reported][T3681][T3687][T3703].

A unit returns to stock by rewriting a vendor image from `0x60_0000` and restoring
`bootcmd=flash imgread 2048; bootm` [consensus][T163][T1673][T3326]. The vendor U-Boot's
`tftpboot` loads at most about 10 MiB in one transfer, so the image goes over in 10 MiB parts
[reported][T163][T596].

## Sources

Citation keys:

- `[DTS]`: OpenWrt `target/linux/airoha/dts/an7581-w1700k-ubi.dts`, main branch at commit
  `706052b20fd1` (2026-11-07).
- `[DTSI]`: the same tree's `an7581.dtsi`.
- `[IMAGE]`: the same tree's `target/linux/airoha/image/an7581.mk`, the `gemtek_w1700k-ubi`
  device.
- `[BASEFILES]`: the same tree's `target/linux/airoha/an7581/base-files/lib/upgrade/platform.sh`.
- `[CHAINLOADER]`: the same tree's
  `package/boot/uboot-airoha/patches/999-airoha-add-gemtek-w1700k.patch`, with its commit
  message, and `package/boot/uboot-airoha/Makefile`.
- `[INSTALLER]`: hurrian/w1700k-ubi-installer at commit `46df1f5`,
  `files/installer/install.sh`.
- `[SDR]`: h0m3us3r/w1700k-sdr at commit `0b0413e`, `include/hw/an7581/w1700k_board.h`. It
  documents the stock partition map compiled into the vendor U-Boot, `tcboot.bin`'s layout,
  the image checks and the NAND geometry.
- `[TEARDOWN]`: hungvu.tech, "Quantum Fiber W1700K teardown, board view, and UART pins",
  2024-10-01. It carries the vendor U-Boot's command list and full environment. It also carries
  a stock boot log from firmware `WXK001-05.00.30.64`.
- `[GH10]`: GitHub discussion 10 in w1700k/builds, "W1700K2 Support?", 2026-05-12 to
  2026-06-06. It carries a HW2.1 unit's stock layout and the RTL8261CE driver work on it.
- `[T<n>]`: post n of the OpenWrt forum thread "Quantum Fiber W1700k support", topic 222776,
  read 2026-11-07.
- `[B<n>]`: post n of the OpenWrt forum thread "Gemtek W1700K Community Builds", topic 249319,
  read 2026-11-07.
