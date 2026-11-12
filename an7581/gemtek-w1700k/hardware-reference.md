# Gemtek W1700K hardware reference

This board-level reference covers the Gemtek W1700K, a Wi-Fi 7 router built on the Airoha
AN7581 SoC and the MediaTek MT7996 Wi-Fi chip. ISPs ship it as the Quantum Fiber, CenturyLink
and Lumen W1700K. It states the board's parts, ports, wiring, addresses and limits. It is
compiled from OpenWrt's device tree, the community installer, a teardown and the community's
posted logs. It is not a manufacturer publication.

No reference unit has been probed yet. The one `[measured]` claim is a reading of the
reference units' labels, and every other claim is `[source]`, `[consensus]` or `[reported]`.
Related documents:

- [Boot chain and flash layout](boot-and-flash.md): the stock bootloader, the stock and
  OpenWrt flash layouts, the bad-block area, the chainloader and the installer
- [OpenWrt status](openwrt-status.md): what OpenWrt drives on the board, per subsystem
- [Corrections and gotchas](corrections-and-gotchas.md): misreported claims and board traps
- The [AN7581 system reference](../system-reference.md) and
  [network reference](../network-reference.md): the SoC's blocks
- The [MT7996 reference](../../mt7996/wifi-reference.md): the Wi-Fi chip

## Overview

| Property | Value |
|---|---|
| Product | Gemtek W1700K. FCC IDs MXF-W1700K (HW1.0, HW1.1) and MXF-W1700K2 (HW2.1) |
| Class | Tri-band Wi-Fi 7 router with fan cooling |
| SoC | Airoha AN7581GT: 4x Cortex-A53 |
| System memory | 2 GiB DDR4-2666 |
| Storage | 512 MiB SPI NAND |
| Wi-Fi | MediaTek MT7996 on PCIe: 4x4 at 2.4 GHz, 4x4 at 5 GHz, 4x4 at 6 GHz with a fifth receive chain |
| Ethernet | Two 10G copper ports through Realtek PHYs, two 1G ports on the SoC's switch |
| Other radios | Silicon Labs EFR32MG21 (Bluetooth LE, 802.15.4), Airoha AG3352Q GNSS |
| Console | ttyS0, 115200 baud, on a 5-pin header |
| Power | 12 V DC barrel input |

The teardown and the forum's opening post give 2 GiB of RAM, 512 MB of storage, two 10G
ports and two 1G ports [reported][TEARDOWN][T1].

## Identity

OpenWrt's tree names the board `model = "Gemtek W1700K (OpenWrt U-Boot layout)"`,
`compatible = "gemtek,w1700k-ubi", "airoha,an7581", "airoha,en7581"` [source][DTS]. Its image
profile is `gemtek_w1700k-ubi`, which also carries the CenturyLink, Lumen and Quantum Fiber
names [source][IMAGE]. The `-ubi` names belong to OpenWrt's flash layout, not to a hardware
variant (see [boot chain and flash layout](boot-and-flash.md)).

The stock firmware's tree is compatible with `econet,en7581` [reported][T30]. EcoNet is
Airoha's former name, and the stock bootloader and environment still carry it
[reported][TEARDOWN].

The MXF-W1700K grant dates from 2023-10-06. A class II permissive change on 2024-04-09 added
the 5 GHz DFS bands in software and an antenna option [source][FCC]. Its cover letter states
that "neither circuit design nor main function has been changed" [source][FCC]. A 6 GHz
standard-power grant followed on 2025-03-17 [source][FCC]. The MXF-W1700K filing's main board
is marked `WAPE-269BE_V01` and `ESX-X21-6401R`, with an Axon Networks logo [source][FCC].

The MXF-W1700K2 grants date from 2025-09-09, and its 6 GHz standard-power grant from
2025-12-10 [source][FCCAPI]. Its test reports name the model `W1700K`, under the brands Q Fiber
and Lumen. Its main board is marked `WAPE-269BE_V2_V03` and `19X-X20-8501R` [source][FCC2].

### Revisions

Three hardware revisions are on the market [consensus][T3417][T3649]:

| Revision | FCC ID | 10G PHYs | SoC product ID | Supply | Fan |
|---|---|---|---|---|---|
| HW1.0 | MXF-W1700K | RTL8261N | 1, pre-E2 silicon | 12 V 5 A, 60 W | 12 V, Sunon or Yingfan |
| HW1.1 | MXF-W1700K | RTL8261N | 2, E2 silicon | 12 V 5 A, 60 W | 12 V, Sunon or Yingfan |
| HW2.1 ("W1700K2", Lumen) | MXF-W1700K2 | RTL8261CE | 2, E2 silicon | 12 V 4 A, 48 W | 60x60x10 mm, Power Logic PLA06010S12M or Fuzhineng FDF6010S12M |

- **Label:** the three reference units' labels read model `W1700K`, FCC ID `MXF-W1700K2`
  and HW version `2.1` [measured]. The model field reads `W1700K` on HW2.1 too, so only the
  FCC ID and HW version fields identify the revision. The HW2.1 label carries Lumen's name
  [reported][T3277].
- **SoC product ID:** the register at `0x1fb0_005c` reads 1 on three HW1.0 units and 2 on
  HW1.1 and HW2.1 units [reported][T3247][T3331]. The
  [system reference](../system-reference.md#silicon-revisions) describes the register.
- **PHYs:** the RTL8261N reads PHY ID `0x001c_caf3`, the RTL8261CE `0x001c_c899`
  [consensus][T3453][T3495][T3683]. The two need different drivers and firmware (see
  [Ethernet](#ethernet)).
- **Flash:** units of every revision carry a Winbond W25N04KV. Some later units carry a
  W25N04LV [reported][T3990][B505]. The stock bootloader AXON 2.5 recognises the W25N04L
  part [reported][T3273].
- **Board straps:** the stock firmware reads the revision from GPIOs. The major number is
  2 x GPIO30 + GPIO18, and the minor number 2 x GPIO45 + GPIO44 [source][SDR].

The HW2.1 board's vendor flash partitioning and stock tree match the W1700K's [reported][T3334].
Its 6 GHz power tables differ [reported][T3286]. Its heatsink has lower fins, and it has no
RF-shield sticker or power choke at the lower left [reported][T3287][T3303].

Two related Gemtek boards share the platform:

| Board | FCC ID | Differences |
|---|---|---|
| W1701K, the access-point sibling | MXF-W1701K | Firmware named `W1K001`/`W1K002` [reported][T3285]. Its grants list Bluetooth LE and 802.15.4 at 0.01 W [source][FCCAPI] |
| Brightspeed XR1710G | MXF-XR1710G | RTL8261BE 10G PHYs [reported][SKY]. No GNSS receiver or EFR32 [reported][T1974], and its grants list no Bluetooth LE or 802.15.4 band [source][FCCAPI] |

The community installer accepts serial numbers that start with `W1700K`, `W1701K` or
`XR1710G` [source][INSTALLER].

## Parts

| Function | Part | Bus and address |
|---|---|---|
| SoC | Airoha AN7581GT | |
| DRAM | 2x ESMT M16U8G16512A, DDR4 | |
| NAND | Winbond W25N04KV, or W25N04LV on some later units | SNFI, SPI, 50 MHz |
| Wi-Fi baseband | MediaTek MT7996AV | PCIe |
| Wi-Fi RF | MediaTek MT7976GN (2.4 GHz), MT7977BN and MT7977AN (5 and 6 GHz) | SPI from the MT7996 |
| 10G PHYs | 2x Realtek RTL8261N (HW1.x) or RTL8261CE (HW2.1) | USXGMII, MDIO 5 and 8 |
| Hardware monitor and fan controller | Nuvoton NCT7802 | i2c0, `0x2e` |
| Bluetooth LE and 802.15.4 | Silicon Labs EFR32MG21A010F512 | ttyS1 |
| GNSS | Airoha AG3352Q | ttyS2 |

The part markings come from the teardown's photographs [reported][TEARDOWN]. The SPI clock and the
monitor's address come from OpenWrt's trees [source][DTS][DTSI].

The two DRAM packages of both FCC samples carry Micron's logo and the code `D8BPK`, where the
teardown's unit carries ESMT parts [source][FCC][FCC2][reported][TEARDOWN].

The FCC photographs mark the monitor NCT7511Y on the W1700K and NCT7512S on the W1700K2
[reported][T3337][SDR]. The W1700K2 sample's SOIC-14 under the SoC reads `NCT7512S`
[source][FCC2]. Linux binds the `nct7802` driver on both, and it reads valid voltages,
temperatures and fan speed [consensus][T546][T3340].

## Ethernet

The board has four Ethernet ports [source][DTS][BASEFILES]:

| Port | Speed | MAC path | PHY | MDIO address | PHY reset |
|---|---|---|---|---|---|
| `wan` | 10/100/1000/2500/5000/10000 Mb/s | GDM2, through the PON PCS | RTL8261N or RTL8261CE | 8 | GPIO31 |
| `lan2` | 10/100/1000/2500/5000/10000 Mb/s | GDM4, through the Ethernet PCS | RTL8261N or RTL8261CE | 5 | GPIO46 |
| `lan3` | 10/100/1000 Mb/s | GDM1, switch port 1 | Internal | 9 | |
| `lan4` | 10/100/1000 Mb/s | GDM1, switch port 2 | Internal | 10 | |

The 1G ports have yellow sockets [consensus][T87][T4009][B350]. The SoC's internal switch
carries `lan3` and `lan4`, and GDM1 is its 10G CPU link. The switch's ports 3 and 4 are
disabled, and no `lan1` exists [source][DTS][DTSI]. The
[network reference](../network-reference.md) describes the GDMs, the PCS blocks and the
switch.

OpenWrt bridges `lan2`, `lan3` and `lan4` as its default LAN, and makes `wan` the WAN
[source][BASEFILES]. The stock firmware calls the same two 10G MACs `ae_wan` and
`hsgmii_lan` [reported][TEARDOWN].

### The 10G PHYs

Both PHYs are clause 45 devices on the switch's MDIO bus, with TX and RX lane polarity inverted
[source][DTS]. Their interrupt lines reach GPIO23 (`wan`) and GPIO22 (`lan2`), and OpenWrt's tree
leaves both unused, so Linux polls the PHYs [source][DTS]. Each PHY's reset is held 200 ms
asserted and 200 ms after release [source][DTS]. The two parts differ in their needs:

- **RTL8261N** (HW1.x): needs at least 150 ms after reset before its management interface
  answers, per Realtek's datasheet [reported][T2823]. Linux loads PHY firmware from
  `rtl826x-firmware` [source][IMAGE].
- **RTL8261CE** (HW2.1): needs firmware from `rtl8261c-firmware` [source][IMAGE]. OpenWrt has
  supported it since about 2026-09-24 [consensus][T3961][B477]. Its SerDes lane polarity
  sits behind a vendor register window that the RTL8261N's method does not reach. Without
  that setting, `wan` links at 10G and receives no frames [reported][GH10]. The W1700K2 FCC
  sample's two PHYs are marked `RTL8261CE`, each beside its own 50 MHz crystal
  [source][FCC2].

Each PHY drives the LEDs of its own port, on its LED outputs 2 (yellow) and 3 (green)
[source][DTS]. Linux also reads each PHY's temperature through the Realtek PHY hwmon driver.
After 60 s of 10G traffic the two read about 61 and 65 °C [reported][T713].

### MAC addresses

The factory data holds two addresses, `wan_mac` and `lan_mac` [source][INSTALLER]. OpenWrt
derives the others from `lan_mac` [source][DTS]:

| Interface | Address |
|---|---|
| `wan` | `wan_mac` |
| GDM1 and `lan2` | `lan_mac` |
| Wi-Fi band 0 (2.4 GHz) | `lan_mac` + 1 |
| Wi-Fi band 1 (5 GHz) | `lan_mac` + 2 |
| Wi-Fi band 2 (6 GHz) | `lan_mac` + 3 |

## Wi-Fi

The MT7996 attaches through two PCIe interfaces [consensus][T50][T275][SDR]:

| Interface | PCI ID | SoC controller | Lanes | Use |
|---|---|---|---|---|
| Primary | `14c3:7990` | pcie0 at `0x1fc0_0000`, with pcie1's MAC as its second lane | x2 | Every band |
| Second | `14c3:7991` | pcie2 at `0x1fc4_0000`, over the USB1 PHY's SerDes | x1 | A second DMA path to the same chip |

The chip's reset line reaches no SoC GPIO: a sweep of every GPIO found none [reported][T1172][T1201]. The
[MT7996 reference](../../mt7996/wifi-reference.md) describes the chip, its bands and its
firmware.

OpenWrt describes one MT7996 node with three bands, band 0 (2.4 GHz), band 1 (5 GHz) and
band 2 (6 GHz) [source][DTS]. It reads the chip's calibration EEPROM, 0x1e00 bytes, from the
`factory` UBI volume [source][DTS]. Wi-Fi offload runs through the SoC's NPU, which the tree
links to the MT7996 node [source][DTS].

The antennas are internal, on U.FL connectors. The W1700K2's antenna report lists them, all
PIFAs [source][FCC2]:

| Antenna | Band | Use | Peak gain |
|---|---|---|---|
| `Dual_Ant1` to `Dual_Ant4` | 2.4 and 5 GHz | Transmit and receive | 1.4 to 3.0 dBi |
| `6E_Ant1` to `6E_Ant4` | 6 GHz | Transmit and receive | 1.3 to 3.3 dBi |
| `6E_Ant5` | 6 GHz | Receive only | 4.2 to 4.5 dBi |
| `DFS` | 5 GHz | Receive only | 4.3 to 4.4 dBi |

A loose connector has caused a 6 GHz range loss that reseating fixed
[reported][T3570][T3592]. Antenna connector 1 is shared by band 0's
chain 0 and band 1's chain 0 [reported][SDR].

## Other radios

### EFR32MG21

A Silicon Labs EFR32MG21A010F512IM32 sits on ttyS1 [consensus][T75][T327][T561]. The
[EFR32MG21 reference](../../efr32mg21/radio-reference.md) describes the chip, its bootloader
and the protocols its firmware speaks. The stock firmware of the router does not use it
[reported][T2591].

The two FCC samples' chips differ in silicon revision [source][FCC][FCC2]:

| Sample | Marking | Silicon revision |
|---|---|---|
| MXF-W1700K | `MG21`, `A010HI`, `B02060`, `2210` | B |
| MXF-W1700K2 (U8) | `MG21`, `A010HI`, `D00ZA…`, `2311` | D |

In the marking, `H` is the 512 kB flash code and the third line's first letter is the silicon
revision [source][DS]. Revision D has its own electrical tables (see the
[EFR32MG21 reference](../../efr32mg21/radio-reference.md#ordering-code)). A Simplicity Commander readout of one unit gave `EFR32MG21A010F512IM32-B`, with
debug lock and secure boot disabled and device erase enabled [reported][DOER].

#### Wiring

| Signal | EFR32 pin | Board connection |
|---|---|---|
| UART TX | PA05, USART0 | ttyS1, and J10 pin 5 |
| UART RX | PA06, USART0 | ttyS1, and J10 pin 4 |
| RESETn | Pin 9 | SoC GPIO24, active low, and J10 pin 3 |
| Bootloader activation | PB0 | SoC GPIO25, active low |
| SWDIO, SWCLK | PA02, PA01 | J10 pins 7 and 8 |
| SWO | PA03 | J10 pin 6 |
| Packet trace (PTI) | PC05 frame, PC04 data | J10 pins 9 and 10 |
| Crystal | HFXO | Y3, 38.4 MHz |

Sources: [consensus][GTX][DOER] for the UART and the GPIOs, [reported][DOER] for J10,
[source][FCC][FCC2] for the crystal. The crystal is marked `38` on the W1700K sample and
`38.400` on the W1700K2 sample. Linux's sysfs numbers GPIO24 and GPIO25 as 536 and 537, from a gpiochip base
of 512 [reported][GTX].

J10 is a 2 x 5 header position at 1.27 mm pitch, in Silicon Labs' Mini Simplicity layout.
Pin 1 is VTARGET and pin 2 GND [reported][DOER]. Retail units leave it unpopulated
[reported][DOER], and the W1700K2 FCC sample carries a fitted header [source][FCC2]. J10's pins 4 and 5 share their wires with
the SoC's UART, so a debug adapter on them contends with ttyS1 [reported][DOER].

No firmware uses CTS or RTS. Whether the board wires them is disputed: two posters read the
layout as wired [reported][T511][T555], and two pin maps show only TX and RX
[reported][GTX][DOER]. The Airoha 16550 driver has no automatic flow control
[reported][T511]. Community builds for the board use USART0 on PA05 and PA06 without flow
control, with HFXO CTUNE 128 [source][SWB][GTX].

#### Factory firmware

| Revision | Application | Bluetooth stack | Bootloader |
|---|---|---|---|
| HW1.x | Bluetooth NCP, BGAPI | 4.2.1, build 342 | `0x0201_0002`: 2.1, customer 2 |
| HW2.1 | Bluetooth NCP, BGAPI | 6.2.0 | `0x0203_0002`: 2.3, customer 2 |

Sources: [consensus][T513][T3769][DOER] for HW1.x, [reported][GTX] for HW2.1. The 4.2.1
stack shipped in Gecko SDK 4.1.3, and 6.2.0 in Gecko SDK 4.3.2 (see the
[EFR32MG21 reference](../../efr32mg21/radio-reference.md#firmware-and-host-protocols)). The
HW1.x unit's revision is [inferred] from its date: the post predates HW2.1's bootloader. The
HW1.x image identifies itself as `W1700K_Axon_BLE` `v0.1` [reported][T504]. A stray byte on
ttyS1 draws `a0 03 01 06 4a 00 00` from it [consensus][Z2M][T513]. That is a Gecko SDK NCP's
incomplete-command error, and its `0x4a` showed on a terminal as "J" [reported][T475].

The factory bootloader speaks BGAPI UART DFU at 115200 baud without flow control, and no
XMODEM [consensus][GTX][DOER]. It boots unsigned applications and ignores `dfu_reset`
[reported][GTX]. It refuses LZ4-compressed GBL files with reason `0x0046` [reported][DOER].
On HW2.1, it does not install an unsigned bootloader upgrade [reported][T3962][GTX].

#### Entering the bootloader

- **By pin:** hold GPIO25 low, pulse GPIO24 low, then release GPIO25. The bootloader answers
  with `dfu_boot` at 115200 [consensus][GTX][DOER].
- **By command, HW2.1:** the factory NCP takes `user_reset_to_dfu`, `20 00 ff 02`
  [reported][GTX].
- **By command, the 4.2.1 image:** `system_reset` in DFU mode boots the application
  [reported][DOER]. The chip reference explains why.

A GPIO24 and GPIO25 sequence in [T3608] restarted the application. That sequence released
GPIO25 before GPIO24, so the pin was already inactive as RESETn rose [inferred].

> [!CAUTION]
> **Dump the EFR32's flash over SWD before a device erase.** A device erase removes the
> factory bootloader. If OTP secure boot is on, only an image signed with Gemtek's key boots
> afterwards, and the dump holds the only such image [inferred][RTSL]. No HW2.1 unit's
> secure-boot state has been published.

#### Radio environment

The antenna is a printed inverted-F trace on the board's edge, 2.94 dBi, with no connector.
A coaxial test receptacle, JB1, sits between it and the chip [source][FCC]. The MXF-W1700K
filing measured 9.71 dBm peak conducted power for Bluetooth LE. It gives 9.72 dBm for 802.15.4,
which it calls Thread, against 10 dBm rated [source][FCC].

The MXF-W1700K2 grants list Bluetooth LE at 2402-2480 MHz and 802.15.4 at 2405-2480 MHz,
0.011 W each, where the MXF-W1700K grants list 0.009 W [source][FCCAPI]. Its Bluetooth report
measured 10.40 dBm peak conducted power, with BGTool 2.13.7 at power index 100, as the
MXF-W1700K tests used [source][FCC2]. Its photographs show the same printed antenna beside
JB1 [source][FCC2]. Its antenna report lists that antenna with a U.FL connector, where its
Bluetooth report gives none [source][FCC2].

The MXF-W1700K filing's simultaneous-transmission tables pair Bluetooth or Thread only with 5 and 6 GHz
Wi-Fi, never with 2.4 GHz Wi-Fi [source][FCC]. That fits a declared rule that the two never
transmit together [inferred]. Gemtek's block diagram shows grant, request and priority lines
between the MT7996 and the EFR32 [reported][T3962]. The MXF-W1700K filing holds its block
diagram as confidential [source][FCC]. The EFR32 pins of those lines are unknown, and the factory
application sets up no pin beyond PA05 and PA06 [reported][GTX][T3988].

The board's own 2.4 GHz Wi-Fi reaches the EFR32 strongly. With Wi-Fi on channel 11, 802.15.4
channel 20 saw −14 to −26 dBm of noise, and with Wi-Fi on channel 1, −52 dBm
[reported][T3962]. Keep the Wi-Fi channel away from the Zigbee or Thread channel. An
OpenThread build defaulted to 0 dBm, and `ot-ctl txpower 10` raised it to the part's maximum
[reported][T3962].

### AG3352Q GNSS

An Airoha AG3352Q GNSS receiver sits under its own shield on the heatsink side
[reported][T3963]. The stock firmware uses it for automated frequency coordination of the
6 GHz band [reported][T431][T2596]. Its firmware reports
`Project Version: AG3352Q_V2.6.0.AG3352_20221230` [reported][T3788]. Its connections:

- **UART:** it sits on ttyS2, the SoC's HSUART. Its line runs at 921600 baud, which Linux
  reaches only with a corrected baud divider (see [UARTs](#uarts)) [reported][T3788][T3804].
- **Control:** SoC GPIOs 13 and 14 reach the receiver. The stock firmware names them power
  control and notify [reported][T3982].
- **PPS:** its PPS output, pin 18, reaches test point TP153 and no SoC pin [reported][T4000].
- **Antenna:** a ceramic patch of about 25 mm on a U.FL lead, fixed at the center of the
  unit's underside [source][FCC]. The W1700K2's antenna report gives it as a GPS L1 patch,
  1575.42 MHz, and its photographs show the receiver marked `AG3352Q` [source][FCC2].
- **Headers:** unpopulated 4-pin headers J24 and J25 sit by the yellow ports. J24 is GND, RX,
  TX, 3.3 V [reported][T3978].

> [!CAUTION]
> **Leave the empty footprint beside TP153 open.** It is a 3.3 V pull-up strap on the
> receiver's GPIO10. Pulled high at power-on, the receiver starts in its SPI download mode
> [reported][T4000].

## Console and UARTs

### The console header

The console is a 5-pin header, pre-populated, near R1317. It carries TX, GND, 3.3 V, an empty
position and RX, at 115200 8N1 [consensus][T55][T62][TEARDOWN].

The pin order is disputed. One account counts TX first from R1317 [reported][T55]. Another
counts the same order from the reset button [reported][T466]. A third lists the pins top to
bottom with the header at the top left [reported][T609].

> [!CAUTION]
> **Identify GND and TX with a meter before connecting an adapter.** GND has continuity to the
> shield, and TX toggles while the boot log prints. Leave the 3.3 V pin unconnected
> [consensus][T470][T512].

The header's 3.3 V pin can power an isolated USB-serial adapter's board side, a few mA
[reported][T3700].

### UARTs

Linux registers three UARTs [consensus][T551][T2701][B166]:

| Device | Base | Tree node | Use |
|---|---|---|---|
| ttyS0 | `0x1fbf_0000` | `uart1`, ns16550 | Console, 115200 baud |
| ttyS1 | `0x1fbf_0300` | `uart2`, Airoha 16550 | EFR32MG21 |
| ttyS2 | `0x1fbe_1000` | `hsuart3`, Airoha 16550 | AG3352Q GNSS |

The tree's `serial0` alias is `uart1`, and its `stdout-path` is `serial0:115200n8`
[source][DTS].

> [!CAUTION]
> **The Airoha UARTs run at twice the requested baud on OpenWrt's 6.18 kernels.** The
> `uart2` driver needs 230400 to talk at 115200, and 460800 on ttyS2 gives 921600
> [reported][T3788][T3804]. A fix exists out of tree (see the
> [corrections](corrections-and-gotchas.md#the-airoha-uart-baud-divider)).

## Memory and storage

The DRAM is 2 GiB of DDR4 at 2666 MT/s, from `0x8000_0000` [consensus][T153][T2743][TEARDOWN].
The SoC's firmware and offload engines keep about 200 MiB of it (see the
[system reference](../system-reference.md#memory)). Linux reports
`Memory: 1863940K/2097148K available` on an OpenWrt 6.12 build [reported][T3453].

The NAND is 512 MiB of SLC SPI NAND with 128 KiB erase blocks, 2048-byte pages and 128 bytes of
spare area per page [consensus][T50][T3453][B115]. The W25N04KV corrects errors on the die,
8 bits per 512-byte step [reported][T3696][SDR]. The
[boot chain and flash layout](boot-and-flash.md) gives the partitions.

The SoC has an eMMC controller and two xHCI controllers. The board brings out no eMMC and no
USB port [consensus][T531][T554]. The USB1 PHY's SerDes carries the MT7996's second PCIe
interface instead [source][DTSI].

## Cooling and monitoring

A 12 V fan cools the board, under the NCT7802's automatic control [source][BASEFILES].
OpenWrt's boot script sets the vendor curve on `pwm1` [source][BASEFILES]:

| Temperature | 40 °C | 50 °C | 60 °C | 70 °C | 80 °C |
|---|---|---|---|---|---|
| PWM value (0-255) | 54 | 69 | 95 | 199 | Not set |

On a Sunon fan those values give about 770, 1000, 1400 and 2700 rpm [reported][T544]. The
factory data names the fan model in `fan_id`, for example `sunon_w1700k`
[reported][T2573][T527].

The NCT7802 reports `in0` (the 3.3 V rail), `fan1`, a thermal diode on `temp1` and `temp4`.
Its `temp2` reads `FAULT` [consensus][T546][T3340]. The SoC's own sensors and trip points are
in the [system reference](../system-reference.md#thermal). The MT7996 reports a temperature
per band [reported][T546].

## Power

The input is 12 V DC on a barrel socket:

| Revision | Supply | Label | Plug |
|---|---|---|---|
| HW1.0, HW1.1 | 12 V 5 A, 60 W, ML60-4120500-A1 | 120 V, 60 Hz [consensus][T262][T745] | 5.5 x 2.1 mm, center positive [consensus][T741][T745] |
| HW2.1 | 12 V 4 A, 48 W, MOSO MS-V4000R120-050A0-US or Lucent Trans 1A133-LHHL [source][FCC2] | 100-120 V [reported][T3539] | 6.3 x 3.0 mm [reported][T3906] |

A 5.5 x 2.5 mm plug can fail to make contact in the HW1.x socket [reported][T741]. One HW1.0
supply's main capacitor is rated 250 V [reported][T3638]. Supplies run at 230 V have been
reported both hot and fine [reported][T3594][T3635].

Measured draw, HW1.x: 18 W typical, 28 W peaks and 36 W under a WireGuard load. A WAN on a 1G
port draws about 3 W less than on a 10G port [reported][T3470]. Measured draw, HW2.1: under
10 W idle with no links, about 12 W as an idle access point and 16-18 W under iperf
[reported][T3670].

> [!CAUTION]
> **Rule out the power supply before blaming a build for Wi-Fi or boot failures.** Three
> failures cleared on a supply change [reported][B258][B311][B447]. PCIe links dropped as the
> 5 GHz amplifier powered on, PCIe raised fatal AER errors, and the board reset under Wi-Fi
> traffic.

## Board GPIO assignments

All of OpenWrt's LEDs and the button are active low [source][DTS]:

| Signal | GPIO | Source |
|---|---|---|
| Reset button (`KEY_RESTART`), and the boot ROM's recovery trigger | 0 | [source][DTS], [reported][T511] |
| MDIO pin mux, driven high | 2 | [source][DTS] |
| `lan3` LEDs, green and yellow | 9, 10 | [source][DTS] |
| GNSS control and notify | 13, 14 | [reported][T3982] |
| Status LED, green | 17 | [source][DTS] |
| Revision strap, major bit 0 | 18 | [source][SDR] |
| Status LED, blue | 19 | [source][DTS] |
| Status LED, white | 20 | [source][DTS] |
| `lan2` PHY interrupt (unused) | 22 | [source][DTS] |
| `wan` PHY interrupt (unused) | 23 | [source][DTS] |
| EFR32 reset sequence | 24, 25 | [reported][T3608] |
| `lan4` LEDs, green and yellow | 27, 28 | [source][DTS] |
| Status LED, red | 29 | [source][DTS] |
| Revision strap, major bit 1 | 30 | [source][SDR] |
| `wan` PHY reset | 31 | [source][DTS] |
| Revision strap, minor bits 0 and 1 | 44, 45 | [source][SDR] |
| `lan2` PHY reset | 46 | [source][DTS] |

OpenWrt maps its boot state to the status LEDs: red for boot, blue for failsafe and
upgrade, green for normal running [source][DTS]. The stock firmware shows red while booting and
green when up [reported][T482].

The tree's GPIO controller maps 50 GPIOs, from pin 13 of the pin controller [source][DTS].
GPIO input registers sit at `0x1fbf_0204` (GPIO0-31) and `0x1fbf_0270` (GPIO32-63)
[reported][T3983].

## The case

One T10 Torx screw holds the case, under the label sticker. About ten clips hold the two halves,
which part between the gray and white shells from the port side or the vents
[consensus][T106][T123][T543]. On HW2.1 the screw sits near the "Factory ID" label
[reported][T3649].

## Sources

Citation keys:

- `[DTS]`: OpenWrt `target/linux/airoha/dts/an7581-w1700k-ubi.dts`, main branch at commit
  `706052b20fd1` (2026-11-07).
- `[DTSI]`: the same tree's `an7581.dtsi`, `an7581-npu-mt7996.dtsi` and
  `an7581-npu-wlan.dtsi`.
- `[BASEFILES]`: the same tree's `target/linux/airoha/an7581/base-files/`:
  `etc/board.d/01_leds`, `etc/board.d/02_network` and `etc/init.d/airoha_fan`.
- `[IMAGE]`: the same tree's `target/linux/airoha/image/an7581.mk`, the `gemtek_w1700k-ubi`
  device.
- `[INSTALLER]`: hurrian/w1700k-ubi-installer at commit `46df1f5`,
  `files/installer/install.sh`.
- `[SDR]`: h0m3us3r/w1700k-sdr at commit `0b0413e`: `include/hw/an7581/w1700k_board.h`,
  `include/hw/adie/adie.h` and the project's READMEs. Its measurements were taken on the
  project's own units, so they are graded `[reported]` here.
- `[TEARDOWN]`: hungvu.tech, "Quantum Fiber W1700K teardown, board view, and UART pins",
  2024-10-01. It carries part photographs, the console pinout, the stock U-Boot's commands and
  environment, and a stock boot log from firmware `WXK001-05.00.30.64`.
- `[FCC]`: Gemtek's FCC filing for FCC ID MXF-W1700K, with grants of 2023-10-06, 2024-04-09
  and 2025-03-17. Its public exhibits "Internal Photos-1", "Test Report_BLE", "MPE", "RF
  Exposure", "Test Report_Co-Location" and "Antenna Test Report-3" were read. So was the
  class II permissive change's cover letter, all from the Internet Archive's copies of
  fcc.report and fccid.io. No exhibit of the MXF-W1700K2 filing was read.
- `[FCC2]`: Gemtek's FCC filing for FCC ID MXF-W1700K2, downloaded from fccid.io. Its
  exhibits read were "Internal Photos 1" and "Internal Photos 2" (project EP560201-02) and
  "Test Report BLE" (FR560201AE, Rev. 01, 2025-09-02). Also the "Antenna Test Report" and
  the 6 GHz test report (FR560201-02AO).
- `[FCCAPI]`: the FCCID.io API's `/frequency` records for MXF-W1700K2, MXF-W1700K,
  MXF-W1701K and MXF-XR1710G, queried 2026-11-08 UTC. They are FCC-sourced grant rows held by
  FCCID.io (fccid.io), marked complete.
- `[DS]`: Silicon Labs, *EFR32MG21 Multiprotocol Wireless SoC Family Data Sheet*, Rev. 1.2,
  2024-05: the package marking key.
- `[RTSL]`: Silicon Labs, "Series 2 and Series 3 Secure Boot with RTSL", docs.silabs.com
  edition 0.3, replacing AN1218.
- `[GTX]`: gtxaspec/w1700k-efr32 at `3b9a61b`, `README.md`: measured on HW2.1, so graded
  `[reported]`.
- `[DOER]`: doer.ee, "W1700K" reverse-engineering pages, 2026-09-13: measured on the author's
  own units, so graded `[reported]`.
- `[SWB]`: hurrian/silabs-w1700k at `80ad5d5`, `manifests/gemtek/`: the community firmware
  builds' pin and clock settings.
- `[GH10]`: GitHub discussion 10 in w1700k/builds, "W1700K2 Support?", 2026-05-12 to
  2026-06-06. It carries a HW2.1 unit's stock layout and the RTL8261CE driver work on it.
- `[Z2M]`: GitHub discussion 28627 in Koenkk/zigbee2mqtt, "Quantum Fiber 360 Wi-Fi 7 Pods
  (Gemtek W1700k) - unknown protocol", 2025-09-14. It carries the EFR32's part number and its
  answer to a keypress.
- `[SKY]`: skyboooox/ImmortalWrt-Gemtek-17xx at `3530f89`,
  `docs/reference/hardware-baseline.md`: measured on an XR1710G.
- `[T<n>]`: post n of the OpenWrt forum thread "Quantum Fiber W1700k support", topic 222776,
  read 2026-11-07. Its posts carry boot logs, command output and FCC-document readings from
  posters' own units.
- `[B<n>]`: post n of the OpenWrt forum thread "Gemtek W1700K Community Builds", topic 249319,
  read 2026-11-07.
