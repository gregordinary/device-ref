# Prose profile: device-ref

A portable style guide holds the general rules. This document holds what that guide asks
each project to supply, because the portable core cannot know it. Where this document is
silent, the core governs. A profile refines the core and never relaxes it.

## What these documents are

Each document is a vendor-style hardware reference. It states what a SoC, a board family
or a board is and does. It also states what an engineer integrating an operating system
onto it has to know. It reads the way a vendor's own reference manual reads.

So a document never describes how a particular image is built. It carries none of the
following:

- An image builder's choices
- Patch numbers or patch series names
- Lock or pin state
- The contents of a particular built image
- An account of how a finding was reached

A fix is described technically: what change it makes, and in which driver or device-tree
node. Measurement provenance stays, because it is evidence.

## Reader and depth budget

| Surface | Reader | Depth |
|---|---|---|
| SoC reference (`<soc>/*.md`) | An engineer writing a device tree, a bootloader port or a driver for any board on that SoC | As deep as the mechanism goes, scoped to the SoC |
| Family reference (`<soc>/<family>/*.md`) | The same engineer, on any board of the family | What the family shares, and what differs board by board |
| Board hardware reference (`hardware-reference.md`) | The same engineer, holding that board | What the board is: wiring, addresses, parts, limits |
| Corrections and gotchas | Someone about to repeat a widely held mistake | The claim, the fact, the evidence, and the rule that follows |
| Defect record (a page for one defect) | Someone hitting that defect, or about to change the code around it | Symptom, mechanism, work-around, and the avenues measured and refuted |
| Status page | Someone deciding what works on which kernel | One row per subsystem, with the kernel and the evidence grade |
| `README.md` | Whoever arrives first | Where each kind of fact lives |

## Fixed terms

<!-- prose-lint: off -- the third column is the list of ruled-out variants, which the gate would otherwise flag in its own table. -->

| Concept | Write | Not |
|---|---|---|
| The physical board a measurement was taken on | the reference unit | our unit, the target unit, the target C201, our own unit |
| Software that assembles a bootable disk image | an image builder | boot2deb |
| A kernel or bootloader built from the projects' own trees | mainline | our kernel, our u-boot, our bootloader, the shipped kernel, the shipped image, this project's |
| Rockchip's downstream Linux tree | the vendor kernel | the BSP kernel |
| A structure containing a field | carries | has a field, holds a field, contains a field |

<!-- prose-lint: on -->

## Evidence

A claim carries its grade in a lowercase tag. The grades are fixed here and used
literally:

| Tag | Meaning |
|---|---|
| `[measured]` | Observed on the reference unit. Carries the date and the kernel where the result depends on them: `[measured, 2026-10-02, 7.2.8]`. |
| `[source]` | Read from source code, a device tree, a datasheet or a design guide. |
| `[consensus]` | Reported working by more than one independent community source. |
| `[reported]` | Reported by one community source and not reproduced. |
| `[inferred]` | A mechanism reasoned from evidence and not isolated. |
| `[expected]` | A prediction nobody has tested. |

A named source is cited with an uppercase key, `[DTS]` or `[HDG]` for example. Each
document defines its keys once, in a section titled **Sources** at its end, together with
the path or link that holds the artifact. A key and a grade can stand together:
`[measured][LOG]`.

Claims that are true only inside a boundary:

- A measurement holds for the reference unit, at the kernel it names.
- An RK3576 measurement comes from one board, the H96 MAX M9. A claim stated for the SoC
  rests on that board unless the text says otherwise.
- An RK3588 measurement comes from one board, the Turing RK1. A claim stated for the SoC
  rests on that board unless the text says otherwise.
- A Veyron measurement comes from one C201. A claim about minnie or mickey is `[source]`
  unless it names other evidence.

## Surfaces that repeat one claim

Where a fact is relevant to several documents, one document owns it and the others link
to it.

