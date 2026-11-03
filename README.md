# Meizu mBlu 21 (M416H) — stock firmware images

Dumped from a real M416H through Unisoc BROM (CVE-2022-38694 download mode) and re-uploaded
here so the phone can be restored without hunting for a vendor package.

- **Model**: Meizu mBlu 21, M416H — Unisoc UMS9230 (Tiger T606), UFS, A/B, dynamic partitions
- **Shipped build**: Android 14, `U01005`, security patch 2025-04-05
- **Source**: read back from the phone's own partitions, hashed at dump time

## Start here

[**docs/FLASH-STOCK.md**](docs/FLASH-STOCK.md) — step-by-step restore on Windows and Linux,
including how to enter download mode, the exact `spd_dump` commands that were executed on
this model, the write order, verification, and troubleshooting.

## Releases

Every image is a release asset (GitHub caps a single asset at 2 GiB, so `super.bin` ships as
three split parts plus the hash of the whole file).

| Release | Contents |
|---|---|
| `mblu21-stock-U01005-20261031` | 44 partition dumps, `super.bin` split, tool bundles, `SHA256SUMS-full_dump.txt` |

## What is inside

| Group | Partitions |
|---|---|
| Boot chain | `boot_a`, `init_boot_a`, `vendor_boot_a`, `dtbo_a`, `dtb_a`, `vbmeta_a` + the five chained `vbmeta_*` |
| System | `super` (system, system_ext, product, vendor, odm, vendor_dlkm, system_dlkm) |
| Bootloader | `splloader`, `uboot_a` |
| Device data | `persist`, `prodnv`, `misc`, `miscdata`, `metadata`, the `l_*` modem/DSP images, `pm_sys_a`, `trustos_a`, `sml_a`, `teecfg_a`, `calinv`, `common_rs*`, `hypervsior_a`, `avbmeta_rs_a`, `sysdumpdb`, `logo`, `fbootlogo`, `pgpt` |

`userdata` is **not** included: it holds personal data and was deliberately never dumped.

## Limits you should know

- `fastboot flash` does not work on this device (writes abort with `Write img fail!`). Use
  download mode; the guide explains both and says which one actually writes.
- These images are the property of Meizu/Unisoc, published only so owners can restore their
  own hardware. No warranty; you flash at your own risk.

## Hash everything

```bash
sha256sum -c SHA256SUMS-full_dump.txt
```
