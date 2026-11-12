# Gemtek W1700K corrections and gotchas

This document collects claims about the W1700K that published sources state wrongly, and the
board's traps. Each entry states the fact, the evidence and the rule that follows. Traps that
another document owns are listed with a link to their owner. No reference unit has been
probed yet.

## Corrected claims

| Published claim | Source | Fact | Evidence |
|---|---|---|---|
| The two RTL8261N are the "switch chipset" | [TEARDOWN] | They are the two 10G PHYs, clause 45 devices at MDIO addresses 5 and 8. The SoC's internal switch carries the 1G ports | [source][DTS] |
| HW1.1 carries RTL8261BE PHYs | [T2662] | HW1.1 carries RTL8261N. Its E2 SoC silicon sets it apart from HW1.0 | [reported][T2681][T3235] |
| `strings /dev/mtd1` gives the vendor U-Boot's version | The OpenWrt wiki | On the OpenWrt layout `mtd1` is the chainloader. The vendor U-Boot is in `mtd0` | [consensus][T3449][T3450] |
| TFTP from U-Boot works on the 10G port next to the 1G ports | The OpenWrt wiki | The vendor U-Boot reaches only the 1G ports, and on HW2.1 brings up no 10G port | [consensus][T3364][T3365][B504] |
| The W1700K2 has no GNSS receiver or EFR32 | A forum post, retracted in [T4049] | That is the Brightspeed XR1710G, which [T4049] misnames XT1700G. The XR1710G's FCC grants list no Bluetooth LE or 802.15.4 band. The W1700K2's list both, and its FCC photographs show both chips | [reported][T1969][T1974], [source][FCCAPI][FCC2] |
| The GNSS receiver supplies a PPS signal | Early posts | Its PPS pin reaches test point TP153 and no SoC pin | [reported][T4000] |
| OpenWrt's bad-block driver rewrites the vendor's table | [SDR] | OpenWrt's W1700K tree binds no bad-block driver, and maps the table's area read-only | [source][DTS][DTSI] |
| The NCT7802's `in0` reads the input power | Early posts | `in0` reads the 3.3 V rail | [reported][T3406] |
| The EFR32 is an old Series 1 radio | [T967][T3224] | It is an EFR32xG21, the first Series 2 family: a Cortex-M33 behind a Secure Engine | [source][RM] |
| universal-silabs-flasher can flash the EFR32 as shipped | [T536] | The flasher speaks only XMODEM, and the factory bootloader only BGAPI UART DFU. The first flash needs a BGAPI DFU tool | [source][USF], [consensus][GTX][DOER] |
| GPIOs 24 and 25 cannot put the EFR32 into its bootloader | [T3608] | GPIO25 held low across a GPIO24 reset pulse enters the factory bootloader | [consensus][GTX][DOER] |

### PHY IDs do not tell the RTL8261N from the RTL8261BE

The RTL8261N and RTL8261BE share PHY ID `0x001c_caf3` [reported][T2669]. MMD 30, register
`0x104`, bits 15:11, tell them apart: `0x2` is the RTL8261N and `0xe` the RTL8261BE
[reported][T2670]. Read that register before applying a model-specific fix.

### 6 GHz throughput does not depend on WED

