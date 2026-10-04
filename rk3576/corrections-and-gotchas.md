# RK3576 corrections and gotchas

This document collects the device-tree and driver traps that hold for any RK3576 board
under a mainline kernel. Each entry states the fact, the evidence and the rule that
follows from it. The measurements come from the reference unit, an H96 MAX M9. A claim
stated for the SoC rests on that board unless the text says otherwise. Values and
symptoms that belong to that board alone are in
[the H96 MAX M9 corrections](h96-max-m9/corrections-and-gotchas.md).

## Device tree and power domains

### The GPU power domain needs its supply

A board must bind the GPU power domain to its rail, or a late Panfrost probe can panic
the boot with an asynchronous SError [measured]. The failure is a power-domain race rather
than a GPU fault. It is the same class of defect that the RK3588 GPU domain and the
RK3576 SFC domains show upstream.

`RK3576_PD_GPU` is fed by the GPU rail, which is `vdd_gpu_s0` (an RK806 buck) on the
reference unit. The Rockchip pmdomain driver marks the RK3588 GPU domain as needing a
regulator and leaves the RK3576 GPU domain unmarked [source]. The pmdomain core therefore
powers the RK3576 domain without first enabling the rail. The TRM's VD_GPU power-on
procedure requires the rail to be stable first [source][TRM]. The rail is
`regulator-boot-on` and not always-on, so the regulator core disables it as unused, 30 s
after its late initcall [source].

When Panfrost is built as a module, it can probe after that pass. A late probe powers the
domain against a dead rail, and the power controller never acknowledges:

```
rockchip-pm-domain 27380000.power-management:power-controller:
  failed to get ack on domain 'gpu', val=0x113ff
```

`panfrost_gpu_soft_reset` then writes the unpowered register block, and the write aborts
asynchronously:

```
SError Interrupt on CPU6 ... Kernel panic - not syncing: Asynchronous SError
```

An early probe that finds the boot-on rail still up powers the domain cleanly and
Panfrost initializes. The failure therefore depends on probe timing, and a passing boot
proves nothing about the next one.

The fix binds the domain to its supply in three parts, and with it Panfrost loads
normally as a module, with no blacklist [measured]:

- The Rockchip pmdomain driver marks `RK3576_PD_GPU` as needing a regulator, as it
  already does for the RK3588 GPU domain.
- `rk3576.dtsi` gains a `pd_gpu` label on the GPU power-domain node, which mainline does
  not carry, so that a board can reference the node.
- The board's `pd_gpu` node carries `domain-supply = <&vdd_gpu_s0>`.

