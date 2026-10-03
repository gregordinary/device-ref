# RK3576 NPU reference

This document describes the RK3576 NPU under the mainline `rocket` DRM-accel driver. It
covers the driver and its device-tree contract, the NPU rail, power-on sequencing, the
second core, the operating point and the rule for register access. It is a SoC-level
reference. The measurements come from the reference unit, an H96 MAX M9. A claim stated for
the SoC rests on that board unless the text says otherwise.

## The driver

The in-tree `rocket` DRM-accel driver matches `rockchip,rk3576-rknn-core` once Jiaxing
Hu's linux-rockchip RFC v2 series is applied and `rk3576.dtsi` carries the NPU nodes
[measured]. The RFC as posted computes one correct operation per power session. The
`TASK_NUMBER` field of `PC_TASK_CON` is 16 bits wide on the RK3576 and 12 bits wide on the
RK3588. The driver writes a fixed RK3588 word, which leaves the program counter
mid-program.

Writing that field at the RK3576 width lets programs run to completion. Reliable
back-to-back submission additionally needs fixes to job retirement, IOMMU lifetime and
uapi extensibility, which are carried out of tree. With these changes in place,
`/dev/accel/accel0` appears and jobs compute:

- An int8 convolution is bit-exact against a CPU model on this silicon
  [measured, 7.1.3].
- Multi-task row-windowed programs stay bit-exact submitted back to back
  [measured, 7.1.3].
- A 69-case single-core test list passes 69 of 69 with the rail switched by the NPU power
  domain [measured, 2026-10-02, 7.2.8].

Each core node in the SoC dtsi lists both NPU power domains, because the convolution
buffer (CBUF) is read through `PD_NPU1` whichever core computes. `rocket` attaches
whatever domains the node lists, so a trimmed node runs half fed rather than refusing to
probe. The binding is what pins the count at two.

On the vendor kernel the NPU runs the RKNPU2 stack. `rocket` is a CNN accelerator driver
with no matmul path, so it does not serve LLM inference. The RK3576 register map is
shifted and re-packed relative to the RK3588, so a userspace needs its own RK3576 encoder.
No released userspace carries one [source].

