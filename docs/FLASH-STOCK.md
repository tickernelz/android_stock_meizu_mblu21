# Restore the stock firmware — Meizu mBlu 21 (M416H)

Every image in this release was dumped **from the phone itself** through Unisoc BROM
(download mode). Writing them back returns the phone to the state it shipped in.

Read the whole page once before touching the phone. The download mode session is short
and every write below has already been executed on this exact model, so the commands are
copies of commands that worked — not guesses.

## 0. What is in the release

| Asset | Bytes | Partition | Slot |
|---|---|---|---|
| `boot_a.bin` | 67108864 | `boot` | a |
| `init_boot_a.bin` | 8388608 | `init_boot` | a |
| `vendor_boot_a.bin` | 104857600 | `vendor_boot` | a |
| `dtbo_a.bin` | 8388608 | `dtbo` | a |
| `dtb_a.bin` | 8388608 | `dtb` | a |
| `vbmeta_a.bin` | 1048576 | `vbmeta` | a |
| `vbmeta_system_a.bin`, `vbmeta_vendor_a.bin`, `vbmeta_odm_a.bin`, `vbmeta_product_a.bin`, `vbmeta_system_ext_a.bin` | 1048576 each | chained AVB | a |
| `super.bin` | 5557452800 | `super` (system, system_ext, product, vendor, odm, vendor_dlkm, system_dlkm) | a |
| `splloader.bin` | 262144 | `splloader` | — |
| `uboot_a.bin` | 3145728 | `uboot` | a |
| `misc.bin`, `miscdata.bin` | 1048576 each | `misc`, `miscdata` | — |
| `persist.bin`, `prodnv.bin`, `l_*.bin`, `pm_sys_a.bin`, `trustos_a.bin`, `sml_a.bin`, `teecfg_a.bin`, `logo.bin`, `fbootlogo.bin`, `metadata.bin`, `pgpt.bin`, `calinv.bin`, `common_rs*_a.bin`, `hypervsior_a.bin`, `avbmeta_rs_a.bin`, `sysdumpdb.bin`, `l_deltanv_a.bin`, `l_fixnv1_a.bin`, `l_fixnv2_a.bin`, `l_runtimenv1.bin`, `l_runtimenv2.bin` | — | remaining stock partitions | — |
| `super.bin.part0/1/2` | 2000 MiB, 2000 MiB, 1300 MiB | `super.bin` split so each asset stays under GitHub's 2 GiB limit | — |
| `tools-windows.zip` | — | `spd_dump.exe`, `gen_spl-unlock.exe`, `chsize.exe`, FDL loaders, Unisoc driver | — |
| `tools-linux.zip` | — | the same tools built for Linux, plus the udev rule | — |
| `SHA256SUMS-full_dump.txt` | — | hashes of the **unsplit** dumps, computed on the phone dump | — |

Slot B was never dumped: this phone boots from slot `a` and slot `b` holds an identical
copy that the device writes itself. Restoring slot `a` is enough.

## 1. Prepare the host

### Linux

```bash
sudo dnf install -y android-tools libusb1      # or: apt install android-tools-adb libusb-1.0-0
sudo cp 51-unisoc-download.rules /etc/udev/rules.d/
sudo udevadm control --reload
unzip tools-linux.zip && cd tools-linux && chmod +x spd_dump gen_spl-unlock chsize
```

### Windows

1. Run `SPD_Driver\Win10\DPInst64.exe` (administrator). On Windows 11 use the `Win10`
   folder; the `Win7`/`Win8` folders are for older systems.
2. Unzip `tools-windows.zip` to `C:\mblu21\tools`.
3. Open **Device Manager**; the phone in download mode appears as
   **`SPD U2S Diag (COM<n>)`** under *Ports*. If it appears with a yellow mark, reinstall
   the driver for that device.

## 2. Enter download mode (BROM)

1. Unplug USB.
2. Power the phone off completely (hold **Power** ~20 s until the screen stays dark).
3. Hold **Volume Up** and keep holding it.
4. Plug the USB cable in while still holding **Volume Up**.
5. Hold ~10 s, then release. The screen stays black — that is correct.

Check the host sees it:

```bash
lsusb | grep 1782          # Linux:  1782:4d00
```

```bat
REM Windows: Device Manager shows SPD U2S Diag (COM<n>)
```

On this model **Volume Up** is the download-mode key. Volume Down also enters it, but the
PMIC "7 s reset" feature listens on Volume Down and reboots the phone mid-flash, so do not
use it.

The BROM session dies every time the phone resets and disappears about a minute after the
last command, so run one command per session, or keep the session short.

## 3. Write the firmware

The loader addresses below are the ones this phone needs. `0x65015f08` is the
CVE-2022-38694 `custom_exec` address that bypasses the BROM signature check; the FDL1 and
FDL2 addresses match the stock partition table.

