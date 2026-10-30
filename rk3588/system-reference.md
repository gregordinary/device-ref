# RK3588 system reference

This document covers the system blocks a bootloader or a new kernel reaches before any
other driver. They are the address map, the interrupt controller, the timers, the UARTs
and the true random number generator. It also covers the IOMMUs, which every media and
accelerator driver programs, and the DMA controllers. It states each block's addresses,
configuration and registers, and the state the boot firmware leaves it in. It closes with
the DRAM's measured bandwidth and the thermal sensors.

The document is SoC-level. Configuration facts come from Rockchip's technical reference
manual [source][TRM] and the mainline device tree [source][DTS]. Measurements come from the
reference unit, a Turing RK1, under mainline u-boot 2026.10 and mainline Linux 7.2.9. The
firmware's side is owned by the [RK3588 boot firmware](boot-firmware.md) reference.

## Address map

The SoC addresses DRAM linearly from zero, up to 32 GiB [source][TRM]. Its peripheral
window runs from `0xf000_0000` to 4 GiB, inside the first 4 GiB of that space
[source][TRM]. The window carries PCIe configuration space and the SoC's peripherals, the
GIC and the UARTs among them.

The DDR firmware gives the bootloader the DRAM ranges, as
[memory the firmware keeps](boot-firmware.md#memory-the-firmware-keeps) describes. On the
reference unit, a 32 GB module, no range stands in for the DRAM behind the window
[measured, 2026-10-08, 7.2.9][IOMEM]. Its ranges add up to 32 GiB less the window's
256 MiB, less 2 MiB at the bottom and two holes of 5 MiB and 1 MiB (see
[the RK1's memory](turing-rk1/hardware-reference.md#memory)). A module with less memory can
carry a different map [expected].

PCIe also maps two 64-bit windows, at `0x9_0000_0000` and `0x9_c000_0000` [measured,
2026-10-08, 7.2.9][IOMEM].

## Interrupt controller

The RK3588 carries a GICv3, `compatible = "arm,gic-v3"` [source][DTS]:

| Block | Base | Size |
|---|---|---|
| Distributor (GICD) | `0xfe60_0000` | `0x1_0000` |
| MBI frame | `0xfe61_0000` | |
| ITS 0 | `0xfe64_0000` | `0x2_0000` |
| ITS 1 | `0xfe66_0000` | `0x2_0000` |
| Redistributors (GICR) | `0xfe68_0000` | `0x10_0000` |

The block is Arm's GIC-600, revision r1p6, built with this configuration [source][TRM]:

| Item | Value |
|---|---|
| Clusters | 1, of 8 cores |
| SPIs | 480, INTIDs 32 to 511 |
| SGIs and PPIs | 16 SGIs and 12 PPIs per core |
| LPIs | Supported, through two ITS blocks |
| Security | Not disabled at build time |
| Address reach of the distributor and ITS | 0 to 32 GiB |

Linux finds 480 SPIs, no extended SPI range, and an MBI range of SPIs 424 to 479, the last
of the 480 [measured, 2026-10-08, 7.2.9][DMESG]. The manual's overview says 512 SPI
sources, which counts the whole INTID range [inferred] (see
[the GIC's SPI count](corrections-and-gotchas.md#the-gic-has-480-spis)).

### Interrupt numbers

Every SPI is level-sensitive, active high [source][TRM]. The manual's interrupt list
numbers each source by INTID, PPIs included [source][TRM]. A device tree numbers SPIs from
zero, so a tree's `GIC_SPI` number is the INTID less 32. The manual gives uart9 INTID 372,
and the tree gives it `GIC_SPI 340` [source][TRM][DTS].

The GIC's maintenance interrupt is PPI 9 in the tree, INTID 25 [source][DTS].

### Redistributors

The redistributor region spans `0x10_0000` for eight cores. Linux finds each core's
redistributor `0x2_0000` above the last, from `0xfe68_0000` for affinity `0x000` to
`0xfe76_0000` for `0x700` [measured, 2026-10-08, 7.2.9][DMESG].

The tree partitions the per-core PPIs by cluster: the four Cortex-A55s in one partition,
and the four Cortex-A76s in the other. The two kinds' performance monitor nodes,
`arm,cortex-a55-pmu` and `arm,cortex-a76-pmu`, each take PPI 7 on their own partition
[source][DTS]. Linux sets both partitions up on the reference unit [measured, 2026-10-08,
7.2.9][DMESG].

### The ITS and shareability

The ITS reaches its tables with non-shareable accesses only [source][TRM]. Linux enables
its workaround `Rockchip erratum RK3588001` for both ITS blocks on the reference unit
[measured, 2026-10-08, 7.2.9][DMESG]. Linux describes the erratum as a GIC-600 integration
without ACE or ACE-Lite, so the block's shareability attributes are unusable
[source][LINUX]. The tree marks the GIC and both ITS blocks `dma-noncoherent` for the same
reason [inferred].

> **Clean the LPI property table and the ITS command queue to memory before the GIC reads
> them.** The GIC does not snoop the cores' caches [inferred]. Linux flushes both on the
> reference unit, and allocates the ITS tables non-shareable [measured, 2026-10-08,
> 7.2.9][DMESG].

### Security states and groups

The GIC runs with two security states. Linux reads `GICD_CTLR.DS` as 0 on the reference
unit. It finds group 0 out of the non-secure world's reach, which it reports as
`SCR_EL3.FIQ=1` [measured, 2026-10-08, 7.2.9][DMESG]. Linux learns the second fact by
writing the priority mask and reading it back [source][LINUX]. The secure firmware
therefore owns group 0, and takes its interrupts at EL3 [inferred].

In that configuration the group registers, `GICD_IGROUPR<n>` and `GICR_IGROUPR0`, read as
zero to the non-secure world and ignore its writes [source][GIC]. With `SCR_EL3.FIQ` set,
an EL1 or EL2 access to the CPU interface's group 0 registers traps to EL3 [source][GIC].
Those registers are `ICC_IAR0_EL1`, `ICC_EOIR0_EL1`, `ICC_HPPIR0_EL1`, `ICC_BPR0_EL1`, the
`ICC_AP0R<n>_EL1` and `ICC_IGRPEN0_EL1`. At EL1, `HCR_EL2.FMO` takes precedence, and sends
the access to the virtual `ICV_` registers instead.

On the reference unit both rules hold [measured, 2026-10-09, u-boot 2026.10][PROBE]:

- `GICR_IGROUPR0` reads 0 and ignores a write.
- A read of `ICC_IGRPEN0_EL1` at EL2 is followed at once by the SPL's banner. Rockchip's
  BL31 resets the board on that trap [inferred].

> **A kernel takes every interrupt as group 1, signalled as an IRQ.** Some code assumes a
> GIC whose group 0 belongs to the non-secure world, such as an FIQ-based timer path. It
> resets the board at its first group 0 access [inferred].

The firmware leaves the distributor with affinity routing on and no non-secure group
enabled. `GICD_CTLR` reads `0x10` when u-boot starts a program [measured, 2026-10-09,
u-boot 2026.10][PROBE]. A kernel enables group 1 there itself [inferred].

The virtual timer reaches the boot core as a group 1 interrupt. A probe at EL2 enables
INTID 27 at priority `0xa0` and fires the timer 1 ms out. `ICC_IAR1_EL1` then acknowledges
INTID 27 [measured, 2026-10-09, u-boot 2026.10][PROBE]. As `CPU_ON` starts a core, the
firmware makes each of its non-secure PPIs group 1. The other cores therefore take the
timer the same way [inferred].

## Timers

The architected timer interrupts are PPIs. The manual's interrupt list and the tree agree,
the tree's PPI numbers sitting 16 below the INTIDs [source][TRM][DTS]:

| Timer | INTID | Tree's PPI |
|---|---|---|
| Hypervisor physical | 26 | 10 |
| Virtual | 27 | 11 |
| Hypervisor virtual | 28 | 12 |
| Secure physical | 29 | 13 |
| Non-secure physical | 30 | 14 |

Each is level-sensitive, active high [source][DTS].

The cores take their count from a high-precision timer block, the HPTIMER. Its clock runs
at 24 MHz, and drops to 32 kHz in low-power states, after which the block corrects the
count [source][TRM]. The count advances at 24 MHz under an operating system [inferred],
and the generic counter runs through a system sleep (see
[the PMU timer](system-sleep.md#the-pmu-timer)).

`CNTFRQ_EL0` reads 24,000,000 when u-boot starts a program [measured, 2026-10-09, u-boot
2026.10][PROBE]. The tree's timer node carries no `clock-frequency`, so Linux takes the
rate from `CNTFRQ_EL0` [source][LINUX], and reports its counter at 24.00 MHz [measured,
2026-10-08, 7.2.9][DMESG].

The count does not start at power-on. On the reference unit the counter reads 5.45 s as
Linux starts [measured, 2026-10-03][BOOTTIME]. A serial capture of the same boot shows the
SPL's banner at least 7.7 s before that. The count therefore starts after the SPL hands
off [inferred]. A boot time read from the counter is a lower bound.

The PMU timer, a separate two-channel block in the PMU's power domain, is owned by the
[RK3588 system sleep reference](system-sleep.md#the-pmu-timer).

## UARTs

The RK3588 carries ten UARTs. Each is `"rockchip,rk3588-uart", "snps,dw-apb-uart"`, a
DesignWare APB UART with `0x100` bytes of registers [source][DTS]. uart0 sits at
`0xfd89_0000`. uart1 to uart9 sit at `0xfeb4_0000` to `0xfebc_0000`,
`0x1_0000` apart. Their interrupts are INTIDs 363 to 372, uart0 to uart9 in order
[source][TRM].

Each UART carries a 64-byte FIFO in each direction. Its registers are 32-bit words, 4
bytes apart. The first eight sit where a 16550's do, from the transmit and receive buffer
at `0x00` to the scratch register at `0x1c` [source][TRM]. DesignWare's own registers
follow:

| Register | Offset | Reset value | Use |
|---|---|---|---|
| `USR` | `0x7c` | `0x6` | Status. Bit 0 is busy, bit 1 transmit FIFO not full, bit 2 transmit FIFO empty |
| `TFL`, `RFL` | `0x80`, `0x84` | 0 | FIFO fill levels |
| `SRR` | `0x88` | 0 | Software reset |
| `CPR` | `0xf4` | `0x0004_3ff2` | Component parameters |
| `UCV` | `0xf8` | `0x3430_322a` | Component version, ASCII `402*` |
| `CTR` | `0xfc` | `0x4457_0110` | Component type, `DW` in its top bytes |

> **`LCR`'s divisor-latch bit and the divisor latch accept writes only while `USR` bit 0
> reads zero** [source][TRM]. A driver that sets the line rate waits for the UART to go
> idle first.

The baud rate is the serial clock over 16 times the divisor [source][TRM]. From a 24 MHz
clock, 115200 baud takes a divisor of 13, 0.16 % fast [inferred].

Polled operation needs three of the 16550's registers: the receive and transmit buffers at
`0x00`, and the line status register, `LSR`, at `0x14`. `LSR` bit 0 reports a received
byte. `LSR` bit 5 reports an empty transmit holding register or FIFO while `IER` bit 7,
the programmable THRE mode, is clear. With that mode set and the FIFOs on, bit 5 reports a
full transmit FIFO instead [source][TRM]. Mainline u-boot's 16550 driver writes `IER` as 0
[source][UBOOT].

A bare-metal program prints on the reference unit's uart9 through the transmit buffer and
`LSR` bit 5 alone. It makes 32-bit accesses, and keeps u-boot's line settings [measured,
2026-10-08, u-boot 2026.10][LOADER]. Which UART carries the console is a board property (see
[the RK1's console](turing-rk1/hardware-reference.md#console)).

## TRNG

The SoC carries a true random number generator of its own, Rockchip's TRNG v1, at
`0xfe37_8000` as `rockchip,rk3588-rng` [source][DTS]. It stands apart from the crypto
engine. Its interrupt is `GIC_SPI 400`, INTID 432 [source][DTS]. Its clock,
`SCMI_HCLK_SECURE_NS`, and its reset belong to the secure firmware, which serves them
through SCMI [source][DTS].

Mainline Linux and u-boot drive it through these registers [source][LINUX][UBOOT]:

| Offset | Register | Use |
|---|---|---|
| `0x00` | `CTRL` | 1 asks for a number, 0 stops the block |
| `0x04` | `STAT` | Bit 9 seeded, bit 30 generating, bit 31 reseeding |
| `0x08` | `MODE` | Bit 3 set: numbers of 256 bits |
| `0x14` | `ISTAT` | Bit 0: a number is ready. Writing a bit back clears it |
| `0x20` to `0x3c` | `RAND0` to `RAND7` | The number, eight words |
| `0x60` | `AUTO_RQSTS` | Reseed after this many numbers of 16 bytes |
| `0xf0` | `VERSION` | Reads `0x46bc` |

Linux waits for `STAT` to show the block seeded and idle before its first number
[source][LINUX]. Mainline u-boot's driver turns no clock on before it reads the block
[source][UBOOT].

At u-boot's prompt on the reference unit, `VERSION` reads `0x46bc`, and `STAT` reads
`0x0804_0308`: seeded, neither generating nor reseeding [measured, 2026-10-09, u-boot
2026.10][TRNGREGS]. The block's clock is therefore on when u-boot starts a program, though
nothing at the prompt has turned it on [inferred]. A bare-metal program started with `go`
takes eight numbers of 32 bytes from it. None is all one byte value, and none repeats the
one before [measured, 2026-10-09, u-boot 2026.10][FIRSTPROC].

Under Linux the TRNG is the `hwrng` source `rockchip-rng`, with a quality of 999. Reading
1 MiB from `/dev/hwrng` takes 0.30 s, about 3.4 MiB/s, and the bytes carry 7.9998 bits of
Shannon entropy each [measured, 2026-10-23, 7.2.9][RNGRATE]. The entropy figure is a
sanity check, not a test of the source.

Neither core kind implements `RNDR` (see [ID registers](cpu-reference.md#id-registers)),
so the TRNG is the hardware entropy a bootloader can hand on.

## IOMMUs

The SoC has no one IOMMU for its on-chip masters. Each media and accelerator block that
translates addresses carries an MMU of its own, in front of its AXI master [source][DTS].
A fault therefore names the block that took it, and a block's MMU is set up, powered and
reset with that block. PCIe is the exception: its controllers share an Arm MMU-600, an
SMMUv3.

| MMU | Master | Banks | Interrupt | Power domain | In the dtsi |
|---|---|---|---|---|---|
| `iommu@fc900000` | The PCIe controllers | One window of `0x20_0000` | `GIC_SPI` 367, 369, 371 and 374, edge | | Enabled |
| `iommu@fcb00000` | None in the dtsi | One window of `0x20_0000` | `GIC_SPI` 379, 381, 383 and 386, edge | | Disabled |
| `iommu@fdab9000` | NPU core 0 | Two of `0x100` | `GIC_SPI 110`, the core's own | `PD_NPUTOP` | Disabled |
| `iommu@fdaca000`, `iommu@fdada000` | NPU cores 1 and 2 | One of `0x100` | `GIC_SPI` 111 and 112, the cores' own | `PD_NPU1`, `PD_NPU2` | Disabled |
| `iommu@fdb50800` | VDPU2 decoder | One of `0x40` | `GIC_SPI 118` | `PD_VDPU` | Enabled |
| `iommu@fdb60f00`, `iommu@fdb70f00` | RGA3 cores 0 and 1 | One of `0x100` | `GIC_SPI` 114 and 115, the cores' own | `PD_RGA30`, `PD_RGA31` | Enabled |
| `iommu@fdba0800` to `iommu@fdbac800` | The four VEPU121 JPEG encoders | One of `0x40` | `GIC_SPI` 121, 123, 125 and 127 | `PD_VDPU` | Enabled |
| `iommu@fdc38700`, `iommu@fdc40700` | rkvdec cores 0 and 1 | Two of `0x40` | `GIC_SPI` 96 and 98 | `PD_RKVDEC0`, `PD_RKVDEC1` | Enabled |
| `iommu@fdca0000` | AV1 decoder | One of `0x600` | `GIC_SPI 109` | `PD_AV1` | Enabled |
| `iommu@fdce0800` | VICAP | Two of `0x40` | `GIC_SPI 113` | `PD_VI` | Disabled |
| `iommu@fdd97e00` | VOP | Two of `0x100` | `GIC_SPI 156`, the VOP's own | `PD_VOP` | Disabled |

Each PCIe controller maps its requester IDs to `iommu@fc900000` through `iommu-map`. Two
controllers sit in `rk3588-base.dtsi`, and the full RK3588 adds three in
`rk3588-extra.dtsi` [source][DTS].

Linux's `arm-smmu-v3` driver finds a 48-bit output address size in `iommu@fc900000`, and
features `0x001c_1ebf` [measured, 2026-10-05, 7.2.9][BOOTLOG]. It finds no MSI domain for
the SMMU, so the SMMU signals through its wired interrupts. Every function behind one root
port shares an IOMMU group with the port. The MMUs of blocks that mainline does not describe, the
VEPU580 encoders and the IEP among them, are in the
[RK3588 video reference](video-reference.md#blocks-without-a-node).

### The Rockchip MMU

Every MMU in the table but the two MMU-600s and the AV1 decoder's is Rockchip's own:
`"rockchip,rk3588-iommu", "rockchip,rk3568-iommu"`, driven by `rockchip-iommu`
[source][LINUX]. A node with two banks carries two MMUs, which the driver programs
identically [source][LINUX]. Each takes its master's AXI and AHB clocks.

Each bank carries these 32-bit registers [source][LINUX]:

| Offset | Register | Use |
|---|---|---|
| `0x00` | `DTE_ADDR` | The directory's address |
| `0x04` | `STATUS` | Bit 0 paging on, bit 1 page fault active, bit 2 stall active, bit 3 idle, bit 4 replay buffer empty, bit 5 the fault was a write |
| `0x08` | `COMMAND` | 0 enable paging, 1 disable paging, 2 enable stall, 3 disable stall, 4 invalidate the TLB, 5 page fault done, 6 force reset |
| `0x0c` | `PAGE_FAULT_ADDR` | The IOVA of the last fault |
| `0x10` | `ZAP_ONE_LINE` | Invalidates the TLB entry of the IOVA written |
| `0x14` | `INT_RAWSTAT` | Interrupt status before the mask |
| `0x18` | `INT_CLEAR` | Acknowledges the bits written |
| `0x1c` | `INT_MASK` | Bit 0 page fault, bit 1 bus error |
| `0x20` | `INT_STATUS` | Interrupt status after the mask |
| `0x24` | `AUTO_GATING` | Clock gating control |

The MMU walks two levels of 4 KiB tables, each of 1,024 32-bit entries. IOVA bits 31:22
index the directory, bits 21:12 the page table, and bits 11:0 are the offset in the page
[source][LINUX]. The IOVA space is therefore 4 GiB, and Linux fixes each domain's aperture
at 0 to 4 GiB.

The second-generation entry format packs a 40-bit physical address into 32 bits
[source][LINUX]:

| Bits | Directory entry | Page entry |
|---|---|---|
| 31:12 | Page table address, bits 31:12 | Page address, bits 31:12 |
| 11:8 | Address bits 35:32 | Address bits 35:32 |
| 7:4 | Address bits 39:36 | Address bits 39:36 |
| 3 | Reserved | Security |
| 2 | Reserved | Writable |
| 1 | Reserved | Readable |
| 0 | Valid | Valid |

`DTE_ADDR` takes the directory's address in the same packed form. The directory and the
page tables can therefore sit anywhere in a 40-bit space, and Linux allocates them with no
zone limit [source][LINUX]. A master behind this MMU addresses 4 GiB of IOVA and reaches
all of DRAM through it.

### Enabling the MMU

Linux enables an MMU as a domain is attached, in this order [source][LINUX]:

1. It turns the clocks on, and stalls the MMU if paging is on. A stall needs paging, so an
   MMU with paging off is not stalled.
2. It checks that `DTE_ADDR` responds, by writing a dummy address and reading it back. It
   then issues the force reset, and polls until `DTE_ADDR` reads 0.
3. For each bank, it writes `DTE_ADDR`, invalidates the TLB, unmasks both interrupts and
   sets `AUTO_GATING` bit 31.
4. It enables paging, polls `STATUS` bit 0, and releases the stall.

A node carrying `rockchip,disable-mmu-reset` skips the force reset, because some instances
misbehave under it [source][BINDING]. The dtsi sets it on the VICAP's MMU [source][DTS].
Linux names `AUTO_GATING` bit 31 `DISABLE_FETCH_DTE_TIME_LIMIT`, and sets it as a
workaround for a blocked MMU [source][LINUX].

Disabling runs the reverse: stall, disable paging, write `INT_MASK` and `DTE_ADDR` as 0,
and release the stall.

On a page fault, Linux reads the IOVA from `PAGE_FAULT_ADDR` and the direction from
`STATUS` bit 5, and reports the fault. It then invalidates the TLB, issues page fault done
and writes the status back to `INT_CLEAR` [source][LINUX]. A bus error is logged and
cleared, with no command. The driver requests its interrupt shared. The NPU cores, the
RGA3 cores and the VOP share their line with their MMU [source][DTS].

### MMU state across power and reset

An MMU loses its state with its master. Linux ties the master's runtime power management
to the MMU's through a device link [source][LINUX]. At runtime suspend it disables the MMU,
and at resume it runs the whole enable sequence again. While an MMU is suspended, an unmap
skips its TLB invalidation, and the resume's invalidation covers it.

A decoder's own reset clears its MMU as well [inferred]. Mainline rkvdec therefore
reprograms the decoder's MMU after a failed job [source][LINUX]. It attaches an empty
domain and detaches it, which restores the default domain and its mappings. On the VDPU381
decoder it runs when the interrupt status at `0x380` carries bit 4 (error), bit 5
(timeout) or bit 9 (soft reset done). The NPU's reset clears its MMU in the same
way (see [detaching a wedged core's MMU](npu-reference.md#detaching-a-wedged-cores-mmu)).

On a Rock 5B decoding corrupt streams, the decoder's status read `0x23`, bits 0, 1 and 5,
and `0xb`, bits 0, 1 and 3. Bit 9 never appeared [reported][YSP]. Bit 3 is the AXI bus
error. Rockchip's register description, as [YSP] quotes it, makes bit 3 reset the decoder
as bit 5 does [reported]. Mainline's restore leaves bit 3 out. A job that fails on bit 3
alone is then expected to leave the MMU unprogrammed [inferred].

### The AV1 decoder's MMU

The AV1 decoder's MMU is a Verisilicon IOMMU,
`"rockchip,rk3588-av1-iommu", "verisilicon,iommu-1.2"`, driven by `vsi-iommu` rather than
`rockchip-iommu` [source][LINUX]. Its format differs:

- A 64-bit table-array entry carries the directory's address, bits 31:10, and a page mode, 0
  for 4 KiB. The register at `0x38c` takes that entry's address.
- Directory entries are 32 bits: a page table address aligned to 64 bytes, and a valid bit.
- Page entries are 32 bits. Bits 31:12 carry page address bits 31:12, and bits 11:4 carry
  address bits 39:32. Bit 2 is writable and bit 0 valid. There is no readable bit.

Linux allocates every table of this MMU below 4 GiB, and fixes its aperture at 0 to 4 GiB
[source][LINUX]. It drives these registers:

| Offset | Use |
|---|---|
| `0x184` | Bit 4 invalidates the whole TLB. The driver invalidates nothing finer |
| `0x1ac` | Bit 28 set at enable: out-of-bound reporting |
| `0x380` | Bit 0 set at enable. Reads back the IOVA of a fault |
| `0x384` | Interrupt status, three bits. Written 0 to acknowledge |
| `0x388` | Bit 0 enables the MMU |

The kernel symbol `CONFIG_VSI_IOMMU` has no default, and the AV1 decoder fails to probe
without it (see [kernel configuration](video-reference.md#kernel-configuration)).

### Masters with no IOMMU

RGA2 has no IOMMU. It walks a page table of its own whose entries are 32-bit physical
addresses, so it reaches the low 4 GiB only (see [RGA](video-reference.md#rga)).

The Ethernet MACs have no IOMMU either. GMAC1 reports a 32-bit DMA address width in its
hardware feature register, and Linux uses 32 bits for host and device [measured,
2026-10-05, 7.2.9][BOOTLOG]. It reaches the low 4 GiB only. On a board with more memory, a
packet buffer above 4 GiB goes through the 64 MiB software bounce buffer Linux sets up at
boot [inferred].

## DMA controllers

The SoC carries three Arm PL330 DMA controllers. Linux reads the same configuration from
each [measured, 2026-10-05, 7.2.9][BOOTLOG]:

| Controller | Base |
|---|---|
| DMAC 0 | `0xfea1_0000` |
| DMAC 1 | `0xfea3_0000` |
| DMAC 2 | `0xfed1_0000` |

Each has 8 channels, 32 peripheral request lines and 16 events, with a data buffer of 128
entries of 8 bytes. The UARTs, SPI and audio interfaces take their request lines from these
controllers [source][DTS].

## DRAM bandwidth

The firmware runs the DRAM clock at 2112 MHz, by its SCMI clock `scmi_clk_ddr`, under BL31
v1.56 and v1.51 alike [measured, 2026-10-23, 7.2.9][CLK]. The DDR monitor counts four channels (see
[DDR traffic counters](video-reference.md#ddr-traffic-counters)), and spreads a streaming
read or write across them within 0.2 % [measured, 2026-10-23, 7.2.9][DDRCH].

Streaming bandwidth from the cores reads as follows on the reference unit. Each thread works
on its own 256 MiB buffers, 16 times [measured, 2026-10-23, 7.2.9][BW]:

| Threads | Read | Write | Copy |
|---|---:|---:|---:|
| One Cortex-A55 | 6.07 GB/s | 22.81 GB/s | 12.15 GB/s |
| One Cortex-A76 | 19.21 GB/s | 30.58 GB/s | 23.56 GB/s |
| Four Cortex-A55s | 22.94 GB/s | 29.83 GB/s | 23.62 GB/s |
| Four Cortex-A76s | 27.77 GB/s | 30.42 GB/s | 24.00 GB/s |
| All eight | 24.60 GB/s | 29.28 GB/s | 21.96 GB/s |

A second RK1 module, with slower clocks, reads and writes 4 to 9 % lower. Its copies land
within 5 % of these (see [delivered frequency](cpu-reference.md#delivered-frequency))
[measured][BW2].

Copy counts the bytes read and the bytes written. Every policy ran at `performance`, which
puts the DSU at 1800 MHz [inferred]. The DSU's clock follows the Cortex-A55s and sets these figures,
as [the DSU clock](cpu-reference.md#the-dsu-clock-follows-the-cortex-a55s) shows.

Four 16-bit channels at 4224 MT/s carry 33.8 GB/s [inferred]. The best write reaches 90 % of
that, and the best read 82 %. Stores need no read for ownership. On the second module, a 4 GiB
fill reads 0.1 GB from DRAM on a Cortex-A55 and 0.6 GB on a Cortex-A76. A copy reads only its
source [measured, 2026-10-23, 7.2.9][BW2].

The monitor's counters agree with the program's traffic, on the second module. Its write
count lands within 1 % of the bytes written. Its read count lands 3 to 9 % above the bytes
read [measured, 2026-10-23, 7.2.9][BW2]. An idle system moves 1.7 MB/s.

## Thermal sensors

The TSADC at `0xfec0_0000` reads seven sensors, on a 2 MHz clock [source][DTS]. The tree
gives each sensor a thermal zone, with a critical trip at 115 °C:

| Channel | Zone | Passive trip and cooling |
|---|---|---|
| 0 | `package-thermal` | None in the SoC tree. A board adds its fan here |
| 1 | `bigcore0-thermal` | 85 °C, the `cpu4` policy |
| 2 | `bigcore2-thermal` | 85 °C, the `cpu6` policy |
| 3 | `littlecore-thermal` | 85 °C, the `cpu0` policy |
| 4 | `center-thermal` | None |
| 5 | `gpu-thermal` | 85 °C, the GPU's devfreq |
| 6 | `npu-thermal` | None |

Each passive trip has 2 °C of hysteresis, and its zone polls every 100 ms while it cools.
Every zone's `polling-delay` is 0 [source][DTS]. A zone therefore learns of a trip from the
TSADC's interrupt [inferred]. At 120 °C the TSADC resets the
SoC through the CRU, with no software involved: `rockchip,hw-tshut-mode` is 0, the CRU
[source][DTS].

All seven zones read 37 to 38 °C on an idle RK1 module in its carrier, with the fan off
[measured, 2026-10-23, 7.2.9][THERMAL]. After the bandwidth runs they read 40.7 to 42.5 °C,
the three CPU zones highest.

## Sources

Citation keys:

- `[TRM]`: Rockchip, *RK3588 Technical Reference Manual*, Rev 1.0, Part 1. Chapter 1 gives
  the address map in Table 1-1 and the interrupt list in Table 1-3. Chapter 10 gives the
  timer clocks in section 10.3.1. Chapter 11 gives the GIC-600 configuration in Table
  11-1, and chapter 19 the UART registers in section 19.4. Part 2, section 1.3.5 and Table
  1-5, gives the DRAM channels' address space.
- `[DTS]`: Linux `arch/arm64/boot/dts/rockchip/rk3588-base.dtsi` at tag `v7.2.9`, and
  `rk3588-extra.dtsi` for the full RK3588's three further PCIe controllers.
- `[LINUX]`: Linux at tag `v7.2.9`. `drivers/irqchip/irq-gic-v3.c` (`gic_has_group0`),
  `drivers/clocksource/arm_arch_timer.c` (`arch_timer_of_configure_rate`),
  `drivers/char/hw_random/rockchip-rng.c` (`rk3588_rng_init`, `rk3588_rng_read`), and the
  `ROCKCHIP_ERRATUM_3588001` entry in `arch/arm64/Kconfig`. For the IOMMUs,
  `drivers/iommu/rockchip-iommu.c` (the register and entry definitions,
  `rk_iommu_enable`, `rk_iommu_disable`, `rk_iommu_force_reset`, `rk_iommu_irq`,
  `rk_iommu_zap_iova`, `rk_iommu_probe_device`, `rk_iommu_domain_alloc_paging` and
  `iommu_data_ops_v2`), `drivers/iommu/vsi-iommu.c`, and
  `drivers/media/platform/rockchip/rkvdec/rkvdec.c` (`rkvdec_iommu_restore`,
  `vdpu381_irq_handler`) with `rkvdec-vdpu381-regs.h`.
- `[BINDING]`: Linux `Documentation/devicetree/bindings/iommu/rockchip,iommu.yaml` at tag
  `v7.2.9`, property `rockchip,disable-mmu-reset`.
- `[YSP]`: Yi Ding, rock-5b-ysp, https://github.com/yisding/rock-5b-ysp, at commit
  `69a3abbf`, a Rock 5B project whose documentation is licensed CC-BY-SA-4.0. Its finding of
  2026-08-01 on the decoder's self-reset, measured under a vendor-derived 6.18 kernel. The
  facts are restated here, not quoted.
- `[UBOOT]`: u-boot at commit `ece349a`. `drivers/rng/rockchip_rng.c` (`rk_trngv1_init`,
  `rk_trngv1_rng_read`). `drivers/serial/ns16550.c`, whose `ns16550_init` and
  `_debug_uart_init` write `IER` as `CFG_SYS_NS16550_IER`, 0 unless a board configuration
  defines it.
- `[GIC]`: Arm, *GIC Architecture Specification, GICv3 and GICv4*, IHI 0069H.b. The
  accessibility pseudocode of the six group 0 CPU interface registers named in the text,
  and the descriptions of `GICD_IGROUPR<n>` and `GICR_IGROUPR0`.

Captures from the reference unit, kept with its evidence outside this repository:

- `[IOMEM]`: `/proc/iomem` read as root under Linux 7.2.9, 2026-10-08.
- `[DMESG]`: the kernel log of the same boot.
- `[PROBE]`: a bare-metal probe of the EL2, GIC and timer state, run from u-boot 2026.10
  with `go`, 2026-10-09.
- `[LOADER]`: a bare-metal program's report, printed on uart9 from u-boot 2026.10 with
  `go`, 2026-10-08.
- `[TRNGREGS]`: `md.l` of the TRNG's `VERSION` and `STAT` at u-boot 2026.10's prompt,
  2026-10-09.
- `[BOOTTIME]`: a serial capture of one boot from power-on, timestamped on the capturing
  host, 2026-10-03. Beside it, the counter's value as Linux started, read from
  `cntvct_el0` against `CLOCK_MONOTONIC_RAW`.
- `[FIRSTPROC]`: a bare-metal loader run from u-boot 2026.10, 2026-10-09, which reads the
  TRNG for a kernel's seed.

Captures from a second Turing RK1 module, kept with its evidence outside this repository. It
is a 32 GB module whose eMMC carries u-boot 2026.07, BL31 v1.51 and DDR firmware v1.19:

- `[BOOTLOG]`: the kernel log of a boot of Linux 7.2.9, 2026-10-05, read from the journal.
- `[CLK]`: `/sys/kernel/debug/clk/clk_summary` on the same boot, 2026-10-23, and on the
  reference unit the same day.
- `[RNGRATE]`: 256 reads of 4 KiB from `/dev/hwrng` by `dd`, timed, 2026-10-23, with the
  byte histogram's entropy computed over the result.
- `[BW2]`: a streaming read, fill and copy of 256 MiB per thread, 16 times, with threads
  pinned to the cores named, 2026-10-23. `perf stat -a` counted the `rockchip_ddr` PMU's
  `read-bytes` and `write-bytes` over each run, setup included.
- `[DDRCH]`: the same read and fill on four Cortex-A76s, counted per channel, 2026-10-23.
- `[THERMAL]`: `/sys/class/thermal` read before and after the bandwidth runs, 2026-10-23.

Captures from the reference unit under Linux 7.2.9 with BL31 v1.56, kept with its evidence
outside this repository:

- `[BW]`: the same streaming read, fill and copy, from static builds of the same program,
  with every cpufreq policy at `performance`, 2026-10-23.
