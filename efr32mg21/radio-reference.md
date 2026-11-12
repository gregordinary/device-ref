# EFR32MG21 radio reference

This document covers the Silicon Labs EFR32MG21, a 2.4 GHz wireless SoC for Bluetooth LE,
Zigbee and Thread. It states the chip's identity, memory, radio, security and bootloader, and
the protocols its firmware speaks to a host. The reader is integrating the chip as a
co-processor on a Linux host's UART, or replacing its firmware from that host.

Conditions: the chip facts come from Silicon Labs' data sheet, reference manual, bootloader
guides and SDK source, and are graded `[source]`. Behavior observed on a board rests on the
Gemtek W1700K, which carries the chip on an Airoha AN7581's UART (see the
[W1700K hardware reference](../an7581/gemtek-w1700k/hardware-reference.md#efr32mg21)). No
reference unit has been probed yet.

## Overview

| Property | Value |
|---|---|
| Family | EFR32xG21, the first Series 2 Wireless Gecko family |
| Part on the W1700K | EFR32MG21A010F512IM32 |
| Core | Arm Cortex-M33 at up to 80 MHz, with FPU and TrustZone |
| Memory | 512 kB of flash in 8 kB pages, 64 kB of RAM |
| Radio | 2.4 GHz only: Bluetooth LE at 1M, 2M, 500k and 125k, and 802.15.4 |
| Transmit power | +10 dBm maximum on the 010 parts |
| Security | Secure Vault Mid, on a Secure Engine with its own CPU |
| Host interfaces | Three USARTs with RTS/CTS, and SWD |
| Supply | 1.71 to 3.8 V on every rail |
| Clock | A 38.4 MHz crystal, the only supported frequency |
| Package | QFN32, 4 x 4 mm, 20 GPIOs |

Sources: [source][DS][RM]. The part on the W1700K is [consensus][T327][DOER].

The data sheet shows only a direct-supply circuit and never mentions a DC-DC converter, so
the chip has none [inferred][DS].

## Identity

### Ordering code

The ordering code `EFR32MG21A010F512IM32-B` decodes field by field [source][DS]:

| Field | Meaning |
|---|---|
| `EFR32MG21` | The Wireless Gecko MG21 family |
| `A` | Secure Vault Mid. `B` is Secure Vault High |
| `010` | A 10 dBm power amplifier. `020` is 20 dBm |
| `F512` | 512 kB of flash |
| `I` | −40 to +125 °C |
| `M32` | QFN, 32 pins |
| `-B` | Silicon revision B, or `-D` for revision D. A trailing `R` marks tape and reel |

The family has 24 ordering codes. They cover security A or B, a 10 or 20 dBm amplifier, 512,
768 or 1024 kB of flash, and revision B or D [source][DS]. The 512 and 768 kB parts carry
64 kB of RAM, and the 1024 kB parts 96 kB [source][DS]. Revision D has its own electrical
tables in the data sheet [source][DS].

Three readouts give the silicon revision:

- **Package marking:** the first letter of the third line is the revision [source][DS].
- **`SYSCFG_CHIPREV`:** MAJOR in bits 5:0, FAMILY in bits 11:6 (48 is MG21) and MINOR in
  bits 19:12 [source][RM].
- **A Bluetooth stack's boot event:** its `hw` field carries the major revision in the upper
  byte and the minor in the lower [source][GSDK].

### Related parts

| Product | Part | Difference from the A010F512 |
|---|---|---|
| Home Assistant Connect ZBT-1 (SkyConnect) | EFR32MG21A020F512IM32 | A 20 dBm amplifier |
| Home Assistant Yellow | MGM210PA32JIA2, a module on the EFR32MG21 | 1024 kB of flash, 96 kB of RAM and a 20 dBm amplifier |

Sources: [source][NCFWB][NCYELLOW][MGM210P]. Firmware built for the ZBT-1 fits the
A010F512's memory, but it sets up a 20 dBm amplifier and the ZBT-1's UART pins
[inferred][NCFWB]. A port to another board changes the pins, the flow control and the power
settings.

### Series 2, not Series 1

The reference manual names xG21 "the first device in the Series 2" Wireless Gecko portfolio
[source][RM]. A Series 1 part such as the EFR32MG12 differs in ways that change an
integration [source][MG12][BF]:

| Property | EFR32MG21, Series 2 | EFR32MG12, Series 1 |
|---|---|---|
| Core | Cortex-M33, 80 MHz | Cortex-M4, 40 MHz |
| Radio | 2.4 GHz | 2.4 GHz and sub-GHz |
| DC-DC converter | None | Integrated |
| Bootloader location | Main flash at `0x0` | Information block at `0x0fe1_0000` |
| First boot stage | The Secure Engine's ROM | A first-stage bootloader in flash |

## Memory map

| Region | Address | Size | Content |
|---|---|---|---|
| Main flash | `0x0000_0000` | 512 kB, 64 pages of 8 kB | Bootloader, application, NVM3 and tokens |
| RAM | `0x2000_0000` | 64 kB | Its first word carries the bootloader request |
| User data | `0x0fe0_0000` | 1 kB | Survives a device erase |
| Device information | `0x0fe0_8000` | 1 kB | Factory device information |

Sources: [source][DS][RM]. With a standard Gecko bootloader, main flash divides as follows
[source][BF][UG489][GSDK]:

| Range | Use |
|---|---|
| `0x0_0000` to `0x0_3fff` | The Gecko bootloader, up to 16 kB |
| `0x0_4000` up | The application |
| `0x0_8000` to `0x1_3fff` | The staging area for a bootloader upgrade, inside the application |
| `0x7_e000` to `0x7_ffff` | The top page: lock bits, manufacturing tokens and bootloader keys |

The top page's range follows from the 512 kB size [inferred]. A TrustZone-aware
communication bootloader takes 24 kB, so its application starts at `0x6000`
[source][UG489][GSDK]. An application must be linked for the base its bootloader leaves free.

## Radio

Two single-ended pins carry the antenna path [source][DS]. The ordering code fixes the
maximum transmit power. On the 010 parts it is +10.5 dBm at revision B and +10 dBm at
revision D [source][DS].

Typical receive sensitivity [source][DS]:

| Mode | Revision B | Revision D |
|---|---|---|
| 802.15.4, 1% PER | −104.5 dBm | −104.3 dBm |
| Bluetooth LE 1 Mbps, 0.1% BER | −97.5 dBm | −97.1 dBm |
| Bluetooth LE 2 Mbps | −94.4 dBm | −94 dBm |
| Bluetooth LE 500 kbps | −100.6 dBm | −100.7 dBm |
| Bluetooth LE 125 kbps | −104.9 dBm | −105 dBm |

Typical supply current at 3.0 V [source][DS]:

| Condition | Current |
|---|---|
| Receive, Bluetooth LE 1 Mbps | 8.8 mA |
| Receive, 802.15.4 | 9.5 mA |
| Transmit 0 dBm | 10.5 mA |
| Transmit 10 dBm on the 10 dBm amplifier | 34.0 mA at revision B, 35.4 mA at revision D |

The 500 kbps and 125 kbps modes are the LE Coded PHY [inferred]. The data sheet
characterizes the two rates without using that name. It states no qualified Bluetooth core
version, and its receive tables cite core specification 5.1 [source][DS].

## Clocks

The radio needs a 38.4 MHz crystal: "Any crystal frequency other than 38.4 MHz is expressly
not supported" [source][DS]. Zigbee needs ±40 ppm over temperature and aging, and Bluetooth
±50 ppm, with a 10 pF load [source][DS]. On-chip capacitors tune the crystal in 0.04 pF
steps, a setting called CTUNE [source][DS].

An SDK build carries its own CTUNE value. With `SL_CLOCK_MANAGER_CTUNE_MFG_HFXO_EN` at its
default of 1, a CTUNE value stored as a manufacturing token takes precedence
[source][SISDK][inferred]. A 32.768 kHz crystal on PD00 and PD01 times the low-energy modes
[source][DS].

## Host interfaces

### UARTs

The chip has three USARTs and no EUSART [source][DS][RM]. Each USART runs an asynchronous
UART with RTS/CTS flow control [source][DS]. Two internal buses route their signals: one
serves ports A and B, the other ports C and D [source][RM]. The USARTs therefore differ in the
pins they reach [source][DS]:

| USART | Ports |
|---|---|
| USART0 | A, B, C and D |
| USART1 | A and B |
| USART2 | C and D |

Only port A and B pins keep working in EM2. Port C and D pins hold their state there
[source][DS][RM].

### Reset

RESETn is pin 9. It has an internal pull-up of about 44 kΩ to DVDD, and a low pulse of
100 ns or more resets the chip [source][DS]. `EMU_RSTCAUSE`, at offset `0x094` in the EMU
block, records the cause of the last reset [source][RM]:

| Bit | Cause |
|---|---|
| 0 | POR: power-on |
| 1 | PIN: RESETn |
| 2 | EM4 wakeup |
| 3, 4 | Watchdogs 0 and 1 |
| 5 | Cortex-M33 lockup |
| 6 | SYSREQ: a Cortex-M33 system reset request |
| 7 to 11 | Brown-out on DVDD, DVDD in low-energy modes, DECOUPLE, AVDD and IOVDD0 |
| 13 | Secure Engine tamper |
| 14, 15 | Secure Engine system reset and lockup |

Software clears the register with `EMU_CMD.RSTCAUSECLR` [source][RM].

> [!CAUTION]
> **Firmware logs print Silicon Labs' reset codes, not `EMU_RSTCAUSE`.** OpenThread on an
> EFR32 logs `Reset info: 0x4 (PWR)` and `Extended Reset info: 0x401 (HV )` for a power-on
> reset [source][SISDK]. The code carries a base type in its upper byte and an extension in
> its lower. Read as a register value, `0x4` is the EM4 wakeup bit [source][RM].

The SDK maps the lowest set bit of `EMU_RSTCAUSE` to the code, so a power-on reset hides any
other cause recorded with it [source][SISDK]. A RESETn pulse, a watchdog and a software reset
each log their own base type [source][SISDK].

### Debug

SWD runs on PA01 (SWCLK) and PA02 (SWDIO). SWO shares PA03 with TDO and TRACEDATA0
[source][DS]. After a power cycle the debug port starts in JTAG mode, and a switch to SWD
lasts until the next power cycle [source][RM]. Firmware can disable the debug pins through
`GPIO_DBGROUTEPEN`, and a reset enables them again [source][RM].

## Security

A Secure Engine with its own CPU runs the first boot stage and the cryptographic functions,
isolated from the Cortex-M33 [source][DS]. Its authentication of its own root code "cannot
be disabled or bypassed" [source][DS]. Secure Vault Mid, on the `A` parts, lacks several
Secure Vault High features [source][DS]:

| Feature | Mid (A) | High (B) |
|---|---|---|
| Secure boot with RTSL, secure debug, a TRNG | Yes | Yes |
| AES, ECDSA and ECDH on P-192 and P-256, SHA-1, SHA-256 | Yes | Yes |
| Anti-tamper, attestation, PUF key wrapping | No | Yes |
| ChaCha20-Poly1305, P-384, P-521, SHA-384, SHA-512, PBKDF2 | No | Yes |

The data sheet's front-page feature list mixes the two levels. Its Table 3.1 separates them
[source][DS].

### One-time settings

The Secure Engine's OTP memory carries the public signing key, the secure-boot flag and the
anti-rollback flag. Each is written once and never changes [source][RTSL]:

- **Secure boot:** the Secure Engine checks the image at `0x0` against the OTP key before it
  releases the Cortex-M33. An unsigned or wrongly signed bootloader leaves the core frozen,
  with boot status `0x12` [source][RTSL].
- **Anti-rollback:** the Secure Engine keeps a 4-byte bootloader version counter and blocks a
  lower version at boot [source][RTSL].
- **Page lock:** validated pages refuse writes from anything but the Secure Engine
  [source][RTSL].

On xG21, the Secure Engine ignores the other OTP options while secure boot is off
[source][RTSL]. It then checks no signature on a bootloader it installs [source][RTSL].

### Debug lock and device erase

| State | Debug port | Reopened by |
|---|---|---|
| Standard lock | Closed | A device erase |
| Secure lock | Closed | A token signed by the owner's key, or a device erase where erase is enabled |
| Permanent lock | Closed, with device erase disabled and no secure-debug key | Nothing |

Source: [source][SDBG]. A device erase wipes main flash and RAM, then unlocks the debug port
[source][SDBG]. It leaves the user data page, the Secure Engine's firmware and its OTP
settings [source][SDBG][RM]. On xG21 the bootloader sits in main flash, so a device erase
removes it too [inferred].

Simplicity Commander's `security` commands read the Secure Engine's state over SWD
[source][RTSL]. Firmware reads it with `sl_se_get_status()` and `sl_se_read_otp()`
[source][GSDK]. The Gecko bootloader's UART protocols carry no command that reports it
[inferred][GSDK].

## The Gecko bootloader

### Placement

The Secure Engine supplies the first boot stage, which no user code can reach. The Gecko
bootloader is the main stage, and it must sit at the start of flash [source][RM][UG489]. The
Secure Engine installs a new bootloader by copying it from a staging area in flash to `0x0`
[source][UG489].

### Communication bootloaders

A communication bootloader takes a new image over a host link. Each build carries one
protocol, because every protocol component supplies the same entry points
[source][UG489][GSDK]:

| Protocol | Silicon Labs pairs it with | Host tools |
|---|---|---|
| UART XMODEM | EmberZNet and Connect NCPs | universal-silabs-flasher, `cpcd -f` |
| BGAPI UART DFU | Bluetooth NCPs | `bt_host_uart_dfu` |
| SPI EZSP | EmberZNet NCPs on SPI | |

Sources: [source][UG489][USF][CPCD]. Both UART protocols default to 115200 baud, 8N1,
without flow control [source][GSDK]. Because each build carries one protocol, a BGAPI
bootloader answers no XMODEM, and an XMODEM bootloader no BGAPI [inferred][GSDK].

### Entry

At every reset, the bootloader runs three checks in order, and the first that matches keeps
it in the bootloader [source][GSDK]:

1. After a system reset request, it reads the RAM word at `0x2000_0000`. Reason `0x0202`
   with signature `0xf00f` is a request to stay.
2. If the build carries GPIO activation, it samples the activation pin, active low by
   default.
3. If the application's reset vector is blank, or the application fails verification, it
   stays with reason BADAPP.

A RESETn pulse is not a system reset request, so a software request left in RAM does not
survive it [inferred][GSDK]. To enter by pin, hold the activation pin active, pulse RESETn,
and release the pin after RESETn rises [source][EZSP].

The bootloader drives the activation pin briefly before it samples it [source][GSDK]. Drive
the pin through a resistor or an open-drain output [inferred]. A board that holds the pin at
its active level keeps the chip in its bootloader at every reset [inferred][GSDK].

Each firmware type enters its bootloader on a host command [source][GSDK][USF]:

| Running firmware | Host command | Bytes or ID |
|---|---|---|
| Bluetooth NCP, Gecko SDK 4.1 or later | `user_reset_to_dfu` | `20 00 ff 02` |
| Bluetooth NCP, Gecko SDK 4.0 | `system_reset` in DFU mode | `20 01 01 01 01` |
| EmberZNet NCP | EZSP `launchStandaloneBootloader` | Frame ID `0x008f` |
| OpenThread RCP | Spinel `CMD_RESET` with reset type 3 | |
| CPC multiprotocol RCP | Set property `0x0202` to 1, then a CPC reset | |

From Gecko SDK 4.1 on, a Series 2 Bluetooth stack ignores the DFU flag of `system_reset` and
boots the application [source][GSDK]. A host tool from Gecko SDK 4.0 therefore restarts a
newer NCP into its application [inferred].

### The BGAPI UART DFU protocol

A frame carries a 4-byte header of type, length, class and ID, then its payload, with no
checksum [source][GSDK]. The bootloader handles these frames [source][GSDK][SISDK]:

| Frame | Bytes | Behavior |
|---|---|---|
| `dfu_boot` event | `a0 04 00 00`, then a 32-bit version | Sent at every bootloader start |
| `dfu_boot_failure` event | `a0 02 00 01`, then a 16-bit reason | Sent before `dfu_boot` after an application fails verification, reason `0x0046` |
| `dfu_flash_set_address` | `20 04 00 01 00 00 00 00` | Deprecated. Does nothing and answers success |
| `dfu_flash_upload` | `20`, n+1, `00 02`, n, then n image bytes | Feeds the image to the GBL parser |
| `dfu_flash_upload_finish` | `20 00 00 03` | Succeeds only for a complete, verified image |
| `system_reset` | `20 01 01 01`, then the mode | Mode 0 boots the application, mode 1 restarts the bootloader |
| `system_reboot` | `20 00 01 1f` | Handled by Simplicity SDK bootloaders only |
| `dfu_reset` | Class `0x00`, ID `0x00` | Ignored |

A response is `20 02 00`, the command's ID, then a 16-bit result. Silicon Labs' host example
sends 48-byte blocks, and the guide gives 128 bytes, a multiple of 4, as the largest usable
block [source][BTBL].

> [!CAUTION]
> **Match the DFU host tool to the bootloader's SDK.** A Simplicity SDK `bt_host_uart_dfu`
> ends with `system_reboot`, and a Gecko SDK bootloader ignores that frame [source][SISDK].
> The chip then stays in its bootloader [inferred]. Send `20 01 01 01 00` to start the
> application.

The Bluetooth stack's DFU mode uses the bootloader's baud rate [source][GSDK]. An NCP and its
bootloader must therefore share one rate for a single-rate host tool to reach both
[inferred].
Silicon Labs' `bt_host_uart_dfu` has these limits [source][GSDK][SISDK]:

- **Rate:** builds from Gecko SDK and Simplicity SDK 2024.6 accept 115200 baud at most.
  Simplicity SDK 2025.6 adds 230400, 460800 and 921600.
- **Flow control:** RTS/CTS is on by default, and `-f` turns it off.
- **Blocks:** 48 bytes per `dfu_flash_upload`.

### The XMODEM bootloader

At start, and again on any byte it does not recognize, the XMODEM bootloader prints its menu
[source][GSDK]:

```
Gecko Bootloader v2.04.02
1. upload gbl
2. run
3. ebl info
BL >
```

The version in the banner is an example. Key `1` starts an XMODEM-CRC transfer in 128-byte
packets, and key `2` boots the application [source][GSDK]. Without a transfer for 60 s, the
bootloader returns to its menu [source][GSDK]. The tools universal-silabs-flasher and `cpcd -f`
speak only this protocol [source][USF][CPCD]. The flasher has no Bluetooth application type
[source][USF].

### GBL images

A GBL file is a sequence of tags, each a 32-bit ID, a 32-bit length and a payload
[source][UG489]. The header tag comes first and begins the file with `eb 17 a6 03`. The end
tag comes last and carries a CRC-32 of the file [source][UG489]. On Series 2, an application
and a bootloader travel in separate GBL files [source][UG489].

| Protection | Mechanism | Required by default |
|---|---|---|
| Signature | ECDSA P-256 over the file | No: `BOOTLOADER_ENFORCE_SIGNED_UPGRADE` is 0 |
| Encryption | AES-128 in CTR mode, though the header flag is named AESCCM | No: `BOOTLOADER_ENFORCE_ENCRYPTED_UPGRADE` is 0 |

Sources: [source][UG489][GSDK]. A bootloader that requires no signature still verifies a
signature tag it finds [source][UG489]. A signed file therefore fails on a bootloader that
holds no matching key [inferred].

### Bootloader upgrade

A running bootloader installs a new bootloader in four steps [source][GSDK][UG489]:

1. It erases the first application page and writes the new bootloader to the staging area,
   `0x8000` by default.
2. After `dfu_flash_upload_finish` succeeds, it compares the new version with its own, and
   continues only for a greater version.
3. It checks the staged image's CRC, then asks the Secure Engine to check the image.
4. If the Secure Engine accepts it, the Secure Engine copies the image to `0x0` and resets
   the chip.

The staging area overlaps the application, so an upgrade invalidates it. An application GBL
follows the bootloader GBL [source][UG489].

> [!CAUTION]
> **A refused bootloader upgrade is silent.** `dfu_flash_upload_finish` answers success
> before the version and Secure Engine checks run [source][GSDK]. The old bootloader stays,
> and the application's first page is already erased. The chip therefore restarts in its
> bootloader with reason BADAPP [inferred]. Send an application GBL to recover.

Four conditions refuse an unsigned bootloader upgrade [source][GSDK][RTSL]:

| Condition | The host sees | Recovery |
|---|---|---|
| The running bootloader requires signed upgrades | `0x0046` from `dfu_flash_upload_finish` | SWD |
| The new version is not greater than the running one | Success, and no upgrade | A higher version |
| OTP secure boot is on, and the image lacks the OTP key's signature | Success, and no upgrade | None for an unsigned image |
| OTP anti-rollback is on, and the version is below the counter | Success, and no upgrade | A signed, higher version |

The grouping into these four rows is [inferred] from the source.

### Version numbers

A bootloader version is a 32-bit value. Major sits in bits 31:24, minor in bits 23:16 and a
customer number in bits 15:0 [source][GSDK]. Every Gecko SDK bootloader is major 2, and
major 3 begins with Simplicity SDK 2024.12.0 [source][GSDK][SISDK]:

| SDK | Bootloader version |
|---|---|
| Gecko SDK 4.1.0 | 2.1 |
| Gecko SDK 4.3.0 | 2.3 |
| Gecko SDK 4.4.0 to 4.5.1 | 2.4 |
| Simplicity SDK 2024.6 | 2.5 |
| Simplicity SDK 2024.12 | 3.0 |
| Simplicity SDK 2025.6 | 3.1 |

The customer field is the one a build raises to replace a bootloader of equal major and minor
[source][UG489]. The running bootloader compares the whole 32-bit value [source][GSDK]. With
a 3.x bootloader in place, no Gecko SDK bootloader can follow it as a GBL
[inferred][T3769]. One poster patched the version check out of the new image and still saw
the upgrade refused [reported][T3769]. The check that runs is the installed bootloader's
[source][GSDK].

### Recovery over SWD

If OTP secure boot is off and device erase is enabled, SWD recovers the chip from any
bootloader state [inferred][SDBG][RTSL]. A device erase clears it, then a bootloader goes to
`0x0` and an application after it. `commander flash --masserase` with a bootloader file
performs the first two steps [source][PROD]. One poster wrote the chip with an ST-Link and
the out-of-tree OpenOCD driver knieriem/openocd-efm32-series2 [reported][T508].

## Firmware and host protocols

| Firmware | Protocol on the UART | Framing | Default rate | Flow control |
|---|---|---|---|---|
| Bluetooth NCP | BGAPI | 4-byte header, no checksum | 115200 | RTS/CTS, the host tools' default |
| Bluetooth RCP | HCI over H4 | HCI packets | 115200 | RTS/CTS on by default |
| EmberZNet NCP | EZSP over ASH | `0x7e` flag, byte stuffing, CRC-CCITT, randomized data | 115200 | Required: RTS/CTS, or XON/XOFF at 57600 |
| OpenThread RCP | Spinel over HDLC-Lite | `0x7e` flag, byte stuffing, CRC-16 | 460800 in `otbr-agent` | Required by the Spinel draft, off in `otbr-agent` by default |
| CPC multiprotocol RCP | CPC | `0x14` start byte, header checksum, no stuffing | 115200 in `cpcd` | On in `cpcd` by default |

Sources: [source][GSDK][HCI][ASH][EZSP][SPINEL][OT][CPCD]. The EmberZNet and OpenThread
versions identify the SDK a build came from [source][GSDK][SISDK]:

| SDK | Bluetooth | EmberZNet | OpenThread |
|---|---|---|---|
| Gecko SDK 4.1.3 | 4.2.1 | | |
| Gecko SDK 4.3.2 | 6.2.0 | | |
| Simplicity SDK 2025.6.1 | | 8.2.1 | 2.7.1.0 |
| Simplicity SDK 2025.6.2 | | 8.2.2 | 2.7.2.0 |

### Telling the firmware apart

What the chip sends identifies its firmware:

- **BGAPI bootloader:** `dfu_boot`, `a0 04 00 00` and a version, at every start
  [source][GSDK].
- **XMODEM bootloader:** its menu text [source][GSDK].
- **Bluetooth NCP:** the boot event `a0 12 01 00`, carrying major, minor, patch, build, a
  bootloader field documented as unused, `hw` and a hash [source][GSDK].
- **EmberZNet NCP:** a RSTACK frame in answer to the host's RST frame, `1a c0 38 bc 7e`
  [source][ASH].
- **OpenThread RCP or CPC:** a Spinel or CPC reply to a probe. universal-silabs-flasher
  probes EZSP, Spinel, CPC and the XMODEM bootloader in turn [source][USF].

A Gecko SDK 4.x Bluetooth NCP times out a partial command after 500 ms. It then emits
`a0 03 01 06 4a 00 00`, a `system_error` event with reason `0x004a`,
`SL_STATUS_COMMAND_INCOMPLETE` [source][GSDK][SISDK]. A single stray byte on the UART draws
that event, and its `0x4a` prints as "J" [inferred]. A Simplicity SDK NCP answers with a
command error by default instead [source][SISDK].

### Protocol notes

- **ASH:** a host resets the link with RST and waits for RSTACK. RSTACK's reset code `0x02`
  means power-on, and `0x09` the bootloader [source][ASH].
- **Spinel:** `otbr-agent` takes a radio URL such as
  `spinel+hdlc+uart:///dev/ttyS1?uart-baudrate=460800`. Its options include
  `uart-flow-control` and `enable-coex` [source][OT].
- **CPC:** `cpcd` encrypts the link with AES-GCM by default. `disable_encryption` needs an RCP
  built without CPC security [source][CPCD], [reported][GTX].
- **CPC framing:** without byte stuffing, a resynchronizing receiver can take payload bytes
  for a header. The daemon's documentation advises security or SPI [source][CPCD].

### Multiprotocol firmware

Home Assistant stopped recommending Silicon Labs' multiprotocol firmware on 2024-01-25. It
deprecated its multiprotocol add-on on 2025-07-31 [source][HA]. Its reasons: Zigbee and
Thread then share one channel and its 250 kbit/s [source][HA]. On a radio shared with
Bluetooth, a BlueZ scan at full duty leaves 802.15.4 no receive time [reported][T3962].

### Host software in OpenWrt

OpenWrt's packages feed carries `openthread-br` 2026.10.0, with `otbr-agent` and a netifd
protocol `openthread` that takes a `radio_url` [source][PKG]. It also carries `ser2net`
4.6.2 and BlueZ 5.87 with `btattach` [source][PKG]. It carries no `cpcd` and no zigbee2mqtt
[source][PKG].

## Coexistence

Packet traffic arbitration (PTA) lets a Wi-Fi radio and the EFR32 share the 2.4 GHz band
through wires between them [source][COEX]:

| Signal | Direction at the EFR32 | Function |
|---|---|---|
| REQUEST | Output | Asks for the medium |
| GRANT | Input | Allows a transmission |
| PRIORITY | Output | Marks a request high priority, statically or by direction |
| RHO | Input | Radio hold-off: blocks transmission |

The wiring takes three forms: one wire (GRANT or REQUEST alone), two (REQUEST and GRANT) or
three (adding PRIORITY). The EFR32 has no FREQ input [source][COEX]. A build sets up PTA
through the "RAIL Utility, Coexistence" component, whose pins sit in
`sl_rail_util_coex_config.h` [source][COEX]. `otbr-agent` turns an RCP's PTA on or off with
`enable-coex` [source][OT].

If a build maps no pin to GRANT, the radio treats GRANT as always asserted [source][COEX].
With PTA on and GRANT never asserted, Bluetooth transmits nothing and still receives
[source][COEX]. 802.15.4 checks GRANT at the end of CCA [source][COEX]. An 802.15.4
transmission therefore fails and retries [inferred].

> [!CAUTION]
> **Enable PTA only where the Wi-Fi side drives GRANT.** Otherwise the radio receives and
> never transmits, which looks like a dead network. Linux's `mt76` has no PTA code for the
> MT7996, and its device-tree binding has no coexistence property [source][MT76].

## Errata

The EFR32BG21 errata, for the same xG21 die, lists HFXO_E301. Setting the HFXO's
`DISONDEMAND` bit while it runs, then a system reset request, can hang the chip until a pin
reset [source][BG21E]. The MG21 shares the die, so the erratum applies to it [inferred]. The
MG21's own errata document is not among the sources.

## Sources

Silicon Labs' docs.silabs.com editions carry a documentation-set version instead of a
revision, and each names the application note or user guide it replaces.

Citation keys:

- `[DS]`: Silicon Labs, *EFR32MG21 Multiprotocol Wireless SoC Family Data Sheet*, Rev. 1.2,
  2024-05.
- `[RM]`: Silicon Labs, *EFR32xG21 Wireless Gecko Reference Manual*, Rev. 1.0, 2024-01.
- `[MG12]`: Silicon Labs, *EFR32MG12 Gecko Multi-Protocol Wireless SoC Family Data Sheet*,
  Rev. 1.8, 2022-06.
- `[MGM210P]`: Silicon Labs, *MGM210P Wireless Gecko Multi-Protocol Module Data Sheet*, Rev.
  1.4, 2024-05.
- `[BG21E]`: Silicon Labs, *EFR32BG21 Errata*, Rev. 1.0, 2024-05.
- `[BF]`: Silicon Labs, "Bootloader Fundamentals", docs.silabs.com edition 0.1, replacing
  UG103.06.
- `[UG489]`: Silicon Labs, "Silicon Labs Gecko Bootloader User's Guide for GSDK 4.0 and
  Higher", docs.silabs.com edition 0.3, replacing UG489.
- `[RTSL]`: Silicon Labs, "Series 2 and Series 3 Secure Boot with RTSL", docs.silabs.com
  edition 0.3, replacing AN1218.
- `[SDBG]`: Silicon Labs, "Series 2 and Series 3 Secure Debug", docs.silabs.com edition 0.3.
- `[PROD]`: Silicon Labs, "Production Programming of Series 2 and Series 3 Devices",
  docs.silabs.com edition 0.2, replacing AN1222.
- `[BTBL]`: Silicon Labs, "Using the Gecko Bootloader with Silicon Labs Bluetooth
  Applications", Bluetooth Mesh documentation 11.0.2, replacing AN1086.
- `[ASH]`: Silicon Labs, "UART-EZSP Gateway Protocol Reference", Zigbee documentation 9.1.1,
  replacing UG101.
- `[EZSP]`: Silicon Labs, "EmberZNet Serial Protocol UART Host Interfacing Guide", Zigbee
  documentation 9.1.1, replacing AN706.
- `[HCI]`: Silicon Labs, "Enabling a Radio Co-Processor using the Bluetooth LE HCI Function",
  Bluetooth Mesh documentation 11.0.2, replacing AN1328.
- `[COEX]`: Silicon Labs, "Zigbee and OpenThread Coexistence with Wi-Fi", replacing AN1017,
  and "Wi-Fi Coexistence Fundamentals", replacing UG103.17, Zigbee documentation 9.1.1. Also
  "Bluetooth Coexistence with Wi-Fi", replacing AN1128, coexistence documentation 1.0.1.
- `[GSDK]`: SiliconLabs/gecko_sdk at v4.5.0 (`90fc93e`), with v4.0.0, v4.1.0, v4.1.3 and
  v4.3.2 for version history: `platform/bootloader/` (`core/`, `config/`, `api/`,
  `communication/`), `protocol/bluetooth/inc/sl_bt_api.h`, `app/bluetooth/common/ncp/sl_ncp.c`,
  `app/bluetooth/example_host/bt_host_uart_dfu/` and `app/bluetooth/common_host/`.
- `[SISDK]`: SiliconLabs/simplicity_sdk at v2025.6.3 (`b41bec3`), with v2024.6.0, v2024.12.0,
  v2025.6.1 and v2025.6.2: the same paths, plus `protocol/bluetooth/inc/sl_bgapi.h`,
  `platform/common/inc/sl_status.h`, `platform/service/legacy_hal/` (`reset-def.h`,
  `diagnostic.c`) and `platform/service/clock_manager/config/`.
- `[SPINEL]`: R. Quattlebaum and J. Woodyatt, "Spinel Host-Controller Protocol",
  draft-rquattle-spinel-unified-00, 2017.
- `[OT]`: openthread/openthread at `4edefd9`: `src/posix/platform/radio_url.cpp`,
  `src/lib/spinel/spinel.h` and `src/lib/hdlc/hdlc.cpp`.
- `[CPCD]`: SiliconLabs/cpc-daemon v4.9.1 (`87f6dbd`): `cpcd.conf`, `readme.md` and `doc/`.
- `[USF]`: NabuCasa/universal-silabs-flasher v1.3.0 (`aa6a133`).
- `[NCFWB]`: NabuCasa/silabs-firmware-builder at `37697c5`, the SkyConnect and Yellow
  manifests.
- `[NCYELLOW]`: NabuCasa/yellow at `c3d51b8`, `Radio.kicad_sch`.
- `[HA]`: Home Assistant, blog post "The State of Matter", 2024-01-25. Also
  home-assistant/addons pull request 3833, "Deprecate multiprotocol", merged 2025-07-31.
- `[MT76]`: openwrt/mt76 at `0936617`, and Linux's
  `Documentation/devicetree/bindings/net/wireless/mediatek,mt76.yaml`.
- `[PKG]`: openwrt/packages at `7eef26d`: `net/openthread-br`, `net/ser2net` and
  `utils/bluez`.
- `[GTX]`: gtxaspec/w1700k-efr32 at `3b9a61b`: `README.md`, `bgapi_dfu.py` and
  `enter_bl.py`.
- `[DOER]`: doer.ee, "W1700K" reverse-engineering and Thread firmware pages, 2026-09-13.
  Their measurements were taken on the author's own units, so they are graded `[reported]`.
- `[T<n>]`: post n of the OpenWrt forum thread "Quantum Fiber W1700k support", topic 222776,
  read 2026-11-07.
