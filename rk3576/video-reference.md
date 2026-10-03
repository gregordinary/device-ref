# RK3576 video, 2D and GPU reference

This reference covers the RK3576's media blocks at SoC level: the video decoder, the RGA
2D accelerator, the Mali-G52 GPU and the video encoder. It states what each block exposes
to a mainline kernel, and which of its ceilings are structural rather than undriven.
Everything here follows from `rk3576.dtsi` and the shared drivers, not from any one board.
The measurements come from one board, the H96 MAX M9, and a measured claim stated for the
SoC rests on that board. Per-board status is in the
[H96 MAX M9 enablement status](h96-max-m9/mainline-enablement-status.md), and the VOP2 and
HDMI side is in the [RK3576 display reference](display-reference.md).

Mainline drives decode and does not drive encode. The 10-bit ceiling on this part is
structural, which it is not on the RK3588.

## Media blocks

The register bases come from the factory device tree [DTB]:

| Block | Factory node | Driver |
|---|---|---|
| Video decoder (VDPU383) | `rkvdec@27b00000` | `rkvdec`, in tree. H.264 and HEVC only |
| Video encoder (VEPU510), cores 0 and 1 | `rkvenc-core@27a00000`, `rkvenc-core@27a10000` | No merged driver. See [Encode](#encode) |
| JPEG decoder and encoder | `jpegd@27910000`, `jpege@27910800` | No driver reaches it |
| RGA2, cores 0 and 1 | `rga@27920000`, `rga@27930000` | The vendor's out-of-tree multicore driver. See [RGA](#rga) |
| GPU (Mali-G52 MC3) | `gpu@27800000` | `panfrost`, in tree. See [GPU](#gpu) |
| NPU | `npu@27700000` | See the [RK3576 NPU reference](npu-reference.md) |

The TRM documents the JPEG block. The standalone `rkjpegd` decoder driver posted upstream
does not reach it either. Its v4 (Sascha Hauer, 2026-09-18, unmerged) matches
`rockchip,rk3568-jpegd` and `rockchip,rk3588-jpegd` only, with DT nodes for those two
SoCs. Mainline also leaves VDPP undriven, and [VDPP](#vdpp) explains why it is no
10-bit consumer even with a driver.

## The decoder

The RK3576 has one codec node, served by one driver for two codecs [source]. The node
binds as shown on the reference unit [measured, 7.1.3]:

| Node | Driver | Compatible | OUTPUT | CAPTURE |
|---|---|---|---|---|
| `/dev/video0` | `rockchip_vdec` (`rkvdec`, `vdpu383_variant`) | `rockchip,rk3576-vdec` at `27b00100.video-codec` | `S265`, `S264` (stateless slice data) | `NV12`, and `NV15` for 10-bit |

`/dev/media0` accompanies the node, which sits in IOMMU group 0 with 384 MiB of CMA
reserved. `CONFIG_VIDEO_ROCKCHIP_VDEC=m` builds the driver as the `rockchip_vdec` module,
which cannot be unloaded on this SoC: see
[rkvdec cannot be unloaded](corrections-and-gotchas.md#rkvdec-cannot-be-unloaded).
Mainline drives stateless H.264 and HEVC decode on the VDPU383 from Linux 7.0 [source].
The VDPU383 silicon also decodes VP9 and AV1, and mainline drives neither on this SoC.

### AV1

AV1 is unavailable on this SoC, for three reasons [source]:

- `rk3576.dtsi` has exactly one codec node.
- `rkvdec`'s `vdpu383_coded_fmts[]` lists `HEVC_SLICE` and `H264_SLICE` only. The
  driver's single other "av1" token is the `av1_fgs_en` register-field name, a silicon
  capability that mainline does not program.
- The hantro/verisilicon driver matches no RK3576 compatible, so
  `CONFIG_VIDEO_HANTRO=m` builds a driver that binds nothing on this SoC.

A VDPU383 AV1 backend landing upstream changes this. On arrival, check its tile loops
against the bounds the V4L2 core enforces in `validate_av1_tile_info()`: at most 64
columns, 64 rows and 512 tiles. Check also that the driver itself enforces any smaller
limit its scratch buffers hold, the way VPU981 checks its 128 on the RK3588.

### The H.264 power-up erratum

Hardware H.264 decode without a warm-up is wrong on roughly one power-up in three. A
work-around decodes a canned frame at every runtime resume, before any other decode of
that power cycle. It measures 0 of 400 power-ups corrupt against an interleaved control's
128 of 400, and a 200-power-up soak stays clean [measured]. HEVC is unaffected
throughout, and serves as the free control for any decoder experiment.

The pixel-level signature, the base rate, the work-around and the avenues measured and
refuted are in [the H.264 power-up erratum](h264-power-up-erratum.md). Read it before
changing `rkvdec` on this SoC or proposing a reset or a udev work-around. Read it also
before citing an H.264 decode result taken on this part.

### The RCB SRAM mapping

Mainline `rkvdec-rcb.c` invents the IOVA for the decoder's row-column-buffer (RCB) SRAM
window, and truncates it to 32 bits [source]. It reserves nothing with the DMA allocator.
Once concurrent 4K decodes walk the allocator past 2 GiB, that IOVA collides with a real
mapping:

```
iova: 0x80780000 already mapped to 0x3ff88000 cannot remap
```

`0x3ff88000` is exactly `rkvdec-sram@0` inside `sram@3ff88000` in `rk3576.dtsi`. The
stream then dies with `Cannot allocate memory`.

Mapping the window through `dma_map_resource()` instead removes the collision [measured,
7.2.3]. The test ran 600 stream starts over 40 ramp cycles of one to five concurrent 4K
HEVC decodes. Each cycle stepped the count up immediately after the smaller batch exited,
which is the condition that produces the collision. It saw zero collisions and zero new
kernel log lines. Without the change the collision rate is 2 in 145, which that result
excludes at p = 2.4e-4.

The RK3588 form of the same change does not apply to this SoC. It sits on the rkvdec
multicore split, whose signature is `rkvdec_free_rcb(rkvdec, core)`. The RK3576 is
single-core and keeps mainline's `rkvdec_free_rcb(ctx)`, so the change is written against
that signature. The RK3588 side is in the
[RK3588 video reference](../rk3588/video-reference.md).

### Control validation

The V4L2 core bounds the HEVC tile count from Linux 7.2.7 [source, 7.2.8
`v4l2-ctrls-core.c`]. With tiles enabled, `std_validate_compound()` rejects a PPS whose
`num_tile_columns_minus1` or `num_tile_rows_minus1` indexes past `column_width_minus1[]`
or `row_height_minus1[]`. The check covers every stateless HEVC driver, so the VDPU383
path needs no driver change for it.

The HEVC SPS reference-picture-set tables still need a bounds check in `rkvdec` itself
[source]. The VDPU383 takes those tables through `EXT_SPS_ST_RPS` and `EXT_SPS_LT_RPS`. A
change to `rkvdec-vdpu383-hevc.c` adds the check, and the same change serves the RK3588.

### Decode throughput

Decode runs end to end through GStreamer [measured, 2026-07-30]. Over 300 frames, 1080p
H.264 decodes at ~190 fps and 1080p HEVC at ~183 fps. No frames drop, and the output is
NV12. The decoders are `v4l2slh264dec` and `v4l2slh265dec`, which the `v4l2codecs`
plugin registers against `rkvdec-proc`.

Through ffmpeg's `-hwaccel v4l2request`, each clip's MD5 equals an independent x86
software decode [measured, 2026-08-26, 7.2.0]. The times below cover the whole clip with
`drm_prime` output, against a software decode on the same board:

| Clip | Hardware | Software |
|---|---|---|
| H.264 1080p, 300 frames | 1.23 s (~244 fps) | 3.50 s |
| HEVC 1080p, 300 frames | 0.67 s (~450 fps) | 3.35 s |
| H.264 4K, 120 frames | 2.18 s (~55 fps) | 6.16 s |
| HEVC 4K, 120 frames | 1.23 s (~97 fps) | 5.49 s |

The 4K H.264 rate of 55 fps shows the [read caches](#the-read-caches) armed. With them
unarmed, 4K H.264 decodes at 25 fps, below software.

## The read caches

Mainline's VDPU383 backends leave the decoder's three reference read caches unarmed
[source]. Probe maps only the `function` and `link` register regions, and not the `cache`
window that `rk3576.dtsi` describes. Arming the caches is worth a median 2.6x across eight
real-content clips, with a range of 2.44x-3.15x [measured, 7.1.7]:

| Stream | Caches unarmed | Caches armed | Software |
|---|---|---|---|
| H.264 1080p | 98 fps | 257 fps | 160 fps |
| H.264 4K | 25 fps | 69 fps | 40 fps |

With the caches unarmed, H.264 decodes slower than HEVC. That gap comes from the missing
cache configuration and not from an H.264 defect: H.264 has the least headroom to hide it.

The register programming comes from the vendor kernel: `rockchip-linux/kernel` branch
`develop-6.1`, `mpp_rkvdec2.c`, in `rkvdec_vdpu383_run` and `rkvdec2_clear_cache`
[source]. The vendor quotes its offsets against register base `0x27b00100`, which is
mainline's `function` base. The offsets therefore map as follows:

| Vendor offset | Mainline offset |
|---|---|
| `0x510` | `cache` + `0x10` |
| `0x518` | `cache` + `0x18` |
| `0x51c` | `cache` + `0x1c` |

The public `SympleNZ/rkvdec-vdpu383-h264-hevc` repository carries the right diagnosis and
no usable patch [source]. Its patch is prose with elided paths and non-numeric hunk
headers, and it does not apply. Its speedup claim reads 7x, ~5x and ~4x in three places
in the one repository.

The change measured here exposes the cache enable as a runtime-writable `read_cache`
parameter (mode 0644). Flipping it between runs is the cheap discriminator before blaming
the caches for a decode fault.

## Capture formats

`rkvdec.c` declares exactly four capture formats, `NV12`, `NV15`, `NV16` and `NV20`, and
no `P010` [source]. The list belongs to the shared driver, so the RK3588 behaves the same
way.

HEVC Main 10 reports `Using CAPTURE buffer format NV15`, and HEVC Main reports `NV12`.
Both decode with zero errors [measured, 2026-08-09 on 7.1.7, and 2026-09-13 on 7.2.3]. Hardware output is bit-exact
against the same ffmpeg's software decoder, by framemd5 per frame:

| Stream | 7.2.3, 300 frames | 7.2.8, 120 frames (2026-10-02) |
|---|---|---|
| H.264 8-bit, 1080p | Baseline and High, bit-exact | Bit-exact |
| H.264 High, 4K | Bit-exact | Bit-exact |
| H.264 High 10, 1080p | Bit-exact | Bit-exact |
| HEVC Main, 1080p | Bit-exact | Bit-exact |
| HEVC Main, 4K | Bit-exact | Bit-exact |
| HEVC Main 10, 1080p | Bit-exact | Not in the run |
| HEVC Main 10, 4K | Bit-exact | Bit-exact, captured as `NV15` |

> **Do not read the capture format from an idle node.** `v4l2-ctl --list-formats` on
> `/dev/video0` reports `NV12` only, because the driver filters the capture list by the
> current stream's bit depth. Read `rkvdec.c`, or decode a real 10-bit clip. The idle
> enumeration leads to the false conclusion that 10-bit has no hardware path.

## 10-bit output

10-bit decodes in hardware and lands as `NV15`. That is 10-bit packed 4:2:0 with no
byte-aligned samples, and the hardware writes it natively. No driver change can substitute
a friendlier format. What this SoC lacks is a hardware consumer, and unlike the RK3588 it
has no hardware route to add one. The display scans `NV15` out and ffmpeg unpacks it on
the CPU, and nothing on this part converts it in silicon.

| Consumer | Takes `NV15` |
|---|---|
| VOP2 display planes | Yes. `DRM_FORMAT_NV15`, `NV20` and `NV30` in `rockchip_drm_vop2.c` [source] |
| RGA2 | Input only. It can read `NV15` and has no 10-bit surface to write |
| VDPP | No. Its pixel path is NV12 and NV21 only. See [VDPP](#vdpp) |
| `hwdownload` and swscale (CPU) | Yes. `format=yuv420p10le` unpacks it, given an ffmpeg that knows `NV15` |
| Vulkan, Mesa GL and EGL, mpv | No. Vulkan has no 10-bit packed 4:2:0 format at all |

10-bit playback through a KMS plane therefore needs no conversion, because VOP2 scans
`NV15` out directly. Filtering or transcoding 10-bit content unpacks it on the CPU, and
that is the only route.

### The ffmpeg build

The `hwdownload` row is a property of the ffmpeg build, not of the driver. It needs an
ffmpeg that knows the NV15 and NV20 pixel formats in libavutil and libswscale, and tags
them on the v4l2request hwcontext. Kwiboo's `v4l2-request-n8.1` branch lacks them. The
changes that add them need neither MPP nor RGA.

A build without them has no 10-bit path at all, and it fails hard rather than falling
back. The hardware decodes the 10-bit stream, and then every consumer fails with
`Failed to transfer data to output frame: -22`. The hwaccel has already accepted the
profile, so no software decode follows, and the caller must drop `-hwaccel` to recover.
Before reading any 10-bit result on this SoC, check that the build knows the format:

```sh
ffmpeg -pix_fmts | grep nv15
```

With that support, the 10-bit path works end to end [measured, 7.2.3]:

- `hwdownload,format=nv15` succeeds on a 10-bit surface and is rejected on an 8-bit one.
- A plain `-hwaccel v4l2request` 10-bit transcode completes.
- All three 10-bit profiles decode bit-exact.
- HEVC Main 10 at 4K decodes at 134 fps for 0.40 cores, against software's 34 fps for
  6.46 cores.

### No RGA3

This part has no RGA3, and that is what makes the 10-bit ceiling structural [source].
`rk3576.dtsi` declares `rga2_core0` and `rga2_core1`, both
`compatible = "rockchip,rga2"`, and no RGA3 node. The RK3588 has `rga3_core0`, `rga3_core1`
and `rga2_core0`. The format table of ffmpeg's RKRGA filters encodes the consequence:

| Pixel format | Format-table entry |
|---|---|
| `P010` | RGA3 only |
| `P210` | RGA3 only |
| `NV15` | RGA2 only, as input |

The same filters do not convert an RGA3-only format to an RGA2-only one.

HDR tone mapping through RGA is therefore impossible on this SoC. It is not a driver gap,
and no patch fixes it. A third-party RK3576 NAS write-up reaches the same conclusion from
the failure side [reported]. It reports that the failure is loud rather than a silent
fallback.

### VDPP

VDPP has no 10-bit picture path either [source, MPP userspace tree]. The RK3576 uses the
`vdpp2_*` implementation, whose pixel-processing block (VEP) checks `src_fmt` against NV12
and NV21 and rejects everything else. `vdpp2_reg.h` gives `sw_vdpp_src_fmt` and
`sw_vdpp_dst_fmt` two bits each. Their only values are YUV444 and YUV420, and there is no
bit-depth field at all.

The 10-bit entries that make VDPP look like a 10-bit consumer map to
`VDPP_DCI_FMT_Y_10bit_SP`. Only the DCI block-size computation consumes that value, and
DCI is a histogram engine. It reads luma and produces statistics for dynamic contrast, and
it emits no picture. VDPP's only 10-bit capability is reading luma for a histogram.

Two further facts limit VDPP [source]:

- MPP compiles the VDPP userspace only with `ENABLE_VPROC_VDPP` on. That option is
  separate from `ENABLE_VPROC`, which carries IEP2 on the RK3588.
- VEP is capped at 1920×2048, so it is not a 4K scaler even for 8-bit content.

## RGA

The RK3576 has two RGA2 cores, at `0x27920000` and `0x27930000`, and both carry DT nodes.
The vendor's out-of-tree multicore driver drives them at `/dev/rga`. Both cores are
functional and correct on it [measured]:

| Kernel | Exercise | Result |
|---|---|---|
| 7.1.7 (2026-08-09) | Both cores over DMA-BUF | Correct |
| 7.2.0 (2026-08-26) | 1080p to 720p NV12 scale, and NV12 to RGBA8888, on both cores | 20 of 20 soak jobs, zero IOMMU faults, zero WARNs |
| 7.2.3 (2026-09-13) | NV12 to RGBA8888, 4K to 1080p downscale, 1080p NV12 copy | All exact, 600 of 600 soak jobs, zero IOMMU faults |
| 7.2.8 (2026-10-02) | Scale and color convert, on both cores | Pass, no new kernel log lines |

At 7.2.3 the `librga` client imports DMA-BUF from `/dev/dma_heap/system`. The color
convert is exact on all four quadrants, and the downscale is exact across every tile
interior. The soak runs 579 jobs/s for the 1080p color convert and 152 jobs/s for the 4K
downscale.

At 7.2.0 the client runs as an unprivileged account. It checks its output against a luma
ramp rather than trusting a "Run successfully" return. It selects each core with
`imconfig(IM_CONFIG_SCHEDULER_CORE, …)`, as 0x4 and 0x8. Core 0x8 behaving identically
proves that the driver's `iommu_attach_group()` for the second core worked. The driver
discards that call's return value, so nothing else reports it.

Link a client against librga's C entry points (`IM_C_API`, the `_t`-suffixed functions
plus `wrapbuffer_handle_t`). The unsuffixed API is C++ with default arguments.

### Buffers

> **Pass RGA its buffers as DMA-BUF fds (`importbuffer_fd`), never as virtual
> addresses.** `wrapbuffer_virtualaddr` faults the IOMMU on roughly a third of jobs, and
> the jobs that complete run ~7x slower. One run took a kernel oops and left the caller
> unkillable. Both cores behave identically, so the fault is not a per-core or geometry
> effect [measured].

`dma_heap` buffers are cached. Bracket every CPU access with `DMA_BUF_IOCTL_SYNC`:

- `DMA_BUF_SYNC_START|WRITE` before filling a buffer.
- `DMA_BUF_SYNC_END|WRITE` after filling it.
- `DMA_BUF_SYNC_START|READ` before checking results.

Without the bracket the device sees a stale source and the CPU reads a stale destination.
That looks exactly like an RGA correctness bug.

"Every CPU access" includes a destination pre-fill. A harness that first sets the
destination to a known value, to prove that RGA wrote something, must bracket that fill
too. Left unbracketed, the dirty cache lines land on the engine's output after the job
completes. `imcopy` then reports a byte-wrong copy on hardware that copied correctly. The
pre-fill is the access a harness most often leaves unprotected, because the source fill is
the obvious one.

A downscale is exactly checkable only inside large flat tiles. A 2:1 decimation misses
2×2 block values by one or two near block edges, because the scaler's filter kernel
reaches across them. Use source tiles big enough to have an interior (64×64 works for 4K
to 1080p), and check only the interiors. Spread the tile values far apart, so a blend
cannot land on a neighbor's value and pass by accident.

### The rga3 init WARN

On 7.2 the vendor multicore driver logs a WARN at module init. It is cosmetic:

```
WARNING: drivers/iommu/iommu.c:2061 at iommu_set_fault_handler+0x1c/0x24
  rga_iommu_bind+0x154/0x1e0 [rga3]
rga: IOMMU binding successfully, default mapping core[0x4]
```

`[rga3]` is the module name of the vendor multicore driver, and not evidence of RGA3
silicon. This SoC has RGA2 cores only.

The WARN is not a NULL domain [source]. The 7.2 condition is
`WARN_ON(!domain || domain->cookie_type != IOMMU_COOKIE_NONE)`, and only the second arm can
fire on this path:

- If `iommu_get_domain_for_dev()` is NULL, `rga_iommu_probe()` returns `ERR_PTR(-EINVAL)`.
- Probe then logs `failed to attach iommu` and sets `scheduler->iommu_info = NULL`.
- `rga_iommu_bind()` skips any core whose `iommu_info` is NULL.

A NULL domain therefore cannot reach the call, and dmesg shows no `failed to attach iommu`.
The domain is the DMA-IOVA default domain, whose `cookie_type` is already taken. The same
explanation holds on the RK3588 and predates 7.2, so the WARN is specific to neither.

`core[0x4]` does not mean that only one core got a mapping. 0x4 is
`IM_SCHEDULER_RGA2_CORE0`'s id, and core 1 is 0x8. The line prints `map_scheduler_index`,
the default mapping core, which is singular by design. The driver points every further
IOMMU core's `info->domain` at the first core's domain, and attaches it with
`iommu_attach_group()`. The two cores are meant to share one domain, so do not give the
second core its own. The soak results above prove that the silent attach works.

The one real consequence is that `iommu_set_fault_handler()` returns early, so the driver
never installs `rga_iommu_intr_fault_handler`. On an IOMMU fault the driver does not
soft-reset the core and fail the job with `-EACCES` at once. Recovery falls to the
1-second job timeout. The WARN taints the kernel, and nothing breaks functionally.

### Limits

Three limits shape any plan that uses RGA on this SoC:

- Color conversion covers BT.601 limited and full range and BT.709, and not BT.2020. The
  RK3588's RGA3 covers BT.2020.
- RGA writes no 10-bit surface at all. See [No RGA3](#no-rga3).
- FFmpeg cannot reach RGA on this SoC. The `scale_rkrga` and `vpp_rkrga` filters require
  `--enable-rkmpp`, and MPP needs the vendor `mpp_service` kernel framework, which is not
  mainline. RGA serves only programs that link `librga` directly.

Mainline review rejects this driver's shape. On the VEPU510 RFC, Heiko Stübner rejected
per-core V4L2 nodes, holding that scheduling belongs inside the driver, as
`rkvdec_disable_multicore()` does.

## GPU

The GPU is an Arm Mali-G52 MC3, a Bifrost part of architecture v7, at register base
`0x27800000` [DTB]. On mainline, Panfrost binds and initializes a `renderD128` render node
[measured, 7.1.3]. Mesa's `panfrost` gallium driver renders through it, with no vendor blob
and no `libmali`.

The GPU power domain must be bound to its `vdd_gpu_s0` supply, or a late Panfrost probe
faults the unpowered block. The mechanism and the device-tree change are in
[the GPU power domain needs its supply](corrections-and-gotchas.md#the-gpu-power-domain-needs-its-supply).

### OpenGL ES

On 7.1.3 with Mesa 26.1.5-1, Panfrost drives OpenGL ES 3.1 and the OpenGL 3.1 core and
compatibility profiles at 4K [measured]:

```
GL_RENDERER:  Mali-G52 r1 MC3 (Panfrost)
GL_VERSION:   OpenGL ES 3.1 Mesa 26.1.5-1      (also GL 3.1 core/compat)
Surface Size: 3840x2160 fullscreen
glmark2-es2-drm  [build] 462 FPS  [texture] 425 FPS  [shading] 421 FPS
```

The result holds on later kernels:

- On 7.2.3 (2026-09-13), a surfaceless GLES 3.1 context rasterizes, reporting Mali-G52 r1
  MC3 under Mesa 26.1.6 [measured].
- On 7.2.8 (2026-10-02), GL renders over GBM and surfaceless [measured].

Three Debian packages turn the bound driver into a usable one: `libgl1-mesa-dri`, which
carries `panfrost_dri.so`, `libegl-mesa0` and `libgbm1`. A program also links the loader
packages `libegl1` and `libgles2`. Mesa ships the backend, and the loader is separate.

On a system with no display server, `eglinfo` reports the GBM platform working and the X11
and Wayland platforms failing [measured]. That is the expected shape: KMS clients go
through GBM.

### Vulkan

This GPU has no Vulkan filter route, at either bit depth [measured, 7.2.3] [source].
Mesa's PanVk refuses Bifrost arch v6 and v7 with `VK_ERROR_INCOMPATIBLE_DRIVER`, unless
`PAN_I_WANT_A_BROKEN_VULKAN_DRIVER` is set. The refusal is in `panvk_physical_device.c`,
both in 26.1.6 as Debian forky ships it and on Mesa main.

With `mesa-vulkan-drivers` installed and nothing set, the only Vulkan device is llvmpipe.
ffmpeg's `-init_hw_device vulkan` takes it without complaint. The same 300-frame
1080p-to-720p scale then costs 28.55 CPU-s through `scale_vulkan`, against 10.47 CPU-s
through `hwdownload` and swscale.

Forced onto the G52, PanVk computes wrong results rather than running slowly. An
`hwupload,hwdownload` round trip is bit-exact, so the transfers are right. Every filter
tested computes wrong chroma, measured against llvmpipe, which matches the CPU filters:

| Filter, G52 against llvmpipe | Y dB | U dB | V dB |
| --- | ---: | ---: | ---: |
| `transpose_vulkan` | inf | 12.8 | 12.8 |
| `scale_vulkan` to 720p | ~18-21 | ~6 | ~7 |
| `gblur_vulkan` | 88.7 | 42.1 | 36.2 |
| `avgblur_vulkan` | inf | 45.0 | 37.4 |
| `bwdif_vulkan` | 63.0 | 37.5 | 32.4 |

Mesa 26.2.2 gives the same numbers as 26.1.6. `PANVK_DEBUG=sync` changes nothing, so the
cause is not a flush or submit race.

Two further obstacles stand in the way:

- In ffmpeg, `hwcontext_vulkan.c` queries external-semaphore support without declaring a
  timeline semaphore, then exports a timeline one. PanVk emulates timelines below arch
  v10 and cannot export one, so `hwupload` fails with `VK_ERROR_INVALID_EXTERNAL_HANDLE`.
  The chroma results above need an ffmpeg with that mismatch fixed. The RK3588's G610 is
  v10, with native timelines, and never hits it.
- `libplacebo` refuses the G52 outright. It needs Vulkan 1.2, and PanVk exposes 1.0 on
  Bifrost in Mesa 26.1 and 26.2.

ffmpeg's filtering on this SoC therefore runs on the CPU, at either bit depth. OpenGL ES
is unaffected: Panfrost drives GLES 3.1 on the same GPU.

## Encode

Mainline has no encoder driver for this SoC, and one is in flight upstream [VEPU510].
`[RFC PATCH 0/3] media: rockchip: VEPU510 H.264 encoder for RK3576` (Jiaxing Hu,
2026-07-22) is a from-scratch V4L2 stateful m2m encoder. It is explicitly not built on the
MPP-service model. Intra frames work on real hardware (a Radxa ROCK 4D), and P-frames
stall the encoder watchdog, which is why it is an RFC. Collabora is separately writing the
RK3588 VEPU580 equivalent, as a Rust kernel module plus a Mesa userspace driver over
Vulkan Video. Nothing is merged.

On mainline, encode on this part runs on the CPU cores.

## Userspace

Upstream FFmpeg has no `v4l2_request` hwaccel: it is absent from `configure` on
`release/8.1` [source]. Debian forky's ffmpeg (`7:8.1.2-2+b1`) therefore cannot use
`/dev/video0`, however it is invoked. GStreamer's `v4l2codecs` plugin reaches the node
with no rebuild, through `v4l2slh264dec` and `v4l2slh265dec`.

An ffmpeg that drives this decoder carries three things:

- The `v4l2_request` hwaccel, from Kwiboo's `v4l2-request-n8.1` branch, configured with
  `--enable-v4l2-request`.
- The NV15 and NV20 pixel formats, tagged on the v4l2request hwcontext. See
  [the ffmpeg build](#the-ffmpeg-build).
- Code that sets the HEVC SPS reference-picture-set controls.

Decode also wants `-hwaccel_output_format drm_prime`. Reading frames back into system
memory costs far more than decoding them.

### HEVC reference-picture-set controls

The kernel exposes `hevc_short_term_ref_sets` and `hevc_long_term_ref_sets`, and an ffmpeg
that sets the SPS reference-picture-set controls populates exactly those. On the RK3588,
leaving them unset costs 15 of 147 HEVC conformance streams. On this SoC their effect is
unproven. Synthetic x265 content does not expose the gap: clips encoded with
`bframes=8 b-pyramid ref=6 open-gop` decode bit-exact with or without the controls set.
Settling it needs the Fluster conformance suite on an RK3576 board.

## Sources

- [DTB]: the H96 MAX M9 factory device tree, decompiled.
- [VEPU510]: the linux-media thread on `[RFC PATCH 0/3] media: rockchip: VEPU510 H.264
  encoder for RK3576` (2026-07-22).
- The vendor kernel's VDPU383 programming: `rockchip-linux/kernel`, branch `develop-6.1`,
  `mpp_rkvdec2.c`.
- The PanVk Bifrost refusal: Mesa `panvk_physical_device.c`, 26.1.6 and main.
- VDPP: the MPP userspace tree, `vdpp2_reg.h` and the `vdpp2_*` implementation.
- Mainline kernel sources: `rkvdec.c`, `rkvdec-rcb.c`, `rockchip_drm_vop2.c` and
  `v4l2-ctrls-core.c`.
