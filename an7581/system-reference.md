# AN7581 system reference

This document covers the Airoha AN7581's processor, boot security, memory, address map, UARTs,
pin mux, PCIe, thermal sensors and crypto engines. The reader is writing a device tree, a
bootloader port or a driver for an AN7581 board. The [network reference](network-reference.md)
covers the packet path: the GDMs, QDMA, the switch, the PCS blocks, the PPE and the NPU.

Conditions: the facts rest on one board, the Gemtek W1700K, and on OpenWrt's `an7581.dtsi`.
No reference unit has been probed yet. Every claim is `[source]`, `[consensus]` or
`[reported]`.

## Overview

| Property | Value |
|---|---|
| Part | Airoha AN7581, formerly EcoNet EN7581. Seen as AN7581GT |
| CPU | 4x Arm Cortex-A53 r0p4, one cluster, 1200 MHz |
| Interrupt controller | GICv3 |
| Memory | DDR4, from `0x8000_0000` |
| Packet engines | Frame engine, 2x QDMA, PPE, NPU, GbE switch |
| PCIe | Three controllers. Two can pair as one x2 port |
| Crypto | Inside Secure EIP93, an inline IPsec engine, a TRNG |
| Storage controllers | SPI NAND (SNFI), eMMC |
| USB | Two xHCI controllers |

Sources: [source][DTSI], [consensus][T50][T2701][B166].

OpenWrt's trees declare `compatible = "airoha,an7581", "airoha,en7581"` [source][DTSI]. The
vendor's own tree uses `econet,en7581` [reported][T30]. EcoNet's EN7529CT datasheet, version
1.3, describes an earlier revision of the part [reported][T657].

## Silicon revisions

| Product ID | Revision | Seen on |
|---|---|---|
| 1 | Before E2 | W1700K HW1.0 |
| 2 | E2 | W1700K HW1.1 and HW2.1 |