| Claim | Owner |
|---|---|
| The RK3576 HDMI bring-up contract and the VOP handover | `rk3576/display-reference.md` |
| IO domains, pin-mux voltage domains, boot straps, the combo PHY 1 coupling | `rk3576/hardware-design-guide.md` |
| The rockusb gadget, mode detection and the 32 MiB read wall | `rk3576/rockusb-reference.md` |
| The RK3576 boot firmware: BL31, OP-TEE, the download key, reboot modes | `rk3576/boot-firmware.md` |
| RK3576 decode, RGA, GPU and the 10-bit ceiling | `rk3576/video-reference.md` |
| The RK3576 NPU | `rk3576/npu-reference.md` |
| The H96 AIC8800D80 Wi-Fi and Bluetooth | `rk3576/h96-max-m9/wifi-bluetooth.md` |
| The RK3588 boot chain, BL31's PSCI, SCMI and SiP services, and u-boot's handoff | `rk3588/boot-firmware.md` |
| The RK3588 cores, ID registers, caches, coherency, errata, CPU operating points, delivered frequency and the DSU clock | `rk3588/cpu-reference.md` |
| The RK3588 address map, GIC, timers, UARTs, TRNG, IOMMUs, DMA controllers, DRAM bandwidth and thermal sensors | `rk3588/system-reference.md` |
| The RK3588 PMU wake sources, PMU timer and system suspend | `rk3588/system-sleep.md` |
| RK3588 decode, encode, RGA, GPU and the HDMI receiver | `rk3588/video-reference.md` |
| The RK3588 NPU | `rk3588/npu-reference.md` |
| The Turing RK1 memory map, console and management-controller paths | `rk3588/turing-rk1/hardware-reference.md` |
| The Veyron per-board deltas, the in-tree board list, the eMMC device node | `rk3288/veyron/family-boards.md` |
| The Veyron boot chain, the payload budget, the factory eMMC layout | `rk3288/veyron/boot-chain.md` |
| The Veyron initramfs driver chains and the panic timeout | `rk3288/veyron/white-screen.md` |
| The RK3288 VOP and HDMI ceilings | `rk3288/display-reference.md` |

## Publishing constraints

No document carries a hostname, an IP address, a credential, or a login name on a
machine. A public forum handle can cite a community source, and a public repository URL
can cite a public project. A **Sources** entry describes an artifact by what it is and
where it was published, never by a path into private storage.

## Domain vocabulary

**A script reads this section.** The words sit in one italicized comma-separated run.

<!-- prose-lint: off -- a vocabulary list, where completeness is the point. -->

*device-ref, prose-lint, u-boot, coreboot, depthcharge, libreboot, vboot, kpart, cgpt,
futility, mkimage, mkdepthcharge, depthchargectl, depthcharge-tools, crossystem, binman,
rkbin, idbloader, extlinux, initramfs, initramfs-tools, mkinitramfs, mkinitcpio, dracut,
initrd, rootfs, systemd, udev, logind, kconfig, defconfig, dtsi, dts, dtb, dtbo, devicetree,
mainline, vendor, maskrom, rockusb, rkdeveloptool, pyrographer, usbplug, fastboot, emmc,
mmcblk0, sdio, sdmmc, spi, gpio0, gpio1, gpio2, gpio3, gpio4, i2c, uart, uart0, uart2,
uart4, drd0, drd1, dwc2, dwc3, xhci, ehci, ohci, usbdp, combphy, hdptx, vop, vop2, vopb,
vopl, panfrost, panthor, panvk, llvmpipe, mesa, libmali, rkvdec, hantro, verisilicon,
rockchipdrm, analogix_dp, rocket, rknpu, librga, rga2, rga3, ffmpeg, gstreamer, fluster,
v4l2request, v4l2-ctl, brcmfmac, hci_bcm, btsdio, btbcm, bluez, hciattach, btattach,
mwifiex, aic8800, rc-core, ir-keytable, cec-ctl, cec-follower, alsa, amixer, alsactl,
pipewire, cpufreq, cpuidle, s2idle, psci, regulator, pmdomain, genpd, armhf, arm64,
aarch64, armmp, speedy, minnie, mickey, jerry, jaq, mighty, brain, pinky, fievel, tiger,
veyron, forky, trixie, sid, x86_64, iommu, dma-buf, dma_heap, nv12, nv15, p010, framemd5,
zimage, vmlinuz, zstd, xz, lz4, ext4, gpt, postmarketOS, velvet-os, linux-firmware,
linux-libre.*

<!-- prose-lint: on -->

The list is extended by use rather than by decision, and it is never exhaustive.
