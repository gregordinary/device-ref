# Turing RK1 hardware reference

This board-level reference covers the Turing RK1, an RK3588 compute module that seats in a
Turing Pi 2 carrier board. It states the module's wiring, addresses, parts, limits and
boot firmware. It is compiled from the mainline device tree, the module's boot firmware and
inspection of the reference unit. It is not a manufacturer publication.

Wiring and addresses come from the mainline tree and the SoC, and hold across firmware
builds. Memory addresses depend on the module's memory size, and the values here are the
reference unit's, a 32 GB module. The SoC documents cover what holds for every RK3588
board:

- The [boot firmware](../boot-firmware.md): BL31, PSCI, SCMI, and u-boot's handoff
- The [processor reference](../cpu-reference.md): cores, caches, coherency and errata
- The [system reference](../system-reference.md): the address map, GIC, timers, UARTs and
  TRNG
- [System sleep](../system-sleep.md): the PMU's wake sources and the firmware's suspend
- The [video reference](../video-reference.md): decode, encode, RGA, the GPU and the HDMI
  receiver, measured on this module
- The [NPU reference](../npu-reference.md): the NPU's cores, power, clock and registers

## Overview

| Property | Value |
|---|---|
| Product | Turing RK1 |
| Class | Compute module, seated in a Turing Pi 2 carrier board |
| SoC | Rockchip RK3588, the full part |
| CPU | 4x Arm Cortex-A76 + 4x Arm Cortex-A55 |
| GPU | Arm Mali-G610 MC4 |
| NPU | Three cores, enabled in the mainline tree |
| System memory | 32 GB on the reference unit |
| Internal storage | eMMC, 8-bit, HS400 Enhanced Strobe |
| Ethernet | Gigabit, RGMII to an RTL8211F PHY |
| Console | uart9, 115200 baud |

Turing's specification gives the memory as LPDDR4 of up to 32 GB, and the eMMC as 32 GB of
eMMC 5.1 [source][TPISPEC].

## Identity

The mainline board tree identifies the module as `model = "Turing Machines RK1"`,
`compatible = "turing,rk1", "rockchip,rk3588"` [source][DTS]. The board tree includes
`rk3588.dtsi`, the full SoC, not `rk3588s.dtsi` [source][DTS]. Its `.dts` describes the
common case: the carrier acts as a backplane with no peripherals of its own that need
describing [source][DTS].

Mainline u-boot supports the module as `turing-rk1-rk3588_defconfig`, with the default FDT
`rockchip/rk3588-turing-rk1.dtb` [source][UBOOT]. u-boot 2026.10 prints
`Model: Turing Machines RK1` and `SoC: RK3588` [measured, 2026-10-08, u-boot
2026.10][CONSOLE].

## Power

An RK806 PMIC on spi2 supplies most rails [source][DTS]. It sits on chip select 0 at
1 MHz, raises its interrupt on gpio0 PA7, active low, and is the system power controller.
Every input it has is fed from the 5 V system rail, directly or through another
regulator.

Discrete regulators supply the Cortex-A76 cores and the NPU [source][DTS]:

| Rail | Part | Bus and address | Range | Loads |
|---|---|---|---|---|
| `vdd_cpu_big0_s0` | RK8602 | i2c0, `0x42` | 550-1050 mV | Cortex-A76 cores `cpu_b0` and `cpu_b1` |
| `vdd_cpu_big1_s0` | RK8603 | i2c0, `0x43` | 550-1050 mV | Cortex-A76 cores `cpu_b2` and `cpu_b3` |
| `vdd_npu_s0` | RK8602 | i2c1, `0x42` | 550-950 mV | The NPU's three cores and the NPU power domain |
| `vdd_cpu_lit_s0` | RK806 DCDC2 | spi2 | 550-950 mV | The four Cortex-A55 cores |
| `vdd_gpu_s0` | RK806 DCDC1 | spi2 | 550-950 mV | The GPU and the GPU power domain |

The RK806 supplies the rest of the module's rails [source][DTS]:

