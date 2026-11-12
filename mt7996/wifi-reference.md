# MT7996 Wi-Fi reference

This document covers the MediaTek MT7996, a tri-band Wi-Fi 7 chip, and the RF dies it
drives. It states the chip's identity, host interface, processors, firmware, calibration
data and RF paths, and the hazards of probing it. The reader is integrating the chip on a
board or writing to its firmware interface. MediaTek markets the chip as Filogic 680.

Conditions: the facts rest on one board, the Gemtek W1700K, where the chip sits on an Airoha
AN7581 (see the [W1700K hardware reference](../an7581/gemtek-w1700k/hardware-reference.md#wi-fi)).
Most chip-level facts come from the w1700k-sdr project's register headers, which mark each
value as read from code or measured on the project's units. No reference unit has been
probed yet.

## Overview

| Property | Value |
|---|---|
| Baseband ("D-die") | MT7996, marked MT7996AV on the W1700K |
| RF dies ("A-dies") | One per band: an MT7976 (2.4 GHz) and two MT7977 (5 and 6 GHz) |
| Standards | 802.11be, with 802.11a/b/g/n/ac/ax |
| Spatial streams | 4 per band, transmit and receive |
| Widths | 40 MHz at 2.4 GHz, 160 MHz at 5 GHz, 320 MHz at 6 GHz |
| Host interface | PCIe x2, Gen3, plus a second x1 interface |
| Processors | Two Andes RISC-V MCUs (WM, WA) at 720 MHz, three DSPs at 640 MHz |
| Linux driver | `mt7996e`, part of `mt76` |

Sources: [consensus][T440][T3868], [source][SDR], [reported][TEARDOWN].

## Identity

| Item | Value |
|---|---|
| Primary PCI ID | `14c3:7990` |
| Second interface PCI ID | `14c3:7991` |
| `iwinfo` hardware string | `0x14c3:0x7990 0x14c3:0x6639 [MediaTek MT7996E]` |
| D-die revision register | `0x7001_0204`: `0x8a00` is E1, `0x8b00` is E2 |
| A-die chip IDs | MT7976 `0x7976_8a20`, MT7977 `0x7977_8a10` |
| Firmware HW/SW version | `0x8a10_8a10` |

Sources: [consensus][T50][T4027][B166], [source][SDR], [reported][SDR] for the measured values.

The W1700K's D-die reads `0x8b00`, E2 [reported][SDR]. MediaTek's firmware then runs its E2
DPD tables [source][SDR]. `mt76` names a different product MT7990, PCI ID `14c3:7993`
[source][SDR].

The pad strap at `0x7000_56f0`, bits 16:15, gives the A-die combination. Value 0 is one
MT7976 and two MT7977, the W1700K's [source][SDR].

## Bands, dies and chains

Each band has its own A-die. The band index, the PHY index and the A-die index are the same
number [source][SDR]:

| Band | RF | A-die | Transmit chains | Receive chains | `mt76` receive mask |
|---|---|---|---|---|---|
| 0 | 2.4 GHz | MT7976 (MT7976GN) | 4 | 4, plus a monitor receiver | `0x000f` |
| 1 | 5 GHz | MT7977 | 4 | 4 | `0x00f0` |
| 2 | 6 GHz | MT7977 | 4 | 5 | `0x1f00` |

The calibration EEPROM's `WIFI_CONF` bytes, `0x190` to `0x194`, set the counts: 4/4/4 transmit
paths and 4/4/5 receive paths. Where a band's receive count exceeds its transmit count,
`mt76` adds a mask bit [source][SDR].

The MT7976GN's package does not bring out the 5 GHz chain path of its second synthesizer. On
the W1700K, die 0 is therefore 2.4 GHz only [reported][SDR]. Band 1's front end passes 4.8 to
5.8 GHz and nothing by 6.0 GHz. Band 2's passes from 5.9 GHz, with 4.8-5.9 GHz still 25-45 dB
down [reported][SDR].

### Capabilities under Linux

`iw` reports one wiphy with three radios [consensus][T568][T3868]:

| Radio | Range | Widest channel |
|---|---|---|
| 0 | 2400-2500 MHz | 40 MHz, EHT |
| 1 | 5000-5900 MHz | 160 MHz, EHT |
| 2 | 5900-7200 MHz | 320 MHz, EHT |

- **Rates:** 4 spatial streams per band, EHT-MCS 0-13 at every width and MCS15
  [reported][T440].
- **Beamforming:** SU and MU beamformer, and a beamformee of 3 streams [reported][T440].
- **Interfaces:** per radio, up to 16 AP or mesh interfaces, 19 interfaces in all, and one
  channel [reported][T3868].
- **Features:** background radar detection, BSS color and airtime queue limits, with radar
  detection up to 160 MHz [reported][T3868].

A 320 MHz link between two W1700Ks ran at `3891.4 MBit/s 320MHz EHT-MCS 6 EHT-NSS 3`
[reported][T2985].

## Host interface

The primary interface is a PCIe Gen3 x2 endpoint. The second, `14c3:7991`, is a Gen2 x1
endpoint that gives the same chip a second DMA path. It is not a second radio [reported][SDR].
On the AN7581 under Linux 6.18 the primary link trains at Gen2 x2 (see the
[AN7581 PCIe](../an7581/system-reference.md#pcie)).

| Property | Value |
|---|---|
| BAR0 | 2 MiB, 32-bit |
| DMA masks | 36-bit streaming, 32-bit coherent |
| Max Payload Size offered | 256 bytes |
| AER | At config offset `0x200`. No DPC |

Sources: [reported][SDR].

### BAR0 map

BAR0's low 1 MiB is a fixed map of the chip's blocks [source][SDR]:

| BAR0 offset | Block |
|---|---|
| `0x0_2000` | MCU_DMA0 |
| `0x1_0000` | PCIe MAC. Its ID register reads `0x0051_1163` |
| `0x8_0000` | MCU SYSRAM |
| `0xb_0000`, `0xc_0000` | WF_TOP_MISC, off and on domains |
| `0xd_0000` | CONN_WFDMA. WFDMA0 is at `0xd_4000` |
| `0xe_0000` | CONN_HOST_CSR_TOP. `FW_STATE` is at `0xe_00f0` |
| `0xf_0000` | CONN_INFRA |

Two windows reach the rest. The L1 window is 64 KiB at BAR0 `0x13_0000`, selected through
`0x15_5024`. The L2 window is 4 KiB at `0x1000`, selected through `0x1b4` [source][SDR].
CONN_INFRA is `0x7c00_0000` in the MCUs' view and `0x1800_0000` through the L1 window
[source][SDR].

WFDMA0's transmit ring n is at BAR0 `0xd_4300 + n * 0x10`, and receive ring n at
`0xd_4500 + n * 0x10`. The second interface's WFDMA0 bank is at `0xd_8000` [source][SDR].

`FW_STATE`, bits 2:0, reads 1 in download mode and 7 once the WA runs [reported][SDR].

> [!CAUTION]
> **Never read the WFSYS domain (`0x820c_0000` to `0x830f_ffff`) or UMAC SYSRAM
> (`0x4000_0000`) through BAR0.** Such reads hard-locked the host. The chip stayed wedged
> through FLR, a secondary bus reset and PERST#, until a power cycle [reported][SDR].
> `mt76`'s debugfs `regidx` and `regval` files reach those addresses.

A read that returns `0xdead_2117` marks a wedged window. A power cycle clears it
[reported][SDR].

## Processors

| Processor | Core | Clock | Role |
|---|---|---|---|
| WM | AndeStar V5 RV32 (A45 class), `marchid` `0x8000_0045` | 720 MHz | Main MCU |
| WA | The same | 720 MHz | Assist MCU. It takes every host command once firmware runs |
| DSP x3 | MediaTek MRV5, RV32IMAFC, one per band | 640 MHz | DPD and MemDPD firmware |

Sources: [reported][SDR]. The Wi-Fi subsystem requests 760 MHz for the MCUs, and the
firmware caps them at 720 MHz [reported][SDR]. The chip's crystal is 40 MHz, its bus 320 MHz
and its bus PLL 640 MHz [reported][SDR].

The WM's memory map [source][SDR]:

| Region | Address | Size |
|---|---|---|
| Mask ROM | `0x0080_0000` | 320 KiB window |
| ULM1: ROM patch and WM code | `0x0090_0000` | 224 KiB |
| ULM2 (data) | `0x0220_0000` | 256 KiB |
| WF MCU SYSRAM, uncached | `0x0040_0000` | 416 KiB |
| CRAM, shared by WM and WA | `0xe000_0000` | 2.5 MiB |
| UMAC frame pool (PLE, PSE, capture) | `0xe040_0000` | 1408 KiB, 704 banks of 2 KiB |

The host sends firmware download on transmit ring 16, WM commands on ring 17 during download,
and WA commands on ring 20 after. Events arrive on receive rings 0 and 1 [reported][SDR].

## Firmware

Linux loads these files from `/lib/firmware/mediatek/mt7996/` [reported][B393]:

| File | Container |
|---|---|
| `mt7996_rom_patch.bin` | ROM patch, one section, loaded to `0x0090_0000` |
| `mt7996_wm.bin` | WM image, 9 regions |
| `mt7996_dsp.bin` | DSP image, 6 regions: one image loaded per band |
| `mt7996_wa.bin` | WA image, 3 regions |

`_233` variants of each file also ship [reported][B393]. The containers and their checks:

- **Encryption:** AES-128-CBC with an all-zero key and IV, reset per region. Region feature
  bit 0 marks an encrypted region [source][SDR].
- **Signature:** the ROM patch ends in a 1024-byte footer with an RSA-2048 key and an
  RSASSA-PSS SHA-256 signature. The ROM checks the key against hashes in eFuse
  [source][SDR].
- **Enforcement:** on the unit the project examined, the eFuse key hashes are blank and the
  pre-authentication gate is open. Patches with any signature load [reported][SDR].
- **Patch mechanism:** a 64-entry instruction-redirect table at `0x0090_0000`. MediaTek's
  patches use 25 entries (2024 builds) or 26 (from 2025-02) [source][SDR].

Builds seen under OpenWrt: 20240809, 20250904, 20251226 and 20260311
[consensus][T50][T551][T3453][B4].

> [!CAUTION]
> **Running a ROM patch from MediaTek's 2025-02 SDK or later can burn an eFuse bit.** These
> patches run an "FT-mapping recovery" at every MCU start [source][SDR]. The recovery acts on a
> chip whose eFuse word `0x2b0` holds FT version 1 or 2 with bits 14 and 15 clear. It programs
> rows `0x3e0`, `0x3d0` and `0x3c0`, then sets bit 14 or 15 of word `0x2b0` for good.
>
> The patches in linux-firmware are of this kind, and build `20240726161535a` is not. The
> bit's effect on later firmware is not known.

## Calibration EEPROM

The chip's calibration EEPROM image is 16 KiB, of which the first `0x1e00` bytes are used.
It starts with the chip ID, bytes `90 79` [source][SDR][INSTALLER]. On the W1700K it lives in
the factory data, and OpenWrt passes it to `mt76` through NVMEM (see
[factory data](../an7581/gemtek-w1700k/boot-and-flash.md#factory-data)).

| Offset | Contents |
|---|---|
| `0x190`-`0x194` | `WIFI_CONF`: band selection and chain counts |
| `0x196` + band | External LNA (bit 0) and PA (bit 1) |
| `0x199` + band | Front-end module type |
| `0x19c` + band | Front-end pin polarity |
| `0xbaa`, `0x57e`, `0x127e` | Resistor calibration, per die |
| `0x1269`, `0x126f` | Crystal trim, and its signed offset |
| `0x5b00` | Receive gain calibration |

Sources: [source][SDR].

The crystal trim lands on die 2's TOP registers `0x654` and `0x658`, bits 30:24. It moves every
LO and the sample clock [source][SDR].

## The A-dies

The MT7996 reaches each A-die through its own SPI master, at MCU address `0x7c09_8000 +
0x1000 * die` [source][SDR]. Each A-die register is a 32-bit word named by a 12-bit index.
Banks 0 to 4 serve chains WF0 to WF4, and bank 15 (TOP) holds the chip ID, crystal trim and
front-end settings [source][SDR].

| Property | Value |
|---|---|
| Chip selects | WF0 CS1, WF1 CS0, WF2 CS6, WF3 CS7, WF4 CS3, TOP CS5 |
| Lock | CONN hardware semaphore 12: a read of `0x7c07_0030` acquires it, a write of 1 to `0x7c07_0230` releases it |
| Cost | One transaction takes 9 register reads, 46 us of WM time |

Sources: [source][SDR], [reported][SDR] for the cost.

The A-dies keep their register values through an MT7996 MCU or Wi-Fi subsystem reset, until a
power cycle [reported][SDR].

> [!CAUTION]
> **Reading the semaphore register acquires the semaphore.** A probe that reads `0x7c07_0030`
> takes the A-die lock from the firmware [source][SDR].

> [!CAUTION]
> **Never read A-die TOP registers `0x200` to `0x2ff`.** Register `0x222` wedges the SPI path
> [reported][SDR].

### Synthesizers

| A-die | PFD frequency | LO from VCO | Range |
|---|---|---|---|
| MT7976 | 80 MHz | VCO x 5/16, or x 3/8 | LO 2.08-2.74 GHz |
| MT7977 | 160 MHz | VCO x 5/8, or x 3/4 | LO 4.65-7.41 GHz |

Sources: [reported][SDR]. Software runs the VCO band search. MediaTek's firmware tunes an
MT7977 from 4850 to 7125 MHz only: 5/8 below 5926 MHz and 3/4 from 5926 MHz [source][SDR].

## Regulatory and transmit power

OpenWrt's `mt7996e` applies no transmit-power limit on the W1700K as of 2026-06. The radios
transmit at their table maximum whatever the configuration asks, and
`/sys/kernel/debug/ieee80211/phy0/mt76/band0/txpower_sku` shows `SKU: disable`
[consensus][T3597][T3620][T3622]. Fixes for `iwinfo`, `mt76` and the wireless scripts are in
review [reported][T3763][T3771][T3773].

The board's vendor tables select SKU index 3 on the W1700K and 4 on the W1700K2 for 6 GHz
[reported][T3286]. The stock firmware is certified for 6 GHz standard power with automated
frequency coordination. OpenWrt runs low-power indoor only [reported][T225][T2596].

With country `US`, OpenWrt's regulatory database marks 6 GHz `NO-IR`, so no access point
starts there [consensus][T2684][B426][B527].

> [!CAUTION]
> **MediaTek's firmware transmits a full-scale burst on a radio reload or a channel switch.**
> A nearby receiver saw +1 dBFS, 41-44 dB above the noise [reported][SDR]. Attenuate before
> cabling an antenna port to an instrument.

## Sources

Citation keys:

- `[SDR]`: h0m3us3r/w1700k-sdr at commit `0b0413e`. Its register headers under
  `include/hw/mt7996/` (`host.h`, `phy.h`, `conn.h`, `mcu_map.h`, `mcu_periph.h`,
  `fw_format.h`, `rom_patch.h`, `secure.h`, `wfdma.h`, `dsp.h`) and `include/hw/adie/`
  (`adie.h`, `mt7976.h`, `mt7977.h`), and `README.md`. The headers mark each value as read
  from code or firmware, graded `[source]` here, or measured on the project's units, graded
  `[reported]`.
- `[INSTALLER]`: hurrian/w1700k-ubi-installer at commit `46df1f5`,
  `files/installer/install.sh`: the EEPROM's location and magic.
- `[TEARDOWN]`: hungvu.tech, "Quantum Fiber W1700K teardown, board view, and UART pins",
  2024-10-01: the chip markings.
- `[T<n>]`: post n of the OpenWrt forum thread "Quantum Fiber W1700k support", topic 222776,
  read 2026-11-07.
- `[B<n>]`: post n of the OpenWrt forum thread "Gemtek W1700K Community Builds", topic 249319,
  read 2026-11-07.
