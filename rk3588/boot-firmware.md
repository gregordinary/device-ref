# RK3588 boot firmware

This document covers the RK3588 boot firmware from the side of an engineer porting a
bootloader or an operating system. It describes the boot chain and its blobs, and the
services Rockchip's BL31 answers: PSCI, SCMI and its SiP calls. It also covers the memory
the firmware keeps and the accesses it refuses the non-secure world. It closes with the
state mainline u-boot leaves for a program it starts.

The document is SoC-level. Hardware behavior is measured on the reference unit, a Turing
RK1, running rkbin's DDR firmware v1.24 and BL31 v1.56 under mainline u-boot 2026.10.
Source reading covers the mainline Linux and u-boot trees and upstream TF-A [source]. The
BL31 analysis is a disassembly of the rkbin blob [BLOB]. System sleep and its wake sources
are owned by the [RK3588 system sleep reference](system-sleep.md).

Some Linux readings come from a second RK1 module, running DDR firmware v1.19 and BL31 v1.51
under u-boot 2026.07. The text names that firmware wherever it cites them.

## The boot chain

The chain runs BootROM -> DDR firmware (TPL) -> SPL -> BL31 (TF-A) -> U-Boot -> kernel.
The BootROM loads its boot code from SPI flash, eMMC or SD, or over USB OTG [source][TRM].

The reference unit boots from its eMMC. The boot area from sector 64 carries these version
strings [measured, 2026-10-08][STRINGS]:

| Component | Version |
|---|---|
| DDR firmware | rkbin, `fwver: v1.24` |
| BL31 | rkbin, `v2.3-975-g6c06d3bad`, `fwver: v1.56` |
| U-Boot and its SPL | mainline 2026.10 |

The chain carries no BL32. The SPL checks the FIT's configuration, three BL31 images,
U-Boot and its FDT, and no TEE image [measured, 2026-10-08, u-boot 2026.10][CONSOLE].

The BL31 version string is the one rkbin's `rk3588_bl31_v1.56.elf` carries [source][BLOB].
That BL31 is built from Rockchip's own tree of TF-A 2.3, not from mainline TF-A
[inferred]. Mainline TF-A carries an RK3588 port of its own, which Rockchip contributed
[source][TFAMAIN].

BL31 v1.56 carries a version-pairing message: `current trust bl31 need match with loader
ddr bin V1.13 or newer` [source][BLOB]. It pairs this BL31 with DDR firmware v1.13 or
newer, which v1.24 meets [inferred].

DDR firmware v1.20 adds support for two-channel DDR. rkbin's release note ties that
support to BL31 v1.53 or later [source][RKBIN]. The requirement belongs to the two-channel
configuration, not to the release as a whole. A four-channel Rock 5B runs DDR firmware
v1.20 with BL31 v1.48 [reported][YSP].

### The boot medium's layout

Mainline u-boot writes the boot stages at fixed offsets on an eMMC or SD card
[source][UBOOT]:

| Offset | Content |
|---|---|
| Sector 64, 32 KiB | The idbloader: the DDR firmware and the SPL behind a version 2 header, magic `RKNS` |
| Sector `0x4000`, 8 MiB | `u-boot.itb`, a FIT carrying BL31, u-boot and its FDT |

