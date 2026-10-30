# RK3576 boot firmware

This document covers the RK3576 boot firmware from the side of an engineer porting a
bootloader or an operating system. It describes the boot chain and its blobs, OP-TEE under
mainline u-boot, and a u-boot configuration trap around the blob variables. It also covers
the SARADC download key, the reboot-mode registers and what BL31 forwards, and the
deep-idle wakeup miss under rkbin BL31 v1.24.

The document is SoC-level. Hardware behavior is measured on the reference unit, an H96
MAX M9. Source reading covers the mainline u-boot v2026.04 and Linux 7.1 trees [source].
The BL31 analysis is a disassembly of the rkbin blob [BLOB].

## The boot chain

The chain runs BootROM -> TPL (DDR init) -> SPL -> BL31 (TF-A) -> BL32 (OP-TEE) -> U-Boot
-> kernel [source]. Vendor firmware runs every stage. Under mainline u-boot, BL32 is
optional, as [OP-TEE and BL32](#op-tee-and-bl32) describes.

The BootROM takes its boot order from a resistor divider on `SARADC_IN0`, read at reset.
[SARADC boot strap and recovery key](hardware-design-guide.md#saradc-boot-strap-and-recovery-key)
lists the orders. The BootROM reads the ID block (idbloader) at sector 64, marked by the
`RKNS` magic. If no strapped medium yields a loader, the BootROM falls through to USB
download, which is maskrom. Holding the eMMC clock at ground at power-on forces that
fall-through on an eMMC-strapped board [measured].

rkbin ships one unified DDR loader for the SoC, at v1.13
`rk3576_ddr_lp4_2112MHz_lp5_2736MHz_v1.13.bin`, which covers LPDDR4, LPDDR4X and LPDDR5.
There is no separate DDR4 file. 2112 MHz is the loader's native LPDDR4X target.

rkbin's newest RK3576 BL31 is v1.25. The reference unit's vendor firmware runs BL31 v1.09
with OP-TEE. Under mainline u-boot the reference unit runs rkbin BL31 v1.24 and DDR
firmware v1.12, without OP-TEE. Every BL31 measurement on this page is v1.24's.

## OP-TEE and BL32

Mainline u-boot boots the RK3576 without OP-TEE [source]. The vendor u-boot does need it:
a vendor build without BL32 hangs after `Starting kernel` [reported].

rkbin's BL32 blob cannot be packaged into a mainline image. Mainline's arm64 TEE FIT node
uses `fit,operation = "split-elf"` with `tee-os { optional; }`. binman therefore needs an
OP-TEE ELF, or a v1 `tee.bin` that carries the `OPTE\x01` header. rkbin's
`rk3576_bl32_*.bin` is a raw aarch64 payload that begins `f3 03 00 aa`, not an ELF, and
binman rejects it:

```
Failed to read ELF file: Magic number does not match
```

rkbin ships no OP-TEE ELF for the RK3576, only BL31 ELFs.

Built without a BL32, mainline u-boot works. binman reports the missing blob and still
produces a working `idbloader.img` and `u-boot.itb`:

```
Image 'simple-bin' is missing optional external blobs but is still functional: tee-os
```

Real OP-TEE requires building `optee_os` from source and passing its `tee.elf`. Running
without OP-TEE has a known cost. Under rkbin BL31 v1.24 without OP-TEE, deep idle can miss
a wakeup (see [Deep idle under BL31 v1.24](#deep-idle-under-bl31-v124)).

## Blob variables at configuration time

Pass the blob make-variables (`BL31=`, `ROCKCHIP_TPL=`, `TEE=`) identically to the
defconfig step and the compile step. Both steps then see one Kconfig environment.

If `TEE=` reaches only the compile step, the u-boot build stops at interactive Kconfig
prompts for `OPTEE_LIB`, `OPTEE_IMAGE`, `SMBIOS_PARSER` and other symbols. The build
appears to hang.

The cause is in u-boot's `lib/optee/Kconfig` [source]. It defines
`config HAS_TEE_IN_BUILD_ENV: def_bool $(success, test -n "$(TEE)")`, which selects
`OPTEE_LIB` and exposes `OPTEE_TZDRAM_SIZE`. `$(TEE)` is a Kconfig environment reference,
and GNU make places command-line variable assignments into each recipe's environment. If
`make <defconfig>` runs without `TEE=` and the compile runs with it, the generated
`.config` lacks those symbols. The compile's `syncconfig` then drops into an interactive
`oldaskconfig`.

The kernel has the same class of defect, where `CROSS_COMPILE` shapes the toolchain-probed
symbols. The configuration and compile steps must agree on the environment.

## The SARADC download key

Mainline u-boot needs two fixes in `rockchip_dnl_key_pressed()` before the SARADC recovery
key reaches maskrom on the RK3576. With both applied, holding the key at power-on drops the
reference unit straight into maskrom [measured]. The entry is then independent of the
factory firmware that mainline u-boot replaces. A vendor u-boot's handling of the key
varies by firmware build.

On stock mainline the key does nothing at power-on [source]. That holds with `CONFIG_ADC`,
`CONFIG_SARADC_ROCKCHIP` and `CONFIG_CMD_ADC` enabled. It also holds with the `saradc` node
set `okay` and a working voltage reference wired to it.

### Finding the device

`setup_boot_mode()` calls the weak `rockchip_dnl_key_pressed()` in
`arch/arm/mach-rockchip/boot_mode.c` [source]. That function walks `UCLASS_ADC` and matches
`strncmp(dev->name, "saradc", 6)`. `dev->name` is the device-tree node name, which
`lists_bind_fdt()` takes from `ofnode_get_name()`. `rk3576.dtsi` declares the block as
`saradc: adc@2ae00000`, where `saradc` is a label. A label exists only in `__symbols__` and
never reaches `dev->name`.

The comparison cannot succeed. The function returns `-ENODEV`, warns "no saradc device
found" and reports the key as unpressed. Every SoC that adopted the generic node-name
convention is affected: rk3568, rk3588 and rk3576. The RK3288 and RK3399 trees still
spell the node `saradc@...`, so the match succeeds there.

The fix matches the driver rather than the node. Every Rockchip SARADC binds
`U_BOOT_DRIVER(rockchip_saradc)`, whatever its node is called.
`if (!strcmp(dev->driver->name, "rockchip_saradc"))` therefore finds it under both the old
and the new node spelling.

### The press window

With the device found, the key still does not register on the RK3576 [measured].
`rockchip_dnl_key_pressed()` counts a press only while the raw sample is within `0..30`. That
window is sized for the 10-bit SARADCs of older SoCs. The RK3576 binds `rk3588_saradc_data`,
which is 12-bit, so the same grounded-key voltage lands about 4x higher in raw counts.

On channel 1 the reference unit reads raw 4087 (~1.8 V) idle and raw 41 (~18 mV) pressed
[measured]. A grounded key does not read raw 0, because ~18 mV of residual remains. A raw
41 sits just outside `0..30`, so the key reads unpressed and the download path never fires.
The fix widens `KEY_DOWN_MAX_VAL` to 100 (~44 mV). That is comfortably above 41 and far
below the 4087 idle reading.

### The download flag

`set_back_to_bootrom_dnl_flag()` writes `BOOT_BROM_DOWNLOAD` (`0xEF08A53C`) to
`CONFIG_ROCKCHIP_BOOT_MODE_REG`, which is `0x26024040` (PMU0_GRF_OS_REG16). That scratch
register survives the reset. The BootROM reads a different register, and BL31 forwards the
value to it on the way down (see [Reboot modes](#reboot-modes)).

The public TRM documents `0x26024040` only as a generic "operation system register", not
as the download flag. The hardware confirms the flag [measured]. At the u-boot prompt, this
sequence enters maskrom:

```
mw.l 0x26024040 0xEF08A53C; reset
```

The key path lands in BootROM download (maskrom), not in a loader.

## Reboot modes

A reboot request on the RK3576 passes through two registers [source]:

| Register | Address | Written by | Read by |
|---|---|---|---|
| PMU0_GRF OS_REG16 | `0x26024040` (PMU0_GRF + 0x40) | The OS or the bootloader | The bootloader after an ordinary reset, and the BL31 system-reset handler |
| PMU1_GRF OS_REG0 | `0x26026200` | BL31, on the way into reset | The BootROM |

NPOR clears OS_REG0, so nothing can write it ahead of a reset. The BL31 system-reset
handler forwards a request from OS_REG16 to OS_REG0. It also masks the reset outputs, so
the value survives into the BootROM. This is the mechanism as Alexey Charkov's RK3576
reboot-mode series describes it (v1, 2026-09-28, unmerged).

The BootROM acts on two kinds of value in OS_REG0:

- `0xEF08A53C` selects USB download (maskrom).
- `0xEF085A3X` selects boot-order mode X, overriding OTP and the SARADC strap.

The bootloader reads the standard Rockchip magics from OS_REG16:

| Magic | Mode |
|---|---|
| `0x5242c300` | Normal |
| `0x5242c301` | Loader (rockusb) |
| `0x5242c303` | Recovery |
| `0x5242c309` | Fastboot |
| `0x5242c30b` | Charge |

The reboot-mode node in a board's device tree maps each `reboot` argument to one of these
magics.

### What rkbin BL31 v1.24 forwards

Under rkbin `rk3576_bl31_v1.24.elf`, maskrom is the only request that reaches the BootROM
[BLOB]. The handler that prints `system reset......` (VA `0x4006e444`) loads `0x26024040`
and compares it with `0xEF08A53C`. On a match it stores the value to `0x26026200`, writes
the reset-mask registers, then resets. Any other value goes straight to the CRU global
reset (`0xfdb9` written to `0x27200c08`).

The boot-order modes need upstream TF-A from commit `2a97ba20e901` onward. Under rkbin
BL31 they boot normally. The `0x5242C3xx` bootloader magics are not affected, because the
bootloader reads them from OS_REG16 after an ordinary reset.

### Maskrom reboot and the NPU rail

A maskrom reboot from Linux also needs the NPU rail on [expected]. A reset that carries a
request leaves the PMIC as the running system left it. The reboot-mode series reports that
the DDR firmware which runs next sets bus QoS in CCI_GRF, DDR_GRF and NPU_GRF [reported].
If a supply behind any of them is down, that firmware takes a synchronous abort.

On the reference unit the DDR and logic rails are `regulator-always-on`. Its NPU rail
follows the NPU power domain and is off whenever the NPU is idle, as
[The NPU rail](npu-reference.md#the-npu-rail) describes. The series adds `*-supply`
properties on the reboot-mode node for this case. A board whose NPU rail follows the power
domain needs that rail named there, as `npu-supply`, or `reboot maskrom` is expected to
fault.

The prediction is untested on hardware. Writing the flag with `mw.l` at the u-boot prompt
is no counter-example, because no kernel has yet switched the rail off.

### Software maskrom entry

A software maskrom entry fills a gap the other paths leave. The download key reaches
maskrom but needs a hand on the board. A u-boot that enters rockusb on the loader magic
gives `reboot loader` a path to rockusb from a running Linux [measured]. That path works
only through a working u-boot. A maskrom reboot lets a host load a candidate u-boot into
RAM and test-boot it without writing the eMMC.

## Deep idle under BL31 v1.24

Under rkbin BL31 v1.24 without OP-TEE, a CPU suspended into the `CPU_SLEEP` idle state can
miss its wakeup. Keep deep idle off: build the kernel with `CONFIG_ARM_PSCI_CHECKER=n`, and
boot with `cpuidle.off=1`.

### The failure

The device-tree `CPU_SLEEP` state is identical in mainline rk3576, in mainline rk3588 and
in the vendor's rk3576 tree. It is PSCI `CPU_SUSPEND` parameter `0x0010000`, core-off, with
`local-timer-stop`. The RK3588 runs it without fault. Under BL31 v1.51, a Turing RK1 entered
it up to 43 million times per core over 17.8 days [measured, 2026-10-23, 7.2.9] (see
[the RK3588's idle states](../rk3588/boot-firmware.md#idle-states)). The device tree and
the kernel are therefore not the variable. The variable in the failing runs is the secure firmware, rkbin BL31 v1.24
without OP-TEE [inferred].

Under that firmware, a CPU that enters `CPU_SLEEP` occasionally never gets its wakeup
delivered [measured]. Idle CPUs wedge inside `cpuidle_enter_state`, and their siblings spin
in `tick_check_broadcast_expired` or `cpu_idle_poll`. The kernel logs
`rcu_preempt kthread timer wakeup didn't happen`. Under normal governor idle the stalls
recover. Where `ARM_PSCI_CHECKER` hammers the state at boot, the result is a hard hang.

The vendor firmware chain, BL31 v1.09 with OP-TEE, showed none of this in its test window.
rkbin's BL31 v1.25 keeps v1.24's program headers, and its disassembly shows no change to
the idle, suspend or reboot-mode paths [inferred][BLOB]. It is not measured on the
reference unit, and offers no known way out. The boot log
carries the fingerprint of a chain without OP-TEE:

```
psci: MIGRATE_INFO_TYPE not supported
```

Linux prints it under any BL31 that runs without OP-TEE, whatever its version, as
[PSCI](../rk3588/boot-firmware.md#psci) describes. On the reference unit it marks the
v1.24-without-OP-TEE condition, the one chain there that runs without OP-TEE.

### Mitigations

The two mitigations cover different halves of the failure:

- `CONFIG_ARM_PSCI_CHECKER=n` removes the boot-time hang.
- `cpuidle.off=1` on the kernel command line covers normal idle.

A device-tree-scoped alternative to the global switch is to drop `cpu-idle-states` from the
board device tree, which leaves WFI-only idle. It is untested. A long run under the vendor
BL31 v1.09 chain is the test that confirms the firmware attribution.

### Retest without the flag

A boot without `cpuidle.off=1` ran clean for 24 minutes [measured, 2026-08-31, 7.2.0].
`psci_idle` registers under the `menu` governor with both states, `WFI` (1 us) and
`cpu-sleep` (370 us). Deep idle is genuinely used: 1,051,724 `cpu-sleep` entries across
the eight cores in 24 minutes. cpu0 holds ~900 s of residency over 914 s of wall clock.

Missed timer wakeups, RCU stalls, soft lockups, oopses and hangs all stayed at zero. The
run covered a 15-minute idle soak, past both points at which the failing capture stalled.
It also covered 7 minutes of forced entry and exit churn on all eight cores.

The flag stays, because the blamed condition is unchanged. The boot still prints
`psci: MIGRATE_INFO_TYPE not supported`, and no newer RK3576 BL31 exists. The fault is
intermittent: the failing capture showed 2 recovered stalls in ~500 s. Against that, a
clean 24 minutes is an unexplained negative, not a fixed defect.

The suspend half is untested. The reference unit has no RTC and wakes only from its
remote's POWER key. The RK3576's `mem_sleep` selects `deep` by default, and `deep` never
resumes [measured]. Dropping the flag needs a multi-hour soak plus a suspend cycle.

## Sources

- [BLOB]: a disassembly of rkbin `rk3576_bl31_v1.24.elf`, and its comparison with
  `rk3576_bl31_v1.25.elf`.
- Source reading: the mainline Linux 7.1 and u-boot v2026.04 trees, with their driver and
  binman code, and upstream TF-A.
- Alexey Charkov's RK3576 reboot-mode series (v1, 2026-09-28, unmerged): the two-register
  mechanism, and the report of the DDR firmware's QoS writes.
- The Armbian forum's RK3576 enablement thread (topic 40483): the report that a vendor
  u-boot built without BL32 hangs.
- The reference unit: an H96 MAX M9 running mainline u-boot and Linux.