| Rail | RK806 output | Voltage |
|---|---|---|
| `vdd_log_s0` | DCDC3 | 675-750 mV |
| `vdd_vdenc_s0` | DCDC4 | 550-950 mV |
| `vdd_ddr_s0` | DCDC5 | 675-900 mV |
| `vdd2_ddr_s3` | DCDC6 | Left to the PMIC |
| `vdd_2v0_pldo_s3` | DCDC7 | 2.0 V |
| `vcc_3v3_s3` | DCDC8 | 3.3 V |
| `vddq_ddr_s0` | DCDC9 | Left to the PMIC |
| `vcc_1v8_s3` | DCDC10 | 1.8 V |
| `avcc_1v8_s0` | PLDO1 | 1.8 V |
| `vcc_1v8_s0` | PLDO2 | 1.8 V |
| `avdd_1v2_s0` | PLDO3 | 1.2 V |
| `vcc_3v3_s0` | PLDO4 | 3.3 V |
| `vccio_sd_s0` | PLDO5 | 1.8-3.3 V |
| `pldo6_s3` | PLDO6 | 1.8 V |
| `vdd_0v75_s3` | NLDO1 | 0.75 V |
| `vdd_ddr_pll_s0` | NLDO2 | 0.85 V |
| `avdd_0v75_s0` | NLDO3 | 0.75 V |
| `vdd_0v85_s0` | NLDO4 | 0.85 V |
| `vdd_0v75_s0` | NLDO5 | 0.75 V |

The tree also labels DCDC1, DCDC2 and DCDC4 `vdd_gpu_mem_s0`, `vdd_cpu_lit_mem_s0` and
`vdd_vdenc_mem_s0` [source][DTS]. On this module each of those blocks' memory supply
shares its logic rail [inferred].

