# H96 MAX M9 hardware reference

This board-level reference covers the H96 MAX M9, an Android TV box built on the Rockchip
RK3576. It states the board's wiring, addresses, parts, limits and factory firmware. It is
compiled from
the board's factory device tree, its factory boot firmware and inspection of the reference
unit. It is not a manufacturer publication.

Register addresses, GPIO assignments, silicon IDs, eMMC geometry and board wiring come from
the SoC and the PCB, and they hold across firmware builds. The behavior of the USB download
path (maskrom, loader mode, the recovery key) depends on the firmware build and the loader
blob in use. This document flags that behavior where it appears. The reference unit runs a
comparatively recent factory build, listed under
[as-shipped firmware revisions](#as-shipped-firmware-revisions).

Four companion documents cover what a board reference does not:

- The [corrections and gotchas](corrections-and-gotchas.md) name the board facts that
  community sources misreport, and the firmware-, loader- and toolchain-dependent traps.
- The [mainline enablement status](mainline-enablement-status.md) records what mainline
  Linux drives on this board, and the community consensus for each subsystem.
- The [factory device tree](factory-device-tree.md) describes the vendor's device tree
  against mainline `rk3576-evb1-v10.dts`.
- [Wi-Fi and Bluetooth](wifi-bluetooth.md) covers the AIC8800D80 radio under mainline.

## Overview

The H96 MAX M9 is an Android TV box. Its Rockchip RK3576 is a 64-bit octa-core
application processor with an Arm Mali GPU and hardware video decode. The box offers HDMI
output up to 4K, Gigabit Ethernet and dual-band Wi-Fi 6.

| Property | Value |
|---|---|
| Product | H96 MAX M9 |
| Class | Android TV box, single-board media device |
| SoC | Rockchip RK3576 |
| CPU | Octa-core: 4x Arm Cortex-A72 + 4x Arm Cortex-A53 |
| GPU | Arm Mali-G52 MC3 |
| System memory | LPDDR4X, up to 8 GB |
| Internal storage | eMMC 5.1, up to 128 GB |
| Factory OS | Android 14 |
| Reference unit | 8 GB RAM, 128 GB eMMC |

The platform ships in several memory and storage configurations: 2, 4 or 8 GB of RAM, and
16, 32, 64 or 128 GB of eMMC. Values that vary by SKU are noted as such. Register
addresses, GPIO assignments and interface descriptions are common to the platform.

## Processor and graphics

### CPU

The RK3576 has eight 64-bit Armv8-A cores in two clusters:

| Cluster | Cores | Part ID |
|---|---|---|
| Performance | 4x Cortex-A72 | `0x411fd080` |
| Efficiency | 4x Cortex-A53 | `0x410fd034` |

The clusters scale independently under dynamic voltage and frequency scaling, supplied by
the on-board RK806 PMIC. The operating points run from 408 MHz to 2016 MHz on the A53
cluster and to 2208 MHz on the A72 cluster [DTB]. Mainline offers the same two lists
through cpufreq. Under a 40 s all-core encode, both clusters held their top point with no
cooling state engaged [measured, 2026-10-02, 7.2.8].

### GPU

The GPU is an Arm Mali-G52 MC3 (Bifrost architecture) at register base `0x27800000`. It
provides 3D acceleration and, together with the display controller, on-screen composition.
The [RK3576 video reference](../video-reference.md#gpu) describes the GPU under mainline.

### Media engines

The RK3576 integrates these media engines:

- A hardware video decoder (H.264 and H.265 up to 4K)
- Two video encoder cores
- A JPEG encoder and decoder pair
- A two-core 2D raster graphics accelerator (RGA)
- A neural processing unit

The [device-tree register map](#device-tree-register-map) lists their register bases. The [RK3576 video reference](../video-reference.md) and the
[RK3576 NPU reference](../npu-reference.md) describe their mainline drivers.

## Memory and storage

| Function | Type | Detail | Register base |
|---|---|---|---|
| System RAM | LPDDR4X | 2112 MHz, 8 GB on the reference unit | n/a |
| Internal flash | eMMC 5.1 | HS400 Enhanced Strobe, 8-bit, 1.8 V, 128 GB on the reference unit | `0x2a330000` |
| Serial flash | None | None described. Both SFC controllers are `disabled` in the factory tree with no flash child, and no boot log names a flash part | `0x2a300000`, `0x2a340000` (controllers) |
| SD card | None | Not fitted. The PCB carries an unpopulated footprint, and the host controller and its pin-mux are wired (see [Boot](#boot)) | `0x2a310000` |

The DRAM is LPDDR4X clocked at 2112 MHz, organized as two channels of 4096 MB (16-bit bus,
8 banks, 2 chip-selects per channel), manufactured by SK Hynix. The boot loader initializes
it with the `rk3576_ddr_lp4_2112MHz` DDR training image, described with the rest of
[the boot chain](../boot-firmware.md#the-boot-chain).

Internal storage is eMMC in HS400 Enhanced Strobe mode at 200 MHz, with the command queue
engine enabled. Mainline reaches the same mode (8-bit, 1.8 V), where `mmc0` reads 198 MHz
actual. It moves 308 MB/s reading the raw device and 166 MB/s writing through ext4, both `O_DIRECT`
at 4 MiB blocks [measured, 2026-10-02, 7.2.8]. The reference unit presents 116 GiB usable
(device name `hDEaP3`) across 16 partitions, alongside 4 MiB hardware boot partitions.
Mainline Linux enumerates it and reads the factory partition table without additional
configuration.

The `mmcblkN` index depends on probe order and is not a stable identifier. The same eMMC
appears as `mmcblk2` under the vendor kernel and as `mmcblk0` under mainline. Address it by
its 116 GiB size and `hDEaP3` name, by `/dev/disk/by-id`, or by `PARTUUID`.

Under mainline the three MMC hosts map as follows [measured, 2026-08-31, 7.2.0], and the
numbering invites the wrong guess:

| Host | Controller | Driver | Function |
|---|---|---|---|
| `mmc0` | `0x2a330000` | `sdhci-dwcmshc` | The eMMC: `mmcblk0`, 116.5 GB, `new HS400 Enhanced strobe MMC card`, with `mmcblk0boot0`, `mmcblk0boot1` and `mmcblk0rpmb` beside it |
| `mmc1` | `0x2a310000` | `dwmmc_rockchip` | The SD host, for a slot this board does not have. It never sees a card |
| `mmc2` | `0x2a320000` | `dwmmc_rockchip` | `type=SDIO`, the AIC8800 Wi-Fi |

HS400ES, boot partitions and RPMB are eMMC-only, which settles the identity of `mmc0`.

Over rockusb, the loader's flash-info report (`K_FW_READ_FLASH_INFO`, rkdeveloptool `rfi`)
gives 244285440 sectors of 512 bytes (116.48 GiB, rendered "119280 MB"). It also reports
block size 512 KB, page size 2 KB, ECC bits 0 and access time 40. The flash-ID query
(`K_FW_READ_FLASH_ID`) answers the ASCII string `"EMMC "`. The report's manufacturer byte
is `0x00`, which rkdeveloptool renders as "SAMSUNG", while the kernel-reported part name
`hDEaP3` follows SK hynix's naming pattern. Identify the eMMC by its kernel-reported part
name, as [flash information](../rockusb-reference.md#flash-information) explains.

### As-shipped eMMC partition map

The map below is read from the reference unit's primary GPT header (128 GB SKU, where
`userdata` absorbs the SKU size difference). It is a standard Android and Rockchip
16-partition layout. The first usable LBA is 34, and the BootROM-readable ID block
(idbloader) sits at sector 64, in the gap before the first partition.

| Name          | First LBA | Sectors   | Size       |
|---------------|-----------|-----------|------------|
| security      | 8192      | 8192      | 4.00 MiB   |
| uboot         | 16384     | 8192      | 4.00 MiB   |
| trust         | 24576     | 8192      | 4.00 MiB   |
| misc          | 32768     | 8192      | 4.00 MiB   |
| dtbo          | 40960     | 8192      | 4.00 MiB   |
| vbmeta        | 49152     | 2048      | 1.00 MiB   |
| boot          | 51200     | 131072    | 64.00 MiB  |
| recovery      | 182272    | 196608    | 96.00 MiB  |
| backup        | 378880    | 786432    | 384.00 MiB |
| cache         | 1165312   | 786432    | 384.00 MiB |
| metadata      | 1951744   | 131072    | 64.00 MiB  |
| frp           | 2082816   | 1024      | 512.00 KiB |
| baseparameter | 2083840   | 2048      | 1.00 MiB   |
| rkalgo        | 2085888   | 2048      | 1.00 MiB   |
| super         | 2087936   | 8388608   | 4.00 GiB   |
| userdata      | 10476544  | 233808832 | 111.49 GiB |

`userdata` ends at sector 244285376, leaving 64 sectors of tail for the backup GPT. The
entire bootloader chain and the partition table sit inside the first 65536 sectors
(32 MiB). One flat dump of that window captures everything needed to repair a damaged
bootloader region.

The whole 116.48 GiB device reads back correctly through the block layer. A full factory
image captured through U-Boot `ums` and a host `dd` verifies, with a valid GPT and all 16
partitions [measured]. A full read over rockusb returns constant `0xCC` fill above the same
32 MiB boundary while reporting success, so a rockusb "full dump" is silently truncated.
That is a loader-software defect rather than an eMMC limit, described under
[the 32 MiB read wall](../rockusb-reference.md#the-32-mib-read-wall).

The partition count varies by factory firmware batch: a community unit examined in
September 2024 carries 15 partitions where the reference unit carries 16 [reported]. Read
the GPT off the unit at hand rather than assuming this table.

## Networking

### Ethernet

One Gigabit Ethernet port (RJ-45) is driven by the RK3576 GMAC0 controller.

| Property | Value |
|---|---|
| Controller | GMAC0 (`rk_gmac-dwmac`) at `0x2a220000` |
| Speeds | 10 / 100 / 1000 Mbit/s |
| Interface | RGMII, `rgmii-rxid` |
| TX delay | `0x1b` |
| PHY reset | gpio2 PB3, active-low |

### Wi-Fi and Bluetooth

The units examined carry an AICSemi AIC8800D80 combo module. The wireless part varies by
production batch: earlier units carry a Broadcom AP6275P instead [consensus]. Treat
wireless as a variant axis, and identify the part by its SDIO vendor ID, never by the
device-tree label described under
[the wlan-platdata label](factory-device-tree.md#the-wlan-platdata-label). The values
below describe the AIC8800D80 units.

| Property | Value |
|---|---|
| Module | AIC8800D80 |
| Wi-Fi | 802.11a/b/g/n/ac/ax (Wi-Fi 6), 2.4 GHz + 5 GHz |
| Bluetooth | Bluetooth 5.4 (LMP 5.4, reported by the controller) |
| Wi-Fi host interface | SDIO (controller at `0x2a320000`), SDIO vendor ID `0xC8A1` |
| Bluetooth host interface | UART: uart4 (`/dev/ttyS4`), 1,500,000 baud, hardware flow control |
| Bluetooth UART pins | uart4 m1 group: gpio1 PC4 TX, PC5 RX, PC3 CTS, PC2 RTS |
| Bluetooth control lines | BT_REG_ON (module `PWR_BT`) gpio1 PC7 and HST_WAK_BT gpio1 PD4, both active-high |

The module identifies itself on the SDIO bus by AICSemi vendor ID `0xC8A1`. The Wi-Fi side
attaches over SDIO and the Bluetooth side over UART. The Bluetooth controller shares one
combo identity with the Wi-Fi side, and its address is the Wi-Fi MAC plus one. The
controller boots into low-power sleep and answers `HCI_Reset` only after BT_REG_ON and
HST_WAK_BT are driven high following the SDIO firmware load [measured]. A static
device-tree assertion applies at GPIO probe, before the firmware, and is too early.
[Bluetooth](wifi-bluetooth.md#bluetooth) describes the attach sequence.

## USB

Two external USB ports, one USB 3.0 and one USB 2.0, are served by the RK3576's two DWC3
dual-role controllers and their PHYs. Two is the SoC's entire USB complement [HDG]. Each
DWC3 embeds one USB 2.0 OTG, and the SoC has no host-only USB 2.0 controllers besides
them ([USB controllers and combo PHY 1](../hardware-design-guide.md#usb-controllers-and-combo-phy-1)).

The board therefore has an internal hub [measured]. A `1a86:8091` 4-port USB 2.0 hub hangs
off `usb_drd1`. It serves the external USB 2.0 receptacle and the two soldered RF receivers
listed under [peripherals and controls](#peripherals-and-controls). Count ports from the
hub, not from the rear panel.

`usb_drd0` is the only port that delivers SuperSpeed [measured, 2026-08-08]. It runs at
5 Gbps and ~154 MB/s over 92 GB of continuous reads, on the usbdp PHY with nothing beyond
the SoC dtsi. A drive that comes up at 480 Mb/s there is a seating fault rather than a
configuration fault, as the [corrections and gotchas](corrections-and-gotchas.md) record.

The SuperSpeed root hub of `usb_drd1` has a single port whose lane reaches no connector,
so its SuperSpeed bus stays permanently empty. Its SuperSpeed PHY must still remain on the
controller, because dropping it makes the board unbootable. The mechanism is described
under [capping a dwc3 port to High Speed](../corrections-and-gotchas.md#capping-a-dwc3-port-to-high-speed).
The boot-time `wait phy status ready timeout` from `combphy1_psu` is that unrouted lane,
and it is cosmetic.

| Function | Detail | Register base |
|---|---|---|
| USB controller 0 (`usb_drd0`) | DWC3, the OTG and device-capable controller, behind the USB 3.0 Type-A port | `0x23000000` |
| USB controller 1 (`usb_drd1`) | DWC3, serving the USB 2.0 Type-A port | `0x23400000` |
| USB 2.0 PHY | Innosilicon USB2 PHY | n/a |
| USB3 / DisplayPort combo PHY | usbdp PHY | `0x2b010000` |
| SerDes combo PHY | naneng combphy | `0x2b050000`, `0x2b060000` |

**The USB 3.0 port is the device port**, and it is the one to use for maskrom, loader mode
and rockusb. Only `usb@23000000` carries an `extcon` property, which points at USB2 PHY 0
(`usb2-phy@0`) [DTB]. Both USB2 PHYs declare the `otg-bvalid`, `otg-id` and `linestate`
interrupts on their `otg-port`, so those do not single out a controller. The tree's
`usb-otg0-pwren` pinctrl entry is unreferenced, and its pin is gpio0 PD1, the status LED.
Which connector a controller reaches is a PCB fact and does not change with firmware.

The u-boot rockusb gadget started as `rockusb 0 mmc 0` takes `0` as the UDC index, which
selects drd0. U-boot confirms the choice by printing *"waiting for usb3 connection"*.

Two behaviors are easy to misread as faults:

- The gadget enumerates at high speed (USB 2.0 signaling) even on the SuperSpeed
  connector. A full 116 GiB eMMC dump therefore takes the better part of an hour. This is
  expected and is not a symptom of the wrong port.
- The factory Linux device tree sets `dr_mode = "host"` on both controllers. Android
  therefore cannot present a USB device on either port, and adb over USB never enumerates.
  That is a
  property of the factory OS configuration, not of the hardware. The BootROM and the u-boot
  rockusb gadget drive drd0 as a device regardless.

Both external ports are Type-A receptacles, so connecting the box to a PC needs a passive
A-to-A cable. An active or bridged A-to-A cable does not work.

In maskrom and in loader mode the USB 3.0 port presents the RK3576 download gadget.
[The download gadget](../rockusb-reference.md#the-download-gadget) lists its descriptors,
and [mode detection](../rockusb-reference.md#mode-detection) explains how to tell the two
modes apart.

## Display and audio

### Display

| Property | Value |
|---|---|
| Output | HDMI, up to 4K UHD |
| HDMI transmitter | `0x27da0000` |
| HDMI PHY | `0x2b000000` |
| Display controller | VOP2 at `0x27d00000` |
| HDMI enable | gpio2 PB0, active-high |
| CEC | Supported by the DW-HDMI-QP controller's CEC block (shares `0x27da0000` with video) |

The HDMI output works end to end at 1080p60 (TMDS, RGB 8bpc) from both mainline u-boot and
mainline Linux [measured]. Under mainline u-boot the logo and vidconsole render on the
panel. Mainline Linux reaches a console and login prompt on the same panel, with EDID read
and the connector's mode list populated. The [RK3576 display reference](../display-reference.md)
carries the register-level notes for the display path, including
[the TMDS rate ceiling](../display-reference.md#the-tmds-rate-ceiling) under mainline.

A bootloader that drives the display must quiesce the VOP on the way out. One that lights
the panel and boots an OS without tearing the VOP down leaves the kernel unable to drive the
display at all.
[Handing the display over to an operating system](../display-reference.md#handing-the-display-over-to-an-operating-system)
describes the teardown.

The HDMI enable pin above is the board's physical wiring. Mainline describes it as
`frl-enable-gpios`, active-low, so the sense inverts against the vendor's usage. Read
[the HDMI data-line bias pin](factory-device-tree.md#the-hdmi-data-line-bias-pin) and the
[corrections and gotchas](corrections-and-gotchas.md) before writing it into a device tree.

### Audio

The device provides three audio output paths:

- HDMI audio, carried on the HDMI output
- S/PDIF digital audio output
- Analog stereo line-out, through an HT5010 amplifier on SAI1 (`sai@2a610000`)

SAI1 drives the HT5010 over I2S on the `sdo2` data pin, and feeds it MCLK
(`CLK_SAI1_MCLKOUT_TO_IO`, 12.288 MHz) on the `sai1m0_mclk` pin. The HT5010 has no control
interface, so a device tree models it as a dummy codec rather than an I2C one. The factory
device tree also declares an `everest,es8388` at i2c3 (`0x2ac60000`) address `0x10`. That
node is EVB carryover and does not bind.

All three paths work under mainline [measured, 2026-07-30]. HDMI audio and S/PDIF need only
their nodes. The analog jack also needs the board device tree to route a stream onto
`sdo2`, and a kernel change to keep that routing written (see below).

The ES8388 node is a dead end on both stacks. Under mainline the codec never acknowledges
on i2c3, every register access ends `-ETIMEDOUT`, and its `asoc-simple-card` fails to probe
with `-110`. Under the vendor OS the node also fails to bind: six vendor boot captures
register three cards, and none of them is the ES8388's
([the three audio cards](factory-device-tree.md#the-three-audio-cards)). Nothing on either
side talks to i2c3, so the silence there is a property of the node rather than a mainline
integration gap.

One measurement characterizes i2c3 itself: it is electrically wedged rather than empty
[measured]. An `i2cdetect` sweep times out on all 117 addresses at ~1 s each, where a
healthy bus NAKs an absent address in microseconds. Both i2c1 and i2c2 sweep in ~60 ms.
Universal timeouts of that kind indicate a slave holding SCL rather than a missing device.
That is consistent with an unclocked ES8388, since MCLK reaches the pin only after
something explicitly enables `CLK_SAI1_MCLKOUT_TO_IO`.

The pin-mux is not the cause. i2c3 uses `i2c3m0_xfer` in both the factory device tree and
mainline, with identical pin definitions (`<4 RK_PB5 11>`). `pinmux-pins` shows
`2ac60000.i2c` holding pins 140 and 141 with no competing claim.

The amplifier enables are not on the analog path either. The factory tree names
`spk-con-gpio` at gpio2 PB1 and `hp-con-gpio` at gpio3 PD6. Mainline's EVB device tree
drives the same pins as `spk_power` and `hp_power` `simple-audio-amplifier` nodes. The
factory tree names them only on the ES8388 card that never probes, and driving both high
from userspace produces no sound [measured, 2026-07-30]. A mainline device tree for this
board declares neither amplifier.

The SoC's internal DAC, `acdcdig-dsm@2a6d0000`, does not feed the 3.5 mm jack. It is
`disabled` in the factory tree [DTB], and it drives differential speaker pins.

The analog jack needs three settings beyond a registered card:

- `rockchip,sai-tx-route = <0 1 0>` in the board device tree, so that `SAI_PATH_SEL`
  carries field 2 = 0 (`0xe4c4`) and data reaches `sdo2`. Field 2 is reachable only by
  writing fields 0 to 2, hence three entries.
- A `clocks` reference on the `simple-audio-card,cpu` sub-node, which enables
  `CLK_SAI1_MCLKOUT_TO_IO`. Without it, MCLK is generated and never leaves the SoC. Both
  gates flip during playback and release at idle [measured].
- A kernel change that re-applies `SAI_PATH_SEL` from `rockchip_sai_runtime_resume()`
  after the cache sync. The upstream driver programs the register once at probe, and the
  value does not survive to the running device.

[SAI transmit routing](../corrections-and-gotchas.md#sai-transmit-routing) describes the
register semantics, which invert the binding's prose, and the driver behavior behind the
last two items.

With all three in place the jack plays [measured, 2026-07-30]. `hw_ptr` advances 48496
frames/s at a 48 kHz rate, `mclk_sai1_8ch` runs at 12.288 MHz, and pinctrl confirms
`sai1m0-mclk`, `-sclk`, `-lrck` and `-sdo2` applied.

## Peripherals and controls

| Peripheral | Detail |
|---|---|
| Status LED | White, gpio0 PD1, active-low |
| Power-key input | gpio0 PD3 (`gpio-keys`, `KEY_POWER`, `wakeup-source`), active-low. No physical button is fitted. The bundled remote's POWER key reaches it, see below |
| Recovery button | Behind the 3.5 mm AV jack. An SARADC channel 1 key, not a GPIO |
| Infrared | IR receiver on gpio0 PC4 (the SoC PWM0 IR pin, in the PMU always-on domain), present in the vendor DT. Reception works [measured, 2026-07-30], with NEC decoded through to input events. The bundled remote is RF, not IR, so it cannot drive this path. Reception was validated with an H96 Max H313 handset |
| Remote control | Bundled "XFY MIC RC" handset: 2.4 GHz RF plus voice, factory-paired to an internal USB receiver (`0a45:0008`) behind an internal hub (`1a86:8091`). It presents as a USB HID keyboard and mouse, and works with no driver, keymap or pairing. It is neither Bluetooth nor IR |
| Debug UART | `0x2ad40000` (uart0), see [serial debug console](#serial-debug-console) |
| Real-time clock | Not fitted, see below |
| Thermal sensors | TSADC `0x2ae70000`, six zones (package, big core, little core, GPU, NPU, DDR) |

The board has no battery-backed RTC. The factory device tree declares a Haoyu HYM8563 at
i2c2 (`0x2ac50000`) address `0x51`, inherited from the Rockchip EVB design, but no such
device is present. Mainline probes it and fails with
`rtc-hym8563 2-0051: could not init device, -6`. An `i2cdetect` sweep of i2c2 completes in
~60 ms with every address empty [measured]. A healthy bus that NAKs every address that
quickly is positive evidence of absence rather than a bus fault, so i2c2 is electrically
fine. The board keeps no wall-clock time across power cycles and depends on NTP, and
`/dev/rtc` does not exist.

No other RTC exists to fall back to, so the absence is final rather than a driver gap to
close. The RK3576 has no on-chip RTC, and the TRM v1.2 describes no such peripheral block
[source]. The TRM's only "RTC" hits are the GMAC Receive Threshold Control register field
and the `cm0_rtc` clock names. The RK806 PMIC offers none under mainline either.
`rtc-rk808` covers RK805, RK808, RK809, RK817 and RK818 only, and the RK806 mfd cell list
instantiates no RTC cell [source].

A mainline device tree for this board therefore leaves i2c2 disabled and declares no RTC.
Declaring the EVB's chip only produces a probe failure on every boot.

The TSADC reports valid temperatures across all six zones, about 39-42 °C at idle in open
air [measured]. Its probe emits a run of `failed reading trim of sensor 0: -EPROBE_DEFER`
messages during boot. These are ordinary deferred-probe retries that resolve, and no
success message follows them. Judge the sensor by whether
`/sys/class/thermal/thermal_zone*/temp` reads, not by the absence of those log lines.

gpio0 PD3 is a power-key input with no button on it. The bundled remote's POWER key wakes the
board from s2idle through this input [measured], although the remote's USB receiver cannot
signal a wakeup over USB. The receiver therefore drives gpio0 PD3 directly [inferred]. See
[suspend configuration](corrections-and-gotchas.md#suspend-configuration).

The recovery button is the board's only button, recessed behind the AV jack so that a
toothpick reaches it without opening the case. It is read as an analog key rather than a
GPIO. The boot firmware's `adc-keys` node places it on SARADC channel 1 (`adc@2ae00000`)
with `press-threshold-microvolt = <1750>` against a 1.8 V reference. Held at power-on under
the factory firmware, it boots Android recovery.

The SARADC is 12-bit. A press measures raw ~41 (about 18 mV) against ~4087 idle [measured].
A press is a short to ground with a small residual, not raw 0. That residual defeats a
press window sized for 10-bit parts, as
[the SARADC download key](../boot-firmware.md#the-saradc-download-key) describes.

The status LED doubles as a rough boot-state indicator [measured]. It glows dim in maskrom
mode, consistent with the pin sitting undriven while the BootROM runs. It glows at full
brightness once firmware is running (Android recovery, normal Android boot). A dim LED after
a power-up with the eMMC pads bridged confirms that the BootROM fell through to USB. The
LED state under the rkbin loader and under u-boot is not recorded.

## Power and mechanical

| Property | Value |
|---|---|
| Power input | 5 V / 2 A DC barrel jack |
| PMIC | Rockchip RK806, I2C address `0x23` (bus 1) |
| Dimensions | 96 × 22.7 × 138 mm (W × H × L) |
| Unit weight | 177.9 g |

## Serial debug console

| Property | Value |
|---|---|
| Controller | uart0 at `0x2ad40000` |
| Baud rate | 1,500,000 (1.5 Mbaud) |
| Framing | 8N1, no flow control |
| Logic level | 3.3 V |
| Early console | `earlycon=uart8250,mmio32,0x2ad40000` |
| Test points | Three bare vias on the board edge next to the 3.5 mm AV jack, ordered TX, GND, RX |

The console pads are unpopulated vias at a pitch finer than 2.54 mm, and they take a
soldered fine-gauge wire.

## Boot

| Property | Value |
|---|---|
| Boot device | eMMC (`0x2a330000`), block device `mmc0` |
| Boot order (SPL) | SD host (`0x2a310000`, `mmc1`), then SPI-NAND, then SPI-NOR, then eMMC |
| Boot order (hardware strap) | `eMMC -> USB`, from SARADC channel 0 reading 4088 [measured, 2026-07-30] |
| Boot chain | BootROM -> DDR init (TPL) -> SPL -> ATF (BL31) -> OP-TEE (BL32) -> U-Boot -> kernel |
| DDR training image | `rk3576_ddr_lp4_2112MHz_lp5_2736MHz` |
| Recovery entry | Maskrom mode over USB, by shorting the two eMMC test pads at power-on |
| USB device port | The USB 3.0 Type-A on the rear panel (controller `usb_drd0`), see [USB](#usb) |

The boot chain above is the factory one. [The boot chain](../boot-firmware.md#the-boot-chain)
describes the SoC's boot stages, and [OP-TEE and BL32](../boot-firmware.md#op-tee-and-bl32)
describes a mainline chain without BL32.

The SPL's `u-boot,spl-boot-order` places the SD host ahead of the eMMC, so a populated card
slot takes boot priority. No slot is fitted, but the controller and its pin-mux are wired.
Bus, clock and command sit on gpio2 PA0-PA5, card-detect on gpio0 PA7, and power-enable on
gpio0 PB6.

That order governs where the boot chain itself loads from, and the BootROM and SPL never
load from USB. A running u-boot can still distro-boot an operating system from a USB stick
(`usb start`, then `run bootcmd_usb0`) on the factory firmware [consensus]. That route
trials an OS and leaves the eMMC untouched.

The BootROM's own order is set in hardware by a resistor divider on `SARADC_IN0` and read
at reset [HDG]. On this board channel 0 reads 4088, which matches the `Rup=10k / Rdown=NC`
row (4095, eMMC then USB) of the
[boot-strap table](../hardware-design-guide.md#saradc-boot-strap-and-recovery-key)
[measured, 2026-07-30]. The next row down is 3680, eMMC then SDMMC0 then USB. The strapped
order has no SD and no FSPI, so the BootROM reaches USB and maskrom only after the eMMC
fails. The recovery key on channel 1 is a fixed SoC function that cannot be remapped. That
is why the recovery input is an SARADC key rather than a GPIO.

Maskrom entry is by shorting the eMMC clock and ground test points. They are a pair of small
pads on the solder side, next to the `EMMC` silkscreen. They sit just outside the white
silkscreen border that carries a row of gold pads. Bridging them at power-on holds the eMMC
clock at ground. The BootROM then cannot clock data out of the flash, and it falls through
to USB device mode. No button enters maskrom, and the pads are the only entry that does not depend on a
working bootloader.

A dim status LED after power-up confirms the fall-through. A BootROM whose download was
interrupted accepts no fresh upload. Power-cycle with the pads bridged to strap it again.

The factory firmware maps `reboot bootloader` to the fastboot magic. "Reboot to bootloader"
from Android recovery therefore lands in fastboot rather than in a flashable USB mode.
[Reboot modes](../boot-firmware.md#reboot-modes) describes the reboot-mode register, the
Rockchip magics, and the single request (maskrom) that rkbin BL31 v1.24 forwards to the
BootROM.

## As-shipped firmware revisions

Two production units have been examined. Both carry the same hardware and differ only in
factory firmware build:

| Component | Unit A | Unit B (the reference unit) |
|---|---|---|
| DDR init | `81dd75088a`, 2024-04-16, fwver `v1.03` | `da1087e33f`, 2024-06-17, fwver `v1.05` |
| U-Boot SPL | `2017.09-ge53b21c53e6-240430` (2024-04-30), fwver `v1.03` | `2017.09-g884f049582f-240614` (2024-06-19), fwver `v1.05` |
| TF-A (BL31) | `v2.3-789-gebfc4b78c` | `v2.3-802-g286c013b2`, fwver `v1.09` (2024-06-28) |
| OP-TEE (BL32) | `3.13.0-791-g185dc3c92`, fwver `v1.01` | `3.13.0-795-gb90c1680e`, fwver `v1.02` |
| U-Boot | `2017.09-240430-dirty` (2024-06-05) | `2017.09-231221-dirty` (2025-09-24) |
| Android kernel | `6.1.57 #73` (2024-06-05) | `6.1.75 #140` (2025-09-24) |

## Device-tree register map

This map lists the blocks this board uses, plus the RK3576's media engines.

| Block | Node | Base address |
|---|---|---|
| GMAC0 Ethernet | `ethernet@2a220000` | `0x2a220000` |
| eMMC | `mmc@2a330000` | `0x2a330000` |
| SD host | `mmc@2a310000` | `0x2a310000` |
| SDIO (Wi-Fi) | `mmc@2a320000` | `0x2a320000` |
| SPI flash controllers (SFC, no flash described) | `spi@2a300000` / `spi@2a340000` | `0x2a300000` / `0x2a340000` |
| USB DWC3 #0 | `usb@23000000` | `0x23000000` |
| USB DWC3 #1 | `usb@23400000` | `0x23400000` |
| GPU (Mali-G52) | `gpu@27800000` | `0x27800000` |
| VOP2 display | `vop@27d00000` | `0x27d00000` |
| HDMI TX | `hdmi@27da0000` | `0x27da0000` |
| HDMI PHY | `hdmiphy@2b000000` | `0x2b000000` |
| USB-DP combo PHY | `phy@2b010000` | `0x2b010000` |
| SerDes combo PHY 0 / 1 | `phy@2b050000` / `phy@2b060000` | `0x2b050000` / `0x2b060000` |
| Video decoder | `rkvdec@27b00000` | `0x27b00000` |
| Video encoder core 0 / 1 | `rkvenc-core@27a00000` / `rkvenc-core@27a10000` | `0x27a00000` / `0x27a10000` |
| JPEG decoder / encoder | `jpegd@27910000` / `jpege@27910800` | `0x27910000` / `0x27910800` |
| 2D accelerator (RGA) core 0 / 1 | `rga@27920000` / `rga@27930000` | `0x27920000` / `0x27930000` |
| NPU | `npu@27700000` | `0x27700000` |
| PMIC I2C bus (i2c1) | `i2c@2ac40000` | `0x2ac40000` |
| Thermal sensor (TSADC) | `tsadc@2ae70000` | `0x2ae70000` |
| SARADC | `adc@2ae00000` | `0x2ae00000` |
| Random number generator | `rng@2a410000` | `0x2a410000` |
| Watchdog | `watchdog@2ace0000` | `0x2ace0000` |
| Clock controller (CRU) | `clock-controller@27200000` | `0x27200000` |
| Interrupt controller (GIC) | `interrupt-controller@2a701000` | `0x2a701000` |
| Debug UART | `serial@2ad40000` | `0x2ad40000` |

### GPIO banks

| Bank | Base address | Domain |
|---|---|---|
| gpio0 | `0x27320000` | PMU, always-on |
| gpio1 | `0x2ae10000` | Main |
| gpio2 | `0x2ae20000` | Main |
| gpio3 | `0x2ae30000` | Main |
| gpio4 | `0x2ae40000` | Main |

### Board GPIO assignments

| Signal | GPIO | Polarity |
|---|---|---|
| Ethernet PHY reset | gpio2 PB3 | Active-low |
| HDMI enable | gpio2 PB0 | Active-high |
| Speaker-amplifier enable | gpio2 PB1 | Active-high |
| Status LED | gpio0 PD1 | Active-low |
| Power-key input | gpio0 PD3 | Active-low |
| IR receiver | gpio0 PC4 | Active-low |
| Wi-Fi WL_REG_ON | gpio1 PC6 | Active-high |
| Bluetooth BT_REG_ON (`PWR_BT`) | gpio1 PC7 | Active-high |
| Bluetooth HST_WAK_BT | gpio1 PD4 | Active-high |

## Observed configurations

These findings hold across every unit examined, and community sources commonly misreport
them:

- The Wi-Fi silicon is AIC8800D80 on both units examined, identified by SDIO vendor ID
  `0xC8A1` (AICSemi) and by the module marking `L8800D 80 WIFI&BT`.
- The device tree's `wlan-platdata` node reports `wifi_chip_type = ap6275p` on both units,
  while both run an AIC8800D80. That string is vendor boilerplate, not a chip identifier.
  Identify the Wi-Fi part by its SDIO vendor ID, never by the device-tree label.
- The factory device tree is Rockchip's EVB1 V10 reference tree. It carries board-specific
  values (Ethernet timing, GPIO assignments) and retains EVB-only nodes such as the IMX415
  camera sensor, which probes and fails on this board.
- Both units report `Model: Rockchip RK3576 EVB1 V10 Board` from U-Boot and from the
  kernel.
- Hardware identity is identical across the units examined: RK3576, LPDDR4X at 2112 MHz,
  the RK806 PMIC and the AIC8800D80. The analog path on both runs SAI1 -> HT5010 rather
  than through the device tree's carried-over ES8388. Only the factory firmware build
  differs.

## Sources

Every value in this document traces to the board's factory firmware, the manufacturer's
published specification, or the reference unit. The source keys:

- `[LOG]` Factory boot logs posted to the Armbian forum's H96 MAX M9 thread (`boot1.txt`,
  `boot2.txt`, `boot3.txt` and `dmesg-19.0.txt`). They carry the ATF DDR-init output and
  the Linux kernel probe from the factory firmware. They record runtime-observed hardware.
- `[DTB]` Factory device tree (`h96-m9_original.dtb` from the same thread, decompiled): the
  vendor's
  own hardware description of register addresses, GPIO assignments and interface
  configuration.
- `[DTSI]` Mainline `rk3576.dtsi` and `rk3576-evb1-v10.dts` (the 7.1 tree), used only to
  resolve device-tree phandles to GPIO
  bank labels and to cross-check register bases.
- `[SPEC]` The manufacturer's product information page: published figures for
  port count, power input, dimensions, CEC and ARC, and codec support. Not independently
  measured.
- `[HDG]` Rockchip's *RK3576 Hardware Design Guide* V1.1: the SoC's own constraints on
  pin muxes, IO domains, boot straps and interface sharing. It states what the silicon
  requires of a board, and the [RK3576 hardware design guide](../hardware-design-guide.md)
  summarizes its software-relevant content.

A `[measured]` claim rests on the reference unit. The evidence is a serial-console boot
capture taken from the unit itself, recorded at 1.5 Mbaud from the board's UART pads. It also includes direct visual inspection
of board markings and LED behavior. The rockusb wire measurements were taken on 2026-07-16
with two independent host implementations (pyrographer and an rkdeveloptool build),
cross-checked against each other.

The factory device tree self-identifies as "Rockchip RK3576 EVB1 V10 Board" [LOG][DTB].
The vendor ships Rockchip's EVB reference tree, so the board-specific values below
(Ethernet timing, GPIO assignments, Wi-Fi module) are the vendor's own deltas on that base.
The [factory device tree](factory-device-tree.md) lists them.

### Processor, memory, storage

| Item | Source | Evidence |
|---|---|---|
| 4x Cortex-A72 + 4x Cortex-A53 | [LOG] | CPU part IDs `0x411fd080` (A72) and `0x410fd034` (A53) across CPU0-7 |
| Mali-G52 at `0x27800000` | [LOG][DTB] | `mali 27800000.gpu: Kernel DDK version g21p0`, DTB `gpu@27800000` |
| H.264 / H.265 decode, up to 4K | [SPEC] | Info-page video-decoder list |
| LPDDR4X at 2112 MHz | [measured][LOG] | DDR-init: `LPDDR4X, 2112MHz` |
| Dual channel, 4096 MB per channel, SK Hynix | [measured] | `channel[0]/[1] ... Size=4096MB`, `Manufacturer ID:0x6` |
| 8 GB RAM (reference unit) | [measured][LOG] | `Memory: 8078128K/8370176K available` |
| eMMC HS400-ES at `0x2a330000` | [LOG][DTB] | `MMC0: HS400 Enhanced Strobe, 200Mhz`, DTB `mmc@2a330000` + `mmc-hs400-enhanced-strobe` |
| 128 GB eMMC (reference unit) | [SPEC] | Product SKU (8 GB / 128 GB), ~116 GiB usable |
| No serial flash, SFC controllers at `0x2a300000` / `0x2a340000` | [DTB][LOG] | `spi@2a300000` and `spi@2a340000` (`rockchip,sfc`), both `status = "disabled"` with no flash child in every factory tree. No flash probe in any boot log |
| CPU OPPs 408-2016 MHz (A53) / 408-2208 MHz (A72) | [DTB][measured] | `cluster0-opp-table` / `cluster1-opp-table`, and cpufreq `scaling_available_frequencies` under mainline |

### Networking

| Item | Source | Evidence |
|---|---|---|
| GMAC0 at `0x2a220000`, `rgmii-rxid`, `tx_delay 0x1b` | [DTB][LOG] | DTB `ethernet@2a220000` `phy-mode="rgmii-rxid"` `tx_delay=<0x1b>`. Boot log `rk_gmac-dwmac 2a220000.ethernet eth0: configuring for phy/rgmii-rxid` |
| PHY reset gpio2 PB3, active-low | [DTB][DTSI] | DTB `snps,reset-gpio=<0xe1 0x0b 0x01>`. Phandle `0xe1`->`gpio@2ae20000`=gpio2 [DTSI], `0x0b`=PB3, `0x01`=active-low |
| One Gigabit RJ-45 | [SPEC] | `LAN Ethernet:1000 standard RJ-45` |
| AIC8800D80, SDIO vid `0xC8A1` | [measured][LOG] | Module silkscreen `L8800D 80 WIFI&BT`, `aicbsp_sdio_probe ... vid:0xC8A1 did:0x0082`, and the `aic8800_bsp` driver binds |
| SDIO controller at `0x2a320000` | [DTB] | `mmc@2a320000` (`no-sd`) |
| Wi-Fi 802.11ax, Bluetooth 5.x | [SPEC] | `WLAN 802.11a/b/g/n/ax`, `BT 5.4` |

### USB, display, audio

| Item | Source | Evidence |
|---|---|---|
| DWC3 controllers at `0x23000000`, `0x23400000` | [DTB] | `usb@23000000` (`usb_drd0_dwc3`), `usb@23400000` (`usb_drd1_dwc3`) |
| One USB 3.0 + one USB 2.0 | [SPEC] | `2*USB 1*USB3.0 and 1*USB2.0` |
| usbdp and combphy PHYs | [DTB] | `phy@2b010000`, `phy@2b050000`, `phy@2b060000` |
| HDMI TX at `0x27da0000` | [DTB] | `hdmi@27da0000` |
| HDMI enable gpio2 PB0, active-high | [DTB][DTSI] | `hdmi@27da0000` `enable-gpios=<0xe1 0x08 0x00>` -> gpio2 PB0 active-high |
| Speaker-amp enable gpio2 PB1, active-high | [DTB][DTSI] | `es8388-sound` `spk-con-gpio=<0xe1 0x09 0x00>`. Mainline evb1 drives `spk_power: speaker-amplifier` from `gpio2 RK_PB1`. Named only on the card that never probes, so never observed driven |
| Headphone-amp enable gpio3 PD6, active-high | [DTB][DTSI] | `es8388-sound` `hp-con-gpio=<0x107 0x1e 0x00>` -> gpio3 PD6. Mainline evb1 drives `hp_power` from the same pin. Same caveat |
| Only two USB controllers exist on the SoC | [HDG] | Section 2.3.4: two USB3 OTG controllers, each embedding one USB 2.0 OTG, and no separate USB 2.0 hosts |
| drd1's USB 2.0 depends on combo PHY 1 | [HDG] | Section 2.3.4.2: with COMBO PHY1 set to PCIe or SATA, "USB3 OTG1 function cannot be used, and USB2.0 PHY1 also cannot be used" |
| HPD and CEC each have two mux options in different IO domains | [HDG] | Section 2.3.8.1: `HDMI_TX_HPDIN` and `HDMI_TX_CEC` in VCCIO6 (M0) and PMUIO1 (M1) |
| HDMI M0 is this board's mux | [DTB] | `hdmi@27da0000` `pinctrl-0` resolves to `hdmi_txm0-pins` + `hdmi-tx-scl` + `hdmi-tx-sda`. The m1 group is unreferenced |
| VOP2 at `0x27d00000`, HDMI PHY at `0x2b000000` | [DTB] | `vop@27d00000`, `hdmiphy@2b000000` |
| HDMI up to 4K, CEC and ARC | [SPEC] | `1*HD support up to 4K@120fps`, `Support CEC ... and ARC` |
| Analog out is SAI1 -> HT5010, not the ES8388 | [measured][DTB][LOG] | The registering card is `ht5010-sound` (`simple-audio-card` over `sai@2a610000` + a `rockchip,dummy-codec`), whose `simple-audio-card,name` is the string `rockchip,es8388`. The ES8388's own card name `rockchip-es8388` appears in none of the six vendor boot captures |
| ES8388 node present but unbound | [measured][DTB] | DTB `everest,es8388` at i2c3 reg `0x10`, `status = "okay"`. No probe line in any capture, and i2c3 answers nothing under mainline |
| SAI1 uses `sdo2`, not the EVB's `sdo0` | [DTB] | `sai@2a610000` `pinctrl-0` -> `sai1m0-lrck`, `-sclk`, `-sdi0`, `-sdo2`. MCLK muxed on the codec node as `sai1m0-mclk` |
| S/PDIF out is `spdif_tx1` on the m1 pin group | [DTB][HDG] | `spdif-tx@2a680000` `pinctrl-0` -> `spdifm1-tx1`. HDG section 2.3.9.5.2 puts `SPDIF_TX1_M1` at `GPIO3_A3` in VCCIO4, matching `<3 RK_PA3 5>` |
| S/PDIF, analog stereo out | [SPEC] | `SPDIF support`, `1*audio out` |

### Peripherals, power, console, boot

| Item | Source | Evidence |
|---|---|---|
| Status LED gpio0 PD1, active-low | [DTB][DTSI] | `gpio-leds` `gpios=<0x6e 0x19 0x01>`. Phandle `0x6e`->`gpio@27320000`=gpio0 [DTSI], `0x19`=PD1 |
| Power-key input gpio0 PD3 (`KEY_POWER`) | [DTB][DTSI] | `gpio-keys` `gpios=<0x6e 0x1b 0x01>` `linux,code=<0x74>`, gpio0 PD3 active-low |
| IR remote receiver | [SPEC] | Ships `Remote *1` |
| RK806 PMIC at I2C `1-0023` | [measured][DTB] | `rk806 1-0023: chip id: RK806,ver:0xa`, DTB `pmic@23` |
| 5 V / 2 A DC | [SPEC] | `1*DC 5V/2A` |
| 96 × 22.7 × 138 mm, 177.9 g | [SPEC] | Product dimensions |
| UART0 at `0x2ad40000` | [LOG][DTB] | Command line `earlycon=uart8250,mmio32,0x2ad40000`, DTB `serial@2ad40000` (`uart0`) |
| 1.5 Mbaud, 8N1 | [measured] | A clean 4,795-line boot captured at 1500000 8N1 directly from the board's UART vias |
| UART pads: three vias by the AV jack | [measured] | Direct board inspection |
| eMMC boot device | [LOG] | Command line `storagemedia=emmc`, `boot_devices=...,2a330000.mmc`, root on `mmcblk2` |
| Boot chain SPL->BL31->BL32->U-Boot->kernel | [LOG] | `BL31: v2.3`, `BL31: Initializing BL32`, `OP-TEE version: 3.13.0`, `Starting kernel` |
| GPIO banks gpio0 to gpio4 | [DTSI][DTB] | DTSI `gpio0:gpio@27320000` ... `gpio4:gpio@2ae40000`. DTB phandles gpio0=`0x6e`, gpio1=`0x15e`, gpio2=`0xe1`, gpio3=`0x107`, gpio4=`0x110` |
| eMMC 116 GiB usable, device `hDEaP3` | [measured] | `mmcblk2: mmc2:0001 hDEaP3 116 GiB`, 16 partitions + 4 MiB boot partitions |
| DRAM reported by U-Boot | [measured] | `DRAM:  8 GiB` |
| As-shipped firmware revisions | [measured][LOG] | DDR, SPL, BL31, OP-TEE and U-Boot banners from both units' boot logs |
| Media engine bases | [DTB] | `rkvdec@27b00000`, `rkvenc-core@27a00000`, `@27a10000`, `rga@27920000`, `@27930000`, `npu@27700000`, `jpegd@27910000`, `jpege@27910800` |
| System block bases | [DTB] | `clock-controller@27200000`, `interrupt-controller@2a701000`, `tsadc@2ae70000`, `adc@2ae00000`, `rng@2a410000`, `watchdog@2ace0000`, `i2c@2ac40000` |

### Rockusb and eMMC wire measurements

These measurements were taken on 2026-07-16.

| Item | Source | Evidence |
|---|---|---|
| 244285440 sectors of 512 B, and the flash geometry | [measured] | `K_FW_READ_FLASH_INFO` via rkdeveloptool `rfi`. Pyrographer's parse agrees field for field, and the sector count is byte-identical |
| Flash ID `"EMMC "` | [measured] | `K_FW_READ_FLASH_ID` reply `45 4d 4d 43 20` |
| Partition map | [measured] | Primary GPT header read off the board over rockusb |
| Status LED dim in maskrom, bright under firmware | [measured] | Direct observation across mode entries |
