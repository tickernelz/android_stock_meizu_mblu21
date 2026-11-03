# Full restore to stock firmware — Meizu mBlu 21 (M416H)

Every image in this release was read **from the phone itself** through the Unisoc BootROM
(download mode), so writing them back returns the phone to the state it left the factory in.

Read the whole page once before touching the phone. One download-mode session covers one
write: each time `spd_dump` finishes, the phone reboots and the connection drops, so the phone
has to be put back into download mode for the next partition.

> **ERASES DATA.** Writing `super`, `misc`, or wiping data ends the current Android
> installation. Personal files are not in this release and cannot be recovered from here. Do
> not run any command below without the phone owner's decision.

## 0. Release contents and partition names

File name = partition name in `/dev/block/by-name`. `boot_a.bin` is written to partition
`boot_a`, `vbmeta_system_a.bin` to `vbmeta_system_a`, `super.bin` to `super`, and so on,
with these exceptions:

| File | Name used in `spd_dump` | Note |
|---|---|---|
| `uboot_a.bin` | `uboot` | the tool maps it itself, not `uboot_a` |
| `l_runtimenv1.bin`, `l_runtimenv2.bin` | `l_runtimenv1`, `l_runtimenv2` | no slot suffix |
| `l_fixnv1_a.bin`, `l_fixnv2_a.bin` | `l_fixnv1_a`, `l_fixnv2_a` | the `_a` suffix is required |
| `custom_exec_no_verify_65015f08.bin` | — | payload for the tool, not a partition |
| `partition_1793459407.xml` | — | partition table from the phone |

**Two pairs of NV partitions hold the same bytes.** In this dump `l_fixnv1_a.bin` is
byte-for-byte identical to `l_fixnv2_a.bin`, and `l_runtimenv1.bin` is identical to
`l_runtimenv2.bin`. That is not a truncation artefact: both files are the full 2 MiB and are
not shifted copies of each other (checked with `cmp` and a `tail`/`head` comparison), and the
phone's partition table (`partitions/partition_1793459407.xml`) does list them as separate
partitions. What this dump shows: both partitions on this phone hold the same data.

**NV partitions cannot be restored with `w` from this release** — see §4.4.

Release contents in detail:

| Asset | Size (bytes) | Target partition |
|---|---|---|
| `super.bin` (from `super.bin.part0..2`) | 5,557,452,800 | `super` |
| `boot_a.bin` | 67,108,864 | `boot_a` |
| `vendor_boot_a.bin` | 104,857,600 | `vendor_boot_a` |
| `init_boot_a.bin` | 8,388,608 | `init_boot_a` |
| `dtbo_a.bin` / `dtb_a.bin` | 8,388,608 | `dtbo_a` / `dtb_a` |
| `vbmeta_a.bin` and five `vbmeta_*_a.bin` | 1,048,576 | AVB chain |
| `misc.bin`, `miscdata.bin` | 1,048,576 | `misc`, `miscdata` |
| `metadata.bin` | 67,108,864 | `metadata` |
| `persist.bin` | 2,097,152 | `persist` |
| `prodnv.bin` | 67,108,864 | `prodnv` |
| `splloader.bin`, `uboot_a.bin` | 262,144 / 3,145,728 | bootloader |
| `system_a.img`, `vendor_a.img`, `product_a.img`, `system_ext_a.img`, `odm_a.img`, `vendor_dlkm_a.img`, `system_dlkm_a.img` | `lpunpack` output | **logical** partitions inside `super` |
| the rest (`logo.bin`, `fbootlogo.bin`, `calinv.bin`, `pm_sys_a.bin`, `sml_a.bin`, `trustos_a.bin`, `teecfg_a.bin`, `common_rs*_a.bin`, `hypervsior_a.bin`, `avbmeta_rs_a.bin`, `sysdumpdb.bin`, `pgpt.bin`, `uboot_log.bin`, `l_*.bin`) | various | same name without `.bin` |

Slot B was never dumped: this phone boots from slot `a`, and slot `b` was empty at dump time
(`*_b.img` is 0 bytes). Restoring slot `a` is enough.

