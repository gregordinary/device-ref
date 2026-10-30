# RK3588 processor reference

This document covers the RK3588's application cores. Its reader writes a bootloader, a
kernel port or a device tree for any board on the SoC. It states the cores,
their revisions, ID registers and numbering, and their caches as the manual and the
registers give them. It also covers the cache maintenance the cores need, the coherency of
the SoC's devices, the errata that apply, and the cores' operating points.

The document is SoC-level. Register readings come from the reference unit, a Turing RK1,
read on each core by a bare-metal program started from mainline u-boot 2026.10. Three
further ID registers come from a NanoPi R6S, and the text says so where it cites them. Linux
readings come from the same unit under mainline Linux 7.2.9. Configuration facts come from
Rockchip's technical reference manual [source][TRM].

Latency, frequency and fuse readings come from the reference unit under the same Linux, with
Rockchip's BL31 v1.56. They also come from a second RK1 module, with BL31 v1.51, and the text
names the module wherever the two differ.

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
| `ID_AA64DFR0_EL1` | `0x1030_5408` [IDREGS2] | `0x1030_5408` [IDREGS2] |
| `ID_AA64ISAR0_EL1` | `0x1000_1021_1120` [IDREGS2] | `0x1000_1021_1120` [IDREGS2] |
| `ID_AA64ISAR1_EL1` | `0x10_0001` | `0x10_0001` |
| `ID_AA64ISAR2_EL1` | 0 | 0 |
| `ID_AA64MMFR0_EL1` | `0x10_1122` | `0x10_1122` |
| `ID_AA64MMFR1_EL1` | `0x1021_2122` | `0x1021_2122` |
| `ID_AA64MMFR2_EL1` | `0x1011` [IDREGS2] | `0x1011` [IDREGS2] |

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

The rows marked `[IDREGS2]` come from a second probe, on a NanoPi R6S, an RK3588S board. It
read one core of each kind, and started the Cortex-A76 through PSCI `CPU_ON` [measured,
2026-10-22, u-boot][IDREGS2]. The three registers read the same on both kinds. The probe
also read every other register in the table, and each value matched the reference unit's.
The new registers decode as follows:

- `MMFR2.NV` reads 0: neither kind implements FEAT_NV or FEAT_NV2. A hypervisor runs at
  EL2 alone, and a guest cannot be given an EL2 of its own with hardware support
  [inferred]. `MMFR2` also sets `CnP`, `UAO` and `IESB`.
- `MMFR0.TGran16_2` reads 0, which defers to `MMFR0.TGran16`. The 16 KiB granule therefore
  holds at stage 2 as well.
- `ISAR0` gives AES with PMULL, SHA1, SHA2, CRC32, the LSE atomics, RDM and the dot
  product. It gives no SHA3, SM3, SM4, FHM or `RNDR`. That agrees with Linux's capability
  list below.
- `DFR0` gives Armv8.2 debug, PMUv3 at the Armv8.1 level, six breakpoints of which two
  match a context, and four watchpoints. It gives no Statistical Profiling Extension and no
  system-register trace interface.

