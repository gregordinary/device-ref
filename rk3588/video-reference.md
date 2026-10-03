# RK3588 video, 2D and GPU reference

This page covers the RK3588's media blocks under a mainline kernel: decode, encode, RGA,
the GPU, the HDMI receiver and the DDR traffic counters. It states which of them mainline
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
| `/dev/video3` | `hantro-vpu` | `rockchip,rk3588-av1-vpu-dec` | `AV1F` | `VT12` (4×4 tiled), `NV12` |

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

### The rkvdec cores

`rkvdec` runs two cores behind one node. With Collabora's unmerged rkvdec multicore series
applied, `fdc38000` and `fdc40000` both register and bind to one master. That master
presents a single `/dev/video0`, so two V4L2 decoder nodes mean the component binding
failed. With both cores bound, four concurrent 4K H.264 streams decode at realtime, and
tiled HEVC conformance stays bit-exact [measured].

The series splits the driver around `struct rkvdec_core`. A change touching that
structure applies only to a kernel carrying the series. An rkvdec fix therefore does not
move unchanged between a kernel with the series and one without it. The case arises
between an RK3588 kernel carrying the series and an RK3576 kernel without it.

### Blocks without a node

Two media blocks in the silicon present no node under mainline: a dedicated JPEG
decoder, and the IEP, the vendor kernels' deinterlacer. The mainline dtsi
carries clock, power-domain and QoS plumbing for both (`HCLK_JPEG_DECODER`,
`qos_jpeg_dec`, `qos_iep`), but no device node or driver [source].

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

- IEP2 takes NV12 8-bit in, writes NV12 8-bit out and is capped at 1080, so it cannot
  process 10-bit content.
- With no IEP driver, `bwdif` deinterlaces on the CPU at 0.93 CPU-s per second of 1080i
  content [measured]. That is 2.84 of 8 cores at realtime, about 1.35x what hardware
  decode saves, while 576i costs ~0.2 cores.

`bwdif_vulkan` produces correct fields but costs the same CPU at a third of the
throughput, and `yadif` is strictly worse. The cost case for an IEP2 port is weak. A port
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

### The RCB SRAM IOVA collision

The rkvdec driver maps its RCB with a hand-rolled `iommu_map()` in `rkvdec_allocate_rcb()`.
That mapping makes 8K 10-bit HEVC decode fail deterministically with `dma alloc of size
49766400 failed`. The line before it is always
`iova: 0x…81178000 already mapped to 0x…ff079000 cannot remap`, byte-identical across
hours and workloads. The failure reads as a 10-bit or an allocation-size ceiling, and it
is neither. Each link of the chain is measured:

1. `rk3588-base.dtsi` gives `system_sram2` two pools, and `vdec1_sram` sits at physical
   address `0xff079000`, the stale physical address in the message.
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

When changing this code, confirm that the SRAM path still serves the RCB. A fallback to
`dma_alloc_coherent` also makes the collision vanish, for the wrong reason and at a
performance cost. To trace an RCB allocation, decode something wider than anything
decoded since boot. The RCB only ever grows, and the driver frees it only on remove.

## Conformance

Each score is read against a software decode on the same unit, and the failures software
shares are subtracted. What remains is attributable to the driver [measured, 2026-08-31
and 2026-09-02, 7.2.2].

