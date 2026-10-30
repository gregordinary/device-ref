# RK3588 corrections and gotchas

This document collects the claims about the RK3588 that sources state wrongly. It also
collects the traps that hold for any board on the SoC under its boot firmware. Each entry states the
fact, the evidence and the rule that follows from it. The measurements come from the
reference unit, a Turing RK1. A claim stated for the SoC rests on that board unless the
text says otherwise.

## Corrected claims

Each entry names a wrong claim and where it comes from, so that an old source does not
reintroduce the wrong value.

### The console UART is a board property

The habit of the RK3588 trees is a console on uart2 at 1,500,000 baud. Of the 24 RK3588
and RK3588S board files in mainline Linux that set `stdout-path` themselves, 21 name
`serial2:1500000n8`, Rockchip's EVB1 among them [source][DTSALL].

On the Turing RK1 the console is uart9 at 115200 baud, `stdout-path = "serial9:115200n8"`
[source][DTSALL]. Mainline u-boot's debug UART for the board is uart9 as well
[source][UBOOT]. Nothing reaches uart9 at 115200 baud before u-boot's SPL [measured,
2026-10-08, u-boot 2026.10][CONSOLE].

Take a board's console from its tree's `stdout-path`, never from the SoC's habit. The
[RK1 hardware reference](turing-rk1/hardware-reference.md#console) gives the RK1's.

### The GIC has 480 SPIs

The manual's overview gives the interrupt controller 512 SPI sources [source][TRM].

Its GIC-600 chapter configures 480 SPIs, INTIDs 32 to 511 [source][TRM]. Linux finds 480
on the reference unit [measured, 2026-10-08, 7.2.9][DMESG]. The 512 counts the whole INTID
range [inferred].

The manual's interrupt list numbers each source by INTID, PPIs included. A device tree
numbers SPIs from zero. Take an INTID from the manual as it stands, or from a tree's
`GIC_SPI` number plus 32 (see
[interrupt numbers](system-reference.md#interrupt-numbers)).

### The manual's CPU feature list describes the Cortex-A55

The CPU chapter of Rockchip's manual lists the processor's features once, for both kinds
of core [source][TRM].

The list credits the processor with AArch32 at every exception level, and an in-order
pipeline [source][TRM]. The Cortex-A76 cores run AArch32 at EL0 only [measured,
2026-10-08, u-boot 2026.10][LOADER], and the Cortex-A76 executes out of order
[reported][A76]. Both items describe the Cortex-A55 alone. The chapter itself defers to
Arm's manuals for each core [source][TRM].

Take a core's features from its own ID registers, or from Arm's manual for that core. Take
the core counts, cache sizes and revisions from Rockchip's table (see
[the processor reference](cpu-reference.md#id-registers)).

## Firmware

### Group 0 belongs to the firmware

Rockchip's BL31 keeps the GIC's group 0 at EL3 (see
[security states and groups](system-reference.md#security-states-and-groups)). With
`SCR_EL3.FIQ` set, an EL2 access to a group 0 system register traps to EL3 [source][GIC]. On the
reference unit, a read of `ICC_IGRPEN0_EL1` at EL2 is followed at once by the SPL's banner
[measured, 2026-10-09, u-boot 2026.10][PROBE]. Mainline u-boot's configuration for the
board enables no watchdog [source][UBOOT], so the read itself resets the board [inferred].
The other group 0 registers trap under the same rule [source][GIC], and are expected to
reset it too [expected].

The trap catches a kernel that takes its timer as an FIQ. Such a kernel makes the timer's
PPI group 0 and writes `ICC_BPR0_EL1` and `ICC_IGRPEN0_EL1`. On the RK3588 the first write
is ignored, and the second resets the board before the kernel prints anything [inferred].

Take every interrupt as group 1, signalled as an IRQ. The virtual timer, INTID 27, reaches
the boot core that way [measured, 2026-10-09, u-boot 2026.10][PROBE]. The firmware's own
setting gives every core the same route [inferred].

### The PMU's wake sources belong to the firmware

The PMU's wake register refuses the non-secure world. A read of `PMU_WAKEUP_INT_CON` at
EL2 ends in a synchronous external abort [measured, 2026-10-10, u-boot 2026.10][SLEEP].
Mainline TF-A's RK3588 port marks the PMU0 and PMU1 register blocks secure
[source][TFAMAIN]. A kernel can neither read which wake sources are armed nor set them
[inferred].

Set them through the firmware's `SIP_SUSPEND_MODE` call. Without it, BL31's defaults arm an
interrupt to the calling core and GPIO0's interrupt (see
[Rockchip's BL31 and the wake sources](system-sleep.md#rockchips-bl31-and-the-wake-sources)).

### The calling core's own interrupts wake a system suspend

Under BL31's defaults, an interrupt the GIC forwards to the core that calls `SYSTEM_SUSPEND`
wakes the system. The core's own SGIs and PPIs count [inferred]. A kernel whose source leaves
its virtual timer's PPI and an SGI enabled made the call. It counted 174 ms asleep, its 3 s
alarm unrung [measured, 2026-10-11, u-boot 2026.10][EARLYWAKE]. Rebuilt to disable every core's
SGIs and PPIs as it goes down, the kernel woke after 3 s with the alarm rung. Its timer had
been due 9.5 ms after the core went down [measured, 2026-10-11, u-boot 2026.10][TIMERWAKE].

Disable every SGI and PPI of the calling core before the call, and leave only the wake
alarm's SPI routed to it (see
[the calling core's own interrupts](system-sleep.md#the-calling-cores-own-interrupts)).

### A starting core loses its earlier GIC setup

The firmware sets a core's redistributor up again as `CPU_ON` powers the core. TF-A 2.3's
common Rockchip code disables every SGI and PPI of the core there, and makes each
non-secure one group 1 [source][TFA]. Rockchip's BL31 comes from its own tree of TF-A 2.3,
and does the same [inferred]. An enable that another core wrote into the redistributor before the start
is lost, the IPI's included [inferred].

A kernel whose starting core waits for an IPI enabled on its behalf waits for good. Enable
the SGIs and PPIs a core needs on that core, as it starts (see
[starting a core](boot-firmware.md#starting-a-core)).

## Processors

### Every core needs the repeated TLBI

Linux's option for the Cortex-A55's broadcast-TLBI erratum, 2441007, is off by default,
and its option for the Cortex-A76's, 4193800, is on [source][LINUXERRATA]. That suggests a
workaround for the Cortex-A76s alone.

Both errata are open on the RK3588's revisions, and Arm gives one workaround for both
[source][SDEN55][SDEN76]. The fault lies in the core that stores, which another core's
broadcast TLBI fails to fence. The core that invalidates therefore needs the extra
`TLBI VALE1IS, XZR` and `DSB ISH` whatever its own kind [inferred].

Follow the DSB that completes every broadcast stage 1 invalidation with that pair, on
every core (see [errata](cpu-reference.md#errata)).

### The L3 is 3 MiB, not the 4 MiB the registers give

Every core's `CCSIDR_EL1` describes the shared L3 as 16 ways of 4,096 sets of 64 bytes,
4 MiB [measured, 2026-10-09, u-boot 2026.10][PROBE]. Code that sizes the cache from the
registers takes it for 4 MiB.

The manual gives 3 MiB, and the mainline tree gives 3 MiB in 4,096 sets [source][TRM][DTS].
Arm's DSU builds a 3 MB L3 as a 4 MB one with a quarter of its ways left out, 12 in each
set. It reports the 4 MB geometry to software [source][DSU]. Linux reports the tree's
figure, because it takes cache sizes from the tree [source][LINUX].

The cache behaves as 3 MiB. A random chain of loads slows steeply between 3 and 3.5 MiB of
working set, on a Cortex-A55 and on a Cortex-A76 alike. Two RK1 modules show the same step
[measured, 2026-10-23, 7.2.9][LATENCY].

Walk set and way maintenance by the registers' geometry, and report the cache's size from
the manual or the tree (see [caches](cpu-reference.md#caches)).

### The grade is the board file's choice

Mainline reads no fuse to pick its operating points. The industrial RK3588J's derated table
exists, in `rk3588j.dtsi`, and a board gets it only by including that file [source][OPP].
The vendor kernel reads the grade from the OTP's specification serial number instead
[source][VENDOR].

A board built on `rk3588.dtsi` therefore runs an RK3588J at up to 1800 MHz on the
Cortex-A55s and 2400 MHz on the Cortex-A76s. Its own table stops at 1296 and 1608 MHz
[inferred].

Include `rk3588j.dtsi` in the board file of any board that carries an RK3588J (see
[operating points](cpu-reference.md#operating-points)).

### 2400 MHz is a request, not a frequency

Mainline offers every RK3588 its Cortex-A76s at 2400 MHz [source][OPP], and cpufreq then
reports 2400 MHz. The vendor kernel offers 2400 MHz only to a die whose process monitor reads
at the top level, 7. Other dies stop at 2256 to 2352 MHz [source][VENDOR].

The firmware clocks the Cortex-A76s from a ring oscillator above 1608 MHz, with one ring
length for every rate up to 2400 MHz [source][TFAMAIN]. The voltage then sets the frequency.
At the 2400 MHz point, the reference unit's Cortex-A76s delivered 2.32 GHz, and a second RK1
module's 2.14 GHz [measured, 2026-10-23, 7.2.9][FREQ]. cpufreq and the SCMI clock read
2400 MHz on both.

The point's rail sits 12.5 mV above the 2208 MHz point's, and it gains about 1 % on both
dies. Count cycles before quoting a core's clock, and expect the delivered clock to differ
from die to die. A board that sees instability at the top point can drop the 2400 MHz entry in its own
tree (see [delivered frequency](cpu-reference.md#delivered-frequency)).

### The Cortex-A55 policy clocks the L3

The firmware sets the DSU's clock to the Cortex-A55 cluster's rate [source][TFAMAIN]. The DSU
carries the L3 and every core's path to memory. A governor that idles the Cortex-A55s
therefore slows the Cortex-A76s' memory traffic too.

With the Cortex-A76s at their top point, four of them read 16.9 GB/s with the Cortex-A55s at
1008 MHz, and 27.8 GB/s at 1800 MHz. A second module shows the same gap [measured,
2026-10-23, 7.2.9][DSUSWEEP]. The vendor
kernel holds the Cortex-A55s at no less than four fifths of the busiest Cortex-A76 cluster's
rate [source][VENDOR]. Mainline sets no such floor.

Raise the Cortex-A55 policy's floor for memory-bound work on the Cortex-A76s (see
[the DSU clock](cpu-reference.md#the-dsu-clock-follows-the-cortex-a55s)).

## Media blocks

### A stalled register access hangs the whole SoC

On a Rock 5B, decode on both rkvdec cores through a vendor-derived driver stopped the
whole SoC four times between 2026-07-29 and 2026-08-01 [reported][YSP]. Each time the log
stopped mid-line, with no oops and nothing in pstore. Panic-on-hang settings armed
beforehand never fired, and only a hardware watchdog brought the board back. With the
second core unbound, the same case passed.

The cause is taken to be an access to a block whose clock or power is off, or which is
held in reset [inferred]. Such an access stalls the bus and never returns, so no CPU is
left to report it. The access itself was never caught. On the RK3576, a read from the NPU
with its AXI clock gated hangs the SoC the same way (see [register
access](../rk3576/npu-reference.md#register-access)).

Reach a media block's registers only with its power domain on, its clocks on and its
resets released. While developing such a driver, keep a hardware watchdog armed and a
serial console attached, because nothing else records the hang.

## Sources

Citation keys:

- `[TRM]`: Rockchip, *RK3588 Technical Reference Manual*, Rev 1.0, Part 1. The overview
  carries the interrupt list in Table 1-3. Chapter 3 is the CPU chapter, and Table 11-1
  gives the GIC-600 configuration.
- `[DTS]`: Linux `arch/arm64/boot/dts/rockchip/rk3588-base.dtsi` at tag `v7.2.9`.
- `[DTSALL]`: the RK3588 and RK3588S board files, `rk3588*.dts`, in Linux
  `arch/arm64/boot/dts/rockchip/` at tag `v7.2.9`. Of the 43 files, 24 set
  `stdout-path` themselves. Two name `serial2:115200n8`, `rk3588-jaguar.dts` and
  `rk3588-tiger-haikou.dts`, and `rk3588-turing-rk1.dts` names `serial9:115200n8`.
- `[UBOOT]`: u-boot at commit `ece349a`: `configs/turing-rk1-rk3588_defconfig`, whose
  `CONFIG_DEBUG_UART_BASE` is `0xFEBC0000`, and `arch/arm/mach-rockchip/Kconfig`. Neither
  enables nor implies a watchdog.
- `[TFAMAIN]`: ARM-software/arm-trusted-firmware at commit `5103554` (2026-10-05):
  `plat/rockchip/rk3588/drivers/secure/secure.c` (`sgrf_init`), and
  `plat/rockchip/rk3588/drivers/scmi/rk3588_clk.c` (`rk3588_cpub0_pvtpll_table`,
  `clk_scmi_cpul_set_rate`).
- `[TFA]`: ARM-software/arm-trusted-firmware at tag `v2.3`, which carries no RK3588 port:
  its common Rockchip code, `plat/rockchip/common/plat_pm.c`
  (`rockchip_pwr_domain_on_finish`), and `drivers/arm/gic/v3/gicv3_main.c` and
  `gicv3_helpers.c` (`gicv3_rdistif_init`).
- `[DSU]`: Arm, *DynamIQ Shared Unit Technical Reference Manual*, r4p1, 100453_0401_05:
  the `L3_CACHE_SIZE` option and the section on implementing a 1.5MB or 3MB L3 cache.
- `[LINUX]`: Linux at tag `v7.2.9`: `arch/arm64/kernel/cacheinfo.c` (`ci_leaf_init`).
- `[LINUXERRATA]`: Linux at tag `v7.2.9`, commit `5fce161`: `arch/arm64/Kconfig`
  (`ARM64_ERRATUM_2441007`, `ARM64_ERRATUM_4118414`) and `arch/arm64/kernel/cpu_errata.c`.
- `[SDEN55]`: Arm, *Cortex-A55 Software Developer Errata Notice*, SDEN-859338, issue 16.0,
  erratum 2441007.
- `[SDEN76]`: Arm, *Cortex-A76 Software Developer Errata Notice*, SDEN-885749, issue 34,
  erratum 4193800.
- `[A76]`: Wikipedia, "ARM Cortex-A76" (`https://en.wikipedia.org/wiki/ARM_Cortex-A76`).
- `[GIC]`: Arm, *GIC Architecture Specification, GICv3 and GICv4*, IHI 0069H.b: the
  accessibility pseudocode of the group 0 CPU interface registers.
- `[OPP]`: Linux `arch/arm64/boot/dts/rockchip/` at tag `v7.2.9`: `rk3588-opp.dtsi`,
  `rk3588j.dtsi`, and the includes of `rk3588.dtsi`.
- `[VENDOR]`: the vendor kernel, rockchip-linux/kernel at commit `b4ef083dc`, branch
  `develop-6.1`: `drivers/cpufreq/rockchip-cpufreq.c` (`rk3588_get_soc_info`),
  `drivers/soc/rockchip/rockchip_opp_select.c` (`rockchip_set_opp_supported_hw`), and the
  CPU tables of `arch/arm64/boot/dts/rockchip/rk3588s.dtsi`. For the floor between clusters,
  `rockchip_cpufreq_update_bus_req` and `cpu_to_bus_freq` in `rockchip-cpufreq.c`.
- `[YSP]`: Yi Ding, rock-5b-ysp, https://github.com/yisding/rock-5b-ysp, at commit
  `69a3abbf`, a Rock 5B project whose documentation is licensed CC-BY-SA-4.0. Its record of
  the four hangs, under a vendor-derived 6.18 kernel. The facts are restated here, not
  quoted.

Captures from the reference unit, kept with its evidence outside this repository:

- `[CONSOLE]`: the console from power-on to u-boot 2026.10's prompt, 2026-10-08.
- `[DMESG]`: the kernel log of Linux 7.2.9, 2026-10-08.
- `[LOADER]`: a bare-metal program's report of each core's ID registers, run from u-boot
  2026.10 with `go`, 2026-10-08.
- `[PROBE]`: a bare-metal probe of the EL2, GIC and cache state, run from u-boot 2026.10
  with `go`, 2026-10-09.
- `[SLEEP]`: a bare-metal sleep probe run from u-boot 2026.10, 2026-10-10.
- `[EARLYWAKE]`: a run of open-source XNU, booted from u-boot 2026.10 by a bare-metal loader,
  2026-10-11. The kernel calls `SYSTEM_SUSPEND` with the PMU timer's alarm set 3 s on. Its
  source leaves the calling core's SGI 1 and PPI 27 enabled, and the run prints no GIC state.
- `[TIMERWAKE]`: a rebuilt kernel, run the same way the same day. It disables each core's SGIs
  and PPIs as the core goes down.

Captures from the reference unit, with BL31 v1.56, and from a second Turing RK1 module, with
BL31 v1.51. Both ran Linux 7.2.9, and the captures are kept outside this repository. The
[processor reference](cpu-reference.md#sources) describes each method:

- `[LATENCY]`: a pointer chase pinned to a Cortex-A55 and to a Cortex-A76, 2026-10-23.
- `[FREQ]`: each Cortex-A76 operating point pinned in turn, with the core's cycles counted
  and a chain of dependent adds timed, 2026-10-23.
- `[DSUSWEEP]`: the Cortex-A55 policy stepped through its operating points with streaming
  reads on the Cortex-A76s, 2026-10-23.