The domain then enables the rail before it powers and holds the rail up while powered.
Building Panfrost in (`=y`) also makes it probe before the disable-unused pass. That
treats the timing and leaves the supply undescribed. The GPU itself, its render node and
its rendering results are in [the RK3576 GPU reference](video-reference.md#gpu).

### rk3576.dtsi has no gpio aliases

Mainline `rk3576.dtsi` carries no `gpio0`-`gpio4` aliases, through 7.2.8 [source].
`rk3288.dtsi` and `rk3588-base.dtsi` carry them. Without an alias, the bank a GPIO
controller binds is its probe order. `rockchip_gpio_probe()` finds a controller's pin
bank with `of_alias_get_id(np, "gpio")`. A node with no alias takes the next value of a
static counter in `drivers/gpio/gpio-rockchip.c` [source].

The bank decides the chip's label. It also decides which chip `rockchip_pinconf_set()`
drives a pad's output through. A probe that defers after drawing its number draws another
on retry [source]. The retry lands on a sibling's bank, or past `gpio4` onto no bank at
all. There `rockchip_gpio_find_bank()` returns NULL and the probe fails.

On the reference unit the counter draws correctly: the five controllers probe in
device-tree order in every mainline boot capture [measured][LOG]. Alexey Charkov's posted
patch adds the five aliases to `rk3576.dtsi`, unmerged as of 7.2.8. It makes the bank
binding a property of the device tree. With it applied, the decompiled DTB changes only
by the five `/aliases` entries [measured].

The `/dev/gpiochipN` numbers follow gpiolib registration order, and the aliases do not
change them. A userspace tool that names a chip by that number works only while the
controllers register in bank order. The chip under the bank's platform device does not
depend on registration order. For gpio1 it is
`/sys/bus/platform/devices/2ae10000.gpio/gpiochip*`. The Bluetooth control lines of the
reference unit are an instance, described in
[the H96 MAX M9 Bluetooth reference](h96-max-m9/wifi-bluetooth.md#bluetooth).

U-Boot needs no aliases. Its `drivers/gpio/rk_gpio.c` takes the bank from `gpio-ranges`,
which every `rk3576.dtsi` GPIO node carries, before it looks for an alias [source].

### A board-level pinctrl-0 replaces the inherited groups

A `pinctrl-0` assigned in a board `.dts` replaces the value the node inherits from the
SoC dtsi. It does not extend it. Nothing warns and the DTB compiles. The loss shows only
in whichever sideband signal the dropped group carried.

On the RK3576, `rk3576.dtsi` gives `&hdmi` three pin groups by default:

| Group | Pins | Signals |
|---|---|---|
| `hdmi_txm0_pins` | gpio4 PC0 and PC1 | CEC and hot-plug detect input |
| `hdmi_tx_scl` | gpio4 PC2 | DDC clock |
| `hdmi_tx_sda` | gpio4 PC3 | DDC data |

A board `pinctrl-0` on `&hdmi` that names only the board's own pins unmuxes all three
groups. The consequences follow from the table:

- With HPD unmuxed, the kernel never detects a hotplug.
- With DDC unmuxed, every EDID read after boot fails.
- With the CEC pin unmuxed, the CEC adapter registers but is not connected to the wire.

HDMI output can still work, which hides the fault. If the bootloader lights the display,
it muxes these pins before the handover and the kernel inherits a live link. The pins
matter for everything after the handover, which
[the display handover](display-reference.md#handing-the-display-over-to-an-operating-system)
describes.

When a board overrides `pinctrl-0` on a node that already has one, it must list the
inherited groups explicitly and append its own:

```dts
&hdmi {
	pinctrl-0 = <&hdmi_txm0_pins &hdmi_tx_scl &hdmi_tx_sda &board_hdmi_pins>;
};
```

Here `board_hdmi_pins` stands for the board's own groups. The reference unit shows the
full failure [measured], and its symptoms and value are in
[its corrections](h96-max-m9/corrections-and-gotchas.md). The mux alternatives and their
IO domains are in
[the pin-mux table](hardware-design-guide.md#pin-mux-alternatives-across-power-domains).

### Vendor-driver log lines are not mainline requirements

The vendor kernel logs missing-regulator messages that name nothing a mainline board
`.dts` can supply. The vendor 6.1 boot logs show two [LOG]:

- `rockchip-vop2 27d00000.vop: ... no regulator (vop) found`, followed by
  `failed to init opp info`. Mainline `rockchip_drm_vop2.c` has no OPP or regulator
  support, and `rk3576.dtsi` defines neither a VOP OPP table nor a `vop-supply` [source].
  There is nothing to describe.
- `rk_gmac-dwmac 2a220000.ethernet: supply phy not found, using dummy regulator`. Mainline
  `dwmac-rk.c` calls `devm_regulator_get(dev, "phy")`, and `rockchip-dwmac.yaml` accepts
  an optional `phy-supply` [source]. The mainline EVB omits it, and the dummy regulator is
  benign.

A `phy-supply` needs a source that names the rail. On the reference unit no source (the
factory DT, the schematic or the mainline EVB) names the rail that feeds the Ethernet
PHY, so the property stays unset rather than guessed. A vendor-driver
log line is not a required property. Check what the mainline driver consumes before
porting anything a vendor log names.

## Audio

### SAI transmit routing

Three mechanisms stand between a registered RK3576 SAI card and audible output
[measured, 2026-07-30]. The route property must carry the hardware's semantics, the
driver must keep the route written, and the MCLK output gate must have a consumer.
In-tree RK3576 boards program the identity route, which hides the first two.

`rockchip,sai-tx-route` programs the transmit fields of `SAI_PATH_SEL`. In hardware, TX
field x selects which stream drives SDO port x: the index is the port and the value is the
stream. The binding's prose describes the inverse, documenting `<3>` as "sdo3 is sending
to data0", with the index as the data stream. The driver writes entry i into field i, so
entry i names the stream for SDO port i. A single-entry `<2>` therefore asks for
"sdo0 <- stream 2".

The driver writes the fields from 0 upward, so reaching field N takes N+1 entries. If the
property is absent, `SAI_PATH_SEL` keeps its reset value, the identity map. Muxing a
non-default SDO pin alone then emits the stream on a lane that reaches no pin. When
checking a route value by hand, compute the register mask programmatically. The reference
unit's route and register word are in
[its hardware reference](h96-max-m9/hardware-reference.md#display-and-audio).

The mainline driver loses the route. `rockchip_sai_parse_paths()` programs
`SAI_PATH_SEL` once at probe, and the value does not survive to the running device.
`regcache_sync()` on resume does not restore it either. On the reference unit, with a
non-identity route present in `/proc/device-tree` and only sdo2 muxed, the register read
`0xe4e4` with the PCM `RUNNING` [measured, 2026-07-30].

The device tree alone cannot fix this. A board on the identity map cannot tell writing
the value from losing it, which is why in-tree boards pass. The fix re-applies the routing
from `rockchip_sai_runtime_resume()` after the cache sync. The route then reaches the
running device and survives PM cycles, and the jack plays end to end on 7.2
[measured, 2026-08-26].

The MCLK output gate needs a consumer. `assigned-clocks` sets a clock's rate, not its
enable state. The SAI driver enables only the clocks it looks up by name, `mclk` and
`hclk`. An MCLK output gate such as `CLK_SAI1_MCLKOUT_TO_IO` therefore stays off, and MCLK
is generated without ever leaving the SoC.

A `clocks` reference on the `simple-audio-card,cpu` sub-node gives the gate its consumer.
On a board with an I2C codec, the codec node plays that role, as the EVB's ES8388 node
does. With the reference in place, both clock gates switch on during playback and release
at idle [measured, 2026-07-30].

## USB

### Capping a dwc3 port to High Speed

Capping an RK3576 dwc3 controller to High Speed needs the speed cap and the PHY removal
together, or the port goes dead [measured, 2026-07-19][LOG]. With
`maximum-speed = "high-speed"` alone and the USBDP PHY still in `phys`, xHCI registers and
the hubs probe. No connect event then arrives at any speed. The working template for
drd0 has three parts:

- `maximum-speed = "high-speed"` on the controller
- `phys = <&u2phy0_otg>`, which drops the USBDP PHY from the controller
- `&usbdp_phy` disabled

This is a fact about dwc3, not a recommendation. The reference unit's drd0 runs at
SuperSpeed uncapped [measured, 2026-08-08], as
[its corrections](h96-max-m9/corrections-and-gotchas.md) record.

drd1 must keep its combo PHY 1 reference, `combphy1_psu`, in `phys`. Dropping it makes
the board unbootable [measured, 2026-08-08]. Nothing on the RK3576 gates that controller's
SuperSpeed port. `rk3576_combphy_grfcfgs` in `phy-rockchip-naneng-combphy.c` carries no
`u3otg1_port_en`, unlike the RK3588's [source]. xHCI therefore enumerates the port whether
or not a PHY backs it.

A PHY node with no consumer never receives `phy_init` or `phy_power_on`, so the PIPE
interface stays unpowered. The PORTSC read in `hub_activate` then takes an external abort.
The kernel reports an `SError` with ESR `0xbf000002` in the call chain
`xhci_portsc_readl <- xhci_hub_control <- hub_activate`, and the boot dies at ~1.85 s.

A combo PHY lane that reaches no connector is no reason to drop the PHY. Such a lane never
trains, and `2b060000.phy: wait phy status ready timeout` on every boot means only that.
The line is cosmetic. The design guide also states that combo PHY 1 must be in USB3 mode
for drd1's USB 2.0 PHY to work [source][HDG]. That statement is not measured on the
reference unit, and
[the combo PHY 1 coupling](hardware-design-guide.md#usb-controllers-and-combo-phy-1)
describes it.

### xHC errors on resume

On resume from s2idle, each RK3576 dwc3 host controller logs an xHC error unless the board
sets `snps,dis_u2_susphy_quirk` [measured]:

```
xhci-hcd xhci-hcd.N.auto: xHC error in resume, USBSTS 0x401, Reinit
```

The cause is an asymmetry in how dwc3 drives the USB2 PHY across suspend [source].
`rk3576.dtsi` sets `snps,dis_enblslpm_quirk` on both controllers, so every RK3576 board
inherits it. The host suspend path, `dwc3_suspend_common`, treats that quirk as reason to
assert both the `ENBLSLPM` and `SUSPHY` bits of `GUSB2PHYCFG` before the PHY driver
suspends. The resume path, `dwc3_resume_common`, clears `SUSPHY` only under
`snps,dis_u2_susphy_quirk`. The `dis_enblslpm_quirk` alone clears just `ENBLSLPM`.

Without the U2 quirk, `SUSPHY` stays asserted after resume. The USB2 PHY never leaves
suspend, and the xHC re-init errors. Adding `snps,dis_u2_susphy_quirk` to both host
controllers (`&usb_drd0_dwc3` and `&usb_drd1_dwc3`) in the board `.dts` makes the resume
path restore `SUSPHY` and clears the error [measured]. The property changes only the
resume path.

## Kernel modules

### rkvdec cannot be unloaded

`rmmod rockchip_vdec` hangs on the RK3576 and leaves the module at refcount -1
[measured, 2026-08-27]. A module in that state neither unloads nor reloads. `modprobe`
then fails with `Device or resource busy`, and `/dev/video0` is gone. Only a reboot
clears it. The hang occurred twice on the reference unit, the second time with a 300 s
window. The driver hangs, so a longer timeout does not help.

Replacing the decoder module on a running system therefore takes a reboot:

1. Back up the installed `rockchip-vdec.ko` under `/lib/modules/<ver>/kernel/`.
2. Copy the new `rockchip-vdec.ko` over it.
3. Run `depmod -a`.
4. Reboot.

Restoring the original is the same copy from the backup, followed by a reboot. The new
module must come from the same build as the running kernel, for example from that build's
`linux-image` package.

### A module built by another toolchain does not load

A module built from byte-identical source with a different toolchain fails to load against
the running kernel [measured]:

```
rockchip_vdec: disagrees about version of symbol video_device_release_empty
```

The measured pair is a kernel built with Debian forky's gcc and the same module built with
a host toolchain. Even with `CONFIG_GENKSYMS=y`, which computes CRCs from source rather
than DWARF, 10,437 of 25,831 exported-symbol CRCs differ between the two builds (40%)
[measured]. That is too many to dismiss as a hashing artifact.

`CONFIG_MODULE_FORCE_LOAD` and `modprobe --force-modversion` are therefore no shortcut.
Forcing the load asserts an ABI nobody checked. Take the module from the same build as the
kernel.

A module built by hand also needs the kernel's `LOCALVERSION`, which is `-1-arm64` on the
measured kernel. Without it, vermagic comes out as `7.x.y+` and no module loads
[measured].

## Sources

A `[source]` claim reads the mainline Linux or U-Boot tree, and the entry names the kernel
version where it matters. The citation keys are:

- [LOG]: boot logs. The vendor 6.1 boot logs are the ones posted to the Armbian forum's
  H96 MAX M9 thread (`boot1.txt` to `boot3.txt` and `dmesg-19.0.txt`). Six serial-console
  captures of the reference unit's own vendor boot join them. The mainline boot logs
  are serial-console captures of the reference unit, the High Speed cap among them.
- [HDG]: Rockchip's RK3576 Hardware Design Guide V1.1, summarized in
  [the hardware design guide notes](hardware-design-guide.md).
- [TRM]: the Rockchip RK3576 TRM V1.2, Part 1, chapter 6 (PMU).
