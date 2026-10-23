# RK3588 processor reference

This document covers the RK3588's application cores. Its reader writes a bootloader, a
kernel port or a device tree for any board on the SoC. It states the cores,
their revisions, ID registers and numbering, and their caches as the manual and the
registers give them. It also covers the cache maintenance the cores need, the coherency of
the SoC's devices, the errata that apply, and the cores' operating points.

The document is SoC-level. Register readings come from the reference unit, a Turing RK1,
read on each core by a bare-metal program started from mainline u-boot 2026.10. Linux
readings come from the same unit under mainline Linux 7.2.9. Configuration facts come
from Rockchip's technical reference manual [source][TRM].

## Cores and revisions

The RK3588 carries four Cortex-A55 cores and four Cortex-A76 cores, behind a DynamIQ Shared
Unit [source][TRM]. The manual names the revisions: Cortex-A55 r2p0, Cortex-A76 r4p0, and
DSU r4p1 [source][TRM]. It states Armv8.2-A for both cores.

The Cortex-A76 implements Armv8.2-A plus three later features [reported][A76]:

- The LDAPR instructions, from Armv8.3-A
- The dot product instructions, from Armv8.4-A
- SSBS and the speculation barriers, from Armv8.5-A

The manual's list of processor features is generic. It credits both cores with AArch32 at
every exception level and an in-order pipeline, and both items describe the Cortex-A55
alone (see
[the manual's CPU features](corrections-and-gotchas.md#the-manuals-cpu-feature-list-describes-the-cortex-a55)).

The mainline tree gives the Cortex-A55 a `capacity-dmips-mhz` of 530 and the Cortex-A76
one of 1024 [measured, 2026-10-08, 7.2.9][FDT].

## Core numbering

The tree lists eight CPUs in two clusters [source][DTS]:

| Nodes | Core | `reg` |
|---|---|---|
| `cpu_l0` to `cpu_l3` | Cortex-A55 | `0x000`, `0x100`, `0x200`, `0x300` |
| `cpu_b0` to `cpu_b3` | Cortex-A76 | `0x400`, `0x500`, `0x600`, `0x700` |

On an Armv8 system, a CPU node's `reg` matches the affinity bits of the core's
`MPIDR_EL1` [source][CPUS]. Aff1 therefore numbers the cores 0 to 7, and Aff0 is zero on
each. `MPIDR_EL1` reads `0x8100_0000` with the affinity on every core, so the MT bit is
set [measured, 2026-10-08, u-boot 2026.10][LOADER].

> **The cores are numbered in Aff1, not Aff0.** Code that tells cores apart by Aff0 alone
> sees eight cores with the same number.

The boot core is the core at affinity zero, a Cortex-A55. Each of the other seven starts
at EL2 through PSCI `CPU_ON` (see [starting a core](boot-firmware.md#starting-a-core)).

## ID registers

The four cores of each kind read alike [measured, 2026-10-08, u-boot 2026.10][LOADER]:

| Register | Cortex-A55, `0x000` to `0x300` | Cortex-A76, `0x400` to `0x700` |
|---|---|---|
| `MIDR_EL1` | `0x412f_d050`: Arm, part `0xd05`, r2p0 | `0x414f_d0b0`: Arm, part `0xd0b`, r4p0 |
| `ID_AA64PFR0_EL1` | `0x1111_2222` | `0x1100_0000_1111_1112` |
| `ID_AA64PFR1_EL1` | `0x10` | `0x10` |
| `ID_AA64ISAR1_EL1` | `0x10_0001` | `0x10_0001` |
| `ID_AA64ISAR2_EL1` | 0 | 0 |
| `ID_AA64MMFR0_EL1` | `0x10_1122` | `0x10_1122` |
| `ID_AA64MMFR1_EL1` | `0x1021_2122` | `0x1021_2122` |

The two kinds differ in AArch32. The Cortex-A55 runs it at every exception level, and the
Cortex-A76 at EL0 alone. Both MIDR revisions agree with the manual's.

These fields read the same on both kinds:

| Field | Value | Meaning |
|---|---|---|
| `ISAR1` APA, API, GPA, GPI and `ISAR2` APA3, GPA3 | 0 | No pointer authentication |
| `PFR1.SME` | 0 | No SME |
| `PFR1.SSBS` | 1 | SSBS, without SSBS2's `SSBS` register |
| `MMFR1.PAN` | 2 | PAN2, not PAN3 |
| `MMFR0.TGran16` | 1 | The 16 KiB translation granule |
| `MMFR0.PARange` | 2 | 40-bit physical addresses |
| `PFR0.GIC` | 1 | The GICv3 CPU interface's system registers |

`RNDR` gives the boot core, a Cortex-A55, no random number [measured, 2026-10-09, u-boot
2026.10][HANDOFF]. Linux reports no `rng` capability, so at least one core lacks FEAT_RNG
[inferred]. The Cortex-A76's `ID_AA64ISAR0_EL1` has not been read. The SoC's TRNG is the hardware source (see
[the TRNG](system-reference.md#trng)).

Under Linux, the reference unit reports these hardware capabilities [measured, 2026-10-08,
7.2.9][CPUINFO]:

```
fp asimd evtstrm aes pmull sha1 sha2 crc32 atomics fphp asimdhp cpuid asimdrdm lrcpc dcpop asimddp
```

No `paca`, `pacg`, `sve`, `sme` or `bti` appears. The list is system-wide and omits what
any core lacks [inferred], and the per-core readings agree with it. Linux also detects the
Virtualization Host Extensions [measured, 2026-10-08, 7.2.9][DMESG].

## Caches

The manual gives each core's caches [source][TRM]:

| Cache | Cortex-A55 | Cortex-A76 |
|---|---|---|
| L1 instruction, per core | 32 KiB | 64 KiB |
| L1 data, per core | 32 KiB | 64 KiB |
| L2, per core | 128 KiB | 512 KiB |
| L3, shared by all eight | 3 MiB | 3 MiB |

Each core's `CCSIDR_EL1` gives the geometry below, with 64-byte lines at every level
[measured, 2026-10-09, u-boot 2026.10][PROBE]:

| Cache | Cortex-A55 | Cortex-A76 |
|---|---|---|
| L1 instruction | 4 ways, 128 sets: 32 KiB | 4 ways, 256 sets: 64 KiB |
| L1 data | 4 ways, 128 sets: 32 KiB | 4 ways, 256 sets: 64 KiB |
| L2 | 4 ways, 512 sets: 128 KiB | 8 ways, 1024 sets: 512 KiB |
| L3 | 16 ways, 4096 sets: 4 MiB | 16 ways, 4096 sets: 4 MiB |

The registers agree with the manual on the L1 and L2 caches, and not on the L3. The manual
gives 3 MiB, and the mainline tree gives 3 MiB in 4,096 sets [source][TRM][DTS].

The DSU carries the L3. It builds a 3 MB cache as a 4 MB one with a quarter of its ways
left out, 12 in each set in place of 16. It reports the 4 MB geometry to software
[source][DSU]. Each core reads the geometry Arm's Cortex-A55 manual gives for a 4 MB L3
[source][A55TRM]. The cache holds 3 MiB, and the registers' figures are the ones set and
way maintenance walks.

Linux on arm64 takes only each cache's level and type from the hardware, and takes the
sizes from the device tree [source][LINUX]. Its report of the L3, 12 ways in 3 MiB, is
therefore the tree's [measured, 2026-10-08, 7.2.9][CACHEINFO].

## Cache maintenance

The cores report their caches' type and maintenance needs in `CTR_EL0` and `CLIDR_EL1`
[measured, 2026-10-09, u-boot 2026.10][PROBE]:

| Register | Cortex-A55 | Cortex-A76 |
|---|---|---|
| `CTR_EL0` | `0x8444_8004`: L1 instruction cache VIPT, IDC 0, DIC 0 | `0x9444_c004`: L1 instruction cache PIPT, IDC 1, DIC 0 |
| `CLIDR_EL1` | `0xc300_0123` | `0xc300_0123` |

`CLIDR_EL1` describes separate L1 instruction and data caches, a unified L2 and a unified
L3. It puts the level of coherence at 3, and LoUU and LoUIS at 0. No cache level therefore
lies inside the point of unification. Instruction coherence needs no data cache clean on
either core, and Linux reports as much for the system as a whole [measured, 2026-10-08,
7.2.9][DMESG]. Neither core sets `DIC`, so instruction coherence does need the instruction
cache invalidated.

Linux treats every VIPT instruction cache as one that can alias [source][LINUX]. Each of
the Cortex-A55's four instruction ways spans 8 KiB [measured, 2026-10-09, u-boot
2026.10][PROBE]. Under 4 KiB pages the
cache's index reaches past the page offset, and under 16 KiB pages it does not, so it
cannot alias there [inferred].

Maintenance by set and way up to the level of coherence visits every line of each data or
unified cache. The registers' figures give these counts, the L3's four absent ways
included [inferred]:

| Cache | Cortex-A55 lines | Cortex-A76 lines |
|---|---:|---:|
| L1 data | 512 | 1,024 |
| L2 | 2,048 | 8,192 |
| L3 | 65,536 | 65,536 |
| Total | 68,096 | 74,752 |

Set and way maintenance is the one way to empty the data caches without knowing what they
hold. A bare-metal program that mainline u-boot starts with its caches on needs it before
the caches go off (see
[entering a program from mainline u-boot](boot-firmware.md#entering-a-program-from-mainline-u-boot)).

## Coherency

No node of the mainline tree carries `dma-coherent`. The GIC and its two ITS blocks carry
`dma-noncoherent` [measured, 2026-10-08, 7.2.9][FDT]. No device is described as snooping
the cores' caches.

A driver therefore cleans the data cache before a device reads memory, and invalidates it
before a core reads what a device wrote [inferred]. The GIC's own case is in
[the interrupt controller](system-reference.md#interrupt-controller).

## Errata

Linux 7.2.9 turns on its workaround for broadcast TLB invalidation on the reference unit,
`Broken broadcast TLBI completion` [measured, 2026-10-08, 7.2.9][DMESG]. Its list of
affected cores carries every Cortex-A76, for Arm's erratum 4193800, under an option that
is on by default. It carries every Cortex-A55 too, for erratum 2441007, under an option
that is off by default [source][LINUXERRATA]. One affected core turns the workaround on for
every core. The reference unit's message therefore needs its Cortex-A76s alone [inferred].

Arm's notices describe the two errata alike [source][SDEN55][SDEN76]:

| | Cortex-A55, 2441007 | Cortex-A76, 4193800 |
|---|---|---|
| Notice | SDEN-859338, issue 16.0 | SDEN-885749, issue 34 |
| Category | B, rare | B |
| Revisions affected | r0p0, r0p1, r1p0 and r2p0. Open | r0p0 to r4p1. Open |
| The RK3588's revision | r2p0, affected | r4p0, affected |

The fault lies in the core that stores. Another core invalidates the store's translation
with a broadcast TLBI, and completes it with a DSB. The store can still be in flight after
that DSB [source][SDEN55][SDEN76].

On the Cortex-A76 the TLBI must cover a byte of the store, and apply to stage 1
information. Only stores are affected, not loads, atomics, exclusives or a translation
walk's own writes [source][SDEN76]. On the Cortex-A55 the other core changes the store's
descriptor by break-before-make [source][SDEN55]. Arm lists the Cortex-A76's erratum as
CVE-2025-10263, since the late store can write through a translation that no longer allows
it [source][SDEN76].

Both notices give the same workaround. The DSB that completes the invalidation is followed
by one more TLBI and a DSB that completes it. That TLBI is broadcast to the shareability
domain that holds the storing core. `TLBI VALE1IS, XZR` then `DSB ISH` suffice, whatever
TLBIs came before [source][SDEN55][SDEN76]. Neither notice names a further condition, and
a local TLBI is outside both.

> **Every core that issues a broadcast TLBI needs the pair, a Cortex-A55 as much as a
> Cortex-A76.** The fault lies in the core that stores, not in the one that invalidates
> [inferred]. Linux 7.2.9 adds the pair to every completion of a broadcast stage 1
> invalidation [source][LINUXERRATA].

The Cortex-A76's older erratum 1286807, which Linux also meets with a repeated TLBI,
affects r0p0 to r3p0 and was fixed in r3p1. The RK3588's r4p0 lacks it [source][SDEN76].

Linux reports more on the reference unit [measured, 2026-10-08, 7.2.9][DMESG]:

- `ARM errata 1165522, 1319367, or 1530923`: a speculative address translation that can use
  the wrong translation regime. The Cortex-A55 r2p0 has 1530923. The Cortex-A76's 1165522
  affects r0p0 to r2p0 only [source][SDEN76]. Linux applies the workaround only as it
  switches to or from a virtual machine, at EL2 [source][LINUXERRATA].
- `SSBS not fully self-synchronizing`: the Cortex-A76's erratum 3324349. A write of 0 to
  `PSTATE.SSBS` by MSR does not take effect at once [source][SDEN76].
- `Spectre-v4` and `Spectre-BHB`, reported as Linux brings up the first Cortex-A76. Linux
  takes the Cortex-A55 as unaffected by Spectre-BHB. It clears the Cortex-A76's branch
  history with a loop of 24 branches on each entry from EL0 [source][LINUXERRATA].

## Operating points

Mainline describes one voltage for each CPU frequency, in `rk3588-opp.dtsi`, which both
`rk3588.dtsi` and `rk3588s.dtsi` include [source][OPP]. The two Cortex-A76 clusters take
identical tables:

| Frequency | Cortex-A55 | Cortex-A76 |
|---|---|---|
| 1008 MHz | 675 mV | |
| 1200 MHz | 712.5 mV | 675 mV |
| 1416 MHz | 762.5 mV | 725 mV |
| 1608 MHz | 850 mV | 762.5 mV |
| 1800 MHz | 950 mV | 850 mV |
| 2016 MHz | | 925 mV |
| 2208 MHz | | 987.5 mV |
| 2400 MHz | | 1,000 mV |

Each entry's maximum is 950 mV on the Cortex-A55s and 1,000 mV on the Cortex-A76s. The
Cortex-A55 table marks 1416 MHz as the suspend frequency [source][OPP].

Mainline reads no fuse to choose or adjust these voltages. The OTP at `efuse@fecc0000`
carries a CPU code, a CPU version and a leakage cell for each cluster [source][DTS]. No
RK3588 device tree consumes them [source][OPP].

The vendor kernel chooses per part and per die, in two steps [source][VENDOR]:

1. The OTP's specification serial number names the grade. `0xd` is the RK3588M and `0xa`
   the RK3588J, and a customer-demand cell adds a further bin. The grade selects which
   operating points exist, through `opp-supported-hw`.
2. Each cluster's PVTM, an on-die process monitor, gives a reading that a per-cluster table
   maps to a level. The level selects a voltage column, `opp-microvolt-L<n>`. The
   Cortex-A55 table runs to level 6, and the Cortex-A76 tables to level 7.

At every frequency the two trees share, mainline's voltage equals the vendor's default
`opp-microvolt` column [source][VENDOR][OPP]. The vendor tables also carry frequencies
mainline leaves out. Every cluster gets 408 to 816 MHz, and the Cortex-A76s get 1008 MHz.

Above 2208 MHz the vendor kernel gives each die one top Cortex-A76 frequency, by its PVTM
level, all at 1,000 mV [source][VENDOR]:

| Level | Top frequency |
|---|---|
| 7 | 2400 MHz |
| 3 or 6 | 2352 MHz |
| 2 or 5 | 2304 MHz |
| 0, 1 or 4 | 2256 MHz |

The level reaches the table as the second word of `opp-supported-hw`, one bit per level.
Mainline offers 2400 MHz to every die [source][OPP]. A die below level 7 therefore runs
faster under mainline than the vendor kernel allows it [inferred].

One Rock 5B die reads bin 0. Its PVTM gives level 5 on the Cortex-A55s, from a reading of
1525, and level 7 on both Cortex-A76 clusters, from 1782 and 1778 [reported][YSP]. Under
the vendor kernel its Cortex-A55 rail held 887.5 mV at 1800 MHz [reported][YSP]. That is
level 5's voltage, where mainline sets 950 mV [source][VENDOR]. Level 7 sets the
Cortex-A76s 50 to 87.5 mV below mainline from 1416 to 2208 MHz, and equal at 1200 and
2400 MHz [source][VENDOR]. Mainline therefore runs such a die at a higher voltage across
the middle of its range [inferred].

The industrial grade has a table of its own. `rk3588j.dtsi` replaces the CPU and GPU
tables with a derated set [source][OPP]:

- Cortex-A55: 1200 and 1296 MHz, at 750 and 775 mV.
- Cortex-A76: 1200, 1416 and 1608 MHz, at 750 to 787.5 mV.
- GPU: 300 to 700 MHz, at 750 mV.

A board file opts in by including `rk3588j.dtsi` in place of `rk3588.dtsi`, and one
in-tree board does. Nothing checks the fuse, so an RK3588J on a board that includes
`rk3588.dtsi` runs the commercial table (see [the grade is the board file's
choice](corrections-and-gotchas.md#the-grade-is-the-board-files-choice)).

## Sources

Citation keys:

- `[TRM]`: Rockchip, *RK3588 Technical Reference Manual*, Rev 1.0, Part 1, chapter 3: the
  CPU configuration in Table 3-1, and section 3.3.
- `[DTS]`: Linux `arch/arm64/boot/dts/rockchip/rk3588-base.dtsi` at tag `v7.2.9`.
- `[CPUS]`: Linux `Documentation/devicetree/bindings/arm/cpus.yaml` at tag `v7.2.9`,
  property `reg`.
- `[LINUX]`: Linux at tag `v7.2.9`: `arch/arm64/kernel/cpuinfo.c` (the VIPT rule),
  `arch/arm64/kernel/cacheinfo.c` (`ci_leaf_init`, which sets a cache's level and type
  alone), and `arch/arm64/kernel/cpufeature.c` (`ARM64_HAS_CACHE_IDC`).
- `[LINUXERRATA]`: Linux at tag `v7.2.9`, commit `5fce161`. `arch/arm64/Kconfig`:
  `ARM64_ERRATUM_2441007`, with no default, and `ARM64_ERRATUM_4118414`, default y, whose
  list names erratum 4193800. `arch/arm64/kernel/cpu_errata.c`: the MIDR ranges each option
  adds to `ARM64_WORKAROUND_REPEAT_TLBI`, of type `ARM64_CPUCAP_LOCAL_CPU_ERRATUM`.
  `arch/arm64/include/asm/tlbflush.h`: `__tlbi_sync_s1ish`. For the other errata,
  `arch/arm64/kernel/proton-pack.c` and the uses of `ARM64_WORKAROUND_SPECULATIVE_AT`, all
  under `arch/arm64/kvm/`.
- `[OPP]`: the RK3588 device trees in Linux `arch/arm64/boot/dts/rockchip/` at tag `v7.2.9`:
  `rk3588-opp.dtsi`, `rk3588j.dtsi`, and the includes of `rk3588.dtsi` and `rk3588s.dtsi`.
  `rk3588-edgeble-neu6b-io.dts` is the one board file that includes `rk3588j.dtsi`. No
  `rk3588*` file carries an `nvmem-cells` property.
- `[VENDOR]`: the vendor kernel, rockchip-linux/kernel at commit `b4ef083dc`, branch
  `develop-6.1`, Linux 6.1.141. `drivers/cpufreq/rockchip-cpufreq.c`
  (`rk3588_get_soc_info`), `drivers/soc/rockchip/rockchip_opp_select.c`
  (`rockchip_set_opp_supported_hw`), and the `cluster0_opp_table` to `cluster2_opp_table`
  nodes of `arch/arm64/boot/dts/rockchip/rk3588s.dtsi`, with their
  `rockchip,pvtm-voltage-sel` tables.
- `[YSP]`: Yi Ding, rock-5b-ysp, https://github.com/yisding/rock-5b-ysp, at commit
  `69a3abbf`, a Rock 5B project whose documentation is licensed CC-BY-SA-4.0. Its finding of
  2026-07-27 on the voltage levels, read from the vendor kernel 6.1.115's log and live
  regulators. The facts are restated here, not quoted.
- `[SDEN55]`: Arm, *Cortex-A55 Software Developer Errata Notice*, SDEN-859338, issue 16.0,
  2026-01-30, erratum 2441007.
- `[SDEN76]`: Arm, *Cortex-A76 Software Developer Errata Notice*, SDEN-885749, issue 34,
  2026-06-25: errata 4193800, 1286807, 1165522 and 3324349.
- `[DSU]`: Arm, *DynamIQ Shared Unit Technical Reference Manual*, r4p1, 100453_0401_05.
  The `L3_CACHE_SIZE` implementation option, the L3 cache among the components, and the
  section on implementing a 1.5MB or 3MB L3 cache.
- `[A55TRM]`: Arm, *Cortex-A55 Core Technical Reference Manual*, 100442_0200_03: the
  `CCSIDR_EL1` encodings, whose 4MB L3 row reads `0x71ff_e07a`.
- `[A76]`: Wikipedia, "ARM Cortex-A76", which cites Arm's Cortex-A76 documentation set
  (`https://en.wikipedia.org/wiki/ARM_Cortex-A76`). Arm's technical reference manual is the
  primary source and has not been read.

Captures from the reference unit, kept with its evidence outside this repository:

- `[LOADER]`: a bare-metal program's report of each core's ID registers, run twice from
  u-boot 2026.10 with `go`, 2026-10-08. The capture path dropped characters. Each value
  comes from lines that kept all 16 digits, and agrees across both runs and the four cores
  of its kind.
- `[PROBE]`: a bare-metal probe of each core's cache ID registers, run from u-boot 2026.10,
  2026-10-09. Each core's values come from that core, and the capture lost no bytes.
- `[HANDOFF]`: a bare-metal loader's report from u-boot 2026.10, 2026-10-09. The loader
  asks the boot core for `RNDR` before any other seed, and reports the fallback it took.
- `[FDT]`: `/sys/firmware/fdt`, the tree u-boot 2026.10 passed to Linux 7.2.9, 2026-10-08.
- `[DMESG]`: the kernel log of the same boot.
- `[CACHEINFO]`: `/sys/devices/system/cpu/cpu0/cache` and `cpu4/cache`, a Cortex-A55 and a
  Cortex-A76, on the same boot.
- `[CPUINFO]`: `/proc/cpuinfo` on the same boot.
