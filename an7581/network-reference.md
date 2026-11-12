# AN7581 network reference

This document covers the AN7581's packet path and its known defects. The path runs through
the frame engine and its GDM ports, two QDMA engines and an internal switch. It also runs
through the 10G ports' PCS blocks, the PPE flow engine and the NPU. The reader is writing a
device tree for an AN7581 board, or deciding which offloads to trust on one. The
[system reference](system-reference.md) covers the rest of the SoC.

Conditions: the facts rest on one board, the Gemtek W1700K, under OpenWrt's 6.12 and 6.18
kernels, with OpenWrt's out-of-tree `airoha_eth` patches. No reference unit has been probed
yet. Every claim is `[source]`, `[consensus]` or `[reported]`.

## Overview

| Block | Base | Function |
|---|---|---|
| Frame engine | `0x1fb5_0000` | Packet switching between the GDMs, the CPU and the NPU |
| QDMA0, QDMA1 | `0x1fb5_4000`, `0x1fb5_6000` | DMA between the frame engine and DRAM, with hardware QoS |
| Switch | `0x1fb5_8000` | GbE switch with four PHY ports, CPU port 6 on GDM1 |
| PON PCS | `0x1fa0_8000` | The PON SerDes, used as a USXGMII PCS |
| Ethernet PCS | `0x1fa0_9000` | The Ethernet SerDes, USXGMII or HSGMII |
| NPU | `0x1e90_0000` | RISC-V cores: flow offload to Wi-Fi |
| PPE | In the frame engine | Flow table for routed and bridged offload |

Sources: [source][DTSI], [reported][T626].

The frame engine connects ports called GDMs. A GDM is a MAC-side port of the frame engine,
either to a PCS or to the internal switch [source][DTSI]. A board picks which GDMs it uses:

| GDM | Connects to | W1700K port |
|---|---|---|
| GDM1 | The switch's CPU port 6, a fixed 10G link | `lan3` and `lan4`, through the switch |
| GDM2 | The PON PCS | `wan` |
| GDM4 | The Ethernet PCS | `lan2` |

Sources: [source][DTS][DTSI]. The tree describes no GDM3, although the frame engine keeps
counters for a third GDM at `0x1fb5_2600` [reported][ACK72].

## Packet path

A received frame enters at a GDM. The PPE looks it up. A bound flow goes straight to its
egress GDM, or to the NPU for a Wi-Fi station. Every other frame goes to the CPU through a
QDMA [reported][T626][T2599].

