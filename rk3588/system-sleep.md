# RK3588 system sleep

This document covers system sleep on the RK3588 from the side of an engineer implementing
suspend in a kernel or a bootloader. It states the SoC's low power modes and the PMU's wake
sources, and the PMU timer that runs through a sleep. It then states how Rockchip's BL31
configures a PSCI `SYSTEM_SUSPEND`, which wake sources it arms, and which of them wake the
reference unit.

The document is SoC-level. Register facts come from Rockchip's technical reference manual
[source][TRM]. The firmware's behavior comes from a disassembly of rkbin's BL31 v1.56
[BLOB]. It also comes from runs on the reference unit, a Turing RK1, under that BL31 and
mainline u-boot 2026.10. Runs that call `SYSTEM_SUSPEND` from a bare-metal program at EL2 are
the main measurement, and two kernel runs at EL1 add to them. The
[RK3588 boot firmware](boot-firmware.md) owns PSCI and the SiP service as a whole.

## Low power modes

The RK3588 defines two low power modes [source][TRM]:

- Power mode 0, ultra low power. Every logic but ALIVE in VD_PMU0 can power down. A GPIO
  interrupt from an IO in the PMUIO1 domain wakes the chip.
- Power mode 1, low power. ALIVE and PD_PMU1 stay on. A GPIO interrupt from the PMUIO1 or
  PMUIO2 domain wakes the chip, or another wake source in PD_PMU1.

## The PMU's wake sources

The PMU sits at `0xfd8d_0000`, and its registers from offset `0x4000` sit in PD_PMU1
[source][TRM]. There, `PMU_WAKEUP_INT_CON`, at offset `0x4010`, enables each of power mode
1's wake sources. Each source takes one bit, and every bit is off at reset [source][TRM]:

| Bits | Wake source |
|---|---|
| 0 to 7 | An interrupt to CPU0 to CPU7 |
| 8 | GPIO0's interrupt |
| 9 | SDMMC's card detect |
| 10 | SDIO's interrupt |
| 11 | USB's detect |
| 12 | UART0's interrupt |
| 13 | The voice activity detector's interrupt |
| 14 | The PMU_TIMER's interrupt |
| 15 | A system interrupt |
| 16 | The PMU's wake-up timeout, counted in `PMU_WAKEUP_TIMEOUT_CNT` |

Under Rockchip's BL31 the wake register belongs to the secure world. A read of
`PMU_WAKEUP_INT_CON` from EL2 ends in a synchronous external abort [measured, 2026-10-10,
u-boot 2026.10][SLEEP]. Mainline TF-A's RK3588 port marks the PMU0 and PMU1 register
blocks and the PMU's SRAM secure [source][TFAMAIN]. The PMU2 registers answer from EL2: `PMU2_PWR_GATE_ST0`,
at `0xfd8d_8180`, reads `0x0011_fff9` [measured, 2026-10-10, u-boot 2026.10][MODES].

> **A kernel can neither read nor set the PMU's wake sources.** Only the firmware's SiP
> call sets them, and nothing reports afterwards which source fired [inferred].

## The PMU timer

The PMU timer, `TIMER_PMU`, is a block of two channels at `0xfd8f_0000`. Both channels
count up at 24 MHz [source][TRM]. It sits in PD_PMU1 with the PMU [source][TRM], so it
keeps power in power mode 1 [inferred]:

| Item | Value |
|---|---|
| Clock | Bits 7 and 8 of `PMU1CRU_CLKSEL_CON02`: the 24 MHz oscillator at reset, or the 32 kHz deep-sleep clock |
| Interrupts | INTIDs 319 and 320, channels 0 and 1 |
| The channel that wakes the PMU | Bit 5 of `PMU1GRF_SOC_CON0`, channel 0 at reset |

These registers answer the non-secure world [measured, 2026-10-10, u-boot
2026.10][MODES]. On the reference unit, the timer's revision reads `0x1565_0302`, and a
channel counts 24.000 MHz against the generic counter. `PMU1CRU_CLKSEL_CON02` reads
`0x420`, the oscillator, and bit 5 of `PMU1GRF_SOC_CON0` reads 0, channel 0.

