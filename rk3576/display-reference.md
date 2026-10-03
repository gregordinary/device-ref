# RK3576 HDMI display path

This document is the register-level reference for the RK3576 HDMI output. The output is a
pipeline of three blocks. They are the VOP2 display controller (`0x27d00000`), the Synopsys
DesignWare HDMI QP transmitter (`0x27da0000`) and a Samsung HDPTX HDMI/eDP combo PHY
(`0x2b000000`). The document covers the bring-up contract, the register-access rules and
the blocks the TRM does not document. It also covers the handover from a bootloader to an
operating system, and the TMDS rate ceiling under mainline Linux.

The bring-up contract is scoped to TMDS at CEA VIC 16 (1080p60, RGB 8bpc), and FRL is out
of scope. The register behavior is measured on the reference unit, an H96 MAX M9, by live
register observation. It is cross-checked against the same board's vendor u-boot with its
display lit. Every load-bearing claim is measured on hardware [measured]. The documents and
code read alongside are listed under [Sources](#sources).

## The bring-up contract

Producing pixels requires every condition below, and each condition is independent.
Violating any one of the eight hardware conditions yields a link that syncs while the
screen stays black: the monitor wakes and reports a mode. Those failures cannot be told
apart from the outside without register access. The two u-boot conditions fail
differently, and their subsections say how.

### Power domain

PD_VO0 must be on, with idle de-asserted and the fabric clocks ungated. The PMU at
`0x27380000` controls it through these registers:

| Register | Offset |
|---|---|
| REQ | `+0x110` |
| ACK | `+0x120` |
| IDLE | `+0x128` |
| CLK_UNGATE | `+0x140` |
| PWR_GATE | `+0x210` |

PD_VO0 is power bit 15 and req/idle bit 11, and its ungate mask is `0x6800`. De-idle and
ungate must run even on a domain that already reads as powered. A prior boot stage can
leave the domain powered but idled.

### PHY PLL lock

The PHY reference clock is `xin24m`, selected by PMU0_GRF OSC_CON6 (`0x26024018`) bit 4
= 0. The PHY's APB clock is PMU-CRU GATE_CON0 bit 1 (`0x27220800`). After the ROPLL is
configured, poll PLL_LOCK_DONE, bit 3 of `GRF_HDPTX_STATUS` (`0x26032080`). PHY_CLK_RDY
(bit 2) is not a lock indicator: it asserts later, in the STANDBY state.

The TRM start-up sequence promises lock in under 60 us [TRM]. The measured lock time is
~1.06 ms [measured]. Use a timeout of at least 10 ms.

### Controller clock gates

The HDMI QP has its own clock manager, the CMU, described under
[The clock manager](#the-clock-manager). `CMU_CONFIG0` (`+0x0a0`) must have VIDQPCLK_OFF
(bit 3) and LINKQPCLK_OFF (bit 5) cleared before the enable path touches any register in
those domains. Otherwise the access raises a synchronous external abort, as
[HDMI QP register access rules](#hdmi-qp-register-access-rules) describes.

`TIMER_BASE_CONFIG0` (`+0x080`) is 396000000 on the RK3576, which is also the silicon reset
value. The RK3588 value, 428571429, is wrong here. Changing the timer base after DDC has
been used requires retiming the I2CM SCL dividers.

### Pixel clock

`dclk_vp0` is muxed onto `clk_hdmiphy_pixel0` by CRU CLKSEL_CON147 (`0x2720054c`) bit 11.
The VOP raster and the TMDS serializer then run from the same PHY PLL.

### VOP and bridge ordering

VP0 is committed and running before the transmitter's video-domain registers are
configured. The video QP register domain decodes only once the VOP pixel interface runs.
When the PHY PLL later delivers dclk, the VP free-starts.

### VP0 output mode

VP0's output mode is AAAA (`0xf`). The VOP-to-HDMI pixel interface is 30 bits wide: one
10-bit lane per color component, with each 8-bit component aligned onto its own lane. P888
(`0x0`) packs components for parallel-RGB panels. Under P888 the HDMI IP samples near-zero
values on every lane, and the output is black. The mainline kernel's RK3576 HDMI glue sets
`ROCKCHIP_OUT_MODE_AAAA` for the same reason.

The vendor firmware also sets the dither-down bits (DSP_CTRL bits 17:16) for 8bpc output.
Dither affects gradients only, not whether video appears.

### Post-display window

The VP0 post-display window must always be programmed. Set POST_DSP_HACT_INFO (`+0xc34`)
and POST_DSP_VACT_INFO (`+0xc38`) to the active window. Set POST_SCL_FACTOR_YRGB (`+0xc3c`)
to unity factors for unscaled output. The window powers up zero-sized. A zero-sized window
emits no pixels at all, background color included, while the raster keeps running.

### AV packetizer datapath

After the PHY lanes are up, open the AV packetizer datapath. Clear
AVP_DATAPATH_VIDEO_SWDISABLE (bit 6) in `GLOBAL_SWDISABLE` (`+0x044`). Then clear AVMUTE
by transmitting a GCP with CLEAR_AVMUTE: set `PKTSCHED_PKT_CONTROL0` to 2, and set
GCP_TX_EN in `PKTSCHED_PKT_EN`.

### Frame buffer cache coherency in u-boot

A u-boot video driver must opt in to the uclass dcache flush with
`video_set_flush_dcache()`. Without it, `video_flush_dcache()` is an unconditional no-op,
with damage tracking or without. CPU-drawn content then stays in the dcache. The panel
shows only the cache lines that happen to evict: a streaky logo, or fragments of text.

### VOP quiesce before an operating system boots

A bootloader that lights the panel must quiesce the VOP before it jumps to an operating
system. Jumping to Linux with the VOP still scanning out leaves the kernel unable to drive
the display at all. This failure shows in the kernel rather than in the bootloader.
[Handing the display over to an operating system](#handing-the-display-over-to-an-operating-system)
gives its signature and the teardown.

## HDMI QP register access rules

The controller's APB decode is strict. A wrong read on a bare-metal system is a
synchronous external abort that resets the CPU. Three rules govern access, and a generated
map of the full mapped register file supports them [REGMAP]. The map lists offset, name and
reset value, extracted from the TRM summary with names joined from the vendor headers.

### Unmapped offsets

Unmapped offsets abort. The register file has many holes, among them `0x010`, `0x11c`,
`0x144`, `0x14c`, `0x804`, `0x810`, `0x868`, `0x8a0`-`0x8dc` and `0x910`. The register
summary in TRM chapter 24.4.2 is authoritative for which offsets exist [TRM]. Every abort
measured corresponds to an offset absent from that table [measured].

`VIDEO_INTERFACE_CONFIG1` (`0x804`, pixel repetition) is documented in the Synopsys
lineage. This silicon configuration does not implement it.

### Write-only registers

Write-only registers abort on read, on fully working systems too. The known members:

| Register | Offset |
|---|---|
| `GLOBAL_SWRESET_REQUEST` | `0x040` |
| `SCDC_CONTROL0` | `0x148` |
| Second TMDS FIFO word | `0x974` |
| `PKTSCHED` control | `0xab0` onward |
| Audio-packet control | `0xa44`, `0xa50` |
| `VIDEO_INTERFACE_CONTROL0` | `0x80c` |
| `AUDIO_INTERFACE_CONTROL0` | `0x82c` |
| HDCP14 block past its first word | `0x904` onward, `0x914` onward |

### Clock-gated domains

Registers in the video and link QP domains abort until the CMU gates are open and the
HDPTX PLL supplies the link clock. The `LINK_*` registers (`0x968`), `TMDS_FIFO_*`
(`0x970`) and `HDCP2LOGIC_*` (`0x8e0`) are in those domains.

### Controller registers

| Register | Offset | Notes |
|---|---|---|
| GLOBAL_SWRESET_REQUEST | 0x040 | Write-only strobes. AVP_DATAPATH_SWINIT_P is bit 6, used only by FRL link training |
| GLOBAL_SWDISABLE | 0x044 | Bit 6 gates the AVP video datapath |
| RESET_MANAGER_STATUS0/1/2 | 0x050-0x058 | Sub-block reset and clock state |
| TIMER_BASE_CONFIG0 | 0x080 | Resets to 0x179a7b00 = 396 MHz, the correct RK3576 value |
| LINK_CONFIG0 | 0x968 | Resets to 0x10: the DVI-mode bit is set out of reset |
| PKTSCHED_PKT_EN | 0xaa8 | AVI bit 13, VSI bit 12, GCP bit 3, NULL bit 0 |
| Packet content RAM | 0xb00-0xcbc | Contiguous. PKT_AVI_CONTENTS0 is at 0xbe0 |
| HDCP2LOGIC_CONFIG0 | 0x8e0 | Bit 0, HDCP2_BYPASS, resets to 0, which routes the datapath through the external HDCP2 ESM. Bring-up sets the bypass |
| HDCP14_CONFIG0 | 0x900 | Resets to 0x00010000 (OESS CTL3 pulse size 2) |

### The clock manager

The controller's clock-manager block at `0x0a0`-`0x0c4` is absent from the TRM's register
summary. It exists and is readable:

| Register | Offset | Notes |
|---|---|---|
| CMU_CONFIG0 | 0x0a0 | Bit 3 VIDQPCLK_OFF, bit 5 LINKQPCLK_OFF. Bit 0 reads 1 in normal operation |
| CMU_CONFIG1-3 | 0x0a4-0x0ac | Read 0x00010001, 0x00010001 and 0x00000001 in normal operation |
| CMU_STATUS | 0x0b0 | Bits 5:0 are DISPLAY_CLK_MONITOR, and 0x15 means locked. A settled, lit system reads 0x255. A sample taken mid-enable reads 0x200 or 0x210 before settling |
| CMU_*_CLK_FREQ | 0x0b4-0x0c4 | Live frequency meters in kHz for IPI, video QP, link QP, audio QP and eARC bpclk. A 148.5 MHz TMDS mode meters 37125 on the first three |

### The video monitor

Offsets `0x880`-`0x89c` measure the incoming video stream live. They are the fastest way to
verify the VOP-to-transmitter interface without touching the VOP:

| Offset | High 16 bits | Low 16 bits |
|---|---|---|
| 0x884 | hsync width | hfront porch |
| 0x888 | total hblank | hback porch |
| 0x88c | htotal | hactive |
| 0x890 | vsync width | vfront porch |
| 0x894 | total vblank | vback porch |
| 0x898 | vtotal | vactive |

A 1080p60 source reads `0x0898` and `0x0780` at `0x88c` (htotal 2200, hactive 1920). It
reads `0x0465` and `0x0438` at `0x898` (vtotal 1125, vactive 1080).

### The color-depth override field

Bits 23:20 of `VIDEO_INTERFACE_CONFIG0` (`0x800`) are `HDMI_COLOR_DEPTH_OVR_VALUE`. The
field is an override value whose enable the TRM does not document anywhere. Rockchip
firmware uses it as a scratchpad and parks reserved color-depth codes there as markers:

- `0x2 << 20` means the bootloader lit the display. The vendor kernel reads it to skip
  re-initialization.
- `0x1 << 20` marks yuv422 10-bit.

The values are inert. A working system runs with either 0 or a marker present, and the
register plays no part in making video appear.

## VOP2 registers

- The VOP2 register file (`0x27d00000`-`0x27d019ff`) has no APB holes and can be dumped
  end to end.
- Writes latch on commit, through REG_CFG_DONE at the base. Writing `0x18001` commits VP0.
- The RK3576 pre-scan delay is computed rather than taken from the RK3588's static table.
  It is the background delay plus half the active width, and VP0 processes 2 pixels per
  clock.
- For 1:1 Esmart region output, the scale-factor registers carry 0 (SCALE_NONE). When the
  output is scaled, the factors are derived from the source and destination sizes.
- Across a warm reset, the GRFs retain state. The CRU and SECURECRU gate registers reset
  to all zeros, with every gate open, so display bring-up needs no explicit CRU ungating.

The VP0 block's registers:

| Register | Offset | Fields |
|---|---|---|
| DSP_CTRL | `+0xc00` | out_mode bits 3:0, dither-down bits 17:16, standby bit 31 |
| DCLK_SEL | `+0xc0c` | |
| DSP_BG | `+0xc2c` | |
| PRE_SCAN_HTIMING | `+0xc30` | |
| Post-display window | `+0xc34`-`+0xc40` | POST_SCL_CTRL (`+0xc40`) resets to 0x10 |
| Display timing | `+0xc48`-`+0xc54` | |

### VO0 GRF display fields

The VO0 GRF at `0x2601a000` carries these display fields:

| Register | Offset | Function |
|---|---|---|
| SOC_CON1 | 0x04 | Bit 0: HDMITX FRL mode (0 = TMDS) |
| SOC_CON8 | 0x20 | HDMITX color format and depth |
| SOC_CON9 | 0x24 | Bit 9: EBC as HDMITX data source |
| SOC_CON12 | 0x30 | The "osda" (DDC SDA) delay line. Vendor firmware tunes it to 0x9001 for EDID reads |
| SOC_CON13 | 0x34 | Bit 4: hdmi_1to4 for the EBC path. Bit 10: HDCP0 ipiclk gate |
| SOC_CON14 | 0x38 | SCLIN, SDAIN, HDMITX grant and I2S routing (0x71 routes all to the controller) |

HPD control and status sit in the VCCIO6 IOC domain. MISC_CON0 at `0x2604a400` carries the
low-level filter time, the filter enable (bits 13:8) and the HPD source select (bits
15:14). The source select's reset default is the glitch-free port. HDMITX_HPD_STATUS is at
`0x2604a440`. The HPD, CEC and DDC signals reach these pins through discrete circuits, as
[HDMI sideband circuits](hardware-design-guide.md#hdmi-sideband-circuits) describes.
Mainline routes them to the controller through the `&hdmi` pin groups that `rk3576.dtsi`
sets, which a board-level `pinctrl-0` replaces (see
[A board-level pinctrl-0 replaces the inherited groups](corrections-and-gotchas.md#a-board-level-pinctrl-0-replaces-the-inherited-groups)).

## Register access on a live system

- A board's vendor u-boot is the working-state ground truth. On the reference unit it is
  reachable without writing anything: power-cycle without the maskrom strap, and press
  Ctrl+C to stop the boot. The display stays lit at the prompt, so register reads there
  show the lit configuration.
- A faulting read at the vendor u-boot prompt costs little, because the board resets back
  into the same vendor u-boot. On a u-boot loaded into RAM from maskrom, the same fault
  costs a fresh USB download. Probe unknown offsets on the vendor side first.
- `md.l` output of large sweeps at 1.5 Mbaud overruns typical terminal scrollback. Capture
  in chunks of at most ~50 lines, or log the port.
- `0x27da0000` reads ASCII "XTQH0100..." (CORE_ID and VER, matching the TRM reset values).
  The read is a quick check that the APB path and the power domain are alive.

## Handing the display over to an operating system

A bootloader that drives the panel must quiesce the VOP before it jumps to the operating
system. A mainline Linux kernel cannot recover a VOP left scanning out. The failure does
not look like a display bug from the outside. The kernel boots and the serial console is
healthy, while the screen is black or shows a mode the monitor rejects as out of range.

### The failure signature

The VOP's IOMMU (`0x27d07e00`, sharing the VOP's interrupt) cannot be stalled or
force-reset while a window is still fetching. The `rk_iommu` probe reports the failure
directly:

```
rk_iommu 27d07e00.iommu: Enable stall request timed out, status: 0x000011
rk_iommu 27d07e00.iommu: Enable stall request timed out, status: 0x00001d
rk_iommu 27d07e00.iommu: FORCE_RESET command timed out
```

The status words decode against `RK_MMU_STATUS_*` in `drivers/iommu/rockchip-iommu.c`:

- `0x11` is PAGING_ENABLED and REPLAY_BUFFER_EMPTY with IDLE clear. That bank has a
  transaction in flight.
- `0x1d` adds STALL_ACTIVE and IDLE. That bank stalled normally.

One live bank is enough to fail the whole stall. PAGING_ENABLED being set at all is the
second half of the failure. It means the bootloader enabled the VOP's own MMU
(`SYS_MMU_CTRL` `+0x020`, RKMMU_V2_EN bit 0). The bootloader then left that MMU translating
against page tables in memory the OS is about to reuse.

From there the damage is total but silent. The VOP raises no interrupt, and its entry in
`/proc/interrupts` stays at zero. Every atomic commit the kernel makes waits out its
vblank timeout:

```
[drm] *ERROR* flip_done timed out
[drm] *ERROR* [CRTC:71:video_port0] commit wait timed out
[drm] *ERROR* [CONNECTOR:73:HDMI-A-1] commit wait timed out
[drm] vblank wait timed out on crtc 0
```

EDID fails with it. The connector reports `connected` while its `modes` list stays empty.
The kernel then falls back to a default timing, which the monitor can refuse as out of
range. This is collateral damage rather than a separate DDC fault, and it clears as soon as
the VOP does.

### The teardown

Disable the window first, then park the video port, commit, wait a frame, and only then
turn the MMU off:

1. Clear WIN_EN (bit 0) in `ESMART0_REGION0_CTRL` (`+0x1810`) so the VOP stops fetching.
2. Set STANDBY_EN (bit 31) in `VP0_DSP_CTRL` (`+0xc00`).
3. Write `REG_CFG_DONE` (`+0x000`) to latch both.
4. Wait one frame (17 ms at 60 Hz) for standby to take effect.
5. Clear RKMMU_V2_EN in `SYS_MMU_CTRL` (`+0x020`).

The order is load-bearing in both directions. The MMU must go off after the fetch has
drained. Turned off earlier, it faults the in-flight transaction it was meant to release.

In u-boot the teardown belongs in the video driver's `.remove`, with `DM_FLAG_OS_PREPARE`
set on the driver. `announce_and_cleanup()` then runs it through
`dm_remove_devices_active()` on the way into `bootm`. The in-tree `rk3288_vop.c` and
`rk3328_vop.c` drivers do exactly this. The rk3328 driver does it under a comment naming
the consequence: "to avoid iommu page-fault in Linux."

### The HDPTX PHY

The HDPTX PHY needs no teardown. When Linux finds the PLL already locked, it logs this
line:

```
rockchip-hdptx-phy 2b000000.hdmiphy: PLL locked by unknown consumer!
```

`rk_hdptx_phy_consumer_get()` only warns, then reconfigures the PLL through
`rk_hdptx_pll_cmn_config()` regardless. A pre-locked PLL is handled rather than fatal, so
no bootloader-side PHY teardown is required.

## The TMDS rate ceiling

Mainline Linux caps the HDMI TMDS character rate at 340 MHz [measured, 2026-08-08]. The
cap is the DW HDMI QP bridge's: `dw_hdmi_qp_bridge_tmds_char_rate_valid()` rejects a faster
mode before it reaches userspace. A 3840×2160 mode at 60 Hz is therefore never offered,
even on a sink that advertises it.

The reference unit's mode list shows the cap. The list was read
with `xrandr` under the Cinnamon desktop, on a 600×340 mm sink whose EDID advertises 4K60.
The EDID carries VIC 97 and an HDMI Forum VSDB with `Max TMDS Character Rate: 600 MHz`.
`HDMI-1` offers these modes:

| Resolution | Refresh rates offered (Hz) |
|---|---|
| 3840×2160 | 30 (preferred and current), 29.97, 25, 24, 23.98 |
| 2560×1440 | 59.95 |
| 1920×1080 | 60, 59.94, 30, 29.97, 24, 23.98 |
| Lower | The usual ladder down to 640×480 |

1440p60 at 241 MHz and 4K30 at 297 MHz both fit under 340 MHz and are offered. No 4K60
entry appears. That is the shape a 340 MHz ceiling predicts.

## Sources

- [TRM]: Rockchip RK3576 TRM V1.2 Part 2, chapter 24 (HDMI TX Controller) and chapter 25
  (HDMI TX/eDP Combo PHY).
- [REGMAP]: an offset, name and reset-value map of the HDMI QP register file. It is
  generated from the TRM's register summary, with names joined from the vendor headers.
- The reference unit: an H96 MAX M9, observed live at the register level. Readings are
  cross-checked against the same board's vendor u-boot with its display lit.
- Mainline Linux drivers: `dw-hdmi-qp`, `rockchip_drm_vop2`, `dw_hdmi_qp-rockchip`,
  `drivers/iommu/rockchip-iommu.c` and the HDPTX PHY driver.
- Mainline u-boot: the video uclass, `rk3288_vop.c` and `rk3328_vop.c`.
- Rockchip's u-boot `next-dev` display stack.
- The `gahingwoo/edk2-rk3576` UEFI port.
