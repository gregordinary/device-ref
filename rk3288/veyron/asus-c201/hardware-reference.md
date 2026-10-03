# ASUS Chromebook C201 hardware reference

This document describes the ASUS Chromebook C201 (C201PA), the `google,veyron-speedy` board,
as mainline Linux sees it. It states the board's own values: its parts, buses, device-tree
nodes and limits. What the C201 shares with the other Veyron boards is in
[the Veyron family boards](../family-boards.md), and how it boots is in
[the Veyron boot chain](../boot-chain.md). Misreported claims and integration traps are in
[the Veyron corrections and gotchas](../corrections-and-gotchas.md).

The primary source is the mainline device tree, because the whole board is upstream.
Measurements come from the reference unit, a libreboot-flashed C201.

## Overview

The ASUS Chromebook C201 (C201PA) is the `google,veyron-speedy` board, an 11.6-inch RK3288
Chromebook from 2015 [source]. It is one of the ten Veyron boards in mainline.
`depthcharge-tools` identifies it as "ASUS Chromebook C201PA", with the HWID match
`^SPEEDY .*` [source][DCT].

| Item | Value |
|---|---|
| SoC | Rockchip RK3288-C, four Cortex-A17 cores at up to 1.8 GHz |
| ISA | ARMv7-A, 32-bit, so Debian `armhf` |
| GPU | Mali-T764 (Midgard), driven by panfrost |
| RAM | 2 GB or 4 GB LPDDR3 (C201PA-DS01, C201PA-DS02) [source][SPEC] |
| Storage | 16 GB eMMC and a microSD slot |
| Display | 11.6-inch 1366×768 TN panel over eDP |
| Firmware | coreboot and depthcharge in a 4 MiB SPI NOR |

The board is fully upstream. `rk3288-veyron-speedy.dts` is in `arch/arm/boot/dts/rockchip/`
and builds as `rk3288-veyron-speedy.dtb` [source].

## Processor and memory

The RK3288-C has four Cortex-A17 cores. The A17 is a rebranded Cortex-A12 (r1p1 and later),
and it can report itself as an A12. The kernel carries the ARM errata 818325 and 852523
workarounds for the A12/A17 conditional-store deadlock [source].

