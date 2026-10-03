# Veyron Wi-Fi and Bluetooth firmware

This document covers the Broadcom BCM4354 firmware on the Veyron boards that carry the
radio: speedy (the ASUS C201), minnie, mickey and brain. It names the three files the radio
loads and which of them Debian packages. It also names the source that holds the correct
board NVRAM, and the directory the files must be installed in. The Marvell Veyron boards
use other firmware, which Debian packages. See
[the in-tree boards](family-boards.md#the-in-tree-boards). Measurements come from the
reference unit, a C201.

## The radio

Every Broadcom Veyron board carries a BCM4354 on SDIO, driven by `brcmfmac` [source]. On
speedy, minnie and mickey, a BCM4354 Bluetooth companion sits on `uart0`
(`brcm,bcm43540-bt`). The Wi-Fi part is an M.2 (NGFF) module, and that fact decides which
NVRAM file is correct. The reference unit
identifies both halves at probe [measured]:

```
brcmfmac: brcmf_fw_alloc_request: using brcm/brcmfmac4354-sdio for chip BCM4354/1
Bluetooth: hci1: BCM4354 (003.001.012)
```

## The three firmware files

`brcmfmac` first requests a file name that carries the device tree's `compatible`, and then
falls back to the generic name. On the C201 it opens, in order:

```
brcm/brcmfmac4354-sdio.google,veyron-speedy-rev9.bin   then  brcm/brcmfmac4354-sdio.bin
brcm/brcmfmac4354-sdio.google,veyron-speedy-rev9.txt   then  brcm/brcmfmac4354-sdio.txt
brcm/BCM4354.google,veyron-speedy-rev9.hcd             then  brcm/BCM4354.hcd
```

The unpacked `firmware-brcm80211` 20260519-1 package shows which of the three Debian
ships [source]:

| File | Role | In Debian |
|---|---|---|
| `brcmfmac4354-sdio.bin` | Wi-Fi firmware | Yes: a symlink to `cypress/cyfmac4354-sdio.bin`, plus a `clm_blob` |
| `brcmfmac4354-sdio.txt` | Board NVRAM and calibration | No |
| `BCM4354.hcd` | Bluetooth patchram | No |

The only BCM4354 NVRAM in Debian is for an nVidia Jetson
(`brcmfmac4354-sdio.nvidia,p2371-2180.txt`). No Debian package in `main` or
`non-free-firmware` ships a BCM4354 `.hcd`. `bluez-firmware` carries `.hcd` files for
BCM43430A1, BCM4345C0 and similar chips, but not for this one. Neither the NVRAM nor the
patchram is in linux-firmware either. Two of the three files therefore come from outside
Debian, and one comes from Debian.

### The Debian firmware build

Debian's `.bin` is a different build from the one ChromeOS and postmarketOS run. The
difference decides whether the `clm_blob` matters [source]:

| | Debian (Cypress) | postmarketOS and ChromeOS |
|---|---|---|
| Version | `7.35.349.104` (2021) | `7.81.3` (2017) |
| Size | 601704 B | 603662 B |
| CLM | Built `noclminc`, so it requires the separate `clm_blob` | Built in, with no `clm_blob` |

Both are `4354a1-roml`. Debian's pairing is self-consistent: its `.bin` requires a
`clm_blob`, and the package ships one.

Debian's `.bin` with the Veyron NVRAM is an attested configuration. Arch Linux ARM runs it:
its `firmware-veyron` package installs only the `.txt` and takes the `.bin` from
`linux-firmware`, the same Cypress build [source]. The other pairing, the ChromeOS `.bin`
with the same NVRAM, is the one postmarketOS attests. The reference unit associates on
Debian forky and trixie with Debian's `.bin` and the Veyron NVRAM [measured].

The alternative is the full set that ChromeOS and postmarketOS run. It takes jenneron's
`.bin` as well and drops the `clm_blob`, which puts three files outside Debian instead of
two.

## The board NVRAM

ChromiumOS's own `linux-firmware` fork carries a `brcm/brcmfmac4354-sdio.txt`. It is the
wrong file for the Veyron boards: a generic sample for a different BCM4354 board variant
[source].

| Field | ChromiumOS `linux-firmware` | Arch Linux ARM and postmarketOS |
|---|---|---|
| Header | `BCM94354 WLBGA iPA, eLNA+switch` | `BCM94354Z NGFF 22x30mm iPA, iLNA` |
| `boardtype` | `0x0703` | `0x0707` |
| `boardrev` | `0x1102` | `0x1224` |
| `devid` | `0x43df` | `0x43a3` |
| `boardflags` | `0x12401201` | `0x02400201` |

`boardtype`, `boardrev` and `devid` identify the module, and are not tuning values. The
Veyron Wi-Fi is the NGFF (M.2) 22×30 mm module, so the `BCM94354Z NGFF` file is correct. The
`WLBGA` file describes a different silicon package with different RF front-end wiring.
Loading it gives, at best, a badly calibrated radio.

**Do not take `brcmfmac4354-sdio.txt` from ChromiumOS `linux-firmware`.** It looks
authoritative because it is first-party to ChromeOS, but it is first-party to a different
board.

### The correct NVRAM

Two sources carry the correct `BCM94354Z NGFF` NVRAM. They are the same board file and
differ only in TX power offsets, which are calibration rather than identity [source]:

| Setting | Arch Linux ARM (`core/linux-veyron`) | postmarketOS (`jenneron`) |
|---|---|---|
| `cckbw202gpo` | `0x1111` | `0x0000` |
| `mcsbw205g{l,m,h}po` | `0x66666666` | `0x22222222` |

The postmarketOS set is the proven one. It is the set postmarketOS ships for
`google-veyron`, where it marks Wi-Fi working on speedy. It is also byte-for-byte the file running on the reference unit, at
2907 bytes [measured]. Its source is
[`jenneron/firmware-google-veyron-brcm`](https://github.com/jenneron/firmware-google-veyron-brcm),
which postmarketOS's `firmware-google-veyron-brcm` package installs:

```
sha256  1ed835efb2aa2f295aef30f00c01a044579e5b1d1fbf65f04e95add6a146f666   brcmfmac4354-sdio.txt   2907 B
sha256  22cd1a7ba3b7872cb368eab61cc4640b6638c3c3e0b25277a3cc2803a3a1de45   BCM4354.hcd            81417 B
sha256  37a09037634aece36cf7539b4e52e4dffc1234c63d8d5386047743ccdc85df49   brcmfmac4354-sdio.bin 603662 B
```

The `.bin` hash is recorded for the alternative set and for comparison. With Debian's
`.bin`, only the NVRAM and the patchram come from outside Debian.

Arch Linux ARM's `firmware-veyron` package is a useful cross-check but a poor source. It
fetches `https://archlinuxarm.org/builder/src/veyron/brcmfmac4354-sdio.txt` and pins only an
md5 (`13e41678ae81bdd70b3778fb2dc29662`).

## The installation path

Install the NVRAM and the patchram in `/lib/firmware/brcm/`, under their generic names
`brcmfmac4354-sdio.txt` and `BCM4354.hcd`. The driver tries the `compatible`-suffixed name
first and falls back to the generic one. The generic name therefore serves every Veyron
revision and every Broadcom board (speedy, minnie, mickey and brain), with no per-revision
copies.

A file in the wrong directory looks like a driver problem. The postmarketOS package installs
the same files in `/lib/firmware/postmarketos/brcm/`. It relies on a firmware search path
that its kernel command line does not always set, and the radio then fails with:

```
Direct firmware load for brcm/brcmfmac4354-sdio.google,veyron-speedy-rev9.bin failed with error -2
Direct firmware load for brcm/brcmfmac4354-sdio.bin                            failed with error -2
```

`-2` is `ENOENT`: the files are present, in a directory the driver does not search. If a
radio never associates on a board whose files are installed, check this directory before
suspecting the silicon.

## linux-libre kernels

A linux-libre kernel removes the BCM4354 entirely, both halves of it [source]. It rewrites
the `brcmfmac` firmware request to `reject_firmware`, and the `btbcm` patchram name to
`/*(DEBLOBBED)*/`. Wi-Fi and Bluetooth are both dead, whichever files are on disk.

The BCM4354 is the only hardware a Broadcom Veyron board loses to deblobbing. The deblob log
has no hits for the other board drivers:

- panfrost, which survives because the Mali-T764 needs no firmware (see
  [the RK3288 GPU](../video-reference.md#gpu))
- hantro and `rockchip_vdec`
- `analogix_dp`
- max98090
- `dw_mmc` and `dwc2`
- `cros_ec`

`ath9k` has no hits either, so an AR9271 USB dongle is the one radio a linux-libre kernel
keeps. Its firmware, `firmware-ath9k-htc`, is free software in Debian `main`. The Veyron
boards have no Ethernet port, so a radio is their only network path.

## Sources that do not apply

Two sources look relevant to the BCM4354 and do not apply to the Veyron boards.

### winterheart/broadcom-bt-firmware

`winterheart/broadcom-bt-firmware` is a well-kept collection of Broadcom Bluetooth `.hcd`
files extracted from Windows drivers. It carries no BCM4354 file [source].

The collection is keyed by USB VID:PID, and its file names encode it: `0489:e031` becomes
`BCM20702A0-0489-e031.hcd`. That is the naming convention for USB-attached Broadcom radios.
Debian's `firmware-misc-nonfree` ships the same style (`BCM-0a5c-6410.hcd`).

The Veyron Bluetooth is attached over UART through serdev (`brcm,bcm43540-bt` on `&uart0`).
`hci_bcm` therefore requests a chip-named file, `brcm/BCM4354.hcd`, with no VID:PID to key
on. The attachment, the naming scheme and the provenance all differ. The patchram comes from
the ChromeOS Veyron firmware set, as [the correct NVRAM](#the-correct-nvram) does.

### The kernel wireless documentation

The brcm80211 page on `wireless.docs.kernel.org` lists the BCM4354 in two tables, with
opposite verdicts [source]:

| Table | Entry | Note |
|---|---|---|
| PCIe devices | `BCM4354`, `14e4:43df` | "Unreleased hw, support dropped in 4.1" |
| SDIO devices | `BCM4354`, `0x4354` | "Supported in 3.15+" |

The Veyron radio is the SDIO part, the supported row. The reference unit logs
`using brcm/brcmfmac4354-sdio for chip BCM4354/1` [measured].

The PCIe BCM4354's device ID, `0x43df`, is also the `devid` in the ChromiumOS
`linux-firmware` NVRAM. The correct Veyron NVRAM carries `devid=0x43a3`. The ChromiumOS file
is therefore keyed to the unreleased PCIe variant, which independently confirms that it is
the wrong file.

## Licensing

The BCM4354 files are redistributable Broadcom blobs, the same ones Debian ships in
`firmware-brcm80211` in its `non-free-firmware` component [source]. A linux-libre build has
no use for them, so it drops the `non-free-firmware` component, the package and the blobs
together.

## Sources

Every `[source]` claim in this document rests on one of the following:

- The Debian packages `firmware-brcm80211` (20260519-1), `bluez-firmware` and
  `firmware-misc-nonfree`, unpacked.
- The ChromiumOS `linux-firmware` fork.
- Arch Linux ARM's `firmware-veyron` and `core/linux-veyron` packages.
- [`jenneron/firmware-google-veyron-brcm`](https://github.com/jenneron/firmware-google-veyron-brcm)
  and postmarketOS's `firmware-google-veyron-brcm` package.
- `winterheart/broadcom-bt-firmware`.
- The brcm80211 page of `wireless.docs.kernel.org`.
- The linux-libre deblob log.

Every `[measured]` claim comes from the reference unit.
