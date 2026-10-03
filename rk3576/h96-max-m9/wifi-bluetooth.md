# H96 MAX M9 Wi-Fi and Bluetooth

This document covers the AICSemi AIC8800D80 Wi-Fi and Bluetooth module of the H96 MAX M9
under a mainline kernel. It describes the out-of-tree driver, what a working `wlan0` and a
working `hci0` need, and the driver's behavior across system suspend. Wireless varies by
production batch: earlier units carry a Broadcom AP6275P instead [consensus], which this
document does not cover. Identify the part by its SDIO vendor ID, as
[the Wi-Fi label correction](corrections-and-gotchas.md#the-wi-fi-label-ap6275p-is-boilerplate)
explains.

## The module

The AIC8800D80 attaches Wi-Fi over SDIO, on the controller at `0x2a320000`, and Bluetooth
over `uart4` [measured]. It identifies on the SDIO bus by AICSemi vendor ID `0xC8A1`. The
Bluetooth controller shares the module's identity: its address is the Wi-Fi MAC plus one.
The [networking section of the hardware reference](hardware-reference.md#networking) lists
the module's pins and control lines.

## The driver

The AIC8800 driver is out of tree permanently. AICSemi's mainline submission was rejected
and will not merge. The submission is the raw RivieraWaves (rwnx) BSP, with a private netlink
ABI, Wireless Extensions and a reimplementation of cfg80211 internals. The maintained fork
is `radxa-pkg/aic8800`.

The SDIO driver is two modules, built out of tree against the board's kernel. `aic8800_bsp`
claims SDIO function 2, powers the chip and downloads the firmware. `aic8800_fdrv`, the
FullMAC module, drives function 1 and creates `wlan0`. The driver measured is
`rwnx 6.4.3.0`, release `2.0.14`, git revision `241c091M`.

On a 7.1 or later kernel, the fork's SDIO tree needs two further changes. One is a cfg80211
shim for the 7.1 `net_device *` signature change, which the fork carries only on its USB
tree. The other is a PM notifier that quiesces the driver across system suspend, described
under [Suspend and resume](#suspend-and-resume). Wi-Fi on both bands works from such a
build [measured].

## Wi-Fi

Four separate problems each block a working `wlan0`, and all four fixes are needed
[measured]. With them, the board runs 2.4 GHz and 5 GHz.

### SDIO enumeration at boot

Without a settled reset and the bus I/O rail, the card fails to enumerate at boot with
`mmc: Failed to initialize a non-removable card`. It comes up only on a manual host rebind,
the mark of a power-on timing race. Two board device-tree changes fix it:

- `&sdio` takes `vqmmc-supply = <&vccio_sd_s0>`, the RK806 PLDO5 1.8 V bus-I/O rail that
  switches SDR104 signaling. The SD host already uses that rail.
- The SDIO power sequence's `post-power-on-delay-ms` rises to 500.

The card then enumerates at boot as an `SDR104 SDIO card`.

### Firmware pairing

The firmware must be the board's own factory build. The driver and its firmware are a
matched pair, because the compiled-in `patch_tbl` and the `NEW_PATCH_BUFFER_MAP` path are
coupled to one firmware patch level. A mismatched `6.4.3.0` firmware fails the `cmd:1037`
wifi-start handshake. At best the handshake times out gracefully, and at worst the kernel
crashes.

The factory `aic8800_fdrv.ko` reports the same git revision as the driver above. The
firmware in the device's own `/vendor/etc/firmware/` is therefore the exact match, and it
can be extracted from an eMMC dump. The GitHub mirrors (goecho, shenmintao, gtxaspec and
ronnyf) carry different patch levels behind the same version string. Use the factory
`_8800d80_u02` set. A matching version string is not enough, because the firmware binary
itself has to pair.

### The function-1 bind on 7.x

By default the FullMAC module registers on the SDIO bus and lets the driver core bind it to
function 1. The 7.x driver core rejects that bind with `-EBUSY` ("Resources present before
probing", `drivers/base/dd.c`), so `wlan0` is never created. The driver was written against
6.1.

`CONFIG_AIC8800_FDRV_NO_REG_SDIO=y` takes the driver's alternate path. The BSP module hands
function 1 to the FullMAC module through a direct probe call (`get_sdio_func` and
`aicwf_sdio_probe_`). That call bypasses the bus bind and its resource check. The path is a
configuration flag rather than a code change, and it leaves the driver without suspend
callbacks (see [Suspend and resume](#suspend-and-resume)).

### 5 GHz and regulatory

With `wlan0` up, scans return 2.4 GHz only. The driver flags its wiphy
`REGULATORY_WIPHY_SELF_MANAGED` and applies an internal domain. That domain defaults to the
world domain "00", which disables the driver's 5 GHz band. The driver ignores the system
regulatory settings and even an explicit `iw reg set`.

Three settings together enable 5 GHz:

- The `custregd=0` module parameter (`options aic8800_fdrv custregd=0`) drops the
  self-managed override, so the driver honors standard cfg80211 regulatory.
- `CONFIG_CFG80211_REQUIRE_SIGNED_REGDB` unset lets cfg80211 load the regulatory database
  unsigned. A self-built kernel lacks the key that signed the distribution's
  `regulatory.db`, so cfg80211 otherwise rejects it ("malformed or signature is
  missing/invalid") and stays in the world domain.
- A country setting, for example `options cfg80211 ieee80211_regdom=US`, selects the
  domain.

With all three, the 5 GHz channels enable (36-48 at full power, DFS above), and 5 GHz access
points appear in a scan [measured].

## Bluetooth

The AIC8800D80's Bluetooth is HCI over `uart4` at 1.5 Mbaud, not SDIO. A registered
`/dev/ttyS4` and a plain attach still get no answer. The attach command and an
edge-triggered wake stand between the UART and a live controller [measured]. With both,
`hci0` comes up `UP RUNNING` at LMP 5.4, and LE and classic scans return devices.

### The Bluetooth firmware

The Bluetooth firmware arrives with the Wi-Fi bring-up, and there is no separate Bluetooth
firmware to fetch. `aicbsp_set_subsys` in `aic8800_bsp` downloads the whole firmware image
on the first subsystem power-on, and that image carries the Bluetooth patch. At every boot
the log prints `aicbt_patch_table_load bt uart_baud[..]:1500000 uart_flowctrl[..]:1` and a
`bt patch version` [LOG].

With `CONFIG_AIC8800_SDIO_BT` off, the driver header sets the Bluetooth transport to UART at
1.5 Mbaud with flow control (`AICBT_BTPORT_UART`). The controller therefore speaks HCI on
`uart4` as soon as Wi-Fi is up. The SDIO-BT transport's Kconfig entry is
`depends on BROKEN`, and that transport and the combined `fmacfwbt` image are a dead end.
UART Bluetooth needs neither.

### The UART

The factory Bluetooth serial node is `serial@2ad70000` [DTB]. It is mainline `uart4`,
exposed as `/dev/ttyS4` through the `serial4` alias, on the uart4 m1 pin group. TX and RX
are gpio1 PC4 and PC5, CTS is PC3, and RTS is PC2. Enabling `&uart4` with `uart-has-rtscts`
and those pins is the whole device-tree change. The factory node lists only the transfer
and CTS pins and drives RTS as a GPIO, and hardware auto-RTS works as well.

### The attach

`btattach -P h4` registers `hci0`, but the controller never answers, and `HCI_Reset` times
out (`Opcode 0x0c03 failed: -110`). The working attach uses `hciattach` with the generic
`any` H4 line discipline:

```sh
hciattach -n -s 1500000 /dev/ttyS4 any 1500000 flow nosleep
```

Both options matter. `flow` enables hardware RTS/CTS, and `nosleep` turns off host-side
HCILL sleep management.

### The wake edge

The controller leaves the firmware load in low-power sleep and stays silent until the host
asserts two lines. They are BT_REG_ON (module `PWR_BT`, gpio1 PC7) and HST_WAK_BT
(host-wake-bluetooth, gpio1 PD4), both active-high. Holding them high is not enough. The
low-to-high transition, after the firmware has loaded, is what wakes the part.

A device-tree gpio-hog or a fixed regulator drives the pins high at gpio-controller probe
(~0.2 s). The Bluetooth patch loads at ~1 s, so no edge follows it and the controller never
wakes. With both pins hogged high, `/sys/kernel/debug/gpio` shows them `out hi` while `hci0`
stays dead with `-110` [measured]. Driving the same pins high from an undriven state at
`multi-user` (~30 s) wakes the controller every time. Driving the wake line low fails.

### The userspace bring-up

No in-tree serdev driver exists for this part, and the wake edge must come late, so the
bring-up runs in userspace. The board device tree enables `uart4` and names BT_REG_ON and
HST_WAK_BT without driving them. A service started after the firmware load holds the lines
high with `gpioset` and then runs `hciattach`. The two tools come from the `gpiod` and
`bluez` packages.

The service starts `gpioset` in the background, so it lives in the service's cgroup, and
then `exec`s `hciattach`. The service manager reaps `gpioset` on stop. A restart re-drives
the lines from released, which gives a fresh wake edge. Lines held high defeat Bluetooth
low-power, which suits an always-on box.

A bring-up that drives the lines through `gpiochip1` relies on the GPIO controllers
registering in bank order, because gpiolib numbers its chips in registration order. On the
reference unit the controllers probe in device-tree order in every mainline boot capture
[LOG]. The chip under `/sys/bus/platform/devices/2ae10000.gpio/gpiochip*` does not depend on
that order (see
[rk3576.dtsi has no gpio aliases](../corrections-and-gotchas.md#rk3576dtsi-has-no-gpio-aliases)).

### Bluetooth audio

Bluetooth audio on a Debian desktop takes one more package, `libspa-0.2-bluetooth`, the
PipeWire Bluetooth plugin. Both `wireplumber` and `pipewire-pulse` list it as Suggests, so
no desktop metapackage pulls it in, even with Recommends enabled. Without it, a headset
pairs and connects and then offers no sink. With it, A2DP earbuds pair and play under
Cinnamon [measured, 2026-08-07].

## Suspend and resume

With the AIC8800 driver loaded, s2idle resume deadlocks unless the driver is quiesced.
Under `CONFIG_AIC8800_FDRV_NO_REG_SDIO`, the driver claims SDIO function 1 without
registering an `sdio_driver`. Its `dev_pm_ops` (`aicwf_sdio_suspend` and
`aicwf_sdio_resume`) attach only to that unregistered structure. The driver is therefore
never told about system suspend. It never raises its `is_bus_suspend` flag and never
quiesces.

The failure is a resume hang, not a suspend failure. During s2idle the dwmmc host is
noirq-suspended and its clock is cut. The driver's contexts keep issuing SDIO: the bus-TX
and power-control kthreads, and the temperature-poll worker `aicwf_temp_ctrl_worker`. That
worker sends a get-temp command straight over SDIO rather than through the TX path. The
first such access to the powered-down controller aborts:

```
Internal error: synchronous external abort ... dw_mci_card_busy
... aicwf_sdio_writeb [aic8800_fdrv] ... aicwf_temp_ctrl_worker
```

The faulting context dies holding the MMC host claim. On resume, `mmc_sdio_resume` then
blocks forever in `__mmc_claim_host`, `dpm_resume` never completes, and userspace never
thaws. The box wakes and answers ping, but it is wedged and needs a power cycle. A stock
kernel reports nothing. A kernel built with `CONFIG_DPM_WATCHDOG` panics on `mmc mmc2:390b`
after the timeout, which names the culprit [LOG].

The hang correlates with HDMI being plugged, and it is not an HDMI or CEC fault. In the
same capture the HDMI resume returns 0 in ~120 ms, and CEC logs one bounded message
timeout. The correlation is timing. Whether the driver's power-save or temperature work
lands inside the suspend window is race-sensitive, and the extra display-resume latency
shifts it. Unloading Wi-Fi (`rmmod aic8800_fdrv aic8800_bsp`) makes s2idle resume clean with
HDMI still plugged [measured].

The fix delivers the missing PM to the direct-probe path. A `pm_notifier` registered in the
NO_REG_SDIO branch of `aicwf_sdio_register` acts on `PM_SUSPEND_PREPARE`:

- It raises `is_bus_suspend`, and the TX and power-control kthreads poll that flag, as the
  OOB IRQ thread already does.
- It stops the temperature poll with the driver's own `aicwf_tp_ctrl_deinit`.
- It parks the chip.

On `PM_POST_SUSPEND` the notifier undoes all three. With it, s2idle suspend and resume
complete cleanly with Wi-Fi loaded and HDMI plugged, driven from the remote's POWER key
[measured, 2026-07-30, 7.1.5]. Wi-Fi works afterward, and a post-resume scan sees access
points.

The board's sleep state and wake source are under
[suspend configuration](corrections-and-gotchas.md#suspend-configuration). The BL31
deep-idle wake miss is a separate fault (see
[deep idle under BL31 v1.24](../boot-firmware.md#deep-idle-under-bl31-v124)).

## Sources

Grades follow the device-ref profile, and every `[measured]` result is from the reference
unit. The `[consensus]` grade rests on the Armbian forum H96 MAX M9 thread (topic 40483).
Keys used in this document:

- `[LOG]`: Serial-console captures of the reference unit under mainline, the suspend and
  resume cycles among them.
- `[DTB]`: The factory device tree posted to that thread, `h96-m9_original.dtb`, decompiled.