The CPU runs the Veyron operating-point table, which adds 1704 MHz and 1800 MHz to the SoC
table and deletes 312 MHz. Frequency scaling is the generic `cpufreq-dt` driver. See
[shared hardware](../family-boards.md#shared-hardware) for the table.

The C201 ships with 2 GB or 4 GB of LPDDR3 [source][SPEC]. The device tree declares a fixed
2 GiB, and the firmware supplies the real size at boot. Use Debian's `armmp` kernel flavor,
not `armmp-lpae`. The memory node and the flavor are explained in
[shared hardware](../family-boards.md#shared-hardware) and
[Debian's armhf kernel](../family-boards.md#debians-armhf-kernel).

## Storage

| Node | Device | Details |
|---|---|---|
| `&emmc` | 16 GB eMMC | 8-bit, HS200, `mmc0` alias, `mmc-pwrseq-emmc` reset on gpio2 `RK_PB1` [source] |
| `&sdmmc` | microSD | `disable-wp` on speedy, card detect through `sdmmc_cd_pin` [source] |
| `&sdio0` | SDIO Wi-Fi | Non-removable, 4-bit, SDR104 [source] |
| `&spi2` | SPI NOR | `jedec,spi-nor`, label `spi_flash`: the firmware chip, a 4 MiB GD25Q32BSIG |

The eMMC holds 30777344 sectors, 14.68 GiB usable [reported][PR260]. On a mainline kernel it
is `mmcblk0`, with `mmcblk0boot0` and `mmcblk0boot1` alongside [measured]. Older kernels and
other Veyron boards number it differently. See
[the eMMC device node](../family-boards.md#the-emmc-device-node).

The factory partition table and the reserved region at the head of the eMMC are in
[the factory eMMC layout](../boot-chain.md#the-factory-emmc-layout). That section also
covers the primary GPT that ChromeOS marks `IGNOREME`.

## Networking

The C201 has no Ethernet port. The RK3288 has a GMAC, but no Veyron board wires it out
[source].

### Wi-Fi

The Wi-Fi is a Broadcom BCM4354 on SDIO, driven by `brcmfmac` [source].
`rk3288-veyron-speedy.dts` includes `rk3288-veyron-broadcom-bluetooth.dtsi`. The SDIO Wi-Fi
has no explicit device-tree child, because `brcmfmac` probes by SDIO ID. The part is an M.2
(NGFF) module, `BCM94354Z NGFF 22x30mm`, with `boardtype 0x0707`. Wi-Fi works on the
reference unit [measured].

The radio loads three firmware files. Debian packages the `.bin` in `firmware-brcm80211`.
Neither Debian nor linux-firmware carries the board NVRAM or the Bluetooth patchram. The
files, their sources, and the ChromiumOS NVRAM that belongs to a different module are in
[the Veyron Wi-Fi and Bluetooth firmware](../wifi-firmware.md).

### Bluetooth

Bluetooth is a `brcm,bcm43540-bt` on `&uart0`, with `max-speed = <3000000>` [source]. Its
host-wake, shutdown and device-wake GPIOs are on gpio4. Mainline drives it with the `hci_bcm`
serdev driver, which loads `brcm/BCM4354*.hcd`, a ChromeOS-sourced file absent from
linux-firmware.

The controller identifies itself at probe on the reference unit, but Bluetooth function is
unverified there [measured]. The postmarketOS device pages mark Bluetooth broken on speedy
specifically, and working on the generic Veyron page [reported]. Treat Bluetooth as an open
item, not as a consequence of the working Wi-Fi.

Two Bluetooth requirements are easy to miss, because Wi-Fi works without either:

- `btsdio` must be blacklisted.
- A host stack must be installed.

Both are explained in
[an HCI device is not a Bluetooth stack](../corrections-and-gotchas.md#an-hci-device-is-not-a-bluetooth-stack).

## Display

### Panel

The panel is an `innolux,n116bge`, 11.6 inches at 1366×768, driven over eDP by the Analogix
controller with a `pwm-backlight` [source]. Speedy sets `force-hpd` on `&edp` and deletes its
pinctrl. The eDP output takes its PHY from the `edp-phy` node, whose driver is the separate
`phy-rockchip-dp` module. See [the eDP PHY](../../display-reference.md#the-edp-phy).

### HDMI

The C201 has a micro-HDMI port, with its `&vcc50_hdmi` enable on gpio5 `RK_PC3` [source]. The
Debian wiki rates HDMI as working [reported][WIKI]. HDMI reaches 3840×2160 at 30 Hz. See
[HDMI output limits](../../display-reference.md#hdmi-output-limits).

A claim that the C201 has no video refers to a bug in the libreboot u-boot payload, which is
firmware, and not to the hardware. See
[the corrected claims](../corrections-and-gotchas.md#corrected-claims).

### Display controllers

Both VOPs are enabled on speedy: one drives the eDP panel and one drives HDMI [source]. The
two differ in maximum output, and HDMI can land on either one at runtime. See
[the two display controllers](../../display-reference.md#the-two-display-controllers).

## Audio

The sound card is `rockchip,rockchip-audio-max98090`, with a `maxim,max98090` codec and a
`ti,ts3a227e` headset-jack detector on `&i2c2` [source]. HDMI audio is routed through the same
card. The ALSA card is named `VEYRON-I2S`, from the `rockchip,model` property at
`rk3288-veyron-analog-audio.dtsi:14`.

On first boot the codec has no audio path. Its DAC is not routed to either amplifier, and
the amplifiers are muted through inverted mute bits [source]. Both are ordinary mixer state,
set once and persisted. The control list is in
[the max98090 starts with no audio path](../corrections-and-gotchas.md#the-max98090-starts-with-no-audio-path).

## Peripherals

| Function | Binding | Bus |
|---|---|---|
| Embedded controller | `google,cros-ec-spi` | `&spi0` |
| EC I2C tunnel | `google,cros-ec-i2c-tunnel` | Alias `i2c20` |
| Keyboard | `cros-ec-keyboard.dtsi` | Through the EC |
| Battery | `sbs,sbs-battery` (`cros-ec-sbs.dtsi`) | Through the EC I2C tunnel |
| Trackpad | `elan,ekth3000` | `&i2c4` |
| TPM | `infineon,slb9645tt` | `&i2c1` |
| PMIC | `rockchip,rk808` at `0x1b`, `system-power-controller` | `&i2c0` |
| Lid switch | `gpio-keys`, `SW_LID` | gpio0 `RK_PA6` |
| Power button | `gpio-keys`, `KEY_POWER` | gpio0 `RK_PA5` |
| Charger | `gpio-charger` (mains) | gpio0 `RK_PB0` |

The keyboard and the battery arrive through the ChromeOS EC, and only mains presence
reaches the SoC directly, as a `gpio-charger` on gpio0 `RK_PB0` [source].
`CONFIG_CROS_EC*` is therefore required, as the board's input and power stack.

## USB

The C201 has two USB 2.0 ports and no USB 3.0 [source]. They use `USB_DWC2` with the legacy
`PHY_ROCKCHIP_USB` PHY, not `PHY_ROCKCHIP_INNO_USB2`.

## Serial console

The console is `stdout-path = "serial2:115200n8"`: UART2 at 115200 8N1 [source]. The setting
travels in the DTB. This rate differs from the 1500000 baud of RK35xx boards. The UART is
unwired on the board.

## Video decode and GPU

The C201 enables both RK3288 codec blocks [source]:

- Hantro, for JPEG encode, and MPEG-2, VP8 and H.264 decode
- rkvdec, for HEVC decode

The Veyron device trees override neither `&vpu` nor `&hevc`. The GPU is the Mali-T764 under
panfrost, enabled in `rk3288-veyron.dtsi`. Kernel configuration and detail are in
[the RK3288 video and GPU reference](../../video-reference.md).

## Limits

### Suspend

Suspend is shallow. Mainline implements no DDR self-refresh for the RK3288, and the ChromeOS
3.14 kernel does [reported]. The light suspend costs ~1.5-2% battery per hour, as velvet-os
measures it [reported].

### HDMI

HDMI tops out at 4K30. The 4K60 mode needs a TMDS clock above the PHY's 340 MHz ceiling,
and a YUV420 output mode the VOP does not have. No driver change reaches it. See
[HDMI output limits](../../display-reference.md#hdmi-output-limits).

### GPU memory pressure

velvet-os reports panfrost allocation problems under Xorg with glamor [reported]. See
[GPU](../../video-reference.md#gpu).

### Audio after resume

Sound is reported lost across a suspend and resume cycle [reported].

The codec's silence on first boot is not a limit. It is mixer state, described under
[Audio](#audio).

## Sources

Every `[source]` claim without a key rests on the mainline device trees,
`arch/arm/boot/dts/rockchip/rk3288-veyron*` (v7.1), or on the mainline driver it names.

Keys:

- `[DCT]`: `depthcharge-tools` 0.6.2-5, `boards.ini`.
- `[SPEC]`: the ASUS retail specifications of the C201PA.
- `[PR260]`: [PrawnOS#260](https://github.com/SolidEva/PrawnOS/pull/260), `fdisk -l` output
  from stock ChromeOS units.
- `[WIKI]`: the Debian, Gentoo and postmarketOS wikis.

The `[reported]` suspend figures and the panfrost report come from velvet-os/imagebuilder
(hexdump0815). The Bluetooth status comes from the postmarketOS device pages.