The timer counts through a sleep. It and the generic counter agree on each sleep's length
to within 1 ms [measured, 2026-10-10, u-boot 2026.10][MODES]. The generic counter
therefore runs through a sleep too. The cores' generic timers do not wake a core that is powered
down [expected]. The PMU timer is the timer to arm for a timed wake. A generic timer's
interrupt that comes due as the firmware takes the system down still wakes it [inferred] (see
[the calling core's own interrupts](#the-calling-cores-own-interrupts)).

In the user-defined count mode a channel does not reload to 0 at its load count
[source][TRM]. A channel loaded with 72,000,000 read 72,000,000 when a kernel took its
interrupt, after the resume [measured, 2026-10-11, u-boot 2026.10][TIMERWAKE]. The kernel reads
the count in its interrupt handler, which runs only after the resume. So on the reference unit
a channel holds at its load count [inferred].

## Rockchip's BL31 and the wake sources

Rockchip's BL31 configures a `SYSTEM_SUSPEND` from two words it keeps. One is a sleep mode.
The other is a set of wake sources, whose bits follow `PMU_WAKEUP_INT_CON`
[inferred][BLOB]. The vendor kernel sets both through one SiP call, `SIP_SUSPEND_MODE`,
function `0x8200_0003`, with a command in `x1` and its argument in `x2` [source][VENDOR]:

| Command | Name | Argument |
|---|---|---|
| `0x01` | `SUSPEND_MODE_CONFIG` | The sleep mode, in `RKPM_SLP_*` bits |
| `0x02` | `WKUP_SOURCE_CONFIG` | The wake sources, in `RKPM_*_WKUP_EN` bits, which follow the register's |
| `0x08` | `SUSPEND_WFI_TIME_MS` | A timeout, in milliseconds |

Mainline Linux and mainline u-boot make no such call, so the firmware's own words stand.
BL31 v1.56 starts with these [inferred][BLOB]:

- The sleep mode `0x0400_0024`. Bit 2 is `RKPM_SLP_ARMOFF_DDRPD`: the cores off, and DRAM
  powered down. Bit 26 is `RKPM_SLP_PMU_DBG` [source][VENDOR]. The firmware's log calls
  bit 5 `slp_from_uboot`, and Rockchip's headers leave it unnamed.
- The wake sources `0x101`: `RKPM_CPU0_WKUP_EN` and `RKPM_GPIO_WKUP_EN` [source][VENDOR].

For the suspend, the firmware writes bits 8 to 16 of the wake sources to
`PMU_WAKEUP_INT_CON` as they stand. Bit 0 moves to the bit of the core that made the call
[inferred][BLOB]. By default, an interrupt to the calling core or GPIO0's interrupt
therefore wakes the system. Mainline TF-A's RK3588 port, which Rockchip contributed, arms
CPU0's interrupt and GPIO0's interrupt, whichever core makes the call. It takes no call to
change them [source][TFAMAIN].

BL31 v1.56 accepts commands `0x01` and `0x02` without checking the argument, and refuses
`0x08` with -1 [inferred][BLOB]. Other details of the firmware's handling:

- The sleep mode's `RKPM_SLP_TIME_OUT_WKUP`, bit 25, makes the PMU's timeout the only wake
  source. The firmware sets that timeout to one second itself, and no call sets a longer
  one [inferred][BLOB].
- A setting lasts until the next reset [inferred][BLOB].
- With bit 26 set, as by default, the firmware logs each suspend and the source that woke
  it on its own console [inferred][BLOB]. On the Turing RK1 that console is not the
  board's, uart9 at 115200 baud (see
  [the RK1's console](turing-rk1/hardware-reference.md#console)).

## Resume

BL31 resumes the calling core from `SYSTEM_SUSPEND` at EL2, with the MMU off, at the entry
the call names [measured, 2026-10-10, u-boot 2026.10][MODES]. A kernel at EL1 has made the
call and resumed [measured, 2026-10-11, u-boot 2026.10][TIMERWAKE]. Its entry runs at either
level, and the run does not show which. BL31 starts a core through `CPU_ON` the same way (see
[starting a core](boot-firmware.md#starting-a-core)).

> **The resume entry a kernel names runs at EL2, with the MMU off.** The firmware resumes
> the core that way whatever level made the call [inferred].

## Measured wakes

Each of three sources wakes the reference unit from BL31's `SYSTEM_SUSPEND`. A bare-metal
program made each call at EL2. It timed each sleep by the generic counter and by the PMU
timer's channel 1 [measured, 2026-10-10, u-boot 2026.10][MODES]:

| Wake source | The firmware's words | Asleep, by the generic counter |
|---|---|---|
| The PMU's timeout | Sleep mode `0x0600_0024` | 1173 ms, and 1174 ms by the PMU timer |
| The PMU timer's channel 0, as `RKPM_TIMER_WKUP_EN` | Wake sources `0x4101` | 4993 ms, for an alarm 5 s on |
| The PMU timer's INTID 319, routed to the calling core | The defaults | 4965 ms, for an alarm 5 s on |

The third row needs no call to the firmware. Under its defaults, an interrupt the GIC
routes to the calling core wakes the system, where the interrupt's device runs through
the sleep [inferred].

A kernel at EL1 armed the third row's route, under the defaults. Once its cores' own SGIs and
PPIs were off, it woke with its 3 s alarm rung [measured, 2026-10-11, u-boot
2026.10][TIMERWAKE] (see [the calling core's own interrupts](#the-calling-cores-own-interrupts)).

Whether GPIO0's interrupt wakes the system under the defaults is not settled. The Turing
RK1's real-time clock signals its alarm there, and is the one source tried (see
[the RK1's real-time clock](turing-rk1/hardware-reference.md#real-time-clock)).

## The calling core's own interrupts

Under BL31's defaults, an interrupt the GIC forwards to the calling core wakes the system.
That covers the core's own SGIs and PPIs, its generic timers' among them [inferred].

A kernel armed the PMU timer's channel 0 for 3 s, routed to the calling core, and took its
other seven cores off. It then called `SYSTEM_SUSPEND` at EL1. The kernel counted 174 ms
asleep, and the alarm was unrung [measured, 2026-10-11, u-boot 2026.10][EARLYWAKE]. That is
about what the firmware's way down and up take, since the PMU's one-second timeout took
1,173 ms [inferred]. The kernel's source leaves the calling core's virtual timer's PPI, INTID
27, and an SGI enabled as the core goes down [inferred]. The run prints no GIC state.

A rebuilt kernel disabled every SGI and PPI of each core as it went down, the calling core's
among them. It woke 3,041 ms after the call by the generic counter, with the alarm rung
[measured, 2026-10-11, u-boot 2026.10][TIMERWAKE]. Its other changes only report [inferred].
As the calling core went down, neither its enabled SGI nor its timer's PPI was pending. Its
timer was due 9.5 ms later [measured, 2026-10-11, u-boot 2026.10][TIMERWAKE]. If the first
run went down alike, its timer came due during the firmware's way down, and woke it
[inferred].

> **Disable the calling core's SGIs and PPIs before `SYSTEM_SUSPEND`.** Any of them enabled
> that becomes pending while the firmware takes the system down wakes it [inferred]. Leave
> only the wake alarm's SPI routed to the core, and enable the rest again after the resume.

## Sources

Citation keys:

- `[TRM]`: Rockchip, *RK3588 Technical Reference Manual*, Rev 1.0, Part 1: the PMU chapter
  (power modes, `PMU_WAKEUP_INT_CON`, `PMU_WAKEUP_TIMEOUT_CNT`), the timer chapter
  (`TIMER_PMU`), and the PMU1CRU and PMU1GRF register descriptions.
- `[BLOB]`: rockchip-linux/rkbin at commit `036a35a`, `bin/rk35/rk3588_bl31_v1.56.elf`,
  SHA-256 `2864c2fc36333e03e7e3fe6459225d320d310850b4ac7620d1f3601efb7d4a9b`. The claims
  rest on a disassembly of `suspend_mode_handler` and its command table, and of the boot
  code that stores the two words. They also rest on the system suspend, which writes
  `PMU_WAKEUP_INT_CON` and `PMU_WAKEUP_TIMEOUT_CNT`. The file carries no symbols, and the firmware's log strings
  name each function.
- `[VENDOR]`: the vendor kernel, rockchip-linux/kernel at commit `77168c8`, branch
  `develop-6.1`: `include/linux/rockchip/rockchip_sip.h` and
  `include/dt-bindings/suspend/rockchip-rk3588.h`.
- `[TFAMAIN]`: ARM-software/arm-trusted-firmware at commit `5103554` (2026-10-05):
  `plat/rockchip/rk3588/drivers/pmu/pmu.c` (`pmu_sleep_config`),
  `plat/rockchip/rk3588/plat_sip_calls.c`, and `plat/rockchip/rk3588/drivers/secure/secure.c`
  (`sgrf_init`).

Captures from the reference unit, kept with its evidence outside this repository:

- `[SLEEP]`: a bare-metal sleep probe run four times from u-boot 2026.10, 2026-10-10. Each
  run stopped at its read of `PMU_WAKEUP_INT_CON`.
- `[MODES]`: the same probe built to report a refused read and go on, run four times from
  u-boot 2026.10, 2026-10-10. The runs were a report, then the timeout, timer and interrupt
  modes.
- `[EARLYWAKE]`: a run of open-source XNU, booted from u-boot 2026.10 by a bare-metal loader,
  2026-10-11. The kernel arms the PMU timer's channel 0 for 3 s and calls `SYSTEM_SUSPEND`.
  Its source leaves the calling core's SGI 1 and PPI 27 enabled, and the run prints no GIC
  state.
- `[TIMERWAKE]`: a rebuilt kernel, run the same way the same day. It disables each core's SGIs
  and PPIs as the core goes down, and reports what the calling core had armed.
