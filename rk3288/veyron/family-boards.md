# Veyron family boards

This document describes the Veyron family, the RK3288 Chromebooks built on
`rk3288-veyron.dtsi`. It states what the boards share, how speedy (the ASUS C201), minnie
(the ASUS C100P) and mickey (the ASUS Chromebit CS10) differ, and where the family departs
from Debian's stock kernel. Its sources are the mainline device trees (v7.1), the shipping
ChromeOS firmware configuration (`firmware-veyron-6588.B`) and the `depthcharge-tools` board
database (0.6.2-5, the version in Debian forky). How the boards boot is in
[the Veyron boot chain](boot-chain.md).

## The in-tree boards

Mainline carries ten Veyron boards. All share `rk3288-veyron.dtsi`, and each differs only
by its own `.dts` [source]:

| DTS | Model | Product | Radio |
|---|---|---|---|
| `rk3288-veyron-speedy` | Google Speedy | ASUS Chromebook C201 | Broadcom |
| `rk3288-veyron-minnie` | Google Minnie | ASUS Chromebook Flip C100P | Broadcom |
| `rk3288-veyron-mickey` | Google Mickey | ASUS Chromebit CS10 | Broadcom |
| `rk3288-veyron-jerry` | Google Jerry | Hisense Chromebook C11, Haier, Poin2 | Marvell |
| `rk3288-veyron-jaq` | Google Jaq | Haier Chromebook 11e | Marvell |
| `rk3288-veyron-mighty` | Google Mighty | (`#include`s jaq) | Marvell |
| `rk3288-veyron-brain` | Google Brain | Development board | Broadcom |
| `rk3288-veyron-pinky` | Google Pinky | Reference board | |
| `rk3288-veyron-fievel` | Google Fievel | | Marvell |
| `rk3288-veyron-tiger` | Google Tiger | | Marvell |

The radio splits the family in two, and it is the main difference between boards [source]:

- Broadcom BCM4354 (`brcmfmac`): speedy, minnie, mickey and brain. The board NVRAM is in
  neither Debian nor linux-firmware. See [the Veyron Wi-Fi firmware](wifi-firmware.md).
- Marvell SD8897 (`mwifiex`): jerry, jaq, mighty, fievel and tiger. Debian packages the
  firmware, `mrvl/sd8897_uapsta.bin`, in `firmware-libertas`.

The Marvell boards therefore install their radio firmware from Debian alone, and the
Broadcom boards need files from outside it.

## Shared hardware

Every board includes `rk3288-veyron.dtsi`, and the laptops add
`rk3288-veyron-chromebook.dtsi`. The include tree decides which hardware a board
has [source]:

```
rk3288-veyron.dtsi                  emmc, sdio0 (Wi-Fi), hdmi, vopb, spi2 (SPI NOR), USB
 |
 |-- mickey                         includes this file directly, plus broadcom-bluetooth
 |
 '-- rk3288-veyron-chromebook.dtsi  the laptop parts
       +-- analog-audio.dtsi        max98090 and ts3a227e headset codec
       +-- edp.dtsi                 eDP panel, backlight, vopl
       +-- sdmmc.dtsi               the SD card slot, the mmc1 alias, vcc33_sd and vccio_sd
       +-- cros-ec-keyboard.dtsi    the internal keyboard
       +-- cros_ec on &spi0         the EC: trackpad, lid, battery tunnel
       |
       |-- minnie                   plus broadcom-bluetooth
       '-- speedy                   plus broadcom-bluetooth and cros-ec-sbs
```

