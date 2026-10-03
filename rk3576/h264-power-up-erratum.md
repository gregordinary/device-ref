# RK3576 H.264 power-up erratum

This defect record covers one RK3576 erratum: hardware H.264 decode on the VDPU383 comes
up wrong on roughly one power-up in three. It states what the defect does at pixel level,
what the work-around is and is not, and which avenues are measured and refuted. The defect
belongs to the VDPU383 block and the shared `rkvdec` driver, so it is SoC-wide. It is
measured on one board, the H96 MAX M9, and nothing about it is board-specific. The
decoder's nodes, formats and ceilings are in the [RK3576 video reference](video-reference.md),
and the board's other subsystems are in the
[H96 MAX M9 enablement status](h96-max-m9/mainline-enablement-status.md).

Every result on this page is [measured] on the reference unit unless tagged otherwise.

## Summary

A work-around exists. It submits a canned 2×2 H.264 IDR job from
`rkvdec_runtime_resume()`, before any other decode of that power cycle can run.
[The work-around](#the-work-around) describes it in full. The latch itself is not
root-caused. The latch is set independently of every register the driver writes and
every reset line the binding declares [inferred]. The open avenue is the warm-up job's own
[register image](#the-warm-up-jobs-register-image).

The RK3588's VDPU381 is a different decoder generation and does not show the defect. With
the same harness and clips, 0 of 66 sessions are corrupt on a Turing RK1 [measured]. Against the
RK3576's 4K rate, that result has p ~ 2e-8. None of the work-arounds on this page applies
to the RK3588.

## The symptom

Without the warm-up, hardware H.264 through `rkvdec` returns an entirely wrong picture for
the whole of a power cycle. PSNR is ~17 dB, and the picture is wrong from frame zero.
Retrying does not help until the block powers down and up again. `rkvdec`'s
`autosuspend_delay_ms` is 100. By default the block therefore power-gates between decodes,
and each decode draws again.

The state is latched at power-up, not per session or per decode. With the autosuspend
delay raised so that three consecutive decodes share one power-up, all twelve groups come
out internally uniform. A bad group returns the same wrong hash three times, whatever the
position in the group. Two consequences shape every experiment on this defect:

- The power-up is the trial. A failure rate computed over sessions is a property of how
  the runs were grouped, not a constant. Compare within a campaign against an interleaved
  control, never across campaigns.
- The corruption is deterministic. A small discrete set of wrong outputs recurs across
  sessions and across days, so it is not DMA garbage. It is also unrelated to the
  read-cache non-determinism that [arming the read caches](video-reference.md#the-read-caches)
  removes.

HEVC is unaffected in every arm of every campaign, which makes it the free control. A
change that breaks HEVC has broken the block rather than fixed anything.

## The damage at pixel level

A good hardware frame 0 is bit-exact with software. A corrupt one differs in exactly one
row in eight: the rows where `r % 8 == 4`. Every pixel in those rows is wrong, every other
row is bit-exact, and there is no column phase at all. The pattern appears in each plane
in its own coordinates. In luma it hits rows 4, 12, 20 and on, and in chroma rows 4, 12,
20 and on of the half-height plane. It therefore tracks the row cache rather than the
4:2:0 geometry.

The damaged row is a deblocking filter output with the p side reading zero. For a damaged
row `r`, with `q0 = ref[r]` and `q1 = ref[r+1]`:

```
bad = q0 - Clip3(-tc, tc, ((q0 - p0) * 4 + (p1 - q1) + 4) >> 3)     p0 = p1 = 0
```

That is the normal-filter equation for `q0` in clause 8.7.2.3 of the H.264 specification.
Where the delta lands inside the clip, the model has no free parameter and is exact on
every pixel of every plane. One latched state runs the strong (`bS=4`) variant instead,
`bad = (q0 + 2*q1 + 2) >> 2`, again with the p side zero. The strong form is the commoner
of the two firing forms, at 46% of corrupt power-ups. So the latch selects the filter
variant as well as the threshold.

The fit settles three things:

- The fitted `tc` is uninitialized state, not a stream-derived threshold. Its support over
  634 corrupt power-ups is 37, 38, 39, 40, 45, 46, 48, 102 and 109. Every value lies
  outside H.264's legal range, which tops out at 25, and 93% of normal-filter draws sit at
  37 or 38.
- Clean is a state, not `tc == 0`. There is no mass near zero, and only 3 of 290,880
  modeled pixels have a zero unclipped delta. The latch is at least three-valued: off,
  normal filter with a threshold, and strong filter. Eleven distinct states appear over
  2000 power-ups, and all 634 corrupt frames fit at 100.000000%.
- The stream's own deblocking flag does not gate it. The phantom filter fires on a clip
  whose slice header carries `no-deblock=1`.

The damage reaches the DPB. Frame 0 is confined to rows 4 mod 8, with 0.0000% of any
other row wrong. Frame 1 already has 12.7% of other rows wrong, and frame 5 has 20.3%. The
filter writes into the reference buffer, and motion compensation spreads the damage across
the picture. The defect is therefore in the reconstruct path, not a display-time
transform.

> **Analyze frame 0 of a clip encoded with deblocking off.** With the real filter on, the
> damage bleeds to rows 2-5 mod 8 and hides the phase. Those rows are the luma deblock
> footprint of an edge at row 4. Disabling deblocking does not reduce the defect: 10 of 30
> power-ups are corrupt with it on, and 7 of 30 with it off. It strips the bleed and
> exposes the signature.

## The rate

The defect hits 31.7% of power-ups without the warm-up, and that is one number. The count
is 634 of 2000 (95% CI 29.7-33.8), from 20 boots of 100 draws each with the warm-up
disabled throughout. The rate has three properties:

- It has no per-boot component and no drift. Cochran's Q is 14.70 on 19 df (p = 0.74), I²
  is 0%, and the between-boot SD is exactly 0. Neither the draw index, the thermal zones,
  the previous draw's outcome nor the previous draw's state predicts anything. The draw is
  i.i.d., so no boot is luckier than another.
- The 1080p and 4K rates are one rate: 92 of 300 against 84 of 300, interleaved (CMH
  p = 0.53).
- A 23.1% rate from a smaller sample is sampling noise, not a different regime. All 25
  arms measured are homogeneous (Q 28.7 on 28 df, p = 0.43), so that figure needs no
  mechanism of its own.

## The work-around

The work-around has two parts:

- `rkvdec_runtime_resume()` submits a canned 2×2 H.264 IDR job before any decode of that
  power cycle can run. It writes the register image into the `function` window and kicks
  `DEC_ENABLE`, the way a codec backend runs a decode. The job runs unconditionally on the
  VDPU383, with no module parameters. It is silent on success, and logs a `dev_warn` on a
  timeout or a non-zero error status, which is what a soak greps dmesg for.
- A guard in the IRQ handler returns on a NULL context: `if (!ctx) return IRQ_NONE;`. The
  warm-up job has no V4L2 context, so the guard is its prerequisite. The guard is also a
  self-contained fix for mainline `rkvdec`, independent of the erratum.

A failed warm-up allocation calls `dev_err_probe` and fails probe. With the warm-up
unconditional, a device that binds without one has no protection against the erratum for
the whole boot. Refusing to bind is the safer end state.

### What the warm-up needs

The block must complete a decode, and nothing else matters. The bisect ran 400 power-ups
per arm, interleaved against a control drawn on the same rounds:

| Arm | Corrupt | Verdict |
| --- | ---: | --- |
| Control | 133/400, 117/400, 126/400 | 31.3% pooled, at the base rate |
| Full vendor job | 0/400 in each of three arms | Works |
| Link engine primed, no job submitted | 129/400 | No effect (p = 0.82) |
| Without the RCB pointers | 0/400 | Removable |
| Without `reg[13]` core timeout | 134/400 | Loses the fix (p = 0.22) |
| Without `reg[16]` error-proc | 0/400 | Removable |
| Without `reg[20]` and `reg[21]` CABAC | 0/400 | Removable |
| All three removals together | 0/400 | The form described above |

`reg[13]` is the one load-bearing register of the four the vendor adds. At zero the job
still completes and reports ready, but the block retires it without decoding. That is the
same "no decode happened" as the arms that hang, reported as success.

A submitted but incomplete job is worse than no job. An empty register image, no stream or
no output pointer each gives a job that the block never completes. That power cycle then
cannot decode either codec, deterministically. The timeout warning is therefore
load-bearing: it means the power cycle is lost.

### Repair

The warm-up repairs a bad power-up as well as preventing one. The test fires it on
power-ups already classified bad, with the domain pinned so nothing power-cycles in
between. It repairs 96 of 96 bad draws and breaks 0 of 204 clean ones. All six latched
states seen are repaired, and the resume counter stays unmoved on all 300 draws.

Link mode is not the mechanism. The vendor submits the same job through the link engine,
and a direct register kick with no link node repairs 36 of 36. The work-around therefore
runs through the same direct register kick that mainline's codec backends use. The
interrupt mask is not the mechanism either (17/17 with one interrupt armed, 33/33 with
both), and neither is the resume callback.

### Validation

The final form validates three ways:

- Prevention measures 0 of 400 corrupt against an interleaved control's 128 of 400 (CMH
  p = 1.4e-34), with HEVC 40 of 40 in every arm.
- A 200-power-up soak of the parameter-free form, alternating 1080p and 4K, shows zero
  corrupt draws and zero warm-up warnings.
- Decode stays correct across an 8.5-hour s2idle with the decoder held resumed. That
  covers `pm_runtime_force_suspend` and `force_resume`.

## Refuted avenues

Each avenue below was built or measured, and none fixes the defect. They are recorded
so that a later change to `rkvdec` does not repeat them.

### Pulsing the DT reset lines

`rk3576.dtsi` declares five resets on the vdec node, and `rkvdec` references
`reset_control` nowhere, which makes the reset lines the obvious candidate. Pulsing them
from runtime resume breaks hardware decode outright, for both codecs. A runtime-writable
mask measures each subset. Every mask containing `SRST_RKVDEC_CORE` (bit 8) destroys HEVC
too. The masks without it are inert: pooled 23 of 100 against 22 of 100 for interleaved
controls (Fisher p = 1.000). The damage comes from `SRST_RKVDEC_CORE`, not from the BIU
lines.

### The LINK_IP_ENABLE codec asymmetry

The two VDPU383 backends kick the block one bit apart, and the codec that clears the bit
is the unreliable one. Making H.264 write `VDPU383_IP_CRU_MODE` like HEVC leaves the rate
unchanged. It also produces the same latched states as an unmodified kernel, which is
evidence that the latch is independent of the kick sequence. The change is harmless and
is not a fix.

### Holding the block resumed through udev

A udev rule that pins `power/control=on` does not fix the defect. A result of "0 of 20
with the block held resumed" is one power-up, not twenty trials. Writing `on` to a
suspended block forces a power-up and holds it, so all 20 decodes share one draw. One
good draw then reads as 0 of 20.

Over 16 forced power-ups measured directly, 3 of 16 are entirely corrupt, and every
power-up is internally uniform. Such a rule gives roughly one boot in four or five a
decoder that stays wrong until the next reboot. That is worse than the default
autosuspend behavior, where each session draws again.

### A soft reset

The vendor kernel's soft-reset sequence, reproduced exactly, repairs 0 of 22 bad
power-ups and breaks 38 of 38 clean ones. What it breaks is not this defect: the
post-reset hashes match none of the states the board produces naturally.

### An ordinary decode as the warm-up

The block must complete a decode, and still not every completed decode repairs the latch.
A userspace decode repairs 0 of 111 latched power-ups across five campaigns. The canned
job repairs 73 of 73 on the same draws (Fisher p = 4e-53). Other decodes fare no better:

| Warm-up candidate | Latched power-ups repaired |
| --- | --- |
| One 16×16 decode, which the block sees as one intra macroblock, exactly as it sees the canned job | 0 of 30 |
| One HEVC decode | 0 of 26 |
| Twenty 16×16 decodes | 0 of 26 |
| One 1080p decode with deblocking on | 0 of 15 |
| Decodes with the auto clock gates off | 0 of 40 |
| Decodes with the read caches off | 0 of 40 |

### Deblocking and the read caches

Deblocking is not the cause, as [the damage at pixel level](#the-damage-at-pixel-level)
shows. The read-cache configuration is not the cause either. Flipping `read_cache`
between interleaved runs gives 6 of 20 against 5 of 20. The two are independent, and the
read caches affect throughput and nothing more.

## The warm-up job's register image

The open avenue is the 248 words of the warm-up job's register image. The canned job and
an ordinary decode go through the same `function` window, so the next bisect is additive.
Start from the canned image, add what an ordinary decode programs, and find the word whose
presence loses the fix. The two images differ as follows, setting aside the
reference-frame slots 169-210 and the colmv slots 217-232:

| Register | Ordinary decode | Canned job |
| --- | --- | --- |
| `reg064_start_decoder` | 0 | `0x76543210` |
| `reg010_block_gating_en` | `0x3ff` | 0 |
| `reg016_error_ctrl_set` | 1 | 0 |
| `reg013_core_timeout_threshold` | `0x00ffffff` | `0x0000ffff` |
| `reg130_cabactbl_base` | The CABAC table | 0 |
| `reg132_scanlist_addr` | The scaling list | 0 |
| `reg140` to `reg159` | Ten RCB row caches | 0 |
| `reg066` to `reg070` | This stream's length and geometry | The canned one's |

Start with `reg064_start_decoder`. It is the only word the canned job sets that mainline
never writes, on any decode of any codec. The subtractive bisect above never varies it,
because every arm there removes something the vendor adds beyond the job. Removing it is
one cheap subtractive arm.

The warm-up's own output cannot serve as a canary. Its luma plane is 32 bytes by 32
lines, and the block writes all 1024. Every byte is `0x51`, identically on bad and clean
power-ups, rows 4, 12, 20 and 28 included. The warm-up's decode is never damaged. A
check-and-repair path needs a canary ordinary enough to show the defect, which is the
same question as the additive bisect.

> **The register dump is not a flat range.** It is `struct vdpu383_regs_h26x`, packed,
> over four disjoint spans: 660 bytes covering regs 8-30, 64-106, 128-161 and 168-232.
> Read flat, every word past the first span is misassigned.

## Power domain, TRM and vendor driver

The TRM has no video-decoder chapter [TRM]. Neither part of the RK3576 TRM carries a
VDPU383 or RKVDEC register map. It covers the surface instead: the CRU chapter for the
`aclk/hclk_rkvdec_root` tree, the reset lines and `CRU_AUTOCS_*` gating, and the PMU
chapter for `PD_VDEC`.

`PD_VDEC`'s power-domain descriptor is byte-identical between the vendor kernel and
mainline [source]. The vendor kernel differs on `PD_VOP` and `PD_VPU` but not on VDEC, so
the power-domain table is not the difference.

`PD_VDEC`'s SRAM configuration shows nothing. `SYS_GRF` at `0x2600A000` reads
byte-identical with the decoder gated, powered and after a decode. It stays constant
across 1600 interleaved power-ups, with no bit tracking the verdict. Both
`MEMFAULT_STATUS` words are zero, and `PMU_BISR_GLB_STATUS` reads `0x30`: repair done,
nothing found. `MEM_CON11` bits 31:16 are documented as read-only, reserved and reset 0,
and they read `0x1410`. That is a documentation gap, since software cannot set a
read-only field.

The vendor kernel's `rkvdec2_runtime_resume()` enables the clocks and the IRQ, calls
`hack_run`, and does nothing else [source]. The work-around carries all of it.

## Measurement protocol

The protocol below makes the numbers on this page comparable:

1. Force `power/control=auto`, and wait for `runtime_status` to read `suspended`.
2. Run `sleep 1`, then write `control=on`. That write is the power-up.
3. Decode N times on that one draw.
4. Treat within-draw uniformity as a free internal control. A mixed draw means the arm is
   producing garbage, not the latched defect.
5. Interleave two arms draw by draw, and stratify the comparison with CMH.
6. Score the state distribution, not only the rate. The link-engine priming arm in the
   bisect table is a complete null, because it moves neither.

A sample of 20 proves nothing on this defect. One arm read 2 of 20 against 6 of 20, then
3 of 24 against 7 of 24 (pooled p = 0.06, which looks like a find). Isolating its
supposedly active bit then moved the result the other way.

> **`runtime_status` read between runs does not show whether a suspend happened.** It
> reads `active` for tens of ms after a decode exits, and the 100 ms timer usually fires
> during the next process's startup. Only `control = on` held across a whole block
> guarantees no suspend. That is a kernel guarantee rather than an observation.

Two further traps cost time on this defect:

- An off-by-one frame gives the same uniform ~17 dB signature as this defect. Check
  per-frame hashes at zero shift before believing a corruption story.
- `rmmod rockchip_vdec` hangs on this SoC and leaves the module unloadable until a
  reboot, so it cannot swap a driver build. See
  [rkvdec cannot be unloaded](corrections-and-gotchas.md#rkvdec-cannot-be-unloaded).

## Sources

- [TRM]: the Rockchip RK3576 TRM, V1.2 Part 1 and Part 2.
- The measurements: power-up campaigns that run the protocol under
  [measurement protocol](#measurement-protocol) on the reference unit, an H96 MAX M9.
  Frame-level analysis compares per-frame hashes, fits the deblocking model per pixel, and
  diffs the warm-up job's register image against an ordinary decode's.
