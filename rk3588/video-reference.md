# RK3588 video, 2D and GPU reference

This page covers the RK3588's media blocks under a mainline kernel: decode, encode, RGA,
the GPU, the HDMI receiver and the DDR traffic counters. It also states what a transcode
costs on them. It states which of them mainline
userspace can reach, and the traps an integrator meets on the way. The scope is the SoC.
Everything here follows from `rk3588.dtsi` and the shared drivers rather than from one
board.

Measurements come from the reference unit, a Turing RK1. A `[measured]` tag carries the
date and the kernel where they are recorded. A result on one kernel is not automatically
a result on the next. A
`[source]` tag marks a fact read from the 7.2.y kernel tree, or from the ffmpeg source
file the tag names.

## The V4L2 node inventory

The SoC presents four V4L2 codec nodes, one per job [measured, 2026-08-09, 7.1.1]. The
`VP9F` format on `/dev/video0` comes from the VP9-on-VDPU381 series [measured,
2026-08-30, 7.2.2].

| Node | Driver | Card | OUTPUT | CAPTURE |
| --- | --- | --- | --- | --- |
| `/dev/video0` | `rkvdec` | `rkvdec` | `S265`, `S264`, `VP9F` (stateless slice data) | `NV12`, and `NV15` for 10-bit |
| `/dev/video1` | `hantro-vpu` | `rockchip,rk3568-vpu-dec` | `S264`, `MG2S`, `VP8F` | `NV12` |
| `/dev/video2` | `hantro-vpu` | `rockchip,rk3588-vepu121-enc` | `YM12`, `NM12`, `YUYV`, `UYVY` | `JPEG` |
| `/dev/video3` | `hantro-vpu` | `rockchip,rk3588-av1-vpu-dec` | `AV1F` | `VT12` (4×4 tiled), `NV12`, and for 10-bit a 4×4 tiled `NV15`, `NV15` and `P010` |

The 10-bit capture formats on `/dev/video3` come from the driver's format lists [source,
7.2.9]. The `v4l2request` hwaccel negotiates `NV15` there, and `hwdownload,format=p010le`
on an AV1 frame is refused, so `P010` goes unused [measured].