## 1. Host prerequisites

### Linux

```bash
sudo dnf install -y android-tools libusb1 unzip usbutils   # Debian/Ubuntu: android-tools-adb libusb-1.0-0 unzip usbutils
```

The udev rule gives the logged-in user access to USB `1782:4d00` (Unisoc download mode), so
`spd_dump` does not need to run as root:

```bash
echo 'SUBSYSTEM=="usb", ATTR{idVendor}=="1782", ATTR{idProduct}=="4d00", TAG+="uaccess"' \
  | sudo tee /etc/udev/rules.d/51-unisoc-download.rules
sudo udevadm control --reload
```

The same rule ships inside `tools-linux.zip` as `51-unisoc-download.rules`.

### Windows

1. Install the Unisoc driver from the folder extracted in §2: run
   `tools\SPD_Driver\Win10\DPInst64.exe` as administrator. The zip only carries the `Win10`
   driver folder; that is the one Windows 10 and 11 use.
2. Install Google platform-tools if you want to use `adb` for verification.
3. Every command on this page runs from `C:\mblu21\stock\tools`, without the `./`
   prefix.

## 2. Download the release, lay out the folder, verify

Use one layout for every command on this page and in `ROLLBACK.md`: the image folder
contains a `tools` subfolder.

```
~/mblu21-stock/
├── super.bin  boot_a.bin  vbmeta_a.bin  ...      (release assets)
└── tools/                                        (contents of tools-linux.zip / tools-windows.zip)
```

Linux:

```bash
mkdir -p ~/mblu21-stock && cd ~/mblu21-stock
gh release download mblu21-stock-U01005-20261031 -R tickernelz/android_stock_meizu_mblu21
cat super.bin.part0 super.bin.part1 super.bin.part2 > super.bin
unzip -oq tools-linux.zip && { [ -d tools ] || mv tools-linux tools; }
chmod +x tools/spd_dump tools/gen_spl-unlock tools/chsize
./tools/spd_dump -h | head -1                # must print "branch:HEAD, sha1:f2fc7792…"
sha256sum -c SHA256SUMS-full_dump.txt        # 44 lines (42 partition images + tool payload + partition table), all "OK"
sha256sum -c SHA256SUMS-logical.txt          # 7 lines, all "OK"
```

Windows (PowerShell):

```powershell
mkdir C:\mblu21\stock -Force; cd C:\mblu21\stock
gh release download mblu21-stock-U01005-20261031 -R tickernelz/android_stock_meizu_mblu21
cmd /c copy /b super.bin.part0+super.bin.part1+super.bin.part2 super.bin
Expand-Archive tools-windows.zip -DestinationPath . -Force
if (-not (Test-Path tools)) { Rename-Item tools-windows tools }
.\tools\spd_dump.exe -h | Select-Object -First 1   # must print "branch:HEAD, sha1:…"
Get-FileHash .\boot_a.bin -Algorithm SHA256      # compare with SHA256SUMS-full_dump.txt
```

Both zips carry a top-level folder (`tools-linux/` and `tools-windows/`), so extract into the
working folder and then rename it. `unzip -d tools` produces `tools/tools-linux/`, and every
later command fails because `spd_dump` is not where the guide says it is.

After the three `super` parts are joined, its hash must be
`0b263da0a621f9544a462dd8486a5e0869a339bd6e509d15654cc8d64d9f1e97` (listed in
`SHA256SUMS-full_dump.txt`). If it differs, stop: download the damaged part again.

> **ERASES DATA.** From this step on, every command touches the phone.

## 3. Enter download mode (BROM)

1. Unplug the USB cable and power the phone off (hold **Power** about 20 seconds until the
   screen stays dark).
2. Hold **Volume Up** and do not let go.
3. Plug in the USB cable while still holding **Volume Up**.
4. Hold about 10 seconds, then release. The screen stays black; that is correct.

Check from the host:

```bash
lsusb | grep 1782          # must show: 1782:4d00
```

Windows: **Device Manager → Ports** must show **`SPD U2S Diag (COM<n>)`**. If it shows up as
an *unknown device*, the Unisoc driver (§1) is not installed yet.