The product ID is at `0x1fb0_005c`. Linux reads it with `devmem 0x1fb0005c`, and the vendor
U-Boot with `md.l 0x1fb0005c 1` [reported][T3235][T3247][T3331]. E2 is a fixed base silicon
[reported][T3816]. The receive calibration a 10G PCS needs depends on the revision (see the
[network reference](network-reference.md#pcs-and-serdes)).

The suffix grades the CPU clock: an AN7581GT is rated 1.3 GHz and an AN7581PT 1.2 GHz
[reported][T3816]. BL2 prints `pkg_type is BGA1` [consensus][T153][T2743].

## CPU

| Property | Value |
|---|---|
| Cores | 4x Cortex-A53, `MIDR_EL1` `0x410f_d034` (r0p4) |
| Entry | Every core enters Linux at EL2 |
| L1 caches | 32 KiB instruction, 32 KiB data, per core |
| L2 | 512 KiB in the tree. The vendor firmware splits it into 256 KiB of cache and 256 KiB of SRAM |
| Timer | Arm generic timer at 25 MHz |
| PSCI | 1.1, through SMC |
| Crypto extensions | Absent |

Sources: [consensus][T50][T143][T3453], [source][DTSI], [reported][T3492]. The vendor firmware
prints `L2C_type:256K_L2C+256K_SRAM` [consensus][T143][TEARDOWN].

### Operating points

The tree gives 15 operating points, 500 to 1200 MHz in 50 MHz steps. Each maps to a level of
the SMCC power domain `airoha,en7581-cpufreq` [source][DTSI]. BL31 holds each level's voltage
in its own table [reported][YMXS]. The power-domain driver computes the frequency as
500 + 50 x level MHz [reported][YMXS].

OpenWrt's 6.18 kernels fail to register the frequency driver. They log
`ATF SMC not available, using direct PLL programming` and then `CPU frequency: 1200 MHz`
[consensus][T2979][T3311][B166]. The cores therefore run at 1200 MHz, with no scaling, as of
2026-11 [consensus][T4046].

CoreMark scores 4074 at 1200 MHz, 3395 per GHz [reported][B487].

> [!CAUTION]
> **Raising a frequency without raising its voltage panics some units.** A table shifted
> 200 MHz up leaves each frequency on the voltage of the one 200 MHz below. Panics on such
> builds stopped once the CPU was capped at 1200 MHz [reported][YMXS]. One HW1.1 unit ran at
> 1.3 GHz and crashed at 1.35 GHz [reported][T3046]. One HW2.1 unit ran at 1.4 GHz and
> panicked at 1.5 GHz [reported][T3338].

## Boot security

The boot ROM loads BL2 from the start of flash. BL2 prints `Secure key exist` on parts whose
secure-boot key is fused, and the boot ROM then runs a BL2 signed with that key only
[consensus][T143][T472][T565]. BL31 verifies the vendor's firmware images against a SHA-512
digest of the root key, held in eFuse [source][SDR].

The boot ROM also has an XMODEM loader. Holding GPIO0 low at power-on starts it: the console
prints `Press x`, then `CCCC` [reported][T511][T565]. With the key fused, the loader accepts
only a FIP whose BL2 carries the key's signature. The FIP sits in the first 512 KiB of flash,
from offset 2 KiB [reported][T565].

## Memory

The SoC addresses DRAM from `0x8000_0000` [source][DTSI]. The tree reserves these regions on
every AN7581 board [source][DTSI]:

| Region | Start | Size | Holder |
|---|---|---|---|
| `atf` | `0x8000_0000` | 2 MiB | BL31 |
| `npu-binary` | `0x8400_0000` | 10 MiB | The NPU's firmware |
| `qdma0-buf` | `0x8700_0000` | 32 MiB | QDMA0's buffers |
| `qdma1-buf` | `0x8900_0000` | 16 MiB | QDMA1's buffers |

A board with an NPU-offloaded Wi-Fi chip reserves four more regions, 110 MiB in all (see the
[NPU](network-reference.md#npu)). Linux's kernel image loads at `0x8020_0000`
[reported][YMXS][T3453].

## Address map

The map lists the blocks OpenWrt's tree describes [source][DTSI]:

| Block | Base | Notes |
|---|---|---|
| GIC distributor | `0x0900_0000` | Redistributors at `0x0908_0000` |
| NPU | `0x1e90_0000` | 0x31_3000 bytes |
| Thermal sensor | `0x1efb_d000` | |
| eMMC | `0x1fa0_e000` | |
| PCS: PCIe, USB, PON, Ethernet | `0x1fa0_4000` to `0x1fa0_9000` | See the [network reference](network-reference.md#pcs-and-serdes) |
| SPI NAND (SNFI) | `0x1fa1_0000`, `0x1fa1_1000` | |
| Chip SCU and pin mux | `0x1fa2_0000` | |
| PCIe 2-lane PHY | `0x1fa5_a000` | Shared by pcie0 and pcie1 |
| TRNG | `0x1faa_1000` | |
| USB0 xHCI, PHY | `0x1fab_0000`, `0x1fac_0000` | |
| USB1 xHCI, PHY | `0x1fad_0000`, `0x1fae_0000` | |
| SCU: clocks and resets | `0x1fb0_0000` | |
| Frame engine | `0x1fb5_0000` | QDMA0 `0x1fb5_4000`, QDMA1 `0x1fb5_6000` |
| Switch | `0x1fb5_8000` | |
| EIP93 crypto | `0x1fb7_0000` | |
| hsuart3 | `0x1fbe_1000` | |
| Audio front end | `0x1fbe_2200` | |
| uart1 | `0x1fbf_0000` | |
| Watchdog | `0x1fbf_0100` | |
| GPIO, PWM | `0x1fbf_0200` | |
| uart2 | `0x1fbf_0300` | |
| uart4, uart5 | `0x1fbf_0600`, `0x1fbf_0700` | |
| i2c0, i2c1 | `0x1fbf_8000`, `0x1fbf_8100` | |
| pcie0, pcie1, pcie2 | `0x1fc0_0000`, `0x1fc2_0000`, `0x1fc4_0000` | |

## UARTs

| UART | Base | Driver | Clock |
|---|---|---|---|
| uart1 | `0x1fbf_0000` | `ns16550` | 1.8432 MHz |
| uart2 | `0x1fbf_0300` | `airoha,en7523-uart` | 7.3728 MHz |
| hsuart3 | `0x1fbe_1000` | `airoha,en7523-uart` | 7.3728 MHz |
| uart4 | `0x1fbf_0600` | `airoha,en7523-uart` | 7.3728 MHz |
| uart5 | `0x1fbf_0700` | `airoha,en7523-uart` | 7.3728 MHz |

Sources: [source][DTSI].

> [!CAUTION]
> **The `en7523-uart` ports run at twice the requested baud on OpenWrt 6.18.** Asking for
> 230400 gives 115200 on uart2, and asking for 460800 gives 921600 on hsuart3
> [reported][T3788][T3804]. Set the divider for the rate the line needs.

## GPIO and pin mux

The GPIO controller and the PWM share the block at `0x1fbf_0200`. Inputs read at
`0x1fbf_0204` for GPIO0-31 and `0x1fbf_0270` for GPIO32-63 [reported][T3983]. A fix for
edge-triggered GPIO interrupts entered Linux 6.18.52 [reported][T3983].

The chip SCU's register at `0x1fa2_0214` selects pin functions. Its reset value `0x0000_4028`
enables the MDIO master and the LED functions of two switch ports [reported][T3737]:

| Bit | Function |
|---|---|
| 3 | uart2 |
| 5 | hsuart |

Posters enabled the HSUART for a GNSS receiver by writing `0x4000` or `0x4020` to it with
`devmem` [reported][T3731][T3740]. Bit 3 of the register at `0x1fb0_009c` routes the USB2
SerDes. It must be 0 for pcie2 to use that SerDes [reported][T276][T280].

## PCIe

| Controller | Base | Memory window | PHY |
|---|---|---|---|
| pcie0 | `0x1fc0_0000` | `0x2000_0000`, 64 MiB | The 2-lane PHY, lane 0 |
| pcie1 | `0x1fc2_0000` | `0x2400_0000`, 64 MiB | The 2-lane PHY, lane 1 |
| pcie2 | `0x1fc4_0000` | `0x2800_0000`, 64 MiB | The USB1 PHY's USB 3 SerDes |

Sources: [source][DTSI][SDR].

pcie0 can take pcie1's MAC as its second lane, as one x2 port. The board's tree then names
both MACs and both reset outputs on pcie0, and disables pcie1 [source][DTS]. The root ports
read `14c3:6899` [consensus][T2701][B166].

PCIe DMA is not cache-coherent: no `dma-coherent` property, no coherent interconnect
[source][SDR]. Software must invalidate each receive buffer before the CPU reads it.

The root ports' Max Payload Size is 128 bytes [reported][SDR]. ASPM L0s and L1 are on by
default. With them on, an MT7996's DMA into DRAM reached 0.32 GB/s, against 1.44 GB/s with
ASPM off [reported][SDR].

A pcie0 x2 link trains at Gen2 under OpenWrt's 6.18 kernels:
`8.000 Gb/s available ... limited by 5.0 GT/s PCIe x2 link` [consensus][T3014][T3683]. A
6.12 kernel retrained it to Gen3 by toggling the SerDes [reported][T3453].

> [!CAUTION]
> **A CPU read that the endpoint never completes hard-locks the SoC.** No panic and no AER log
> follow, even with completion timeouts enabled on both ends [reported][SDR]. Keep register
> reads to an endpoint's documented addresses.

The PCIe reset outputs (`PERSTOUT`) have a dedicated control in the SCU. For the PCIe
controller's reset logic, 0 asserts and 1 releases [source][ACK72].

## Thermal

The thermal block `airoha,en7581-thermal` sits at `0x1efb_d000`. Its sensor 0 reads the main
ADC and sensor 1 the CPU [source][DTSI]. The `cpu-thermal` zone's trips [source][DTSI]:

| Trip | Temperature | Action |
|---|---|---|
| Hot | 95 °C | Cools all four cores |
| Critical | 120 °C | Shuts down |

On the W1700K the CPU zone idles about 41 °C, and a full load on four cores raises it less than
1 °C [reported][SDR]. At 1.4 GHz under the `performance` governor it read 62-63 °C
[reported][YMXS].

## Crypto

### EIP93

The Inside Secure EIP93 at `0x1fb7_0000` is a look-aside engine, `inside-secure,safexcel-eip93ies`,
on SPI 44 [source][DTSI]. Linux identifies it as `EIP93:3:0:1,PE(0xfe007:0x41710041)`
[consensus][T50][B372].

Through AF_ALG, AES-128-CBC reaches about 101 MB/s with 16 KiB requests [reported][B374].
Software is faster below 512-byte requests, and for every hash [reported][B374]. OpenSSL's
AES-128-CTR runs 126.9 MB/s on the engine and 67.9 MB/s in software, at 16 KiB
[reported][T3863].

### Inline IPsec engine

A second engine processes IPsec ESP inline with the frame engine. The vendor tree names it
`ipsec@1fbfa000`, `econet,ecnt-ipsec`, on SPI 79 [reported][T3571]. It handles tunnel mode only,
at about 4-5 Gb/s [reported][T3571][B343]. It exposes no raw cipher, so it cannot accelerate
WireGuard [reported][B346][B354]. An RFC driver was posted to netdev on 2026-06-14
[reported][T3668].

### TRNG

The TRNG at `0x1faa_1000` runs in DRBG mode under Linux [reported][B166].

## Sources

Citation keys:

- `[DTSI]`: OpenWrt `target/linux/airoha/dts/an7581.dtsi`, main branch at commit
  `706052b20fd1` (2026-11-07).
- `[DTS]`: the same tree's `an7581-w1700k-ubi.dts`.
- `[SDR]`: h0m3us3r/w1700k-sdr at commit `0b0413e`: `include/hw/an7581/pcie.h`, `memmap.h`
  and `w1700k_board.h`. Its measurements were taken on the project's own W1700K units, so
  they are graded `[reported]` here.
- `[ACK72]`: ACK72/openwrt-w1700k-builds at commit `2ca01b7`, the description of
  `patches/w1700k/0004-airoha-initialize-perstout-reset-value.patch`.
- `[YMXS]`: ymxs/w1700k-openwrt at commit `fda97fb`, `BUG_REPORT.md`: the CPU table, the
  power-domain formula, three kernel panics and the thermal readings.
- `[TEARDOWN]`: hungvu.tech, "Quantum Fiber W1700K teardown, board view, and UART pins",
  2024-10-01: a stock boot log.
- `[T<n>]`: post n of the OpenWrt forum thread "Quantum Fiber W1700k support", topic 222776,
  read 2026-11-07.
- `[B<n>]`: post n of the OpenWrt forum thread "Gemtek W1700K Community Builds", topic 249319,
  read 2026-11-07.
