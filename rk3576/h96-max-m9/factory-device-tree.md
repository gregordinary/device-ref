# H96 MAX M9 factory device tree

This document describes the device tree that the H96 MAX M9 ships with, at board level. It
states what the factory tree contains and how it differs from mainline
`rk3576-evb1-v10.dts` in the 7.1.3 tree. It serves an engineer writing a mainline device
tree for this board. The [hardware reference](hardware-reference.md) states what the board
is, and the [corrections and gotchas](corrections-and-gotchas.md) name the values that
community sources misreport.

## The EVB1 base

The vendor ships the H96 with Rockchip's EVB1 V10 device tree, lightly modified [DTB][LOG].
The tree self-identifies as "Rockchip RK3576 EVB1 V10 Board". It still carries the EVB's
IMX415 camera-sensor nodes (24 references) and its ES8388 codec.

The factory tree is therefore authoritative only for the values the vendor changed for the
board. Those are the Ethernet timing, the board GPIOs, the HDMI enable pin and the Wi-Fi
type. Its
EVB-carryover nodes, the cameras and the ES8388 among them, do not describe the board.
Mainline `rk3576-evb1-v10.dts` is already close to what the H96 needs, so a board tree
derived from it is small. The [board deltas](#board-deltas-against-the-evb1-tree) list
the differences.

## The three audio cards

The factory tree declares three sound cards, all `status = "okay"` [DTB]. Two of them bind
the same CPU DAI (`sai@2a610000`, SAI1, phandle `0x1a6`), so only one of those two can
register. The ALSA device list shows which:

| DT node | `compatible` | Card name string | Codec | Registers |
|---|---|---|---|---|
| `es8388-sound` | `rockchip,multicodecs-card` | `rockchip-es8388` (hyphen) | `es8388@10` on i2c3 | No |
| `ht5010-sound` | `simple-audio-card` | `rockchip,es8388` (comma) | `ht5010_sound-out`, a `rockchip,dummy-codec` | Yes |
| `spdif-tx0-sound` | `simple-audio-card` | `rockchip,spdif-tx1` | `linux,spdif-dit` | Yes |

Six independent vendor boot captures (`h96-boot-01` to `h96-boot-06`) print the identical
list [measured]:

```
ALSA device list:
  #0: rockchip-hdmi
  #1: rockchip,es8388
  #2: rockchip,spdif-tx1
```

Registration of the multicodecs card, `rockchip-es8388`, is the only proof that an ES8388
bound. That card appears in no capture, and no ES8388, i2c-codec or probe-defer line
appears either. The name `rockchip,es8388` on card #1 is the vendor's
`simple-audio-card,name` on the HT5010 node, not an identification of the part. A comma in
place of a hyphen is the whole difference between the two strings.

This matches two measurements on the reference unit: i2c3 is electrically wedged, and no
address on it answers [measured]. The [audio section of the hardware reference](hardware-reference.md#audio)
records both. Nothing talks to the ES8388 node under the vendor OS either, so its silence
under mainline is not a mainline gap. The board's analog path is SAI1 -> HT5010.

The HT5010 card carries three details that a mainline tree matches:

- `clocks = <&mclkout_sai1>` with `pinctrl-0 = <&sai1m0_mclk>` on the codec node. The
  amplifier receives MCLK even though it has no control interface.
- SAI1 muxes `sai1m0-lrck`, `-sclk`, `-sdi0` and `-sdo2`, where the EVB uses `sdo0`.
- No amplifier-enable GPIO sits on the path. The `spk-con-gpio` (gpio2 PB1) and
  `hp-con-gpio` (gpio3 PD6) properties sit on `es8388-sound`, the card that never probes,
  so the vendor audio stack never drives those pins.

Under mainline the jack plays without either amplifier pin. Driving both pins high changes
nothing, and the jack plays once `SAI_PATH_SEL` routes a stream to `sdo2`
[measured, 2026-07-30]. The [audio section of the hardware reference](hardware-reference.md#audio)
lists the settings a mainline tree needs.

## HDMI sideband mux

The `pinctrl-0 = <0xde 0xdf 0xe0>` of `hdmi@27da0000` resolves to three groups [DTB]:

- `hdmi_txm0-pins`: gpio4 pins 16 and 17, carrying CEC and HPD, function 9
- `hdmi-tx-scl`: gpio4 pin 18
- `hdmi-tx-sda`: gpio4 pin 19

The `hdmi_txm1-pins` alternative (gpio0 pins 19 and 14) is defined but unreferenced.

The choice matters because HPD and CEC each exist in two mux locations, in different IO
power domains [HDG]. M0 sits in VCCIO6 and M1 in PMUIO1. The two are not interchangeable, as
[HDMI sideband circuits](../hardware-design-guide.md#hdmi-sideband-circuits) describes.

A mainline tree for this board uses M0. It names all three groups explicitly alongside its
own `hdmi_frl_en` pin, because a board-level `pinctrl-0` replaces the inherited value rather
than extending it. Listing only the FRL pin unmuxes HPD and DDC. The mechanism is described
under
[a board-level pinctrl-0 replaces the inherited groups](../corrections-and-gotchas.md#a-board-level-pinctrl-0-replaces-the-inherited-groups).

One vendor-only property on the same node has no mainline equivalent:
`rockchip,sda-falling-delay-ns = <360>`, a DDC timing adjustment. Mainline's
`dw_hdmi_qp-rockchip.c` reads no such property [source].

## The HDMI data-line bias pin

The factory tree names gpio2 PB0 `enable-gpios` on `hdmi@27da0000`, active-high [DTB]. The
vendor 6.1 driver (`dw_hdmi-rockchip.c`) drives it low for FRL and high for TMDS [source]:

```c
gpiod_direction_output(hdmi->enable_gpio, hdmi->link_cfg.frl_mode ? 0 : 1)
```

Without the pin, the vendor driver refuses modes above 600 MHz. The pin is therefore the
external bias switch for the HDMI data lines, not a power enable.

Mainline's `dw_hdmi_qp-rockchip.c` models that pin as the optional `frl-enable-gpios`
[source]. The binding `rockchip,rk3588-dw-hdmi-qp.yaml`, which `rockchip,rk3576-dw-hdmi-qp`
shares, describes it this way:

> asserted when operating in HDMI 2.1 FRL mode and deasserted for HDMI 1.4/2.0 TMDS ...
> used to control external voltage bias for HDMI data lines

Mainline asserts the line in FRL mode, where the vendor drives it physically low, so the
logical sense inverts. A mainline tree describes the pin as:

```dts
frl-enable-gpios = <&gpio2 RK_PB0 GPIO_ACTIVE_LOW>;
```

Mainline has no `enable-gpios` property on this node. Setting one is silently ignored and
fails `dtbs_check`. The [corrections and gotchas](corrections-and-gotchas.md) record the
inverted polarity as a correction.

## USB controllers

The factory tree runs both DWC3 controllers as hosts with a SuperSpeed PHY [DTB]:

| Node | `dr_mode` | `phys` | SuperSpeed provider |
|---|---|---|---|
| `usb@23000000` (drd0) | `host` | u2phy0 `otg-port` + usbdp `u3-port` | usbdp `2b010000`, `rockchip,dp-lane-mux = <2 3>` |
| `usb@23400000` (drd1) | `host` | u2phy1 `otg-port` + combphy `<&combphy1 PHY_TYPE_USB3>` | naneng combphy `2b060000` |

Both controllers are `okay`, and the `u3-port` and `dp-port` of `phy@2b010000` are both
enabled. The `<2 3>` lane mux is the design guide's "USB3.2 Gen1x1 OTG0 + DP 2Lane, Swap ON"
configuration. It is the only configuration that also permits firmware download over
USB 3.0 [HDG].

The factory tree sets `snps,dis_rxdet_inp3_quirk` (plus `dma-coherent`) on drd1 only, and
drd0 reaches SuperSpeed without it. A mainline drd0 also reaches SuperSpeed on the usbdp PHY
with the same `<2 3>` lane mux, as the [USB section of the hardware reference](hardware-reference.md#usb)
records.

drd1 keeps its SuperSpeed PHY (`combphy1`) in the factory tree, although the lane reaches no
connector on this board. A mainline tree keeps it as well, because dropping it makes the
board unbootable. The mechanism is described under
[capping a dwc3 port to High Speed](../corrections-and-gotchas.md#capping-a-dwc3-port-to-high-speed).

## The wlan-platdata label

The factory tree's `wlan-platdata` node reports `wifi_chip_type = ap6275p`, which reads as
a Broadcom part [DTB]. The boot log's SDIO enumeration reports `vid:0xC8A1` (AICSemi), and
the aic8800 driver loads AIC8800D80 firmware (`fw_..._8800d80_u02.bin`) [LOG]. Broadcom's
SDIO vendor ID is `0x02D0`, and no `brcmfmac` or `bcm` line appears.

The reference unit's own factory tree also reports `ap6275p` [measured]. On that unit the
module silkscreen reads `L8800D 80 WIFI&BT`, and the serial boot enumerates
`vid:0xC8A1 did:0x0082`. The `aic8800_bsp` driver binds there as well.

The label is vendor boilerplate, so identify the Wi-Fi part by its SDIO vendor ID.
[Wi-Fi and Bluetooth](wifi-bluetooth.md) covers the radio under mainline.

## Vendor regulator messages

The vendor 6.1 boot log reports a missing VOP regulator and falls back to a dummy GMAC `phy`
supply [LOG]. Both messages are properties of the vendor drivers, as
[vendor-driver log lines are not mainline requirements](../corrections-and-gotchas.md#vendor-driver-log-lines-are-not-mainline-requirements)
explains. No source names the rail that feeds this board's Ethernet PHY, and the dummy
regulator is benign. A mainline tree for this board therefore leaves `phy-supply` unset.

## Board deltas against the EVB1 tree

The [GPIO banks](hardware-reference.md#gpio-banks) table gives the bank base addresses
from `rk3576.dtsi`, where gpio0 sits in the PMU domain. The address `0x2ae00000` is the
SARADC, not a GPIO bank. The factory tree encodes each pin as an offset within its bank:

| Pin | Offset |
|---|---|
| PB0 | `0x08` |
| PB1 | `0x09` |
| PB3 | `0x0b` |
| PB4 | `0x0c` |
| PC6 | `0x16` |
| PC7 | `0x17` |
| PD0 | `0x18` |
| PD1 | `0x19` |
| PD2 | `0x1a` |
| PD3 | `0x1b` |
| PD6 | `0x1e` |

The table below compares each board value with mainline `rk3576-evb1-v10.dts`, and names
what a mainline tree for this board carries [DTB][DTSI]:

| Item | Mainline `rk3576-evb1-v10.dts` | Factory H96 tree | Mainline H96 tree |
|---|---|---|---|
| GMAC0 `tx_delay` | `0x21` | `0x1b` | `0x1b` |
| GMAC0 `phy-mode` | `rgmii-rxid` | `rgmii-rxid` | Inherited |
| GMAC0 PHY reset (on `rgmii_phy0`) | gpio2 RK_PB5, active-low | gpio2 RK_PB3, active-low | gpio2 RK_PB3, active-low |
| HDMI data-line bias | None | gpio2 RK_PB0 | `frl-enable-gpios`, active-low, see [the HDMI data-line bias pin](#the-hdmi-data-line-bias-pin) |
| Speaker-amplifier enable | gpio2 RK_PB1, active-high (`spk_power`) | gpio2 RK_PB1, active-high | Not needed, see [the gpio2 pin map](#the-gpio2-pin-map) |
| LEDs labeled green and red | None | gpio0 RK_PD1 / RK_PD0, active-low, green default-on | Adds `gpio-leds` |
| Power key | None | gpio0 RK_PD3, `KEY_POWER`, active-low | Adds `gpio-keys` |
| eMMC `2a330000` | HS400-ES | HS400-ES, 1.8 V | Inherited |
| SD `2a310000` and SFC | Present | Present | Inherited |
| SDIO `2a320000` | Not enabled (the EVB's Wi-Fi is PCIe) | Wi-Fi bus | Enabled, with `mmc-pwrseq` |
| Wi-Fi | PCIe `pci14e4,449d` on `&pcie0` | SDIO AIC8800D80 | Drops the PCIe Wi-Fi child |
| IMX415 cameras (i2c) | Absent | Present, 24 references | Nothing to remove |

The reference unit's boot confirms the GMAC0 value live:
`rk_gmac-dwmac 2a220000.ethernet: TX delay(0x1b)` [measured]. `rx_delay` stays unset,
because the PHY supplies that delay through `rgmii-rxid`.

The IMX415 camera node probes and fails on missing regulators under the vendor kernel,
which confirms it as EVB carryover [measured]. Mainline `rk3576-evb1-v10.dts` carries no
camera nodes at all, so a tree derived from it has none to remove.

One EVB pin is repurposed on this board and must not be inherited. `gpio0 RK_PD3` is the
EVB's `simple-audio-card,hp-det-gpios`, and on this board it is the `gpio-keys` power input
(`KEY_POWER`, active-low, `wakeup-source`). No physical button is fitted to it.

## The gpio2 pin map

Every gpio2 (phandle `0xe1`) reference in the factory tree names one of four pins, and PB0
and PB1 serve unrelated functions [DTB]:

| Pin | Offset | Node | Property | Polarity | Function |
|---|---|---|---|---|---|
| PB0 | `0x08` | `hdmi@27da0000` | `enable-gpios` | Active-high | HDMI data-line bias |
| PB1 | `0x09` | `es8388-sound` | `spk-con-gpio` | Active-high | Speaker-amplifier enable |
| PB3 | `0x0b` | `ethernet@2a220000` | `snps,reset-gpio` | Active-low | Ethernet PHY reset |
| PB4 | `0x0c` | `pcie@2a200000` | `reset-gpios` | Active-high | PCIe reset |

Mainline `rk3576-evb1-v10.dts` agrees on both amplifier pins [DTSI]. Its `gpio2 RK_PB1`
drives `spk_power`, a `simple-audio-amplifier` speaker amplifier. Its `gpio3 RK_PD6` drives
the matching `hp_power` headphone amplifier. Those are the factory's `spk-con-gpio` and
`hp-con-gpio` pins, with the same polarity.

Both amplifier pins are named only on `es8388-sound`, the factory card that never probes,
so no captured boot shows either pin driven. A mainline tree for this board declares neither
amplifier. Driving both pins high from userspace produces no sound, alone or together with
the routing settings the jack needs [measured, 2026-07-30]. Both pins read `input`
(undriven) at boot, so nothing contends. Stream routing, not the amplifier enable, decides
whether the jack plays.

## Factory firmware

The vendor boot logs that accompany the factory tree report these firmware versions [LOG]:

- DDR `v1.03`
- SPL `v1.03`
- BL31 based on v2.3, built 2024-05, with a blank fwver
- OP-TEE `v1.01` (3.13.0)
- U-Boot dated 06/05/2024

The [as-shipped firmware revisions](hardware-reference.md#as-shipped-firmware-revisions)
compare the two units examined. The boot device is eMMC (`2a330000.mmc`). The kernel
command line also enumerates a UFS controller node, but the storage is eMMC.

## Sources

The source keys:

- `[DTB]` The factory Android device tree that cmuki posted to the Armbian RK3576
  enablement thread (`h96-m9_original.dtb`), decompiled to 12,605 lines of source. It
  self-identifies as "Rockchip RK3576 EVB1 V10 Board".
- `[LOG]` The vendor 6.1 boot logs posted to the same thread (`boot1.txt` to `boot3.txt`
  and `dmesg-19.0.txt`): the ATF DDR init and the kernel probe.
- `[DTSI]` Mainline `rk3576-evb1-v10.dts` and `rk3576.dtsi` from the 7.1.3 tree, the base
  of every comparison in this document.
- `[HDG]` Rockchip's *RK3576 Hardware Design Guide* V1.1, sections 2.3.4.1 (the USB lane
  mux) and 2.3.8.1 (the HDMI sideband mux options).

A `[measured]` claim rests on the reference unit. The evidence is its 1.5 Mbaud UART
captures of the factory Android boot, six of them, and its own factory tree.

The same thread carries a.ni's community mainline H96 device tree,
`rk3576-evb1-v10-main-h96-v5.dts` (edge 6.19, from an AIC8800 unit). Its Wi-Fi and
Bluetooth section is
specific to the AIC8800, so it applies to AIC8800 units.

A `[source]` claim is read from driver or binding source: the vendor 6.1
`dw_hdmi-rockchip.c`, mainline `dw_hdmi_qp-rockchip.c` and
`rockchip,rk3588-dw-hdmi-qp.yaml`.
