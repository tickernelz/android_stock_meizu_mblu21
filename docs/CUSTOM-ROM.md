# Installing LineageOS, OrangeFox, and SHRP

This page describes the **actual status** as of 2026-11-03 and points to each project's
releases. If a project has not released an image yet, do not use generic instructions from
the internet: instructions for other Unisoc devices can leave this phone unable to boot.

> **ERASES DATA.** Installing another operating system formats `/data`. Make sure the stock
> dump has been downloaded and verified ([FLASH-STOCK.md](FLASH-STOCK.md)) before you start.

## 1. Bootloader rules on this phone (measured)

| Fact | Consequence |
|---|---|
| `vbmeta` is only accepted when signed by Meizu | a self-built `vbmeta`, or one signed with a test key → the phone stops at the Meizu logo; the LineageOS and recovery builds therefore **keep** the stock `vbmeta` |
| `vendor_boot` must carry Meizu's AVB blob | an image with the blob from the stock dump loads; an image without that blob stops at the Meizu logo (measured 2026-11-02 on SHRP/OrangeFox before the builds were resealed) |
| `boot` is verified | `boot` is left stock |
| `init_boot` is **not** verified | it can be patched for root |
| `fastboot flash` cannot write | every write goes through download mode (`spd_dump`) or from inside recovery |
| An unlocked bootloader does not remove the rules above | only the state changes (`verifiedbootstate=orange`), not the image verification rules |

Therefore: `vbmeta*` and `boot` stay stock, and every `vendor_boot` that is written must
carry Meizu's AVB blob. The LineageOS 24.0 build and the recovery builds in these repositories
do this (the blob is taken from the stock dump); a self-made image that does not attach that
blob stops at the logo.

## 2. LineageOS 24.0 (Android 17, unofficial)

Status as of 2026-11-03: **boots and is in daily use on one M416H**, with a partial test
matrix. The `lineage-24.0-20261103-UNOFFICIAL-mblu21` release is still SELinux permissive,
with dm-verity off for the logical partitions. A local build with device tree commits `79d312d`
and `0d92969` was tested on the phone on 2026-11-03: SELinux enforcing, calls and SMS work,
and Volume Down no longer resets the phone; that build has not been released on its own yet.
Not tested yet: Bluetooth pairing, VoLTE, outdoor GPS, deep sleep, OTA.

- The project's releases and official install steps:
  <https://github.com/tickernelz/lineage_build_mblu21/releases>
- Sources: [android_device_meizu_mblu21](https://github.com/tickernelz/android_device_meizu_mblu21),
  [android_vendor_meizu_mblu21](https://github.com/tickernelz/android_vendor_meizu_mblu21),
  [android_kernel_meizu_mblu21](https://github.com/tickernelz/android_kernel_meizu_mblu21),
  [lineage_build_mblu21](https://github.com/tickernelz/lineage_build_mblu21)
- Raw flash guide (written before AVB was understood, so it still contains the `fastboot` path
  that does not work on this phone): [FLASH.md](https://github.com/tickernelz/lineage_build_mblu21/blob/main/FLASH.md)

Summary of the install path that project uses; the exact details and order are in the
release notes, follow those:

1. Unlock the bootloader ([UNLOCK-RELOCK.md](UNLOCK-RELOCK.md)) and take the dump of 42 partition images.
2. Write `vendor_boot.img` to `vendor_boot_a` from download mode (`spd_dump`). This image
   carries Meizu's AVB blob, so the bootloader accepts it.
3. Boot into recovery (Lineage Recovery lives in the `vendor_boot` ramdisk).
4. Join the `super.img` parts, then write `super.img` (sparse) + `init_boot.img` from recovery
   with the `sparse_flash.py` that ships with the release.
5. Format data before the first boot when coming from stock.

## 3. Recovery: OrangeFox and SHRP

Status as of 2026-11-03: OrangeFox built locally from the `fix/fox-legacy-drm` branch
(`b3dbb0c`) and resealed with Meizu's stock AVB blob has been tested on one M416H: the UI and
touchscreen work, the LineageOS PIN decrypts `/data`, and Android still boots normally. The
OrangeFox release already published was built before that fix and has not been tested.
SHRP-Reborn has not been tested. `vendor_boot` on this phone is AVB-verified, so a recovery
image only loads when it is resealed with Meizu's stock AVB blob; older images without that
blob stop at the Meizu logo (recorded 2026-11-02). Do not rely on an untested recovery to
rescue the phone.

- Releases: <https://github.com/tickernelz/android_device_meizu_mblu21-recovery/releases>
- Repository: <https://github.com/tickernelz/android_device_meizu_mblu21-recovery>

Before installing: read the release notes. If they still say "UNTESTED", the right choice for
recovery is the Lineage Recovery that comes with the LineageOS build, or stock recovery.

What to know when using these recoveries:

- Only one recovery fits: Lineage Recovery, stock recovery, OrangeFox, and SHRP replace one
  another in `vendor_boot`.
- A ROM update/OTA overwrites `vendor_boot`, so the recovery is lost too and has to be
  installed again.
- These recoveries use the same `vendor_boot` that normal boot uses, so do not write a recovery
  `vendor_boot` to a phone in daily use without a backup of `vendor_boot_a.bin`.

## 4. ZIPs that do not exist yet

This project provides no official LineageOS/OrangeFox/SHRP ZIP package for the mBlu 21. If an
official ZIP release appears later, its steps will be added to this page **after** they are
tested on the device. Until then, the only documented paths are:

- restore to stock: [FLASH-STOCK.md](FLASH-STOCK.md)
- go back from LineageOS: [ROLLBACK.md](ROLLBACK.md)
- install LineageOS 24.0: release links in §2

## 5. Back to stock

If an attempt fails, follow [ROLLBACK.md](ROLLBACK.md). The BootROM cannot be overwritten, so
download mode can always be used to write the stock firmware again.
