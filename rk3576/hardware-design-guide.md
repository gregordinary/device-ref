# RK3576 hardware design guide

This document covers the parts of Rockchip's RK3576 Hardware Design Guide that change a
software decision [HDG]. The guide (V1.1, 2024-05-28, 245 pages) is written for schematic
and PCB engineers. Most of it has no software consequence and is not repeated here:
stackups, decoupling, trace impedance and reflow profiles, for example. What remains
constrains a device tree, names a pin's alternatives or decides a bring-up test. The
document is SoC-level, and it cites the guide by section number.

## IO power domains

The RK3576 detects the voltage of its configurable IO domains in hardware, so a board
device tree carries no `io-domain` node. `PMUIO1` and `VCCIO1`-`VCCIO6` are each strapped
to 1.8 V or 3.3 V on the board. The guide states that "the RK3576 chip can automatically
detect the hardware-configured voltage, so there is no need for software configuration"
[HDG 2.1.11.4]. `PMUIO0` and `VCCIO0` are fixed at 1.8 V and not configurable at all.
`VCCIO7` is 1.2 V or 1.8 V.

Mainline carries no IO-domain support for the RK3588 either. Linux 7.2.9's `io-domain.c`
matches no RK3588 compatible, and no mainline RK3588 board tree carries an IO-domain node
[source]. On either SoC a mainline board `.dts` correctly carries none, and its absence
explains no peripheral fault.

What still matters to software is which domain a pin's mux alternative sits in, because
that fixes the pin's signaling voltage. The alternatives are not interchangeable, and
several peripherals are muxed across two domains.

## Pin-mux alternatives across power domains

Where a function appears at more than one pin, the alternatives usually sit in different
IO domains. Choosing the wrong alternative can then mean a wrong voltage as well as a wrong
pin. The board's schematic decides, and a vendor device tree that boots is the practical
authority.

| Function | M0 | M1 | M2 | M3 | [HDG] |
|---|---|---|---|---|---|
| `HDMI_TX_HPDIN` | VCCIO6 | PMUIO1 | | | 2.3.8.1 |
| `HDMI_TX_CEC` | VCCIO6 | PMUIO1 | | | 2.3.8.1 |
| `HDMI_TX_SCL`, `HDMI_TX_SDA` | VCCIO6 | | | | 2.3.8.1 |
| `GMAC0` | VCCIO5 | VCCIO4 | | | 2.3.11 |
| `GMAC1` | VCCIO4 | VCCIO3 | | | 2.3.11 |
| `SAI0` | VCCIO4 | PMUIO1 | VCCIO0 | | 2.3.9.1 |
| `SAI1` | VCCIO2 | VCCIO5 | | | 2.3.9.1 |
| `SAI2` | VCCIO3 | VCCIO4 | VCCIO5 | | 2.3.9.1 |
| `SAI3` | VCCIO0 | VCCIO3 | VCCIO4 | VCCIO1 | 2.3.9.1 |
| `SAI4` | VCCIO2 | VCCIO5 | VCCIO6 | VCCIO4 | 2.3.9.1 |
| `SPDIF_TX0` | VCCIO2, `GPIO4_B5` | VCCIO5, `GPIO4_A1` | VCCIO4, `GPIO2_D7` | | 2.3.9.5.1 |
| `SPDIF_TX1` | VCCIO5, `GPIO3_B5` | VCCIO4, `GPIO3_A3` | VCCIO3, `GPIO1_D5` | | 2.3.9.5.2 |
| `SPDIF_RX0` | VCCIO2, `GPIO4_B4` | VCCIO5, `GPIO4_A0` | VCCIO4, `GPIO2_D6` | | 2.3.9.5.3 |
| `SPDIF_RX1` | VCCIO5, `GPIO3_B4` | VCCIO4, `GPIO3_A2` | VCCIO3, `GPIO1_D4` | | 2.3.9.5.4 |
| `DSM` | VCCIO1 | VCCIO6 | | | 2.3.9.1 |

