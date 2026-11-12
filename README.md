# device-ref

Hardware reference for a set of SoCs, board families, boards and discrete chips. The SoCs are
Rockchip's RK3288, RK3576 and RK3588 and the Airoha AN7581. The discrete chips are the
MediaTek MT7996 Wi-Fi chip and the Silicon Labs EFR32MG21 radio.

## Layout

The tree is ordered by SoC > Family > Board, with a folder of its own for each discrete chip:

- SoC folders hold the documents that hold for every board on that SoC.
- Family folders appear only where several boards share documents, e.g., the Veyron Chromebooks.
- Board folders hold what is true of that board alone.
- Chip folders hold what is true of a discrete chip, such as a Wi-Fi chip, on any board that
  carries it.

## Documents

### RK3288

| Document | Subject |
|---|---|
| [`rk3288/display-reference.md`](rk3288/display-reference.md) | The two VOPs, the HDMI ceiling and the eDP PHY |
| [`rk3288/video-reference.md`](rk3288/video-reference.md) | The Hantro and rkvdec decoders, and the Mali-T764 GPU |
| [`rk3288/veyron/family-boards.md`](rk3288/veyron/family-boards.md) | What the Veyron Chromebooks share, and how each board differs |
| [`rk3288/veyron/boot-chain.md`](rk3288/veyron/boot-chain.md) | Coreboot, depthcharge and the signed kernel partition |
| [`rk3288/veyron/corrections-and-gotchas.md`](rk3288/veyron/corrections-and-gotchas.md) | Widely repeated claims that are wrong, and integration traps |
| [`rk3288/veyron/white-screen.md`](rk3288/veyron/white-screen.md) | The white screen after Ctrl+U: its causes and how to tell them apart |
| [`rk3288/veyron/wifi-firmware.md`](rk3288/veyron/wifi-firmware.md) | The BCM4354 Wi-Fi and Bluetooth firmware |
| [`rk3288/veyron/asus-c201/hardware-reference.md`](rk3288/veyron/asus-c201/hardware-reference.md) | The ASUS Chromebook C201 |

### RK3576

| Document | Subject |
|---|---|
| [`rk3576/hardware-design-guide.md`](rk3576/hardware-design-guide.md) | What Rockchip's hardware design guide says that changes a software decision |
| [`rk3576/boot-firmware.md`](rk3576/boot-firmware.md) | BL31, OP-TEE, the partitions u-boot scans, the download key and the reboot modes |
| [`rk3576/rockusb-reference.md`](rk3576/rockusb-reference.md) | The rockusb gadget, mode detection and the 32 MiB read wall |
| [`rk3576/display-reference.md`](rk3576/display-reference.md) | The HDMI display path at register level |
| [`rk3576/video-reference.md`](rk3576/video-reference.md) | Decode, RGA, the GPU and the 10-bit ceiling |
| [`rk3576/h264-power-up-erratum.md`](rk3576/h264-power-up-erratum.md) | The VDPU383 H.264 power-up defect |
| [`rk3576/npu-reference.md`](rk3576/npu-reference.md) | The NPU under the mainline `rocket` driver |
| [`rk3576/corrections-and-gotchas.md`](rk3576/corrections-and-gotchas.md) | Device-tree and driver traps that hold for every RK3576 board |
| [`rk3576/h96-max-m9/hardware-reference.md`](rk3576/h96-max-m9/hardware-reference.md) | The H96 MAX M9 TV box |
| [`rk3576/h96-max-m9/factory-device-tree.md`](rk3576/h96-max-m9/factory-device-tree.md) | The H96 factory device tree, against the mainline EVB1 tree |
| [`rk3576/h96-max-m9/corrections-and-gotchas.md`](rk3576/h96-max-m9/corrections-and-gotchas.md) | Misreported H96 claims, and board-level traps |
| [`rk3576/h96-max-m9/wifi-bluetooth.md`](rk3576/h96-max-m9/wifi-bluetooth.md) | The AIC8800D80 Wi-Fi and Bluetooth under mainline |
| [`rk3576/h96-max-m9/mainline-enablement-status.md`](rk3576/h96-max-m9/mainline-enablement-status.md) | What mainline Linux drives on the H96, per subsystem |

