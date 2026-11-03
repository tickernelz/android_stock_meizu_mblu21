# Rollback: back to stock from LineageOS or a custom ROM

> **ERASES DATA.** A rollback restores the factory software, not personal files.
> Save your data first (`adb pull /sdcard`), then run it with the phone owner's decision.

The folder layout is the same as in [FLASH-STOCK.md §2](FLASH-STOCK.md#2-download-the-release-lay-out-the-folder-verify):
the image folder contains a `tools` subfolder.

## 1. What changes when LineageOS 24.0 is installed

According to the LineageOS 24.0 release notes for this phone, the install writes:

| Partition | Written at install | Back to stock needs |
|---|---|---|
| `vendor_boot_a` | LineageOS `vendor_boot.img` (carries Lineage Recovery) | `vendor_boot_a.bin` |
| `super` | sparse `super.img` (logical partitions: system, system_ext, product, vendor, odm, vendor_dlkm, system_dlkm) | `super.bin` |
| `init_boot_a` | LineageOS `init_boot.img` (or a root-patched image) | `init_boot_a.bin` |
| `boot`, `dtbo`, `vbmeta*` | not changed by a normal install | not needed, unless they were overwritten |

## 2. Steps

Every command runs from the tool folder, with the phone in download mode
([FLASH-STOCK.md §3](FLASH-STOCK.md#3-enter-download-mode-brom)) before **every** line.

```bash
cd ~/mblu21-stock/tools
Q="exec_addr 0x65015f08 fdl fdl1-dl.bin 0x65000800 fdl fdl2-dl.bin 0x9efffe00 exec"

# 1. system: stock super (5.56 GB); repeat until "Write Part Done"
./spd_dump --wait 300 $Q timeout 600000 w super ../super.bin reset

# 2. boot chain
./spd_dump --wait 300 $Q w vendor_boot_a ../vendor_boot_a.bin reset
./spd_dump --wait 300 $Q w init_boot_a   ../init_boot_a.bin   reset
./spd_dump --wait 300 $Q w boot_a        ../boot_a.bin        reset
./spd_dump --wait 300 $Q w dtbo_a        ../dtbo_a.bin        reset

# 3. AVB + boot-control block
./spd_dump --wait 300 $Q w vbmeta_a ../vbmeta_a.bin reset
./spd_dump --wait 300 $Q w misc     ../misc.bin     reset
```

Windows: the same, without `./`, with `set Q=...` + `%Q%` and `..\` instead of `../`.

Evidence note: writing the stock partitions as above has been repeated several times on this
phone, and it booted Android 14 again every time.

## 3. Wiping the data of the previous system

The three partitions above are enough for the firmware, but `/data` is still encrypted with
the Android 17 key. The path used: **factory reset from stock recovery**, after
`vendor_boot_a` is stock again.

1. If stock Android cannot boot yet because `/data` comes from Android 17, the phone stops at
   the logo screen or asks for a reset; that is the sign the data must be formatted.
2. Enter stock recovery with `adb reboot recovery` (adb in recovery shows up as device
   `1782:d001`). The hardware key combination for factory recovery has not been measured for
   this guide, so do not rely on it.
3. Choose **Factory reset / Wipe data** → confirm → **Reboot system**.
4. Wait 2–4 minutes for the first boot.

Stock recovery on this phone is proven to boot (it appears as `adb` device `1782:d001`).

**Do not use `spd_dump e userdata` for this.** That command does not wipe `/data`: the tool
writes a `boot-recovery / --wipe_data` control block to `misc` and then **erases the
`persist` partition** (`common.c:1144-1154`), so device-specific calibration is lost and the
phone is sent to recovery; `/data` is not what gets cleaned. If `persist` has already been
erased, write `persist.bin` from this release back (this partition is written raw, byte for byte).

## 4. Verification

```bash
adb reboot                      # leave recovery first
adb shell getprop ro.build.version.release   # 14
adb shell getprop ro.build.display.id        # U01005 ...
adb shell getprop sys.boot_completed         # 1
```

After a stock restore there is no root, so partitions are compared by reading them back
through download mode (not `adb shell su`):

```bash
mkdir -p ~/mblu21-stock/readback
cd ~/mblu21-stock/tools
Q="exec_addr 0x65015f08 fdl fdl1-dl.bin 0x65000800 fdl fdl2-dl.bin 0x9efffe00 exec"
./spd_dump --wait 300 $Q path ../readback r boot_a r init_boot_a r vendor_boot_a r dtbo_a r vbmeta_a reset
cd ../readback && sha256sum boot_a.bin init_boot_a.bin vendor_boot_a.bin dtbo_a.bin vbmeta_a.bin
```

All five values must match the line for the same file in `SHA256SUMS-full_dump.txt`.
`super` is not compared per logical partition; compare it by reading `super` back
(`r super`, 5.56 GB) and matching its hash.

## 5. If it still does not boot

1. Rewrite **all** 42 partition images from the release, not only the three above;
   differences in modem/NV, `splloader`/`uboot`, or the `vbmeta` chain also block booting.
2. Check `vbmeta_a` and the `vbmeta_*` chain: this phone rejects any `vbmeta` that is not
   from Meizu and stops at the logo.
3. A phone that will not boot is not bricked: the BootROM is always reachable through
   download mode (Volume Up + plug in USB), so this release can always be written again.
