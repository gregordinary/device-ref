# Gemtek W1700K OpenWrt status

This page states what OpenWrt drives on the W1700K, one row per subsystem. Each result
carries its evidence grade and the date of the latest report it rests on. The
[hardware reference](hardware-reference.md) states what the board is. The
[AN7581 network reference](../network-reference.md) and the
[MT7996 reference](../../mt7996/wifi-reference.md) describe the mechanisms behind the rows.

## Builds

OpenWrt supports the board in its main branch, as `gemtek_w1700k-ubi`, from 2026-03-10
[source][HISTORY]. The 25.12 release branch carries no W1700K device tree [source][TREE].
Snapshot images are therefore the only official builds.

| Property | Value |
|---|---|
| Kernel | Linux 6.18 since 2026-06-04. The snapshot of 2026-11-07 runs 6.18.60 |
| Package manager | `apk` |
| Device packages | `airoha-en7581-mt7996-npu-firmware`, `fitblk`, `kmod-hwmon-nct7802`, `kmod-mt7996-firmware`, `wpad-basic-mbedtls`, `rtl826x-firmware`, `rtl8261c-firmware` |
| Layout | Compat version 2.0 (see [layout versions](boot-and-flash.md#layout-versions)) |

Sources: [source][IMAGE][HISTORY][SNAPSHOT].

Community builds carry patches that main does not. Their results do not transfer to the
official snapshot, and the reverse. A row below states which a result holds for.

## Evidence grades

`[consensus]` means more than one poster in the forum threads under [Sources](#sources)
reports the result, and `[reported]` means one does. No result here is measured on a
reference unit. A row's date is the date of the newest post or commit it rests on.

## Subsystem status

| Subsystem | Status on the official snapshot | Evidence | Reference |
|---|---|---|---|
| Boot, console, sysupgrade | Works | [consensus][T4046][T4037], 2026-11 | [Boot chain](boot-and-flash.md) |
| NAND and UBI | Works. Linux leaves the vendor bad-block area alone | [source][DTS] | [Bad-block area](boot-and-flash.md#the-vendor-bad-block-area) |
| `wan`, 10G, HW1.x | Works | [consensus][T3453][T4046], 2026-11 | [Ethernet](hardware-reference.md#ethernet) |
| `lan2`, 10G, HW1.x | Fails to link at some boots on some units | [reported][T3879][T3883], 2026-08 | [PLL band search](../network-reference.md#erratum-1-the-10g-link-does-not-come-up) |
| `wan` and `lan2`, 10G, HW2.1 | Works from about 2026-09-24 | [consensus][T3961][B477], 2026-09 | [The 10G PHYs](hardware-reference.md#the-10g-phys) |
| `lan3`, `lan4`, 1G | Works | [consensus][T3453][B166] | [Ethernet](hardware-reference.md#ethernet) |
| VLAN-aware bridge across `lan2` and the switch ports, with Wi-Fi | Works from commit `5808e5db4be0`, 2026-10-05 | [reported][B500][B501] | [Bridging across GDMs](../network-reference.md#bridging-across-the-switch-and-a-gdm) |
| PPPoE on a tagged WAN | Works | [reported][T3854], 2026-08 | |
| Hardware flow offload (PPE) | Unreliable: a kernel panic on 6.18.41, and IPv6 dropped with a tagged WAN | [reported][T3861][B302], 2026-08 | [PPE](../network-reference.md#ppe) |
| Hardware GRO | Corrupts IPv4 TCP streams without timestamps | [reported][T3997], 2026-09 | [Erratum 2](../network-reference.md#erratum-2-hardware-gro-corrupts-tcp-without-timestamps) |
| NPU Wi-Fi offload | Works from the firmware package of 2026-08-12 | [consensus][T3863][T3877][B392] | [NPU](../network-reference.md#npu) |
| Wi-Fi 2.4 and 5 GHz | Works | [consensus][T3630][T3649][T3868] | [MT7996](../../mt7996/wifi-reference.md) |
| Wi-Fi 6 GHz, 320 MHz | Works. The access point does not start with country `US` | [consensus][T3813][B216] for 320 MHz, [consensus][T2684][B426][B527] for `US`, 2026-10 | [Regulatory](../../mt7996/wifi-reference.md#regulatory-and-transmit-power) |
| Wi-Fi 7 multi-link (MLO) | Unstable: firmware timeouts and crashes after hours | [consensus][T3873][T3913][B424], 2026-09 | |
| Wi-Fi transmit power control | Ignored: the radios transmit at the table maximum | [consensus][T3597][T3620][T3622], 2026-06 | [Regulatory](../../mt7996/wifi-reference.md#regulatory-and-transmit-power) |
| Two bands carrying traffic in the same direction | Throughput falls to a fraction of one band's | [reported][B463], 2026-09 | |
| A power-saving client that drops off | All bands stall for about a minute | [reported][B198], 2026-06 | |
| CPU frequency scaling | Not supported: the driver fails to register, and the CPU runs at 1200 MHz | [consensus][T3453][T4046][B166], 2026-11 | [CPU](../system-reference.md#cpu) |
| EIP93 crypto engine | Works. Slower than the CPU below 512-byte requests and for hashes | [consensus][T3863][B374] | [Crypto](../system-reference.md#crypto) |
| Inline IPsec engine | Not supported. An RFC driver is posted | [reported][T3571][T3668] | [Crypto](../system-reference.md#crypto) |
| Fan and hardware monitor | Works, on the vendor fan curve | [consensus][T3489][B15], [source][BASEFILES] | [Cooling](hardware-reference.md#cooling-and-monitoring) |
| LEDs and reset button | Works. The 10G port LEDs from 2026-09-24 | [source][DTS][HISTORY] | [GPIO assignments](hardware-reference.md#board-gpio-assignments) |
| GNSS on ttyS2 | Works at 921600, set as 460800 without the baud fix | [reported][T3759][T3804], 2026-07 | [AG3352Q](hardware-reference.md#ag3352q-gnss) |
| EFR32 on ttyS1 | Works at 115200, set as 230400 without the baud fix. Zigbee NCP, OpenThread RCP and multiprotocol RCP firmware run after a BGAPI DFU flash | [consensus][T3718][T3803][T3805][T3962], 2026-09 | [EFR32MG21](hardware-reference.md#efr32mg21) |

`5808e5db4be0` fixes a Wi-Fi to LAN regression for layouts with 802.1Q subinterfaces in
separate bridges [reported][B501]. Earlier VLAN-filtering setups dropped DHCP from Wi-Fi
clients on some community builds [reported][B179][B182].

## Community builds

These builds were most discussed as of 2026-11-07 [consensus][B527][B530]:

| Build | Base | Differences from the snapshot |
|---|---|---|
| OpenWRT-fanboy `snapshot` | The official snapshot | PHY and NPU load fixes, a regulatory database that allows 6 GHz in the US, official kernel ABI |
| OpenWRT-fanboy `ubi2` | Main, rebuilt daily | Bridge flowtable offload, `lan2` link fixes, the regulatory patch, more packages |
| `-oc` variants of either | As above | CPU table shifted 200 MHz up at unchanged voltages |

The AN7581's [operating points](../system-reference.md#operating-points) give the panics an
`-oc` table causes on some units.

## Sources

Citation keys:

- `[TREE]`: OpenWrt's `openwrt-25.12` branch, `target/linux/airoha/dts/`, read 2026-11-07.
- `[HISTORY]`: the commit history of OpenWrt main's
  `target/linux/airoha/dts/an7581-w1700k-ubi.dts`, read 2026-11-07. It runs from
  `99307582de` "airoha: add support for Gemtek W1700K" (2026-03-10) to `18304dd747`
  "airoha: an7581: add PHY LEDs for the Gemtek W1700K" (2026-09-24).
- `[DTS]`, `[IMAGE]`, `[BASEFILES]`: OpenWrt main at commit `706052b20fd1`:
  `an7581-w1700k-ubi.dts`, `image/an7581.mk`, and `base-files/etc/init.d/airoha_fan`.
- `[SNAPSHOT]`: downloads.openwrt.org/snapshots/targets/airoha/an7581/, build
  `r38181-85e54359cf` of 2026-11-07: `profiles.json` and the sysupgrade image's FIT header.
- `[T<n>]`: post n of the OpenWrt forum thread "Quantum Fiber W1700k support", topic 222776,
  read 2026-11-07.
- `[B<n>]`: post n of the OpenWrt forum thread "Gemtek W1700K Community Builds", topic 249319,
  read 2026-11-07.