### Linux

```bash
cd tools-linux
Q="exec_addr 0x65015f08 fdl fdl1-dl.bin 0x65000800 fdl fdl2-dl.bin 0x9efffe00 exec"

# whole firmware in one session, writing into ./restored
./spd_dump --wait 300 $Q r boot r init_boot r vendor_boot r dtbo r vbmeta \
  r splloader r uboot r super r misc w boot_a ../boot_a.bin w init_boot_a ../init_boot_a.bin \
  reset
```

The reliable recipe is one write per command, for example:

```bash
./spd_dump --wait 300 $Q w boot ../boot_a.bin reset
./spd_dump --wait 300 $Q w init_boot ../init_boot_a.bin reset
./spd_dump --wait 300 $Q w vendor_boot ../vendor_boot_a.bin reset
./spd_dump --wait 300 $Q w dtbo ../dtbo_a.bin reset
./spd_dump --wait 300 $Q w misc ../misc.bin reset
```

`super` is 5.5 GB and needs its own session with a long wait:

```bash
cat super.bin.part0 super.bin.part1 super.bin.part2 > super.bin
sha256sum -c SHA256SUMS-full_dump.txt --ignore-missing
./spd_dump --wait 900 $Q w super ../super.bin reset
```

Success prints `Write Part Done: <partition>` and `target: 0x…, written: 0x…` with the two
numbers equal.

### Windows

Same commands; drop the `./` prefix and use backslashes for paths:

```bat
cd C:\mblu21\tools
set Q=exec_addr 0x65015f08 fdl fdl1-dl.bin 0x65000800 fdl fdl2-dl.bin 0x9efffe00 exec
spd_dump.exe --wait 300 %Q% w boot C:\mblu21\fw\boot_a.bin reset
spd_dump.exe --wait 300 %Q% w init_boot C:\mblu21\fw\init_boot_a.bin reset
spd_dump.exe --wait 300 %Q% w vendor_boot C:\mblu21\fw\vendor_boot_a.bin reset
spd_dump.exe --wait 300 %Q% w dtbo C:\mblu21\fw\dtbo_a.bin reset
spd_dump.exe --wait 900 %Q% w super C:\mblu21\fw\super.bin reset
```

### Order that was tested

1. `boot_a`, `init_boot_a`, `vendor_boot_a`, `dtbo_a`, `vbmeta_a`
2. `misc` (clears the boot-control block; a stale `boot-recovery` there sends the phone to
   recovery instead of Android)
3. `super`
4. `splloader`, `uboot_a` only if you also unlocked/changed the bootloader

Then unplug, press **Power** once, and wait. First boot takes 2-4 minutes.

## 4. What "fastboot" will not do on this phone

`fastboot flash` fails here even with the bootloader unlocked: `fastboot flash vbmeta`,
`fastboot flash dtbo` and `fastboot flash vendor_boot` all abort with
`Write img fail!` or `Status read failed (No such device)`. Only `fastboot flash misc`
succeeded. Use download mode for real writes; `fastboot` is still useful for
`fastboot getvar`, `fastboot reboot` and reading the serial.

## 5. Verify after flashing

```bash
adb shell getprop ro.build.version.release     # 14
adb shell getprop ro.build.display.id          # U01005
adb shell getprop ro.boot.verifiedbootstate    # green
adb shell getprop sys.boot_completed           # 1
adb shell "su -c 'sha256sum /dev/block/by-name/boot_a'"   # compare with SHA256SUMS
```

With root (the tested SukiSU package) you can also read each partition back and compare
it against `SHA256SUMS-full_dump.txt`.

## 6. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `libusb_open_device failed` / `LIBUSB_ERROR_TIMEOUT` | stale USB node from a previous session | unplug, power off 20 s, hold Volume Up, plug in, retry |
| Phone boots Android instead of download mode | Volume Up released too early | hold it until the host lists `1782:4d00` |
| `Write Part Done` never appears, phone reboots | Volume Down was used | Volume Up only |
| Phone stops at the Meizu logo after restoring | `vbmeta_a` or `super` not written, or written from a different build | rewrite `vbmeta_a` and `super` from this release |
| "Your device is corrupt" | `super` half written (session died mid-write) | redo the `super` write with `--wait 900` |
| `spd_dump` says `custom_exec_no_verify… not found` | tool must be started from its own directory | `cd` into the tools folder first |

## 7. Safety

- These are stock images of an already-unlocked phone; they contain no secrets and no
  user data. `userdata` was never dumped and is not in this release.
- Writing `super` overwrites the operating system. Back up anything you care about first;
  this restores the phone's software, it does not restore your files.
- Do not write an image from another model. The Unisoc partition table differs between
  `ums9230` boards even when the marketing name is the same.
