# Pulih total ke firmware stok — Meizu mBlu 21 (M416H)

Semua image di rilis ini dibaca **dari HP itu sendiri** lewat Unisoc BootROM (mode download),
jadi menulisnya kembali mengembalikan HP ke kondisi saat dikirim pabrik.

Baca seluruh halaman sekali sebelum menyentuh HP. Satu sesi mode download hanya untuk satu
penulisan: setiap kali `spd_dump` selesai, HP reboot dan koneksinya putus, jadi HP harus
dimasukkan lagi ke mode download untuk partisi berikutnya.

> **MENGHAPUS DATA.** Menulis `super`, `userdata`-terkait, atau `misc` mengakhiri isi Android
> yang sekarang. Berkas pribadi tidak ada di rilis ini dan tidak bisa dipulihkan dari sini.
> Jangan jalankan satu pun perintah di bawah tanpa keputusan pemilik HP.

## 0. Isi rilis dan nama partisi

Nama berkas = nama partisi di `/dev/block/by-name`. `boot_a.bin` ditulis ke partisi
`boot_a`, `vbmeta_system_a.bin` ke `vbmeta_system_a`, `super.bin` ke `super`, dan
seterusnya, dengan tiga pengecualian:

- `uboot_a.bin` ditulis ke nama `uboot` (tool memetakan sendiri; bukan `uboot_a`).
- `l_runtimenv1.bin` dan `l_runtimenv2.bin` dipanggil tanpa akhiran slot
  (`l_runtimenv1`, `l_runtimenv2`), sedangkan `l_fixnv1_a.bin`/`l_fixnv2_a.bin` tetap
  memakai akhiran `_a`.
- `custom_exec_no_verify_65015f08.bin` adalah payload untuk tool (bukan partisi) dan
  `partition_1793459407.xml` adalah tabel partisi.

| Aset | Ukuran | Partisi tujuan |
|---|---|---|
| `super.bin` (dari `super.bin.part0..2`) | 5.557.452.800 | `super` |
| `boot_a.bin` | 67.108.864 | `boot_a` |
| `vendor_boot_a.bin` | 104.857.600 | `vendor_boot_a` |
| `init_boot_a.bin` | 8.388.608 | `init_boot_a` |
| `dtbo_a.bin` / `dtb_a.bin` | 8.388.608 | `dtbo_a` / `dtb_a` |
| `vbmeta_a.bin` dan lima `vbmeta_*_a.bin` | 1.048.576 | rantai AVB |
| `misc.bin`, `miscdata.bin`, `metadata.bin` | 1.048.576 / 1.048.576 / 67.108.864 | `misc`, `miscdata`, `metadata` |
| `splloader.bin`, `uboot_a.bin` | 262.144 / 3.145.728 | bootloader |
| `persist.bin`, `prodnv.bin`, `l_*_a.bin`, `l_runtimenv*.bin`, `logo.bin`, `fbootlogo.bin`, `calinv.bin`, `common_rs*_a.bin`, `hypervsior_a.bin`, `avbmeta_rs_a.bin`, `sysdumpdb.bin`, `pm_sys_a.bin`, `sml_a.bin`, `trustos_a.bin`, `teecfg_a.bin`, `pgpt.bin`, `uboot_log.bin` | beragam | nama yang sama tanpa `.bin` |
| `system_a.img`, `vendor_a.img`, `product_a.img`, `system_ext_a.img`, `odm_a.img`, `vendor_dlkm_a.img`, `system_dlkm_a.img` | hasil `lpunpack` | partisi **logis** di dalam `super` |

Slot B tidak pernah di-dump: HP ini boot dari slot `a` dan slot `b` kosong pada saat dump
(`*_b.img` berukuran 0). Memulihkan slot `a` sudah cukup.

## 1. Prasyarat host

### Linux

```bash
sudo dnf install -y android-tools libusb1 unzip        # Debian/Ubuntu: android-sdk-platform-tools-common libusb-1.0-0
unzip tools-linux.zip && cd tools-linux
chmod +x spd_dump gen_spl-unlock chsize

sudo cp 51-unisoc-download.rules /etc/udev/rules.d/
sudo udevadm control --reload
```

