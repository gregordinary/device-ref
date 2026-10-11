# RK3588 NPU reference

This document describes the RK3588 NPU and the mainline `rocket` DRM-accel driver that runs
it. It covers the block and its integration: the cores, their addresses, interrupts and MMUs,
the power domains, the rails, the clock and the thermal zone. It then states the
register-level behavior a driver relies on, and the traps around it. It is a SoC-level
reference. The measurements come from the reference unit, a Turing RK1. A claim stated for the
SoC rests on that board unless the text says otherwise.

Register encodings, benchmarks and the evidence behind each measurement are published in
[rockchip-npu-notes](https://github.com/gregordinary/rockchip-npu-notes), cited here as
[NOTES]. This page states the hardware facts and the driver contract, and points there for
the detail.

## The block

The NPU is derived from NVIDIA's open NVDLA design, and carries three identical cores
[source][NOTES]. Each core runs a fixed pipeline, CNA -> CORE -> DPU, and carries these
blocks:

| Block | Role | NVDLA counterpart |
|---|---|---|
| PC | Fetches a register-command program from memory and writes it into the other blocks | None, a Rockchip addition |
| CNA | Feature and weight loading, and the convolution sequencer | CDMA and CSC |
| CORE | The MAC array and its accumulator | CMAC and CACC |
| DPU | Bias, scale, elementwise and lookup-table stages on the accumulator output, and the write-out | SDP |
| PPU | Max and average pooling | PDP |
| DDMA, SDMA | Arbitration toward DRAM and toward on-chip SRAM | MCIF, SRAMIF |

Each core carries its own convolution buffer (CBUF) of 12 banks of 32 KiB, 384 KiB in all
[source][NOTES]. The RK3576 instead shares one CBUF between its two cores, as
[its NPU reference](../rk3576/npu-reference.md#clocks) describes.

The accumulator is NVDLA's. Products accumulate at 34 bits for int8, 48 bits for int16 and
FP44/FP48 for fp16, each rounded to 32 bits before the DPU [source][NOTES]. The accumulator
sums within one hardware task and never across tasks. NVDLA's channel-reduction, copy and
reshape engines (CDP, BDMA, RUBIK) are absent, so the host packs every operand into the NPU's
cube layouts [source][NOTES].

The NPU has no matrix-multiply primitive. A matmul runs as a 1×1 convolution, on the same
pipeline and the same register program as any convolution [measured][NOTES]. Each of int4,
int8, int16, fp16, bf16 and tf32 runs a native matmul [measured][NOTES]. Rockchip rates the
NPU at 6 TOPS, its int8 convolution peak across the three cores [source][NOTES].

## Device tree and address map

`rk3588-base.dtsi` describes each core as its own `rockchip,rk3588-rknn-core` node with its
own MMU node [source][DTS]. The SoC dtsi leaves all six nodes `disabled`, and a board enables
the ones it uses. The RK1 enables all three cores and their MMUs.

| Core | Node | `pc`, `cna`, `core` windows | MMU | Interrupt | Power domain |
|---|---|---|---|---|---|
| 0 | `npu@fdab0000` | `0xfdab_0000`, `0xfdab_1000`, `0xfdab_3000` | `iommu@fdab9000`, banks at `0xfdab_9000` and `0xfdab_a000` | `GIC_SPI 110`, INTID 142 | `RK3588_PD_NPUTOP` |
| 1 | `npu@fdac0000` | `0xfdac_0000`, `0xfdac_1000`, `0xfdac_3000` | `iommu@fdaca000`, one bank | `GIC_SPI 111`, INTID 143 | `RK3588_PD_NPU1` |
| 2 | `npu@fdad0000` | `0xfdad_0000`, `0xfdad_1000`, `0xfdad_3000` | `iommu@fdada000`, one bank | `GIC_SPI 112`, INTID 144 | `RK3588_PD_NPU2` |

Each register window is 4 KiB, and each MMU bank 256 bytes [source][DTS]. Core `n` takes
the clocks `ACLK_NPUn`, `HCLK_NPUn`, `SCMI_CLK_NPU` and `PCLK_NPU_ROOT`, and the resets
`SRST_A_RKNNn` and `SRST_H_RKNNn`. Its MMU takes the same AXI and AHB clocks. The
[interrupt numbers](system-reference.md#interrupt-numbers) section explains the SPI-to-INTID
offset of 32.

Each core shares its one interrupt line with its MMU. `rocket` and `rockchip-iommu` both
request the line with `IRQF_SHARED` [source][LINUX]. The MMU nodes are
`"rockchip,rk3588-iommu", "rockchip,rk3568-iommu"`, the second-generation format whose page
tables reach 40-bit physical addresses [source][LINUX].

### The register map

Within a core, the register commands address each block at a fixed offset [source][LINUX]:

| Block | Offset |
|---|---|
| PC | `0x0000` |
| CNA | `0x1000` |
| CORE | `0x3000` |
| DPU | `0x4000` |
| DPU-RDMA | `0x5000` |
| PPU | `0x6000` |
| PPU-RDMA | `0x7000` |
| DDMA | `0x8000` |
| SDMA | `0x9000` |
| `GLOBAL_OPERATION_ENABLE` | `0xf008` |

No block occupies `0x2000` to `0x2fff` (see
[the undocumented 0x2000 page](#the-undocumented-0x2000-page)). The kernel maps only the PC,
CNA and CORE windows. On the CPU side, core 0's MMU banks sit at offsets `0x9000` and
`0xa000` from the core base. Cores 1 and 2 each have one bank at `0xa000` [source][DTS].

## Power domains

The NPU's domains form one tree under the NPU voltage domain [source][DTS]:

| Domain | Parent | Powers | Clocks | QoS ports |
|---|---|---|---|---|
| `RK3588_PD_NPU` | None | The tree below it, and switches the domain supply | None | None |
| `RK3588_PD_NPUTOP` | `RK3588_PD_NPU` | Core 0 and its MMU | `HCLK_NPU_ROOT`, `PCLK_NPU_ROOT`, `CLK_NPU_DSU0`, `HCLK_NPU_CM0_ROOT` | `qos_npu0_mwr`, `qos_npu0_mro`, `qos_mcu_npu` |
| `RK3588_PD_NPU1` | `RK3588_PD_NPUTOP` | Core 1 and its MMU | `HCLK_NPU_ROOT`, `PCLK_NPU_ROOT`, `CLK_NPU_DSU0` | `qos_npu1` |
| `RK3588_PD_NPU2` | `RK3588_PD_NPUTOP` | Core 2 and its MMU | `HCLK_NPU_ROOT`, `PCLK_NPU_ROOT`, `CLK_NPU_DSU0` | `qos_npu2` |

Core 0 has no domain of its own. It sits in `PD_NPUTOP`, the parent of the other two cores'
domains. A job on core 1 or core 2 therefore powers core 0's domain as well [source][DTS].
Each core node lists one domain, and the binding allows only one. Each RK3576 core instead
lists both of that SoC's NPU domains (see
[the RK3576 driver](../rk3576/npu-reference.md#the-driver)).

The pmdomain driver flags `RK3588_PD_NPU` `need_regulator` [source][LINUX]. It therefore
takes the `domain-supply` of the `pd_npu` node, enables it before powering the domain and
disables it after. It also powers the domain off at probe, because the supply's state is
unknown then. The RK3576's `RK3576_PD_NPU` carries no such flag at the same tag.
[The RK3576 NPU rail](../rk3576/npu-reference.md#the-npu-rail) describes the trap that
leaves.

The RK1 gives `pd_npu` `domain-supply = <&vdd_npu_s0>`, and leaves the rail boot-on but not
always-on [source][DTS]. The rail therefore follows the domain, as
[the RK1's power section](turing-rk1/hardware-reference.md#power) states.

## Rails

Rockchip's hardware design guide gives the NPU two supplies [source][NOTES]:

| Supply | Figures from the design guide |
|---|---|
| `VDD_NPU` | 0.850 V, 4.00 A and 3.40 W at the 1000 MHz peak, over 200 uF of capacitance |
| `VDD_NPU_MEM` | 405 mA, separately regulatable |

The guide describes tying `VDD_NPU_MEM` to the logic rail as a concession for low-power
designs [source][NOTES]. The binding requires both `npu-supply` and `sram-supply` on every
core node. The RK1 points both at `vdd_npu_s0`, an RK8602 on i2c1 at `0x42`, 550-950 mV
[source][DTS].

Mainline `rocket` references no regulator API, so both properties are inert [source][LINUX].
Under mainline the rail sits at 0.80 V [measured][NOTES]. No mainline driver sets its voltage
[source][LINUX]. The secure firmware sets only the PLL when it changes the rate, so nothing
raises the voltage with the clock [source][TFAMAIN][NOTES].

The vendor's frequency-to-voltage map for the NPU is [source][NOTES]:

| Rate | Voltage |
|---|---|
| 300-700 MHz | 0.70 V |
| 800 MHz | 0.75 V |
| 900 MHz | 0.80 V |
| 1000 MHz | 0.85 V |

A second RK1 module, under the vendor kernel, ran the NPU at 300 MHz on 675 mV and at
1000 MHz on 787.5 mV [measured, 2026-09-27, 6.1.172][NOTES]. Both sit below that map. Its voltages therefore vary
per chip [inferred]. The SoC dtsi describes the NPU's leakage cell, `npu-leakage@28` in the
OTP at `efuse@fecc0000`, and no mainline node consumes it [source][DTS].

## The clock

The NPU's compute clock is one SCMI clock, `SCMI_CLK_NPU` (SCMI clock 6), shared by all three
cores [source][DTS]. Each core node assigns it 200 MHz, and mainline `rocket` never changes the
rate. It reads 200 MHz idle and under load [measured][NOTES]. The rated maximum is 1 GHz
[source][NOTES]. [SCMI](boot-firmware.md#scmi) describes the firmware's agent.

The secure firmware programs this clock. In mainline TF-A's RK3588 port, 200 MHz divides
GPLL, and 300 to 1000 MHz in 100 MHz steps run from the NPU's PVT-PLL [source][TFAMAIN]. The
PVT-PLL is programmed through the NPU GRF at `0xfd5a_2000`. CRU `CLKSEL_CON74` bit 0 reads 1
while the PVT-PLL drives the clock. A rate outside that table returns
`SCMI_INVALID_PARAMETERS`. Rockchip's BL31 also adjusts the NPU PVT-PLL against per-chip OTP
bounds at boot, which sets each chip's ceiling [source][NOTES].

> **Setting the rate while the NPU domains are off wedges the secure firmware.** An
> `assigned-clock-rates` of 600 MHz in the device tree hangs the boot [measured][NOTES]. The
> platform bus applies clock assignments before it attaches the power domain [source][LINUX].
> A `clk_set_rate()` at idle hangs a running system the same way [measured][NOTES].

The 200 MHz assignment in the SoC dtsi is harmless in the same place, since it selects the
GPLL path [inferred]. Set any other rate from the driver's runtime resume, after the domain
is up. Park the clock at 200 MHz in runtime suspend, before the domain goes down. Rockchip's
own driver sets the clock only while the device is runtime-active [source][NOTES].

Runtime PM runs per core, and the clock rate is shared. A driver that parks the rate in a
core's suspend callback drops it under jobs still running on the other cores [measured][NOTES].
The rate then flaps between 200 and 600 MHz. Leave the park to the last active core to
suspend.

### Operating point

600 MHz is the measured operating point. It computes bit-exact with the rail at 0.80 V
[measured, 2026-06-22, 7.1.0][NOTES]. The package stays between 48 and 57 °C under that load,
with no throttling [measured][NOTES]. 900 MHz adds no throughput over 600 MHz at that rail
[measured][NOTES].

> **The SoC hard-locked twice as the NPU domain cycled with the rate at 900 MHz.** Each lock
> came at module removal, with the device's runtime PM pinned on [measured][NOTES]. The pin
> bypasses the park in runtime suspend, so the domain cycles at the raised rate [inferred].

`PD_NPU1` and `PD_NPU2` routinely power on with the rate already raised to 600 MHz, and no
failure is recorded [measured][NOTES]. An upstream frequency-scaling series for the part
reports that an island powered up above its boot rate never acknowledges [reported][NOTES].
[NOTES] `perf/clock.md` holds the gain at each rate and the limits past 600 MHz.

## Thermal

The NPU has its own thermal zone, `npu-thermal`, on TSADC channel 6 [source][DTS]. Its one
trip is critical, at 115 °C, with no passive trip and no cooling map. At 120 °C the TSADC
resets the SoC through the CRU. Mainline has no NPU cooling device, so nothing throttles the
NPU before the critical trip shuts the system down [inferred]. On the RK1 the fan follows the
package zone, as [the RK1's cooling section](turing-rk1/hardware-reference.md#cooling)
describes.

## The `rocket` driver

`rocket` is a generic register-command submitter [source][LINUX]. Userspace builds each task's
register program in a buffer object, and the kernel points a core's PC at it and starts it.
The kernel never inspects the program. It runs any CNA -> CORE -> DPU or PPU program, not
only the operations a particular userspace emits. LLM prefill runs on it as tiled 1×1
convolutions [measured][NOTES].

The uAPI is four ioctls on one `/dev/accel/accelN` node: `CREATE_BO`, `SUBMIT`, `PREP_BO`
and `FINI_BO` [source][LINUX]. One DRM device fronts every core node that probes.

### Scheduling and address space

The driver runs one `drm_sched` scheduler per core, each taking one job at a time
[source][LINUX]. Each open file descriptor is one scheduling entity across all the cores. An
entity moves to the least-loaded core only once its queue is empty and its last job has
finished. One descriptor's jobs therefore serialize onto one core, and reaching all three
cores takes three or more descriptors [measured][NOTES].

Each descriptor gets its own IOMMU domain, whose aperture is 0 to 4 GiB [source][LINUX].
Each descriptor therefore has its own 4 GiB IOVA window. Buffers are allocated upward from
IOVA 0, and a buffer at 0 is valid [measured][NOTES]. The register program's address fields
are 32 bits wide, which matches that window [source][NOTES]. The driver sets a 40-bit DMA
mask, so the buffers behind the window can sit anywhere in DRAM [source][LINUX].

### Job execution

`rocket` runs one task per kick [source][LINUX]. `rocket_job_hw_submit()` writes the task's
address and length into the PC, arms the interrupt mask, and sets `PC_TASK_CON` with a task
count of 1. The completion handler starts the next task of the job, and signals the job's
fence after the last.

The interrupt mask arms only the DPU's two completion bits. The handler returns `IRQ_NONE`
unless one of them is set. A job that never raises a DPU completion runs into the scheduler's
500 ms timeout. The timeout handler resets the core by pulsing `SRST_A_RKNNn` and
`SRST_H_RKNNn` for 10 us.

The driver attaches the job's IOMMU domain as the job starts, and detaches it at completion
[source][LINUX]. Each attach and detach runs the MMU's stall, reset and paging handshake. That
costs 15-20 us per job on the RK3588 [measured, 2026-06-22, 7.1.0][NOTES].

The driver's runtime-PM autosuspend delay is 50 ms [source][LINUX]. Linux delivers each core's
interrupt to cpu0, a Cortex-A55, until its affinity is set [measured, 2026-06-22,
7.1.0][NOTES]. [NOTES] `perf/iova-and-multicore.md` measures what binding it to a Cortex-A76
saves.

### The vendor stack

On the vendor kernel the NPU runs Rockchip's RKNPU2 stack through its own `rknpu` driver. That
driver describes the NPU as one device that maps each core as a 64 KiB window. Its userspace
submits through that driver's own interface [source][NOTES]. Both drivers claim the
same registers and MMUs, so one kernel runs one of them [inferred].

## Register-level behavior

### The register program

A task's register program is a run of 64-bit words, each
`(target << 48) | (value << 16) | register` [source][NOTES]. The PC writes each value into the
named register. `rocket` writes `PC_REGISTER_AMOUNTS` as `(count + 1) / 2 - 1`, a length in
pairs of words [source][LINUX].

A program ends in a trailer [source][NOTES]. The trailer's second word holds a
`PC_REGISTER_AMOUNTS` op, and its last word writes the block bitmap into
`PC_OPERATION_ENABLE`. The bitmap starts the task.

### `PC_OPERATION_ENABLE` is a block bitmap

`PC_OPERATION_ENABLE` (`0x0008`) selects which blocks a task runs [measured,
2026-08-24][NOTES]. The TRM documents it as one enable bit, bit 0, with bits 1-31 reserved.
The mainline register map and Mesa's model it the same way [source][LINUX]. Both are wrong
about the upper bits on this silicon.

A convolution's trailer writes `0x1d`, and a pool's writes `0x60`. A `0x60` pool runs with bit
0 clear, and the same pool with `0x1d` in its trailer leaves its output untouched. Both words
decode under the layout of `GLOBAL_OPERATION_ENABLE` (`0xf008`) [inferred][LINUX]:

| Bit | Block |
|---|---|
| 0 | CNA |
| 2 | CORE |
| 3 | DPU |
| 4 | DPU-RDMA |
| 5 | PPU |
| 6 | PPU-RDMA |

The register self-clears, so a read returns 0 whatever was written [measured][NOTES].

### Completion interrupts

Each block raises its own completion in `PC_INTERRUPT_RAW_STATUS`, two bits per block, one for
each of its two register groups [source][LINUX][NOTES]:

| Bits | Source |
|---|---|
| 0, 1 | CNA feature load |
| 2, 3 | CNA weight load |
| 4, 5 | CNA sequencer |
| 6, 7 | CORE |
| 8, 9 | DPU |
| 10, 11 | PPU |
| 12 | DMA read error |
| 13 | DMA write error |

A pool runs on the PPU alone, so it raises the PPU pair and never the DPU pair
[measured][NOTES]. Mainline arms only the DPU pair, so every pool job waits out the 500 ms
timeout and a core reset. The output is already correct at that point. Armed for the PPU pair
instead, the same submits complete with no timeout [measured, 2026-08-24, 7.2.0][NOTES].

### Task chaining

The PC runs several tasks from one kick, with each task's trailer linking to the next
[measured, 2026-06-24][NOTES]. Three things make the chain:

1. The tasks' programs sit contiguously in one buffer.
2. Each trailer carries a `PC_BASE_ADDRESS` op holding the next task's address, and a
   `PC_REGISTER_AMOUNTS` op holding the next task's length.
3. The kernel programs the first task only, and sets `PC_TASK_CON.TASK_NUMBER` to the task
   count.

The PC then raises one completion, after the last task. `TASK_NUMBER` is 12 bits wide, so one
kick runs at most 4095 tasks [source][LINUX]. Without the `PC_BASE_ADDRESS` op the PC runs the
first task and stops, even with the length op in place [measured][NOTES].

Chained tasks run in order, and a later task reads what an earlier one wrote in the same kick
[measured, 2026-06-30][NOTES]. Independent int8 tasks chained into one kick compute bit-exact,
as fp16 tasks do [measured, 2026-09-25][NOTES]. Chained int4 is untested.

> **Do not count completed tasks through `PC_TASK_STATUS` (`0x3c`).** At interrupt time it reads
> `0x0000f000`, with its low 12 bits zero [measured][NOTES]. Rely on the single completion the
> task count gates.

`PC_TASK_DMA_BASE_ADDR` does not point the PC at a descriptor table. A task that runs with the
register at 0 stalls once the register points at a table of task descriptors [measured][NOTES].

### Register groups and the core bit

Each block carries two register groups, which the hardware alternates between tasks. Every
block's `S_POINTER` controls the alternation [source][NOTES]. Register state persists in each
group across tasks and across jobs [measured, 2026-09-25][NOTES]. With the alternation
on, a task executes the group written two tasks before it. A task that writes only some
registers therefore computes against whatever the rest of its group holds.

`rocket` writes the CNA and CORE `S_POINTER` before every task [source][LINUX]. The value is
`0xe`, which turns the alternation on, plus `0x1000_0000` times the core index. The TRM marks
bits 28 and 29 reserved [source][NOTES].

> **The core bit must match the hardware core.** Written without it or with another core's,
> the task never completes [measured, 2026-09-25][NOTES]. It raises no interrupt, leaves its
> output unwritten, and takes the 500 ms timeout, 24 of 24 times.

A register program therefore leaves the CNA and CORE `S_POINTER` to the kernel. Mainline
derives the index from probe order [source][LINUX]. A probe deferred behind a sibling's, or
a disabled core other than the last, gives a core the wrong bit [inferred].

### The undocumented 0x2000 page

**Do not read a core's `0x2000` page.** A read of core 0's `0xfdab_2000` page hard-locks the
SoC, and only a power cycle recovers [measured][NOTES]. A write there survives. Rockchip's
driver keeps DMA byte counters at offsets `0x22xx` and `0x24xx` on other SoCs, and marks them
unsupported on the RK3588 [source][NOTES]. An independent probe on a vendor kernel reports a
contained bus-error oops from the same page instead [reported][NOTES].

The DDMA block at `0x8000` reads cleanly with the domain powered. It reports configuration and
an idle bit, and no byte count [measured][NOTES]. The same independent probe reports a hang
reading it [reported][NOTES]. Read it only with the domain powered. The NPU's traffic is
measurable at the DDR controller instead (see
[DDR traffic counters](video-reference.md#ddr-traffic-counters)).

### Register access and power

Reach the NPU's registers only with its domain powered and its clocks on, as `rocket` does from
its runtime-PM and job paths. On the RK3576 a read with the AXI clock gated hard-hangs the SoC
(see [register access](../rk3576/npu-reference.md#register-access)). The RK3588 is expected to
behave the same way [expected].

### Detaching a wedged core's MMU

`rocket_reset()` detaches the core's IOMMU group before it resets the core [source][LINUX].
After a DMA fault, the timeout reaches it with the core still mid-DMA and the fault
outstanding in its MMU. The MMU cannot stall, and `rk_iommu` logs
`Enable stall request timed out` once per bank, for every faulting job [measured, 2026-07-31,
7.1.1][NOTES]. The same handshake runs again as the group returns to its default domain. A
failure there raises a `WARN` in the IOMMU core [source][NOTES].

Resetting the core first quiesces the master, and the detach then completes with no stall
timeout [measured, 2026-07-31, 7.1.1][NOTES]. The core reset also clears the MMU's page-table
base, which is why the driver drops the domain after a reset [inferred].

## Per-core fp16 results

The three cores compute fp16 identically, except on output lane 0 of each 16-channel group
[measured, 2026-09-27][NOTES]. There each core rarely writes one result slightly low, 2^-16 on
the characterized data, on its own set of inputs. The effect holds on both `rocket` and the
vendor driver, and does not move with the clock or the voltage. Two RK1 modules give identical
outputs core for core on the tile compared. The integer paths compute identically on every
core.

`rocket` places a job on a core by load. A comparison of one fp16 run against another
therefore compares core draws. An equality check on fp16 output excludes lane 0, or accepts
the one-step difference. [NOTES] `chips/rk3588.md` holds the characterization and the event
model.

## Concurrent jobs

Jobs in flight on several cores at once compute exactly on the RK3588 [measured][NOTES]. A
convolution whose tiles run concurrently on all three cores is bit-identical to the same tiles
run from one descriptor. Each core stages into its own CBUF, so nothing an encoder emits
contends between cores [inferred].

The RK3576 differs. Two jobs in flight on its two cores compute wrong answers, as
[its concurrent-jobs section](../rk3576/npu-reference.md#concurrent-jobs) describes.

## Sources

A `[source]` claim reads the mainline Linux tree at tag `v7.2.9` unless its key names another
artifact. The citation keys are:

- `[DTS]`: Linux `arch/arm64/boot/dts/rockchip/rk3588-base.dtsi` and
  `rk3588-turing-rk1.dtsi` at tag `v7.2.9`. The binding is
  `Documentation/devicetree/bindings/npu/rockchip,rk3588-rknn-core.yaml`.
- `[LINUX]`: Linux at tag `v7.2.9`. `drivers/accel/rocket/` (`rocket_drv.c`, `rocket_core.c`,
  `rocket_job.c`, `rocket_gem.c` and `rocket_registers.h`) and
  `include/uapi/drm/rocket_accel.h`. `drivers/pmdomain/rockchip/pm-domains.c`
  (`rk3588_pm_domains`, `rockchip_pd_regulator_enable`, `rockchip_pd_attach_dev`),
  `drivers/iommu/rockchip-iommu.c`, `drm_sched_entity_select_rq` in
  `drivers/gpu/drm/scheduler/sched_entity.c`, and `platform_probe` in
  `drivers/base/platform.c`.
- `[TFAMAIN]`: ARM-software/arm-trusted-firmware at commit `5103554` (2026-10-05),
  `plat/rockchip/rk3588/drivers/scmi/rk3588_clk.c` (`rk3588_npu_pvtpll_table`,
  `clk_npu_set_rate`, `clk_scmi_npu_get_rate`) and `plat/rockchip/rk3588/rk3588_def.h`.
- `[NOTES]`: rockchip-npu-notes, https://github.com/gregordinary/rockchip-npu-notes, at commit
  `b4efde2`. The notes cited are `hardware-overview.md` and `nvdla-lineage.md` for the block,
  and `chips/rk3588.md` for the design-guide figures and the per-core results. The clock and
  rail are in `perf/clock.md`, and scheduling, the IOMMU and chaining in
  `perf/iova-and-multicore.md`. `encodings/regcmd-task-model.md` holds the trailer, the
  bitmap and the register groups. `perf/pool-completion.md` holds the pool completion, and
  `perf/hw-byte-counters.md` the `0x2000` page. Each note records its own method, date and
  kernel.
