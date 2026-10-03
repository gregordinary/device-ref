# Veyron white screen

This document covers the white screen that a Veyron Chromebook shows after Ctrl+U when a
boot stalls. It is written at family level, for the RK3288 `google,veyron` boards such as
the ASUS C201, the C100P and the Chromebit. Its measurements come from one C201, the
reference unit. The symptom has two distinct causes. One is a deterministic gap in the
initramfs, which can be diagnosed offline. The other is a non-deterministic firmware failure
on warm reboot.

## The symptom

At the developer-mode splash, Ctrl+U selects the external media and the firmware accepts
it. The screen then shows solid white indefinitely, with no text, no cursor, no panic and no
beep. The symptom is strongly associated with mainline kernels: the same machine boots an old
ChromeOS 3.14 kernel image without it. Reports of it span years across the
Arch Linux ARM, Debian and libreboot forums.

## The developer-mode splash

The white screen is the absence of any display update, not an error display. The
developer-mode splash on this firmware is mostly white. Depthcharge does not blank the panel
as it jumps to the kernel. The panel keeps scanning out the firmware's last framebuffer
until a kernel driver repaints it.

A mainline distribution kernel builds the display stack (`rockchipdrm`, analogix DP, the
panel and the backlight) as modules loaded from the root filesystem. The kernel produces no
output until the root filesystem is mounted, udev loads the DRM stack and fbcon binds. That
is typically 15-30 seconds into a healthy boot.

Two consequences follow:

- Some white screen at the start of every boot is normal. Its length depends on what the
  initramfs carries and what it waits for (see [Timing](#timing)). Wait it out before
  concluding anything.
- A white screen that lasts means the boot stalled before the display stack loaded,
  almost always before the root filesystem mounted. The machine is usually alive in an
  initramfs rescue shell that cannot draw, and often cannot read the keyboard either.

The kernel is not crashing on video init. It hangs earlier, silently, and nothing repaints
the panel.

## The PMIC driver chain

The deterministic cause is an initramfs that cannot power the SD card. The dependency behind
it is one that no packaging tool can see.

### The hardware chain

On every Veyron board with an SD slot, the slot's power rails are outputs of the RK808 PMIC,
which sits on I2C bus 0. The mapping comes from `rk3288-veyron-sdmmc.dtsi`, confirmed against the
decompiled `rk3288-veyron-speedy.dtb` in Debian forky's `linux-base 7.1.3+deb14` [DTS][DTB]:

| sdmmc property | Regulator | RK808 output |
|---|---|---|
| `vmmc-supply` (card power, 3.3 V) | `vcc33_sd` | `LDO_REG5` |
| `vqmmc-supply` (I/O voltage, for UHS switching) | `vccio_sd` | `LDO_REG4` |

The mmc host driver, `dw_mmc-rockchip`, asks the regulator framework for `vmmc` at probe.
When the device tree names a supply whose driver has not registered, the answer is
`-EPROBE_DEFER`, a request to retry once the supply exists. If nothing ever provides it, the
host controller never binds and no `mmcblk` device appears.

The initramfs then gives up on root after its 30 s. It drops to a rescue shell nobody can
see, or reboots under a panic timeout, and the screen stays white. The firmware sets no
precedent here. Depthcharge powers the SD card itself to load the kernel partition, then
hands over a machine whose kernel must redo that from scratch.

### The software gap

Beyond the mmc driver itself, the kernel needs three drivers to provide those regulators:

1. The I2C bus driver for the PMIC's bus, `i2c-rk3x`.
2. The RK808 MFD core, `rk8xx-i2c`.
3. The RK808 regulator function driver, `rk808-regulator`.

In Debian forky's `armmp` kernel (7.1.3+deb14), the MFD core is built in and the other two
are modules. The pattern holds for other mainline distribution kernels:

```
CONFIG_MFD_RK8XX_I2C=y      built in
CONFIG_I2C_RK3X=m           module
CONFIG_REGULATOR_RK808=m    module
CONFIG_MMC_DW_ROCKCHIP=m    module
CONFIG_EXT4_FS=m            module
```

An initramfs generator that computes module dependencies resolves symbol dependencies
(`modinfo -F depends`). initramfs-tools, mkinitcpio and dracut all work this way. They
correctly pull `dw_mmc-pltfm` and `dw_mmc` behind `dw_mmc-rockchip`, and `jbd2` behind
`ext4`. The link from the SD controller to an I2C-attached PMIC's regulator driver exists
only in the device tree, and no module metadata records it.

The obvious initramfs, the mmc drivers plus ext4, therefore builds without a warning and
hangs on hardware before a single line of output. A build-time check of the module names
passes too.

On stock Veyron firmware the failure is invisible twice over. At the 16 MiB payload ceiling
the display stack stays out of the initramfs (see
[The payload budget](boot-chain.md#the-payload-budget)), so the rescue shell cannot draw.
The internal keyboard sits behind the ChromeOS EC on a SPI bus. Its driver, `cros_ec_spi`,
needs `spi-rockchip`, another modular chain, and without both the shell cannot read the
keyboard. The result is a white screen and a dead keyboard on a machine that is neither
frozen nor crashed.

### The ChromeOS 3.14 kernel

The images the old guides build on use the ChromeOS 3.14 kernel, which avoids the problem
entirely. It builds the whole chain into the kernel and boots with no initramfs at all.
Arch Linux ARM's `ArchLinuxARM-veyron-latest.tar.gz` (kernel `3.14.0-26-ARCH`) shows
this [ALARMTAR]:

- Its signed command line is
  `console=ttyS2,115200n8 ... root=PARTUUID=%U/PARTNROFF=1 rootwait rw noinitrd`.
- Its `modules.builtin` lists `i2c-rk3x`, `rk808` (the MFD), `rk808-regulator`, `dw_mmc`,
  `dw_mmc-pltfm`, `dw_mmc-rockchip`, `ext4`, `spi-rockchip`, `rockchip-io-domain` and
  `rockchipdrm` itself.

On that kernel, mounting root needs no initramfs, and the console comes up early. The
postmarketOS initramfs avoids the problem differently, with its own module set. A mainline distribution kernel builds all
of the above as modules, so the boot depends on the initramfs contents. The observation that
Veyron "white-screens on mainline" describes this modularization gap, not a mainline display
regression.

The display stack itself is complete. Debian forky, for example, carries everything the
Veyron panel path needs: `CONFIG_DRM_ROCKCHIP=m` with `ROCKCHIP_ANALOGIX_DP=y` and
`ROCKCHIP_VOP=y`, `CONFIG_PHY_ROCKCHIP_DP=m`, `CONFIG_DRM_PANEL_EDP=m`,
`CONFIG_BACKLIGHT_PWM=m`, `CONFIG_PWM_ROCKCHIP=m`, and fbcon. Once the root filesystem
mounts, the console appears on its own.

### The module list

Put the device-tree dependencies into the initramfs explicitly. They are small, and no
generator finds them:

```
i2c-rk3x           # the PMIC's I2C bus
rk808-regulator    # the SD rails, and the io-domain supplies
clk-rk808          # the PMIC's 32 kHz output, which the wifi power sequence waits on
io-domain          # the SoC I/O voltage domains, fed from the same PMIC
dw_mmc             # the SD/eMMC controller
dw_mmc-pltfm
dw_mmc-rockchip
ext4               # or your root fs

phy-rockchip-usb   # root on USB needs the whole USB chain as well
ehci-platform
ohci-platform
dwc2
usb-storage
uas
sd_mod

spi-rockchip       # the EC bus, which gives a rescue shell a working keyboard
cros_ec
cros_ec_spi
cros_ec_keyb
usbhid             # ... and a USB keyboard, for the same shell
hid-generic
```

Without the USB group, a USB-rooted boot enumerates the eMMC and the Wi-Fi and then waits
for a root device that cannot appear. The result is the same white screen from a different
missing driver. The IO-domain driver's module is `io-domain`, not `rockchip_io_domain`.

Each generator takes the list in its own syntax:

- initramfs-tools (Debian): list the modules in `/etc/initramfs-tools/modules`, or in a
  drop-in under `usr/share/initramfs-tools/modules.d/`. mkinitramfs reads every file in that
  directory and concatenates them, so a board can add to a family's list without editing it.
- mkinitcpio (Arch): `MODULES=(i2c-rk3x rk808-regulator dw_mmc_rockchip ...)`.
- dracut: `force_drivers+=" i2c-rk3x rk808-regulator dw_mmc-rockchip "`.

For a kernel built from source, `=y` for the whole chain is equally valid, as the ChromeOS
kernel shows. With initramfs-tools, use `MODULES=list` and assert the built initrd's
contents with `lsinitramfs` as a build step. A name that does not resolve is dropped with
only a warning. [MODULES=dep builds the wrong initramfs](corrections-and-gotchas.md#modulesdep-builds-the-wrong-initramfs)
gives the reason `dep` fails here.

The failure mode reaches beyond Veyron. Any board whose device tree feeds `vmmc-supply` or
`vqmmc-supply` from an I2C PMIC has it. It appears on any distribution kernel that builds the
PMIC stack as modules. On Veyron it presents as a white screen instead of a readable console
message.

## The display stack in the initramfs

An initramfs that carries the display stack lights the panel during the initramfs rather
than after the root filesystem mounts. It fits only under the 32 MiB libreboot payload
ceiling: see [The payload budget](boot-chain.md#the-payload-budget). The stack takes five
modules:

- `rockchipdrm`: the display controller and its eDP output. It pulls `analogix_dp`, `drm`,
  `drm_kms_helper`, `drm_display_helper` and `drm_client_lib` behind it by symbol dependency.
- `phy-rockchip-dp`: the eDP output's PHY, which a symbol walk never finds.
- `panel-simple`: `rk3288-veyron-edp.dtsi` declares an `innolux,n116bge`, which is a
  `panel-simple` entry rather than a driver of its own.
- `pwm_bl` and `pwm-rockchip`: both ends of the backlight. A panel driven with its backlight
  dark is indistinguishable from a panel not driven at all.

The PHY module is the trap. The eDP controller at `dp@ff970000` takes its PHY through `phys`
from a separate node, `edp-phy` (`rockchip,rk3288-dp-phy`), which
[The eDP PHY](../display-reference.md#the-edp-phy) describes. `analogix_dp_probe()` takes the
PHY with `devm_phy_get()`, which defers until `phy-rockchip-dp` registers it. rockchipdrm
binds its display device only once every output it matched has probed
(`rockchip_drm_match_add()`).

Without the PHY module, the whole stack, both VOPs and HDMI included, stays unbound in the
initramfs. The panel lights only after the root filesystem supplies the module, and udev
loads it from there. That is the same blind boot as an initramfs with no display driver at
all, the PMIC chain's shape on the display side. `CONFIG_PHY_ROCKCHIP_DP` is `=m` in
Debian's armhf kernel.

The stack is worth more than the seconds it saves. An initramfs that can draw can report its
own failure. Without one, a slow boot and a dead one look identical, which is most of why
this symptom has the reputation it has.

## The warm-reboot Ctrl+U failure

The second cause is in the firmware and is non-deterministic. On a warm reboot, Ctrl+U
sometimes leaves the machine on the white splash [ALARMFORUM][reported]. The firmware fails
to re-initialize the SD card and never loads the kernel. The reported workarounds are:

- Boot from a full cold power-off rather than a reboot.
- Use the microSD slot directly rather than a USB card reader.

This failure happens before the kernel exists, so no initramfs change reaches it. Nothing on
the card changes during such an attempt, and the same card boots after a cold power-off. If
a boot white-screens intermittently, cold-boot first and only then start debugging software.

## The panic timeout

A 30-second panic timeout on the signed command line makes a failed boot return to the
firmware splash instead of staying on a white screen. A kernel panic, or an initramfs that
gives up waiting for root, then reboots the machine. On a board with no serial console,
whether it reboots and after how long is the most informative thing a failed boot produces.
The timeout costs nothing.

Set it as `sysctl.kernel.panic=30`, not `panic=30`. initramfs-tools' `/init` reads the
command line too and exports any `panic=` word. Its `panic()` (`scripts/functions`,
initramfs-tools 0.151 in forky) takes that branch first. It sleeps, runs `reboot -f` and
exits before `run_scripts /scripts/panic`, so a `panic=` word silently disables every panic
hook.

The kernel applies `sysctl.kernel.panic=` itself, and `/init` never matches it. The panic
hooks then run, and a hook does the reboot. The sysctl spelling has one cost. The kernel
applies sysctl arguments in `do_sysctl_args()`, which `kernel_init()` calls just before it
executes `/init` (`init/main.c`). A kernel panic earlier than that, during driver probe, has
no timeout yet and hangs.

## Telling the causes apart

Without a serial console, the delay before the splash returns tells the causes apart. Two
mechanisms make a blind boot report on itself, and both are cheap enough to belong in any
image for this hardware. One is [the panic timeout](#the-panic-timeout), and the other is an
initramfs panic hook that writes to storage.

### A panic hook that writes to storage

initramfs-tools runs every script in `/etc/initramfs-tools/scripts/panic/` at the moment the
boot gives up. A hook there can mount each ext4 partition it finds and append a report to a
file on it. The report carries the kernel command line, `/proc/partitions`,
`/sys/kernel/debug/devices_deferred`, the loaded modules and `dmesg`. That turns the boot
medium into a flight recorder. Appending rather than overwriting lets one card witness
several attempts, and the hook then honors `/proc/sys/kernel/panic` itself with a sleep and
`reboot -f`.

Write the hook for klibc, not busybox. An initramfs-tools image built with no busybox
installed carries `klibc-utils` instead. That provides `cat`, `dmesg`, `ls`, `mount`,
`sleep`, `reboot` and a handful more, but no `grep`, `tail` or `wc` (checked by unpacking a
built forky initramfs). A hook that pipes through `grep` prints nothing and does not visibly
fail. Filter with the shell's own `case` instead.

### Reading the delay

Each path waits a known time, so the delay before the splash returns identifies it:

| Observation | What ran |
|---|---|
| A report is on the card | The initramfs ran and reached storage. Read the report: `devices_deferred` names the missing driver outright. |
| No report, and back at the splash after about a minute | The initramfs ran and waited its 30 s for root (`scripts/local`: `max(30, rootdelay)`). It gave up, found no ext4 partition to write to, and rebooted after the 30 s timeout. |
| No report, and back after about 30 s | A kernel panic once `/init` had started, with no root wait before it. Either `/init` itself died, or the kernel hit a fault it cannot survive. |
| No report, and no reboot | Nothing reached a panic with the timeout armed. The cause is a hang, a kernel panic before `/init`, or an initramfs that failed to unpack. After a failed unpack the kernel mounts root itself, and plain `rootwait` waits forever (`init/do_mounts.c`). The warm-reboot Ctrl+U failure also lands here. |

A root that resolves to the wrong partition gives the same one-minute cycle, with every
driver loaded and the intended root present. The initramfs waits for the single device that
`root=` resolves to, and only that one, to carry a filesystem.
[Two media written from one image share a PARTUUID](corrections-and-gotchas.md#two-media-written-from-one-image-share-a-partuuid)
describes how two media come to answer to one PARTUUID.

### Examining the card

Without either mechanism the card is still a flight recorder, only a coarser one. After a
white-screen attempt, examine the card on another machine:

1. **Did the root filesystem ever mount read-write?** Run `dumpe2fs -h /dev/sdX2 | grep -i
   'mount'`. ext4 counts only read-write mounts (`ext4_setup_super()` returns early for a
   read-only one), and the initramfs mounts root read-only. A mount count that did not move,
   or a last mount time before the attempt, proves the boot never reached systemd's remount.
   That points at the PMIC driver chain, or, where the kernel never ran, at the warm-reboot
   failure.
2. **Did the system reach userspace?** Look for freshly written `/var/log/journal/` entries,
   DHCP leases, or fresh `/etc/ssh/ssh_host_*` keys on an image that regenerates them on
   first boot. If these exist, the machine is running. A white screen then points at the
   display stack itself, which is a different and much rarer problem.
3. **Is it the warm-reboot failure?** That failure strikes before the kernel is read, so
   nothing on the card changes at all, and the same card boots after a cold power-off.

Before any of that, give the boot the time its arrangement takes: see [Timing](#timing).
Impatience has diagnosed many healthy boots as dead.

## Timing

On the reference unit, a mainline 7.1 kernel with the full module list and an xz initramfs
loads the display stack from the root filesystem. Its white screen after Ctrl+U lasts as
follows [measured, 7.1]:

| Boot medium | White screen after Ctrl+U |
|---|---|
| Internal eMMC | About 5 seconds |
| USB | About 8 seconds |

The medium accounts for those three seconds on its own. A USB root has to bring up the
controller and enumerate the stick inside the initramfs before it can mount.

An unmodified distribution install, with nothing added to the initramfs beyond what the
generator finds, is slower again, on the order of 15-30 seconds. Its display stack loads
only once the root filesystem is mounted and udev reaches it.

Carrying the display stack in the initramfs changes the shape more than the seconds: the
panel lights during the initramfs rather than after it. A 32 MiB image with a zstd
initramfs and the display stack boots the reference unit from internal eMMC
[measured, 7.2.8]. Its white-screen time and the panel lighting during its initramfs are not
yet observed.

Everything the initramfs waits for sits inside the white screen. The largest single
contributor is often a wait that never needed to start, rather than a driver. An initramfs
whose `RESUME=` names a swap device the machine lacks spends about thirty seconds in
`/scripts/local-block` before giving up, with the panel blank throughout. An image with no
swap needs `RESUME=none`. Read `/etc/initramfs-tools/conf.d/` before blaming the drivers.

## Sources

- [ALARMFORUM]: the Arch Linux ARM forum thread viewtopic f=60 t=11848.
- [ALARMTAR]: Arch Linux ARM's `ArchLinuxARM-veyron-latest.tar.gz`, kernel `3.14.0-26-ARCH`.
- [DTB]: the decompiled `rk3288-veyron-speedy.dtb` from Debian forky's
  `linux-base 7.1.3+deb14`.
- [DTS]: the mainline `rk3288-veyron*` device trees (v7.1).

The reference unit is an ASUS C201 (veyron-speedy) running libreboot r20160818 with its
depthcharge payload. Debian forky armhf `7.1.3+deb14-armmp` and trixie
`6.12.94+deb13-armmp` boot it to a login prompt with working Wi-Fi [measured]. Both board
profiles boot: stock `speedy` with a DTB-patched initrd, and `speedy-libreboot` with a
firmware-loaded FIT ramdisk.

The timings come from a mainline 7.1 kernel booted from USB and from internal eMMC. The
32 MiB configuration, zstd with the display stack and `phy-rockchip-dp`, boots from internal
eMMC on mainline 7.2.8. Firmware behavior is checked against the r20160818 depthcharge
source. initramfs-tools behavior is checked against initramfs-tools 0.151 as forky ships it.
The family-level claims are cross-referenced with the Arch Linux ARM veyron tarball and
postmarketOS v26.06 `google-veyron`.