Node numbers follow probe order. A kernel that also binds the in-tree RGA driver adds V4L2
nodes for the RGA, and the codec nodes can then take other numbers (see [RGA](#rga)).

Three facts about this inventory are easy to overlook, and each changes a conclusion:

- `/dev/video3` is a working mainline AV1 decoder. `rockchip,rk3588-av1-vpu` binds
  `hantro-vpu` on the `rk3588_vpu981_variant` [source]. See [AV1 decode](#av1-decode)
  for what is bit-exact.
- `/dev/video2` is a mainline hardware encoder, and it encodes JPEG only.
  `rockchip,rk3588-vepu121` binds `hantro-vpu` on the `rk3568_vepu_variant` [source]. It
  is the VEPU121 that Armbian threads discuss, and it has no role in H.264 or HEVC
  transcode.
- `/dev/video1` is a second decoder, VDPU2. It covers H.264, MPEG-2 and VP8 through the
  older Hantro path.

The SoC has four VEPU121 JPEG encoder cores, at `fdba0000` to `fdbac000`, and the mainline
dtsi describes all four. The driver registers the first and logs `missing multi-core
support, ignoring this instance` for the rest [measured, 2026-08-31, 7.2.2].

### Kernel configuration

arm64's defconfig builds some of these drivers and leaves others out [source, 7.2.9]:

| Symbol | Driver | In the defconfig |
| --- | --- | --- |
| `CONFIG_VIDEO_HANTRO` | `hantro-vpu`: VDPU2, the JPEG encoder and the AV1 decoder | Module |
| `CONFIG_VSI_IOMMU` | The AV1 decoder's IOMMU | Not set, and the symbol has no default |
| `CONFIG_VIDEO_ROCKCHIP_VDEC` | `rkvdec` | Not set |
| `CONFIG_ROCKCHIP_IOMMU` | The other media blocks' IOMMUs | Built in |
| `CONFIG_VIDEO_ROCKCHIP_RGA` | `rockchip-rga` | Module |
| `CONFIG_DRM_PANTHOR` | The GPU | Module |
| `CONFIG_DRM_ACCEL_ROCKET` | The NPU | Not set |

The AV1 decoder sits behind its own IOMMU, `iommu@fdca0000`
(`rockchip,rk3588-av1-iommu`, `verisilicon,iommu-1.2`), which `vsi-iommu` drives
[source, 7.2.9]. Without `CONFIG_VSI_IOMMU` the AV1 decoder fails at probe [measured]. A
kernel built from the defconfig alone therefore decodes on VDPU2 only [inferred].

### The rkvdec cores

`rkvdec` runs two cores behind one node. With Collabora's unmerged rkvdec multicore series
applied, `fdc38000` and `fdc40000` both register and bind to one master. That master
presents a single `/dev/video0`, so two V4L2 decoder nodes mean the component binding
failed. With both cores bound, four concurrent 4K H.264 streams decode at realtime, and
tiled HEVC conformance stays bit-exact [measured].

The series schedules one job per m2m context, so each core decodes a different stream
[source]. A single stream never uses both cores. Aggregate throughput doubles at the second
stream and stays flat after it. Decode only, with frames kept as DRM PRIME
[measured, 2026-08-13, 7.1.7, and the 1080p row 2026-10-07, 7.2.9]:

| Stream | One stream | Aggregate ceiling | Concurrent at realtime |
| --- | ---: | --- | --- |
| 1080p H.264, 10 Mb/s | 9.21x | 18.2x | 6 tested, at 3.03x each |
| 4K H.264 | 2.51x | 4.9x, from three streams | 4, and a fifth runs at 0.98x |
| 4K HEVC 10-bit | 2.95x, on 7.2.9 | 5.5x, at two streams | 2 (see [the RCB SRAM IOVA collision](#the-rcb-sram-iova-collision)) |

Two streams of 4K H.264 still run at 2.40x each, against 2.51x for one. The second stream
therefore takes the second core rather than half of the first. Figures that repeat need the
CPU governor at `performance`. Under `ondemand` the cores sat at 1.2 GHz [measured].

VDPU2 is a single core. It decodes one 1080p H.264 stream at 3.15x and holds that total,
about 94 frames per second, however many streams share it [measured, 2026-10-07, 7.2.9].
Both decoders cost 0.4-0.8 CPU-s per 30 s stream. Which node a stream lands on therefore
sets the 1080p H.264 ceiling, as [attributing a failure](#attributing-a-failure) describes.

The tree assigns each rkvdec core 800 MHz for AXI, 600 MHz for its core and CABAC clocks,
and 1 GHz for its HEVC CABAC clock [source]. The CRU delivers 786.4 MHz for AXI, from the
audio PLL, 594 MHz for the core and CABAC clocks, and 1 GHz [measured, 2026-10-23,
7.2.9][CLK]. The AV1 decoder's AXI clock and the JPEG encoders' run at 396 MHz, and the RGA
cores at 750 MHz [measured][CLK].

The decoder's timeout register, `0x080` (register 32), counts AXI clock cycles. Mainline
sets it by picture size: `0xefffff` up to 1080p, `0x2cfffff` up to 4K, `0x4ffffff` up to
8K, and `0xffffffff` above [source]. It arms its own watchdog at twice that threshold over
the AXI clock's rate. On a Rock 5B, the intervals between timeout resets fit a count of AXI
cycles [reported][YSP]. A decoder that resets itself also clears its MMU, which the driver
then reprograms (see [MMU state across power and
reset](system-reference.md#mmu-state-across-power-and-reset)).

The series splits the driver around `struct rkvdec_core`. A change touching that
structure applies only to a kernel carrying the series. An rkvdec fix therefore does not
move unchanged between a kernel with the series and one without it. The case arises
between an RK3588 kernel carrying the series and an RK3576 kernel without it.

### Blocks without a node

Two media blocks in the silicon present no node under mainline: a dedicated JPEG
decoder, and the IEP, the vendor kernels' deinterlacer. The mainline dtsi
carries clock, power-domain and QoS plumbing for both (`HCLK_JPEG_DECODER`,
`qos_jpeg_dec`, `qos_iep`), but no device node or driver [source].

The vendor kernel describes the IEP as `iep@fdbb0000`, `rockchip,iep-v2`, with `0x500`
bytes of registers and its own MMU at `fdbb0800`. The block and its MMU share
`GIC_SPI 117`. It takes an AXI, an AHB and a core clock, with the AXI clock at 594 MHz,
three resets, and sits in `PD_VDPU` [source][VENDOR]. The RK3588 has no VDPP [inferred].
No vendor RK3588 tree describes one, and the vendor VDPP driver matches only the RK3528
and RK3576 [source][VENDOR].

Sascha Hauer's mainline `rkjpegd` driver is in review upstream, at v4 (2026-09-18) and
unmerged. It describes the JPEG decoder as `video-codec@fdb90000` in `PD_VDPU`, with its
own IOMMU at `fdb90480`, clocked at 600 MHz.

The HDMI receiver is the opposite case. It has a device node and a mainline driver, but
the Turing RK1 module does not route it (see [HDMI receiver](#hdmi-receiver)).

### IEP2 in userspace

Rockchip's MPP library already carries the IEP2 userspace. `ENABLE_VPROC` is on by
default, and the reference unit's `librockchip_mpp.so` exports nine `iep2` symbols
[measured]. MPP selects the IEP version by probing device nodes rather than by SoC id.
Version 2 binds `/dev/mpp_service`, which the out-of-tree `mpp_service` VEPU580 port
already provides. Once a kernel-side `mpp_service` client 28 answers, libmpp binds it
with no userspace change.

A standalone C API (`iep2_api.h`) lets an ffmpeg filter drive IEP2 without going through
`mpp_dec_vproc.c`. The missing pieces are that kernel client and the filter. Two facts
bound the payoff:

- IEP2 is an 8-bit block capped at 1080, so it cannot process 10-bit content. It reads and
  writes 4:2:0 and 4:2:2, semi-planar or planar [source][VENDOR].
- With no IEP driver, `bwdif` deinterlaces on the CPU at 0.93 CPU-s per second of 1080i
  content [measured], about 1.35x what hardware decode saves. A whole 1080i to 1080p60
  transcode costs about 1.1 cores per realtime stream, against 0.15 without
  deinterlacing. 576i costs ~0.2 cores.

`bwdif_vulkan` produces correct fields but costs the same CPU at about a seventh of the
throughput (15.4 against 110 fps), and `yadif` is strictly worse. The cost case for an IEP2 port is weak. A port
also leaves a kernel client and an ffmpeg filter to maintain against a moving kernel.

### Formats with no stateless uAPI

The datasheet lists two more decode formats that no mainline driver can reach. AVS2 (in
the datasheet's decoder line) and MPEG-4 ASP and H.263 (VDPU2-class silicon) have no V4L2
stateless uAPI. In hardware they are reachable only through the vendor stack, whatever
the driver work. AVS2 and AVS3 have a software route through ffmpeg's `libdavs2` and
`libuavs3d` decoders, and it is a 1080p capability rather than a 2160p one [measured].

## Decoder capture formats

`rkvdec.c` declares exactly four capture formats, `NV12`, `NV15`, `NV16` and `NV20`, and
no `P010` [source]. The list belongs to the shared driver rather than to the part. The
RK3576 reads the same list from the same driver (see [the RK3576 capture
formats](../rk3576/video-reference.md#capture-formats)).

> **An idle node lists only `NV12`.** The driver filters the capture list by the current
> stream's bit depth, so `v4l2-ctl --list-formats` on an idle node shows no 10-bit
> format. To see `NV15`, read `rkvdec.c` or decode a real 10-bit clip. An idle node's
> list is no evidence against a 10-bit path.

Measured through an ffmpeg build carrying the `v4l2request` hwaccel [measured,
2026-08-09, 7.1.1]:

| Input | Capture format | Decode errors |
| --- | --- | --- |
| HEVC Main (8-bit) | `NV12` | 0 |
| HEVC Main 10 | `NV15` | 0 |
| VP9 profile 0 | `NV12` | 0 |
| VP9 profile 2 | `NV15` | 0 |

The VP9 rows are from 2026-08-30 on 7.2.2. VP9's bit depth can change at any frame, and
the capture format follows it mid-stream. Both profiles are bit-exact against `libvpx` over
120 frames, and the kernel log stays clean over all 305 conformance vectors [measured,
2026-08-30, 7.2.2]. The hardware has no VP9 profile 1 or 3, and the profile control
refuses them [source].

## Resolution ceilings

Decode at 8K is bit-exact on every codec tested, at both bit depths, and 8K HEVC encode
round-trips [measured, 7.2.2]. The clips are 4K jellyfish content upscaled to 7680×4320
on an x86 host and re-encoded there. The references are `framemd5` hashes from software
decode on the same host.

| Stream | Frames | Result |
| --- | ---: | --- |
| 8K H.264 | 60 | Bit-exact, 1.08 CPU-s |
| 8K HEVC 8-bit | 60 | Bit-exact, 1.33 CPU-s |
| 8K HEVC 10-bit | 60 | Bit-exact, with the RCB mapped through `dma_map_resource()` (see [the RCB SRAM IOVA collision](#the-rcb-sram-iova-collision)) |
| 8K VP9 | 30 | Bit-exact, 0.72 CPU-s |
| 8K HEVC encode | 30 | `hevc_rkmpp -b:v 60M` output decodes back in hardware with 0 errors |

Nothing in the rkvdec driver limits 8K. `--list-framesizes` reports 1×1 to 65472×65472
for HEVC and VP9, and 1×1 to 65520×65520 for H.264. Decode at 8K and 8 bits therefore
needs no driver change.

AV1 is the exception, because its cap is a driver constant: 4096×2304 in the mainline
driver. Raising the AV1 decoder's frame-size cap to 8192×4352 is enough, and decode is
bit-exact at exactly that size (see [AV1 decode](#av1-decode)).

The cap sits in two format lists: the tiled formats (`NV12_4L4`, `NV15_4L4`) and the
post-processor's `rockchip_vpu981_postproc_fmts[]` [source, 7.2.9]. `hantro_try_fmt()`
applies the capture format's limits. A cap raised in one list alone therefore returns
4096×2304, with no error [measured, 2026-09-02].

### The RCB SRAM IOVA collision

The rkvdec driver maps its RCB with a hand-rolled `iommu_map()` in `rkvdec_allocate_rcb()`.
That mapping makes 8K 10-bit HEVC decode fail deterministically with `dma alloc of size
49766400 failed`. The line before it is always
`iova: 0x…81178000 already mapped to 0x…ff079000 cannot remap`, byte-identical across
hours and workloads. The failure reads as a 10-bit or an allocation-size ceiling, and it
is neither. Each link of the chain is measured:

1. `rk3588-base.dtsi` gives `system_sram2` two pools: `vdec0_sram`, 480 KiB at
   `0xff001000`, and `vdec1_sram`, 476 KiB at `0xff079000`. The second address is the
   stale physical address in the message.
2. `rkvdec_allocate_rcb()` maps the RCB out of that gen_pool into the shared domain,
   using the CPU virtual address as the IOVA.
3. That domain is the device's DMA default domain, whose IOVA allocator serves every
   `dma_alloc_coherent`, and nothing reserves the RCB range in it.
4. `rockchip-iommu` masks IOVAs to 32 bits, and `__iommu_map()` never checks the aperture.
   The 64-bit ioremap VA silently aliases to `0x81178000`.
5. vb2 capture buffers take IOVAs from the top of the 4 GiB aperture downward. Once
   ~2 GiB is in use, the allocator hands out a range covering that address, and the map
   returns `-EADDRINUSE`.

Size, bit depth and codec are all incidental. Only the cumulative IOVA demand in the one
domain matters, which is why three concurrent 8-bit 8K decodes fail at the same address.

Measurement excludes the other candidate causes:

- A leak: `iommu:map` and `iommu:unmap` ftrace over full runs leaves zero residue.
- Stale page tables: the page tables are zeroed.
- The v2 DTE and PTE format: the 40-bit encoding round-trips.
- CMA exhaustion and fragmentation.

`-extra_hw_frames` does not help at 0, 2 or 4.

Replacing the hand-rolled `iommu_map()` with `dma_map_resource()`, and with
`dma_unmap_resource()` on free, fixes it. The IOVA then comes from the same allocator the frame buffers use, so it cannot
be issued twice. With that change, the 8K 10-bit decode passes 5 of 5 runs against an
unchanged control that fails 4 of 5 [measured]. Three and six concurrent 8K decodes run
with no `EADDRINUSE`, and 8K 10-bit output is bit-exact [measured].

The consequence that matters is concurrency rather than 8K. The collision caps any
workload whose cumulative IOVA demand descends far enough. Three concurrent 8K streams
trip it, and many-stream 4K work is exposed to it as well.

Without the change, three concurrent 4K HEVC 10-bit decodes already fail. The kernel logs
`dma alloc of size 19701760 failed`, and aggregate throughput falls below one stream's
[measured, 2026-08-13, 7.1.7]. The signature fits this collision [inferred]. Three or more
4K 10-bit streams are not measured with the change.

The driver allocates the RCB buffer by buffer. Each buffer comes from SRAM if it fits, and
from DRAM otherwise [source]. VDPU381's size table, each buffer rounded to 4 KiB, totals
about 196 KiB at 1080p, 372 KiB at 4K and 736 KiB at 8K [inferred]. A 4K stream's RCB
therefore fits in one core's pool, and an 8K stream's spills part of it to DRAM.

The vendor kernel avoids the collision by construction. Each decoder core's node carries
`rockchip,rcb-iova`, a fixed 1 MiB window at `0xfff00000` for core 0 and `0xffe00000` for
core 1. The driver reserves that window in the IOVA allocator and maps the RCB into it
[source][VENDOR].

When changing this code, confirm that the SRAM path still serves the RCB. A fallback to
`dma_alloc_coherent` also makes the collision vanish, for the wrong reason and at a
performance cost. To trace an RCB allocation, decode something wider than anything
decoded since boot. The RCB only ever grows, and the driver frees it only on remove.

## Conformance

Each score is read against a software decode on the same unit, and the failures software
shares are subtracted. What remains is attributable to the driver [measured, 2026-08-31
and 2026-09-02, 7.2.2, the VP9 row 2026-08-30, 7.2.2, and the H.264 rows 2026-10-07,
7.2.9].

| Suite | Node | Score | Residual |
| --- | --- | --- | --- |
| `JCT-VC-HEVC_V1` | rkvdec | 146/147 via GStreamer, 141/147 via ffmpeg | The one GStreamer miss, `TSUNEQBD_A_MAIN10` (unequal luma and chroma depth), fails in software too. Software ffmpeg also scores 141/147. Four of the ffmpeg misses fail in software as well. `CONFWIN_A_Sony_1` and `DELTAQP_A_BRCM_4` fail in hardware only, and pass under GStreamer, so they lie in the `v4l2request` hwaccel. The 141 needs [the SPS-level RPS controls](#hevc-reference-picture-sets) |
| `JVT-AVC_V1` | rkvdec | 129/135 via ffmpeg and via GStreamer | Software fails five of the six: FMO x3, SP x2. The sixth, `CVFC1_Sony_C` (left and top crop offsets), is in upstream's published failure set |
| `JVT-AVC_V1` | VDPU2 | 129/135 via ffmpeg and via GStreamer | The same six |
| `JVT-FR-EXT` | rkvdec | 65/69 via ffmpeg and via GStreamer | 4:2:2 8-bit and 10-bit (`NV16`, `NV20`) decode correctly. Fails: monochrome x2 (no capture format), MBAFF 4:2:2 x2 |
| `JVT-FR-EXT` | VDPU2 | 44 of the 46 vectors it accepts | Monochrome x2, as on rkvdec. It refuses the 10-bit and 4:2:2 vectors |
| `VP8-TEST-VECTORS` | VDPU2 | 59/61, the software score exactly | None attributable |
| `VP9-TEST-VECTORS` | rkvdec | 188/305 raw, 224/305 attributable | Software ffmpeg fails the same 36 `resize_*` vectors, whose mid-stream size change ffmpeg's harness cannot follow. The 224 matches the score the VDPU381 VP9 series reports upstream |
| `VP9-TEST-VECTORS-HIGH` | rkvdec | 1/6 | The one pass, `vp92-2-20-10bit-yuv420`, is the only vector in the hardware's reach. The rest are 12-bit or 4:2:2 and 4:4:4 |
| `AV1-TEST-VECTORS` | VPU981 | 201/242 | See [AV1 decode](#av1-decode) |
| `MPEG2_VIDEO-MAIN` | VDPU2 | 20/43 raw | Three failure classes, one real defect (see [MPEG-2 on VDPU2](#mpeg-2-on-vdpu2)) |

### Attributing a failure

Running both userspaces settles attribution. Fluster's `GStreamer-*-V4L2SL` decoders work
once `gstreamer1.0-plugins-good` and `gstreamer1.0-plugins-bad` are installed. A vector
that fails under ffmpeg's `v4l2request` but passes under GStreamer is a userspace gap. The
HEVC row, 146 against 141, is that case. A vector that fails under both, while software
passes, is the driver's. A 43-vector suite takes about a minute, so this is the default
second step whenever a score looks wrong.

The node a vector lands on is not obvious. `VP8F` and `MG2S` exist only on VDPU2, but
H.264 exists on both nodes. The request-API hwaccel as posted walks the media devices in
udev's order, sorted by sysfs path, and takes the first that accepts the stream. VDPU2
(`fdb50000`) sorts before rkvdec (`fdc38000`), so there every 8-bit 4:2:0 H.264 vector up
to 1920×1088 lands on VDPU2.

An ffmpeg that ranks the decoders by the widest frame size they report for the coded
format tries rkvdec first, at 65520×65520 against VDPU2's 1920×1088. There every H.264
vector lands on rkvdec, and VDPU2 takes one only if rkvdec refuses it.

Neither honors `-hwaccel_device`: the hwcontext discards the device string. To score
one node, hide the other's media node in a private mount namespace, and the probe skips
it. Hiding `/dev/media0` scores VDPU2, and hiding `/dev/media1` scores rkvdec:

```sh
unshare -rm sh -c 'mount --bind /dev/null /dev/media0 && exec "$@"' sh ffmpeg ...
```

`udevadm info -q path -n /dev/mediaN` names the block behind each media node.

GStreamer names the two H.264 decoders as separate elements: `v4l2slh264dec` is VDPU2, and
`v4l2slvideo0h264dec` is rkvdec. VDPU2 accepts `S264` and `MG2S` from 48×48 to 1920×1088 in
steps of 16, and `VP8F` from 48×48 to 3840×2160.

### HEVC reference picture sets

The VDPU381 parses each slice header in hardware. A slice can select its short-term
reference picture set from the SPS by index, so the driver needs the SPS's sets from
userspace. Two controls carry them, `V4L2_CID_STATELESS_HEVC_EXT_SPS_ST_RPS` and
`V4L2_CID_STATELESS_HEVC_EXT_SPS_LT_RPS` [source, 7.2.9].

Without the controls, the driver logs `Long and short term RPS not set` and decodes anyway
[source, 7.2.9]. Inter frames then predict from wrong references, and ffmpeg reports
success. An ffmpeg that sets neither control scores 15/147 on `JCT-VC-HEVC_V1`, against
141/147 with both [measured, 2026-08-31, 7.2.2].

Whether a stream needs the controls depends on its encoder. x265 writes each picture's set
inline in its slice headers (`num_short_term_ref_pic_sets = 0`). Its output therefore
decodes correctly without them. The HM reference encoder writes SPS-level sets, as the conformance
vectors do [measured]. `-bsf:v trace_headers` shows which kind a file is.

> **A missing control produces wrong pictures, not an error.** The only sign is one
> rate-limited kernel warning. Real-world x265 content passes, so test with conformance
> vectors.

### MPEG-2 on VDPU2

A fluster `MPEG2_VIDEO-MAIN` `Success` means a match within ±2, not bit-exactness:
fluster's `compare_yuv_files()` takes `tolerance: int = 2`. Measured against that, the VDPU2
MPEG-2 path is almost never bit-exact. Of the passing vectors checked, only one matches
software. Every other one sits at a maximum deviation of exactly 2, on the boundary. The
pass/fail line is a tolerance boundary rather than a behavioral one. Progressive streams
are no exception, and the failures are not one interlaced phenomenon.

The 16 failures fall into three classes:

- Nine are not defects. Their maximum deviation is 3-4, their mean 0.02-0.05, with no
  spatial structure, and four of them fail on a single sample in ~30 million. This is
  IDCT accuracy drift, which 13818-4 permits, accumulating through inter prediction. That
  is why all six long and short pairs from the same source split on length alone.
- Six are one real defect: field reference parity on bottom-field-first field pictures.
  Every one is `bb`, and no `tt` stream is among them. The intra frame is clean, then the
  first inter-predicted picture splits by row parity. Even rows measure 68.1 against 9.9
  for odd rows, a 6.9x split across 100% of macroblocks. `sony-ct3` is `bb` and passes, so
  the trigger is narrower than `bb`.
- One is separate and unexplained. `teracom_vlc4` has a maximum deviation of 255 but a
  mean of 0.043, over 0.02% of samples: sparse catastrophic outliers with no parity
  structure.

For the field-parity defect, read the first bad frame rather than the worst one. The
signature dilutes as the bad field becomes a reference, and then returns.

Dual prime is not the mechanism: `toshiba_DPall-0` is dual prime throughout and passes.
Six more of the 43 vectors use MPEG-1 syntax, which the stateless path does not decode.
ffmpeg flips `codec_id` at runtime, so the hwaccel never engages, and GStreamer fails them
too. One more vector is 32×32, below the node's 48×48 floor.

Comparable results need fluster's own flags: `-fps_mode passthrough` and the
`hwdownload,format=nv12` filter. A per-frame MD5 tells nothing here, because every frame
differs. Measure the maximum absolute sample deviation and the mean per row parity, since
PSNR hides the parity split. Hardware decode is deterministic: every vector decodes
identically twice [measured].

## AV1 decode

`/dev/video3` decodes AV1 under mainline, through `hantro-vpu` on the
`rk3588_vpu981_variant` [source]. It is different silicon from the RK3576's VDPU383,
where AV1 needs the vendor MPP stack. The out-of-tree VDPU383 AV1 driver and its
known-wrong output have no bearing on this block (see [the RK3576
decoder](../rk3576/video-reference.md#the-decoder)).

The mainline dtsi describes the decoder as `video-codec@fdc70000`, one `0x800` window on
`GIC_SPI 108` [source]. The vendor kernel describes two more banks of `0x400` each: a
cache at `fdc80000` on `GIC_SPI 107`, and an AFBC unit at `fdc90000` on `GIC_SPI 106`
[source][VENDOR]. Mainline drives neither.

The decoder accepts frames from 64×64 up to its driver cap. With the cap raised to
8192×4352, decode is bit-exact at 7680×4320 and at 8192×4352 [measured, 7.2.2]. Output
is bit-exact over ordinary content, with four bounded gaps (see [Gaps](#gaps)).

The conformance run decodes the 242 AOM `av1-test-data` vectors through `-hwaccel
v4l2request` [measured, 2026-08-11, 7.1.7]. It compares per-frame MD5s against AOM's own
published reference `.md5` files. A further 1200 frames of locally encoded 1080p and 4K
content add a second check.

> `ffmpeg -pix_fmt yuv420p -f framemd5` output equals AOM's reference `.md5` files byte
> for byte. That makes the published vectors a normative external check rather than a
> self-comparison. On an x86 host, all 242 vectors pass this comparison under libdav1d.

| Outcome | Vectors | Meaning |
| --- | ---: | --- |
| Bit-exact | 136 | Matches AOM's reference MD5 on every frame |
| Refused, correctly | 32 | Below the driver's advertised minimum frame size |
| Decoded, unreadable | 65 | 10-bit, decoded with zero errors. The ffmpeg build used had no `NV15` readback. Not re-scored with one that can (see [10-bit AV1](#10-bit-av1)) |
| Refused | 2 | Monochrome. The hwaccel offers no pixel format |
| Wrong output | 7 | Three at the size boundary, one IntraBC, three scalability |

Everything ordinary passes, including:

- Film grain synthesis, 8-bit, bit-exact
- `superres`, `cdfupdate`, `mfmv` and `allintra`
- Mid-stream resolution change
- All 64 non-square size combinations from 196×196 to 226×226

Content encoded locally with SVT-AV1 from the jellyfish clips, compared against libdav1d,
gives 1200 frames with zero mismatches:

| Clip | Tiles | Result |
| --- | --- | --- |
| 1080p 8-bit, 300 frames | 1 | Bit-exact |
| 1080p 8-bit, 300 frames | 8 | Bit-exact |
| 4K 8-bit, 300 frames | 1 | Bit-exact |
| 4K 8-bit, 300 frames | 32 | Bit-exact |

### Tile-count bounds

Linux bounds crafted tile counts upstream from 7.2.7 [source, 7.2.8]. The V4L2 core's
`validate_av1_tile_info()` caps `tile_cols` and `tile_rows` at 64, and their product
at 512. `rockchip_vpu981_av1_dec_prepare_run()` refuses a frame claiming more tiles than the
tile-group control carries, or more than the 128 descriptors this core's scratch holds.
The kernel itself therefore closes the out-of-bounds writes and the divide-by-zero that a
crafted stream reaches without these checks. `std_validate_compound()` bounds the HEVC
side the same way, for `rkvdec` as for every stateless HEVC driver.

One consequence is a real limit: this hardware refuses a conforming AV1 frame of more than
128 tiles. The refusal runs the driver's `->run()` error path. On its own, that path
finishes the job twice (a `WARN_ON(!src)` splat) and arms a watchdog for a job that never
starts. The watchdog aborts an unrelated concurrent decode about 2 s later. Sascha Hauer's
unmerged `verisilicon` run-failure cleanup makes the refusal clean.

### Gaps

Every wrong-output case is deterministic: repeated runs produce identical MD5s. That sets
these gaps apart from the run-to-run variation of the [RK3576 H.264 power-up
erratum](../rk3576/h264-power-up-erratum.md).

#### Frame height of 64

A frame height of exactly 64 returns an all-zero buffer. The result is deterministic at
every width tested (256, 640, 1920), with natural content and no screen-content tools.
Heights 66, 72, 80, 96 and 128 are all bit-exact, so the boundary is exact and narrow.
ffmpeg reports success and zero decode errors, and the buffer is never written.

A height of 64 is both the driver's advertised minimum and exactly one superblock row. The
driver's own height arithmetic (`height_in_sb = ALIGN(h, 64) / 64`, `stripe_num`) is
well-formed at that value [source]. A read of the buffer sizing does not reveal the cause.

#### IntraBC

IntraBC (intra block copy) decodes wrong. An A/B test on one clip isolates it. The same
256×256 content, encoder and preset mismatches on all 30 frames with `allow_intrabc=1` in
the bitstream. Re-encoded with `-svtav1-params scm=0`, the clip is bit-exact.

The failure is structural rather than a rounding difference: the frame is correct through
the first superblock row, and wrong below it. AOM's
`av1-1-b8-16-intra_only-intrabc-extreme-dv` fails the same way, leaving most of the frame
unwritten. The driver plumbs the tool: it writes `av1_allow_intrabc` and has an IntraBC
MV-context path [source]. The failure is therefore a defect rather than an omission.

`allow_intrabc` in the bitstream predicts the failure exactly. Across 16 content×encoder
combinations, every stream carrying it decoded wrong on every frame. Every stream without
it was bit-exact, with no partial cases. To check a file before playing it, search its
headers for the flag:

```sh
ffmpeg -v trace -i FILE -c copy -bsf:v trace_headers -f null - 2>&1 | grep allow_intrabc
```

#### Scalability

Scalability (SVC) decodes wrong. All three AOM SVC vectors (`L1T2`, `L2T1-2`, `L2T2-2`)
return flat mid-gray, 7.8-13.3 dB against the reference. Temporal and spatial layering
both fail.

#### Monochrome

The driver refuses monochrome at format negotiation, 8-bit and 10-bit alike, with `Failed
to get pixel format`. The node has no gray capture format.

### IntraBC in encoder output

Six content classes, each encoded at stock settings by both encoders, then decoded on the
hardware [measured, 2026-08-11]:

| Content | SVT-AV1 default | libaom default | libaom `-tune-content screen` |
| --- | --- | --- | --- |
| Camera (jellyfish) | Clean | Clean | Not run |
| Animation-like (flat banded regions) | Clean | Clean | Not run |
| Screen capture (text on a solid background) | Clean | Clean | Clean |
| Dense terminal (17 near-identical text lines) | Clean | Clean | Clean |
| UI grid (32 repeated identical boxes) | IntraBC | Clean | Clean |
| Synthetic test pattern (`testsrc2`) | IntraBC | Clean | Not run |

The `libaom` encoder never emitted IntraBC in any test, including runs tuned for screen
content. The encoder behind most AV1 in the wild therefore does not reach this defect.
SVT-AV1 enables IntraBC unprompted, with no user opt-in, but only on imagery with strong
repeated structure. Camera and animation content, and a realistic screen capture, all
stayed clean.

For a media library, the defect is close to unreachable. The realistic trigger is SVT-AV1
re-encoding synthetic or UI-like imagery. The failure mode is what makes it worth knowing:
**every frame is wrong, and nothing reports an error.** ffmpeg exits 0, and the decoder
reports zero decode errors.

> Validate this decoder with camera content, or force `scm=0`, and never with `testsrc2`
> or another synthetic pattern. SVT-AV1 enables screen-content tools on such patterns, so
> the output mismatches for a reason unrelated to the test.

### The upstream conformance baseline

The driver's upstream submission reported its own conformance: Fluster 200/239 on
`AV1-TEST-VECTORS` via `GStreamer-AV1-V4L2SL-Gst1.0`. The submission is Benjamin
Gaignard's (Collabora) `[PATCH v3 00/13] AV1 stateless decoder for RK3588`, January 2023,
which reported 151/239 at v1. The cover letter attributed the failures to three causes:

- The two 2-spatial-layer tests ("few errors in luma/chroma values")
- Tests below the 64×64 hardware limit
- The 10-bit film grain test

Measured on 7.1.7 through ffmpeg rather than GStreamer, the residual matches:

| | Upstream, January 2023 | Reference unit, 2026-08-11 |
| --- | ---: | ---: |
| Decoding correctly | 200 / 239 | 201 / 242 ¹ |
| Attributed failures | 35 | 34 ² |
| Unattributed or unnamed failures | 4 | 7 ³ |

¹ 136 verified bit-exact, plus 65 ten-bit vectors that decode with zero errors but that
the ffmpeg build used failed to read back. That 201 lands one off upstream's 200 is a
useful cross-check. GStreamer converts `NV15` itself, and so scores those 65 as passes.

² 32 below the advertised minimum size, plus 2 monochrome.

³ The 7 wrong-output cases. Two of them are upstream's named spatial-layer failures.

The decoder's residual failure set is small and stable, and each failure in it is named.
The cover letter does not cover three of them:

- IntraBC, though its vector is in the suite.
- A frame height of exactly 64. The cover letter excuses "resolution < hardware limit
  (64×64)", but a 256×64 or 1920×64 frame is at that limit rather than below it. The
  driver advertises it as supported.
- Monochrome. Both monochrome vectors are in the current suite, and the driver refuses
  them.

The SVC result on the reference unit is also worse than the cover letter describes. The
letter reports "few errors in luma/chroma values", and the measured output is flat
mid-gray at 7.8-13.3 dB. `L1T2` fails as well, though it has only one spatial layer.
Whether that is drift since 2023 or a difference between the two userspaces is untested.

A search of linux-media and the patch history on 2026-08-11 found no report of the
IntraBC or height-64 behavior.

### 10-bit AV1

The 65 ten-bit vectors in the outcome table are not decoder failures. The hardware
selects `NV15` capture and reports zero decode errors. Reading the frames back needs an
ffmpeg whose `v4l2request` hwcontext tags the CAPTURE buffer as a real `NV15` frame. With
that, `hwdownload,format=nv15` followed by `format=yuv420p10le` brings a 10-bit AV1 frame
to system memory. The decoder's automatic transfer handles it too.

The RGA filters take the same frames in hardware at any width once the AV1 decoder pads
its capture stride to 64 bytes (see [the 10-bit stride](#the-10-bit-stride)). With both
changes, 1280-, 1920- and 3840-wide AV1
10-bit clips blit and download bit-exact against a software decode made on another
machine [measured, 2026-08-31, 7.2.2]. The 65 vectors are not re-scored against AOM's MD5s with that readback. The
locally encoded clips stand in for them.

### Decode cost

Decode-only cost over 300 frames (`-hwaccel_output_format drm_prime -f null -`), so with
no readback or conversion [measured, 2026-08-11, 7.1.7]:

| Clip | Wall | CPU | Throughput | CPU-s per 30 s of video |
| --- | ---: | ---: | ---: | ---: |
| 1080p 8-bit | 1.14 s | 0.20 s | 263 fps (8.8x realtime) | 0.60 |
| 4K 8-bit | 3.45 s | 0.35 s | 87 fps (2.9x realtime) | 1.04 |
| 4K 10-bit | 3.50 s | 0.38 s | 86 fps (2.9x realtime) | 1.14 |

That puts AV1 in the same cost class as HEVC decode through `v4l2request` on this SoC.
HEVC costs 0.63 CPU-s per 30 s at 1080p 8-bit, and 2.01 at 4K 10-bit [measured]. Tile
count does not change the cost. At 4K, 32 tiles cost the same as one, so there is no
tile-level parallelism to exploit and no penalty for heavily tiled streams.

Reading frames back costs far more than decoding them. The same 4K clip takes 18.7
CPU-seconds once the frames are pulled into system memory and converted from `NV12` to
`yuv420p` on the CPU. That figure includes MD5 hashing, against 0.35 for the decode
itself. A consumer of AV1 frames on this SoC takes them as DRM PRIME to avoid that cost.

## NV15 consumers

Decoding 10-bit content works, and consuming the output is the constraint. `NV15` is
10-bit packed 4:2:0 (no byte-aligned samples), and the hardware writes it natively. No
driver change can substitute a friendlier format.

| Consumer | Takes NV15 |
| --- | --- |
| VOP2 display planes | Yes. `rockchip_drm_vop2.c` handles `DRM_FORMAT_NV15`, `NV20` and `NV30` [source] |
| RGA | Yes, at a stride that is a multiple of 64 bytes (see [the 10-bit stride](#the-10-bit-stride)). It converts `NV15` to `P010` and `NV20` to `P210`, the one RGA capability nothing in mainline replaces |
| `hwdownload` and swscale | Yes, in an ffmpeg whose hwcontext tags the buffer as a real `NV15` frame. `format=yuv420p10le` then unpacks it on the CPU, which is what feeds a GPU tone mapper |
| Vulkan and PanVk | No. Vulkan has no 10-bit packed 4:2:0 format |
| Mesa GL and EGL | Yes, from Mesa 25.0. The DRI frontend imports `DRM_FORMAT_NV15`, and panfrost samples it as a native 10-bit YUV texture on this GPU [source][MESA]. Not measured on the reference unit |
| ffmpeg Vulkan hwcontext | No. Its DRM import table carries no multi-planar YUV fourcc |

Two consequences follow:

- 10-bit playback through a KMS plane needs no conversion, because VOP2 scans `NV15` out
  directly.
- 10-bit transcode converts on RGA to stay in hardware, or downloads and unpacks on the
  CPU. That serial unpack is the cost a tone-mapping chain pays.

A result from a player such as Kodi or mpv therefore does not transfer to a transcode
pipeline, because the consumers differ.

## Transcode costs

Hardware decode is the largest lever in a transcode on this SoC. Per stage, in CPU-seconds
for a 30 s clip of 900 frames [measured, 2026-08-10, 7.1.7]:

| Stage | 1080p HEVC 8-bit to 720p | 4K H.264 8-bit to 1080p |
| --- | ---: | ---: |
| Decode, software | 33.30 | 126.76 |
| Decode, `v4l2request` | 0.63 | 1.81 |
| Download to system memory | 2.89 | 11.81 |
| Scale, swscale | 17.38 | 35.98 |
| Encode, `hevc_rkmpp` or `h264_rkmpp` | ~0 | ~0 |

Hardware decode is 53x cheaper at 1080p and 70x at 4K. Once decode is in hardware, the
download and the CPU scale are most of what remains. RGA and the GPU take that work over.

### 10-bit

A 10-bit hardware path spends its CPU unpacking `NV15`. For 4K HEVC Main 10, 900 frames
[measured, 2026-10-07, 7.2.9]:

| Stage | CPU-s | Speed |
| --- | ---: | ---: |
| Decode, frames kept as DRM PRIME | 2.21 | 2.95x |
| Plus the download | 16.2 | 2.09x |
| Plus `format=yuv420p10le`, the `NV15` unpack | 75.8 | 1.93x |
| Plus a scale to 1080p `NV12`, in place of the unpack | 95.6 | 1.66x |
| Software decode alone | 316.2 | 0.60x |

The `NV15` reader in swscale is scalar C, and it takes ~80% of the hardware path's CPU.
`scale_rkrga` converts `NV15` to `NV12` on the RGA instead, which keeps the unpack off the
CPU (see [the 10-bit stride](#the-10-bit-stride)).

H.264 High 10 decodes on rkvdec to `NV15`, bit-exact against software over 900 frames
[measured, 2026-10-07, 7.2.9]. VDPU2 refuses its SPS during the probe, and logs
`Failed to set 1 control(s): Invalid argument (22)` once. 10-bit AV1 from SVT-AV1 matches
both libdav1d and libaom on all 900 frames.

### HDR tone mapping

The cheapest HDR10 to SDR route scales on RGA first, then tone maps in libplacebo
[measured, 2026-08-30, 7.2.2]. The filter chain runs `scale_rkrga`, then
`hwdownload,format=nv15,format=yuv420p10le`, then `libplacebo`, with no `hwupload`:

| Clip | Route | CPU-s | Speed |
| --- | --- | ---: | ---: |
| 1080p HDR10 to 720p, 900 frames | RGA prescale, then libplacebo | 18.84 | 26.5 fps |
| 1080p HDR10 to 720p, 900 frames | Software decode, then libplacebo | 50.37 | 24.6 fps |
| 1080p HDR10 to 720p, 900 frames | Hardware decode and RGA scale, no tone map | 1.83 | 335 fps |
| 4K HDR10 to 1080p, 300 frames | RGA prescale, then libplacebo | 12.50 | 12.1 fps |
| 4K HDR10 to 1080p, 300 frames | Software decode, then libplacebo | 56.90 | 9.9 fps |

The hardware-decoded tone map is bit-identical to the software-decoded one. A CPU tone map
through `zscale` costs 4.1x libplacebo's CPU at 1080p. 4K HDR10 stays short of realtime on
every route.

A graph carries one `-filter_hw_device`, and `scale_rkrga` needs the RKMPP device. With no
Vulkan device bound, libplacebo creates its own and does its own upload and download.
Frames do not move between RKMPP and Vulkan in either direction: `hwmap` and `hwupload`
with `derive_device` fail with `ENOSYS`.

`P010` in this chain corrupts the picture without an error:

- `scale_rkrga=format=p010` followed by `hwdownload,format=p010le` reads a `P010` stride
  from a buffer laid out at the `NV15` stride. `NV15` output costs the same, 18.68 against
  18.84 CPU-s.
- A `p010le` frame uploaded for libplacebo reads as all zeros, and `libplacebo=format=p010le`
  followed by `hwdownload` gives a flat green field. `yuv420p10le` works on both sides.
  ffmpeg's own Vulkan `P010` round trip is bit-exact, which places the fault in
  libplacebo's frame mapping [inferred].

> **Distrust the picture from any graph that carries `P010`, until a frame has been
> looked at.**

### Subtitle burn-in

RGA is the cheapest and the fastest burn-in route. A 1080p HEVC clip goes to 720p HEVC
through `hevc_rkmpp`, with typeset ASS burned in at the output size, over 900 frames
[measured, 2026-08-30, 7.2.2]:

| Route | CPU-s | Speed |
| --- | ---: | ---: |
| No subtitles: hardware decode and `scale_rkrga` | 2.04 | 401 fps |
| RGA: `scale_rkrga` and `overlay_rkrga` | 12.88 | 77.5 fps |
| Vulkan: `libplacebo` and `overlay_vulkan` | 19.36 | 33.3 fps |
| Hardware decode, swscale and `subtitles` | 27.29 | 73.0 fps |
| Everything on the CPU | 49.36 | 75.8 fps |

All four subtitled routes produce the same picture. Most of what the RGA route spends is
libass. Typeset ASS costs it 9.85 CPU-s per 900 frames at 1080p, and 34.82 at 4K. At 25 fps
the 1080p figure is about a quarter of one core. `overlay_rkrga` also composites directly
over a 3840-wide `NV15` frame, with no conversion stage.

## GPU and Vulkan

The GPU is a Mali-G610 MC4 (Valhall, CSF), driven by the mainline panthor driver. With
`vulkan-tools` and `mesa-vulkan-drivers` installed on a headless system, the reference
unit enumerates two Vulkan devices [measured, 2026-08-09]:

```
GPU0: Mali-G610 MC4   driverName = panvk   driverID = DRIVER_ID_MESA_PANVK
      Mesa 26.1.5-1   apiVersion = 1.4.354   conformanceVersion = 1.4.1.2
GPU1: llvmpipe (software)
```

Every extension that ffmpeg's DRM-PRIME import requires is present:

- `VK_EXT_image_drm_format_modifier` (rev 2)
- `VK_EXT_external_memory_dma_buf`
- `VK_KHR_external_memory_fd`
- `VK_KHR_external_semaphore_fd`
- `VK_EXT_queue_family_foreign`
- `VK_KHR_sampler_ycbcr_conversion` (rev 14)

Panthor reads the GPU's identity and feature registers at probe, and returns them through
its `DEV_QUERY` ioctl [measured, 2026-10-23, 7.2.9][GPUQUERY]:

| Property | Value |
|---|---|
| `GPU_ID` | `0xa867_0005`: architecture 10.8.6, product major 7, r0p0 |
| `CSF_ID` | `0x040a_0412`: command stream hardware 1.0.10, MCU 1.1.2 |
| Shader cores | `shader_present` `0x50005`, four cores |
| L2 and tiler | One L2 (`l2_present` 1), `L2_FEATURES` `0x0712_0306` with 64-byte lines, one tiler |
| `MEM_FEATURES` | `0x301`, coherent core groups |
| GPU MMU | `MMU_FEATURES` `0x2830`: 48-bit virtual and 40-bit physical addresses. Eight address spaces, `AS_PRESENT` `0xff` |
| Threads | 2,048 per core, workgroups of up to 1,024, `THREAD_FEATURES` `0x0401_0000` |
| Coherency | `COHERENCY_FEATURES` 0: no ACE or ACE-Lite protocol. Panthor selects none |
| Command stream interface | 8 group slots, 8 stream slots, 96 registers, 8 scoreboard slots |
| Timestamp | 24 MHz, the system counter's rate |

The GPU does not snoop the cores' caches, in line with the rest of the SoC (see
[coherency](cpu-reference.md#coherency)). The CSF firmware reports interface v1.5.0, and
supports no protected mode entry [measured, 2026-10-05, 7.2.9][BOOTLOG].

The GPU takes its operating clock from the CRU and its voltage from `vdd_gpu_s0`. Its devfreq
offers 300 to 1000 MHz in 100 MHz steps, under `simple_ondemand` [source]. It idles at
300 MHz with the rail at 675 mV [measured, 2026-10-23, 7.2.9][CLK]. The `gpu-thermal` zone
limits it at 85 °C (see [thermal sensors](system-reference.md#thermal-sensors)).

Two facts matter for planning around the GPU:

- `/dev/dri/renderD128` exists on a headless system with no display attached, and `card0`
  binds panthor. The CSF firmware, `arch10.8/mali_csffw.bin` from the
  `firmware-misc-nonfree` package, is what makes that work.
- `llvmpipe` enumerates alongside PanVk. Anything that selects a Vulkan device by default
  can silently land on a software rasterizer. It then appears to work while running
  slower than a CPU path. Pin the device and assert `driverName == panvk`.

Mesa 26.2 changes how PanVk samples YUV on this GPU generation, so a Vulkan result taken
on 26.1.x is not automatically one for 26.2 [source]. MR !42353 samples NV12, NV16 and
I420 through the hardware's native YUV textures on arch v9 to v13, the G610 included.
MR !42451 does the same for P010. Mesa 26.1.x treats every YCbCr format as separate
per-plane images.

The native 10-bit path has to rescale, because the hardware normalizes a 10-bit sample by
1020 (`0xFF << 2`) rather than 1023 (`panvk_vX_nir_lower_descriptors.c`). Both MRs merged
ahead of the 26.2 branch, and every Vulkan result this page cites was taken on 26.1.x.
PanVk allows no storage use on any YCbCr format (`panvk_physical_device.c`), so the native
path serves sampling only.

The G610 is arch v10 with native timeline semaphores, which is what keeps ffmpeg's Vulkan
hwcontext working on this SoC. The RK3576's Bifrost G52 has neither a usable PanVk nor
that path (see [the RK3576 GPU](../rk3576/video-reference.md#gpu)).

### Vulkan filters

With hardware decode on both sides, PanVk scales far below the CPU's cost
[measured, 2026-08-11, 7.1.7]:

| 1080p scale, 900 frames | CPU-s | Speed | Luma against swscale |
| --- | ---: | ---: | --- |
| swscale | 19.98 | 144 fps | |
| `scale_vulkan` | 1.11 | 409 fps | 53.79 dB |
| `libplacebo` | 3.52 | 108 fps | 49.4 dB |

An 8-bit `NV12` DRM PRIME frame imports into Vulkan without a copy only as two layers, an
`R8` luma layer and a `GR88` chroma layer. ffmpeg's import table carries no multi-planar
YUV fourcc, and a single `NV12` layer fails with `Unsupported DMABUF layer format
0x3231564e` [source]. `overlay_vulkan` takes its two inputs only in one software format,
and otherwise fails with `Mismatching sw formats!` [measured].

### OpenCL

Mesa's rusticl drives the G610 only with `RUSTICL_ENABLE=panfrost` set
[measured, 2026-08-30, Mesa 26.1.6]. It then reports OpenCL 3.0 with OpenCL C 1.2, four
compute units at 800 MHz, a work-group size of 256, and image support. It has no double
precision, no `cl_arm_import_memory` and no dma-buf sharing extension. A decoded frame
therefore reaches OpenCL only through a copy.

`clpeak` measures the device as follows [measured, 2026-08-30, Mesa 26.1.6]:

| Quantity | Result |
| --- | ---: |
| Global memory bandwidth | 21.7 GB/s |
| Single precision, `float4` | 410 GFLOPS |
| Half precision, `half4` | 768 GFLOPS |
| 32-bit integer | 106 GIOPS |
| Host to device, `enqueueWriteBuffer` | 6.56 GB/s |
| Device to host, `enqueueReadBuffer` | 1.11 GB/s |
| Kernel launch latency | 162 us |

## RGA

The RGA has three cores, and the mainline dtsi describes all three, enabled
[source, 7.2.9]:

| Core | Node | Compatible | Power domain | IOMMU |
| --- | --- | --- | --- | --- |
| RGA3 core 0 | `rga@fdb60000` | `rockchip,rk3588-rga3` | `PD_RGA30` | `iommu@fdb60f00` |
| RGA3 core 1 | `rga@fdb70000` | `rockchip,rk3588-rga3` | `PD_RGA31` | `iommu@fdb70f00` |
| RGA2 | `rga@fdb80000` | `rockchip,rk3588-rga`, `rockchip,rk3288-rga` | `PD_VDPU` | None. RGA2 walks its own page table |

Two drivers can take them, and each needs a different device tree.

RGA2 reaches only the low 4 GiB of DRAM. Its page table's entries are 32-bit physical
addresses, and the in-tree driver writes only the low 32 bits of each page's address
[source]. That driver sets a 32-bit DMA mask, and allocates its own buffers in
`ZONE_DMA32`. An imported buffer above 4 GiB is then bounced through SWIOTLB, whose slots
are 2 KiB, at most 128 of them contiguous [source]. A scatter-gather segment larger than
256 KiB cannot be bounced [inferred].

The vendor driver keeps a job off RGA2 when any of its buffers lies above 4 GiB
[source][VENDOR]. Through that driver, a blit from a buffer above 4 GiB that reached RGA2
ended in a bus error, `INTR[0x701]` [reported][YSP]. On a board with more than 4 GiB, a job
only RGA2 can run needs its buffers below 4 GiB.

The in-tree `rockchip-rga` V4L2 driver matches `rockchip,rk3288-rga`,
`rockchip,rk3399-rga` and `rockchip,rk3588-rga3` [source, 7.2.9]. On the stock tree it
binds the RGA2 through its fallback compatible, and the first RGA3 core. The second RGA3
core logs `missing multi-core support, ignoring this instance`. arm64's defconfig builds
the driver as a module. Each bound core is its own V4L2 M2M node, and the RGA3 backend
lists 8-bit RGB and YUV formats only, with no `NV15` or `P010`.

The vendor's out-of-tree multicore driver presents one `/dev/rga` that schedules work
across all three cores, and it is what librga and the `*_rkrga` filters use. It needs two
device-tree changes per node:

- The compatibles become `rockchip,rga3_core0`, `rockchip,rga3_core1` and
  `rockchip,rga2_core0`, which tell the two RGA3 cores apart.
- Each `reg` window grows to 4 KiB, because the driver programs the core's IOMMU through
  it at offset `0xf00`. The mainline windows are `0x200` and `0x180`.

Under the vendor driver the reference unit reports `rga3_core0` and `rga3_core1` at hw
3.0.76831, and `rga2_core0` at hw 3.2.63318 [measured]. Load its `rga3` module by name,
because it carries no alias table [source]. It creates `/dev/rga` as `0600 root:root`, and
librga segfaults on a refused open [measured].

RGA3 covers BT.2020. The RK3576 has no RGA3, and its RGA2 does not cover BT.2020 (see
[the RK3576 RGA](../rk3576/video-reference.md#rga)).

Only RGA3 handles 10-bit, and a job reaches RGA3 only inside its window. The `*_rkrga`
filters route a job to RGA2 in any of these cases [source, `libavfilter/rkrga_common.c`]:

- Either axis scales outside 1/8 to 8x.
- The source is narrower than 68, or wider or taller than 8176.
- A format that only RGA2 handles is involved.

RGA2 reads no `P010` or `P210`, writes no `NV15` or `NV20`, and caps its output at
4096×4096. A 10-bit job outside RGA3's window therefore has no hardware route, and the
filter fails to configure with `ENOSYS`.

For a 90 or 270 degree rotation, the scale ratio pairs each output axis with the other
input axis. The `*_rkrga` filters must compute the ratio that way. Computed without the
swap, a rotated downscale that RGA3 can do is pushed onto RGA2 and refused.

As on the RK3576, the `scale_rkrga`, `vpp_rkrga` and `overlay_rkrga` filters need
`--enable-rkrga`, which ffmpeg's configure gates behind `--enable-rkmpp`. They accept the
V4L2 decoder's DRM PRIME frames only with an RKMPP device named for their output pool:

```sh
ffmpeg -init_hw_device rkmpp=rk -filter_hw_device rk ...
```

Without that device, the filters fail with `No RKMPP hardware context provided`. Device
derivation does not substitute: `hwmap=derive_device=rkmpp` fails with `ENOSYS`.

### The 10-bit stride

The RGA takes an `NV15` or `NV20` source only at a byte stride that is a multiple of 64
[measured, 2026-08-30, 7.2.2]. That holds through the vendor driver and the `*_rkrga`
filters. The
packed formats carry five bytes per four samples. A line of width `w` therefore takes
`w × 5 / 4` bytes. That product is a multiple of 64 only for a width that is a multiple of
256.

The rule is the software's, not the silicon's. For a 10-bit format, the vendor driver's
alignment check counts 10 bits per unit of the stride it is given. It requires a multiple of
128 bits [source][VENDOR]. Given a stride in bytes, the check demands a byte stride that
is a multiple of 64. A stride in bytes fits every row of the table below [inferred].

ffmpeg's `rkrga_common.c` requires the
same multiple before it routes a 10-bit job to RGA3 [source]. With the check counting 8
bits per byte, RGA3 converts `NV15` at a 1920-wide frame's tight 2400-byte stride
bit-exactly [reported][YSP].

Mainline rkvdec and the AV1 decoder size their `NV15` capture through
`v4l2_fill_pixfmt_mp()`, with no alignment beyond that product [source, 7.2.9]. A
1920-wide frame gets a 2400-byte stride, and the RGA refuses it:

| Width | `NV15` stride | Stride mod 64 | RGA blit |
| ---: | ---: | ---: | --- |
| 1280 | 1600 | 0 | Succeeds |
| 1600 | 2000 | 16 | Fails, -22 |
| 1920 | 2400 | 32 | Fails, -22 |
| 2560 | 3200 | 0 | Succeeds |
| 3200 | 4000 | 32 | Fails, -5 |
| 3840 | 4800 | 0 | Succeeds |

The failure is on the blit, not on configuration. Height padding and bit depth are not the
variable: 8-bit `NV12` at 1920×1080 blits. The RGA3's debugfs advertises
`byte_stride_align: 16`, the 128 bits the check requires.

The decoders themselves take every stride register as `bytesperline / 16`, so a padded
stride costs them nothing [source]. The fix rounds `bytesperline` up to 64 for the packed
formats, in rkvdec's `rkvdec_fill_decoded_pixfmt()` and the AV1 decoder's
`hantro_try_fmt()`. With it, 11 of 11 tested widths blit [measured, 2026-08-30 and
2026-08-31, 7.2.2]. A 1920×1080 to 1280×720 `scale_rkrga` run on the padded stride is byte-exact.

> **On a mainline decoder, 1080p 10-bit fails the vendor driver's check.** It has no RGA
> path. 4K survives only because 3840 happens to be a multiple of 256.

## Encode

Mainline drives one encoder on this SoC, the VEPU121 JPEG encoder at `/dev/video2`. H.264
and HEVC encode on VEPU580 runs through an out-of-tree driver. Mainline VEPU580 support is
in development upstream and unmerged.

### JPEG encode from ffmpeg

Upstream ffmpeg has no V4L2 JPEG M2M encoder wrapper. The only attempt is a 2018 patch
that never merged. A wrapper over `v4l2_m2m`, exposed as `mjpeg_v4l2m2m`, reaches the
VEPU121 encoder [measured, 2026-08-31]. The wrapper depends on two generic fixes to
`v4l2_m2m`. Because it rides `v4l2_m2m`, it needs no configure flag.

Quality matches the software encoder. Hardware at `-q:v 90` and ffmpeg's `mjpeg` at its
best `-q:v 2` land within 0.06 dB luma of each other. The quantization tables are
byte-identical to GStreamer `v4l2jpegenc` at the same control value. That cross-check
shows the wrapper drives the hardware the same way GStreamer does.

Three behaviors differ from software `mjpeg`:

- `-q:v` is inverted. For the hardware encoder it is the JPEG quality percentage (5..100,
  larger is better), and for `mjpeg` it is a quantizer. `-q:v 5` gives 5 percent
  quality, not high quality.
- The gain is CPU rather than speed. For 120 frames of 1080p at comparable output size,
  hardware takes 2.25 CPU-s and 1.93 s of wall time, and software 5.15 CPU-s and 0.81 s.
  That is ~2.3x less CPU and ~2.4x slower in wall-clock time. Software `mjpeg` threads
  across eight cores, and the V4L2 path submits serially. The trade suits trickplay and
  snapshots competing with a transcode, and not a job where wall-clock throughput matters.
- A JPEG decodes as full-range `yuvj420p`. Compared against a limited-range `nv12` source,
  luma is rescaled and PSNR sits near 26 dB whatever the quality. Software JPEG measured
  the same way gives the same 26 dB, so run that control before trusting a luma number.

### The JPEG encoder's V4L2 contract

A V4L2 client of this encoder meets seven driver behaviors, each confirmed on the
reference unit [measured, 2026-08-31]:

- Set the CAPTURE (coded) format before the OUTPUT (raw) one. `hantro_set_fmt_cap()`
  calls `hantro_reset_raw_fmt()`, so the reverse order leaves the hardware on its default
  `YM12` with no error, and the chroma comes out wrong.
- `TRY_FMT` substitutes and still returns 0. A request for `NV12` on the multiplanar queue
  returns `YM12`, so compare the fourcc that comes back.
- The coded format enumerates as `V4L2_PIX_FMT_JPEG` only, never `V4L2_PIX_FMT_MJPEG`.
- `V4L2_CID_JPEG_COMPRESSION_QUALITY` belongs to the JPEG control class. Set under the
  MPEG class, it fails with `EINVAL`.
- The OUTPUT crop rectangle sets the JPEG's dimensions. Without it, a 1080-line source
  writes a 1088-line JPEG.
- A requested CAPTURE size of 0 lets the driver size the buffer, at the header plus the
  full raw picture. A 1080p JPEG at quality 95 reaches 480 KB.
- The driver sets no `V4L2_BUF_FLAG_KEYFRAME`, though every JPEG is an intra picture.

At matched quality the files are 7-30% larger than software `mjpeg`'s, because the
driver uses fixed Annex K tables and Huffman codes [measured]. Quality 50 gives about
40.6 dB at 116 KB for a 1080p frame.

### The JPEG encoder's frame-size cap

The mainline driver advertises `Stepwise 96x32 - 8192x8192 with step 16/16`. Any
dimension of exactly 8192 produces a ~3 KB file whose SOF reads 0x0 [measured]. JPEG
encode at 8K works: 7680×4320 gives a valid 894 KB file, and so does 8176×8176 at 1.83 MB.

The mechanism covers both axes. `rockchip,rk3588-vepu121` matches `rk3568_vepu_variant`,
whose `HANTRO_MODE_JPEG_ENC` op is `rockchip_vpu2_jpeg_enc_run()`. Dimensions therefore go
out through `rockchip_vpu2_regs.h` as macroblock counts in nine-bit masked fields:
`VEPU_REG_MB_WIDTH(x) (((x) & 0x1ff) << 8)` and the matching height. A dimension of 8192
pixels is 512 macroblocks, and `512 & 0x1ff` is 0. The header then describes a frame of no
size, in either axis.

The H1 pair of fields is nine bits each as well, so the same limit holds for the variants
on that path. GStreamer `v4l2jpegenc` produces the identical 3 KB file from the same input,
which places the defect in the driver rather than in userspace.

Capping the shared `V4L2_PIX_FMT_JPEG` entry in `rockchip_vpu_enc_fmts[]` at 8176×8176
(511 macroblocks) fixes it for every Rockchip JPEG encoder variant. With that cap, the
driver reports `Stepwise 96x32 - 8176x8176`, and 8176×4592 encodes correctly [measured,
2026-09-02].

> **With the cap at 8176, a request for 8192 clamps, and a wrapper that only warns writes a
> corrupt picture.** Measured with a wrapper whose visible-rectangle step logs
> `AV_LOG_WARNING` and returns `void`, ffmpeg exits 0. It writes a JPEG whose SOF reads
> `8176x4592`, at 7.81 dB luma against the reference, where the 8176 encode measures the
> control value of 26.07 [measured].

The driver also changes rectangles benignly: it applies `round_up(rect->width, 4)`, so 854
becomes 856 and the picture is intact. A V4L2 encoder wrapper must therefore check the
direction of the change. A rectangle the driver grew is a warning, and one it shrank must
fail the encoder open. Against a wrapper without that check, clamp the input to 8176 in
each axis.

### JPEG decode

JPEG decode is a separate block. Sascha Hauer's standalone `rkjpegd` driver (RK3588 and
RK356x only, stateful M2M, JPEG in and NV12 out) was first posted in August 2026. It is
at v4 (2026-09-18) and unmerged. When it lands, an ffmpeg decoder wrapper is a small
mirror of the encoder wrapper [expected]. Both directions depend on one `v4l2_m2m` fix,
because `v4l2_get_coded_format()` serves both and hits the same MJPEG-versus-JPEG fourcc
dead end.

### VEPU580 encode

VEPU580 H.264 and HEVC encode runs through the out-of-tree `mpp_service` VEPU580 port. The
port presents `/dev/mpp_service`, and ffmpeg's `hevc_rkmpp` and `h264_rkmpp` encoders
drive it. The matching `*_rkmpp` decoders build, but never open on a mainline kernel. MPP
finds no decode client there, because `rkvdec` is a V4L2 stateless driver rather than an
MPP service.

`-hwaccel rkmpp` therefore decodes in software, with no message. It costs 49.85 CPU-s
against 50.19 for no hwaccel at all, on the same 900 frames [measured, 2026-08-11, 7.1.7].

The mainline dtsi carries no VEPU580 node [source, 7.2.9]. The port describes two cores
[source]:

| Core | Registers | Interrupt | IOMMU | Power domain |
| --- | --- | --- | --- | --- |
| 0 | `fdbd0000`, `0x6000` bytes | `GIC_SPI 101` | `fdbdf000`, two banks, `GIC_SPI 99` and `100` | `PD_VENC0` |
| 1 | `fdbe0000`, `0x6000` bytes | `GIC_SPI 104` | `fdbef000`, two banks, `GIC_SPI 102` and `103` | `PD_VENC1` |

The port's tree requests 500 MHz for each core's AXI clock and 800 MHz for its core clock.
The core clock's parents are GPLL, CPLL, AUPLL and NPLL [source]. On a Rock 5B it reads
786,431,998 Hz, the audio PLL's rate, rather than 800 MHz [reported][YSP]. On an RK1 module
both cores' clocks read 786,431,991 Hz from the audio PLL, and their AXI clocks 500 MHz
[measured, 2026-10-23, 7.2.9][CLK]. The core clock
then follows any change to the audio PLL [inferred].

The encoder's output buffer must be physically contiguous, because the driver programs
only the buffer's first scatter-gather segment [measured]. On mainline that buffer comes
from the CMA heap, `/dev/dma_heap/default_cma_region`, since the vendor kernel's heap
names do not exist there. arm64's defconfig reserves 32 MiB of CMA [source, 7.2.9]. The
measurements on this page ran with 384 MiB, set by `cma=384M`.

The CMA heap's buffers are cached, so MPP must sync each one through dma-buf around the
hardware's access. An allocator that marks the heap uncached makes MPP skip that sync.
The stream then reaches the CPU with the slice start codes at each IDR overwritten
[measured, 2026-08-11, 7.1.7]:

| Encode of 90 frames | Heap marked uncached | Heap marked cached |
| --- | --- | --- |
| `h264_rkmpp` | 24/90 decode back | 90/90 |
| `hevc_rkmpp` | 11/90 decode back | 90/90 |

> **Decode every encoder test's output.** The broken streams above encode with no error,
> and only decoding them shows the damage.

VEPU580 encodes 8-bit only. MPP maps every 10-bit input format to `VEPU5xx_FMT_BUTT`, its
invalid-format sentinel, so a 10-bit source is converted to 8-bit before the encoder
[source]. MPP releases before commit `264553f9` (2026-04-22) write the VEPU580 bitstream's
top address as the buffer size rather than size minus one. The hardware then page-faults
as it wraps its output [source].

### Upstream encoder work

Collabora (Detlev Casanova) is writing a mainline VEPU580 driver, unmerged: a thin Rust
kernel module plus a Mesa userspace driver over Vulkan Video [LORE]. It had P-frames working by
July 2026. A separate RFC covers the RK3576's VEPU510 as a V4L2 stateful encoder.
Neither takes the `mpp_service` shape, and Nicolas Dufresne has stated that shape is not
the community direction [LORE].

### MPP probe messages

An ffmpeg built with MPP support prints eight lines of MPP probe output to stderr on every
invocation, even for `-version`: `mpp_platform: client N driver is not ready!`. MPP is
probing for vendor service clients that a mainline kernel does not have. The lines are
benign, and they land in any caller's stderr.

## HDMI receiver

The full RK3588 has an HDMI 2.0 receiver, a Synopsys DesignWare block, driven by mainline
`snps_hdmirx` [source, 7.2.8]. `rk3588-extra.dtsi` describes it as
`hdmi_receiver@fdee0000` (`rockchip,rk3588-hdmirx-ctrler`, in `PD_VO1`), with a 160 MiB
CMA pool of its own, at `status = "disabled"`. `rk3588.dtsi` includes that file and
`rk3588s.dtsi` does not, so the RK3588S has no receiver.

The receiver's TMDS lanes are dedicated pins. Pinctrl muxes only the CEC, hot-plug and DDC
lines (`hdmim0_rx_*` to `hdmim2_rx_*`), so a board either routes the receiver or does not.
The 7.2.8 tree enables it on these boards:

- Rock 5B, 5B+, 5T and 5 ITX
- Orange Pi 5 Plus
- NanoPC-T6
- CM3588 NAS
- EVB1

The Turing RK1 is a full RK3588 and inherits the node, but its device tree enables nothing.
The RK1 module connector carries no HDMI-input pins, so no carrier board reaches the
receiver. Turing's I/O specification lists HDMI out, DP, DSI and MIPI-CSI in. No figure on
this page therefore comes from the receiver.

The driver offers the following [source, 7.2.8]:

| Property | Behavior |
| --- | --- |
| Timings | 640×480 to 4096×2160 at a 20-600 MHz pixel clock (4K60 fits), CEA-861, progressive and interlaced |
| Capture format | Set by what the source sends, not by the consumer: RGB gives `BGR24`, 4:2:2 `NV16`, 4:4:4 `NV24`, 4:2:0 `NV12`. All are 8-bit. A deep-color link's depth only feeds the pixel-clock arithmetic |
| Buffers | `VB2_MMAP` and `VB2_DMABUF`, so a frame can go to RGA or the encoder without a copy |
| Line pitch | Every plane aligned to 64 bytes |
| EDID | Must be written (`v4l2-ctl --set-edid`) before a source will send, unless `VIDEO_SYNOPSYS_HDMIRX_LOAD_DEFAULT_EDID` preloads one with modes up to 4K30 |
| InfoFrames | AVI only, through debugfs `infoframes/avi` and the RGB-range and content-type controls. No HDR InfoFrame |
| CEC | Yes, in the same module |
| Audio | None in 7.2. An audio-capture series was at v4 on 2026-07-21, unmerged |
| HDCP | None. The driver only overrides the HDCP 2 switch, so the source must have HDCP off |

Two traps wait for a consumer of the receiver:

- Padded lines. Where a line is not a multiple of 64 bytes (720-wide `BGR24`, for one),
  the buffer is larger than a tightly packed frame. Stock ffmpeg's `v4l2` input empties
  every such buffer with `Dequeued v4l2 buffer contains N bytes, but M were expected`
  [source, `libavdevice/v4l2.c`]. The 1280-, 1920- and 3840-wide modes are unpadded in
  every format.
- The `NV24` chroma pitch. The driver lays the chroma plane at `ALIGN(2 * width, 64)` but
  reports only the luma pitch, `ALIGN(width, 64)`. V4L2 defines a single-plane `NV24`
  chroma pitch as twice the luma one, so the two disagree whenever `width % 64` falls in
  1..32. Common widths that hit it are 720, 800, 1366, 1440 and 1680. This is read from
  the source and not observed on hardware.

The receiver writes frames to memory and does no scaling, conversion or encoding. A
hardware-decoded stream reaches RGA and the encoders as the same kind of buffer. The
kernel's `vivid` driver (`CONFIG_VIDEO_VIVID`, with CEC) emulates an HDMI input over the
same V4L2 interface, so a consumer can be developed without the hardware.

## DDR traffic counters

Mainline drives the DDR monitor [source, 7.2.8]. `rk3588-base.dtsi` describes
`dfi@fe060000` (`rockchip,rk3588-dfi`), and `rockchip-dfi` matches it. The driver's
kconfig symbol is `CONFIG_DEVFREQ_EVENT_ROCKCHIP_DFI`. Besides its devfreq event, the driver registers a
system-wide perf PMU, `rockchip_ddr`. It counts `read-bytes`, `write-bytes` and `bytes`,
per channel (`read-bytes0` to `3`) and in total, scaled to MB.

The DRAM traffic of a decode, a filter chain or a transcode is therefore a measurement
(`perf stat -a -e rockchip_ddr/bytes/`) rather than an argument. No figure on this page
comes from it. Its counts agree with a known load's traffic, and its four channels split that
traffic evenly (see [DRAM bandwidth](system-reference.md#dram-bandwidth)). The RK3576 has no mainline DFI. Its DDRMON is a newer version (0x40) that
the driver does not know, and its support is posted upstream, unmerged.

## Related pages

- [RK3576 video reference](../rk3576/video-reference.md): the same subsystems on the other
  SoC, where the decoder is VDPU383, there is no RGA3, and encode is undriven.
- [RK3576 H.264 power-up erratum](../rk3576/h264-power-up-erratum.md): the RK3588 does
  not show it [measured, 2026-08-12]. Of 66 decode sessions, none corrupts while both
  rkvdec power domains cycle off. Nothing from it applies here.

## Sources

Citation key:

- `[LORE]`: the linux-media thread on the RK3576 VEPU510 RFC, with Nicolas Dufresne's
  reply on the encoder direction (2026-07-22).
- `[MESA]`: Mesa's `src/gallium/frontends/dri/dri_helpers.c` (the `DRM_FORMAT_NV15`
  mapping and `alt_pipe_format()`) and `src/panfrost/lib/pan_format.c` (the
  `R10_G10B10_420_UNORM` entry for architecture 7 onward). Both are present on the 25.0
  and 26.1 branches and on main at `9965f2d5` (2026-09-28), and absent on 24.2.
- `[VENDOR]`: the vendor kernel, rockchip-linux/kernel at commit `b4ef083dc`, branch
  `develop-6.1`, Linux 6.1.141. `arch/arm64/boot/dts/rockchip/rk3588s.dtsi` (the `iep`,
  `iep_mmu`, `rkvdec0`, `rkvdec1` and `av1d` nodes), `drivers/video/rockchip/mpp/`
  (`mpp_iep2.c`, `rockchip_iep2_regs.h`, `mpp_vdpp.c` and `mpp_rkvdec2.c`), and the vendor
  RGA driver's `rga_policy.c` (`rga_check_align`) and `rga_job.c`.
- `[YSP]`: Yi Ding, rock-5b-ysp, https://github.com/yisding/rock-5b-ysp, at commit
  `69a3abbf`, a Rock 5B project whose documentation is licensed CC-BY-SA-4.0. Three of its
  findings: 2026-07-24 on 10-bit RGA strides and the RGA2 bus error, 2026-07-28 on decoder
  timeouts, and 2026-08-16 on the encoder's clock. Each was measured under a vendor-derived
  6.18 kernel. The facts are restated here, not quoted.

Captures from a Turing RK1 module under Linux 7.2.9 with BL31 v1.51, kept with its evidence
outside this repository:

- `[CLK]`: `/sys/kernel/debug/clk/clk_summary` and the regulator summary, 2026-10-23, with
  the media blocks idle.
- `[GPUQUERY]`: a program issuing panthor's `DRM_IOCTL_PANTHOR_DEV_QUERY` for the GPU,
  command stream interface and timestamp information on `/dev/dri/renderD128`, 2026-10-23.
- `[BOOTLOG]`: the kernel log of the boot, 2026-10-05, read from the journal.

External sources named in the text:

- Benjamin Gaignard (Collabora), `[PATCH v3 00/13] AV1 stateless decoder for RK3588`,
  January 2023, and its cover letter's conformance figures
- Collabora's unmerged rkvdec multicore series
- Sascha Hauer's `rkjpegd` driver, v4 (2026-09-18), and his unmerged `verisilicon`
  run-failure cleanup
- Mesa merge requests !42353 and !42451
- Turing's RK1 I/O specification
- AOM's `av1-test-data` vectors and their reference `.md5` files
- Fluster's `compare_yuv_files()`