The SPL loads the FIT from raw sector `0x4000`, u-boot's default for every Rockchip SoC.
The reference unit's eMMC carries both stages at these offsets [measured]. Nothing in
this layout reserves the space, so a partition table must start its first partition
above the FIT. The [RK1's boot and recovery](turing-rk1/hardware-reference.md#boot-and-recovery)
section names a case that breaks this rule.

## Memory the firmware keeps

Mainline u-boot leaves the bottom 2 MiB of DRAM to BL31. It takes its DRAM banks from the
DDR firmware's handoff, the `DDR_MEM` tag, or, lacking one, from a fixed layout. Either way
it moves the first bank up to 2 MiB [source][UBOOT]. It also splits a bank around any
device range of its own memory map, the peripheral window below 4 GiB among them.

BL31 v1.56 is an ELF of three load segments, with its entry at `0x6_0000` [source][BLOB]:

| Segment | Content |
|---|---|
| `0x6_0000` to `0xd_1000` | Code, data, stacks, BSS and translation tables |
| `0xf_0000` to `0xf_6000` | The section `.ddr_m0_bin` |
| `0xff10_0000` to `0xff10_9000` | The section `.text_pmusram`, code in the PMU's SRAM |

Mainline u-boot's binman splits the ELF into one FIT image per segment [source][UBOOT].
The three BL31 images the SPL reports are these segments. The SPL itself runs from DRAM
at 0. Every RK3588 board configuration caps it at `0x4_0000` bytes, so it ends below
BL31's first segment [source][UBOOT].

The first segment's base has moved. BL31 v1.42, v1.45, v1.48 and v1.51 load it, and enter,
at `0x4_0000`, where the SPL's cap ends. v1.56 loads it at `0x6_0000` [source][BLOBS]. A
Rock 5B's BL31 v1.54 loads at `0x6_0000` too [reported][YSP]. The other two segments stay
at `0xf_0000` and `0xff10_0000`. A bootloader that places BL31 by address, rather than by
the ELF's program headers, must follow the blob's version.

The firmware's SCMI channel sits inside that 2 MiB. It is 256 bytes at `0x10f000`,
described as `/reserved-memory/shmem@10f000` with `no-map` [measured, 2026-10-08,
7.2.9][FDT]. No node describes a secure OS's memory [measured, 2026-10-08, 7.2.9][FDT]. The tree's
one other reserved region, a pool for the HDMI receiver, is disabled. Its one
`/memreserve/` holds the initial RAM disk the bootloader loaded. DRAM from 2 MiB upward is
therefore free of firmware reservations [inferred].

### Across a warm reset

DRAM keeps its contents across a warm reset. A Rock 5B ran DDR firmware v1.20 and BL31
v1.48. Its ramoops region at `0x11_8000` to `0x1e_7fff` came back intact across at least
ten warm resets [reported][YSP]. Earlier kernel builds on the same board found the region
zeroed after a reset, and the cause was not found.

The same source maps the rest of the bottom 2 MiB by inspection [reported][YSP]:

| Range | Use |
|---|---|
| `0x10_0000` to `0x10_ffff` | BL31's shared memory, which holds the SCMI channel at `0x10f000` |
| `0x11_0000` to `0x11_7fff` | A debug ring of the DDR firmware |
| `0x1f_0000` to `0x1f_ffff` | The vendor stack's reserved tail. The vendor u-boot writes its ATAGS from `0x1f_e000` |

The vendor kernel places ramoops at `0x11_0000`, `0xe_0000` bytes long, over the DDR
firmware's ring. A ramoops region placed in the bottom 2 MiB must stay clear of
`0x10_0000` to `0x11_7fff` [inferred].

## PSCI

BL31 answers PSCI 1.1 through SMC, under SMC Calling Convention 1.2 [measured, 2026-10-08,
7.2.9][DMESG]. The mainline tree describes the interface as `arm,psci-1.0` with
`method = "smc"` [source][DTS]. Through `PSCI_FEATURES`, the firmware reports `CPU_OFF`
and `SYSTEM_SUSPEND` as implemented [measured, 2026-10-10, u-boot 2026.10][OFFLINE].

Linux logs `psci: MIGRATE_INFO_TYPE not supported` on the reference unit [measured,
2026-10-08, 7.2.9][DMESG]. TF-A answers that call with `NOT_SUPPORTED` unless a secure
payload dispatcher has registered a handler for it [source][TFA]. OP-TEE's dispatcher
registers one once OP-TEE has started, and Linux then logs
`psci: Trusted OS migration not required` instead. The line therefore marks a chain with
no OP-TEE running, whatever the BL31 version, as here [inferred].

BL31 runs without OP-TEE. v1.56 carries a message for that case, which begins
`No OPTEE provided by BL2 boot loader` [source][BLOB]. It goes on to say that calls meant
for OP-TEE return `SMC_UNK`. On a Rock 5B, BL31 v1.48 logs that warning and
`Error initializing runtime service opteed_fast`, then exits to the normal world as usual
[reported][YSP]. The error is not fatal. A console that falls silent after it has stopped
in a later stage, not in BL31 [inferred].

### Starting a core

BL31 starts a core at EL2, at the entry `CPU_ON` names, with the context ID in `x0`
[measured, 2026-10-08, u-boot 2026.10][LOADER]. It accepts an entry in DRAM above 16 GiB,
from `0x4_0000_0000` [measured, 2026-10-09, u-boot 2026.10][EIGHT]. TF-A 2.3's common
Rockchip code supplies no entry-point check. PSCI's generic code then refuses only an odd
address from an AArch64 caller [source][TFA].

The firmware sets a core's GIC redistributor up again as it powers the core on. Public TF-A
2.3 carries no RK3588 port, but its common Rockchip code calls `gicv3_rdistif_init` there.
That call disables every SGI and PPI of the core and puts each in non-secure group 1. It
then returns the platform's secure ones to the secure groups and enables them again
[source][TFA]. Mainline TF-A's RK3588 port names two: the secure physical timer, INTID 29,
in secure group 1, and SGI 14 in group 0 [source][TFAMAIN]. Rockchip's BL31 comes from its
own tree of TF-A 2.3, and does the same [inferred].

> **An enable written into a core's redistributor before `CPU_ON` is lost.** That
> includes the enable of the IPI the starting core waits for [inferred]. Enable the SGIs
> and PPIs a core needs on that core, as it starts.

`CPU_OFF` and `CPU_ON` cycle each core but the boot core [measured, 2026-10-10, u-boot
2026.10][OFFLINE].
After `CPU_OFF`, `AFFINITY_INFO` reports the core off, and a later `CPU_ON` succeeds. All
seven cores taken off together come back within 31 ms.

The boot core cycles as well, once another core runs the system. Linux's PSCI checker takes
every core but one off and back, the boot core among them. It then suspends each core ten
times in `CPU_SLEEP`. Every step passes on the reference unit, and under BL31 v1.51 on a
second module [measured, 2026-10-23, 7.2.9][BOOT23][BOOTLOG].

### Idle states

The mainline tree gives every CPU one idle state, `CPU_SLEEP` [measured, 2026-10-08,
7.2.9][FDT]:

| Property | Value |
|---|---|
| `arm,psci-suspend-param` | `0x0001_0000`: StateType 1, a power-down state |
| `local-timer-stop` | Present |
| Entry latency | 100 us |
| Exit latency | 120 us |
| Minimum residency | 1000 us |

The RK3576 trees carry the same state (see
[deep idle under BL31 v1.24](../rk3576/boot-firmware.md#deep-idle-under-bl31-v124)).

Linux drives the state through `psci_idle`, with the `menu` governor. It reports an exit
latency of 220 us, the entry and exit latencies summed [measured, 2026-10-23, 7.2.9][IDLE].
Over 17.8 days of uptime, under BL31 v1.51, every core entered `CPU_SLEEP` between 8.9 and 43
million times. Each core spent 97 to 99 % of the uptime there, and the system ran without a
fault [measured, 2026-10-23, 7.2.9, BL31 v1.51][IDLE].

Linux counts 1.1 to 1.4 % of those entries as rejected, the entry returning an error
[measured][IDLE]. The core then returns at once, and the idle loop carries on [inferred].

## SCMI

BL31 serves an SCMI agent through SMC [measured, 2026-10-08, 7.2.9][FDT]:

| Property | Value |
|---|---|
| `compatible` | `arm,scmi-smc` |
| `arm,smc-id` | `0x8200_0010` |
| Shared memory | 256 bytes at `0x10f000` |
| Protocols | Clock (`0x14`) and reset (`0x16`) |

Linux reports the agent as SCMI protocol version 2.0 [measured, 2026-10-08, 7.2.9]. The agent
names its vendor `rockchip`, with no sub-vendor and firmware version 0. BL31 v1.51 reports the
same [measured, 2026-10-23, 7.2.9][BOOT23][BOOTLOG]. Linux then:

- Sets the channel's limits at a 30 ms receive timeout, 104-byte messages and 20 messages
  in flight.
- Polls for the reset protocol's replies.
- Enables its quirk `quirk_clock_rates_triplet_out_of_spec`, for the agent's clock rate
  ranges.

The SMCCC `SOC_ID` call is not implemented, so Linux registers no SoC identity from the
firmware [measured, 2026-10-23, 7.2.9][BOOT23]. The mainline tree takes these
clocks and resets from the agent [source][DTS]:

| Consumer | SCMI clock or reset |
|---|---|
| The Cortex-A55 cores | `SCMI_CLK_CPUL` |
| Cortex-A76 cores `cpu_b0` and `cpu_b1` | `SCMI_CLK_CPUB01` |
| Cortex-A76 cores `cpu_b2` and `cpu_b3` | `SCMI_CLK_CPUB23` |
| The GPU | `SCMI_CLK_GPU`, as an assigned rate of 200 MHz. Its operating clocks come from the CRU |
| The three NPU cores | `SCMI_CLK_NPU`, one clock for all three, assigned 200 MHz (see [the NPU reference](npu-reference.md)) |
| The SD card controller, `mmc@fe2c0000` | `SCMI_HCLK_SD` and `SCMI_CCLK_SD` |
| The TRNG | `SCMI_HCLK_SECURE_NS`, and the reset `SCMI_SRST_H_TRNG_NS` (see [the TRNG](system-reference.md#trng)) |

The agent describes more clocks than the tree consumes, and Linux registers each one. These
are the ones no tree node names, with the rates the agent reports [measured, 2026-10-23,
7.2.9][CLK]:

| Clock | Rate |
|---|---|
| `scmi_clk_ddr` | 2112 MHz, the DRAM clock |
| `scmi_clk_dsu` | Follows `scmi_clk_cpul` (see [the DSU clock](cpu-reference.md#the-dsu-clock-follows-the-cortex-a55s)) |
| `scmi_spll` | 702 MHz |
| `scmi_crypto_c`, `scmi_crypto_p`, `scmi_crypto_r` | 350, 350 and 175 MHz |
| `scmi_aclk_se_ns` | 350 MHz |
| `scmi_tclk_wdt` | 24 MHz |
| `scmi_otp_phy`, `scmi_otpc_rd`, `scmi_otpc_arb` | 24 MHz each |
| `scmi_dclk_sd` | 198 MHz |
| `scmi_aclk_vop_r` | 500 MHz. BL31 v1.51 does not describe it |

The CPU clocks' rates are the ones Linux asked for, not the cores' delivered frequency (see
[delivered frequency](cpu-reference.md#delivered-frequency)). Linux finds all three CPU
clocks at 816 MHz, a rate no operating point lists. It moves the Cortex-A55s to 1008 MHz and
the Cortex-A76s to 1200 MHz. BL31 v1.51 leaves the same rates [measured, 2026-10-23,
7.2.9][BOOT23][BOOTLOG]. The boot firmware therefore leaves every cluster at 816 MHz
[inferred].

## SiP calls

BL31's SiP service reports 4 calls, at revision major 0 [measured, 2026-10-10, u-boot
2026.10][SLEEP]. One of them, `SIP_SUSPEND_MODE` (`0x8200_0003`), sets the sleep mode and
the wake sources for `SYSTEM_SUSPEND`. The
[RK3588 system sleep reference](system-sleep.md#rockchips-bl31-and-the-wake-sources)
describes it.

## Accesses the firmware refuses

BL31 keeps several registers from the non-secure world. On the reference unit, under BL31
v1.56, an access from EL2 has these results [measured, 2026-10-09 and 2026-10-10, u-boot
2026.10][PROBE][SLEEP]:

| Access from EL2 | Result |
|---|---|
| A read of the PMU's `PMU_WAKEUP_INT_CON` at `0xfd8d_4010` | A synchronous external abort |
| A read of `ICC_IGRPEN0_EL1` | The board resets: the SPL's banner follows at once |
| A write to `GICR_IGROUPR0` | Ignored. The register reads 0 |

Mainline u-boot's configuration for the RK1 enables no watchdog [source][UBOOT], so the read
of `ICC_IGRPEN0_EL1` itself causes the reset [inferred]. Mainline TF-A's RK3588 port marks
the PMU's PMU0 and PMU1 registers and the PMU's SRAM secure [source][TFAMAIN].
[Security states and groups](system-reference.md#security-states-and-groups) describes the
GIC side.

## Entering a program from mainline u-boot

u-boot's `go` command runs a bare-metal program in place of an operating system. It calls
the program as a C function, `entry(argc, argv)`, and prints the value it returns
[source][UBOOT]. The command touches neither the MMU nor the caches before the call
[source][UBOOT].

u-boot 2026.10 enters such a program at EL2, on its own page tables. The MMU and both
caches are on: `SCTLR_EL2` reads `0x30c5_1835` [measured, 2026-10-08, u-boot
2026.10][LOADER]. `x0` carries `argc`, which is 1 for `go` with an address alone.

u-boot's arm64 build reserves `x18` for its global data pointer [source][UBOOT]. A program
that returns to u-boot restores `x18` along with the registers the C calling convention
preserves.

u-boot 2026.10 leaves this EL2 configuration [measured, 2026-10-09, u-boot
2026.10][PROBE]:

| Register | Value | Meaning |
|---|---|---|
| `HCR_EL2` | `0x22` | RW 0: EL1 runs AArch32. AMO 1: SErrors go to EL2. SWIO 1. IMO and FMO 0 |
| `CNTHCTL_EL2` | `0x3` | EL1 reaches the physical counter and timer |
| `CNTVOFF_EL2` | 0 | The virtual count equals the physical count |
| `CNTFRQ_EL0` | 24,000,000 | The counter runs at 24 MHz |
| `ICC_SRE_EL2` | `0xf` | The GIC's system registers on at EL2, and open to EL1 |
| `MDCR_EL2` | `0x6` | EL1 sees six event counters of the PMU |
| `TCR_EL2` | `0x8081_351c` | A 4 KiB granule and 36-bit addresses |
| `MAIR_EL2` | `0xff_440c_0400` | Device nGnRnE, nGnRE and GRE, Normal non-cacheable, Normal write-back |

> **A loader that drops to EL1 sets `HCR_EL2.RW` itself, or EL1 runs AArch32** [inferred].

The GIC's distributor has no non-secure group enabled at that point. `GICD_CTLR` reads `0x10`,
affinity routing alone, and the firmware leaves `ICC_PMR_EL1` at `0xf0` and
`ICC_IGRPEN1_EL1` at 0 [measured, 2026-10-09, u-boot 2026.10][PROBE]. A kernel enables
group 1 in the distributor before it waits for its first interrupt [inferred].

u-boot's page tables map every DRAM range as Normal write-back memory, inner shareable
[source][UBOOT]. They map the peripheral window from `0xf000_0000` as Device-nGnRnE, and
the SCMI channel's 4 KiB at `0x10_f000` as Normal non-cacheable [source][UBOOT]. On a
32 GB module, `AT S1E2R` probed 7,126 pages in four windows, three below 4 GiB and one
above 16 GiB. Every page maps to itself as Normal write-back [measured, 2026-10-09, u-boot
2026.10][FIRSTPROC]. Its DRAM entries stop at 32 GiB, and leave two ranges unmapped, 5 MiB at
`0x3_fc00_0000` and 1 MiB at `0x3_fff0_0000` [source][UBOOT].

A program started this way runs with its data cache on. It cleans the data cache before it
turns the caches off, or hands memory to code that runs with them off [inferred]. The
[RK3588 processor reference](cpu-reference.md#cache-maintenance) gives the line counts
that set and way maintenance walks.

## UEFI firmware

The community edk2-rk3588 project builds UEFI firmware for RK3588 boards, which replaces
mainline u-boot [source][EDK2]. It shares the DDR firmware with u-boot's chain, and differs
at three later stages:

- The SPL is Rockchip's prebuilt `rk3588_spl_v1.12.bin`, with no matching source.
- BL31 is mainline TF-A, with the project's patches for the PMIC, SCMI voltage, SPI, SD
  and the TRNG.
- The OP-TEE image, rkbin's `rk3588_bl32_v1.23.bin`, is always packed into the FIT, so
  this chain runs a BL32.

Its release image puts a GPT at 0, whose one partition, `uboot`, covers 1 to 9 MiB. The
idbloader sits at 32 KiB, and a FIT at 1 MiB carries BL31, OP-TEE, the 5 MiB firmware
volume and the DTB. The variable store takes `0x7c_0000` to `0x7f_0000`. The project
requires that no u-boot remain on any boot medium beside it.

Mainline TF-A takes its console from `/chosen/stdout-path` only in the form
`serialN:baud`, with N from 0 to 5, and otherwise prints on uart2 [source][TFAMAIN]. On a
board whose console is uart6 to uart9, this BL31 prints nothing the console shows.

The firmware hands the OS either a device tree, built from the upstream tree, or ACPI
tables. In device-tree mode Linux needs 6.10 or later, and 6.15 or later for HDMI
[source][EDK2]. Under ACPI, Linux 7.2.9 binds far less [source][EDK2][LINUX]:

| Block | Under ACPI |
|---|---|
| Ethernet (GMAC) | Not bound. The tables describe it as `PRP0001` with `snps,dwmac-4.20a`, and `stmmac_probe_config_dt()` reads every property through the device's `of_node`, which an ACPI device lacks |
| eMMC | Not bound. Its `_HID` is `RKCP0D40`, and the only ACPI ID `sdhci-of-dwcmshc` matches is `MLNXBF30` |
| Thermal | None. Nothing starts the TSADC |
| CPU frequency scaling | None. The tables carry no `_CPC` |
| PCIe, NVMe, USB and SATA | Bound, through `PNP0A08` and MCFG with the Graviton ECAM quirk, and the standard XHCI and AHCI IDs |
| GPU, video codecs, NPU and HDMI | Not described. HDMI shows the firmware's GOP framebuffer only |

The project's README states that it develops and tests ACPI against Windows only
[source][EDK2]. Linux on this firmware therefore runs in device-tree mode.

## Sources

Citation keys:

- `[TRM]`: Rockchip, *RK3588 Technical Reference Manual*, Rev 1.0, Part 1, section 1.2
  (system boot).
- `[BLOB]`: rockchip-linux/rkbin at commit `036a35a`, `bin/rk35/rk3588_bl31_v1.56.elf`,
  SHA-256 `2864c2fc36333e03e7e3fe6459225d320d310850b4ac7620d1f3601efb7d4a9b`. Its version
  string is `v2.3-975-g6c06d3bad`, as the reference unit's boot area carries.
- `[TFA]`: ARM-software/arm-trusted-firmware at tag `v2.3`, the release Rockchip's BL31
  builds on. It carries no RK3588 port, so the citations are its common Rockchip code and
  its generic code. `plat/rockchip/common/plat_pm.c` (`rockchip_pwr_domain_on_finish`,
  `plat_rockchip_psci_pm_ops`) and `plat/rockchip/common/rockchip_gicv3.c`.
  `drivers/arm/gic/v3/gicv3_main.c` and `gicv3_helpers.c` (`gicv3_rdistif_init`,
  `gicv3_ppi_sgi_config_defaults`, `gicv3_secure_ppi_sgi_config_props`).
  `lib/psci/psci_common.c` (`psci_spd_migrate_info`, `psci_validate_entry_point`), and
  `services/spd/opteed/` (`opteed_cpu_migrate_info`).
- `[EDK2]`: edk2-porting/edk2-rk3588 at commit `3189d482` (2026-10-09): its `README.md`,
  `build.sh`, the ACPI tables `Gmac1.asl` and `Emmc.asl`, `RkFvbDxe`, and the release image
  layout. Its TF-A submodule at `51d8794` (2026-09-15) carries
  `plat/rockchip/common/params_setup.c` (`plat_rockchip_dt_process_fdt_uart`) and
  `plat/rockchip/rk3588/rk3588_def.h` (`RK_DBG_UART_BASE`).
- `[LINUX]`: Linux at tag `v7.2.9`: `drivers/net/ethernet/stmicro/stmmac/stmmac_platform.c`
  (`stmmac_probe_config_dt`) and `drivers/mmc/host/sdhci-of-dwcmshc.c` (its ACPI match
  table).
- `[TFAMAIN]`: ARM-software/arm-trusted-firmware at commit `5103554` (2026-10-05):
  `plat/rockchip/rk3588/drivers/secure/secure.c` (`sgrf_init`), and
  `plat/rockchip/rk3588/rk3588_def.h` (the secure interrupts).
- `[UBOOT]`: u-boot at commit `ece349a`: `arch/arm/mach-rockchip/sdram.c`
  (`rockchip_dram_init_banksize`), `cmd/boot.c` (`do_go_exec`), `arch/arm/config.mk`,
  `arch/arm/include/asm/global_data.h`, and `arch/arm/mach-rockchip/rk3588/rk3588.c`
  (`rk3588_mem_map`). `tools/rkcommon.c` (the idbloader header), `common/spl/Kconfig`
  (`SYS_MMCSD_RAW_MODE_U_BOOT_SECTOR`), `arch/arm/dts/rockchip-u-boot.dtsi` (the
  `split-elf` BL31 images), `arch/arm/mach-rockchip/Kconfig` (`SPL_TEXT_BASE`), and the
  `CONFIG_SPL_MAX_SIZE` of the 30 RK3588 board configurations. `configs/turing-rk1-rk3588_defconfig` and
  `arch/arm/mach-rockchip/Kconfig`, which neither enable nor imply a watchdog.
- `[DTS]`: Linux `arch/arm64/boot/dts/rockchip/rk3588-base.dtsi` at tag `v7.2.9`.
- `[RKBIN]`: rockchip-linux/rkbin, `doc/release/RK3588_EN.md`, the entry for
  `rk3588_ddr_lp4_2112MHz_lp5_2400MHz_v1.20.bin` (2025-08-25). Its one new item adds
  support for 2-channel DDR, and with it requires BL31 v1.53 or later.
- `[BLOBS]`: rockchip-linux/rkbin's `bin/rk35/rk3588_bl31_v1.42.elf`, `v1.45`, `v1.48`,
  `v1.51` and `v1.56`: the entry address and load segments of each ELF's program headers.
- `[YSP]`: Yi Ding, rock-5b-ysp, https://github.com/yisding/rock-5b-ysp, at commit
  `69a3abbf`, a Rock 5B project whose documentation is licensed CC-BY-SA-4.0. Its firmware
  findings of 2026-07-07 and 2026-07-24 (the blob versions and BL31's load address), of
  2026-07-08 (BL31 v1.48's console without OP-TEE), and of 2026-07-28 with its ramoops
  notes (retention across warm resets, and the bottom 2 MiB). The facts are restated here,
  not quoted.

Captures from the reference unit, kept with its evidence outside this repository:

- `[STRINGS]`: printable runs of 8 or more characters in sectors 64 to 32767 of the eMMC,
  filtered for version strings, 2026-10-08.
- `[CONSOLE]`: the console from power-on to u-boot 2026.10's prompt, `version` and
  `bdinfo`, 2026-10-08.
- `[FDT]`: `/sys/firmware/fdt`, the tree u-boot 2026.10 passed to Linux 7.2.9, 2026-10-08.
- `[DMESG]`: the kernel log of the same boot.
- `[LOADER]`: a bare-metal program's report of each core's ID registers, run twice from
  u-boot 2026.10 with `go`, 2026-10-08. It starts each core through `CPU_ON`.
- `[PROBE]`: a bare-metal probe of the EL2, GIC and cache state, run from u-boot 2026.10
  with `go`, 2026-10-09. A second run read `ICC_IGRPEN0_EL1`, and the capture runs on to
  the prompt after the reset.
- `[FIRSTPROC]`: a bare-metal loader run from u-boot 2026.10, 2026-10-09. Before it handed
  off to a kernel, it asked the MMU about each page of its working windows.
- `[EIGHT]` and `[OFFLINE]`: the console of an open-source XNU kernel started from u-boot
  2026.10, on 2026-10-09 and 2026-10-10. The second run's root takes cores offline and
  back, and asks the firmware's `PSCI_FEATURES`.
- `[SLEEP]`: a bare-metal sleep probe run four times from u-boot 2026.10, 2026-10-10. Each
  run stopped at its read of `PMU_WAKEUP_INT_CON`.

- `[BOOT23]`: the kernel log of Linux 7.2.9 after a power cycle, 2026-10-23. The kernel is
  built with `CONFIG_ARM_PSCI_CHECKER`.
- `[CLK]`: `/sys/kernel/debug/clk/clk_summary` on the same boot, 2026-10-23, on this unit and
  on the second module.

Captures from a second Turing RK1 module, kept with its evidence outside this repository. Its
eMMC carries u-boot 2026.07, BL31 v1.51 and DDR firmware v1.19:

- `[BOOTLOG]`: the kernel log of a boot of Linux 7.2.9, 2026-10-05, read from the journal.
- `[IDLE]`: each core's `cpuidle` statistics under `/sys/devices/system/cpu`, read after
  17.8 days of uptime, 2026-10-23.