[`rocket-userspace`](https://github.com/gregordinary/rocket-userspace), a public RK3588
userspace, is bit-exact on the RK3588 and names the
RK3576 as its next target. Its machine parameters (CBUF size, core count, datatype mask)
need confirming on the RK3576 first. The only posted RK3576 encoder is Mesa MR !43804
(Jiaxing Hu, open since 2026-08-17, unmerged) [source]. It handles regular uint8
convolutions only and stays within one count of a CPU reference on a Radxa ROCK 4D. It
declines depthwise, pointwise and image convolutions of up to four channels to the CPU.

Mainline therefore delivers an NPU device on the RK3576, not an inference stack.

## The NPU rail

Mainline `rocket` references no regulator API, so the `npu-supply` property the binding
requires is inert [source]. The regulator core's disable-unused pass switches off a rail
that no consumer claims. On the reference unit `vdd_npu_s0` goes down at t~33 s, well
after `rocket` probed cleanly at t~8 s [measured]. The NPU fails from then on, and the
boot does not. The GPU domain shows the same class of defect with a different symptom, as
[the GPU power-domain entry](corrections-and-gotchas.md#the-gpu-power-domain-needs-its-supply)
describes.

Two device-tree arrangements hold the rail:

- `regulator-always-on` on `vdd_npu_s0` pins the rail up. The 7.1.3 results in
  [the driver section](#the-driver) ran with the rail pinned this way [measured].
- The NPU power domain takes the rail. The `pd_npu` node carries
  `domain-supply = <&vdd_npu_s0>`, and the rail keeps `regulator-boot-on` and drops
  `regulator-always-on`.

In the second arrangement the rail comes up before the domain powers and drops after it.
The GPU domain uses the same arrangement, and the posted upstream RK3576 NPU series gives
it to the Radxa ROCK 4D. It works on the reference unit [measured, 2026-10-02, 7.2.8]:

- The rail reads `disabled` at rest and `enabled` at 750 mV with a job in flight.
- A job submitted while the rail is off computes correctly.
- The single-core test list passes 69 of 69.

The second arrangement has three traps:

- `domain-supply` does nothing unless the pmdomain driver flags the domain
  `need_regulator`. Mainline 7.2.8 does not flag `RK3576_PD_NPU`, and the posted series'
  pmdomain patches do [source].
- A `domain-supply` added while `regulator-always-on` stays shows no effect, because a
  pinned rail cannot follow the domain.
- `num_users = 1` in `regulator_summary` is not evidence that a consumer claimed the rail.
  Rails with no device-tree consumer at all, such as `vddq_ddr_s0` and `vdd_logic_s0`,
  read 1 as well [measured].

### The rail at rest

Read the at-rest state only after a job has run. From boot until the first job, the rail
reads `enabled` with the device runtime-suspended [measured, 2026-10-02, 7.2.8]. The genpd
`active_time` for `npu`, `nputop` and `npu0` equals the uptime, which was 49 h on the
reference unit. The `npu1` domain of the disabled core read 143 s. After the first job the
domains power off about 50 ms after each burst.

The domain chain therefore stays powered from boot until the first runtime-PM cycle. That
costs idle power on a system that never runs a job. Read from the source, the IOMMU probes
after genpd's single unused-domain sweep (the PMIC driver loads from the initramfs), and
its single-domain attach powers the chain on [source]. Why the probe-time autosuspend in
`rocket` does not power the chain back down is not isolated.

## Power-on sequencing

The NPU power domain needs sequencing at power-on, and mainline provides none. Between
releasing the idle request and restoring QoS, the posted series' pmdomain patches take two
steps in this order:

1. Pulse the core's bus-interface reset, `SRST_A_RKNN0_BIU` on `PD_NPU0` or
   `SRST_A_RKNN1_BIU` on `PD_NPU1`.
2. Wait a 15 us settle.

Without the reset, the first register access after power-on hangs. Without the settle,
the QoS writes land while the domain is still coming up. The first cold power-on of the
NPU domain then takes an asynchronous SError, with the rail pinned or switched alike.

The reset goes first so that the settle, not the deassert, sits directly ahead of the QoS
writes. At 15 us the series' v14 cover letter (2026-09-24) reports 536 clean cold
power-ons [reported]. The reference unit has not reproduced that count.

## The second core

The RK3576 has two NPU cores. Every result in [the driver section](#the-driver) and
[the rail section](#the-npu-rail) is core 0. Core 1 binds and runs jobs once its node is
described correctly, which takes two corrections to the RFC's node [measured]. Jobs that
run on both cores at once compute wrong answers.

### Register base

Core 1's register banks are at `0x27708000` [measured]. The RFC places them at
`0x27710000`, which is `RKNN_NSP`, a different module. Pointed there, the driver reads
boot-dependent noise for a version register, the IOMMU never resets, and every job times
out.

Three sources agree on `0x27708000`: the TRM address map, the factory DTB's `reg` pair
and the RFC's own `rknn_mmu_1: iommu@2770a000` [source][TRM][DTB]. That MMU address is
`0x27708000 + 0x2000`, the offset core 0's MMU uses. At `0x27708000` the version register
reads back the same word as core 0's.

### Clocks

Both cores need all six clocks [measured]. The RFC gives core 1 four. `rocket` takes the
clocks with a hard `devm_clk_bulk_get()` sized per SoC rather than per node. Core 1
therefore fails probe with `-ENOENT: Failed to get clk 'aclk_cbuf'`.

`ACLK_RKNN_CBUF` and `HCLK_RKNN_CBUF` are single clocks in the CRU, with no `_CBUF1`
variant, because the CBUF is shared. It is one 1 MB buffer of 16 banks of 64 KiB, in
`RKNN_TOP` under `PD_NPUTOP`. The RK3588 instead gives each core its own 12 banks of
32 KiB. The bank geometry comes from the machine parameters in Mesa MR !43804 [source].

### Concurrent jobs

With both cores bound, two `rocket` jobs in flight at once compute wrong answers
[measured]. Of the calls, 96-100% are wrong, over as much as 94% of the surface, with
deltas spanning the whole int8 range. The corruption crosses process boundaries.

Simultaneous execution is the sole discriminator. In the control, the same two file
descriptors alternate across both cores through a mutex, with one job in flight. They
are exact over 96 calls per cell across three shapes. Run concurrently, they corrupt. The
measurements rule out the second core's existence, job-to-job inheritance and the IOMMU
mapping.

The shared CBUF is the mechanism this points at, by elimination rather than proof
[inferred]. It is one 1 MB region at `0x3fe80000` outside either core's bank, behind one
`NPU_GRF_MEMGATE_CON0` gate and one clock pair. Nothing an encoder emits selects a core,
so both cores stage into the same place.

The vendor driver is the only Rockchip NPU configuration with a `cache_sgt_init`, and it
hands each core a disjoint slice of the CBUF [source]. Core 0 takes banks 0-6 and 14, and
core 1 takes banks 7-13 and 15. Mainline `rocket` has no such partitioning, which leaves
the split to the userspace encoder. Mesa MR !43804 budgets all 16 banks with no per-core
split, so it does not provide one either.

### Throughput

A second core adds no throughput that core 0 does not deliver alone [measured]. The table
compares both cores bound against a one-core control, two file descriptors on core 0:

| Shape | Two descriptors on core 0 | Both cores bound |
|---|---|---|
| 8×64×32 | 1.51-1.69x | 1.55-1.74x |
| 32×1024×512 | 1.62-1.66x | 1.49-1.68x |
| 128×1024×1024 | 1.49-1.51x | 1.38-1.42x |

Both cores stay within the one-core range at the small shapes and fall below it at the
largest. The gain is submit pipelining at a ~340 us submit floor rather than parallelism,
and one core delivers it alone.

The NPU nodes in the SoC dtsi default `rknn_core_1` and `rknn_mmu_1` to `disabled`. A
board that keeps that default in its `.dts` keeps core 1 off across kernel package
upgrades, which a runtime unbind does not. `rknn_core_0` lists both NPU power domains, so
the CBUF stays fully powered with core 1 down. The reference unit's device tree enables
core 0 and its MMU only.

## Operating point

The operating point is one untested candidate cause of the two-core corruption.
`CLK_RKNN_DSU0` is the one clock behind both cores' `aclk` and the CBUF's. Every
concurrency measurement ran it at 786 MHz, the rate the bootloader leaves in place
without a device-tree assignment [measured]. AUPLL read 786431952 and 786431998 Hz.
`vdd_npu_s0` stayed at the 750 mV the PMIC boots it at, with no OPP table or devfreq on
the NPU to move either.

The posted upstream series reports a two-core defect of its own at exactly that point on a
ROCK 4D [source]. Two cores running together write 11 to 25 wrong words in each 5400-row
pass, and the defect clears at 594 MHz. The series notes that Rockchip's OPP table asks
only 725 mV for its 500 and 600 MHz steps. Its clock assignment sets 594 MHz (GPLL / 2,
exact) on both core nodes.

With that assignment `clk_rknn_dsu0` reads 594000000 Hz idle and under load
[measured, 2026-10-02, 7.2.8]. A two-core run at 594 MHz is untested, because core 1 stays
disabled in that configuration.

### The voltage bin

The reference unit's own voltage bin argues against undervolt as the cause. Every
vendor-kernel capture of the unit logs `RKNPU ... bin=0 leakage=4 pvtm=843 pvtm-volt-sel=7`
[measured, 6.1.75][LOG]. The factory NPU table gives that bin (`opp-microvolt-L7`) these
voltages [source][DTB]:

| Rate | `opp-microvolt-L7` | Default row |
|---|---|---|
| 800 MHz | 725 mV | 800 mV |
| 900 MHz | 775 mV | 850 mV |
| 950 MHz | 800 mV | 875 mV |

Below 15 °C the table sets a 750 mV floor. By the vendor's own table, 786 MHz on
750 mV is not short of voltage on this chip. The caveat is that the vendor kernel clocks
the NPU from a PVT-PLL and mainline clocks it from AUPLL. Together with corruption far
heavier than upstream reports, that makes a different defect the likelier reading
[inferred]. Device trees for testing both cores at other rates and voltages are listed
under [OPP] in the sources.

Read the rate back before trusting a request for 786 MHz in a DTB. `clk_rknn_dsu0` is a
composite over GPLL, CPLL, AUPLL and SPLL that takes the closest rate at or below the
request. An AUPLL a few hertz above 786.432 MHz therefore silently yields SPLL's 702 MHz
[OPP].

## Register access

Do not read NPU registers while the block's AXI clock is gated. **The read hard-hangs the
SoC** [measured]. Nothing logs and nothing panics, and only a power cycle recovers. Before
any MMIO read at these addresses, check `enable_count` in `clk_summary` for the core's
`aclk`. After a power-on, the first access also hangs unless the bus-interface reset was
pulsed, as [power-on sequencing](#power-on-sequencing) describes.

The NPU addresses this document uses:

| Block | Address |
|---|---|
| NPU node in the vendor device tree | `npu@27700000` [DTB] |
| Core 1 register banks | `0x27708000` |
| Core 1 MMU, `rknn_mmu_1` | `0x2770a000` |
| `RKNN_NSP`, a separate module | `0x27710000` |
| Shared CBUF, 1 MB | `0x3fe80000` |

The register layout inside each core differs from the RK3588's, as
[the driver section](#the-driver) describes.

## Sources

A `[source]` claim reads the mainline Linux tree at the kernel the entry names, a posted
series, or the Mesa merge request it cites. The citation keys are:

- [DTB]: the H96 MAX M9 factory device tree (`h96-m9_original.dtb`, from the Armbian
  forum's H96 MAX M9 thread), decompiled.
- [LOG]: six serial-console captures of the reference unit's vendor-kernel boot, all on
  6.1.75.
- [OPP]: the operating-point experiment. Its device trees run both cores at 786 or 594 MHz,
  and 786 MHz at 800 and 850 mV. It also records the composite-clock rate note.
- [TRM]: the Rockchip RK3576 TRM V1.2, Part 1 and Part 2.
