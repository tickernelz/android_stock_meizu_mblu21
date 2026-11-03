# Bootloader unlock and relock — Meizu mBlu 21 (M416H)

> **ERASES DATA / CAN BREAK BOOT.** Every command on this page touches the bootloader. A
> mistake here can leave the phone unable to boot. Do not run anything without the phone
> owner's decision and without the verified dump of 42 partition images ([README](../README.md)).

## 1. Factual status

- **Meizu provides no official unlock path for the mBlu 21.** There is no code-request page,
  and this phone's stock ROM does not route unlocking through the standard fastboot protocol
  (`fastboot flashing unlock` is not an available path on this unit).
- **The only existing path is a community tool** by TomKing062 for CVE-2022-38694, which uses
  a flaw in the Unisoc BootROM. This page only uses that tool; the exploit details are not
  copied here, so follow the original source:
  - Repository: <https://github.com/TomKing062/CVE-2022-38694_unlock_bootloader>
  - Releases (binaries + the `ums9230_universal_unlock_UFS` package): <https://github.com/TomKing062/CVE-2022-38694_unlock_bootloader/releases>
  - Official step-by-step wiki: <https://github.com/TomKing062/CVE-2022-38694_unlock_bootloader/wiki>
  - Partition-writing tool used: <https://github.com/TomKing062/spreadtrum_flash>
- This phone is **UFS**, not eMMC: use the `ums9230_universal_unlock_UFS` package, not the
  eMMC variant.

## 2. What changes after unlocking (measured on this unit, 2026-11-02)

| Before | After |
|---|---|
| `ro.boot.flash.locked=1`, `verifiedbootstate=green` | `ro.boot.flash.locked=0`, `verifiedbootstate=orange`, splash "LOCK FLAG IS: UNLOCK" |
| bootloader locked | unlock flag stored in `miscdata` |

**This path does not wipe data.** Measured on this phone: after unlocking, `/data` still held
2.9 GB, 255 installed packages, and FBE was still active. The CVE-2022-38694 method only
changes the flag in `miscdata`; it does not run a wipe the way `fastboot flashing unlock` does
on other devices.

What is lost: passing **Play Integrity** (the bootloader is open), and
the splash screen shows an orange warning. Some banking apps refuse to run.

## 3. Unlock: the sequence used on this unit

