# device-ref

Hardware reference for a set of Rockchip SoCs, board families and boards.

## Layout

The tree is ordered by SoC > Family > Board:

- SoC folders hold the documents that hold for every board on that SoC.
- Family folders appear only where several boards share documents, e.g., the Veyron Chromebooks.
- Board folders holds what is true of that board alone.

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
| [`rk3576/boot-firmware.md`](rk3576/boot-firmware.md) | BL31, OP-TEE, the download key and the reboot modes |
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
| [`rk3588/video-reference.md`](rk3588/video-reference.md) | Decode, encode, RGA, the GPU, the HDMI receiver and what a transcode costs |

## Evidence

Every claim carries its grade: `[measured]`, `[source]`, `[consensus]`, `[reported]`,
`[inferred]` or `[expected]`. `PROFILE.md` defines each one. A named source is cited by
an uppercase key, which the document defines in its closing **Sources** section.
