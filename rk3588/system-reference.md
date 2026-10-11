# RK3588 system reference

This document covers the system blocks a bootloader or a new kernel reaches before any
other driver. They are the address map, the interrupt controller, the timers, the UARTs
and the true random number generator. It states each block's addresses, configuration and
registers, and the state the boot firmware leaves it in.

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

`RNDR` gives the boot core no random number (see [ID registers](cpu-reference.md#id-registers)),
so the TRNG is the hardware entropy a bootloader can hand on.

## Sources

Citation keys:

- `[TRM]`: Rockchip, *RK3588 Technical Reference Manual*, Rev 1.0, Part 1. Chapter 1 gives
  the address map in Table 1-1 and the interrupt list in Table 1-3. Chapter 10 gives the
  timer clocks in section 10.3.1. Chapter 11 gives the GIC-600 configuration in Table
  11-1, and chapter 19 the UART registers in section 19.4. Part 2, section 1.3.5 and Table
  1-5, gives the DRAM channels' address space.
- `[DTS]`: Linux `arch/arm64/boot/dts/rockchip/rk3588-base.dtsi` at tag `v7.2.9`.
- `[LINUX]`: Linux at tag `v7.2.9`. `drivers/irqchip/irq-gic-v3.c` (`gic_has_group0`),
  `drivers/clocksource/arm_arch_timer.c` (`arch_timer_of_configure_rate`),
  `drivers/char/hw_random/rockchip-rng.c` (`rk3588_rng_init`, `rk3588_rng_read`), and the
  `ROCKCHIP_ERRATUM_3588001` entry in `arch/arm64/Kconfig`.
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