The AN7581 offloads Wi-Fi through its NPU, so `mt7996e` runs with `wed_enable=N` by design
(see the [NPU](../network-reference.md#npu)) [consensus][B136][B438]. The router itself is a
CPU-bound test endpoint: iperf to or from it reaches about 740 Mb/s [reported][B83][B382].
Measure Wi-Fi throughput between a client and a host behind the router.

### The 1.0-layout bricks did not come from the SPI clock

Sysupgrades failed about 3% of the time under the 1.0 layout [reported][T2754]. The SPI
clock was suspected, and a 25 MHz clock stopped the 10G ports working [reported][T2821][T2850].
The cause was the vendor U-Boot rewriting its bad-block table inside UBI (see
[the vendor bad-block area](boot-and-flash.md#the-vendor-bad-block-area)) [reported][T3039].
Keep the SPI clock at the tree's 50 MHz.

## Traps

### The Airoha UART baud divider

On OpenWrt's 6.18 kernels, the `airoha,en7523-uart` ports run at the wrong rate for some
settings [reported][T3788][T3804]. Configure ttyS1 at 230400 to talk to the EFR32 at 115200
[reported][T3805]. On ttyS2, a setting of 460800 gives the GNSS receiver's 921600, and 230400
gives 230400 [reported][T3788][T3804]. A fix exists in a community tree, commit `39b52709`
[reported][T3804]. Check the rate on a scope before blaming the peripheral.

### A missing NPU firmware looks like a brick

An image without the NPU firmware waits 60 s for it at boot, and the Ethernet ports stay down
meanwhile [consensus][T3726][T3654][B77]. Wait 90 s before declaring a first boot dead. The
[network reference](../network-reference.md#npu) owns the mechanism.

### The power supply fails before the board does

Failing supplies have caused dead Wi-Fi, PCIe errors and boot loops on several units (see
[power](hardware-reference.md#power)). Swap the supply before blaming a build.

### Owned by other documents

| Trap | Owner |
|---|---|
| Data in the last 250 blocks of flash is erased at the next boot | [The vendor bad-block area](boot-and-flash.md#the-vendor-bad-block-area) |
| A `tftpboot` started late stalls | [The vendor U-Boot's networking](boot-and-flash.md#networking) |
| `dsd` is unique per unit | [Factory data](boot-and-flash.md#factory-data) |
| Sysupgrade refuses a different layout version | [Layout versions](boot-and-flash.md#layout-versions) |
| The console's pin order is disputed | [The console header](hardware-reference.md#the-console-header) |
| A GNSS strap footprint starts the receiver in download mode | [AG3352Q GNSS](hardware-reference.md#ag3352q-gnss) |
| A frequency above its voltage panics some units | [Operating points](../system-reference.md#operating-points) |
| A CPU read the endpoint never completes hard-locks the SoC | [AN7581 PCIe](../system-reference.md#pcie) |
| Reads in the MT7996's WFSYS domain wedge the chip | [MT7996 BAR0 map](../../mt7996/wifi-reference.md#bar0-map) |
| Current MT7996 ROM patches can burn an eFuse bit | [MT7996 firmware](../../mt7996/wifi-reference.md#firmware) |
| A bootloader of major version 3 blocks every Gecko SDK bootloader after it | [EFR32MG21 version numbers](../../efr32mg21/radio-reference.md#version-numbers) |
| A refused EFR32 bootloader upgrade reports success | [EFR32MG21 bootloader upgrade](../../efr32mg21/radio-reference.md#bootloader-upgrade) |
| A device erase removes the EFR32's factory bootloader | [EFR32MG21 factory firmware](hardware-reference.md#factory-firmware) |
| PTA enabled without a driven GRANT silences the EFR32 | [EFR32MG21 coexistence](../../efr32mg21/radio-reference.md#coexistence) |

## Sources

Citation keys:

- `[DTS]`: OpenWrt `target/linux/airoha/dts/an7581-w1700k-ubi.dts`, main branch at commit
  `706052b20fd1` (2026-11-07).
- `[DTSI]`: the same tree's `an7581.dtsi`: the NAND node, which carries no bad-block property.
- `[SDR]`: h0m3us3r/w1700k-sdr at commit `0b0413e`, `platform/bmt/README.md`: its account of
  OpenWrt's `mtk_bmt`.
- `[TEARDOWN]`: hungvu.tech, "Quantum Fiber W1700K teardown, board view, and UART pins",
  2024-10-01.
- `[FCC2]`: Gemtek's FCC filing for FCC ID MXF-W1700K2, downloaded from fccid.io. The
  exhibits read are "Internal Photos 1" and "Internal Photos 2" (project EP560201-02), and
  "Test Report BLE" (FR560201AE, Rev. 01, 2025-09-02). The "Antenna Test Report" and the
  6 GHz test report (FR560201-02AO) were read as well.
- `[FCCAPI]`: the FCCID.io API's `/frequency` records for MXF-W1700K2, MXF-W1700K,
  MXF-W1701K and MXF-XR1710G, queried 2026-11-08 UTC. They are FCC-sourced grant rows held by
  FCCID.io (fccid.io), marked complete.
- `[RM]`: Silicon Labs, *EFR32xG21 Wireless Gecko Reference Manual*, Rev. 1.0, 2024-01.
- `[USF]`: NabuCasa/universal-silabs-flasher v1.3.0 (`aa6a133`).
- `[GTX]`: gtxaspec/w1700k-efr32 at `3b9a61b`, `README.md`: measured on HW2.1.
- `[DOER]`: doer.ee, "W1700K" reverse-engineering pages, 2026-09-13: measured on the author's
  own units.
- `[T<n>]`: post n of the OpenWrt forum thread "Quantum Fiber W1700k support", topic 222776,
  read 2026-11-07.
- `[B<n>]`: post n of the OpenWrt forum thread "Gemtek W1700K Community Builds", topic 249319,
  read 2026-11-07.