Aturan udev memberi akses USB `1782:4d00` (mode download Unisoc) ke user yang sedang login,
jadi `spd_dump` tidak perlu dijalankan sebagai root.

### Windows

1. Ekstrak `tools-windows.zip` ke path pendek yang bisa ditulis, mis. `C:\mblu21\tools`.
   Tool menulis berkas ke folder kerjanya.
2. Pasang driver Unisoc: jalankan `SPD_Driver\Win10\DPInst64.exe` sebagai administrator
   (Windows 7/8: folder `Win7`/`Win8`).
3. Pasang platform-tools Google kalau ingin memakai `adb`/`fastboot` untuk verifikasi.
4. Buka Command Prompt di folder tool: `cd /d C:\mblu21\tools`. Semua perintah di bawah
   sama, tanpa awalan `./`.

## 2. Ambil dan verifikasi rilis

```bash
mkdir -p ~/mblu21-stock && cd ~/mblu21-stock
gh release download mblu21-stock-U01005-20261031 -R tickernelz/android_stock_meizu_mblu21
cat super.bin.part0 super.bin.part1 super.bin.part2 > super.bin
sha256sum -c SHA256SUMS-full_dump.txt        # 44 baris, semuanya harus "OK"
```

Windows (PowerShell), satu berkas:

```powershell
(Get-FileHash .\boot_a.bin -Algorithm SHA256).Hash.ToLower()   # bandingkan dengan SHA256SUMS-full_dump.txt
```

Setelah ketiga bagian `super` digabung, hash-nya harus
`0b263da0a621f9544a462dd8486a5e0869a339bd6e509d15654cc8d64d9f1e97` (tercatat di
`SHA256SUMS-full_dump.txt`). Kalau berbeda, jangan lanjut: unduh ulang bagian yang rusak.

> **MENGHAPUS DATA.** Mulai dari langkah ini setiap perintah menyentuh HP.

## 3. Masuk mode download (BROM)

1. Cabut kabel USB, matikan HP (tahan **Power** ±20 detik sampai layar tetap gelap).
2. Tahan **Volume Up** dan jangan dilepas.
3. Colok kabel USB sambil tetap menahan **Volume Up**.
4. Tahan ±10 detik, lalu lepas. Layar tetap hitam — itu benar.

Cek dari host:

```bash
lsusb | grep 1782          # harus muncul: 1782:4d00
```

Windows: **Device Manager → Ports** harus menampilkan **`SPD U2S Diag (COM<n>)`**. Kalau
muncul sebagai *unknown device*, driver Unisoc (§1) belum terpasang.

Pada unit ini **Volume Up** adalah tombol mode download; **Volume Down hanya sampai layar
pengisian baterai** (diukur 2026-11-02). Catatan terpisah: di dalam recovery, tombol Volume
Down fisik memicu reset PMIC — itu perilaku recovery, bukan mode download.

## 4. Menulis firmware

Bentuk perintahnya (dari dokumentasi tool):

```
spd_dump [OPSI] [PERINTAH] [PERINTAH KELUAR]
```

- `--wait 300` — menunggu sampai 300 detik HP muncul di USB (**bukan** batas waktu penulisan).
- `exec_addr 0x65015f08` — alamat CVE-2022-38694 yang dipakai unit ini untuk melewati
  pemeriksaan tanda tangan BROM.
- `fdl fdl1-dl.bin 0x65000800` dan `fdl fdl2-dl.bin 0x9efffe00` — loader FDL1/FDL2 dengan
  alamat yang cocok untuk UMS9230 ini.
- `exec` — menjalankan FDL2.
- `timeout 600000` — batas waktu satu perintah baca/tulis dalam milidetik (opsional; pakai
  untuk `super`).
- `w` / `write_part` — dua ejaan untuk perintah yang sama: tulis berkas ke satu partisi.
- `reset` — perintah keluar: HP reboot dan koneksi download putus.

### 4.1 Satu partisi per sesi (cara yang dipakai saat root)

Linux:

