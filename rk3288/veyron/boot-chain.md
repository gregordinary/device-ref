# Veyron boot chain

This document describes how the Veyron Chromebooks, the RK3288 boards that mainline
groups as `google,veyron`, boot an operating system. It is written at family level. It
covers the firmware in SPI NOR, the signed kernel partition and the payload limits the
firmware enforces. It also covers what an image has to carry to meet them. Measurements come
from one ASUS C201 (veyron-speedy), the reference unit.

Veyron boots differently from the Rockchip boards that boot through rkbin and u-boot. It has no
maskrom loader, no idbloader, no `u-boot.itb` and no raw gap. The boot artifact is a
vboot-signed FIT image written raw into a ChromeOS kernel partition, and GPT attribute
bits select it.

## The stock boot chain

The chain runs from the mask ROM through coreboot to depthcharge, which loads the signed
FIT [LB][CROS]:

```
RK3288 mask ROM
  -> coreboot bootblock          (SPI NOR, 4 MiB GD25Q32BSIG, read-only region)
  -> verstage / vboot            (selects RW slot A|B or recovery, GBB holds the keys)
  -> romstage -> ramstage
  -> depthcharge                 (coreboot's payload, the ChromeOS bootloader)
       - EC software sync against the cros_ec
       - scans the GPT of each boot medium for ChromeOS kernel partitions
       - picks by attribute bits, verifies the vboot signature
  -> the signed FIT              (zImage + DTB[s] [+ initrd]), booted directly
```

The bootloader lives in SPI flash. Nothing an operating-system image writes to its disk is
a bootloader, and the firmware never reads `/boot`. On a board that boots through rkbin and
u-boot, a raw gap ahead of the rootfs carries idbloader and u-boot. Veyron has no such gap.

The kernel partition is a raw partition holding a vboot blob, not a filesystem. The blob is
a keyblock and a preamble (the signature header), followed by the FIT payload and the
kernel command line.

## The kernel partition

The firmware finds the kernel partition by its type GUID and chooses among candidates by
GPT attribute bits [CROS]:

| Property | Value | Source |
|---|---|---|
| Type GUID | `FE3A2A5D-4F32-41A7-B725-ACCC3285A309` | [CROS] |
| Rootfs type GUID (ChromeOS) | `3CB8E202-3B7E-47DD-8A3C-7FF2A13CFCEC` | [CROS] |
| Selection | GPT attribute bits 48-63 | [CROS] |
| Priority | Bits 48-51. 15 is the highest, and 0 means never boot. | [CROS] |
| Tries | Bits 52-55, decremented on each failed attempt | [CROS] |
| Successful | Bit 56. When it is set, tries are not decremented. | [CROS] |

`cgpt add -P <priority> -T <tries> -S <successful>` is the conventional way to set the
attribute bits. They are plain bits in a standard GPT partition entry's 64-bit attribute
field. Any GPT writer that exposes the partition type GUID and the raw attribute field can
therefore create the kernel partition. No `cgpt` or other ChromeOS host tool is needed.