Mickey skips `rk3288-veyron-chromebook.dtsi`, and every mickey entry in
[the per-board differences](#per-board-differences) follows from that one omission. Mickey
is a separate design, not a laptop with parts removed.

Speedy, minnie and mickey are identical in the following, so each is a family
property [source]:

- The RK808 PMIC chain
- The eMMC
- The SDIO radio wiring and its power sequence
- Bluetooth on `uart0` (`brcm,bcm43540-bt`)
- The SPI NOR firmware flash on `spi2`
- The TPM
- The Mali GPU, described in [the RK3288 GPU reference](../video-reference.md#gpu)
- The CPU and GPU operating-point tables
- The serial console, `stdout-path = "serial2:115200n8"`

### Memory

`rk3288-veyron.dtsi` declares a fixed 2 GiB memory node [source]:

```
memory { reg = <0x0 0x0 0x0 0x80000000> }
```

A comment beside it explains the literal:

`the default coreboot on veyron devices ignores memory@0 nodes and would instead create another memory node`

The firmware supplies the real memory size at boot, which is how 4 GB units work. Leave the
2 GiB value as it is.

### CPU operating points

`rk3288-veyron.dtsi` overrides the SoC operating-point table [source]. It adds 1704 MHz and
1800 MHz and deletes 312 MHz, with the comment
`rk3288-c used in Veyron Chrome-devices has slightly changed OPPs`. Frequency scaling is the
generic `cpufreq-dt` driver, with no SoC-specific driver.

### PMIC

The PMIC is an RK808 (`rockchip,rk808`) on `i2c0`. Its bucks supply `vdd_cpu` and `vdd_gpu`,
and its LDOs supply the SD card slot's rails [source]. Those rails make the PMIC's driver
chain an initramfs requirement. See [the PMIC driver chain](white-screen.md#the-pmic-driver-chain).

### Networking

No Veyron board has Ethernet. The RK3288 has a GMAC, but the boards do not wire it out
[source]. The SDIO radio is the network path. On the Broadcom boards, Bluetooth is a
`brcm,bcm43540-bt` on `uart0`.

### Serial console

The console is `stdout-path = "serial2:115200n8"`: UART2 at 115200 8N1 [source]. The UART is
unwired on speedy, minnie and mickey. Its rate differs from the 1500000 baud of RK35xx boards.

## Per-board differences

The three boards differ as follows [source]:

| | speedy (C201) | minnie (C100P) | mickey (Chromebit CS10) |
|---|---|---|---|
| Product | Chromebook C201PA | Chromebook Flip C100PA | Chromebit CS10 |
| SD slot | microSD | microSD | None: `&sdmmc` stays `disabled`, and there is no `mmc1` alias |
| eMMC | 16 GB, `mmcblk0` | 16 GB, `mmcblk0` | 16 GB, `mmcblk0` |
| USB ports | 2 | 2 | 1: `usb_host0_ehci` and `usb_host1` are disabled, leaving only `usb_otg` (dwc2) |
| Keyboard | EC on `spi0` | EC on `spi0` | None: `&spi0` is disabled, and there is no `cros_ec` |
| Trackpad | Elan ekth3000 | Elan ekth3000 | None |
| Touchscreen | None | Elan ekth3500 on `i2c3` | None |
| Other input | None | Volume buttons | Power-button input only, and no power switch (see [Entering developer mode](#entering-developer-mode)) |
| Panel | eDP 1366×768, `innolux,n116bge` | eDP 1280×800, `auo,b101ean01` | None: HDMI only, with `vopl` and `edp` disabled |
| Backlight | Scale 0-255 | Scale 3-255 (the PWM must be >= 1%) | None |
| Audio | max98090 | max98090 | HDMI only |
| ALSA card (driver default name) | `ROCKCHIP-MAX98090-HDMI` | `ROCKCHIP-MAX98090-HDMI` | `ROCKCHIP-HDMI` |
| Battery | `sbs-battery` | `ti,bq27500` | None (mains) |
| Thermal | Trips lowered (critical at 90) | Inherited (critical at 100) | A fanless ladder of its own (critical at 90) |
| Radio | BCM4354 | BCM4354 | BCM4354 |
| DTB | `rk3288-veyron-speedy.dtb` | `rk3288-veyron-minnie.dtb` | `rk3288-veyron-mickey.dtb` |

The `rockchip,model` property overrides the driver's default card name. On speedy and
minnie, `rk3288-veyron-analog-audio.dtsi` sets it to `VEYRON-I2S` [source].

### The SDMMC_DET_L line name on mickey

`rk3288-veyron-mickey.dts` names a GPIO line `"SDMMC_DET_L"`. It is a pin label only: no
`sdmmc` node is enabled, and no `cd-gpios` property consumes it [source]. A search for
`SDMMC` therefore returns a false positive on mickey.

## The eMMC device node

On a mainline kernel the eMMC is `mmcblk0` on every Veyron board, because
`rk3288-veyron.dtsi` carries `aliases { mmc0 = &emmc; }` [source]. On the boards with an SD
slot, `rk3288-veyron-sdmmc.dtsi` adds `mmc1 = &sdmmc`. On the reference C201 under Debian's
7.1.3 `armmp` kernel, the eMMC is `mmc0` and `mmcblk0`, with `mmcblk0boot0` and
`mmcblk0boot1` alongside. The SD slot is `mmc1`, and the SDIO Wi-Fi is `mmc2`
[measured, 7.1.3].

Under the stock ChromeOS 3.14 kernels the node varies by board. `fdisk -l` output from two
units running stock ChromeOS shows it [reported][PR260]:

| Board | Product | eMMC node under stock ChromeOS |
|---|---|---|
| mickey | ASUS Chromebit CS10 | `/dev/mmcblk1` |
| minnie | ASUS Chromebook Flip C100P | `/dev/mmcblk2` |

The mechanism is board topology. The Chromebit has no SD slot, so one fewer mmc controller
registers ahead of the eMMC, and every index shifts down. PrawnOS#260 (merged 2020-12-16)
fixed exactly this bug in the PrawnOS install scripts. Community sources disagree on the
node [reported]:

- The Debian wiki gives `mmcblk0` [WIKI].
- The Gentoo wiki says it varies by kernel version, between `mmcblk0` and `mmcblk2` [WIKI].
- PrawnOS gives `mmcblk2`.

Name no device node in the kernel command line. The mainline alias is a property of the
family's device tree, and an image moves between media. Root on the root partition's own
PARTUUID instead:

- It travels with the partition, so one image boots from SD, USB or eMMC on any board
  unchanged.
- It needs no custom code at any layer.
- `depthcharge-tools` derives it from `/etc/fstab` by itself.

The `PARTNROFF` idiom from the Veyron guides does not work under a Debian initramfs. See
[PARTNROFF does not work under initramfs-tools](corrections-and-gotchas.md#partnroff-does-not-work-under-initramfs-tools).
To identify the eMMC from a script, look for its `mmcblkNboot0` device, as PrawnOS does.

## Debian's armhf kernel

Debian's armhf `armmp` kernel boots the C201 unmodified. Forky (7.1.x) and trixie (6.12.x)
both reach a login prompt with working Wi-Fi [measured, 2026-07-14]. Use the `armmp` flavor,
not `armmp-lpae`. The Cortex-A17 implements LPAE, but with 2 GiB and a range the device tree
declares, there is nothing to gain [source].

The `boot/config-*` of Debian's `linux-image-armmp`, read in both suites, leaves two minnie
drivers unbuilt [source]:

| Board hardware | Driver | Symbol | forky (7.1.x) | trixie (6.12.x) |
|---|---|---|---|---|
| minnie touchscreen (`elan,ekth3500`) | `elants_i2c` | `TOUCHSCREEN_ELAN` | Not set | Not set |
| minnie fuel gauge (`ti,bq27500`) | `bq27xxx_battery_i2c` | `BATTERY_BQ27XXX_I2C` | Not set | Not set |
| minnie and speedy trackpad (`elan,ekth3000`) | `elan_i2c` | `MOUSE_ELAN_I2C` | `=m` | `=m` |
| speedy battery | `sbs-battery` | `BATTERY_SBS` | `=m` | `=m` |

On a stock Debian kernel, therefore, the C100P has no touch input and no battery percentage.
The rest of its hardware has drivers in that kernel [source]. The C201 and the Chromebit are
unaffected: neither has a touchscreen, the C201's battery is SBS, and the Chromebit has no
battery.

The gap comes from Debian's configuration history rather than from the hardware. Debian bug
#836251 (2016) enabled armhf Veyron support for veyron-speedy specifically. That is where
`BATTERY_SBS` and `MOUSE_ELAN_I2C` come from, and minnie is not covered. `BATTERY_BQ27XXX_I2C`
is off on every Debian architecture, overriding an upstream `default y`.

Only the I2C glue is off: `BATTERY_BQ27XXX=m` and `BATTERY_BQ27XXX_HDQ=m` are both set. The
trackpad driver is a different Elan driver from the touchscreen one. A working Elan
trackpad is therefore no evidence about the touchscreen either way.

Forky's armhf kernel (7.1.x) also leaves two symbols unset that the C201's device
tree uses [source]:

- `CONFIG_REGULATOR_PWM`: the `vdd_logic` pwm-regulator does not probe. The impact appears
  low, because nothing in the Veyron device trees consumes `vdd_logic`, and `vdd_cpu` and
  `vdd_gpu` are RK808 bucks. The driver is still absent. Debian's arm64 configuration has
  it, and its armhf configuration does not.
- `CONFIG_TCG_TIS_I2C_INFINEON`: the on-board TPM is unusable. It does not block boot.

Upstream `multi_v7_defconfig` carries both missing minnie symbols, so a kernel built from
mainline closes that gap with no patch. Such a kernel still has to fit the payload ceiling.
A kernel with the boot chain built in is larger, not smaller, so it does not drop into a
16 MiB slot beside a systemd initramfs. See [the payload budget](boot-chain.md#the-payload-budget).

## The depthcharge-tools board profiles

`depthcharge-tools` describes each board in `boards.ini` (0.6.2-5, byte-identical to
upstream master) [source]. Inheritance runs `[boards/arm]` -> `[boards/arm/rk3288]` ->
`[boards/arm/rk3288/veyron]` -> the board.

Every payload-shaping key is inherited from `[boards/arm/rk3288/veyron]`, and is identical
on speedy, minnie and mickey:

| Key | Value | Consequence |
|---|---|---|
| `image-max-size` | 16 MiB | The payload ceiling, which the firmware enforces |
| `image-start-address` | `0x2000000` | The base for the `/chosen` initrd address |
| `loads-fit-ramdisk` | `False` | `mkdepthcharge` must patch `linux,initrd-start\|end` into every DTB |
| `boots-lz4-kernel`, `boots-lzma-kernel` | `False`, `False` | The kernel goes into the FIT uncompressed |

Each board's own stanza is four lines, `codename`, `name`, `hwid-match` and `dt-compatible`.
They are metadata and DTB matching, and nothing in them changes the build.

`speedy/libreboot` is the only libreboot profile in the 2903-line file. It sets a 32 MiB
ceiling and `loads-fit-ramdisk = True`. Minnie and mickey have no libreboot profile, so both
are stock-firmware-only, and on both the `crossystem dev_boot_usb=1` step is mandatory. Only
a libreboot C201 can skip it.

A profile and a kernel-slot size come as a pair. A payload can use the firmware's buffer
only up to the size of the partition that holds it. A stock-profile image boots on stock and
on libreboot firmware, and a libreboot-profile image boots only on libreboot. The libreboot
profile carries `hwid-match = None`, so a running `depthchargectl` cannot autodetect it and
resolves to the plain `speedy` constraints. What the larger ceiling buys is in
[the payload budget](boot-chain.md#the-payload-budget).

## Firmware input drivers

The firmware's input drivers decide which keyboard works at the firmware screens, and so how
a board is installed. The shipping `firmware-veyron-6588.B` branch configures them as
follows [source]:

| Board | depthcharge `defconfig` | libpayload `CONFIG_LP_USB_HID` | At the Ctrl+U screen |
|---|---|---|---|
| speedy | `CONFIG_DRIVER_INPUT_MKBP=y` | Not set | The EC keyboard only. A USB keyboard does nothing. |
| minnie | `CONFIG_DRIVER_INPUT_MKBP=y` | Not set | The EC keyboard only |
| mickey | `CONFIG_DRIVER_INPUT_USB=y`, with no EC driver | `=y`, plus `CONFIG_LP_USB_HUB=y` | A USB keyboard only, and a keyboard behind a hub works |

Firmware USB HID is enabled on mickey, the one board of the three with no internal
keyboard. None of the three firmware configurations has a Bluetooth driver, so a paired
Bluetooth keyboard does nothing at the firmware screens on any board. A 2.4 GHz keyboard with
its own USB receiver works on mickey, because it enumerates as USB HID.

### The Chromebit install topology

The Chromebit has one USB port, and an install needs a keyboard and the boot stick in it at
once. A hub is therefore a prerequisite. It works because `LP_USB_HUB=y` with `LP_USB_HID=y`
is exactly that topology. Use a self-powered hub: ASUS's documentation warns against drawing
more than 500 mA from that port.

## Entering developer mode

The procedure differs by board. The volume-button combination Google documents is for
detachables and tablets, and the Veyron firmware is not built as one. The convertible
minnie's volume buttons therefore play no part in it.

### Speedy and minnie

1. Power the laptop off.
2. Hold Esc and Refresh, and tap Power.
3. At the recovery screen, press Ctrl+D.
4. Press Enter to confirm.

### Mickey

Mickey has no power button, and it boots the instant DC is applied. Its recovery button is a
pinhole on the underside, opposite the HDMI plug, and it needs a paperclip. To enter
developer mode:

1. Hold the recovery pinhole and, still holding it, plug in the barrel jack.
2. Press Ctrl+D on the USB keyboard.
3. Press the recovery button again to confirm.
4. Press Ctrl+D again.

On a device with no keyboard, the firmware treats the recovery button as Enter, which is why
step 3 confirms. The wipe takes 10-15 minutes.

## Open risks

The open risks are per board:

- Mickey's USB boot has public failure reports [reported]. A 2015 review failed to boot a
  stick, and an unanswered forum thread reports Ctrl+U flashing black on a postmarketOS
  image. Both fit malformed boot media better than a firmware limit, such as a kernel
  partition not in ChromeOS format or unset GPT attribute bits [inferred]. Neither is
  explained.
- Mickey has no public confirmation of an eMMC install. The one public attempt fails with
  `Primary GPT header is being ignored`, the sign of a preserved factory GPT [reported]. A
  whole-disk image does not hit this. See
  [the factory eMMC layout](boot-chain.md#the-factory-emmc-layout).
- Minnie has no open risk of its own beyond [its kernel gaps](#debians-armhf-kernel). Like
  the C201, it has a card slot, a keyboard and two USB ports, and postmarketOS installs to
  its eMMC [reported].

Every Veyron measurement comes from one C201, the reference unit. Claims about minnie and
mickey are `[source]` unless they name other evidence.

## Sources

Every `[source]` claim in this document rests on one of the following:

- Mainline Linux `arch/arm/boot/dts/rockchip/rk3288-veyron*` (v7.1).
- The ChromeOS `firmware-veyron-6588.B` branch: the depthcharge and libpayload
  configurations of each board.
- `depthcharge-tools` 0.6.2-5 `boards.ini`, as Debian forky ships it.
- The kernel configuration (`boot/config-*`) of Debian's armhf `linux-image-armmp`, forky
  and trixie.
- Debian bug #836251.

Keys:

- `[PR260]`: [PrawnOS#260](https://github.com/SolidEva/PrawnOS/pull/260), `fdisk -l` output
  from stock ChromeOS units, merged 2020-12-16.
- `[WIKI]`: the Debian, Gentoo and postmarketOS wikis.
