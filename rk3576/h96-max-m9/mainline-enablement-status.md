# H96 MAX M9 mainline enablement status

This page states what Linux drives on the H96 MAX M9, one row per subsystem, on the vendor
kernel and on mainline. Each mainline result carries the kernel it was measured on and its
evidence grade. The mechanism behind a result lives in the document its row links to. The
[hardware reference](hardware-reference.md) states what the board is, and the RK3576
references state what the SoC blocks do.

## Kernel lines

The kernel line decides most results. The vendor kernel is Rockchip's 6.1 tree, where video
decode, video encode and the NPU work and the GPU does not [consensus]. Mainline drives the
GPU, and it decodes H.264 and HEVC in hardware from Linux 7.0. The Armbian thread's
mainline builds are 6.19 edge kernels. The measurements on this page span the 7.1 and 7.2
stable lines.

A result on one line implies nothing about the other, so each row states both. A
measurement holds for the kernel its tag names. A later kernel on the same stable line does
not inherit it until it is measured again.

The Armbian thread confirms that mainline carries no `rk3576-h96*` device tree. Every
mainline result here runs on a board device tree written against the in-tree
`rk3576.dtsi`. [The factory device tree](factory-device-tree.md#board-deltas-against-the-evb1-tree)
records the board's deltas against Rockchip's EVB1 tree.

## Evidence grades

`[consensus]` means more than one independent poster in the community threads under
[Sources](#sources) reports the result, and `[reported]` means one poster does. `[measured]`
results come from the reference unit. A tag with a date and no kernel marks a measurement
whose record names no kernel. Vendor cells carry a grade where the record gives one.

## Subsystem status

| Subsystem | Vendor kernel (6.1) | Mainline | Mainline evidence | Reference |
|---|---|---|---|---|
| Boot to console (eMMC, USB) | Works [consensus] | Works | [measured, 2026-08-31, 7.2.0] from eMMC, [consensus] | [Boot](hardware-reference.md#boot) |
| Serial console (1.5 Mbaud) | Works [measured] | Works | [measured] | [Serial debug console](hardware-reference.md#serial-debug-console) |
| Ethernet (GMAC0) | Works [consensus] | Works | [measured], [consensus] | [Networking](hardware-reference.md#networking) |
| eMMC (HS400 Enhanced Strobe) | Works | Works | [measured, 2026-10-02, 7.2.8], 308 MB/s raw read | [Memory and storage](hardware-reference.md#memory-and-storage) |
| HDMI video output | Works at 1080p60 and 4K [consensus] | Works up to 4K30, no 4K60 | [measured, 2026-08-08] | [The TMDS rate ceiling](../display-reference.md#the-tmds-rate-ceiling) |
| HDMI hotplug and EDID re-read | Broken [consensus] | Works, with the `hdmi_txm0` pin-mux fix | [measured, 2026-08-08, 7.1.6] | [H96 corrections](corrections-and-gotchas.md) |
| GPU and OpenGL ES (Mali-G52, Panfrost) | Blacklisted [consensus] | Works, with the GPU power-domain supply fix | [measured, 2026-10-02, 7.2.8], GBM and surfaceless | [GPU](../video-reference.md#gpu) |
| GPU Vulkan (PanVk) | Not recorded | No usable driver | [measured, 7.2.3], Mesa 26.1.6 and 26.2.2 | [GPU](../video-reference.md#gpu) |
| USB 2.0 host | Works | Works | [measured, 7.2.0], no error or disconnect over a 2 d 22 h soak, [consensus] | [xHC errors on resume](../corrections-and-gotchas.md#xhc-errors-on-resume) |
| USB 3.0 SuperSpeed | Works [reported] | Works on drd0 only | [measured, 2026-08-08, 7.1.3], 5 Gbps | [USB](hardware-reference.md#usb) |
| Wi-Fi (2.4 and 5 GHz) | Works | Works, with the out-of-tree AIC8800 driver | [measured], AIC8800D80 | [Wi-Fi](wifi-bluetooth.md#wi-fi) |
| Bluetooth | Works | Works, as a UART HCI | [measured, 2026-08-07], A2DP playback | [Bluetooth](wifi-bluetooth.md#bluetooth) |
| Suspend and resume (s2idle) | Not recorded | Works, with the AIC8800 driver quiesced across suspend | [measured, 2026-07-30, 7.1.5] | [Suspend and resume](wifi-bluetooth.md#suspend-and-resume) |
| Video decode (H.264, HEVC) | Works [consensus] | Works, 8-bit and 10-bit | [measured, 2026-10-02, 7.2.8], 6 of 6 clips bit-exact | [The decoder](../video-reference.md#the-decoder) |
| 10-bit frames out of the decoder | Works, through MPP | Works, `NV15` unpacked on the CPU | [measured, 2026-09-13, 7.2.3] | [10-bit output](../video-reference.md#10-bit-output) |
| Video encode | Works [consensus] | Absent | [source], no merged driver | [Encode](../video-reference.md#encode) |
| RGA (2D raster) | Works | Works on both RGA2 cores, with the out-of-tree vendor driver | [measured, 2026-10-02, 7.2.8] | [RGA](../video-reference.md#rga) |
| NPU | Works (RKNPU2) | Core 0 works through `rocket` with the posted RK3576 series, no released userspace | [measured, 2026-10-02, 7.2.8] | [The NPU driver](../npu-reference.md#the-driver) |
| DDR bandwidth monitor (DFI) | Not recorded | Absent | [source, 7.2.8] | [DDR bandwidth monitor](#ddr-bandwidth-monitor) |
| Analog audio | Works (HT5010) | Works, with `rockchip,sai-tx-route` and the SAI routing fix | [measured, 2026-08-26, 7.2] | [SAI transmit routing](../corrections-and-gotchas.md#sai-transmit-routing) |
| S/PDIF | Works | Works | [measured, 2026-07-30], optical output | [Display and audio](hardware-reference.md#display-and-audio) |
| HDMI audio | Works | Works | [measured, 2026-07-30] | [Display and audio](hardware-reference.md#display-and-audio) |
| Bundled remote (2.4 GHz RF) | Works [measured] | Works | [measured] | [Peripherals and controls](hardware-reference.md#peripherals-and-controls) |
| IR receiver | Works, with configuration [consensus] | Works | [measured, 2026-07-30] | [H96 corrections](corrections-and-gotchas.md) |
| HDMI-CEC | Not recorded | Box drives TV. TV drives box untested end to end | [measured, 2026-07-30] | [H96 corrections](corrections-and-gotchas.md) |
| SD card | Absent, not fitted [consensus] | Absent | [consensus] | [SD card](#sd-card) |

## Display

Mainline u-boot and mainline Linux drive a console and login prompt at 1080p60 and 4K30
[measured]. The vendor kernel drives 1080p60 and 4K [consensus]. The mainline mode list
stops at 4K30, even on a sink whose EDID advertises 4K60. The DW-HDMI-QP bridge rejects
every mode above 340 MHz TMDS, as [the TMDS rate ceiling](../display-reference.md#the-tmds-rate-ceiling)
shows.

Mainline detects hotplug and re-reads EDID [measured, 2026-08-08, 7.1.6]. Both need the HPD
and DDC pins muxed, and a board-level `pinctrl-0` on `&hdmi` replaces the groups
`rk3576.dtsi` gives the controller. [The inherited-groups rule](../corrections-and-gotchas.md#a-board-level-pinctrl-0-replaces-the-inherited-groups)
has the mechanism. On the vendor kernel the panel must be connected at boot, and a hotplug
logs `i2c read err!` without recovering [consensus].

The mainline display path has hard preconditions, listed in
[the bring-up contract](../display-reference.md#the-bring-up-contract). A bootloader that
lights the panel must also quiesce the VOP before it boots Linux, as
[the display handover](../display-reference.md#handing-the-display-over-to-an-operating-system)
describes.

## GPU

State the kernel line with any GPU result, because the result follows it. On the vendor
kernel Panfrost is blacklisted or non-functional. Through 2025, several posters blacklisted
the module to boot at all [consensus]. On mainline the Mali-G52 binds and renders. One
poster runs a Panfrost desktop on a 6.19 edge kernel [reported]. Mesa's panfrost driver
renders on 7.2.8 over GBM and surfaceless contexts [measured, 2026-10-02].

Mainline Panfrost needs the GPU power domain bound to its supply rail. Without that binding
a late Panfrost probe intermittently panics the boot with an asynchronous SError, as
[the GPU power domain needs its supply](../corrections-and-gotchas.md#the-gpu-power-domain-needs-its-supply)
explains.

No Vulkan driver is usable on this GPU [measured, 7.2.3]. PanVk declines the Bifrost G52
unless forced. Forced, it computes wrong chroma in every ffmpeg Vulkan filter tested, on
Mesa 26.1.6 and 26.2.2. With `mesa-vulkan-drivers` installed, the only Vulkan device is
llvmpipe, which is slower than swscale.

## USB

On the reference unit's wiring only drd0, the USB 3.0 port, reaches SuperSpeed [measured,
2026-08-08, 7.1.3]. The drd1 controller's SuperSpeed lane reaches no connector, and its
USB 2.0 port sits behind an internal hub. A drive that enumerates at 480 Mb/s on drd0 is
usually not fully seated, as [the H96 corrections](corrections-and-gotchas.md) describe.
The vendor kernel and a 6.19 edge mainline build are reported to reach USB 3.0 [reported].

USB 2.0 host stays clean across s2idle resume once both dwc3 controllers carry
`snps,dis_u2_susphy_quirk` [measured]. Without it the xHC logs an error on every resume, as
[xHC errors on resume](../corrections-and-gotchas.md#xhc-errors-on-resume) explains.

## Wi-Fi and Bluetooth

The wireless part varies by production batch [consensus]. Earlier units carry a Broadcom
AP6275P, and later units, the reference unit among them, carry an AICSemi AIC8800D80.
Identify the part by its SDIO vendor ID, never by the device-tree label. The mainline
results on this page are for the AIC8800D80. Its driver stays out of tree, because
AICSemi's mainline submission was rejected. [The H96 Wi-Fi and Bluetooth reference](wifi-bluetooth.md)
covers its bring-up.

## Suspend and idle

System suspend works with the sleep state `freeze` (s2idle). The `mem` state selects
`deep` by default, and the board never resumes from `deep` [measured]. The wake source is
the remote's POWER key through gpio0 RK_PD3. The RF receiver leaves the remote-wakeup bit
clear in its configuration descriptor, so it cannot wake the board over USB. The board has
no RTC. After `poweroff` the board returns only once its supply is cycled, because the box
has no physical power button.

Mainline s2idle also needs the out-of-tree AIC8800 driver quiesced across suspend. Without
that, the driver touches SDIO after the host suspends, and the resume deadlocks [measured].
[Suspend and resume](wifi-bluetooth.md#suspend-and-resume) has the mechanism.

Keep deep CPU idle off with `cpuidle.off=1` under rkbin BL31 v1.24, where a CPU in the
`cpu-sleep` state can miss its wakeup [measured]. A boot without the flag ran 24 minutes
clean on 7.2.0 [measured, 2026-08-31]. Against an intermittent fault that is an
unexplained negative, not a fix. A suspend cycle with deep idle enabled is untested.
[Deep idle under BL31 v1.24](../boot-firmware.md#deep-idle-under-bl31-v124) has the record.

## Video

Mainline decodes H.264 and HEVC through the VDPU383 driver from Linux 7.0. A poster on a
6.19 edge kernel reports no hardware acceleration [reported], the expected state for a
kernel before that merge. On the vendor kernel, hardware decode drives Kodi at 4K and
Jellyfin [consensus].

The 7.2.8 result covers H.264 at 1080p and 4K plus High 10 at 1080p. It covers HEVC at
1080p and 4K plus Main 10 at 4K. Each clip is bit-exact over 120 frames [measured,
2026-10-02, 7.2.8].

Without a warm-up work-around, hardware H.264 decode is wrong on roughly one power-up in
three [measured]. The measured kernel carries the work-around described in
[the H.264 power-up erratum](../h264-power-up-erratum.md), and HEVC is unaffected.

Debian forky's ffmpeg 8.1 cannot drive the decoder, because upstream FFmpeg 8.1 has no
`v4l2_request` hwaccel [source]. GStreamer's `v4l2codecs` plugin reaches it. So does an
ffmpeg built from Kwiboo's `v4l2-request-n8.1` branch with `--enable-v4l2-request`.
[Userspace](../video-reference.md#userspace) has the details.

HEVC Main 10 decodes in hardware to `NV15`. VOP2 scans `NV15` out directly, and every other
consumer unpacks it on the CPU, because no block on this SoC converts it in hardware. The
CPU unpack needs an ffmpeg that knows the NV15 and NV20 pixel formats and tags them on the
v4l2request hwcontext. Without one, every consumer fails with `-22` and no software
fallback. [10-bit output](../video-reference.md#10-bit-output) has the consumer table.

Mainline has no encode driver for this SoC, so encode runs on the CPU cores [source].
Jiaxing Hu's RFC VEPU510 H.264 encoder (2026-07-22) is a V4L2 stateful driver. Its intra
frames work on a Radxa ROCK 4D, and its P-frames stall the encoder watchdog. Nothing is
merged.

Both RGA2 cores work through the out-of-tree vendor driver, with buffers imported as
DMA-BUF. Passing virtual addresses faults the IOMMU [measured]. The SoC has no RGA3, so
RGA writes no 10-bit surface. FFmpeg cannot reach RGA on this SoC, because its RGA filters
need the vendor MPP kernel framework.

## NPU

Mainline drives the NPU through the in-tree `rocket` driver, with Jiaxing Hu's
linux-rockchip RFC v2 series and further fixes applied [measured]. The driver matches
`rockchip,rk3576-rknn-core` once that series is applied and `rk3576.dtsi` gains the NPU
nodes. [The NPU driver](../npu-reference.md#the-driver) describes both. Mainline delivers
a device and not a stack. `rocket` has no matmul path, so it does not serve LLM inference,
and no released userspace carries an RK3576 encoder.

The board device tree enables core 0 only. With both cores bound, two jobs in flight at
once compute wrong answers [measured]. Those runs clocked the NPU at 786 MHz, and a
two-core run at the 594 MHz operating point is untested. [The second core](../npu-reference.md#the-second-core)
and [the operating point](../npu-reference.md#operating-point) have the measurements.

The 7.2.8 result runs with `vdd_npu_s0` as the NPU power domain's `domain-supply`. `rocket`
claims no regulator, so the rail needs that consumer or `regulator-always-on`, as
[the NPU rail](../npu-reference.md#the-npu-rail) explains.

## DDR bandwidth monitor

Mainline has no DFI support for this SoC [source, 7.2.8]. `rk3576.dtsi` carries no `dfi`
node, and `rockchip-dfi` matches only the RK3399, RK3568 and RK3588. The RK3576 DDRMON is
version 0x40, with a control-register layout the driver does not know. Support is posted
upstream as a v1 series (2026-09-17) and is unmerged.

## Audio

Mainline analog output works on 7.2 [measured, 2026-08-26]. It needs a stream routed to
`sdo2` and the MCLK output gate enabled. It also needs a change to the SAI driver that
keeps `SAI_PATH_SEL` programmed. [The H96 audio section](hardware-reference.md#display-and-audio) gives the
board values. [SAI transmit routing](../corrections-and-gotchas.md#sai-transmit-routing)
gives the mechanism.

An SSH session is not seated, so the logind `uaccess` ACL on `/dev/snd/*` never applies to
it. The device nodes are `root:audio 0660`, so an account that plays audio over SSH needs
the `audio` group. A console install also needs a sound server for headless audio. The
2026-07-30 measurement ran under a desktop session, which supplies both a seat and
PipeWire.

## Remote control and CEC

The bundled remote is a 2.4 GHz RF handset on an internal USB HID receiver. It works on
both kernels with no configuration [measured], and it cannot drive the IR receiver.

On the vendor kernel an IR remote works with configuration, and posters have published the
usercode and keymap tables [consensus]. Mainline receives IR through `gpio-ir-receiver` and
the rc-core NEC decoder, since the vendor's PWM-capture `remotectl` driver has no mainline
counterpart. Reception works with a borrowed H96 Max H313 handset [measured, 2026-07-30].
None of the six
recovered vendor keymaps matches that handset, and none is hardware-verified.

Mainline drives CEC through the DW-HDMI-QP controller with `CONFIG_DRM_DW_HDMI_QP_CEC`.
The CEC pin is in `hdmi_txm0_pins`, so CEC needs the same pin-mux fix as hotplug. The box
drives an LG TV through standby, wake and Active Source [measured, 2026-07-30]. The TV
forwards its remote keys to the box [measured]. Turning them into input events needs
`CONFIG_MEDIA_CEC_RC` with `CONFIG_RC_CORE=y`, and that end-to-end key path is untested.

## SD card

The SD slot is depopulated on every unit examined: the footprint and pin-mux are present,
and the power-path parts are not [consensus]. Restoring it takes a solder mod and SD
regulator nodes in the device tree. The mod is an LPW5202ADB5F load switch plus passives,
or a 0-ohm jumper for always-on power. The regulator nodes describe the SD 3.3 V and 1.8 V
supplies.

## Flashing and recovery

The community's flashing practice, with the reference unit's measurements where they
differ:

- Writing a full image over USB works through the vendor `RK3576_MiniLoaderAll.bin`, with
  `rkdeveloptool db` and then `wl` or `write`, or through Windows `RKDevTool` [consensus].
- No permanent brick is confirmed, and recovery is routine [consensus]. A common trap is
  DTB corruption from an unfrozen `apt` kernel upgrade, so freeze kernel updates on
  Armbian [consensus].
- Reading a full image over USB does not work, because the rockusb read path truncates at
  32 MiB. Read through the block layer instead, with `ums` and `dd` or a `dd` from a booted
  system. See [the 32 MiB read wall](../rockusb-reference.md#the-32-mib-read-wall).
- Holding the AV-jack button before connecting USB reaches Loader mode on the thread's
  firmware builds [consensus]. On the reference unit's newer factory firmware it boots
  Android recovery, a firmware-version difference the [H96 corrections](corrections-and-gotchas.md)
  record.
- With mainline u-boot carrying both download-key fixes, the same button enters maskrom
  [measured]. See [the SARADC download key](../boot-firmware.md#the-saradc-download-key).
- Shorting the eMMC test pads at power-on enters maskrom under any firmware, as
  [Boot](hardware-reference.md#boot) describes.
- Any u-boot with `ums` exposes the eMMC as a USB block device with `ums 0 mmc 0`. The host
  then writes the whole image with one `dd`, and no vendor loader is needed [measured].

## Sources

- The Armbian forum H96 MAX M9 thread (topic 40483), five pages from 2024-08 to 2026-05.
  The `[consensus]` and `[reported]` grades rest on it.
- The parallel LibreELEC thread, cross-referenced with the Armbian thread.
- Measurements on the reference unit, which back every `[measured]` tag.
