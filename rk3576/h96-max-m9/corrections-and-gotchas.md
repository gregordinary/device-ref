# H96 MAX M9 corrections and gotchas

This document covers the H96 MAX M9, an RK3576 TV box, at board level. It records claims
about the board that community sources state wrongly, each with the fact and its evidence.
It also records the board-level traps of a mainline kernel bring-up on it. Traps
that hold for every RK3576 board live in the SoC documents, listed under
[SoC-wide traps](#soc-wide-traps). The [hardware reference](hardware-reference.md) states
what the board is, and the [enablement status](mainline-enablement-status.md) states what
works on which kernel.

## Corrected claims

Each entry names a wrong claim and where it comes from, so that an old source does not
reintroduce the wrong value. The [hardware reference](hardware-reference.md) states every
value here correctly.

### DRAM is LPDDR4X-2112

Community enablement notes give the memory as DDR4, with the warning that "the loader blob
must match DDR4" [THREAD].

The memory is LPDDR4X at 2112 MHz, two channels of 4096 MB, made by SK Hynix (manufacturer
ID `0x6`) [measured][LOG]. The DDR-init output reads `LPDDR4X, 2112MHz`.

The rkbin loader `rk3576_ddr_lp4_2112MHz_lp5_2736MHz_v1.12.bin` is the right one, because
2112 MHz is its native LPDDR4X target. For this SoC, rkbin ships one unified LPDDR4/LPDDR5
loader and no DDR4 file (see [the boot chain](../boot-firmware.md#the-boot-chain)). The
board therefore runs the standard RK3576 DDR loader and needs no board-specific DDR blob.

### The gpio0 bank is at 0x27320000

A GPIO-bank list mined from the vendor tree gives `gpio0 = 0x2ae00000` [THREAD].

The gpio0 bank is at `0x27320000`, in the PMU (always-on) domain [DTSI]. No GPIO bank sits
at `0x2ae00000`: that address is the SARADC (`adc@2ae00000`). The main-domain banks are
gpio1 at `0x2ae10000`, gpio2 at `0x2ae20000`, gpio3 at `0x2ae30000` and gpio4 at
`0x2ae40000`. The factory DTB agrees, and maps the phandle of `gpio@27320000` to gpio0.

Adopting the wrong base resolves every gpio0 pin to the wrong controller. Those pins include
the status LED, the power-key input, the RK806 PMIC interrupt and the RTC interrupt.

### The HDMI bias pin is frl-enable-gpios, active-low

The pin is gpio2 PB0 [DTB]. The factory device tree names it
`enable-gpios = <&gpio2 RK_PB0 GPIO_ACTIVE_HIGH>` on `hdmi@27da0000`, and the hardware
reference lists it as the factory tree does. Community-derived values carry that property
into mainline trees unchanged, and the mainline binding differs.

The pin is the external bias switch for the HDMI data lines, not a power enable [DTB]. The
vendor 6.1 driver (`dw_hdmi-rockchip.c`) drives it low for HDMI 2.1 FRL and high for HDMI
1.4/2.0 TMDS. Without the pin, that driver refuses modes above 600 MHz.

Mainline `dw_hdmi_qp-rockchip.c` has no `enable-gpios` property [source]. It models this pin
as the optional `frl-enable-gpios`. The binding describes that line as "asserted when
operating in HDMI 2.1 FRL mode and deasserted for HDMI 1.4/2.0 TMDS". It is "used to
control external voltage bias for HDMI data lines". The binding is
`rockchip,rk3588-dw-hdmi-qp.yaml`, which `rockchip,rk3576-dw-hdmi-qp` shares.

The polarity inverts. Mainline asserts the line in FRL mode, while the vendor driver drives
it physically low in FRL. A mainline board device tree therefore uses
`frl-enable-gpios = <&gpio2 RK_PB0 GPIO_ACTIVE_LOW>`. An `enable-gpios` property on a
mainline node is silently ignored and fails `dtbs_check`.

### The mainline base has no camera nodes

Community bring-up advice is to fork `rk3576-evb1-v10.dts` and strip the IMX415 camera
nodes, which probe and fail on this board [THREAD].

Mainline `rk3576-evb1-v10.dts` carries no camera nodes [DTSI]. It names no IMX415, no
`&csi*` or `&mipi*` node and no camera-only regulator. The IMX415 is EVB carryover present
only in the vendor DTB, so a fork of the mainline base has nothing to remove.

### The analog codec is an HT5010 on SAI1

The thread reports "HT5010 on SAI1" [THREAD], and the report is correct, although the
factory device tree also declares an ES8388.

The factory device tree declares two codecs, each with its own sound card [DTB]. One is an
`everest,es8388` at i2c reg `0x10`. The other is an HT5010 on `sai@2a610000`, fronted by a
`rockchip,dummy-codec`. Only the HT5010 card registers [LOG]. Six vendor boot captures print
the same three-card list, with no es8388 or i2c-codec probe line in any of them.

The ES8388's own card name, `rockchip-es8388`, comes from `rockchip,card-name` on the
`rockchip,multicodecs-card` node, and no capture shows it. Card `#1: rockchip,es8388` in
those captures is the HT5010 card: the vendor set
`simple-audio-card,name = "rockchip,es8388"` on it. A comma in place of a hyphen is the only
difference between the two strings. The [factory device tree](factory-device-tree.md#the-three-audio-cards)
lists all three cards.

On the reference unit the ES8388 path is dead under mainline too: i2c3 never acknowledges,
and the access fails with `-110` [measured]. Nothing drives that node under the vendor OS
either. The ES8388 node is EVB carryover, and the board's analog path is SAI1 -> HT5010.

A board device tree therefore points `simple-audio-card` at SAI1 with a dummy codec and no
i2c codec. The data line is `sdo2` rather than the EVB's `sdo0`, and MCLK is on
`sai1m0_mclk`. Keeping the ES8388 node inherits a bus that no unit has been seen to answer
on.

The amplifier enables are a dead end [measured, 2026-07-30]. `spk-con-gpio` (gpio2 PB1) and
`hp-con-gpio` (gpio3 PD6) are named only on the card that never probes. Driving both high
from userspace produces no sound, alone or combined with every other change. Both read
`input` and undriven at boot, so nothing contends for them.

The jack is fed from `sdo2`. A stream reaches it with `rockchip,sai-tx-route = <0 1 0>` and a
kernel change that keeps `SAI_PATH_SEL` written (see
[SAI transmit routing](../corrections-and-gotchas.md#sai-transmit-routing)). The board's
values and playback results are in the
[hardware reference](hardware-reference.md#display-and-audio).

### The Wi-Fi label ap6275p is boilerplate

In the factory device tree, the `wlan-platdata` node reports
`wifi_chip_type = "ap6275p"`, which reads as a Broadcom part [DTB].

The fitted part is an AIC8800D80 on SDIO (`mmc@2a320000`) [measured][LOG]. Three independent
observations confirm it:

- The module silkscreen reads `L8800D 80 WIFI&BT`.
- The SDIO enumeration reports `vid:0xC8A1 did:0x0082`, and `0xC8A1` is AICSemi.
- The `aic8800_bsp` driver binds.

Broadcom's SDIO vendor ID is `0x02D0`, and no `brcmfmac` or `bcm` line appears.

The label is correct on some units by coincidence. Wireless varies by production batch:
earlier units carry a genuine Broadcom AP6275P [consensus][THREAD]. Later units run an
AIC8800D80 under the same label, and so do both units examined (see
[observed configurations](hardware-reference.md#observed-configurations)). The vendor copies
the label whatever part is fitted.

The `wifi_chip_type` string in a Rockchip vendor DTB does not identify the part. Identify
the Wi-Fi silicon by its SDIO vendor ID. The Wi-Fi part does not gate a mainline boot, and
the [Wi-Fi and Bluetooth reference](wifi-bluetooth.md) covers the AIC8800D80's driver and
bring-up.

### The USB device port is the USB 3.0 connector

The manual's rear-panel order (Audio, DC, SPDIF, Ethernet, HDMI, USB 3.0, USB 2.0) suggests
that the corner USB 2.0 Type-A is the dual-role port and the USB 3.0 port is host-only
[SPEC].

The USB 3.0 port is the device port [measured]. Starting u-boot's rockusb gadget with
`rockusb 0 mmc 0` prints "waiting for usb3 connection". The box enumerates to the host only
with the cable in the USB 3.0 connector.

The device tree shows the same [DTB]. Of the two DWC3 controllers, only `usb@23000000`
(`usb_drd0`) carries an `extcon` property, which points at USB2 PHY 0. `usb@23400000` carries
none. The `otg-bvalid`, `otg-id` and `linestate` interrupts sit on both USB2 PHYs, so they
identify neither controller. The `0` in `rockusb 0 mmc 0` is the
UDC index, and it selects drd0.

A failed adb-over-USB attempt says nothing about the routing. The factory Linux device tree
sets `dr_mode = "host"` on both controllers, so adb cannot enumerate on either port.

Identify the OTG port by the controller that carries the dual-role wiring. The connector
type, the manual's port order and a failed enumeration do not identify it.
Controller-to-connector routing is a PCB fact and does not change with firmware.

The gadget enumerates at high speed (USB 2.0 signaling) even on the SuperSpeed connector, so
a full eMMC dump takes the better part of an hour. "Device is high speed" is not a sign of
the wrong port.

### The bundled remote is a 2.4 GHz RF handset

The box ships with a handset remote, and the vendor device tree carries an IR receiver with
six NEC keymap variants. Together these suggest that the bundled remote is that IR handset.

The bundled "XFY MIC RC" handset is 2.4 GHz RF with voice, not infrared [measured]. It works
pointed in any direction, on first use and with no pairing step. It is factory-paired to an
internal USB receiver (`0a45:0008`) behind an internal USB hub (`1a86:8091`), both soldered
inside the box. Both enumerate at ~3.2 s into boot, and neither sits on an external port.

The receiver presents as a standard USB HID keyboard and mouse, so the remote drives Linux
with no driver, keymap or configuration. Up, down and OK work immediately. The power key
reaches logind as USB `KEY_POWER` and powers the box off. An rc-core capture of the IR
receiver stays empty throughout.

The remote is not Bluetooth [measured]. `hci0` is up but holds no connection to the remote,
and the RF link runs entirely through the internal USB receiver.

The board's IR receiver is a separate path, and it works (see
[the IR receiver](#the-ir-receiver)). The bundled remote cannot drive it, so the recovered
IR keymap serves IR and universal remotes rather than this handset.

Identify a TV-box remote by what receives it: an rc-core IR device, a USB HID receiver or a
Bluetooth link. An IR node in the vendor device tree does not identify it. Several receivers
can coexist. This box carries an IR receiver and two internal USB RF receivers, a Logitech
`046d:c52b` beside the XFY receiver.

## Integration gotchas

These traps are specific to this board. Each states the symptom, the cause and the rule
that follows.

### The recovery button depends on the firmware build

The recovery button sits behind the AV jack and reads as SARADC channel 1. What it does at
power-on depends on the factory bootloader build, not on the hardware.

On the reference unit, holding the button at power-on boots Android recovery [measured]. A
rockusb `Loader` appears only by interrupting the factory U-Boot on the serial console and
running `rockusb 0 mmc 0`. The recovery menu's "reboot to bootloader" lands in fastboot,
because the factory DTBO maps `mode-bootloader` to `0x5242c309` (`BOOT_FASTBOOT`).

Units in the Armbian thread behave differently [consensus][THREAD]. Several posters reach
Loader mode directly by holding the button with a toothpick before connecting USB, with no
serial console. One sums it up as "Loader mode sufficient, Maskrom not required". Their
factory U-Boot enters its rockusb gadget on the key press, and the reference unit's does
not.

The download-key handling lives in the factory U-Boot and DTBO, which vary by firmware
build. The button, the SARADC channel and the threshold are the same on every unit. What
differs is what the factory bootloader does with a pressed key. Check a unit's firmware
before assuming that the button reaches a flashing mode. The eMMC-pad short into maskrom
works under any firmware (see [the boot notes](hardware-reference.md#boot)).

With mainline u-boot carrying both download-key fixes, the button enters maskrom
[measured]. It lands in BootROM download rather than Loader, whatever factory firmware the
board shipped with. [The SARADC download key](../boot-firmware.md#the-saradc-download-key)
describes both fixes.

### A 480 Mb/s link on drd0 is a seating fault

A drive at 480 Mb/s in the blue port is almost always not fully inserted
[measured, 2026-08-08]. The board sits back from the enclosure's front panel. A plug bottoms
out on the case before the connector's five recessed SuperSpeed contacts mate. The four
USB 2.0 contacts at the leading edge do mate. Take the board out of the case, or push the
plug fully home, before suspecting anything else.

Software cannot see the fault, and it mimics a configuration problem exactly:

- The port enumerates at high speed, at full USB 2.0 throughput (39 MB/s measured), with no
  error of any kind.
- The SuperSpeed root hub of drd0 registers correctly (1 port, `speed=5000`) and then logs
  nothing: no failed training, no warm reset and no timeout.
- SuperSpeed receiver detection is a DC test for the device's 50 Ω termination, so an
  unmated pair reads exactly like no device.
- A configuration with `snps,dis_rxdet_inp3_quirk` and one without it behave identically,
  on 7.1.3 and on 7.1.7.

Seated properly, the port runs at 5 Gbps [measured, 2026-08-08]. Eleven consecutive 8 GB
reads (92 GB in 606 s) ran at 145-158 MB/s with zero link errors. The port still reported
`speed=5000` afterwards. That rate is the drive's flash, not the link. The result
reproduces on a 7.1.3 kernel.

Intermittent contact has its own signature. A device trains at SuperSpeed, runs, then drops
into a `-62`/`-71` SetAddress loop that no reset recovers [LOG]. A link with a real
signal-integrity margin problem does not then run 92 GB clean at 154 MB/s.

`snps,dis_rxdet_inp3_quirk` is no fix for that loop. Neither the factory DTB nor a
community device tree for this board sets it on drd0. The quirk does not suppress
SuperSpeed [measured], and the fault it targets is mechanical.

The drd0 controller needs nothing beyond the SoC dtsi to reach SuperSpeed [DTB]. That is
`dr_mode = "host"`, `status = "okay"`, and `&usbdp_phy` enabled with
`rockchip,dp-lane-mux = <2 3>`. The unit's own factory DTB, extracted from the `boot`
partition of an eMMC backup, gives drd0 exactly that plus `extcon`. That DTB identifies
itself as `Rockchip RK3576 EVB1 V10 Board`.

The board has no VBUS regulator to describe. The mainline EVB1 and Radxa ROCK 4D trees give
the drd0 port a switched `phy-supply` on `u2phy0_otg`, and this board's factory DT
references none. Its `usb-otg0-pwren` pinctrl entry is dead EVB boilerplate: the phandle is
unreferenced, and the pin, gpio0 PD1, drives the status LED. The port's 5 V is hardwired. A
third-party log from the vendor 6.1 kernel shows the same `Looking up vbus-supply ...
failed` on drd0's `otg-port` [reported].

A u-boot built without `CONFIG_PHY_ROCKCHIP_USBDP` runs drd0 at High Speed only. USB booting
from the port still works.

### The drd1 controller and combo PHY 1

The USB 2.0 Type-A port belongs to drd1, behind an internal `1a86:8091` hub, and drd1's
SuperSpeed lane reaches no connector [measured]. That lane's PHY is naneng combo PHY 1
(`combphy1_psu`, `0x2b060000`). The `2b060000.phy: wait phy status ready timeout` line on
every boot is the unrouted lane failing to train, and it is cosmetic.

That line is no reason to drop the PHY. The drd1 controller must keep `combphy1_psu` in its
`phys`. Without it the board does not boot, and dies on an SError in `hub_activate`
[measured, 2026-08-08]. The mechanism is under
[capping a dwc3 port to High Speed](../corrections-and-gotchas.md#capping-a-dwc3-port-to-high-speed).

`combphy1_psu` also stays enabled for drd1's High Speed side. The design guide states that
combo PHY 1 must be in USB3 mode for USB2.0 PHY1 to work [HDG]. On this board that PHY
serves the USB 2.0 Type-A port. The statement is the guide's about the SoC and is not
measured on this board (see
[USB controllers and combo PHY 1](../hardware-design-guide.md#usb-controllers-and-combo-phy-1)).

### The HDMI pinctrl-0 override

A board device tree that sets `pinctrl-0 = <&hdmi_frl_en>` on `&hdmi` loses HDMI hotplug and
every EDID re-read after boot. Output still works, but the panel must be connected at boot.
Hot-plugging fires no udev event, and an EDID re-read fails with `i2c read err!`. The vendor
kernel shows the same signature, so the fault reads as a platform limitation rather than a
board bug.

The board-level property replaces the three groups that `rk3576.dtsi` gives the controller,
rather than extending them. Those groups carry hot-plug detect, CEC and DDC (see
[a board-level pinctrl-0 replaces the inherited groups](../corrections-and-gotchas.md#a-board-level-pinctrl-0-replaces-the-inherited-groups)).
Output survives only because u-boot muxes these pins before the handover, and the kernel
inherits a live link.

The fix restates the inherited groups and appends the board's FRL pin:

```dts
pinctrl-0 = <&hdmi_txm0_pins &hdmi_tx_scl &hdmi_tx_sda &hdmi_frl_en>;
```

Three independent sources agree that m0 is this board's variant. They are the SoC dtsi
default, upstream `rk3576-evb2-v10.dts`, and the board's factory tree, which muxes bank 4
pins 16-19 at function 9. With the fix, both plug transitions fire a udev `change` on
`card0` [measured, 2026-07-30]. The EDID re-read after a replug returns a byte-identical
256-byte block with the same 30 modes. Hotplug also holds under a desktop session
[measured, 2026-08-08].

The CEC pin shares `hdmi_txm0_pins`, so the CEC adapter reaches the wire only with this fix
in place.

### The IR receiver

The board carries an IR receiver on gpio0 PC4, and mainline drives it. The bundled remote
is RF (see [the bundled remote](#the-bundled-remote-is-a-24-ghz-rf-handset)), so this path
serves an IR or universal remote.

The vendor kernel drives the IR pin with an out-of-tree PWM-capture block,
`rockchip,remotectl-pwm-v4` on `pwm@27330000`. Its key tables sit inline as
`rockchip,usercode` and `rockchip,key_table` nodes. Mainline has no such driver. It uses
`gpio-ir-receiver` (`CONFIG_IR_GPIO_CIR`) feeding the rc-core NEC decoder, the standard path
for these boxes.

Reception works end to end [measured, 2026-07-30]. The node binds as `rc0` with the NEC
decoder available. The test used a borrowed H96 Max H313 handset, because the bundled
remote cannot drive the path. Raw capture from `/dev/lirc0` shows standard NEC framing: a
9 ms leader, a 4.5 ms space, 560 us bits and 2.25 ms repeat codes. The decoder emits
scancodes, and they reach the input layer as `EV_MSC`/`MSC_SCAN`.

#### The receiver pin

The demodulated IR line lands on the SoC's PWM0 IR-capture pin, gpio0 PC4. The factory
pinmux `<0 0x14 12>` decodes as bank 0, PC4, PWM function 12, the same `pwm0m0_ch0` pin
that mainline names. A board device tree muxes it as a GPIO input and attaches a
`gpio-ir-receiver` node with `gpios = <&gpio0 RK_PC4 GPIO_ACTIVE_LOW>`. The demodulator
idles high and pulses low on carrier. The pin is in the PMU (always-on) domain, so it can
serve as a wake source. The path needs no PWM node and no vendor driver.

#### The keymap

The protocol is NEC, and the vendor tree's keymaps can be recovered, but they do not cover
every handset. The vendor's six `ir_key*` nodes are six handset variants. Each carries a
`usercode` (the NEC address pair) and a flat `key_table` of `(command, keycode)` pairs.

Translating them into rc-core scancodes is mechanical once the NEC packing is fixed.
Standard NEC packs `addr << 8 | cmd` where the two address bytes are complementary.
Extended NECX packs `addr << 16 | addr_inv << 8 | cmd` otherwise. The packing is checked
against `drivers/media/rc/ir-nec-decoder.c` [source]. The six tables sit on disjoint
addresses (`0x4040`, `0xff`, `0x1dcc`, `0x7f`, `0xfe` and `0xfd01`). They therefore coexist
in one `ir-keytable` TOML map, and only the matching handset's keys fire.

The handset tested sits on address `0x04` and matches none of the six [measured]. Its
presses decode cleanly and produce zero key events. A 13-button table captured from that
handset is verified on hardware, and none of the six vendor tables is. Capture a press
before trusting any recovered table.

A table's `protocol` must match how the press decodes. rc-core does not match a 16-bit
`nec` scancode against a `necx` table, whose scancodes are 24-bit, and the failure is
silent.

`ir-keytable -t` block-buffers its output into a pipe. A capture piped to another command
loses its output on a timeout, so write the capture to a file.

The vendor maps the center (OK) button to `KEY_REPLY`, a translation through an Android
keylayout. If a frontend expects `KEY_OK` or `KEY_ENTER`, remap those entries.

#### Configuration

The receiver is board configuration and needs no kernel patch. The device tree adds the
`gpio-ir-receiver` node and its pinmux. The kernel needs `RC_CORE`, `RC_DECODERS`,
`IR_NEC_DECODER`, `IR_GPIO_CIR` and `LIRC`. Use `ir-keytable` to load the map onto the rc
device whose driver is `gpio_ir_recv`, not onto a fixed rc index. A USB remote on another rc
index then keeps its own map.

### HDMI-CEC

HDMI-CEC is a control wire on the HDMI cable, separate from the IR remote. Mainline supports
it on the RK3576 inside the DW-HDMI-QP controller that drives video. The SoC dtsi's
`hdmi@27da0000` already declares the `cec` interrupt and the `vo0_grf` CEC mux, so the board
needs only `&hdmi` enabled.

#### Kernel configuration

Beyond the CEC core and the DRM HDMI-CEC helper, the box-drives-TV direction needs one
symbol: `CONFIG_DRM_DW_HDMI_QP_CEC=y`. With it and HDMI enabled, the driver exposes
`/dev/cecN`. The TV-drives-box direction needs two more symbols, `CONFIG_RC_CORE=y` and
`CONFIG_MEDIA_CEC_RC=y`.

The CEC core creates an RC device only under `MEDIA_CEC_RC`. Without it the adapter does not
advertise `CEC_CAP_RC`, and the TV's key presses die in userspace. Kconfig does not offer
that symbol while `CEC_CORE=y` and `RC_CORE=m` (`depends on CEC_CORE=m || RC_CORE=y`). Set
both symbols together. A narrower config fragment that sets `RC_CORE=m` silently undoes it.

`cec-ctl` and `cec-follower` come from `v4l-utils`.

#### Box drives TV

With `/dev/cec0` present, the box controls a CEC-capable TV. One Touch Play (Image View On
and Active Source) wakes the TV and switches it to this input. Standby turns it off. This is
the direction a media frontend's CEC integration drives on play and stop.

The round trip works against an LG TV [measured, 2026-07-30]. The adapter comes up as driver
`dwhdmiqp-rockchip` on `27da0000.hdmi`, as `/dev/cec0`, at CEC 2.0. It reads its physical
address (`1.0.0.0`) from the sink EDID and claims a Playback logical address. Standby puts
the TV into standby, and the TV broadcasts `STANDBY (0x36)` to the bus. Image View On wakes
it, the TV accepts the box's Active Source announcement, and directed queries answer in
35-55 ms.

The physical address survives the TV being in standby, so waking a cold TV needs no address
re-acquisition. The `cec-ctl` measured needs an explicit `phys-addr=<addr>` for
`--active-source`:

```sh
cec-ctl -d /dev/cec0 --active-source phys-addr=1.0.0.0
```

Send standby directed rather than broadcast. `cec-ctl --to 0 --standby` sleeps the TV alone.
A broadcast also sleeps every other CEC device on the bus.

The IR remote's power key does not reach the TV on its own, because the remote talks to the
box over infrared. Bridging the two is software glue: map the key, and run
`cec-ctl --standby` from it.

#### TV drives box

The TV's own remote can drive the box through remote-control passthrough, with no IR
handset at all. Running `cec-follower -m` while pressing the TV remote captures
`USER_CONTROL_PRESSED`/`USER_CONTROL_RELEASED` pairs addressed to Playback Device 1
[measured]. The captured keys include `up`, `select` and `left`, so the TV does forward
them. The TV forwards keys only to the active source, so the box must claim Active Source
first. Delivery of those keys as input events through `MEDIA_CEC_RC` is untested on hardware.

No `--rc-passthrough` flag exists in v4l-utils 1.32.0. "Remote Control Passthrough Feature"
is only a help section for `--user-control-pressed`. The flag is also unnecessary, because
plain `--playback` already reports `Logical Addresses: 1 (Allow RC Passthrough)`.

#### Faults on the sink side

A sink reporting `Physical Address: 1.0.0.0` proves nothing about CEC support, because the
field is mandatory in EDID. An LG monitor on the same cable never acknowledges a message
(`Tx, Not Acknowledged (4), Max Retries`). LG ships CEC on its televisions and not on its
desktop monitors.

A TV that times out `GIVE_OSD_NAME` or `GIVE_CEC_VERSION`, or rejects CEC 2.0
`REPORT_FEATURES` as `unrecognized-op`, is omitting optional messages. The box's transmits
are acknowledged, so these are not faults on the box.

### Suspend configuration

The board suspends to idle (s2idle) and resumes once the Wi-Fi driver is quiesced, which
[the Wi-Fi suspend notes](wifi-bluetooth.md#suspend-and-resume) cover. Three board facts go
with it.

Sleep must use `freeze`, not `mem`. The kernel offers `deep` in `mem_sleep` and selects it by
default, and the board never resumes from it. Under systemd, `SuspendState=freeze` selects
s2idle.

The wake source is the remote's POWER key, through gpio0 PD3 (`RK_PD3`). The RF receiver's USB side
leaves the remote-wakeup bit clear in its configuration descriptor. It therefore has no
`power/wakeup` node and cannot wake anything over USB. The board has no RTC to wake it
either (see [peripherals and controls](hardware-reference.md#peripherals-and-controls)).

The box has no physical power button, and after `poweroff` only a supply cycle brings it
back. Mapping the power key to suspend (logind's `HandlePowerKey=suspend`) suits
it for that reason.

Two SoC-level settings go with the board's. Keep deep CPU idle off under rkbin BL31 v1.24
(see [deep idle under BL31 v1.24](../boot-firmware.md#deep-idle-under-bl31-v124)). Both dwc3
controllers need `snps,dis_u2_susphy_quirk`, or the xHC errors on every resume (see
[xHC errors on resume](../corrections-and-gotchas.md#xhc-errors-on-resume)).

## SoC-wide traps

Several traps met on this board hold for every RK3576 board, and the SoC documents own
them:

| Trap | Owner |
|---|---|
| Mainline u-boot needs no rkbin BL32, and rkbin's BL32 is not an ELF | [OP-TEE and BL32](../boot-firmware.md#op-tee-and-bl32) |
| The blob variables must reach the defconfig step as well as the compile | [Blob variables at configuration time](../boot-firmware.md#blob-variables-at-configuration-time) |
| The SARADC download key needs two fixes to fire | [The SARADC download key](../boot-firmware.md#the-saradc-download-key) |
| Under rkbin BL31 v1.24 a reboot can request maskrom and nothing else | [Reboot modes](../boot-firmware.md#reboot-modes) |
| A CPU in `CPU_SLEEP` can miss its wakeup under rkbin BL31 v1.24 | [Deep idle under BL31 v1.24](../boot-firmware.md#deep-idle-under-bl31-v124) |
| A full rockusb read truncates at 32 MiB | [The 32 MiB read wall](../rockusb-reference.md#the-32-mib-read-wall) |
| `bcdUSB` does not tell maskrom from the loader | [Mode detection](../rockusb-reference.md#mode-detection) |
| The loader's flash-info manufacturer byte does not identify the eMMC | [Flash information](../rockusb-reference.md#flash-information) |
| The HDMI path has register-level preconditions that all fail alike | [The bring-up contract](../display-reference.md#the-bring-up-contract) |
| A bootloader that lights the panel must quiesce the VOP | [Handing the display over to an operating system](../display-reference.md#handing-the-display-over-to-an-operating-system) |
| The RK3576 has no `io-domain` node to write | [IO power domains](../hardware-design-guide.md#io-power-domains) |
| Combo PHY 1 gates drd1's USB 2.0 | [USB controllers and combo PHY 1](../hardware-design-guide.md#usb-controllers-and-combo-phy-1) |
| Vendor regulator messages name no mainline requirement | [Vendor-driver log lines](../corrections-and-gotchas.md#vendor-driver-log-lines-are-not-mainline-requirements) |
| A late Panfrost probe SErrors without the GPU domain supply | [The GPU power domain needs its supply](../corrections-and-gotchas.md#the-gpu-power-domain-needs-its-supply) |
| The xHC errors on every resume without the U2 suspend-PHY quirk | [xHC errors on resume](../corrections-and-gotchas.md#xhc-errors-on-resume) |
| Capping a dwc3 port needs both overrides, and drd1 keeps its SuperSpeed PHY | [Capping a dwc3 port to High Speed](../corrections-and-gotchas.md#capping-a-dwc3-port-to-high-speed) |
| A board-level `pinctrl-0` drops the inherited groups | [The pinctrl-0 override](../corrections-and-gotchas.md#a-board-level-pinctrl-0-replaces-the-inherited-groups) |
| `rmmod` of the video decoder hangs | [rkvdec cannot be unloaded](../corrections-and-gotchas.md#rkvdec-cannot-be-unloaded) |
| A module built by another toolchain fails its symbol CRCs | [Module toolchain](../corrections-and-gotchas.md#a-module-built-by-another-toolchain-does-not-load) |
| GPIO banks bind in probe order | [rk3576.dtsi has no gpio aliases](../corrections-and-gotchas.md#rk3576dtsi-has-no-gpio-aliases) |
| `rockchip,sai-tx-route` and `SAI_PATH_SEL` | [SAI transmit routing](../corrections-and-gotchas.md#sai-transmit-routing) |
| The NPU rail needs a consumer, and `rocket` is not one | [The NPU rail](../npu-reference.md#the-npu-rail) |
| The NPU power domain needs a reset pulse and a settle at power-on | [Power-on sequencing](../npu-reference.md#power-on-sequencing) |
| The `rga3` init WARN is cosmetic, and `dma_heap` buffers are cached | [RGA](../video-reference.md#rga) |

## Sources

Grades follow the device-ref profile, and every `[measured]` result is from the reference
unit. Keys used in this document:

- `[THREAD]`: The Armbian forum H96 MAX M9 enablement thread (topic 40483).
- `[LOG]`: The factory boot logs posted to that thread (`boot1.txt`, `boot2.txt`,
  `boot3.txt`, `dmesg-19.0.txt`), and serial-console captures of the reference unit. The
  SuperSpeed contact-loss loop is in one of the reference unit's mainline captures.
- `[DTB]`: The factory device tree posted to that thread, `h96-m9_original.dtb`, decompiled.
- `[DTSI]`: Mainline `rk3576.dtsi` and `rk3576-evb1-v10.dts` from the 7.1 tree.
- `[SPEC]`: The manufacturer's product specification and user manual.
- `[HDG]`: Rockchip's RK3576 Hardware Design Guide V1.1, summarized in the
  [hardware design guide](../hardware-design-guide.md).