```bash
./spd_dump --wait 300 exec_addr 0x65015f08 fdl fdl1-dl.bin 0x65000800 \
    fdl fdl2-dl.bin 0x9efffe00 exec w boot_a ../boot_a.bin reset
```

Windows:

```bat
spd_dump --wait 300 exec_addr 0x65015f08 fdl fdl1-dl.bin 0x65000800 ^
    fdl fdl2-dl.bin 0x9efffe00 exec w boot_a ..\boot_a.bin reset
```

(`^` adalah penyambung baris di `cmd.exe`.)

Berhasil bila output berisi `Write Part Done`, lalu `reset`. Kalau muncul
`libusb_open_device failed`, `LIBUSB_ERROR_IO`, atau `find port failed`, HP keluar dari mode
download sebelum selesai: partisi tetap berisi data lama, tidak setengah tertulis. Masukkan
lagi HP ke mode download (§3) dan ulangi perintah yang sama.

### 4.2 Urutan yang dipakai saat restore

1. `super` — 5,56 GB, sediakan waktu; pakai `timeout 600000`.
2. `boot_a`, `init_boot_a`, `vendor_boot_a`, `dtbo_a`, `dtb_a`.
3. Rantai AVB: `vbmeta_a`, `vbmeta_system_a`, `vbmeta_system_ext_a`, `vbmeta_vendor_a`,
   `vbmeta_product_a`, `vbmeta_odm_a`.
4. `misc` — menghapus blok kendali boot; isi `boot-recovery` yang tertinggal membuat HP
   masuk recovery, bukan Android.
5. `splloader`, `uboot_a` — hanya kalau bootloader ikut diubah (mis. sesudah unlock).
6. Partisi lain yang berubah (modem/DSP/NV) sesuai kebutuhan.

### 4.3 Perintah per partisi

Linux, satu sesi per partisi (HP dimasukkan ulang di antara baris):