`GMACx_M0` and `GMACx_M1` cannot be used simultaneously, so a board uses one set at a time.
The guide recommends 1.8 V for RGMII and RMII, for signal quality.

## HDMI sideband circuits

None of the three HDMI sideband signals connects straight to an SoC pin [HDG 2.3.8.1].
Each passes through a discrete circuit:

- HPD: `MOS isolated conversion`. The connector's HPD is multiplexed with `HDMI_TX_SBDN`,
  and insertion drives the SoC pin high.
- CEC: `MOS isolated conversion`, and the isolation is mandatory. The CEC protocol applies
  3.3 V through a 27 kΩ resistor and forbids more than 1.8 µA of leakage. With the board
  powered off and a live TV on the cable, an unisolated CEC pin leaks past that limit. The
  guide specifies 27 kΩ, not to be changed, plus a 2SK3018-class FET chosen for its
  junction capacitance.
- DDC (`SCL` and `SDA`): `MOS level conversion`, because DDC is a 5 V bus and RK3576 IO is
  not 5 V tolerant. A series diode stops the sink leaking back into `VCC5V0_DEVICE_S0`.

`HDMI_TX_ON_H` is the TMDS bias switch, not a power enable. Driven low, it keeps the four
series FETs off, and the link runs HDMI 2.1 FRL. Driven high, the FETs conduct. The 590 Ω
pulldowns and the sink's 50 Ω pullups then form the ~3 V DC bias that TMDS at HDMI 2.0
and below needs. Cost-reduced boards that only ever run TMDS can delete the FETs and tie
the 590 Ω resistors to ground, which makes the pin inert.

Mainline models this pin as `frl-enable-gpios`. The binding defines it as asserted in
HDMI 2.1 FRL mode and deasserted for TMDS [source]. In the reference circuit the pin is low
in FRL mode, so a mainline device tree describes it as `GPIO_ACTIVE_LOW`.

`HDMI/EDP_TX_REXT` needs an 8.2 kΩ 1% resistor to ground. No software setting compensates
for a wrong value.