The firmware never reads a partition's number or start sector. It scans for the type GUID
and orders the matches by priority. Start and size are free choices, and the projects
disagree on them: see [Where the sources disagree](#where-the-sources-disagree).

## The signed FIT payload

The payload is built in two steps. `mkimage -f auto` can generate the FIT from its parts
with no hand-written `.its`, as `depthcharge-tools` does. Arch Linux ARM and PrawnOS write a
`kernel.its` instead [ALARM][PRAWNOS]. The vboot step is common to all
[ALARM][PMOS][HEX]:

```sh
# 1. FIT: zImage + one or more DTBs (+ optionally an initrd)
mkimage -D "-I dts -O dtb -p 2048" -f auto \
        -A arm -O linux -T kernel -C none -a 0 \
        -d zImage $(printf ' -b %s' arch/arm/boot/dts/rockchip/rk3288-veyron-*.dtb) \
        kernel.itb
# mkimage emits type "kernel" for -f auto. It must be "kernel_noload", which fdtput fixes.

# 2. vboot: sign the FIT with the *public* ChromeOS developer keys
dd if=/dev/zero of=bootloader.bin bs=512 count=1        # 512 zero bytes, required, unused on ARM
vbutil_kernel --pack vmlinux.kpart --version 1 --arch arm \
  --vmlinuz  kernel.itb \
  --config   cmdline \
  --bootloader bootloader.bin \
  --keyblock    /usr/share/vboot/devkeys/kernel.keyblock \
  --signprivate /usr/share/vboot/devkeys/kernel_data_key.vbprivk
```

The signature is a format requirement, not a security boundary. The keys are the public
ChromeOS developer keys shipped in `vboot-kernel-utils`. The device accepts them only
because `crossystem dev_boot_signed_only=0` is set. Signing proves nothing. It only
makes the blob parseable by depthcharge.

`vbutil_kernel` and `futility` are interchangeable front ends (`futility vbutil_kernel ...`),
and Debian ships both in `vboot-kernel-utils` [DEB]. The FIT type that `mkimage -f auto`
writes is wrong for depthcharge: see
[the FIT image type](corrections-and-gotchas.md#mkimage--f-auto-emits-the-wrong-fit-image-type).

## The kernel command line

The kernel command line is part of the signed blob and cannot change without re-signing.
Depthcharge substitutes `%U` with the PARTUUID of the kernel partition it booted from.
`mkdepthcharge` prepends `kern_guid=%U` to every command line it signs, and the firmware
prepends `cros_secure` [DCT].

The idiom `root=PARTUUID=%U/PARTNROFF=1` names the partition one index after the kernel
partition as root. Arch Linux ARM and PrawnOS use it so that one image boots unchanged from
SD, USB or eMMC [ALARM][PRAWNOS]. It does not work on a Debian image. The kernel,
initramfs-tools and `depthchargectl` each reject it, as
[PARTNROFF does not work under initramfs-tools](corrections-and-gotchas.md#partnroff-does-not-work-under-initramfs-tools)
sets out.

Root on the rootfs partition's own PARTUUID instead, as `root=PARTUUID=<rootfs uuid>`. It
needs no custom code at any layer, and depthcharge-tools derives it from `/etc/fstab` by
itself. It still boots from SD, USB or eMMC, because a PARTUUID travels with its partition.
Images that root on a concrete PARTUUID boot from SD, USB and internal eMMC with no custom
initramfs script and no patched tooling [measured]. Their signed command line reads:

```
kern_guid=%U console=tty1 rootwait ro sysctl.kernel.panic=30 loglevel=4 root=PARTUUID=<rootfs uuid>
```

The `%U` that survives is `kern_guid=%U`. `mkdepthcharge` prepends it after the
configuration round trip in `depthchargectl`, so it never meets
[the percent-sign defect](corrections-and-gotchas.md#depthchargectl-cannot-carry-a-percent-sign).
The panic timeout is spelled as a sysctl so that initramfs-tools still runs its panic hooks:
see [The panic timeout](white-screen.md#the-panic-timeout).

The device gives no diagnostic for a wrong command line: it never finds root. The blob
stores the command line as plain text, so read it back out of the signed `.img` with
`grep -a` and assert on it.

## The FIT ramdisk

Stock Veyron depthcharge ignores the FIT's ramdisk node: `loads-fit-ramdisk = False` for
`boards/arm/rk3288/veyron` [DCT]. A hand-rolled FIT with a correct ramdisk node therefore
boots with no initrd and panics on rootfs. Nothing in the failure points at the cause.

depthcharge-tools works around it. When `loads_fit_ramdisk` is false, it sets
`patch_dtbs = True` and computes where the initramfs lands from
`image-start-address = 0x2000000`. It then writes `/chosen/linux,initrd-start` and
`/chosen/linux,initrd-end` into every DTB in the FIT with `fdtput` [DCT].

### Libreboot r20160818

The libreboot r20160818 depthcharge loads the FIT ramdisk node. Its `src/boot/fit.c`
carries `fit_add_ramdisk()`, which writes `/chosen/linux,initrd-start` and
`linux,initrd-end` itself [LBSRC]. The firmware parses the FIT in place at
`KERNEL_START = 0x2000000`. Its computed addresses are therefore byte-identical to the
ones `mkdepthcharge` pre-patches for the stock profile.

A pre-patched `/chosen` is redundant on this firmware rather than harmful, because
`update_chosen()` only adds `bootargs` and never rewrites the node. A stock-profile kpart
therefore boots identically on both firmwares [LBSRC]. Debian forky and trixie images boot
the reference unit to a login prompt with working Wi-Fi under both board profiles
[measured, 2026-07-14]. The boot medium was a USB stick, selected with Ctrl+U at the splash.

### The libreboot memory layout

The layout below is read from the r20160818 source [LBSRC]. DRAM starts at physical `0x0`.

| Address | Size | Content |
|---|---|---|
| `0x00200000` | 128 KiB | coreboot ramstage |
| `0x01000000` | 1 MiB | coreboot post-RAM CBFS cache |
| `0x02000000` | 32 MiB | The kernel buffer (`KERNEL_START`), which receives the whole signed FIT as read from the kernel partition. `KERNEL_SIZE` defaults to `0x2000000`, and the board does not override it, so the buffer spans `0x2000000-0x4000000`. |
| `0x06400000` | As flattened | The chosen DTB after the firmware's edits (`KERNEL_FIT_FDT_ADDR`), entered in the DTB's own reserve map |
| `0x10000000` | 2 MiB | coreboot DMA-coherent region |
| `0x10800000` | 8 MiB | The framebuffer holding the developer-mode splash |
| `0x43000000` | 16 MiB heap | depthcharge itself (`BASE_ADDRESS`) |

Three properties follow from the code:

- Nothing is copied out of the buffer. `boot_arm_linux()` jumps to the kernel image node's
  data where it lies inside the FIT. The ARM32 path refuses a compressed kernel node, so the
  FIT carries a self-decompressing zImage with `compression = none`. The ramdisk is handed
  over in place too: `fit_add_ramdisk()` writes its address inside the buffer into `/chosen`.
- `fit_load()` calls `fit_add_ramdisk()` for any chosen configuration with a ramdisk node.
  The firmware therefore rewrites `/chosen` on a stock-profile FIT as well, with the
  addresses `mkdepthcharge` already patched in.
- The decompressed kernel must end below `0x2000000` [inferred]. The zImage decompresses to
  the bottom of RAM (`AUTO_ZRELADDR`, so `0x8000`). A kernel whose image and BSS reach the
  buffer overwrites the FIT it runs from and the ramdisk inside it. A mainline 7.2.8 kernel
  built for this family ends at physical `0x14fab04` (`_end` in its `System.map`), well clear.

### Kernels without an initrd

Two projects avoid the initrd entirely. Arch Linux ARM passes `noinitrd` and builds mmc and
ext4 into the kernel [ALARM]. PrawnOS embeds the initramfs inside the kernel through
`CONFIG_INITRAMFS_SOURCE` [PRAWNOS].

A stock Debian armhf kernel can do neither. Its `CONFIG_MMC_DW=m`, `CONFIG_MMC_DW_ROCKCHIP=m`
and `CONFIG_EXT4_FS=m` mean it requires an initramfs to mount root, while `MMC_BLOCK` is
`=y` (checked in `linux-base-7.1.3+deb14-armmp`) [DEB].

## The payload budget

The firmware enforces a ceiling on the signed payload, and the kernel and the initramfs
share it:

| Firmware | `image-max-size` | `loads-fit-ramdisk` | Source |
|---|---|---|---|
| Stock Veyron | 16 MiB | False | [DCT] |
| speedy with libreboot r20160818 | 32 MiB | True | [DCT]. Both values match the depthcharge source (`CONFIG_KERNEL_SIZE = 0x2000000` by default, `fit_add_ramdisk()` present) [LBSRC] and the reference unit [measured]. |

Debian's armhf `vmlinuz` is 6,832,640 bytes, 41% of the stock budget before an initrd
exists [DEB]. A `MODULES=most` initramfs cannot fit, so the initramfs takes `MODULES=list`
with an explicit module set. `MODULES=dep` builds the wrong initramfs under a cross-build:
see [MODULES=dep builds the wrong initramfs](corrections-and-gotchas.md#modulesdep-builds-the-wrong-initramfs).

`depthchargectl build` enforces the ceiling itself and fails with a distinct
`InitramfsSizeTooBigError`. An overflow is therefore a loud build error, never a silent
truncation [DCT].

### Compression and the display stack

Two choices that are ordinarily free follow from the ceiling alone: the initramfs
compressor, and whether the initramfs carries the display stack.

| Payload ceiling | Initramfs compressor | Display stack in the initramfs |
|---|---|---|
| 16 MiB | xz | No |
| 32 MiB | zstd | Yes |

At 16 MiB the budget goes on fitting at all. A zstd initramfs measures 9,752,964 bytes. With
the `vmlinuz` above it totals 16,635,856 bytes against a 16,777,216-byte ceiling, a margin of
141 KB before the FIT and vboot headers. xz recovers about 2 MB, which is reserve for a
kernel that grows at every update rather than space for drivers. xz is also the slowest
common compressor to decompress, and on this SoC that costs seconds of blank panel per boot.

At 32 MiB neither constraint binds. zstd decompresses several times faster for about 2 MB
more payload. The display stack fits beside it: `rockchipdrm`, `phy-rockchip-dp`,
`panel-simple`, `pwm_bl` and `pwm-rockchip`. The panel can then light during the initramfs
instead of after the rootfs mounts.

The eDP PHY module is the one to miss, because no symbol dependency pulls it in: see
[The display stack in the initramfs](white-screen.md#the-display-stack-in-the-initramfs).
The 32 MiB combination of zstd and the display stack boots the reference unit from internal
eMMC [measured, 7.2.8].

The kernel partition can be larger than the payload ceiling. The partition is 16 MiB in
[ALARM] and [PMOS], and 32 MiB in [PRAWNOS] and [HEX]. Only the payload ceiling is enforced
by the firmware.

## Libreboot

"Libreboot on a C201" names two different firmware flows. The flow that boots keeps
depthcharge, and the flow with a u-boot payload has no working video.

### Flow A: libreboot 20160907

The libreboot 20160907 C201 port replaces coreboot's blobs and Google branding and keeps
depthcharge as the payload. It preserves the whole vboot model (RO and RW slots, GBB,
recovery) and no longer requires Google-signed kernels. GNU Boot still ships this flow and
describes the device as using depthcharge, "a very old [version], from 2016" [LB].

The signed-kernel-partition flow is unchanged. An image still needs a ChromeOS kernel
partition (`cgpt add -t kernel`), a `vbutil_kernel --pack` blob and Ctrl+U at the splash.
Libreboot buys deblobbed firmware and freedom from Google's keys, not an extlinux `/boot`.

The r20160818 depthcharge builds `CONFIG_DRIVER_VIDEO_ROCKCHIP=y` and
`CONFIG_DRIVER_STORAGE_MMC_ROCKCHIP=y`, so the payload has working display and SD [LBSRC].
Reports of broken video on a libreboot C201 concern the Flow B u-boot payload.
depthcharge-tools models this firmware as its own board profile, with a 32 MiB payload
budget and working FIT ramdisks [DCT].

### Flow B: libreboot 20221214 and 20230319

Flow B replaces depthcharge with a U-Boot-as-coreboot-payload port by Alper Nebi Yasak. The
port offers standard u-boot boot: UEFI, `extlinux.conf` or a prompt [LB]. On this board it
is unusable:

- Libreboot states that "This board is known to have non-functioning video init ... It is
  as yet unsolved" (19 February 2023). The ROM images are deleted from both releases.
- Upstream libreboot's Chromebook page lists only `nyan` and `gru` as officially
  supported, with `veyron` "temporarily removed". Its `c201.html` and `chromebooks.html`
  contradict each other on the current support status.
- The last known-good u-boot on Veyron is v2021.01, and two Rockchip video-reset commits
  break it afterwards. [HEX] pins exactly `v2021.01-cbr` for speedy, which corroborates this
  independently.

Do not plan around Flow B. The extlinux path on libreboot is a trap.

### Identifying the flow

Both flows are called libreboot, and they differ at the firmware screen. Flow A shows a
depthcharge-style screen and honors the Ctrl+U and Ctrl+D shortcuts. Flow B drops to a
u-boot prompt, which has no working video on this board. A unit that reaches depthcharge runs Flow A, and
the depthcharge boot method applies to it unchanged.

The reference unit runs Flow A [measured]. Ctrl+U boots external media on it, and it
installs to internal eMMC from USB and from SD. Its libreboot firmware retains depthcharge,
so it bypasses nothing in this document.

### The libreboot board profile

depthcharge-tools gives the libreboot firmware its own board profile, `speedy/libreboot`,
with a 32 MiB budget and working FIT ramdisks. It sets `hwid-match = None` on that profile,
so the profile cannot be auto-detected [DCT]. A running `depthchargectl` on a libreboot
unit resolves to the plain `speedy` profile and applies the stock constraints: 16 MiB and a
DTB-patched initrd.

That outcome is the conservative one. A stock-profile image boots on stock firmware and on
libreboot, and the reverse does not hold. The 32 MiB profile is an opt-in
(`--board speedy-libreboot`) for an image that needs the larger budget.

## Boot architectures

Every project that targets this family picks one of three mutually exclusive boot
architectures. Architecture A is the one that works on stock and libreboot 20160907
firmware with upstream, packaged tooling. It is upstream-maintained, needs no patched
bootloader, and has its on-device update path packaged in Debian.

### A: signed kernel in the kernel partition

The FIT carries the kernel, and the signed blob is the kernel. No bootloader exists
anywhere on the disk. The projects using it are postmarketOS, Arch Linux ARM, PrawnOS,
crawfishos, the Debian wiki recipe and the 2025 Debian Trixie guide.

A kernel update must re-sign the payload and rewrite the partition. Debian packages the
tooling for that in `depthcharge-tools`, whose kernel and initramfs hooks do exactly this [DEB].
See [Kernel updates must re-sign the payload](corrections-and-gotchas.md#kernel-updates-must-re-sign-the-payload).
The constraints are the 16 MiB budget on stock firmware, the DTB-patched initrd and a
command line fixed at signing time.

### B: chainloaded u-boot in the kernel partition

The kpart carries a signed u-boot, which reads `extlinux.conf` from an ext4 `/boot` and
boots a conventional kernel. It looks attractive because it gives a conventional `/boot`.
The projects using it are velvet-os/imagebuilder [HEX], ALARM's `uboot-veyron` and
atopuzov.

Upstream u-boot cannot build it. `chromebook_speedy_defconfig` builds
`u-boot-rockchip-spi.bin` for the SPI flash (`u-boot,spl-boot-order = &spi_flash`,
`# CONFIG_SPL_MMC is not set`), which is architecture C rather than a payload [UBOOT]. The
chainload builds are u-boot 2018.07 and 2021.01 with out-of-tree patches
(`chainloading-v2021.01.patch`), five years stale. ALARM never shipped a `uboot-speedy`.

Architecture B does not avoid vboot either. It still signs a kpart, and it adds an old,
patched u-boot to the same vboot machinery. It is strictly more work than A.

### C: u-boot in SPI flash in place of the firmware

`chromebook_speedy_defconfig` sets `CONFIG_ROCKCHIP_SPI_IMAGE=y`, which produces
`u-boot-rockchip-spi.bin` for SPI offset 0. Simon Glass maintains it upstream [UBOOT]. It is
a firmware replacement, not a payload. It requires defeating the write-protect screw and
destroys the stock recovery path, so it is a firmware project rather than an
operating-system image concern.

## Developer-mode prerequisites

Each unit needs firmware-side preparation once, from ChromeOS or any booted system. Enter
developer mode first. The key sequence differs by board, as
[Entering developer mode](family-boards.md#entering-developer-mode) describes, and on the
laptops the transition wipes the device in about 15-20 minutes. Then set two flags on stock
firmware [CROS][ALARM][DEBWIKI]:

```sh
crossystem dev_boot_usb=1          # allow external media (covers SD as well as USB)
crossystem dev_boot_signed_only=0  # allow dev-key-signed kernels (required for a self-signed payload)
```

At the developer-mode splash, Ctrl+U boots external media and Ctrl+D boots internal
storage. The splash times out after 30 s. Setting `dev_boot_signed_only=1` with a custom
kernel installed bricks the boot [CROS].

On libreboot r20160818 the `crossystem` fields have no backing store, so the commands fail.
Ctrl+U boots external media without them: see
[crossystem fails on libreboot](corrections-and-gotchas.md#crossystem-fails-on-libreboot).
The write-protect screw next to the SPI chip matters only for reflashing firmware, and
architecture A never needs it.

## Where the sources disagree

These positions conflict. Each resolution below follows from the firmware or from a
measurement, not from an average:

| Question | Positions | Resolution |
|---|---|---|
| Kernel-partition start | 8192 [ALARM][HEX], 24576 [PMOS], 20480 (PrawnOS eMMC partition map) | Free, because the firmware scans by type GUID and not by location. On eMMC the first partition must start clear of the reserved 8 MiB region at sector 64, which 8192 does not. 20480 is where the factory GPT puts KERN-A [PR260]. See [The factory eMMC layout](#the-factory-emmc-layout). |
| Kernel-partition size | 16 MiB [ALARM][PMOS], 32 MiB [PRAWNOS][HEX] | Partition size and payload ceiling are different quantities. The firmware enforces a 16 MiB payload on stock firmware and 32 MiB on libreboot [DCT]. The factory KERN-A is 16 MiB [PR260]. |
| Initrd | In the FIT and DTB-patched [PMOS][DCT], none with `noinitrd` [ALARM], embedded in the kernel [PRAWNOS] | Debian's modular kernel requires an initrd, which on stock firmware means the DTB-patching path [DCT]. See [The FIT ramdisk](#the-fit-ramdisk). |
| USB boot | The primary install path [PRAWNOS], against "not at all from usb devices" and picky about SD cards [HEX] | Unit-dependent. The reference unit boots from USB and SD [measured]. See [USB boot is unit-dependent](corrections-and-gotchas.md#usb-boot-is-unit-dependent). |
| eMMC device node | `mmcblk0` [DEBWIKI], "varies by kernel version (mmcblk0 or mmcblk2)" [GENTOO], `mmcblk2` [PRAWNOS] | Never hardcode it. A mainline kernel pins the eMMC to `mmcblk0`, and `root=PARTUUID=<rootfs uuid>` is independent of the medium. See [The eMMC device node](family-boards.md#the-emmc-device-node). |

## The factory eMMC layout

Stock ChromeOS units carry this eMMC partition table, from `fdisk -l` on stock
units [PR260]:

```
/dev/mmcblkXp11       64    16447    16384    8M unknown          <- RO firmware area
/dev/mmcblkXp2     20480    53247    32768   16M ChromeOS kernel   <- KERN-A
/dev/mmcblkXp4     53248    86015    32768   16M ChromeOS kernel   <- KERN-B
/dev/mmcblkXp12   249856   282623    32768   16M EFI System
/dev/mmcblkXp5    282624  4476927  4194304    2G ChromeOS root fs  <- ROOT-B
/dev/mmcblkXp3   4476928  8671231  4194304    2G ChromeOS root fs  <- ROOT-A
/dev/mmcblkXp1   8671232 30744575 22073344 10.5G Microsoft basic   <- STATE
```

The factory kernel partition is 16 MiB, which corroborates 16 MiB as the natural size on
this hardware. The 8 MiB region at sector 64 is reserved, so the first partition must start
well clear of it. A start at sector 24576 (12 MiB) does, and one at 8192 (4 MiB) does not.
PrawnOS reports that "all C201s have these bad blocks at the beginning of the emmc"
[reported].

### The IGNOREME primary GPT

The factory table's primary GPT is corrupt by design. ChromeOS marks it `IGNOREME` and keeps
the real table in the secondary. This is a property of the factory partition table, not of
the eMMC or the firmware. It affects only an installer that preserves the factory GPT, and
such an installer needs one of two fixes, both in use:

- Patch the kernel [PMOS][PRAWNOS]. `veyron.ignore-gpt-ignoreme.patch` adds
  `GPT_HEADER_SIGNATURE_IGNORED` (`'IGNOREME'`) to `block/partitions/efi.c`, ported from
  ChromeOS 3.14.
- Never write the primary GPT [HEX]. The installer writes the backup GPT directly at
  `seek = sectors-34` and patches u-boot to suppress the EFI errors.

A whole-disk image needs neither. It carries its own GPT, primary and secondary both valid,
and writing it to the eMMC replaces the factory table outright. A Debian kernel then reads
it like any other disk. On the reference unit, a whole-disk image replaces the factory GPT
and boots with a readable primary GPT header [measured].

### Geometry

eMMC geometry is as free as on removable media. A postmarketOS install on the reference unit
writes a fresh GPT with its kernel partition at sector 24576. It boots with a readable
primary GPT header [measured]. What the firmware reads is the type GUID and the attribute
bits, which that install sets to `priority=10 tries=5 successful=1`.

No board in the family needs a kernel patch to boot or to install this way [source]. The
reference unit boots and installs without one [measured]. Debian's own `linux-image-armmp` therefore serves
unmodified. The one per-board risk is the Chromebit, where no
eMMC install is publicly confirmed: see [Open risks](family-boards.md#open-risks).

## Integrating with depthcharge-tools

An image for this family lets depthcharge-tools, installed in the rootfs, generate and
install the signed payload. This section collects what an image builder needs for that.

### Differences from rkbin and u-boot boards

The depthcharge boot method differs from the rkbin and u-boot one throughout the image:

| Area | Rockchip boards that boot through rkbin and u-boot | Veyron |
|---|---|---|
| Bootloader build | u-boot -> `idbloader.img` and `u-boot.itb` | None, because the firmware is in SPI flash |
| Payload | Raw-gap writes at fixed offsets | A vboot-signed FIT in a GPT partition |
| GPT | The rootfs, after the raw gap | Two ChromeOS kernel partitions and the rootfs |
| rkbin blobs | ATF and TPL from rkbin | None. RK3288 has no rkbin chain. |
| Boot wiring in the rootfs | `extlinux.conf` and its kernel hook | The `depthcharge-tools` package and its kernel hooks |
| Kernel updates | Regenerate `extlinux.conf` | Re-sign and rewrite a kernel slot |
| Root resolution | `root=UUID=` | `root=PARTUUID=`, never `PARTNROFF` |
| Initramfs module set | `MODULES=most` fits | `MODULES=list`, because of the payload ceiling |

### The A/B kernel slots

An image carries two kernel slots, and depthcharge-tools implements the complete ChromeOS
A/B protocol over them. KERN-A carries the signed payload. KERN-B sits behind it, empty, at
priority 0 ("never boot"). `depthchargectl write` targets the slot the system is not running
from and blesses it with one try. `depthcharge-tools.service` marks the booted slot
successful once `boot-complete.target` is reached.

An upgrade that does not come up therefore spends its one try. The firmware then falls back
to the slot that still holds the old kernel. A single slot defeats the protocol. The hook
then overwrites the running kernel in place, and a bad kernel leaves external media as the
only way in.

The slot size follows the board profile. With the first slot at 12 MiB, clear of the
reserved region, two 16 MiB slots put the rootfs at 44 MiB on stock firmware. Two 32 MiB
slots put it at 76 MiB on the libreboot profile. The per-slot PARTUUIDs must differ,
because `kern_guid=` is how the running system knows which slot it can overwrite.

The fallback protects upgrades, not the first install. A first slot written at priority 10
with its successful bit set never has its tries decremented (see
[The kernel partition](#the-kernel-partition)). An image whose first kernel does not come up
is then retried on every power-on, with no other slot to fall back to. Only external media
ends that boot loop.

The `root=` signed into the kpart names the rootfs PARTUUID for the life of the install. A
first-boot step that grows the root partition must keep that PARTUUID. Every medium written
from one image shares it: see
[Two media written from one image share a PARTUUID](corrections-and-gotchas.md#two-media-written-from-one-image-share-a-partuuid).

### Upgrades

A `.kpart` is not an upgrade artifact. The signed FIT carries the kernel, the DTB and the
initramfs, but not `/lib/modules/<release>`. The upgrade unit is the kernel `.deb`, whose
maintainer script fires the same hook whether the package came from `apt` or from `dpkg -i`.

The kernel flavor changes none of this. A locally compiled `linux-image-*.deb` fires the
hook from its maintainer script and takes the same A/B path as Debian's own package. Only
its delivery differs.

### Building in a chroot

No ChromeOS tooling runs on the build host. depthcharge-tools runs inside the rootfs and
hooks the kernel's own `postinst.d`, so the package can generate the signed blob while the
rootfs is customized. That mirrors how kernel hooks generate `extlinux.conf` and the
initrd on an rkbin board. The disk-assembly step then only places that blob into a kernel partition, and
`futility`, `mkimage` and `cgpt` never run on the host.

Three details apply in a chroot:

- The build must call `depthchargectl build` explicitly. The package's kernel hook runs
  `depthchargectl write`, which needs a target partition. In a chroot it finds none and
  exits 0, so the hook alone produces nothing. The output lands in `images-dir`, by default
  `/boot/depthcharge/<release>.img`.
- Disable the system hooks for the duration of the build, and re-enable them in the
  finished image. `depthchargectl write` searches the build host's disks: see
  [The kernel postinst hook writes to any disk it finds](corrections-and-gotchas.md#the-kernel-postinst-hook-writes-to-any-disk-it-finds).
- Keep every `%` out of what `depthchargectl` reads, and put the command line in
  `/etc/kernel/cmdline` rather than in the depthcharge configuration. The reason is
  [the percent-sign defect](corrections-and-gotchas.md#depthchargectl-cannot-carry-a-percent-sign).

### DTB paths

Debian's armhf DTB path differs from arm64's. The forky armhf kernel package is split into
`linux-image`, `linux-binary`, `linux-modules` and `linux-base`. The DTBs ship flat in
`linux-base` at `/usr/lib/modules/<release>/dtb/`, with no `rockchip/` subdirectory and none
of arm64's vendor subdirectories [DEB]. A kernel built with `bindeb-pkg` puts its DTBs under
`/usr/lib/linux-image-<release>/`.

`depthchargectl` finds both. It globs `usr/lib/linux-image-*` first and
`usr/lib/modules/*/dtb` second, so either kernel flavor works unchanged [DCT]. It selects the
DTB by matching the board's `dt-compatible` against each file's `/compatible`, which on the
C201 resolves to exactly `rk3288-veyron-speedy.dtb`. Anything hand-rolled must not assume
one layout.

## Sources

- [ALARM]: the Arch Linux ARM `linux-armv7` PKGBUILD.
- [CROS]: the Chromium OS disk-format and developer-mode documentation.
- [DCT]: `depthcharge-tools` 0.6.2-5 as Debian forky packages it, `boards.ini` and
  `mkdepthcharge.py`.
- [DEB]: Debian forky packages, unpacked.
- [DEBWIKI]: the Debian wiki.
- [GENTOO]: the Gentoo wiki.
- [HEX]: the hexdump0815 and velvet-os imagebuilder.
- [LB]: the libreboot and GNU Boot C201 documentation.
- [LBSRC]: the libreboot r20160818 source. It covers depthcharge's
  `board/veyron_speedy/defconfig`, `src/image/Kconfig`, `src/boot/fit.c` and
  `src/arch/arm/fit.c`, and coreboot's `src/soc/rockchip/rk3288/include/soc/memlayout.ld` at
  the revision the build pins for `veyron_speedy` (`1bf5e640`).
- [PMOS]: postmarketOS `device-google-veyron`.
- [PR260]: [PrawnOS#260](https://github.com/SolidEva/PrawnOS/pull/260), with `fdisk -l` from
  stock ChromeOS units.
- [PRAWNOS]: PrawnOS (`SolidEva/PrawnOS`, the renamed `SolidHal/PrawnOS`).
- [UBOOT]: u-boot master, `configs/`, `board/google/veyron/` and `arch/arm/dts/*-u-boot.dtsi`.

A `[measured]` result comes from the reference unit, an ASUS C201 (veyron-speedy) running
libreboot r20160818 with its depthcharge payload. Ctrl+U works on it, it boots from USB and
SD, and it installs to its internal eMMC.
