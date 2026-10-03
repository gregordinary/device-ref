# Veyron corrections and integration gotchas

This document covers the Veyron family of RK3288 Chromebooks at family level, with the ASUS
C201 as the measured board. It lists claims about the family that are widely repeated and
wrong, and integration traps that are true but are not hardware facts. Each correction
states the claim, the fact and the evidence. The
[C201 hardware reference](asus-c201/hardware-reference.md) states what the board is, and the
[boot chain](boot-chain.md) states how the family boots.

The family carries a large body of stale documentation. A decade of guides exists, most
written against the ChromeOS 3.14 kernel or jessie-era Debian, and many still rank highly.

## Corrected claims

### Libreboot keeps depthcharge

A widely repeated claim holds that flashing libreboot gives the C201 a conventional
bootloader, so a plain `/boot` with extlinux replaces the signed kernel partition.
Libreboot 20160907, the release in wide use and the one GNU Boot still ships, keeps
depthcharge as the coreboot payload [LB]. It deblobs the firmware and stops enforcing
Google's keys, and it leaves the boot contract unchanged. An image still signs a FIT with
`vbutil_kernel`, writes it into a ChromeOS kernel partition and boots through Ctrl+U.

The u-boot-payload releases, 20221214 and 20230319, do offer extlinux. On this board their
video init does not work ("as yet unsolved", February 2023), and their ROM images are
deleted from both releases. Upstream libreboot's Chromebook page lists only `nyan` and `gru`
as supported. The boot method on libreboot is therefore the signed kernel partition, not an
extlinux `/boot`. [Libreboot](boot-chain.md#libreboot) describes both flows and how to tell
them apart.

### The C201 radio is Broadcom

A claim repeated in several places gives the C201 a Marvell `mwifiex` SD8897 radio. It is a
natural guess, because other Veyron boards carry one. The C201 radio is a Broadcom BCM4354
on SDIO, driven by `brcmfmac`. The device tree settles it: `rk3288-veyron-speedy.dts`
includes `rk3288-veyron-broadcom-bluetooth.dtsi`, and the Bluetooth companion is
`brcm,bcm43540-bt` [DTS]. [The in-tree boards](family-boards.md#the-in-tree-boards) gives the
radio of every board.

The difference is not a small one for firmware. The Marvell firmware is in Debian
(`firmware-libertas`). The Broadcom board NVRAM is in neither Debian nor linux-firmware and
must be vendored: see [The installation path](wifi-firmware.md#the-installation-path).
Getting the radio backwards means planning for the easy case and meeting the hard one.

### HDMI and hardware video decode work

The Debian wiki, written in the jessie and bookworm era, rates hardware video decoding and
wireless as not working ("Hardware video decoding: X-(", "Wireless: X-("). Debian forky
armhf builds `VIDEO_HANTRO=m`, `VIDEO_HANTRO_ROCKCHIP=y` and `VIDEO_ROCKCHIP_VDEC=m` [DEB].
The RK3288 has a mainline rkvdec HEVC variant, driver and DT node, from v6.19 and v7.0
(September 2025). Hantro gives MPEG-2, VP8 and H.264 decode plus JPEG encode.
[Video decoders](../video-reference.md#video-decoders) has the detail.

The board has a micro-HDMI port, and Debian rates HDMI as working. Stories of no video on
the C201 concern the libreboot u-boot payload's broken video init, a firmware defect rather
than a missing port.

HDMI does 4K30 on a stock Debian kernel with nothing to configure. Reports that 4K needs
out-of-tree patches are stale, because the Armbian and Kwiboo stack is in mainline. Every
kernel Debian ships carries both `drm/rockchip: dw_hdmi: Add support for 4k@30 resolution`
(`83b61f817f43`, February 2023) and `drm/rockchip: vop: limit maximum resolution to hardware
capabilities` (`8e140cb60270`).

The 4K60 mode is a hardware ceiling that no patch reaches.
[HDMI output limits](../display-reference.md#hdmi-output-limits) gives the driver constants.
[The two display controllers](../display-reference.md#the-two-display-controllers) gives the
VOPB and VOPL asymmetry that explains old "partial screen at 4K" reports.

### PrawnOS is frozen and crawfishos is archived

Every guide that still points at PrawnOS or crawfishos implies that both are live bases.
PrawnOS (`SolidEva/PrawnOS`) is not archived, but its armhf target is frozen on kernel
5.15.6, and its recent work is all arm64 and RK3588. `austin987/crawfishos` is formally
archived (GitHub `archived: true`, last commit 2021-11-29). It pins armhf at kernel 5.9.12
deliberately, because "5.10.1 breaks booting on speedy/emmc for me."

The maintained projects are velvet-os/imagebuilder (hexdump0815's, on Debian trixie),
postmarketOS and Arch Linux ARM, and all three track a current mainline kernel. If a newer
kernel regresses eMMC boot, the crawfishos 5.10.1 note is the precedent to remember.

### The kernel-partition size is two numbers

Guides give the kernel-partition size as 16 MiB or as 32 MiB, and treat one of the two as
wrong. Both are real, and they measure different things. The partition size is whatever the partitioner
chooses: 16 MiB in Arch Linux ARM and postmarketOS, 32 MiB in PrawnOS and velvet-os. The
payload ceiling is enforced by the firmware: `image-max-size` is 16 MiB on stock Veyron and
32 MiB on libreboot [DCT]. See [The payload budget](boot-chain.md#the-payload-budget).

### Libreboot r20160818 loads FIT ramdisks

The depthcharge-tools board database's stock Veyron values, `loads-fit-ramdisk = False` and
`image-max-size = 16 MiB`, do not describe the libreboot r20160818 firmware. Its depthcharge
loads the FIT ramdisk node, computes the same addresses `mkdepthcharge` pre-patches, and has
a 32 MiB kernel buffer [LBSRC]. The `speedy-libreboot` profile is therefore correct for this
firmware. A stock `speedy` image (16 MiB, patched DTBs) is a strict subset that boots on both
stock and libreboot firmware. On the reference unit's libreboot firmware it boots exactly as
the source predicts [measured, 2026-07-14]. [The FIT ramdisk](boot-chain.md#the-fit-ramdisk) has the source-level
detail and the firmware's memory layout.

### Developer mode still requires a signature

Guides imply, by omission, that developer mode boots an unsigned kernel. Developer mode
relaxes the check to "looks like a correctly signed kernel image": any valid vboot
structure, whatever the key. A blob with no vboot keyblock and preamble does not boot.
`dev_boot_signed_only=1` tightens the check back to the official Google keys, and setting it
with a custom kernel installed bricks the boot [CROS].

The signing step is therefore unavoidable, as a format requirement. The keys are the public
ChromeOS developer keys that Debian ships in `vboot-kernel-utils`.
[The signed FIT payload](boot-chain.md#the-signed-fit-payload) gives the signing commands.

## Integration gotchas

### Stock depthcharge ignores a FIT ramdisk

This trap is the one most likely to cost a day. The stock Veyron board profile sets
`loads-fit-ramdisk = False` [DCT]. A FIT with a correct `ramdisk` node therefore boots with
no initrd at all and panics on rootfs. Nothing in the failure points at the cause.

depthcharge-tools patches the initramfs address into every DTB in the FIT instead
(`patch_dtbs = not loads_fit_ramdisk`). Its board database encodes this and the size budget,
so use it rather than a hand-rolled FIT. [The FIT ramdisk](boot-chain.md#the-fit-ramdisk)
gives the mechanism and the libreboot behavior.

### MODULES=dep builds the wrong initramfs

A stock Debian kernel requires an initramfs on this family. Debian's armhf `armmp`
configuration builds `CONFIG_MMC_DW`, `CONFIG_MMC_DW_ROCKCHIP` and `CONFIG_EXT4_FS` as
modules (see [Kernels without an initrd](boot-chain.md#kernels-without-an-initrd)). The
`noinitrd` shortcut that Arch Linux ARM takes, with those drivers built in, needs a custom
kconfig on Debian. The armhf `vmlinuz` is 6,832,640 bytes, so a `MODULES=most` initramfs
cannot fit the 16 MiB budget.

Use `MODULES=list`, not `MODULES=dep`. `dep` looks like the right answer and fails for a
reason unrelated to size. It guesses the module set from the building system's mounted
`/sys` and `/`, through `dep_add_modules()` in `hook-functions`. That function warns "MODULES
dep requires mounted sysfs on /sys".

In an armhf chroot on an x86_64 host, `dep` therefore resolves the host's root device. It
produces an initramfs carrying none of this board's drivers. `dep` is meaningful only for an
initramfs built on the target. An explicit list is deterministic, works in both places, and
is what postmarketOS uses for the same reason.

`MODULES=list` silently drops names it cannot resolve, and `mkinitramfs` only warns. Assert
the list against the real kernel at build time, for example with `lsinitramfs` on the built
initrd. On a board with no console, a typo is a black screen with no diagnostic. A name check
does not catch a missing device-tree dependency such as the RK808 regulator chain: see
[The PMIC driver chain](white-screen.md#the-pmic-driver-chain).

### PARTNROFF does not work under initramfs-tools

`root=PARTUUID=%U/PARTNROFF=1` is the command-line idiom the Veyron guides use, and it does
not work under an initramfs-tools initramfs. The boot dies in "Gave up waiting for root
device", with nothing pointing at the cause.

The kernel parses `PARTUUID=<uuid>/PARTNROFF=<n>` in `name_to_dev_t()`. That code runs only
on the no-initramfs path, where the kernel mounts root itself. With an initramfs,
initramfs-tools does its own root resolution, and version 0.151 has no PARTNROFF support at
all: the string does not occur in it. Its `resolve_device()` hands the whole `root=` value to
`blkid -l -t`, which cannot match the `/PARTNROFF=1` suffix.

Other projects avoid the failure. Arch Linux ARM passes `noinitrd` and builds mmc and ext4
in, so the kernel parses the idiom. The postmarketOS initramfs never sees the idiom. Its
command line on the reference unit is `... pmos_root_uuid=<uuid>`, which its own initramfs
resolves [measured]. Debian's modular armhf kernel can do neither.

#### Resolving the idiom in a boot script

Making the idiom work takes a `local-top` boot script that resolves it and hands the answer
back through `/conf/param.conf`. That file is initramfs-tools' supported way for a boot
script to change `ROOT`. `hook-functions` writes `[ -e /conf/param.conf ] && . /conf/param.conf`
into the `ORDER` file that `run_scripts` sources, and `local_top` runs before
`local_mount_root()` reads `ROOT`.

The script has to resolve the kernel partition by its PARTUUID and then walk sysfs for the
sibling at `partition + n`. Rewriting the device name as a string does not work, because
`mmcblk1p1 -> mmcblk1p2` and `sda1 -> sda2` follow different rules.

#### depthchargectl and the root

`depthchargectl` also strips the idiom. It resolves `root` before it reads the configured
command line. It breaks the cyclic dependency between the two by yielding an empty command
line first. It then falls back to the root in `/etc/fstab`, decides that the `%U` root
differs, and removes it:

```
Kernel cmdline has a different root 'PARTUUID=%U/PARTNROFF=1', removing it.
Appending 'root=PARTUUID=<the one from fstab>' to kernel cmdline.
```

The image still boots, because a concrete PARTUUID travels with its partition, but the `%U`
is silently gone. Passing `depthchargectl build --root 'PARTUUID=%U/PARTNROFF=1'` stops the
stripping, because `self.root` then matches the command line. The build then meets
[the percent-sign defect](#depthchargectl-cannot-carry-a-percent-sign).

`root` is a command-line option only. Exactly seven keys are readable from
`/etc/depthcharge-tools/config`: `board`, `images-dir`, `vboot-keyblock`,
`vboot-public-key`, `vboot-private-key`, `kernel-cmdline` and `zimage-initramfs-hack`. The
package's own `/etc/kernel/postinst.d/zz-depthcharge-tools` runs `depthchargectl write` with
no `--root`. On the device, the first kernel upgrade therefore re-signs the kpart with the
root from `/etc/fstab` and drops PARTNROFF. The machine keeps booting, and the medium
independence does not survive `apt upgrade`.

#### A concrete PARTUUID instead

The two candidate roots compare as follows:

| Property | `root=PARTUUID=%U/PARTNROFF=1` | `root=PARTUUID=<rootfs uuid>` |
|---|---|---|
| initramfs-tools support | None, so it needs a custom `local-top` script | Native |
| `depthchargectl` can express it | No, because it cannot carry a `%` at all | Yes |
| `depthchargectl build` | Strips it unless given `--root` | Derives it from `/etc/fstab` unprompted |
| Survives an on-device kernel upgrade | No, because the postinst hook drops it | Yes, because the hook regenerates exactly this |
| Boots from SD, USB and eMMC | Yes | Yes, because a PARTUUID travels with its partition |
| Two attached media with the same root PARTUUID | Tolerated, unless the kernel-slot GUIDs are duplicated too | Ambiguous |

PARTNROFF is rejected at five independent layers:

- The kernel parses it only on the no-initramfs path.
- initramfs-tools has no support for it.
- `depthchargectl` strips a root that differs from `/etc/fstab`.
- `depthchargectl` cannot carry a `%` in a command line at all.
- The postinst hook drops it on the next kernel upgrade.

Over a concrete PARTUUID it buys only tolerance of a duplicate root PARTUUID. It loses even
that where the kernel-slot GUIDs are duplicated too, as on two media written from one image.
The `%U` is then as ambiguous as the root it replaces. Duplicates are real: see
[Two media written from one image share a PARTUUID](#two-media-written-from-one-image-share-a-partuuid).
The concrete PARTUUID needs no custom code at any layer, and depthcharge-tools converges on
it by itself.

The idiom is near-universal in Veyron guides because Arch Linux ARM uses it. Arch Linux ARM
passes `noinitrd`, where the kernel parses the idiom in `name_to_dev_t()` and none of the
above applies. [The kernel command line](boot-chain.md#the-kernel-command-line) gives the
form that boots from SD, USB and eMMC.

### depthchargectl cannot carry a percent sign

`depthchargectl` in depthcharge-tools 0.6.2 cannot build any image whose command line
contains a `%`. This is an upstream defect, not a configuration mistake, and it is what
makes the PARTNROFF idiom unreachable in practice.

`depthchargectl` writes its computed configuration back into a `ConfigParser`, so that the
`depthchargectl config` subcommand can report autodetected values
(`depthchargectl/__init__.py:370`):

```python
self.config.update({
    ...
    'kernel-cmdline': " ".join(self.kernel_cmdline),
    ...
})
```

That parser is a plain `configparser.ConfigParser()`, so `BasicInterpolation` is on.
`BasicInterpolation.before_set()` rejects a raw `%` on write:

```
ValueError: invalid interpolation syntax in 'root=PARTUUID=%U/PARTNROFF=1' at position 14
```

The error fires wherever the command line comes from: the configuration file,
`/etc/kernel/cmdline`, or `--kernel-cmdline` on the command line. `config_options` is a
`@Group` that runs regardless. No escaping works, because `%%U` in the configuration file is
un-escaped to a raw `%U` on read and then rejected on write-back. The upstream fix is one
argument, `ConfigParser(..., interpolation=None)`, since a kernel command line is not a
template.

`root=PARTUUID=%U/PARTNROFF=1` is the example in the `--cmdline` documentation of
[`mkdepthcharge(1)`](https://manpages.debian.org/unstable/depthcharge-tools/mkdepthcharge.1.en.html)
itself.

`mkdepthcharge` is unaffected: it carries no configparser and takes `--cmdline` straight
from argv. The `%U` idiom works at that level and is unusable only through `depthchargectl`,
which is the layer Debian's kernel hooks call. Images rarely meet the defect by accident.
`mkdepthcharge` prepends `kern_guid=%U` after the configuration round trip in
`depthchargectl`, so every normal image has a `%U` in its final command line and still
builds.

Keep `%` out of the command line handed to `depthchargectl`. If its configuration has no
`kernel-cmdline` key, `depthchargectl` reads `/etc/kernel/cmdline`, a standard Debian
location, with a plain `read_text()` and `shlex.split()`. Put the command line there, let
`depthchargectl` append `root=` from `/etc/fstab`, and keep `board` in the configuration.

### The kernel postinst hook writes to any disk it finds

The depthcharge-tools hook `/etc/kernel/postinst.d/zz-depthcharge-tools` runs
`depthchargectl write`, which searches the available block devices for a ChromeOS kernel
partition. Inside a build chroot, `/proc` and `/sys` belong to the build host. On a typical
host the search finds nothing and exits 0. If the target SD card is plugged in during the
build, the hook finds it and writes to it.

Set `enable-system-hooks = False` for the duration of the build, and build the image
explicitly with `depthchargectl build`. Then set it back to `True`, so that the installed
system's kernel upgrades keep re-signing (see
[Kernel updates must re-sign the payload](#kernel-updates-must-re-sign-the-payload)).

### The initramfs needs the RK808 PMIC chain

An initramfs with `dw_mmc-rockchip` and `ext4` looks complete and hangs on a white screen.
On Veyron the SD slot's rails are RK808 LDOs, so `dw_mmc-rockchip` defers until those
regulators register, and Debian builds their providers as modules. Neither provider is a
symbol dependency of anything, so `MODULES=list` resolution cannot pull them in, and the
build-time name check passes. List `i2c-rk3x` and `rk808-regulator` explicitly, plus
`spi-rockchip` for the rescue-shell keyboard. `io-domain` takes its supplies from the same
PMIC, so this also unblocks it.

[The PMIC driver chain](white-screen.md#the-pmic-driver-chain) has the full chain, the
module list and the diagnosis without a serial console.

### Two media written from one image share a PARTUUID

A concrete PARTUUID travels with its partition, which also means it travels to every copy.
Every medium written from one image carries the same root PARTUUID and the same kernel-slot
GUIDs. An image builder that derives these identifiers reproducibly, rather than drawing
them per build, extends that to every image it builds for one configuration.

With two such media attached, the root is whichever partition the lookup returns.
initramfs-tools resolves `root=PARTUUID=` with `blkid -l -t PARTUUID=... -o device`
(`resolve_device()` in `scripts/functions`), which returns one device. `local_device_setup()`
in `scripts/local` then waits for that device to carry a filesystem and never moves on to the
twin.

A damaged twin therefore takes the boot down, whichever medium the firmware loaded the
kernel from. The boot waits 30 s for root, takes the panic path and reboots about a minute
after Ctrl+U, then repeats. Every driver is loaded, and the intended root sits on the other
medium.

On the reference unit, an eMMC root partition holding a whole-disk image, written into the
partition rather than onto the disk, produces this cycle [measured]. Every boot of an image
with the same root PARTUUID, from USB or from eMMC, cycles at about a minute. Writing a fresh
image over the whole eMMC ends the cycle, and the same image then boots from eMMC. The cycle
reads as a payload or firmware fault and is neither. The zstd initramfs, the display stack in
the initramfs and the 32 MiB slot all boot on the reference unit once the partition is sound.

Three practices avoid the trap:

- Do not boot a medium while another medium written from the same image is attached, unless
  both are known-good. On a machine installed to eMMC from an image, that covers every stick
  written from that image.
- Inspect or repair an install from a stick whose root PARTUUID differs from the installed
  one, so the boot cannot land on the eMMC.
- Run `findmnt /` on a booted system to see which medium the root came from.

### The display stack needs phy-rockchip-dp

Listing `rockchipdrm` in the initramfs looks sufficient, since it pulls `analogix_dp` and the
DRM helpers behind it by symbol dependency. It lights nothing. The eDP controller defers
until `phy-rockchip-dp` registers its PHY, and rockchipdrm binds only once every output has
probed. Without the module, both VOPs and HDMI stay unbound until the root filesystem
supplies it.

An initramfs carrying the display stack needs `rockchipdrm`, `phy-rockchip-dp`,
`panel-simple`, `pwm_bl` and `pwm-rockchip`.
[The display stack in the initramfs](white-screen.md#the-display-stack-in-the-initramfs)
gives the mechanism.

### `mkimage -f auto` emits the wrong FIT image type

`mkimage -f auto -T kernel` writes `type = "kernel"`, but depthcharge needs
`type = "kernel_noload"` with `load = <0>; entry = <0>;`. A FIT with the wrong type does not
boot. Every project corrects the type after the fact or avoids it. depthcharge-tools rewrites
it with `fdtput`, and PrawnOS and Arch Linux ARM hand-write an `.its`.

### Kernel updates must re-sign the payload

The signed blob carries the kernel, so every kernel upgrade invalidates it. Each project
solves this its own way. PrawnOS ships a `.deb` whose postinst runs
`FlashKernelPartition.sh`. Arch Linux ARM ships an interactive pacman `.install` hook that
writes a pre-signed kpart with `dd`.

For Debian the package exists. depthcharge-tools is in forky and ships
`/etc/kernel/postinst.d/zz-depthcharge-tools`, a matching `postrm.d` hook and an
`/etc/initramfs/post-update.d` hook [DEB]. The hook finds the running kernel partition from
`kern_guid=` in `/proc/cmdline`, re-signs the payload and rewrites the partition. The
package depends on `vboot-utils`, `vboot-kernel-utils`, `cgpt`, `u-boot-tools` and
`device-tree-compiler`.

Install the package rather than writing a hook.
[Integrating with depthcharge-tools](boot-chain.md#integrating-with-depthcharge-tools)
covers its A/B slots and its use in an image build.

### `flash-kernel` has no Veyron support

Debian's `flash-kernel` `all.db` carries no Google, Veyron or Chromebook entries [DEB]. Its
only RK3288 machines are Firefly, Tinker and Rock 2, all on a `/boot/boot.scr` u-boot flow
that does not apply here. depthcharge-tools is the Debian-native equivalent for this family:
in effect, flash-kernel for depthcharge.

### A device node is the wrong root

Guides disagree on the eMMC device node, because under stock ChromeOS 3.14-era kernels it
varies by board. Name no device node in the kernel command line, and root on the rootfs
partition's own PARTUUID instead: see [The eMMC device node](family-boards.md#the-emmc-device-node).

### crossystem fails on libreboot

Every guide, including Debian's, Gentoo's and Arch Linux ARM's, sets the developer-mode flags
with this command:

```sh
crossystem dev_boot_usb=1 dev_boot_signed_only=0
```

On the libreboot r20160818 firmware those NV fields have no backing store, and `crossystem`
reports `(error)` for all of them [measured]:

```
dev_boot_usb         = (error)
dev_boot_signed_only = (error)
devsw_boot           = 1        <- developer switch IS on
```

Ctrl+U boots external media anyway, because the libreboot firmware permits it. On a
libreboot Veyron, skip the `crossystem` step: it fails, and it is not a prerequisite. On
stock ChromeOS firmware it stays required.

### USB boot is unit-dependent

The velvet-os documentation states that Veyron "seem[s] to not boot properly with all sd
cards (and as it seems not at all from usb devices)" [HEX]. PrawnOS documents USB as its
primary install path [PRAWNOS]. The two contradict each other directly, most likely
depending on the firmware version [inferred].

USB boot works on the reference unit [measured]. Forky and trixie images, under both board
profiles, boot from a USB stick through Ctrl+U, and the internal eMMC installs from one. The
velvet-os claim therefore holds for some units or firmware rather than for the family. USB
boot is still not something to depend on for an untried board. SD is the fallback, and on a
warm reboot either medium can hit
[the warm-reboot Ctrl+U failure](white-screen.md#the-warm-reboot-ctrlu-failure).

### eMMC geometry is free

PrawnOS places its eMMC kernel partition at sector 20480, which suggests that an eMMC install
must reuse the factory KERN-A slot. It need not. The firmware scans for the type GUID, and a
fresh GPT laid over the factory one boots from the reference unit's eMMC [measured]. The
first partition must start clear of the reserved 8 MiB region at sector 64.
[The factory eMMC layout](boot-chain.md#the-factory-emmc-layout) gives the factory table and
the measured installs.

### chromebook_speedy_defconfig is not a payload

Upstream u-boot's `chromebook_speedy_defconfig` builds a firmware replacement for the SPI
flash, not a payload to chainload [UBOOT]. Seeing the defconfig and concluding that u-boot
can be chainloaded on this board is a misreading.
[Boot architectures](boot-chain.md#boot-architectures) gives the defconfig's settings and the
out-of-tree chainload builds.

### Warm reboots can fail at Ctrl+U

On a warm reboot, Ctrl+U sometimes leaves the machine on the white splash. The firmware fails
to re-initialize the SD card, and the kernel is never read. The failure is in the firmware
and is non-deterministic, unlike the initramfs hang of the PMIC chain.
[The warm-reboot Ctrl+U failure](white-screen.md#the-warm-reboot-ctrlu-failure) gives the
workarounds and how to tell it apart.

### The max98090 starts with no audio path

On a fresh boot the max98090 codec is silent for two independent reasons, and the mute is the
less important one. Clearing the mute alone leaves a board that is still silent. Both
reasons are in `sound/soc/codecs/max98090.c` [source]:

1. The DAC is not routed to either amplifier. The speaker and headphone amplifiers are fed by
   DAPM mixers whose DAC input switches default to off (`max98090.c:949-1000`). These are
   the `SOC_DAPM_SINGLE("Left DAC Switch", ...)` entries under the `Left/Right Speaker Mixer`
   and `Left/Right Headphone Mixer` widgets at `:1189-1201`. No path from the DAC to the
   speakers exists to unmute.
2. The amplifiers are muted. `Speaker Left/Right Switch` and `Headphone Left/Right Switch`
   (`max98090.c:671-679`) are inverted mute bits, so control value `1` means unmuted.

Both are ordinary ALSA controls, so this is mixer state, not a driver defect. They are ASoC
control names rather than simple-mixer elements. `amixer sset Speaker unmute` therefore does
not touch them, and the default `alsamixer` view does not obviously show them. The full set
is:

```
Left Speaker Mixer Left DAC Switch        on     # route: DAC -> amp
Right Speaker Mixer Right DAC Switch      on
Left Headphone Mixer Left DAC Switch      on
Right Headphone Mixer Right DAC Switch    on
Speaker Left Switch                       on     # unmute (inverted mute bit)
Speaker Right Switch                      on
Headphone Left Switch                     on
Headphone Right Switch                    on
Speaker Volume                            70%
Headphone Volume                          60%
```

Set each control with `amixer cset name='...'`, then run `alsactl store` once. It persists
the result to `/var/lib/alsa/asound.state`, which `alsa-restore.service` replays on every
later boot. Installing `alsa-utils` alone therefore does not give working audio, because
there is no `asound.state` to restore and the defaults are silent.

The codec setup comes from `rk3288-veyron-analog-audio.dtsi`, which speedy and minnie share,
so the same defaults apply on minnie [source]. Mickey has HDMI audio only: see
[Per-board differences](family-boards.md#per-board-differences).

### An HCI device is not a Bluetooth stack

The kernel loads `BCM4354.hcd` and brings up `hci0` at probe, whether or not any userspace
can use it. An image that carries the patchram but no `bluez` therefore looks correct in
`dmesg` and does nothing at all. Ship the stack, or do not ship the blob.

Blacklist `btsdio` in either case. The BCM4354 is a combo part, and its SDIO side also
advertises a Bluetooth function. If `btsdio` claims that function, Wi-Fi stops surviving
suspend and resume. Bluetooth on the Broadcom Veyron boards is on UART (`uart0`,
`brcm,bcm43540-bt`) through the `hci_bcm` serdev driver, not on SDIO [DTS]. On the reference
unit Wi-Fi works and Bluetooth function is unverified [measured].

## Sources

- [CROS]: the Chromium OS developer-mode documentation.
- [DCT]: `depthcharge-tools` 0.6.2-5 as Debian forky packages it.
- [DEB]: Debian forky packages, unpacked.
- [DTS]: the mainline `rk3288-veyron*` device trees (v7.1).
- [HEX]: the hexdump0815 and velvet-os imagebuilder.
- [LB]: the libreboot and GNU Boot C201 documentation.
- [LBSRC]: the libreboot r20160818 depthcharge source.
- [PRAWNOS]: PrawnOS (`SolidEva/PrawnOS`).
- [UBOOT]: u-boot master, `configs/`, `board/google/veyron/` and `arch/arm/dts/*-u-boot.dtsi`.

A `[measured]` result comes from the reference unit, an ASUS C201 (veyron-speedy) running
libreboot r20160818 with its depthcharge payload.