The controller-side HPD registers are listed under
[VO0 GRF display fields](display-reference.md#vo0-grf-display-fields).

## USB controllers and combo PHY 1

The RK3576 has two USB3 OTG controllers, each with an embedded USB 2.0 OTG [HDG 2.3.4].
That is the whole USB complement. A board with more than two USB ports has a hub.

OTG0 pairs with the USB3/DP combo PHY (`0x2b010000`). SuperSpeed and DP share its lanes,
giving five board configurations, from Type-C 4-lane with DP down to USB 2.0 only. The
BootROM downloads firmware over `USB2_OTG0_DP/DM`. Every design must therefore reserve that
interface, even where the product does not expose it, and connect `USB2_OTG0_VBUSDET` with
it. Firmware download over USB 3.0 uses `SSRX1` and `SSTX1`. That requires the 2-lane DP,
swap-on configuration (`rockchip,dp-lane-mux = <2 3>`).

OTG1 pairs with combo PHY 1 (`0x2b060000`), which the SoC shares between PCIe1, SATA1 and
USB3 OTG1. The coupling reaches beyond the SuperSpeed lanes:

<!-- prose-lint: off -- a verbatim quotation from the design guide. -->

> "When the COMBO PHY1 of PCIE1/SATA1/USB3 OTG1 is set to PCIe or SATA functionality, USB3
> OTG1 function cannot be used, and USB2.0 PHY1 also cannot be used. Therefore, to use
> USB2.0 OTG1, the COMBO PHY1 of PCIE1/SATA1/USB3 OTG1 must be set to USB3 functionality!"
> [HDG 2.3.4.2]

<!-- prose-lint: on -->

By the guide, combo PHY 1 in USB3 mode gates USB 2.0 PHY1 as well as OTG1's SuperSpeed.
Disabling `combphy1_psu` therefore risks taking OTG1's USB 2.0 port with it. This
dependency is the guide's statement about the SoC, and it is not measured [source]. The
guide is a design document rather than a driver reference. Whether mainline's naneng
combphy driver leaves the mux in a working state with the node absent is untested.

On mainline the PHY must also stay in the `phys` of OTG1's controller (`usb_drd1_dwc3`).
Removing it makes the board unbootable, with an SError in `hub_activate` at ~1.85 s
[measured, 2026-08-08].
[Capping a dwc3 port to High Speed](corrections-and-gotchas.md#capping-a-dwc3-port-to-high-speed)
gives the mechanism. Keep `combphy1_psu` enabled and in that controller's `phys`, whether
or not its lane reaches a connector.

On a board whose lane reaches no connector, the SuperSpeed port never trains. Combo PHY 1
then logs `2b060000.phy: wait phy status ready timeout` on every boot, and the line is
cosmetic [measured, 2026-08-08].

Other pin-level facts with software visibility:

- `USB2_OTGx_ID` has a ~12 kΩ internal pullup to `USB2_OTG_AVDD1V8`. High selects device,
  and pulled low selects host.
- `USB2_OTGx_VBUSDET` has a 40 kΩ internal pulldown. High (2.7-3.3 V) means device mode.
  Device mode also requires it above 2.3 V before the controller pulls DP high to
  enumerate.
- A port starts in device mode before u-boot and is reconfigured afterwards.
- A dwc3 runs SuperSpeed and its embedded USB 2.0 in the same role. One cannot be host
  while the other is device.

## SARADC boot strap and recovery key

SARADC channel 0 is the boot strap, and channel 1 is the recovery key [HDG 2.3.2]. Both are
fixed functions.

`SARADC_IN0_BOOT` is a resistor divider read at reset, and its value selects the BootROM's
boot order. Reading channel 0 on a running system therefore reports the order the board is
strapped to:

| Rup (kΩ) | Rdown (kΩ) | ADC | Boot order |
|---|---|---|---|
| NC | 10 | 0 | USB (maskrom) |
| 10 | 1.13 | 416 | FSPI0 -> USB |
| 10 | 2.49 | 816 | FSPI1_M0 -> eMMC -> USB |
| 10 | 4.3 | 1231 | FSPI1_M1 -> eMMC -> USB |
| 10 | 6.8 | 1658 | FSPI0 -> UFS -> USB |
| 10 | 10 | 2048 | FSPI1_M0 -> UFS -> USB |
| 10 | 14.7 | 2437 | UFS -> USB |
| 10 | 23.2 | 2862 | UFS -> SDMMC0 -> USB |
| 10 | 40.2 | 3279 | Reserved |
| 10 | 88.7 | 3680 | eMMC -> SDMMC0 -> USB |
| 10 | NC | 4095 | eMMC -> USB |

`SARADC_IN1` is the recovery-mode key, and it cannot be remapped. It idles high through a
10 kΩ 1% pullup to `VCCA_1V8_S0`. Held low at startup, it puts the SoC in loader-flashing
mode. Releasing it once the PC has enumerated the device lets the flash proceed. The guide
warns against reducing the pullup, because a floating `IN1` can disturb the boot decision.

What the key does at power-on is the boot firmware's decision. Mainline u-boot needs two
fixes on the RK3576 before the key reaches maskrom, as
[The SARADC download key](boot-firmware.md#the-saradc-download-key) describes.

For key arrays on the remaining channels, the guide asks for more than 300 counts (more
than 132 mV) between adjacent key values.

## Thermal

The SoC has one internal T-sensor, and the platform's default governor is
`power_allocator` [HDG 4.2]. The guide adds nothing else that software acts on, and it
defers configuration to an SDK document.

## Sources

- [HDG]: Rockchip, RK3576 Hardware Design Guide V1.1, 2024-05-28, cited by section number.
  `pdftotext -layout` extracts the PDF.
- Linux at tag `v7.2.9`: `drivers/soc/rockchip/io-domain.c` and the RK3588 board trees,
  `arch/arm64/boot/dts/rockchip/rk3588*.dts*`.
- The reference unit: an H96 MAX M9. The combo PHY 1 log line and the SError on removing
  that PHY from `phys` are measured on it.