On this unit **Volume Up** is the download-mode button; **Volume Down only reaches the
battery-charging screen** (measured 2026-11-02). A separate note: inside recovery, the
physical Volume Down button triggers a PMIC reset; that is recovery behaviour, not download
mode.

## 4. Writing the firmware

Command form (from the tool's documentation):

```
spd_dump [OPTIONS] [COMMANDS] [EXIT COMMANDS]
```

- `--wait 300`: waits up to 300 seconds for the phone to appear on USB (**not** a write
  timeout).
- `exec_addr 0x65015f08`: the CVE-2022-38694 address this unit uses to bypass the BROM
  signature check.
- `fdl fdl1-dl.bin 0x65000800` and `fdl fdl2-dl.bin 0x9efffe00`: the FDL1/FDL2 loaders with
  the addresses that match this UMS9230.
- `exec`: runs FDL2.
- `timeout 600000`: time limit for one read/write command, in milliseconds (optional; use it
  for `super`).
- `w` / `write_part`: two spellings of the same command: write a file to one partition.
- `reset`: exit command; the phone reboots and the download connection drops.

### 4.1 One partition per session (the method used while rooting)

Linux:

```bash
cd ~/mblu21-stock/tools
./spd_dump --wait 300 exec_addr 0x65015f08 fdl fdl1-dl.bin 0x65000800 \
    fdl fdl2-dl.bin 0x9efffe00 exec w boot_a ../boot_a.bin reset
```

Windows:

```bat
cd /d C:\mblu21\stock\tools
spd_dump --wait 300 exec_addr 0x65015f08 fdl fdl1-dl.bin 0x65000800 ^
    fdl fdl2-dl.bin 0x9efffe00 exec w boot_a ..\boot_a.bin reset
```

(`^` is the line continuation in `cmd.exe`.)

It succeeded when the output contains `Write Part Done`. The write runs block by block: if the
connection drops mid-way (`libusb_open_device failed`, `LIBUSB_ERROR_IO`, `find port failed`),
the partition can be left **partly** written. Put the phone back into download mode (§3) and
**repeat the same command until `Write Part Done` appears**.

### 4.2 Order used for a restore

1. `super`: 5.56 GB, allow time; use `timeout 600000`.
2. `boot_a`, `init_boot_a`, `vendor_boot_a`, `dtbo_a`, `dtb_a`.
3. AVB chain: `vbmeta_a`, `vbmeta_system_a`, `vbmeta_system_ext_a`, `vbmeta_vendor_a`,
   `vbmeta_product_a`, `vbmeta_odm_a`.
4. `misc`: clears the boot-control block; a leftover `boot-recovery` entry sends the phone
   into recovery instead of Android.
5. `splloader`, `uboot`: only if the bootloader was changed too (e.g. after an unlock).
6. Other partitions that changed (modem/DSP/NV) as needed; read §4.4 first.

### 4.3 Per-partition commands

Linux (put the phone back into download mode between lines):

```bash
cd ~/mblu21-stock/tools
Q="exec_addr 0x65015f08 fdl fdl1-dl.bin 0x65000800 fdl fdl2-dl.bin 0x9efffe00 exec"

./spd_dump --wait 300 $Q timeout 600000 w super        ../super.bin        reset
./spd_dump --wait 300 $Q w boot_a        ../boot_a.bin        reset
./spd_dump --wait 300 $Q w init_boot_a   ../init_boot_a.bin   reset
./spd_dump --wait 300 $Q w vendor_boot_a ../vendor_boot_a.bin reset
./spd_dump --wait 300 $Q w dtbo_a        ../dtbo_a.bin        reset
./spd_dump --wait 300 $Q w dtb_a         ../dtb_a.bin         reset
./spd_dump --wait 300 $Q w vbmeta_a      ../vbmeta_a.bin      reset
./spd_dump --wait 300 $Q w vbmeta_system_a     ../vbmeta_system_a.bin     reset
./spd_dump --wait 300 $Q w vbmeta_system_ext_a ../vbmeta_system_ext_a.bin reset
./spd_dump --wait 300 $Q w vbmeta_vendor_a     ../vbmeta_vendor_a.bin     reset
./spd_dump --wait 300 $Q w vbmeta_product_a    ../vbmeta_product_a.bin    reset
./spd_dump --wait 300 $Q w vbmeta_odm_a        ../vbmeta_odm_a.bin        reset
./spd_dump --wait 300 $Q w misc          ../misc.bin          reset
```

Windows: one `flash-stock.cmd` file in `C:\mblu21\stock\tools`, run line by line
(`set Q=...` first, then one command each time the phone enters download mode):

```bat
cd /d C:\mblu21\stock\tools
set Q=exec_addr 0x65015f08 fdl fdl1-dl.bin 0x65000800 fdl fdl2-dl.bin 0x9efffe00 exec

spd_dump --wait 300 %Q% timeout 600000 w super ..\super.bin reset
spd_dump --wait 300 %Q% w boot_a ..\boot_a.bin reset
spd_dump --wait 300 %Q% w init_boot_a ..\init_boot_a.bin reset
spd_dump --wait 300 %Q% w vendor_boot_a ..\vendor_boot_a.bin reset
spd_dump --wait 300 %Q% w dtbo_a ..\dtbo_a.bin reset
spd_dump --wait 300 %Q% w dtb_a ..\dtb_a.bin reset
spd_dump --wait 300 %Q% w vbmeta_a ..\vbmeta_a.bin reset
spd_dump --wait 300 %Q% w vbmeta_system_a ..\vbmeta_system_a.bin reset
spd_dump --wait 300 %Q% w vbmeta_system_ext_a ..\vbmeta_system_ext_a.bin reset
spd_dump --wait 300 %Q% w vbmeta_vendor_a ..\vbmeta_vendor_a.bin reset
spd_dump --wait 300 %Q% w vbmeta_product_a ..\vbmeta_product_a.bin reset
spd_dump --wait 300 %Q% w vbmeta_odm_a ..\vbmeta_odm_a.bin reset
spd_dump --wait 300 %Q% w misc ..\misc.bin reset
```

Bootloader (only if needed; the partition name is `uboot`, not `uboot_a`):

```bash
./spd_dump --wait 300 $Q w splloader ../splloader.bin reset
./spd_dump --wait 300 $Q w uboot     ../uboot_a.bin  reset
```

```bat
spd_dump --wait 300 %Q% w splloader ..\splloader.bin reset
spd_dump --wait 300 %Q% w uboot ..\uboot_a.bin reset
```

### 4.4 NV partitions: what `w` actually does

This is the easiest part to misread, and the behaviour comes from the tool's code
(`spreadtrum_flash`, sha1 `f2fc7792`):

| Command | What actually happens |
|---|---|
| `w l_runtimenv1 <file>` / `w l_runtimenv2 <file>` | the partition is **erased**. The tool writes no data for any name containing `runtimenv` (`common.c:1174`), so the result is `Erase Part Done`, not `Write Part Done`. |
| `w l_fixnv1_a <file>` | the file is read as an **NV packet**: the `0x200` header is dropped, the records are parsed, CRC16 is recomputed, and the result is written back (`load_nv_partition`). The partition contents are **not** byte-identical to the file, so its hash will not match `SHA256SUMS-full_dump.txt`. |
| `w l_fixnv2_a <file>` | written raw, byte for byte. |
| `w calinv <file>` | **does nothing**: the tool skips the name `calinv` (`common.c:1175`). |
| `w prodnv <file>`, `w persist <file>` | written raw. |

Practical conclusion: from this release, the NV partitions that `w` can restore are
`l_fixnv2_a`, `prodnv`, and `persist`. `l_runtimenv*` **cannot** be restored with this tool
(the command erases it), `l_fixnv1_a` can only be put back as a reconstructed NV packet (not
the original bytes), and `calinv` cannot be written at all. Do not run NV commands unless you
are repairing this very phone; see the warnings in the README.

## 5. Verify after flashing

Unplug the cable, press **Power** once, and wait. The first boot takes 2–4 minutes.

```bash
adb shell getprop ro.build.version.release      # 14
adb shell getprop ro.build.display.id           # U01005 ...
adb shell getprop ro.boot.verifiedbootstate     # green when the bootloader is locked
adb shell getprop sys.boot_completed            # 1
```

After a stock restore there is no root, so compare partitions by reading them back through
download mode, not through `adb shell su`. Run it from a separate folder so the downloaded
images are not overwritten:

```bash
mkdir -p ~/mblu21-stock/readback
cd ~/mblu21-stock/tools
Q="exec_addr 0x65015f08 fdl fdl1-dl.bin 0x65000800 fdl fdl2-dl.bin 0x9efffe00 exec"
./spd_dump --wait 300 $Q path ../readback r boot_a r init_boot_a r vbmeta_a r dtbo_a reset
cd ../readback && sha256sum boot_a.bin init_boot_a.bin vbmeta_a.bin dtbo_a.bin
```

The command still runs **from the `tools` folder**: `exec_addr` opens
`custom_exec_no_verify_<addr>.bin` and `fdl` opens `fdl1-dl.bin`/`fdl2-dl.bin` relative to
the working folder (`spd_dump.c:758`). The files produced by `r` are redirected to `readback/`
with the tool's `path` command ("save directory for `r`, `read_part(s)`, `read_flash`, `read_mem`").

The values must match the line for the same file in `SHA256SUMS-full_dump.txt`
(`boot_a` `3a78e139…`, `vbmeta_a` `83789a72…`). Windows:

```bat
cd /d C:\mblu21\stock\tools
set Q=exec_addr 0x65015f08 fdl fdl1-dl.bin 0x65000800 fdl fdl2-dl.bin 0x9efffe00 exec
mkdir ..\readback
spd_dump --wait 300 %Q% path ..\readback r boot_a r init_boot_a r vbmeta_a r dtbo_a reset
cd ..\readback
powershell -c "Get-FileHash .\boot_a.bin, .\vbmeta_a.bin -Algorithm SHA256"
```

## 6. Troubleshooting

| Symptom | Cause | Action |
|---|---|---|
| `libusb_open_device failed`, `LIBUSB_ERROR_TIMEOUT`, `find port failed` | the phone is not in download mode, or the previous session still holds the USB node | unplug, power off for 20 seconds, hold **Volume Up**, plug in again |
| The phone boots to Android, not download mode | Volume Up was released too early | hold it until `lsusb` shows `1782:4d00` |
| Windows does not see `SPD U2S Diag (COM<n>)` | the Unisoc driver is missing, or the tool path contains spaces | install `SPD_Driver\Win10\DPInst64.exe`, move the tools to `C:\mblu21\stock\tools` |
| `Write Part Done` never appears, the phone reboots | wrong button held | use **Volume Up** only |
| `LIBUSB_ERROR_IO` in the middle of `super` | the phone left download mode | enter it again and repeat the `super` command; the partition may already be partly written, so the rewrite is mandatory |
| The phone stops at the Meizu logo | `vbmeta`, `vendor_boot`, or `super` is not from this build | rewrite `vbmeta_a` + the AVB chain + `super` from this release |
| "Your device is corrupt" | `super` is half written | repeat the `super` write with `timeout 600000` until `Write Part Done` |
| `custom_exec_no_verify… not found` | the tool was run from another folder | `cd` into the `tools` folder first |
| `w l_runtimenv1` answers `Erase Part Done` | that is how the tool behaves | do not write that partition; see §4.4 |

## 7. Why `fastboot` is not used here

On the M416H, `fastboot flash` accepts `Sending … OKAY` and then fails:

- `fastboot flash vbmeta`, `dtbo`, `vendor_boot` → `Status read failed (No such device)`;
- `fastboot boot` never runs the image.

Only `fastboot getvar`, `fastboot reboot`, and `fastboot devices` work. That is why this whole
guide uses Unisoc download mode. The BootROM cannot be overwritten, so a phone that will not
boot can still be restored through this path.