### RK3588

| Document | Subject |
|---|---|
| [`rk3588/boot-firmware.md`](rk3588/boot-firmware.md) | The boot chain, USB download mode, the partitions u-boot scans, BL31's PSCI, SCMI and SiP services, and u-boot's handoff to a bare-metal program |
| [`rk3588/cpu-reference.md`](rk3588/cpu-reference.md) | The cores, their ID registers and numbering, the caches and their latency, coherency, errata, operating points and the frequency the cores deliver |
| [`rk3588/system-reference.md`](rk3588/system-reference.md) | The address map, the GIC-600, the timers, the UARTs, the TRNG, the IOMMUs, the DMA controllers, DRAM bandwidth and the thermal sensors |
| [`rk3588/system-sleep.md`](rk3588/system-sleep.md) | The PMU's wake sources, the PMU timer, and BL31's `SYSTEM_SUSPEND` |
| [`rk3588/video-reference.md`](rk3588/video-reference.md) | Decode, encode, RGA, the GPU, the HDMI receiver and what a transcode costs |
| [`rk3588/npu-reference.md`](rk3588/npu-reference.md) | The NPU: its cores, power, clock and register-level behavior under the mainline `rocket` driver |
| [`rk3588/corrections-and-gotchas.md`](rk3588/corrections-and-gotchas.md) | Misstated RK3588 claims, and traps that hold for every board on the SoC |
| [`rk3588/turing-rk1/hardware-reference.md`](rk3588/turing-rk1/hardware-reference.md) | The Turing RK1: power, memory, storage, Ethernet, USB, console, RTC, boot and recovery |

### AN7581

| Document | Subject |
|---|---|
| [`an7581/system-reference.md`](an7581/system-reference.md) | Silicon revisions, the CPU and its operating points, boot security, the address map, UARTs, pin mux, PCIe, thermal and crypto |
| [`an7581/network-reference.md`](an7581/network-reference.md) | The packet path: GDMs, QDMA, the switch, the PCS blocks, the PPE, the NPU, and the network errata |
| [`an7581/gemtek-w1700k/hardware-reference.md`](an7581/gemtek-w1700k/hardware-reference.md) | The Gemtek W1700K: revisions, parts, ports, radios, console, power and GPIOs |
| [`an7581/gemtek-w1700k/boot-and-flash.md`](an7581/gemtek-w1700k/boot-and-flash.md) | The vendor U-Boot, the vendor bad-block area, the stock and OpenWrt flash layouts, the chainloader and the installer |
| [`an7581/gemtek-w1700k/openwrt-status.md`](an7581/gemtek-w1700k/openwrt-status.md) | What OpenWrt drives on the W1700K, per subsystem |
| [`an7581/gemtek-w1700k/corrections-and-gotchas.md`](an7581/gemtek-w1700k/corrections-and-gotchas.md) | Misreported W1700K claims, and board traps |

### MT7996

| Document | Subject |
|---|---|
| [`mt7996/wifi-reference.md`](mt7996/wifi-reference.md) | The MT7996 and its RF dies: identity, host interface, processors, firmware, calibration EEPROM, and probing hazards |

### EFR32MG21

| Document | Subject |
|---|---|
| [`efr32mg21/radio-reference.md`](efr32mg21/radio-reference.md) | The EFR32MG21 Bluetooth LE and 802.15.4 radio as a host's co-processor: identity, memory, security, the Gecko bootloader, host protocols and coexistence |

## Evidence

Every claim carries its grade: `[measured]`, `[source]`, `[consensus]`, `[reported]`,
`[inferred]` or `[expected]`. `PROFILE.md` defines each one. A named source is cited by
an uppercase key, which the document defines in its closing **Sources** section.