| Suite | Node | Score | Residual |
| --- | --- | --- | --- |
| `JCT-VC-HEVC_V1` | rkvdec | 146/147 via GStreamer, 141/147 via ffmpeg | The one GStreamer miss, `TSUNEQBD_A_MAIN10` (unequal luma and chroma depth), fails in software too. The ffmpeg gap is in userspace: the `v4l2request` hwaccel must set the HEVC SPS-level RPS controls, and one that does closes it |
| `JVT-AVC_V1` | rkvdec | 128/135 raw via ffmpeg. Control-subtracted, the failures are upstream's 129/135 set | The six known upstream failures: FMO x3, SP x2, `CVFC1` crop offsets |
| `JVT-AVC_V1` | VDPU2 | 129/135 | The same failure set |
| `JVT-FR-EXT` | rkvdec | 65/69 | 4:2:2 8-bit and 10-bit (`NV16`, `NV20`) decode correctly. Fails: monochrome x2 (no capture format), MBAFF 4:2:2 x2 |
| `VP8-TEST-VECTORS` | VDPU2 | 59/61, the software score exactly | None attributable |
| `AV1-TEST-VECTORS` | VPU981 | 201/242 | See [AV1 decode](#av1-decode) |
| `MPEG2_VIDEO-MAIN` | VDPU2 | 20/43 raw | Three failure classes, one real defect (see [MPEG-2 on VDPU2](#mpeg-2-on-vdpu2)) |

### Attributing a failure

Running both userspaces settles attribution. Fluster's `GStreamer-*-V4L2SL` decoders work
once `gstreamer1.0-plugins-good` and `gstreamer1.0-plugins-bad` are installed. A vector
that fails under ffmpeg's `v4l2request` but passes under GStreamer is a userspace gap. The
HEVC row, 146 against 141, is that case. A vector that fails under both, while software
passes, is the driver's. A 43-vector suite takes about a minute, so this is the default
second step whenever a score looks wrong.

The node a vector lands on is not obvious. ffmpeg's `v4l2request` picks a node by coded
format, so `VP8F` and `MG2S` go to VDPU2 automatically. Only H.264 exists on both nodes.
ffmpeg takes the first match (rkvdec) unless given `-hwaccel_device /dev/media1`.

GStreamer names the two H.264 decoders as separate elements: `v4l2slh264dec` is VDPU2, and
`v4l2slvideo0h264dec` is rkvdec. VDPU2 accepts `S264` and `MG2S` from 48×48 to 1920×1088 in
steps of 16, and `VP8F` from 48×48 to 3840×2160.

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

The RGA filters take the same frames in hardware at any width once the AV1 decoder driver
pads its stride the way `rkvdec` does. With both changes, 1280-, 1920- and 3840-wide AV1
10-bit clips blit and download bit-exact against software decode [measured, 2026-08-31,
7.2.2]. The 65 vectors are not re-scored against AOM's MD5s with that readback. The
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
| RGA | Yes. It converts `NV15` to `P010` and `NV20` to `P210`, the one RGA capability nothing in mainline replaces |
| `hwdownload` and swscale | Yes, in an ffmpeg whose hwcontext tags the buffer as a real `NV15` frame. `format=yuv420p10le` then unpacks it on the CPU, which is what feeds a GPU tone mapper |
| Vulkan and PanVk | No. Vulkan has no 10-bit packed 4:2:0 format |
| Mesa GL and EGL, mpv | No, per nyanmisaka's `ffmpeg-rockchip` wiki [reported] |
| ffmpeg Vulkan hwcontext | No. Its DRM import table carries no multi-planar YUV fourcc |

Two consequences follow:

- 10-bit playback through a KMS plane needs no conversion, because VOP2 scans `NV15` out
  directly.
- 10-bit transcode converts on RGA to stay in hardware, or downloads and unpacks on the
  CPU. That serial unpack is the cost a tone-mapping chain pays.

A result from a player such as Kodi or mpv therefore does not transfer to a transcode
pipeline, because the consumers differ.

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

## RGA

The RGA has three cores, reached through the vendor's out-of-tree multicore driver at
`/dev/rga` [measured]: `rga3_core0` and `rga3_core1` (hw 3.0.76831), and `rga2_core0`
(hw 3.2.63318). The in-tree `rockchip-rga` V4L2 driver binds only `rockchip,rk3288-rga`
and `rockchip,rk3399-rga` [source], so it does not drive this SoC's RGA.

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

JPEG decode is a separate block. Sascha Hauer's standalone `rkjpegd` driver (RK3588 and
RK356x only, stateful M2M, JPEG in and NV12 out) was first posted in August 2026. It is
at v4 (2026-09-18) and unmerged. When it lands, an ffmpeg decoder wrapper is a small
mirror of the encoder wrapper [expected]. Both directions depend on one `v4l2_m2m` fix,
because `v4l2_get_coded_format()` serves both and hits the same MJPEG-versus-JPEG fourcc
dead end.

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

### VEPU580 encode

VEPU580 H.264 and HEVC encode runs through the out-of-tree `mpp_service` VEPU580 port. The
port presents `/dev/mpp_service`, and ffmpeg's `hevc_rkmpp` and `h264_rkmpp` encoders
drive it. The matching `*_rkmpp` decoders build, but never open on a mainline kernel. MPP
finds no decode client there, because `rkvdec` is a V4L2 stateless driver rather than an
MPP service.

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
comes from it. The RK3576 has no mainline DFI. Its DDRMON is a newer version (0x40) that
the driver does not know, and its support is posted upstream, unmerged.

## Related pages

- [RK3576 video reference](../rk3576/video-reference.md): the same subsystems on the other
  SoC, where the decoder is VDPU383, there is no RGA3, and encode is undriven.
- [RK3576 H.264 power-up erratum](../rk3576/h264-power-up-erratum.md): the RK3588 does
  not show it [measured], so nothing from it applies here.

## Sources

Citation key:

- `[LORE]`: the linux-media thread on the RK3576 VEPU510 RFC, with Nicolas Dufresne's
  reply on the encoder direction (2026-07-22).

External sources named in the text:

- Benjamin Gaignard (Collabora), `[PATCH v3 00/13] AV1 stateless decoder for RK3588`,
  January 2023, and its cover letter's conformance figures
- Collabora's unmerged rkvdec multicore series
- Sascha Hauer's `rkjpegd` driver, v4 (2026-09-18), and his unmerged `verisilicon`
  run-failure cleanup
- Mesa merge requests !42353 and !42451
- The nyanmisaka `ffmpeg-rockchip` wiki
- Turing's RK1 I/O specification
- AOM's `av1-test-data` vectors and their reference `.md5` files
- Fluster's `compare_yuv_files()`
