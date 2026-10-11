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

Walk set and way maintenance by the registers' geometry, and report the cache's size from
the manual or the tree (see [caches](cpu-reference.md#caches)).

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
  `plat/rockchip/rk3588/drivers/secure/secure.c` (`sgrf_init`).
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
