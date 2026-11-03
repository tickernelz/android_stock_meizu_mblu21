# Meizu mBlu 21 (M416H) — stock firmware images

Stock partition images read straight from a physical phone through the Unisoc BootROM
(download mode), kept here so the phone can be returned to its factory state without hunting
for a vendor package Meizu never released.

[Bahasa Indonesia](docs/id/README.md) (optional translation; this English text is the
reference).

| | |
|---|---|
| Model | Meizu mBlu 21, M416H (board `ums9230_6h10`) |
| SoC | Unisoc UMS9230 / Tiger T606 |
| Storage | UFS, A/B, dynamic partitions (`super`) |
| Stock build | Android 14, `U01005` (`1745906188`), security patch 2025-04-05 |
| Source | dump of 42 partition images from this phone on 2026-10-31 (41 `.bin` + `super`), hashed at dump time |
| Release | [`mblu21-stock-U01005-20261031`](../../releases/tag/mblu21-stock-U01005-20261031) |

## Documents

| File | Contents |
|---|---|
| [docs/FLASH-STOCK.md](docs/FLASH-STOCK.md) | Full restore to stock: host prerequisites, entering download mode, per-partition `spd_dump` commands on Windows and Linux, write order, verification, troubleshooting |
| [docs/UNLOCK-RELOCK.md](docs/UNLOCK-RELOCK.md) | Bootloader unlock and relock: factual status, the community tool used, relock conditions that do not leave the phone dead |
| [docs/ROLLBACK.md](docs/ROLLBACK.md) | Going back to stock from LineageOS or another custom ROM |
| [docs/CUSTOM-ROM.md](docs/CUSTOM-ROM.md) | Installing LineageOS 24.0 and recovery (OrangeFox, SHRP): what works, what does not yet |

## Release contents

Every image is a GitHub release asset, not a file inside git (one asset is capped at 2 GiB,
so `super.bin` is split into three parts).

| Group | Files |
|---|---|
| Boot chain | `boot_a.bin`, `init_boot_a.bin`, `vendor_boot_a.bin`, `dtbo_a.bin`, `dtb_a.bin`, `vbmeta_a.bin`, `vbmeta_system_a.bin`, `vbmeta_system_ext_a.bin`, `vbmeta_vendor_a.bin`, `vbmeta_product_a.bin`, `vbmeta_odm_a.bin` |
| System | `super.bin` (split into `super.bin.part0`–`part2`): `system`, `system_ext`, `product`, `vendor`, `odm`, `vendor_dlkm`, `system_dlkm` |
| Bootloader | `splloader.bin`, `uboot_a.bin`, `uboot_log.bin`, `sml_a.bin`, `trustos_a.bin`, `teecfg_a.bin`, `pm_sys_a.bin` |
| Modem / DSP / NV | `l_modem_a.bin`, `l_ldsp_a.bin`, `l_gdsp_a.bin`, `l_agdsp_a.bin`, `l_deltanv_a.bin`, `l_fixnv1_a.bin`, `l_fixnv2_a.bin`, `l_runtimenv1.bin`, `l_runtimenv2.bin`, `prodnv.bin` |
| Calibration / other | `persist.bin`, `calinv.bin`, `common_rs1_a.bin`, `common_rs2_a.bin`, `hypervsior_a.bin`, `avbmeta_rs_a.bin`, `sysdumpdb.bin`, `logo.bin`, `fbootlogo.bin`, `misc.bin`, `miscdata.bin`, `metadata.bin`, `pgpt.bin` |
| Logical images from `lpunpack` | `system_a.img`, `vendor_a.img`, `product_a.img`, `system_ext_a.img`, `odm_a.img`, `vendor_dlkm_a.img`, `system_dlkm_a.img` |
| Tables & summary | `partition_1793459407.xml` (partition table from the phone), `dump_summary.json` |
| Tools | `tools-windows.zip` (`spd_dump.exe`, `gen_spl-unlock.exe`, `chsize.exe`, FDL loaders, Unisoc driver), `tools-linux.zip` (the same Linux binaries + udev rule) |
| Checksums | `SHA256SUMS-full_dump.txt` (hashes of the dump as taken, whole `super.bin`), `SHA256SUMS-logical.txt` (hashes of the logical images) |
| Release notes | `RELEASE-NOTES.md` (release summary; not part of the dump, not covered by the checksums) |

`userdata` is not in this release: that partition holds personal data and was never
dumped.

File names match the partition names in `/dev/block/by-name` (e.g. `boot_a.bin` →
partition `boot_a`). The naming exceptions (`uboot_a.bin` is written as `uboot`,
`l_runtimenv*` has no slot suffix, and two files are not partitions) are listed in
[docs/FLASH-STOCK.md §0](docs/FLASH-STOCK.md#0-release-contents-and-partition-names).

## Verify before writing anything

Linux:

```bash
gh release download mblu21-stock-U01005-20261031 -R tickernelz/android_stock_meizu_mblu21
cat super.bin.part0 super.bin.part1 super.bin.part2 > super.bin
sha256sum -c SHA256SUMS-full_dump.txt
sha256sum -c SHA256SUMS-logical.txt
```

Windows PowerShell:

```powershell
Get-FileHash .\boot_a.bin -Algorithm SHA256
Select-String -Path .\SHA256SUMS-full_dump.txt -Pattern boot_a
```

## Warnings

- **`fastboot flash` cannot write to this phone.** On the M416H, `fastboot flash` and
  `fastboot boot` answer `Sending ... OKAY` and then fail at the `Writing` stage
  (`Status read failed (No such device)`) or never run. Every write in these guides goes
  through Unisoc download mode (`spd_dump`). `fastboot` is only useful here for `getvar` and
  `reboot`.
- **Writing stock partitions erases data.** Restoring the firmware does not restore personal
  files; `userdata` is not in this release and cannot be recovered from here.
- **Do not write device-specific partitions to another phone.** `prodnv`, `l_fixnv1_a`,
  `l_fixnv2_a`, `l_runtimenv1`, `l_runtimenv2`, `persist`, and `miscdata` hold this phone's
  production data; `l_runtimenv1.bin`, among others, contains an IMEI-style 15-digit number.
  Writing them to another device is illegal in many countries and damages that device.
- **Not every NV partition can be restored with `w`.** This tool erases `l_runtimenv*`
  instead of writing it, rewrites `l_fixnv1_a` as an NV packet (not the original bytes), and
  skips `calinv` entirely. Details in
  [docs/FLASH-STOCK.md §4.4](docs/FLASH-STOCK.md#44-nv-partitions-what-w-actually-does).
- **Two pairs of NV partitions hold the same bytes.** `l_fixnv1_a` = `l_fixnv2_a` and
  `l_runtimenv1` = `l_runtimenv2` in this dump; both are full-size and not shifted copies.
- These images do not belong to this project: the rights are Meizu's and Unisoc's, and they
  are republished only so device owners can restore their own devices. No warranty of any
  kind.