Linux's `fan53555` driver detects all three discrete regulators as `Option[10] Rev[1]`,
from their ID registers [measured, 2026-10-05, 7.2.9][BOOTLOG]. Idle, the RK806 rails Linux
sets read 750 mV on `vdd_log_s0` and `vdd_vdenc_s0`, and 850 mV on `vdd_ddr_s0`
[measured, 2026-10-23, 7.2.9][REGULATORS]. The CPU rails follow the operating points, as
[delivered frequency](../cpu-reference.md#delivered-frequency) shows.

The tree names each rail for the state it serves. Every `_s0` rail is off in suspend, and
every `_s3` rail stays on [source][DTS]. `vdd_npu_s0` and `vdd_gpu_s0` are boot-on but not
always-on, and each is its power domain's supply [source][DTS]. Each therefore follows
its domain [inferred].

## Cooling

The tree drives a fan on PWM0, through the `pwm0m2` pins, at a 50 us period [source][DTS].
Its cooling levels are 0, 25, 95, 145, 195 and 255. The fan's tachometer reaches gpio0
PA4 as a falling-edge interrupt, with the pin pulled up.

The package thermal zone polls every second, and steps the fan one level at each of five
trips [source][DTS]. The trips sit at 45, 50, 60, 70 and 80 °C, each with 5 °C of
hysteresis.

Each trip's cooling map names one fan level as both its lower and its upper bound
(`<&fan N N>`) [source][DTS]. Linux's `step_wise` governor steps such maps down correctly
from 7.2.9. In earlier kernels a trip that no longer applies can keep its vote, and hold
the fan above the level the temperature asks for [source][LINUX].

A 30-minute all-core memory stress run on the reference unit peaked at 64.7 °C
[measured, 2026-10-08, 7.2.9]. The fan held level 145, the 60 °C trip's.

## Memory

u-boot 2026.10 reports `DRAM: 32 GiB (total 31.7 GiB)` [measured, 2026-10-08, u-boot
2026.10][CONSOLE]. Linux 7.2.9 finds the DRAM in four ranges, 31.74 GiB in all [measured,
2026-10-08, 7.2.9][IOMEM]:

| Start | End | Size |
|---|---|---|
| `0x0_0020_0000` | `0x0_efff_ffff` | 3,838 MiB |
| `0x1_0000_0000` | `0x3_fbff_ffff` | 12,224 MiB |
| `0x3_fc50_0000` | `0x3_ffef_ffff` | 58 MiB |
| `0x4_0000_0000` | `0x7_ffff_ffff` | 16,384 MiB |

DRAM starts at 2 MiB. u-boot leaves the 2 MiB below it to BL31 (see
[memory the firmware keeps](../boot-firmware.md#memory-the-firmware-keeps)). Two holes split
the DRAM above 4 GiB, 5 MiB at `0x3_fc00_0000` and 1 MiB at `0x3_fff0_0000`. Mainline
u-boot's MMU map leaves both unmapped [source][UBOOT].

The SoC's peripheral window takes `0xf000_0000` to 4 GiB, and no range stands in for the
DRAM behind it (see [the address map](../system-reference.md#address-map)). The largest
contiguous range is the 16 GiB at `0x4_0000_0000`. A module with less memory carries a
different map [expected].

u-boot reports the same four banks, and the FDT it hands Linux names them in `/memory`
[measured, 2026-10-08, u-boot 2026.10][CONSOLE][FDT]. The FDT declares two reservations [measured,
2026-10-08, 7.2.9][FDT]:

| Reservation | Start | Size | Holder |
|---|---|---|---|
| `/reserved-memory/shmem@10f000`, `no-map` | `0x10_f000` | 256 bytes | The firmware's SCMI channel |
| `/memreserve/` | `0xeb14_b000` | 22.7 MiB | The initial RAM disk, as `/chosen` gives it |

No reservation falls inside the 16 GiB at `0x4_0000_0000`. A kernel given that range whole
overwrites nothing the firmware declares [inferred].

### Memory under u-boot

u-boot 2026.10 relocates itself to `0xefef_9000`, below 4 GiB, and keeps its control FDT at
`0xedbd_19c0` [measured, 2026-10-08, u-boot 2026.10][CONSOLE][UBOOTENV]. Its memory map marks every
bank above 4 GiB as reserved and not to be overwritten [measured, 2026-10-08, u-boot
2026.10][CONSOLE].

> **u-boot's `load` places nothing above 4 GiB.** u-boot caps its usable RAM at
> `0xf000_0000`. The RK1's defconfig leaves `CONFIG_LMB_LIMIT_DMA_BELOW_RAM_TOP` on, so
> u-boot reserves every bank above that cap as not to be overwritten [source][UBOOT].
> A program that u-boot starts reads and writes that RAM through u-boot's page tables. A
> kernel bound for the range at `0x4_0000_0000` is therefore copied there by that program.

The environment places Linux's images low in DRAM [source][UBOOT], and the reference unit
prints the same values [measured, 2026-10-08, u-boot 2026.10][UBOOTENV]:

| Variable | Address |
|---|---|
| `scriptaddr` | `0x00c0_0000` |
| `loadaddr` | `0x00c0_0800` |
| `pxefile_addr_r` | `0x00e0_0000` |
| `kernel_addr_r` | `0x0200_0000` |
| `kernel_comp_addr_r` | `0x0a00_0000`, with 128 MiB for a compressed kernel |
| `fdt_addr_r` | `0x1200_0000` |
| `fdtoverlay_addr_r` | `0x1210_0000` |
| `ramdisk_addr_r` | `0x1218_0000` |

## Storage

The eMMC sits on the SoC's SDHCI controller, `mmc@fe2e0000` [source][DTS]. The tree
describes it as 8-bit, HS400 at 1.8 V with Enhanced Strobe, and non-removable, with SD and
SDIO commands off. It is alias `mmc0`. u-boot 2026.10 lists the controller as device 0,
and its SPL reports `Trying to boot from MMC1` [measured, 2026-10-08, u-boot
2026.10][CONSOLE].

The mainline tree describes no SPI flash, and enables no SD controller [source][DTS]. The
boot firmware therefore lives in the eMMC user area, beside the operating system (see
[the boot medium's layout](../boot-firmware.md#the-boot-mediums-layout)).

The tree enables two PCIe controllers [source][DTS]:

| Controller | Base | Lanes | PHY | Reset | PCI domain |
|---|---|---|---|---|---|
| `pcie3x4` | `0xfe15_0000` | PCIe 3.0 x4 | `pcie30phy` | gpio4 PB6 | 0 |
| `pcie2x1l1` | `0xfe18_0000` | PCIe 2.0 x1 | `combphy2_psu` | gpio4 PA2 | 1 |

`pcie3x4` takes its 3.3 V from `vcc3v3_pcie30`, a switched rail enabled by gpio2 PC5, with
a 5 ms startup delay [source][DTS]. `pcie2x1l1` names no 3.3 V supply, and the carrier
powers whatever sits behind it [inferred].

Both controllers train at their full rate with a device behind them. Two modules in their
carrier slots show the same links [measured, 2026-10-23, 7.2.9][LSPCI]:

| Controller | Device | Link | ASPM |
|---|---|---|---|
| `pcie3x4` | An NVMe disk: a Silicon Motion SM2263 on the reference unit, a Phison E13 on the second module | 8 GT/s, x4 | L1. The root port supports no L1 substates |
| `pcie2x1l1` | A Coral Edge TPU, on the carrier's slot | 5 GT/s, x1 | L0s |

Each root port offers a 256-byte maximum payload, and Linux runs both links at 128 bytes
[measured][LSPCI]. The second module's NVMe disk reads 1.3 GB/s sequentially with 4 MiB requests at queue
depth 1. That tests the disk more than the link [measured, 2026-10-23, 7.2.9][DISKS].

Both modules measured carry the same Samsung eMMC part, which Linux names `BJTD4R`
[measured, 2026-10-23, 7.2.9][MMC]. Turing's specification names no part, so another module
can carry a different one [expected]:

| Property | Value |
|---|---|
| Manufacturer ID | `0x15`, Samsung |
| Capacity | 29.1 GiB in the user area, two 4 MiB boot partitions, a 4 MiB RPMB partition |
| Standard | eMMC 5.1, `EXT_CSD` revision 8 |
| Cache | 8 MiB, enabled |
| Mode | HS400 with Enhanced Strobe, command queueing on |
| Manufactured | October 2022 |

It reads 276 MB/s sequentially with 4 MiB requests, and 19.2 MB/s with 4 KiB requests at
queue depth 1 [measured, 2026-10-23, 7.2.9][DISKS].

## Networking

One Gigabit Ethernet port is driven by GMAC1, `ethernet@fe1c0000` [source][DTS]. u-boot
2026.10 brings it up as `eth0` [measured, 2026-10-08, u-boot 2026.10][CONSOLE].

| Property | Value |
|---|---|
| Controller | GMAC1 at `0xfe1c_0000` |
| Interface | RGMII, `rgmii-rxid`, clock output from the SoC |
| Delays | `tx_delay` `0x43`, `rx_delay` 0 |
| PHY | Realtek RTL8211F at MDIO address 1, `ethernet-phy-id001c.c916` |
| PHY reset | gpio3 PB7, active low, 15 ms asserted and 50 ms settling |

Linux reads GMAC1's version as `0x51` with user ID `0x30`, a DWMAC 5.1 core [measured,
2026-10-05, 7.2.9][BOOTLOG]. The core offers checksum offload in both directions, TSO and
Wake-on-LAN by magic packet, and addresses 32 bits of DMA (see
[masters with no IOMMU](../system-reference.md#masters-with-no-iommu)). The link trains at
1000 Mb/s, full duplex, with EEE negotiated but inactive [measured, 2026-10-23, 7.2.9][NET].

`rgmii-rxid` asks the PHY for the receive delay alone, and the MAC's `tx_delay` supplies the
transmit delay [source][DTS]. Linux programs the RTL8211F to match, on the PHY's page
`0xd08` [source][LINUX]. It sets the receive delay, bit 3 of register `0x15`, and clears
the transmit delay, bit 8 of register `0x11`. Firmware that leaves the PHY alone runs
on the PHY's strapped delays, which are not measured on the reference unit.

The SoC generates the RGMII clocks (`clock_in_out = "output"`), and the tree leaves the
`gmac1_mclkinout` pin, gpio3 PB6, unmuxed [source][DTS].

## USB

The tree enables three USB buses [source][DTS]:

| Bus | Controller | Base | Use |
|---|---|---|---|
| USB 0 | `usb_host0_xhci` | `0xfc00_0000` | USB 2.0 only, OTG-capable, `maximum-speed = "high-speed"` |
| USB 1 | `usb_host1_xhci` | `0xfc40_0000` | USB 3.0, host only, on USBDP PHY 1 |
| USB 2 | `usb_host0_ehci`, `usb_host0_ohci` | `0xfc80_0000`, `0xfc84_0000` | USB 2.0, host only |

On the RK1, USBDP PHY 0 drives the DisplayPort pins, and no USB lane reaches it
[source][DTS]. The tree still enables the PHY, with `rockchip,dp-lane-mux = <0 1 2 3>`.

> **Keep USBDP PHY 0 enabled for USB 0 to work.** The USB 0 controller has to learn from its
> GRF that it has no USB 3 port. Mainline Linux sets those GRF bits in the USBDP PHY driver
> [source][DTS].

## Display and media

The mainline tree enables no display path [source][DTS]. The SoC's HDMI receiver is out of
reach on this module, as [the HDMI receiver](../video-reference.md#hdmi-receiver)
describes.

The tree enables the Mali-G610 on `vdd_gpu_s0`, and the NPU's three cores on `vdd_npu_s0`
[source][DTS]. The [RK3588 video reference](../video-reference.md) describes the media
blocks under mainline, measured on this module. The [RK3588 NPU reference](../npu-reference.md)
describes the NPU.

## Console

The console is uart9 at 115200 baud, 8N1: `stdout-path = "serial9:115200n8"`
[source][DTS]. uart9 sits at `0xfebc_0000`, on its `uart9m0` pins. The board tree also
enables uart2, on its `uart2m0` pins [source][DTS].

Mainline u-boot's debug UART matches [source][UBOOT]:

| Setting | Value |
|---|---|
| `CONFIG_DEBUG_UART_BASE` | `0xFEBC0000` |
| `CONFIG_DEBUG_UART_CLOCK` | 24000000 |
| `CONFIG_DEBUG_UART_SHIFT` | 2 (registers 4 bytes apart) |

Neither the DDR firmware, which runs before u-boot's SPL, nor BL31 prints a banner on uart9
at 115200 baud [measured, 2026-10-08, u-boot 2026.10][CONSOLE].

### The console through the carrier

The carrier's management controller exposes the module's UART as one of its own serial
devices, a `/dev/ttyS` device for the module's slot. Its daemon, `bmcd`, holds that device
open [measured, 2026-10-08]. Which device serves which slot depends on the controller's
firmware [source][TPIDOCS]:

| Node | Firmware 2.0.5 and older | Firmware 2.1.0 and newer |
|---|---|---|
| 1 | `/dev/ttyS2` | `/dev/ttyS1` |
| 2 | `/dev/ttyS1` | `/dev/ttyS2` |
| 3 | `/dev/ttyS4` | `/dev/ttyS3` |
| 4 | `/dev/ttyS5` | `/dev/ttyS4` |

`tpi uart --node <n> get` reads the same stream through `bmcd` [source][TPIDOCS].

> **A capture taken beside a running `bmcd` drops characters.** It never adds or changes
> one [measured, 2026-10-08][CONSOLE][LOADER]. `bmcd` and the capture share one input queue, and each
> byte `bmcd` reads is lost to the capture [inferred].

Counted from the start of a burst, one byte in every 129 is lost: 128 arrive and the next
does not [measured, 2026-10-08][LOADER]. At times a whole 128-byte block goes missing as
well. The count restarts once the line goes idle for 1 ms, and an idle gap of 100 us does
not restart it [measured, 2026-10-08][PACE]. A program that prints the same text therefore
loses the same characters each run.

With `bmcd` stopped, a capture loses nothing over 40 KB [measured, 2026-10-09, u-boot
2026.10][PROBE]. Stop the daemon for the capture, and start it again after:

```sh
kill -STOP $(pidof bmcd)
picocom -b 115200 --logfile console.log /dev/ttyS<n>
# Ctrl-A Ctrl-X ends picocom.
kill -CONT $(pidof bmcd)
```

The controller's `tpi` commands go through `bmcd`, so they wait until it runs again
[expected]. A power cycle through `tpi` therefore follows the `SIGCONT`.

A capture checks itself where u-boot prints `printenv`. The variables' lines, each with its
line feed, add up to one less than u-boot's `Environment size`. That figure counts the
export buffer's closing NUL [source][UBOOT].

## Real-time clock

The module carries a Haoyu HYM8563 real-time clock on i2c6 (`0xfec8_0000`), at address
`0x51` [source][DTS]. Its interrupt line reaches gpio0 PB0, level low, which the clock pulls
low for its alarm against the pin's pull-up. The tree marks the clock a wake source.

Linux 7.2.9 drives the clock as `rtc-hym8563 6-0051`. Awake, the alarm raises its
interrupt: GPIO0's pin 8, PB0, counts one interrupt for an alarm a minute on [measured,
2026-10-10, 7.2.9][RTCALARM].

The HYM8563 flags its time invalid when its supply falls too low to keep it. After a power
cycle, the reference unit's Linux logged `no valid clock/calendar values available`, and
left the system time unset from the clock [measured, 2026-10-23, 7.2.9][BOOT23]. A second
module logged the same at a boot [measured, 2026-10-05, 7.2.9][BOOTLOG]. The clock therefore
loses its time whenever the module loses power [inferred].

> **Linux 7.2.9 did not resume from suspend on the reference unit, to memory or to idle.**
> Each was tried once. Suspended with `rtcwake -m mem` and the alarm 20 s on, Linux takes
> every core but the boot core offline, then prints nothing more [measured, 2026-10-10,
> 7.2.9][RTCWAKE]. Suspended to idle with `rtcwake -m freeze`, it did not resume either
> [measured, 2026-10-10, 7.2.9][RTCALARM].

A suspend to idle makes no call to the firmware's `SYSTEM_SUSPEND`. Linux's own suspend
path therefore fails first on the reference unit, whatever the firmware does [inferred]. Whether
the firmware woke the system from the suspend to memory is not known. A bare-metal program
does wake from the firmware's suspend on other sources (see
[measured wakes](../system-sleep.md#measured-wakes)).

## Boot and recovery

The module boots from its eMMC. The boot area carries rkbin's DDR firmware v1.24 and BL31
v1.56, and mainline u-boot 2026.10 [measured, 2026-10-08][STRINGS].

The firmware and the operating system share the eMMC user area, so an installer that
repartitions the eMMC erases the firmware. The Proxmox VE 9.2 ISO installer, for one, runs
`sgdisk -Z` and zeroes 256 MiB from 1 MiB, which takes u-boot's FIT at 8 MiB
[source][PVE]. Point such an installer at an NVMe disk instead.

The edk2-rk3588 UEFI firmware carries no RK1 platform, and the project closed the request
for one as not planned [source][EDK2]. [UEFI firmware](../boot-firmware.md#uefi-firmware)
describes that firmware on the SoC. The u-boot string
matches the banner the running u-boot prints, so the image is the one that booted
[inferred]. [The boot chain](../boot-firmware.md#the-boot-chain) describes the blobs.

### The carrier's management controller

The carrier's management controller writes the module's eMMC and nothing else. An NVMe
disk the module reaches over PCIe is invisible to it [measured]. The controller has two
routes to the eMMC:

- `tpi flash -n <node> -l -i <image>` writes an image the controller holds to the eMMC
  [measured].
- `tpi advanced msd --node <node>` reboots the module and exposes its eMMC as a USB
  mass-storage disk on the controller [source][TPI].

The controller also reaches the SoC's maskrom. With the module powered off,
`tpi usb -n <node> flash` puts the module in flashing mode and its USB OTG port in device
mode. Powered on, the module then answers `rkdeveloptool` in maskrom [source][TPIDOCS][TPI].
That route needs nothing on the eMMC [inferred].

The mainline defconfig leaves u-boot's `ums` command off [source][UBOOT]. A u-boot built
with it, `CONFIG_CMD_USB_MASS_STORAGE`, can present an NVMe disk to the controller over the
module's USB OTG port, as `ums 0 nvme 0` [expected]. The disk is then writable with no
operating system on the module.

The controller also switches each module's power. `tpi power reset --node <n>` cuts module
`<n>`'s power for a second, then restores it, and leaves the other modules on. `tpi power
off` and `tpi power on` act on all four modules unless `--node` names one [source][TPI]. A
module that sleeps with no wake source armed comes back this way [inferred].

### Mainline u-boot

Built from `turing-rk1-rk3588_defconfig`, u-boot 2026.10 keeps its environment nowhere,
`CONFIG_ENV_IS_NOWHERE` [source][UBOOT]. A `setenv` therefore lasts until the next reset.
`bootcmd` runs `bootflow scan -lb` after 2 seconds, over
`mmc1 mmc0 nvme scsi usb pxe dhcp spi` [source][UBOOT]. The reference unit's console shows
both [measured, 2026-10-08, u-boot 2026.10][UBOOTENV].

`bootflow scan` looks only at bootable partitions [source][UBOOT]. A bootable GPT partition
carries the legacy BIOS bootable attribute, bit 2, or the EFI system partition's type
GUID. With no such partition on a disk, the scan reads partition 1
alone. It logs each skipped partition at debug level only, so the console shows nothing.

> **Mark the partition that holds `extlinux.conf` bootable, or place it first.** Otherwise
> u-boot passes over it in silence.

An `extlinux.conf` label with no `fdt` or `fdtdir` line boots with u-boot's own control
FDT, from `fdtcontroladdr` [source][UBOOT]. A kernel package that ships no device trees
therefore still boots, on the tree u-boot was built with.

The defconfig builds in these commands, among others [source][UBOOT]:

- Boot: `bootelf`, `booti`, `bootefi` and `go`
- Loading: `loadb`, `loadx`, `loady` and `tftpboot`
- Recovery: `rockusb`, the rockusb gadget on the USB OTG port
- Memory: `md`, `mw` and `dm`

It builds no `dcache`, `icache`, `rng`, `clk`, `scmi`, `setexpr` or `ums` command, and no
watchdog driver [source][UBOOT]. A program started with `go` therefore starts with both caches on,
and cleans the data cache itself (see
[entering a program from mainline u-boot](../boot-firmware.md#entering-a-program-from-mainline-u-boot)).

## Device-tree register map

This map lists the blocks the module's tree enables, plus the system blocks a port reaches
first [source][DTS][DTSBASE]. `pcie3x4`, `usb_host1_xhci` and USBDP PHY 1 are described in
`rk3588-extra.dtsi`. The RK3588S trees do not include that file, so the three exist on the
full part alone [source][DTSBASE].

| Block | Node | Base address |
|---|---|---|
| GIC distributor | `interrupt-controller@fe600000` | `0xfe60_0000` |
| Console UART (uart9) | `serial@febc0000` | `0xfebc_0000` |
| uart2 | `serial@feb50000` | `0xfeb5_0000` |
| eMMC (SDHCI) | `mmc@fe2e0000` | `0xfe2e_0000` |
| GMAC1 Ethernet | `ethernet@fe1c0000` | `0xfe1c_0000` |
| PCIe 3.0 x4 | `pcie@fe150000` | `0xfe15_0000` |
| PCIe 2.0 x1 | `pcie@fe180000` | `0xfe18_0000` |
| USB 0 (xHCI) | `usb@fc000000` | `0xfc00_0000` |
| USB 1 (xHCI) | `usb@fc400000` | `0xfc40_0000` |
| USB 2 (EHCI / OHCI) | `usb@fc800000` / `usb@fc840000` | `0xfc80_0000` / `0xfc84_0000` |
| GPU (Mali-G610) | `gpu@fb000000` | `0xfb00_0000` |
| NPU core 0 | `npu@fdab0000` | `0xfdab_0000` |
| i2c0 (Cortex-A76 regulators) | `i2c@fd880000` | `0xfd88_0000` |
| i2c1 (NPU regulator) | `i2c@fea90000` | `0xfea9_0000` |
| i2c6 (RTC) | `i2c@fec80000` | `0xfec8_0000` |
| spi2 (RK806 PMIC) | `spi@feb20000` | `0xfeb2_0000` |
| PWM0 (fan) | `pwm@fd8b0000` | `0xfd8b_0000` |
| Thermal sensor (TSADC) | `tsadc@fec00000` | `0xfec0_0000` |
| TRNG | `rng@fe378000` | `0xfe37_8000` |
| PMU power domains | `power-management@fd8d8000` | `0xfd8d_8000` |

### GPIO banks

| Bank | Base address |
|---|---|
| gpio0 | `0xfd8a_0000` |
| gpio1 | `0xfec2_0000` |
| gpio2 | `0xfec3_0000` |
| gpio3 | `0xfec4_0000` |
| gpio4 | `0xfec5_0000` |

### Board GPIO assignments

| Signal | GPIO | Polarity, as the tree gives it |
|---|---|---|
| RK806 PMIC interrupt | gpio0 PA7 | Level low |
| RTC alarm | gpio0 PB0 | Level low, pulled up |
| Fan tachometer | gpio0 PA4 | Falling edge, pulled up |
| Ethernet PHY reset | gpio3 PB7 | Active low |
| PCIe 3.0 x4 reset | gpio4 PB6 | Active high |
| PCIe 2.0 x1 reset | gpio4 PA2 | Active high |
| PCIe 3.0 x4 3.3 V enable | gpio2 PC5 | Active high |

## Sources

Citation keys:

- `[DTS]`: Linux `arch/arm64/boot/dts/rockchip/rk3588-turing-rk1.dts` and
  `rk3588-turing-rk1.dtsi` at tag `v7.2.9`. The USBDP PHY 0 note is the tree's own
  comment on `&usbdp_phy0`.
- `[LINUX]`: Linux at tag `v7.2.9`: `drivers/net/phy/realtek/realtek_main.c`
  (`rtl8211f_config_init`, `RTL8211F_RGMII_PAGE`) and `drivers/thermal/gov_step_wise.c`.
- `[TPISPEC]`: Turing Pi's documentation, the page `turing-rk1-specs-and-io-ports`, read
  2026-10-11.
- `[DTSBASE]`: Linux `arch/arm64/boot/dts/rockchip/rk3588-base.dtsi` and
  `rk3588-extra.dtsi` at the same commit, for the base addresses.
- `[UBOOT]`: u-boot at commit `ece349a`. `configs/turing-rk1-rk3588_defconfig` and the
  configuration it resolves to. `include/configs/rk3588_common.h` and
  `include/configs/rockchip-common.h`, for the environment and boot targets.
  `arch/arm/mach-rockchip/sdram.c` (`board_get_usable_ram_top`), `lib/lmb.c`
  (`lmb_add_memory`) and `arch/arm/mach-rockchip/rk3588/rk3588.c` (`rk3588_mem_map`).
  `doc/board/rockchip/rockchip.rst`, and `lib/hashtable.c` (`hexport_r`).
  `boot/bootdev-uclass.c` (`bootdev_find_in_blk`), `disk/part.c` (`part_get_bootable`),
  `disk/part_efi.c` (`get_bootable`), `cmd/bootflow.c`, and `boot/pxe_utils.c` (the
  `fdtcontroladdr` fallback).
- `[PVE]`: the Proxmox VE 9.2 arm64 installer ISO, `proxmox-ve_9.2-1-arm64.iso`,
  inspected: its disk-preparation step, which places the EFI system partition at 1 MiB.
- `[EDK2]`: edk2-porting/edk2-rk3588, its platform list at commit `3189d482`
  (2026-10-09), and its issue 121, the RK1 request.
- `[TPI]`: turing-machines/tpi at commit `07e3bfe`, `src/cli.rs` (`PowerArgs`, and the help
  text of the `advanced msd` and `usb flash` modes) and `src/legacy_handler.rs`
  (`handle_power_nodes`). turing-machines/bmcd at commit `a15e8fc`,
  `bmcd/src/hal/power_controller.rs` (`reset_node`).
- `[TPIDOCS]`: Turing Pi's documentation at docs.turingpi.com, read 2026-10-10. The RK1
  flashing page, `turing-rk1-flashing-os`, enters maskrom through the controller's USB
  flash mode. The UART page, `tpi-uart`, gives the node-to-device table.

Captures from the reference unit, kept with its evidence outside this repository. Those
marked as the second module's come from another 32 GB RK1, whose eMMC carries u-boot 2026.07,
BL31 v1.51 and DDR firmware v1.19:

- `[CONSOLE]`: the console from power-on to u-boot 2026.10's prompt, `version` and
  `bdinfo`, 2026-10-08.
- `[UBOOTENV]`: u-boot 2026.10's banner, `printenv` and `help`, 2026-10-08.
- `[STRINGS]`: printable runs in sectors 64 to 32767 of the eMMC, filtered for version
  strings, 2026-10-08.
- `[IOMEM]`: `/proc/iomem` read as root under Linux 7.2.9, 2026-10-08.
- `[FDT]`: `/sys/firmware/fdt`, the tree u-boot 2026.10 passed to Linux 7.2.9, 2026-10-08.
- `[LOADER]`: a bare-metal program's report, run twice from u-boot 2026.10 with `go`,
  2026-10-08, captured beside a running `bmcd`.
- `[PACE]`: the same program printing line pairs split by idle gaps of 0 to 500 ms, then
  its report paced, run twice, 2026-10-08.
- `[PROBE]`: a bare-metal probe run from u-boot 2026.10, 2026-10-09, captured with `bmcd`
  stopped. Three checks hold over its 40 KB. The variables add up to the size u-boot
  reports, `help` matches an earlier capture, and two runs differ only where their inputs
  do.
- `[RTCWAKE]`: the console of Linux 7.2.9 during `rtcwake -m mem -s 20`, with
  `console_suspend` off and the log level at 8, 2026-10-10.
- `[RTCALARM]`: the shell of Linux 7.2.9 after a power cycle, 2026-10-10: the alarm's
  interrupt count awake, then `rtcwake -m freeze`.
- `[BOOT23]`: the reference unit's kernel log of Linux 7.2.9 after a power cycle,
  2026-10-23.
- `[BOOTLOG]`: the second module's kernel log of a boot of Linux 7.2.9, 2026-10-05, read
  from the journal.
- `[REGULATORS]`: the second module's `/sys/kernel/debug/regulator/regulator_summary`,
  idle, 2026-10-23.
- `[LSPCI]`: `lspci -vvv` on both modules, 2026-10-23, with the NVMe disks in use.
- `[MMC]`: the eMMC of both modules, from `/sys/block/mmcblk0/device` and the boot log,
  2026-10-23. On the second module, also `mmc extcsd read`.
- `[DISKS]`: `dd` with `iflag=direct` from the second module's eMMC and NVMe disk,
  2026-10-23, after dropping the page cache. It read 1 GiB of the eMMC in 4 MiB blocks and
  100 MiB in 4 KiB blocks, then 4 GiB of the NVMe disk.
- `[NET]`: the second module's `ethtool` reports for `end0`, 2026-10-23.
