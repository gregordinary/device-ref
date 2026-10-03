# RK3576 rockusb reference

This document covers the Rockchip rockusb download protocol as the RK3576 presents it. It
describes the USB identity of the BootROM and loader gadgets, and how to tell maskrom from
a loader. It also covers the 32 MiB read wall in the rockusb read handler and the loader's
flash-information report. It ends with writes over rockusb and how to capture a full eMMC
image.

The document is SoC-level. The measurements come from the reference unit, an H96 MAX M9
with a 128 GB-class eMMC [measured]. The wire measurements were taken on 2026-07-16 with
two independent host implementations, pyrographer and an rkdeveloptool build, which
cross-check each other.

## The download gadget

Maskrom and loader mode present one USB identity, VID:PID `2207:350e` [measured]. The
BootROM downloads over the OTG0 USB 2.0 interface, as
[USB controllers and combo PHY 1](hardware-design-guide.md#usb-controllers-and-combo-phy-1)
describes. The rkbin SPL loader's loader-mode descriptors, condensed from `lsusb -v`, are:

```
idVendor 0x2207, idProduct 0x350e, bcdUSB 2.00, bcdDevice 2.23
iManufacturer "Rockchip", iProduct "USB download gadget", iSerial "Rockchip"
bDeviceClass 0, bMaxPacketSize0 64, one configuration
Configuration: bNumInterfaces 1, bmAttributes 0xc0 (self powered), MaxPower 2mA
Interface 0, alternate 0: class 0xFF (vendor), subclass 0x06, protocol 0x05
  EP 0x81 IN  bulk, wMaxPacketSize 512, bInterval 0
  EP 0x02 OUT bulk, wMaxPacketSize 512, bInterval 1
```

The interface class triple `0xFF/0x06/0x05` matches mainline u-boot's `f_rockusb` gadget
descriptor. True maskrom, a fresh strap before any upload, presents the identical triple
and the same bulk endpoint pair [measured]. The cached descriptor sets differ in these
fields:

| Field | Maskrom | Loader (rkbin SPL) |
|---|---|---|
| `bcdDevice` | `1.00` | `2.23` |
| String descriptors | Absent | Present |
| Power | Bus-powered, 400 mA | Self-powered, 2 mA |
| EP-OUT `bInterval` | Differs from the loader's | `1` |

The SoC identifies itself through `K_FW_GET_CHIP_VER`. The reply is 16 bytes: the SoC's
four ASCII digits, byte-reversed and zero-padded. On the RK3576 it reads `36 37 35 33`,
which is "6753" [measured].

u-boot's rockusb gadget enumerates at USB 2.0 high speed, even on a SuperSpeed connector
[measured].
Sustained rockusb read throughput measures 20-26 MiB/s with both host tools, and a 4 MiB
read is `cmp`-identical between them [measured].

## Mode detection

Classify the mode by capability: probe whether rockusb commands are answered. The low bit
of `bcdUSB` does not identify the mode on the RK3576.

The rkdeveloptool source and the Rockchip wiki read the low bit of `bcdUSB` as the mode
flag, even for maskrom and odd for a loader. With the rkbin SPL loader the flag never
flips, as measured on the reference unit:

- True maskrom (a fresh strap, before any upload) presents `bcdUSB 0x0200` and serves no
  rockusb command. `rfi` fails outright, because a bare BootROM has no DRAM and no storage
  stack.
- After `db` of the rkbin SPL loader, the board re-enumerates at a new bus address with
  `bcdUSB` still `0x0200`. It then serves rockusb fully.

rkdeveloptool's own `ld` labels the loader state `Maskrom`, while its own `rfi` and `rl`
work against it. Only the capability changes, never the flag.

The factory u-boot's `f_rockusb` sets the odd `bcdUSB 0x0201` correctly [measured]. The
broken even flag is therefore specific to the rkbin SPL loader rather than to the SoC. The
flag still cannot be trusted, because one of the two loaders an RK3576 board can run
misreports it. It also says nothing about whether reads are truthful: the correctly flagged
factory gadget carries [the 32 MiB read wall](#the-32-mib-read-wall).

The reliable classifier is a probe command. A loader answers `TEST_UNIT_READY` and `rfi`,
while maskrom's bulk endpoints are decorative and fault the first transfer. The descriptor
fields in [The download gadget](#the-download-gadget) are candidate list-time classifiers,
pending more boards. The interface class triple is not one.

### Upload behaviors

Two bootstrap behaviors look like failures:

- A retried `db` against an already running loader fails. The loader holds the connection
  and accepts no second upload. That failure is itself evidence that the first upload
  took, since a bare BootROM accepts an upload.
- An interrupted upload leaves a BootROM that accepts no fresh `db` either. Recovery is a
  power cycle back into maskrom with the strap reapplied, which on the reference unit means
  re-bridging the eMMC clock pads.

## The 32 MiB read wall

Reads over rockusb return constant `0xCC` fill at and above sector 65536 (exactly 32 MiB),
with a success status [measured]. Everything below reads truthfully. Every other read
path on the same board reads the whole part. Those paths are USB Mass Storage (`ums`), the
u-boot console's own `mmc read`, and a `dd` from the booted OS.

The wall lives in the rockusb `K_FW_LBA_READ` (opcode `0x14`) handler. That handler reads
sectors into a RAM buffer and ships the buffer back over USB. Past sector 65536 the buffer
is not filled. The caller receives the buffer's poison fill (`0xCC`) and a status that
reports success. The flash, the storage selection and the controller are not involved, and
the same code path is present in more than one Rockchip rockusb provider.

### Properties of the wall

- Boundary: sector 65536 (`0x10000`), exactly 32 MiB at 512-byte sectors. Below it is real
  flash, and at and above it is constant `0xCC`.
- Silent: the device reports the read as successful. Nothing in the status channel flags
  the fill, so a backup taken this way looks complete.
- Poison: `0xCC` is the value of a buffer the handler never filled, not data. Unwritten
  flash reads `0x00`, and a full image read through the block layer shows `0x00`, never
  `0xCC`, in unwritten regions.
- Unbounded: a probe at sector 8388608 (the 4 GiB point) is still `0xCC`, as is everything
  past the boundary all the way up.
- Not a storage mis-selection: `K_FW_READ_FLASH_INFO` (rkdeveloptool `rfi`) reports the full
  244285440-sector (116.48 GiB) device. The loader sees the whole part and serves fill above
  32 MiB anyway.
- Deterministic: every read returns the same `0xCC`, so a dump-then-re-read comparison
  cannot detect it. Only a check that knows about the wall detects it.

### Read paths by access mode

| Access mode | Reads flash through | Wall | Notes |
|---|---|---|---|
| Maskrom BootROM | Serves no flash reads | Not applicable | Accepts only the download boot. Upload a loader with `db` first |
| rkbin SPL loader v1.12.108 (uploaded with `db`), rockusb | rockusb `K_FW_LBA_READ` | Yes | Boundary confirmed with both rkdeveloptool and pyrographer |
| Factory u-boot, rockusb gadget (`rockusb 0 mmc 0`) | `f_rockusb` `K_FW_LBA_READ` | Yes | Measured on the wire. Sector 0 matches the full image byte for byte. Sector 8388608 returns all `0xCC` where the flash holds `0x00` |
| Factory u-boot, UMS gadget (`ums 0 mmc 0`) | Block and mmc layer | No | The host sees a normal block device and reads the whole eMMC. This is the full-dump route |
| Any u-boot console, `mmc read` | The mmc controller directly | No | Truthful everywhere, with no gadget involved |
| Mainline u-boot loaded into RAM, rockusb | `f_rockusb` | Not measured | The one attempt bound to the empty SD host `mmc1` and failed with `request not queued to ep1in-bulk`. This gadget does not answer `K_FW_READ_FLASH_INFO` (`0x1a`). Use `ums` from this loader |
| Host `dd` over the UMS block device | Block layer | No | Captured the full, verified image |

Every path through the rockusb read handler hits the wall. No path through the block layer
or the controller does.

### Evidence that the read path is the cause

Three independent lines of evidence place the wall in the rockusb read path rather than in
the flash:

- Two different rockusb providers hit it: the rkbin usbplug loader and the factory
  u-boot's `f_rockusb`. They are separate binaries that share Rockchip's rockusb read
  code.
- The block layer does not hit it. `ums` and console `mmc read`, on the same u-boot and the
  same eMMC, read all 116.48 GiB truthfully.
- The captured image shows the flash is sound. Sector 65536, the exact wall point, holds
  real AArch64 boot code in the full dump.

The signature, `0xCC` with a success status, is that of a handler that returns its read
buffer whether or not the underlying read filled it.

### Reads that cross the boundary

A read that crosses the boundary can lose data below it too. How much depends on the read
shape the host tool issues [measured]:

- rkdeveloptool, reading 8 sectors from LBA 65532, flips per sector. Sectors 65532-65535
  are real, and `0xCC` starts at sector 65536, at byte `0x800` of the reply.
- pyrographer, reading 16 sectors from LBA 65528 in a single command, gets the entire
  window as `0xCC`. That includes sectors 65528-65535, which hold real boot data in the
  flash. The handler poisons the whole command once its range reaches past 65536.

To read up to the wall over rockusb, keep each read window entirely below sector 65536. A
window that straddles the boundary can lose the good sectors below it. A full dump goes
through `ums` or `dd`, which bypass the handler.

### Scope across boards and firmware builds

The wall is not specific to one board revision or firmware build:

- It reproduces on loaders that are not the board's own firmware. The rkbin SPL loader is
  a blob uploaded from the host, so its wall is independent of the firmware the board
  shipped with.
- An independent, older unit hits the same wall [reported][THREAD]. A 2024-manufacture
  board's full rockusb dump came back truncated and `0xCC`-filled. Its operator, cmuki,
  reported a ~200 MB file with the `baseparameter` region "full of cc" in a hex editor,
  without pinning the exact boundary. That unit ran older factory firmware than the
  reference unit.
- The community impression that RK3576 users do not hit the wall conflates writing with
  reading. Thread users flash images over rockusb routinely and successfully. The read path
  is less exercised, and the one documented full-read attempt hit the wall.

Whether the wall is a deliberate anti-clone cap or an incidental buffer limit is not
observable from outside. The evidence leans incidental [inferred]:

- Block-layer paths (`ums`, `mmc read`) on the same firmware read the whole part. A measure
  that leaves those open is no effective lockdown, since anyone who can reach rockusb can
  reach `ums`.
- The failure mode is an uninitialized-buffer poison with a success status, not a clean
  refusal or a zero fill. A deliberate feature more plausibly denies the read or returns
  zeros. Shipping stale poison reads like a window cap nobody guarded.
- 32 MiB is exactly the round number a read buffer or a sector-count cap lands on.

Whether any rockusb loader build lacks the wall is not established. Every build measured or
reported carries it, and the read handler appears shared across those builds. A bisection
across loader and firmware versions is the way to localize it.

### Working within the wall

- Do not trust a rockusb dump past 32 MiB. It completes successfully and is silently
  `0xCC`.
- Verify every rockusb-sourced dump before using it. Check the primary GPT, the backup GPT
  and the absence of `0xCC` filler, or compare a sector above 65536 against a known value.
- Take full images through the block layer: u-boot `ums` with a host `dd`, or a `dd` from
  the booted OS.
- Over rockusb, keep each read window entirely below sector 65536.

On the reference unit's factory layout, sectors 0-65535 are a complete bootloader-repair
kit. They hold the ID block (sector 64, `RKNS` magic), the primary GPT and every partition
up through the first 7 MiB of `boot`. The GPT backup header (last LBA) sits behind the
wall, so only the primary can be read or repaired over rockusb.

## Writes over rockusb

The read wall does not extend to writes through the vendor loader. Posters in the Armbian
thread flash full images through the vendor `RK3576_MiniLoaderAll.bin` and boot them
[consensus][THREAD]. They use `rkdeveloptool db` followed by `wl` or `write`, or the
Windows `RKDevTool`.

That evidence covers the vendor loader only. Do not assume the write path of a different
loader is honest from its read behavior. A responder that acknowledges reads it never
performed can acknowledge writes it never performed. Characterize `wl` against an
out-of-band read-back before relying on it.

The block layer is also a write route, and it needs no vendor loader. From any u-boot with
`ums`, run `ums 0 mmc 0` and write the image to the exposed block device with a host `dd`
[measured].

## Flash information

The loader's flash-information report (`K_FW_READ_FLASH_INFO`, rkdeveloptool `rfi`) on the
reference unit reads [measured]:

| Field | Value |
|---|---|
| Sectors | 244285440 of 512 bytes (116.48 GiB, rendered "119280 MB") |
| Block size | 512 KB |
| Page size | 2 KB |
| ECC bits | 0 |
| Access time | 40 |
| Manufacturer | `0x00`, rendered "SAMSUNG" by rkdeveloptool |

pyrographer's parse agrees with rkdeveloptool field for field, and the sector count is
byte-identical. The flash-ID query (`K_FW_READ_FLASH_ID`) answers the ASCII string
`"EMMC "`, bytes `45 4d 4d 43 20`. Mainline u-boot's `f_rockusb` does not answer
`K_FW_READ_FLASH_INFO`.

The manufacturer byte is not an identification. `0x00` is the first entry in the host
tool's manufacturer enum, and the kernel-reported part name `hDEaP3` follows SK hynix's
naming pattern. At least one of the two is wrong, and the enum rendering is the suspect.
Identify the eMMC by its kernel-reported part name or CID, not by the loader's
manufacturer byte.

## Capturing a full image

Two routes reach a full, truthful image. They differ only in how a block-layer reader is
started.

### From a board that boots

Interrupt the normal boot into the factory u-boot, and start the UMS gadget at its prompt:

```
ums 0 mmc 0
```

The eMMC appears on the host as a normal block device (`/dev/sdX`, 116.48 GiB on the
reference unit). Copy it with `dd`:

```sh
sudo dd if=/dev/sdX of=factory-full.img bs=4M status=progress conv=noerror,sync
```

### From maskrom

This route does not depend on a bootable eMMC, so it doubles as an unbrick route. From
maskrom, upload a RAM-bootable mainline u-boot with `db`, then run `ums 0 mmc 0` and `dd`
as above. The image is built with `CONFIG_ROCKCHIP_MASKROM_IMAGE` and runs entirely from
RAM, writing nothing to flash.

The RAM-boot image must advertise `SPL_TEXT_BASE` at byte 8 of its arm64 header, not
`CONFIG_TEXT_BASE`. With `CONFIG_TEXT_BASE` there, the BootROM places the download 8 MiB
high, and it never runs.

### Verifying a capture

The reference unit's full capture verifies [measured]. It is 125074145280 bytes, or
244285440 sectors (116.48 GiB), with a valid GPT and all 16 partitions. That includes
`super` and `userdata`, which both sit entirely behind the wall over rockusb.

A rockusb-based dump past 32 MiB (`rkdeveloptool rl`, pyrographer `dump`) completes
without error and is silently `0xCC`. Check every dump against the primary GPT, the backup
GPT and `0xCC` filler before trusting it.

## Sources

- [THREAD]: the Armbian forum's RK3576 enablement thread, topic 40483.
- The reference unit: an H96 MAX M9 (RK3576, 8 GB RAM, 128 GB eMMC). Its rockusb wire
  measurements were taken on 2026-07-16 with pyrographer and a local rkdeveloptool build,
  cross-checked against each other. The descriptor sets come from `lsusb -v` in true
  maskrom and again after a loader upload.
- The full factory capture, `h96-factory-full.img`, taken through u-boot `ums` and a host
  `dd`.