```bash
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

Windows, sama tanpa `./` dan dengan `\`:

```bat
set Q=exec_addr 0x65015f08 fdl fdl1-dl.bin 0x65000800 fdl fdl2-dl.bin 0x9efffe00 exec
spd_dump --wait 300 %Q% timeout 600000 w super ..\super.bin reset
spd_dump --wait 300 %Q% w boot_a ..\boot_a.bin reset
spd_dump --wait 300 %Q% w init_boot_a ..\init_boot_a.bin reset
spd_dump --wait 300 %Q% w vendor_boot_a ..\vendor_boot_a.bin reset
spd_dump --wait 300 %Q% w dtbo_a ..\dtbo_a.bin reset
spd_dump --wait 300 %Q% w vbmeta_a ..\vbmeta_a.bin reset
spd_dump --wait 300 %Q% w misc ..\misc.bin reset
```

Partisi bootloader (hanya kalau perlu) memiliki alamat `uboot` tanpa slot:

```bash
./spd_dump --wait 300 $Q w splloader ../splloader.bin reset
./spd_dump --wait 300 $Q w uboot     ../uboot_a.bin  reset
```

> **MENGHAPUS DATA.** `miscdata` menyimpan status unlock; menulisnya kembali mengunci
> bootloader lagi dan (pada HP ini) tidak menghapus data, tetapi jangan menulisnya kalau
> tidak berniat mengunci ulang. Lihat [UNLOCK-RELOCK.md](UNLOCK-RELOCK.md).

Partisi modem/NV hanya untuk pemulihan HP ini sendiri (lihat peringatan di README):

```bash
./spd_dump --wait 300 $Q w l_modem_a     ../l_modem_a.bin     reset
./spd_dump --wait 300 $Q w l_ldsp_a      ../l_ldsp_a.bin      reset
./spd_dump --wait 300 $Q w l_gdsp_a      ../l_gdsp_a.bin      reset
./spd_dump --wait 300 $Q w l_agdsp_a     ../l_agdsp_a.bin     reset
./spd_dump --wait 300 $Q w l_deltanv_a   ../l_deltanv_a.bin   reset
./spd_dump --wait 300 $Q w l_fixnv1_a    ../l_fixnv1_a.bin    reset
./spd_dump --wait 300 $Q w l_fixnv2_a    ../l_fixnv2_a.bin    reset
./spd_dump --wait 300 $Q w l_runtimenv1  ../l_runtimenv1.bin  reset
./spd_dump --wait 300 $Q w l_runtimenv2  ../l_runtimenv2.bin  reset
./spd_dump --wait 300 $Q w prodnv        ../prodnv.bin        reset
./spd_dump --wait 300 $Q w persist       ../persist.bin       reset
```

### 4.4 Kalau hanya butuh satu partisi logis

`super.bin` menimpa seluruh sistem sekaligus, dan itu jalur yang dipakai di sini. Tujuh image
hasil `lpunpack` di rilis ini berguna untuk **memeriksa** isi partisi logis atau menyusun
ulang `super` sendiri; menulisnya langsung ke HP ini hanya bisa lewat Lineage Recovery
(`sparse_flash.py` di rilis LineageOS menulis ke dalam `super` dari dalam recovery), karena
`fastboot flash` tidak bisa menulis di unit ini dan mode download hanya mengenal partisi
fisik `super`. Untuk pemulihan, tulis `super.bin` utuh.

## 5. Verifikasi setelah flash

Lepas kabel, tekan **Power** sekali, tunggu. Boot pertama 2–4 menit.

```bash
adb shell getprop ro.build.version.release      # 14
adb shell getprop ro.build.display.id           # U01005 ...
adb shell getprop ro.boot.verifiedbootstate     # green (kalau bootloader terkunci)
adb shell getprop sys.boot_completed            # 1
```

Bandingkan isi partisi dengan hash rilis (butuh root, mis. modul SukiSU/KernelSU):

```bash
adb shell su -c 'sha256sum /dev/block/by-name/boot_a'
adb shell su -c 'sha256sum /dev/block/by-name/vbmeta_a'
```

Nilainya harus sama dengan baris `boot_a.bin` dan `vbmeta_a.bin` di
`SHA256SUMS-full_dump.txt` (`3a78e139…` dan `83789a72…`).

## 6. Troubleshooting

| Gejala | Sebab | Tindakan |
|---|---|---|
| `libusb_open_device failed`, `LIBUSB_ERROR_TIMEOUT`, `find port failed` | HP tidak di mode download, atau sesi sebelumnya masih memegang node USB | cabut, matikan 20 detik, tahan **Volume Up**, colok lagi |
| HP boot ke Android, bukan mode download | Volume Up dilepas terlalu cepat | tahan sampai `lsusb` menampilkan `1782:4d00` |
| Windows tidak melihat `SPD U2S Diag (COM<n>)` | driver Unisoc belum ada, atau path tool mengandung spasi | pasang `SPD_Driver\Win10\DPInst64.exe`, pindahkan tool ke `C:\mblu21\tools` |
| `Write Part Done` tidak pernah muncul, HP reboot | tombol yang ditahan salah | pakai **Volume Up** saja |
| `LIBUSB_ERROR_IO` di tengah `super` | HP keluar dari mode download | masukkan lagi, ulangi perintah `super` |
| HP berhenti di logo Meizu | `vbmeta`, `vendor_boot`, atau `super` bukan dari build ini | tulis ulang `vbmeta_a` + rantai AVB + `super` dari rilis ini |
| "Your device is corrupt" | `super` setengah tertulis | ulangi penulisan `super` dengan `timeout 600000` |
| `custom_exec_no_verify… not found` | tool dijalankan dari folder lain | `cd` ke folder tool dulu |

## 7. Kenapa `fastboot` tidak dipakai di sini

Di unit M416H, `fastboot flash` menerima `Sending … OKAY` lalu gagal:

- `fastboot flash vbmeta`, `dtbo`, `vendor_boot` → `Status read failed (No such device)`;
- `fastboot boot` tidak pernah menjalankan image.

Yang berfungsi hanya `fastboot getvar`, `fastboot reboot`, dan `fastboot devices`. Karena itu
seluruh panduan ini memakai mode download Unisoc. BootROM tidak bisa ditimpa, jadi HP yang
tidak mau boot tetap bisa dipulihkan lewat jalur ini.