The driver uses QDMA0 for the LAN side and QDMA1 for the WAN side. Each QDMA has 32 receive
rings and two transmit-completion contexts, and raises its interrupts on four banks. Linux
names them `airoha_eth.0` to `airoha_eth.7` [source][ACK72]. The driver exposes no receive
hashing to spread one ring's flows across CPUs [source][ACK72]. The tree reserves DRAM for
each QDMA's buffers (see the system reference's [memory map](system-reference.md#memory)).

### Receive offloads

QDMA's large-receive engine serves 8 queues, receive rings 24 to 31. One interface per QDMA
can use it at a time, and it is off by default [source][ACK72], [reported][T3524]. With LRO, one TCP
stream reaches about 8.1 Gb/s, against about 2.7 Gb/s with software GRO
[reported][T3524].

The `wan` port takes LRO only in router mode, with the WAN up [reported][T3556][T3583]:

```sh
ip link set wan down
ethtool -K wan lro on
ip link set wan up
```

### Transmit congestion

`airoha_eth` enables QDMA's `TXQ_CNGST_DROP_EN`. A full transmit queue then drops frames
with no error to the sender [source][SDR]. A 10G export from DRAM sustains about 1.22 GB/s
with QDMA's `MAX_ISSUE_NUM` at 3, and 1.14 GB/s at 2 [reported][SDR].

## The switch

The switch is `airoha,en7581-switch`, driven by Linux's `mt7530` DSA driver. Its ports 1 to 4
have internal clause 22 GbE PHYs at MDIO addresses 9 to 12 [source][DTSI]. Linux names them
`Airoha AN7581 PHY` [consensus][T551][T2701]. The switch's MDIO bus also carries a board's
external PHYs, as on the W1700K [source][DTS].

Each port keeps a MIB block at `0x1fb5_c000 + port * 0x100` [source][ACK72]:

| Offset | Counter |
|---|---|
| `0x08` | TX unicast |
| `0x48` | TX octets |
| `0x68` | RX unicast |
| `0x6c` | RX multicast |
| `0x78` | RX CRC errors |
| `0x8c` | RX pause |
| `0x12c` | TX pause |

LED functions for the switch's PHYs share pins with the HSUART. Selecting the `phy1_led0`
pin group stops the HSUART [reported][T3737][T3749].

### Bridging across the switch and a GDM

A bridge between a switch port and a GDM port spans two packet paths. The switch forwards
between its own ports. Traffic between a switch port and a GDM port crosses the CPU, or the
PPE once a flow binds [reported][B314][B330].

> [!CAUTION]
> **Test a VLAN-aware bridge that joins a switch port, a GDM port and Wi-Fi.** On builds
> before OpenWrt commit `5808e5db4be0`, Wi-Fi clients lost traffic to LAN ports in such
> bridges [reported][B179][B500][B501]. The driver offloads a bridged flow only with the
> switch's software FDB populated [reported][B8].

## PCS and SerDes

Each 10G GDM reaches its PHY through a PCS in USXGMII mode, with in-band status
[source][DTS][DTSI]:

| PCS | Registers | Analog block | GDM |
|---|---|---|---|
| PON | `0x1fa0_8000`, `0x1fa8_0000` to `0x1fa8_b000` | `0x1fa8_a000` | GDM2 |
| Ethernet | `0x1fa0_9000`, `0x1fa7_0000` to `0x1fa7_b000` | `0x1fa7_a000` | GDM4 |
| PCIe | `0x1fa0_4000`, `0x1fa0_5000` | | |
| USB | `0x1fa0_7000` | | |

The PCS driver is not in mainline Linux. OpenWrt carries it as a patch in its 6.18 series
[reported][T3883]. The vendor bootloader selects each SerDes's mode from its environment,
`serdes_pon` and `serdes_ethernet` (see the
[W1700K's boot chain](gemtek-w1700k/boot-and-flash.md#environment)).

The PCS's receiver needs a manual calibration on silicon before E2 [reported][T3701]. The
W1700K2's PON PCS needs it on E2 silicon as well [reported][T3701][T3883].

Each XSI MAC keeps debug counters from offset `0x104` (TX octets) and `0x190` (RX pause).
Bits 5 and 4 of offset 0 enable TX and RX flow control [source][ACK72].

> [!CAUTION]
> **Leave the XSI MAC's `TX_FRAG_LEN` at the vendor's 31.** Writing 30 on a running MAC stalls
> its transmit. Lower values underrun the TX FIFO and drop the link [reported][ACK72].

## PPE

The PPE binds a flow after a set number of packets. `REG_PPE_BIND_RATE` holds 30 (`0x1e`)
[reported][T3890]. A threshold of 8 cut a loaded network's 90th-percentile latency from
157 ms to 15 ms [reported][T3890]. The PPE handles NAT, VLAN tags and PPPoE [reported][T2599].

A bound IPv6 flow rewrites its source MAC from a slot table rather than the entry. GDM base
slots are 1 to 4, and slot 15 preserves the source MAC. The pool holds at least 11 more
distinct MACs [source][ACK72]. In a PPE entry's `ib1` word, bits 19:16 hold the VLAN layer and
bits 21:20 the VPM [reported][B301].

Linux exposes the table in debugfs, at `/sys/kernel/debug/ppe/bind` and `entries`
[source][ACK72].

## NPU

The NPU runs flow offload between the frame engine and a PCIe Wi-Fi chip. It takes the place
of MediaTek's WED on this SoC, so the `mt7996e` module runs with `wed_enable=N`
[consensus][B136][B436][B438].

| Property | Value |
|---|---|
| Base | `0x1e90_0000`, 0x31_3000 bytes, 15 interrupts [source][DTSI] |
| Cores | 8 RISC-V harts, `misa` `0x4080_1105`, at 800 MHz [reported][SDR][TEARDOWN] |
| Firmware | `airoha/en7581_MT7996_npu_rv32.bin` and `en7581_MT7996_npu_data.bin` [source][DTSI] |
| Firmware version | `0.1111` [consensus][T2871][B392] |

The tree reserves the NPU's memory [source][DTSI]:

| Region | Start | Size | Use |
|---|---|---|---|
| `npu-binary` | `0x8400_0000` | 10 MiB | Firmware |
| `npu-pkt` | `0x8a00_0000` | 44 MiB | Wi-Fi offload |
| `npu-txpkt` | `0x8cc0_0000` | 64 MiB | Wi-Fi offload |
| `npu-txbufid` | `0x90c0_0000` | 26 KiB | Wi-Fi offload |
| `npu-ba` | `0x90c0_6800` | 2 MiB | Wi-Fi offload |

A board with no NPU-offloaded Wi-Fi chip reserves `npu-binary` alone [source][DTSI].

The NPU firmware is not signature-checked: a firmware rebuilt from reverse-engineered code
ran Wi-Fi offload [reported][B397][B398].

> [!CAUTION]
> **Ship the NPU firmware with the kernel.** Without it, the NPU driver waits 60 s, and the
> Ethernet and switch drivers stay in deferred probe [consensus][B77][T3652][T3726].

> [!CAUTION]
> **Never read the NPU's window, `0x1e90_0000` to `0x1ebf_ffff`, from the CPU.** A read there
> hangs the SoC [reported][SDR].

## Errata

### Erratum 1: the 10G link does not come up

**Description:** the Ethernet PCS's PLL can settle in the wrong VCO band. The analog block's
`TCLVAR` field at `0x1fa7_a030` reads `0x0000_301b` in that state.

**Conditions:** `lan2` on some W1700K HW1.x units, mostly at a cold power-on.

**Symptom:** `lan2` shows no link after boot. A cable replug does not help.

**Workaround:** write `0x0000_301d` to `0x1fa7_a030`. The link comes up in 1-2 s. The
community patch writes `TCLVAR` 3, then 5, after `airoha_pcs_config()`.

**Status:** open in OpenWrt main as of 2026-08-26. Units that never fail are also reported.

Sources: [reported][T3879][T3883][T3884][T3885].

### Erratum 2: hardware GRO corrupts TCP without timestamps

**Description:** OpenWrt's 6.18 series adds hardware GRO in patch `916-02`, "Implement HW-GRO
TCP support". For an IPv4 TCP segment without the timestamp option, it treats the segment's
pad bytes as payload.

**Conditions:** IPv4 TCP streams without timestamps, received on a 10G port.

**Symptom:** corrupt data in the stream.

**Workaround:** `ethtool -K <port> rx-gro-hw off` on each 10G port.

**Status:** open as of 2026-09-29.

Sources: [reported][T3997].

### Erratum 3: a receive ring stops after descriptor exhaustion

**Description:** `airoha_eth` enables the `RX_NO_CPU_DSCP` interrupt but does not handle it.
A ring that runs out of descriptors is never refilled.

**Conditions:** a receive burst that empties a ring.

**Symptom:** the ring's traffic stops for good.

**Workaround:** handle the interrupt by refilling the ring.

**Status:** a fix is posted to linux-mediatek, 2026-08 (OpenWrt issue 24715).

Sources: [source][ACK72].

### Erratum 4: an oops with NAT loopback on a bridge

**Description:** `airoha_eth` reserves 14 bytes of receive headroom.
`br_nf_pre_routing_finish_bridge` writes 16 bytes before the packet data, through
`neigh_hh_bridge`.

**Conditions:** NAT loopback with `nf_call_iptables=1` on a bridge.

**Symptom:** a kernel oops.

**Workaround:** disable NAT loopback on each redirect.

**Status:** fixes posted to netdev on 2026-05-13: `airoha-rx-headroom-v1` and
`nf-neigh_hh_bridge-fix-v3`.

Sources: [reported][B109][B119].

### Erratum 5: the XSI MAC transmits after a pause frame

**Description:** the WAN's XSI MAC sends 2-3 more frames after it receives XOFF.

**Conditions:** `wan` at 1 Gb/s, with a link partner that sends pause frames.

**Symptom:** a single TCP upload reaches about 27 Mb/s on a 100 Mb/s service.

**Workaround:** advertise only `100baseT/Full` on `wan`, or carry the WAN on a switch port.

**Status:** open as of 2026-11.

Sources: [reported][ACK72].

## Sources

Citation keys:

- `[DTSI]`: OpenWrt `target/linux/airoha/dts/an7581.dtsi`, `an7581-npu-mt7996.dtsi` and
  `an7581-npu-wlan.dtsi`, main branch at commit `706052b20fd1` (2026-11-07).
- `[DTS]`: the same tree's `an7581-w1700k-ubi.dts`.
- `[ACK72]`: ACK72/openwrt-w1700k-builds at commit `2ca01b7`: `docs/offload-behavior.md`,
  `docs/network-steering.md`, the probe scripts under `package/w1700k-custom/root/etc/`, and the
  descriptions of `patches/w1700k/0004`, `0008` and `0015`.
- `[SDR]`: h0m3us3r/w1700k-sdr at commit `0b0413e`: `include/hw/an7581/eth.h`, `npu.h` and
  `memmap.h`, and its patch `0010-net-airoha-backpressure-instead-of-tx-drop.patch`.
- `[TEARDOWN]`: hungvu.tech, "Quantum Fiber W1700K teardown, board view, and UART pins",
  2024-10-01: the stock boot log, which reports the NPU's clock.
- `[T<n>]`: post n of the OpenWrt forum thread "Quantum Fiber W1700k support", topic 222776,
  read 2026-11-07.
- `[B<n>]`: post n of the OpenWrt forum thread "Gemtek W1700K Community Builds", topic 249319,
  read 2026-11-07.
