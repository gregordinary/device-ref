# RK3288 display reference

This document describes the RK3288 display path under mainline Linux, at the SoC level. It
covers the two VOP display controllers, the HDMI output ceiling and the eDP PHY, and it
holds for every RK3288 board. The Veyron Chromebooks are the worked example, because they
light both display controllers at once. Board wiring lives in each board's own reference,
for example [the ASUS C201 hardware reference](veyron/asus-c201/hardware-reference.md).

## The two display controllers

The RK3288 display controller is the first-generation VOP, driven by `CONFIG_ROCKCHIP_VOP`
[source]. VOP2 is a different block, found on the RK3568 and later SoCs.

The SoC carries two VOPs, and they differ in output size [source]:

| Controller | Maximum output |
|---|---|
| VOPB ("big") | 3840×2160 |
| VOPL ("little") | 2560×1600 |

The driver cannot tell the two apart, so both advertise the VOPB limit. A comment on
`rk3288_vop.max_output` (`rockchip_vop_reg.c:775-779`) states it [source]. The limit is the
VOPB's, and the VOPL can only do 2560×1600, "but we can't distinguish them as they have the
same compatible".

### Runtime CRTC assignment

The SoC device tree does not bind HDMI to one VOP. `rk3288.dtsi:1246-1262` exposes both
`hdmi_in_vopb` and `hdmi_in_vopl`, and the HDMI driver picks one at runtime. It writes the
GRF `lcdsel` bit to match (`dw_hdmi-rockchip.c:289-309`) [source].

On the Veyron Chromebooks both VOPs are lit, one for the eDP panel and one for HDMI. Nothing
in their device trees pins HDMI to the VOPB [source]. When DRM lands HDMI on the VOPL, a 4K
mode is advertised and accepted, and then rendered by a controller limited to 2560×1600.
That matches older community reports of a partial or tiled screen at 4K30 [inferred].

The assignment is a runtime CRTC choice, so no kernel configuration or device-tree setting
pins it. Either of these shows which controller HDMI received:

- `dmesg | grep -i vop`
- `/sys/kernel/debug/dri/*/state`

## HDMI output limits

Mainline drives RK3288 HDMI up to 3840×2160 at 30 Hz, a 297 MHz pixel clock [source]. Debian's
forky (7.1.x) and trixie (6.12.x) kernels both support it with nothing to configure, patch or
enable. The limits are compile-time constants in the driver, not kconfig symbols or
device-tree properties.

4K30 support needs no out-of-tree patch. Two mainline commits carry it, and both are in
Debian's forky and trixie kernels [source]:

- `drm/rockchip: dw_hdmi: Add support for 4k@30 resolution` (`83b61f817f43`, February 2023)
- `drm/rockchip: vop: limit maximum resolution to hardware capabilities` (`8e140cb60270`)

Three ceilings apply, all set in mainline source [source]:

| Ceiling | Value | Where it is set |
|---|---|---|
| TMDS clock | 340 MHz | `rk3288_chip_data.max_tmds_clock = 340000` (`drivers/gpu/drm/rockchip/dw_hdmi-rockchip.c:437`), enforced in `dw_hdmi_rockchip_mode_valid()` (`:241-243`) |
| VOP output size | 3840×2160 | `rk3288_vop.max_output` (`drivers/gpu/drm/rockchip/rockchip_vop_reg.c:765-781`) |
| PHY and MPLL tables | A 297 MHz entry | `rockchip_phy_config` (`dw_hdmi-rockchip.c:193`) and the 340 MHz MPLL bucket (`:160-164`) |

The TMDS cap is the HDMI 1.4 ceiling. It passes 297 MHz (4K30) and rejects 594 MHz (4K60
RGB). The 297 MHz clock is exact rather than approximated: `PLL_GPLL = 594000000`
(`rk3288.dtsi:868-877`) divided by 2 is 297.000 MHz.

### 4K60

4K60 is a hardware ceiling, and no patch reaches it [source]. The PHY has no SCDC or
scrambling above 340 MHz. The VOP cannot emit YUV420, because the encoder hard-codes
`ROCKCHIP_OUT_MODE_AAAA` (`dw_hdmi-rockchip.c:319`), so there is no half-rate path either.

Upstream raised `max_tmds_clock` to 594000 for the RK3399 and the RK356x (`28f0ae48e7fd`,
September 2024). It left the RK3288 at 340000 deliberately. A patch that raises it produces
a mode the hardware cannot transmit.

## The eDP PHY

The eDP output takes its PHY from a node of its own [source]. `rk3288.dtsi` declares
`edp_phy: edp-phy`, with the compatible `rockchip,rk3288-dp-phy` and the clock
`SCLK_EDP_24M`. The node sits inside the GRF syscon at `0xff770000`. `rk3288.dtsi` disables
it, and a board enables it: on the Veyron Chromebooks, `rk3288-veyron-edp.dtsi` does.

The eDP controller, `dp@ff970000`, names the PHY in its `phys` property. The PHY driver is a
separate module, `phy-rockchip-dp` (`CONFIG_PHY_ROCKCHIP_DP`, `=m` in Debian's armhf kernel).
The eDP output waits on it at probe, in two steps [source]:

1. `analogix_dp_probe()` takes the PHY with `devm_phy_get()`, which defers until
   `phy-rockchip-dp` registers it.
2. rockchipdrm binds its display device only once every output it matched has probed
   (`rockchip_drm_match_add()`).

Without the PHY module the whole display stack stays unbound, both VOPs and HDMI included.
No symbol dependency pulls `phy-rockchip-dp` in, so a module set built by following symbol
dependencies misses it. An initramfs that carries the display stack must name it. See
[the display stack in the initramfs](veyron/white-screen.md#the-display-stack-in-the-initramfs).

## Kernel configuration

Debian forky's armhf kernel builds the whole eDP panel path of the Veyron
Chromebooks [source]:

| Symbol | Debian forky armhf |
|---|---|
| `CONFIG_DRM_ROCKCHIP` | `=m` |
| `CONFIG_ROCKCHIP_VOP` | `=y` |
| `CONFIG_ROCKCHIP_ANALOGIX_DP` | `=y` |
| `CONFIG_PHY_ROCKCHIP_DP` | `=m` |
| `CONFIG_DRM_PANEL_EDP` | `=m` |
| `CONFIG_BACKLIGHT_PWM` | `=m` |
| `CONFIG_PWM_ROCKCHIP` | `=m` |

The framebuffer console (fbcon) is enabled in the same kernel.

## Sources

Every `[source]` claim in this document rests on one of the following:

- Mainline Linux `drivers/gpu/drm/rockchip/` (`dw_hdmi-rockchip.c`, `rockchip_vop_reg.c`)
  and `arch/arm/boot/dts/rockchip/` (`rk3288.dtsi`, `rk3288-veyron*.dts*`, v7.1). Line
  numbers locate the cited code.
- The mainline commits `83b61f817f43`, `8e140cb60270` and `28f0ae48e7fd`.
- The kernel configuration of Debian's armhf `linux-image-armmp`, forky (7.1.3+deb14) and
  trixie (6.12.x), from the unpacked packages.