Neither kind implements FEAT_RNG, by `ISAR0.RNDR` [measured, 2026-10-22, u-boot][IDREGS2].
On the reference unit, `RNDR` gives the boot core no random number [measured, 2026-10-09,
u-boot 2026.10][HANDOFF], and Linux reports no `rng` capability [measured, 2026-10-08,
7.2.9][CPUINFO]. The SoC's TRNG is the hardware source (see
[the TRNG](system-reference.md#trng)).

Under Linux, the reference unit reports these hardware capabilities [measured, 2026-10-08,
7.2.9][CPUINFO]:

```
fp asimd evtstrm aes pmull sha1 sha2 crc32 atomics fphp asimdhp cpuid asimdrdm lrcpc dcpop asimddp
```

No `paca`, `pacg`, `sve`, `sme` or `bti` appears. The list is system-wide and omits what
any core lacks [inferred], and the per-core readings agree with it. Linux also detects the
Virtualization Host Extensions [measured, 2026-10-08, 7.2.9][DMESG].

Linux finds six event counters and the cycle counter on each kind, through
`armv8_cortex_a55` and `armv8_cortex_a76` [measured, 2026-10-23, 7.2.9][BOOT23]. The two
are separate perf PMUs, so a count on a Cortex-A76 names `armv8_cortex_a76`.

Both kinds report hardware dirty-bit management in `MMFR1.HAFDBS`, which reads 2. Linux
uses it on the Cortex-A76s alone, and reports it `on CPU4-7` [measured, 2026-10-23,
7.2.9][BOOT23]. Its erratum 1024718 entry lists every Cortex-A55 revision as having broken
dirty-bit management [source][LINUX].

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

### Measured latency

A chain of dependent loads, one per 64-byte line in random order, gives each level's load
latency. The table comes from the reference unit, with every cluster at its top operating
point [measured, 2026-10-23, 7.2.9][LATENCY]. The clocks are the delivered ones (see
[delivered frequency](#delivered-frequency)). The buffer sits in 4 KiB pages, so the
largest working sets add translation misses to each load.

| Working set | Cortex-A55, 1934 MHz | Cortex-A76, 2318 MHz |
|---|---:|---:|
| 32 KiB | 1.07 ns | 1.73 ns |
| 96 KiB | 5.04 ns | 5.03 ns |
| 256 KiB | 12.95 ns | 5.24 ns |
| 1 MiB | 15.00 ns | 16.47 ns |
| 2 MiB | 15.11 ns | 17.71 ns |
| 3 MiB | 20.57 ns | 18.58 ns |
| 3.5 MiB | 38.55 ns | 32.28 ns |
| 4 MiB | 49.18 ns | 46.15 ns |
| 8 MiB | 87.52 ns | 87.76 ns |
| 512 MiB | 140.00 ns | 132.02 ns |

The Cortex-A76's L1 hit is 4 cycles at its delivered clock, and the Cortex-A55's is 2
[inferred]. The Cortex-A55's latency rises past 32 KiB and 128 KiB, and the Cortex-A76's
past 64 KiB and 512 KiB, the L1 and L2 sizes the manual gives. Latency climbs
steeply between 3 and 3.5 MiB on both kinds, the step a 3 MiB L3 gives (see
[the L3's size](corrections-and-gotchas.md#the-l3-is-3-mib-not-the-4-mib-the-registers-give)).
The DSU's clock sets the L3's latency, as [the DSU clock](#the-dsu-clock-follows-the-cortex-a55s)
describes.

The second RK1 module shows the same steps. Its cache hits take 6 to 8 % longer, as its
slower clocks predict [measured, 2026-10-23, 7.2.9][LATENCY2].

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

Linux exposes the OTP as the nvmem device `rockchip-otp0`. Two dies' cells read as follows
[measured, 2026-10-23, 7.2.9][OTP]:

| Cell | Offset | Reference unit | Second module |
|---|---|---|---|
| CPU code | `0x02`, 2 bytes | `0x3588` | `0x3588` |
| Specification serial number | `0x06`, bits 4:0 | `0x01` | `0x01` |
| Leakage: Cortex-A76 cluster 0, cluster 1, Cortex-A55 | `0x17` to `0x19` | 14, 15 and 17 | 7, 7 and 9 |
| Leakage: logic, GPU | `0x1a`, `0x1b` | 56 and 27 | 29 and 12 |
| CPU version | `0x1c`, bits 5:3 | 1 | 1 |
| Customer demand | `0x22`, bits 7:4 | 0 | 0 |
| Leakage: NPU, codec | `0x28`, `0x29` | 14 and 23 | 6 and 10 |
| Per-block operating-point information | `0x3d` to `0x6c` | All zero | All zero |

A specification serial number of `0x01` names neither the RK3588M nor the RK3588J. The
offsets of the serial number, the customer-demand bits and the operating-point information
are the vendor tree's [source][VENDOR]. The reference unit's die leaks about twice as much in
every cell, and its cores run faster at the same voltage (see
[delivered frequency](#delivered-frequency)).

The vendor kernel chooses its CPU, GPU and NPU voltages by PVTM rather than by leakage. It
reads the logic and codec leakage for the memory controller, the display controller and the
encoder [source][VENDOR]. The second module's values both map to level 0. The reference
unit's logic leakage maps to level 2 and its codec leakage to level 1.

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
Mainline offers 2400 MHz to every die [source][OPP]. Both kernels run their top point at
1,000 mV, and above 1608 MHz the voltage sets the delivered clock (see
[delivered frequency](#delivered-frequency)). A die below level 7 therefore runs at much the
same clock under either kernel [inferred].

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

### Delivered frequency

The secure firmware clocks the cores, through SCMI (see [SCMI](boot-firmware.md#scmi)).
Linux sets each cluster's voltage, then asks the firmware for the rate. In mainline TF-A's
port, the Cortex-A76 clusters run from a PVT-PLL from 816 MHz up, and the Cortex-A55
cluster from 816 to 1800 MHz [source][TFAMAIN]. A PVT-PLL is a ring oscillator, whose
frequency follows the die's speed at its supply [inferred].

The firmware sets only the ring's length for each rate. Mainline TF-A gives the Cortex-A76
table one length, 11, for every rate from 1608 to 2400 MHz [source][TFAMAIN]. Above
1608 MHz the voltage therefore decides the frequency, and the rate asked for does not
[inferred].

Two RK1 modules bear that out, under Rockchip's BL31 v1.56 and v1.51. The PMU's cycle
counter and a timed chain of dependent adds agree within 0.5 % on each [measured,
2026-10-23, 7.2.9][FREQ][FREQ2]. The dies sat at 34 to 38 °C, and the rails read
their operating points' voltages throughout. Each column is the mean of the two clusters.
They differ by under 1 %:

| Operating point | Rail | Cortex-A76, reference unit | Cortex-A76, second module |
|---|---|---:|---:|
| 1200 MHz | 675 mV | 1228 MHz | 1117 MHz |
| 1416 MHz | 725 mV | 1562 MHz | 1419 MHz |
| 1608 MHz | 762.5 mV | 1792 MHz | 1627 MHz |
| 1800 MHz | 850 mV | 2022 MHz | 1846 MHz |
| 2016 MHz | 925 mV | 2187 MHz | 2004 MHz |
| 2208 MHz | 987.5 mV | 2303 MHz | 2115 MHz |
| 2400 MHz | 1,000 mV | 2324 MHz | 2136 MHz |

| Operating point | Rail | Cortex-A55, reference unit | Cortex-A55, second module |
|---|---|---:|---:|
| 1008 MHz | 675 mV | 1065 MHz | 964 MHz |
| 1200 MHz | 712.5 mV | 1335 MHz | 1212 MHz |
| 1416 MHz | 762.5 mV | 1550 MHz | 1414 MHz |
| 1608 MHz | 850 mV | 1747 MHz | 1604 MHz |
| 1800 MHz | 950 mV | 1930 MHz | 1781 MHz |

The reference unit runs 8 to 11 % above the second module at every point. It runs above the
nominal rate at all but the top one. Its die also leaks about twice as much (see the OTP
above). The two modules also run different BL31 versions, so the measurements do not separate
the die from the firmware. Both Cortex-A76 clusters loaded at once deliver the same as each
alone.

> **Neither cpufreq nor the SCMI clock reports the delivered frequency.** Both read the rate
> Linux asked for at every row [measured, 2026-10-23, 7.2.9][FREQ][FREQ2]. Count cycles to
> know what a core runs at.

On both dies the 2400 MHz point gains about 1 % over 2208 MHz, since its rail is only
12.5 mV higher. See
[2400 MHz is a request](corrections-and-gotchas.md#2400-mhz-is-a-request-not-a-frequency).

### The DSU clock follows the Cortex-A55s

The DSU clocks the L3 and the clusters' path to memory. Linux describes no DSU clock. In
mainline TF-A, setting the Cortex-A55 clock sets the DSU clock to the same rate
[source][TFAMAIN]. The Cortex-A55s' operating point therefore sets every core's L3 latency
and memory bandwidth.

With the Cortex-A76s held at 2400 MHz, the Cortex-A55 policy alone moves the Cortex-A76s'
results. The table gives the reference unit's latency and both modules' read bandwidth
[measured, 2026-10-23, 7.2.9][DSUSWEEP][DSUSWEEP2]:

| Cortex-A55 point | A76 latency, 1 MiB | A76 latency, 256 MiB | Four A76s, reference unit | Four A76s, second module |
|---|---:|---:|---:|---:|
| 1008 MHz | 22.91 ns | 147.01 ns | 16.86 GB/s | 15.23 GB/s |
| 1200 MHz | 20.31 ns | 138.72 ns | 20.93 GB/s | 19.04 GB/s |
| 1416 MHz | 18.68 ns | 134.53 ns | 24.02 GB/s | 22.09 GB/s |
| 1608 MHz | 17.33 ns | 131.96 ns | 26.29 GB/s | 24.66 GB/s |
| 1800 MHz | 16.53 ns | 129.70 ns | 27.80 GB/s | 26.66 GB/s |

One Cortex-A76 alone reads 13.58 GB/s at the bottom row and 19.26 GB/s at the top, on the
reference unit. Under the `ondemand` governor, with the Cortex-A55s idle, four Cortex-A76s
read at the 1008 MHz row's rate on both modules [measured, 2026-10-23,
7.2.9][DSUSWEEP][DSUSWEEP2].

The vendor kernel ties the two through a frequency floor. While a Cortex-A76 cluster runs at
a given rate, it holds the Cortex-A55 policy at no less than four fifths of it
[source][VENDOR]. Mainline has no such floor [source][LINUX].

> **Under mainline, raise the Cortex-A55 policy for memory-bound work on the Cortex-A76s.**
> A floor on the Cortex-A55s' `scaling_min_freq` does it, or the `performance` governor.
> Otherwise an idle Cortex-A55 cluster costs the Cortex-A76s 39 to 43 % of their bandwidth.

## Sources

Citation keys:

- `[TRM]`: Rockchip, *RK3588 Technical Reference Manual*, Rev 1.0, Part 1, chapter 3: the
  CPU configuration in Table 3-1, and section 3.3.
- `[DTS]`: Linux `arch/arm64/boot/dts/rockchip/rk3588-base.dtsi` at tag `v7.2.9`.
- `[CPUS]`: Linux `Documentation/devicetree/bindings/arm/cpus.yaml` at tag `v7.2.9`,
  property `reg`.
- `[LINUX]`: Linux at tag `v7.2.9`: `arch/arm64/kernel/cpuinfo.c` (the VIPT rule),
  `arch/arm64/kernel/cacheinfo.c` (`ci_leaf_init`, which sets a cache's level and type
  alone), and `arch/arm64/kernel/cpufeature.c` (`ARM64_HAS_CACHE_IDC`, and
  `cpu_has_broken_dbm`, whose `ARM64_ERRATUM_1024718` entry takes every Cortex-A55).
  `drivers/cpufreq/cpufreq-dt.c` and `cpufreq-dt-platdev.c`, which drive the RK3588's CPU
  operating points and set no frequency floor between policies.
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
  `rockchip,pvtm-voltage-sel` tables. For the floor between clusters,
  `rockchip_cpufreq_update_bus_req` and `cpu_to_bus_freq` in `rockchip-cpufreq.c`, and the
  `rockchip,opp-shared-dsu` property of `cluster0_opp_table`. For the fuses, the `otp` node's
  cells in `rk3588s.dtsi`, and the `rockchip,leakage-voltage-sel` tables of
  `dmc_opp_table`, `vop_opp_table` and `venc_opp_table`.
- `[TFAMAIN]`: ARM-software/arm-trusted-firmware at commit `5103554` (2026-10-05),
  `plat/rockchip/rk3588/drivers/scmi/rk3588_clk.c`: the `rk3588_cpul_pvtpll_table` and
  `rk3588_cpub0_pvtpll_table` tables, `clk_cpul_set_rate`, `clk_cpub01_set_rate`,
  `clk_scmi_cpub01_get_rate`, and `clk_scmi_cpul_set_rate`, which calls
  `clk_scmi_dsu_set_rate` with the same rate. Rockchip's BL31 is built from its own tree,
  which is not public.
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

A capture from another RK3588 board, kept with its evidence outside this repository:

- `[IDREGS2]`: a bare-metal feature probe on a NanoPi R6S, run from u-boot with `go`,
  2026-10-22. It reads the AArch64 identification block on the boot core, a Cortex-A55,
  and on a Cortex-A76 at affinity `0x400` that it starts through PSCI `CPU_ON`. It prints
  all 16 digits of each value, and the capture lost no bytes.

Captures from the reference unit under Linux 7.2.9, with u-boot 2026.10, BL31 v1.56 and DDR
firmware v1.24, kept with its evidence outside this repository:

- `[BOOT23]`: the kernel log of a boot, 2026-10-23.
- `[OTP]`: `/sys/bus/nvmem/devices/rockchip-otp0/nvmem` read as root, 2026-10-23, on both
  modules.
- `[LATENCY]`: a pointer chase pinned to `cpu0` and to `cpu4`, 2026-10-23. It walks one
  64-byte line per node in a random cycle, 20 million loads per working set, with every
  cpufreq policy at `performance`.
- `[FREQ]`: each cpufreq operating point in turn, pinned through `scaling_min_freq` and
  `scaling_max_freq`, 2026-10-23. At each point it timed 200 million iterations of ten
  dependent adds by `CLOCK_MONOTONIC`, best of five, and read the rail's `microvolts` and
  `cpuinfo_cur_freq`. A second pass at six points counted the core's cycles with `perf stat`
  for one second under a busy loop.
- `[DSUSWEEP]`: the Cortex-A55 policy pinned at each of its operating points with the
  Cortex-A76 policies at `performance`, 2026-10-23. At each point the pointer chase ran on
  `cpu4`. A streaming read of 256 MiB per thread then ran 16 times, on `cpu4` alone and on
  `cpu4` to `cpu7`. The `ondemand` figure is the same read with every policy at `ondemand`.

Captures from a second Turing RK1 module, kept with its evidence outside this repository. It
is a 32 GB module, and its eMMC carries u-boot 2026.07, BL31 v1.51 and DDR firmware v1.19:

- `[LATENCY2]`, `[FREQ2]` and `[DSUSWEEP2]`: the same three methods, 2026-10-23. `[FREQ2]`
  counted cycles with `perf stat` at every point. Static builds of the same programs, run on
  both modules, reproduced this module's native builds within 1 %.