Every command on this page runs from the tool folder, `~/mblu21-stock/tools` (the same layout
as [FLASH-STOCK.md §2](FLASH-STOCK.md#2-download-the-release-lay-out-the-folder-verify)),
because `spd_dump` opens the payload and FDLs relative to the working folder.

Prerequisites: the tools from the CVE-2022-38694 release in one folder (`spd_dump`, `gen_spl-unlock`,
`chsize`, `fdl1-dl.bin`, `fdl2-dl.bin`, `fdl2-cboot.bin`, `misc-wipe.bin`,
`spl-unlock.bin`, `custom_exec_no_verify_65015f08.bin`), the phone in download mode
([FLASH-STOCK.md §3](FLASH-STOCK.md#3-enter-download-mode-brom)), and a verified partition
dump.

The command sequence (Linux; Windows is the same without `./`, and the tool package includes
`unlock_autopatch_9230.bat`, which runs this sequence with pauses):

```bash
# 1. read SPL + U-Boot, then erase splloader (automatic backup to splloader_bak)
./spd_dump --wait 300 exec_addr 0x65015f08 fdl fdl1-dl.bin 0x65000800 \
    fdl fdl2-dl.bin 0x9efffe00 exec r splloader r uboot e splloader e splloader_bak reset

# 2. re-sign SPL with the unlock patch, prepare U-Boot from the dump
./gen_spl-unlock splloader.bin
mv splloader.bin u-boot-spl-16k-sign.bin
./chsize uboot.bin
mv uboot.bin uboot_bak.bin

# 3. run the unlock payload through U-Boot
./spd_dump --wait 300 exec_addr 0x65015f08 fdl fdl1-dl.bin 0x65000800 \
    fdl fdl2-dl.bin 0x9efffe00 exec w uboot fdl2-cboot.bin reset
sleep 10
./spd_dump exec_addr 0x65015f08 fdl spl-unlock.bin 0x65000800

# 4. check the flag: 64 zero bytes = still locked; device string + two hashes = unlocked
./spd_dump exec_addr 0x65015f08 fdl fdl1-dl.bin 0x65000800 fdl fdl2-dl.bin 0x9efffe00 exec \
    verbose 2 read_part miscdata 8192 64 m.bin reset
xxd m.bin | head

# 5. restore the original bootloader and clear the boot-control block
./spd_dump exec_addr 0x65015f08 fdl fdl1-dl.bin 0x65000800 fdl fdl2-dl.bin 0x9efffe00 exec \
    r boot w splloader u-boot-spl-16k-sign.bin w uboot uboot_bak.bin w misc misc-wipe.bin reset
```

If step 4 still shows 64 zero bytes, repeat step 3 (run it twice, as the upstream
documentation does), then check again.

Call it a success when the phone boots normally into Android and:

```bash
adb shell getprop ro.boot.flash.locked     # 0
adb shell getprop ro.boot.verifiedbootstate # orange
```

## 4. Relock: only once everything is stock

Relocking is **not** just locking again: the bootloader verifies `vbmeta` with Meizu's key,
so locking a phone that still carries images not signed by Meizu leaves it **unable to boot
at all** (measured: a foreign `vbmeta`, including a `vbmeta` signed with the AOSP test key, stops
the phone at the Meizu logo; only Meizu's original `vbmeta` passes).

### 4.1 Mandatory prerequisites

1. Every partition is stock from this release: `super`, `boot_a`, `init_boot_a`,
   `vendor_boot_a`, `dtbo_a`, `dtb_a`, all `vbmeta*_a`, `splloader`, `uboot_a`,
   `misc`.
2. The hash of every partition matches `SHA256SUMS-full_dump.txt`; check it with root before
   locking:
   ```bash
   for p in boot_a init_boot_a vendor_boot_a dtbo_a vbmeta_a super misc; do
     printf "%-14s " "$p"
     adb shell su -c "sha256sum /dev/block/by-name/$p"
   done
   ```
3. No root module patches `init_boot_a`. If the phone was ever rooted, write the stock
   `init_boot_a.bin` back first.

### 4.2 Path A: fastboot command

```bash
fastboot flashing lock
# or, on bootloaders that use the old OEM command:
fastboot oem lock
```

On this unit `fastboot` **cannot write partitions** and the ROM does not route unlocking
through fastboot, so the commands above may be answered with `FAILED (remote: 'unknown command')`
or have no effect. Run them only after §4.1 is satisfied; if they are refused, use path B.

### 4.3 Path B: restoring the flag through download mode

The dump in this release was taken while the phone was **still locked**, so `miscdata.bin` in
the release is the locked state: the 64 bytes at offset 8192 are zero (check it yourself:
`dd if=miscdata.bin bs=1 skip=8192 count=64 status=none | xxd`). Writing it back
returns the flag to the locked state.

```bash
Q="exec_addr 0x65015f08 fdl fdl1-dl.bin 0x65000800 fdl fdl2-dl.bin 0x9efffe00 exec"
./spd_dump --wait 300 $Q w splloader ../splloader.bin reset      # original bootloader
./spd_dump --wait 300 $Q w uboot     ../uboot_a.bin  reset
./spd_dump --wait 300 $Q w miscdata  ../miscdata.bin reset
./spd_dump --wait 300 $Q w misc      ../misc.bin     reset
```

After the reboot, check:

```bash
adb shell getprop ro.boot.flash.locked       # 1
adb shell getprop ro.boot.verifiedbootstate  # green
```

Path B **has not been executed on this unit yet**; it is derived from measured facts (the
contents of `miscdata` while locked, and the fact that the unlock path changes the flag
there). Treat it as a procedure to prove on your own phone, not a proven recipe.

### 4.4 Warnings

- Relocking with images not signed by Meizu → the phone stops at the logo, no Android.
  Recovery is only through download mode (the BootROM is always reachable), so keep this
  release and a cable that is known to work.
- Relocking a phone that was rooted without putting the stock `init_boot_a` back can make
  the boot fail; restore it first.
- Relocking does not bring back Play Integrity if other traces remain, and
  does not bring back data that was already wiped.
