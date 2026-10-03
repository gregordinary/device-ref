# RK3288 video and GPU reference

This document describes the RK3288 video decoders and GPU under mainline Linux, at the SoC
level. It covers the two codec blocks, their kernel configuration and their device-tree
state, and the Mali-T764 GPU with its driver. The Veyron Chromebooks are the boards it cites
for device-tree state. Display output is in
[the RK3288 display reference](display-reference.md).

## Video decoders

The RK3288 has two codec blocks, and mainline drives both [source]:

| Block | Compatible | Kconfig | RK3288 capabilities |
|---|---|---|---|
| Hantro VPU | `rockchip,rk3288-vpu` | `CONFIG_VIDEO_HANTRO`, `CONFIG_VIDEO_HANTRO_ROCKCHIP` | JPEG encode, and MPEG-2, VP8 and H.264 decode |
| rkvdec | `rockchip,rk3288-vdec` | `CONFIG_VIDEO_ROCKCHIP_VDEC` | HEVC decode only, up to 4096×2304 |

The RK3288 rkvdec variant is in mainline from v6.19 / v7.0. Two changes carry it [source]:

- The driver variant, `media: rkvdec: Add RK3288 variant` (September 2025)
- The device-tree node, `ARM: dts: rockchip: Add vdec node for RK3288`

The rkvdec driver is a mainline media driver outside staging.

### Device-tree state

`rk3288.dtsi` declares both blocks, as `&vpu` and `&hevc`. The Veyron device trees override
neither node, so both inherit their `rk3288.dtsi` defaults and are enabled [source].

### Kernel configuration

Debian forky's armhf kernel builds both drivers: `VIDEO_HANTRO=m`, `VIDEO_HANTRO_ROCKCHIP=y`
and `VIDEO_ROCKCHIP_VDEC=m` [source]. Mainline `multi_v7_defconfig` enables neither. A kernel
built from that defconfig needs a configuration fragment that adds them.

A linux-libre kernel leaves both drivers intact, because the deblob log has no hits for
`hantro` or `rockchip_vdec` [source]. The Veyron firmware reference lists what linux-libre
does remove. See [linux-libre kernels](veyron/wifi-firmware.md#linux-libre-kernels).

## GPU

The RK3288 GPU is a Mali-T764, a Midgard part, and its mainline driver is panfrost [source].
Panthor and lima do not drive it.

`rk3288.dtsi` declares the GPU node and disables it by default. Its compatible is
`"rockchip,rk3288-mali", "arm,mali-t760"`. A board enables it. On the Veyron boards,
`rk3288-veyron.dtsi` sets `&gpu` to enabled with `mali-supply = <&vdd_gpu>` [source].

The Mali-T764 needs no firmware, unlike the CSF Mali parts that linux-libre disables. A
linux-libre kernel leaves panfrost intact, and the deblob log has no hits for it [source].

Under Xorg with glamor, velvet-os reports panfrost allocation problems with a browser and
several tabs open [reported]. The velvet-os images ship a `blacklist-panfrost.conf`,
commented out, as an escape hatch.

## Sources

Every `[source]` claim in this document rests on one of the following:

- Mainline Linux `arch/arm/boot/dts/rockchip/` (`rk3288.dtsi`, `rk3288-veyron*.dts*`, v7.1),
  and `arch/arm/configs/multi_v7_defconfig`.
- The mainline commits `media: rkvdec: Add RK3288 variant` and
  `ARM: dts: rockchip: Add vdec node for RK3288`.
- The kernel configuration of Debian forky's armhf `linux-image-armmp`, from the unpacked
  package.
- The linux-libre deblob log.

The `[reported]` claim comes from velvet-os/imagebuilder (hexdump0815), a Debian-based
image builder for these boards.
