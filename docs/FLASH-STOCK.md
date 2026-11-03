# Pulih total ke firmware stok — Meizu mBlu 21 (M416H)

Semua image di rilis ini dibaca **dari HP itu sendiri** lewat Unisoc BootROM (mode download),
jadi menulisnya kembali mengembalikan HP ke kondisi saat dikirim pabrik.

Baca seluruh halaman sekali sebelum menyentuh HP. Satu sesi mode download hanya untuk satu
penulisan: setiap kali `spd_dump` selesai, HP reboot dan koneksinya putus, jadi HP harus
dimasukkan lagi ke mode download untuk partisi berikutnya.

> **MENGHAPUS DATA.** Menulis `super`, `misc`, atau menghapus data mengakhiri isi Android yang
> sekarang. Berkas pribadi tidak ada di rilis ini dan tidak bisa dipulihkan dari sini. Jangan
> jalankan satu pun perintah di bawah tanpa keputusan pemilik HP.

## 0. Isi rilis dan nama partisi

Nama berkas = nama partisi di `/dev/block/by-name`. `boot_a.bin` ditulis ke partisi
`boot_a`, `vbmeta_system_a.bin` ke `vbmeta_system_a`, `super.bin` ke `super`, dan
seterusnya, dengan pengecualian:

| Berkas | Nama yang dipakai di `spd_dump` | Catatan |
|---|---|---|
| `uboot_a.bin` | `uboot` | tool memetakan sendiri, bukan `uboot_a` |
| `l_runtimenv1.bin`, `l_runtimenv2.bin` | `l_runtimenv1`, `l_runtimenv2` | tanpa akhiran slot |
| `l_fixnv1_a.bin`, `l_fixnv2_a.bin` | `l_fixnv1_a`, `l_fixnv2_a` | wajib akhiran `_a` |
| `custom_exec_no_verify_65015f08.bin` | — | payload untuk tool, bukan partisi |
| `partition_1793459407.xml` | — | tabel partisi dari HP |

**Dua pasang partisi NV berisi byte yang sama.** Di dump ini `l_fixnv1_a.bin` identik
byte-per-byte dengan `l_fixnv2_a.bin`, dan `l_runtimenv1.bin` identik dengan
`l_runtimenv2.bin`. Itu bukan artefak pemotongan: kedua berkas berukuran penuh 2 MiB dan bukan
salinan tergeser satu sama lain (dicek dengan `cmp` dan perbandingan `tail`/`head`), dan tabel
partisi HP (`partitions/partition_1793459407.xml`) memang mencantumkan keduanya sebagai partisi
terpisah. Yang bisa dikatakan dari dump ini: kedua partisi di HP ini berisi data yang sama.

**Partisi NV tidak bisa dipulihkan dengan `w` dari rilis ini** — lihat §4.4.

Rincian isi rilis:

| Aset | Ukuran (byte) | Partisi tujuan |
|---|---|---|
| `super.bin` (dari `super.bin.part0..2`) | 5.557.452.800 | `super` |
| `boot_a.bin` | 67.108.864 | `boot_a` |
| `vendor_boot_a.bin` | 104.857.600 | `vendor_boot_a` |
| `init_boot_a.bin` | 8.388.608 | `init_boot_a` |
| `dtbo_a.bin` / `dtb_a.bin` | 8.388.608 | `dtbo_a` / `dtb_a` |
| `vbmeta_a.bin` dan lima `vbmeta_*_a.bin` | 1.048.576 | rantai AVB |
| `misc.bin`, `miscdata.bin` | 1.048.576 | `misc`, `miscdata` |
| `metadata.bin` | 67.108.864 | `metadata` |
| `persist.bin` | 2.097.152 | `persist` |
| `prodnv.bin` | 67.108.864 | `prodnv` |
| `splloader.bin`, `uboot_a.bin` | 262.144 / 3.145.728 | bootloader |
| `system_a.img`, `vendor_a.img`, `product_a.img`, `system_ext_a.img`, `odm_a.img`, `vendor_dlkm_a.img`, `system_dlkm_a.img` | hasil `lpunpack` | partisi **logis** di dalam `super` |
| sisanya (`logo.bin`, `fbootlogo.bin`, `calinv.bin`, `pm_sys_a.bin`, `sml_a.bin`, `trustos_a.bin`, `teecfg_a.bin`, `common_rs*_a.bin`, `hypervsior_a.bin`, `avbmeta_rs_a.bin`, `sysdumpdb.bin`, `pgpt.bin`, `uboot_log.bin`, `l_*.bin`) | beragam | nama sama tanpa `.bin` |

Slot B tidak pernah di-dump: HP ini boot dari slot `a` dan slot `b` kosong pada saat dump
(`*_b.img` berukuran 0). Memulihkan slot `a` sudah cukup.

## 1. Prasyarat host

### Linux

```bash
sudo dnf install -y android-tools libusb1 unzip      # Debian/Ubuntu: android-sdk-platform-tools-common libusb-1.0-0 unzip
```

Aturan udev memberi akses USB `1782:4d00` (mode download Unisoc) ke user yang sedang login,
jadi `spd_dump` tidak perlu dijalankan sebagai root:

```bash
echo 'SUBSYSTEM=="usb", ATTR{idVendor}=="1782", ATTR{idProduct}=="4d00", TAG+="uaccess"' \
  | sudo tee /etc/udev/rules.d/51-unisoc-download.rules
sudo udevadm control --reload
```

Aturan yang sama ada di dalam `tools-linux.zip` sebagai `51-unisoc-download.rules`.

### Windows

1. Pasang driver Unisoc dari `tools-windows.zip`: jalankan
   `SPD_Driver\Win10\DPInst64.exe` sebagai administrator (Windows 7/8: folder `Win7`/`Win8`).
2. Pasang platform-tools Google kalau ingin memakai `adb` untuk verifikasi.

## 2. Ambil rilis, susun folder, verifikasi

Pakai satu tata letak untuk semua perintah di halaman ini dan di `ROLLBACK.md`: folder image
berisi subfolder `tools`.

```
~/mblu21-stock/
├── super.bin  boot_a.bin  vbmeta_a.bin  ...      (aset rilis)
└── tools/                                        (isi tools-linux.zip / tools-windows.zip)
```

Linux:

```bash
mkdir -p ~/mblu21-stock && cd ~/mblu21-stock
gh release download mblu21-stock-U01005-20261031 -R tickernelz/android_stock_meizu_mblu21
cat super.bin.part0 super.bin.part1 super.bin.part2 > super.bin
unzip -q tools-linux.zip -d tools && chmod +x tools/spd_dump tools/gen_spl-unlock tools/chsize
sha256sum -c SHA256SUMS-full_dump.txt        # 44 baris, semuanya harus "OK"
sha256sum -c SHA256SUMS-logical.txt          # 7 baris, semuanya harus "OK"
```

Windows (PowerShell):

```powershell
mkdir C:\mblu21\stock -Force; cd C:\mblu21\stock
gh release download mblu21-stock-U01005-20261031 -R tickernelz/android_stock_meizu_mblu21
cmd /c copy /b super.bin.part0+super.bin.part1+super.bin.part2 super.bin
Expand-Archive tools-windows.zip -DestinationPath tools -Force
Get-FileHash .\boot_a.bin -Algorithm SHA256      # bandingkan dengan SHA256SUMS-full_dump.txt
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

(`^` adalah penyambung baris di `cmd.exe`.)

Berhasil bila output berisi `Write Part Done`. Penulisan berjalan per blok: kalau koneksi
putus di tengah (`libusb_open_device failed`, `LIBUSB_ERROR_IO`, `find port failed`),
partisi bisa tertinggal **sebagian** tertulis. Masukkan lagi HP ke mode download (§3) dan
**ulangi perintah yang sama sampai `Write Part Done` muncul**.

### 4.2 Urutan yang dipakai saat restore

1. `super` — 5,56 GB, sediakan waktu; pakai `timeout 600000`.
2. `boot_a`, `init_boot_a`, `vendor_boot_a`, `dtbo_a`, `dtb_a`.
3. Rantai AVB: `vbmeta_a`, `vbmeta_system_a`, `vbmeta_system_ext_a`, `vbmeta_vendor_a`,
   `vbmeta_product_a`, `vbmeta_odm_a`.
4. `misc` — menghapus blok kendali boot; isi `boot-recovery` yang tertinggal membuat HP
   masuk recovery, bukan Android.
5. `splloader`, `uboot` — hanya kalau bootloader ikut diubah (mis. sesudah unlock).
6. Partisi lain yang berubah (modem/DSP/NV) sesuai kebutuhan — baca §4.4 dulu.

### 4.3 Perintah per partisi

Linux (HP dimasukkan ulang ke mode download di antara baris):

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

Windows — satu berkas `flash-stock.cmd` di `C:\mblu21\stock\tools`, jalankan per baris
(`set Q=...` dulu, lalu satu perintah per HP masuk mode download):

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

Bootloader (hanya kalau perlu; nama partisi `uboot`, bukan `uboot_a`):

```bash
./spd_dump --wait 300 $Q w splloader ../splloader.bin reset
./spd_dump --wait 300 $Q w uboot     ../uboot_a.bin  reset
```

```bat
spd_dump --wait 300 %Q% w splloader ..\splloader.bin reset
spd_dump --wait 300 %Q% w uboot ..\uboot_a.bin reset
```

### 4.4 Partisi NV: apa yang sebenarnya dilakukan `w`

Ini bagian yang paling mudah disalahpahami, dan perilakunya berasal dari kode tool
(`spreadtrum_flash`, sha1 `f2fc7792`):

| Perintah | Yang benar-benar terjadi |
|---|---|
| `w l_runtimenv1 <file>` / `w l_runtimenv2 <file>` | partisi **dikosongkan**. Tool tidak menulis data apa pun untuk nama yang mengandung `runtimenv` (`common.c:1174`), jadi hasilnya `Erase Part Done`, bukan `Write Part Done`. |
| `w l_fixnv1_a <file>` | berkas dibaca sebagai **paket NV**: header `0x200` dibuang, record di-parse, CRC16 dihitung ulang, lalu ditulis ulang (`load_nv_partition`). Isi partisi **tidak** byte-identik dengan berkas, jadi hash-nya tidak akan cocok dengan `SHA256SUMS-full_dump.txt`. |
| `w l_fixnv2_a <file>` | ditulis mentah, byte-per-byte. |
| `w calinv <file>` | **tidak melakukan apa pun** — nama `calinv` dilewati tool (`common.c:1176`). |
| `w prodnv <file>`, `w persist <file>` | ditulis mentah. |

Kesimpulan praktis: dari rilis ini, partisi NV yang bisa dipulihkan dengan `w` adalah
`l_fixnv2_a`, `prodnv`, dan `persist`. `l_runtimenv*` **tidak bisa** dipulihkan dengan tool ini
(perintahnya menghapus), `l_fixnv1_a` hanya bisa dikembalikan sebagai paket NV hasil
rekonstruksi (bukan byte asli), dan `calinv` tidak bisa ditulis sama sekali. Jangan jalankan
perintah NV kalau tidak sedang memperbaiki HP ini sendiri; lihat peringatan di README.

## 5. Verifikasi setelah flash

Lepas kabel, tekan **Power** sekali, tunggu. Boot pertama 2–4 menit.

```bash
adb shell getprop ro.build.version.release      # 14
adb shell getprop ro.build.display.id           # U01005 ...
adb shell getprop ro.boot.verifiedbootstate     # green kalau bootloader terkunci
adb shell getprop sys.boot_completed            # 1
```

Setelah restore stok tidak ada root, jadi bandingkan partisi dengan membacanya kembali lewat
mode download, bukan lewat `adb shell su`. Jalankan dari folder terpisah supaya tidak menimpa
image yang sudah diunduh:

```bash
mkdir -p ~/mblu21-stock/readback && cd ~/mblu21-stock/readback
Q="exec_addr 0x65015f08 fdl fdl1-dl.bin 0x65000800 fdl fdl2-dl.bin 0x9efffe00 exec"
../tools/spd_dump --wait 300 $Q r boot_a r init_boot_a r vbmeta_a r dtbo_a reset
sha256sum boot_a.bin init_boot_a.bin vbmeta_a.bin dtbo_a.bin
```

Nilainya harus sama dengan baris berkas yang sama di `SHA256SUMS-full_dump.txt`
(`boot_a` `3a78e139…`, `vbmeta_a` `83789a72…`). Windows:

```bat
cd /d C:\mblu21\stock\tools
set Q=exec_addr 0x65015f08 fdl fdl1-dl.bin 0x65000800 fdl fdl2-dl.bin 0x9efffe00 exec
mkdir ..\readback
cd ..\readback
spd_dump --wait 300 %Q% r boot_a r init_boot_a r vbmeta_a r dtbo_a reset
powershell -c "Get-FileHash .\boot_a.bin, .\vbmeta_a.bin -Algorithm SHA256"
```

## 6. Troubleshooting

| Gejala | Sebab | Tindakan |
|---|---|---|
| `libusb_open_device failed`, `LIBUSB_ERROR_TIMEOUT`, `find port failed` | HP tidak di mode download, atau sesi sebelumnya masih memegang node USB | cabut, matikan 20 detik, tahan **Volume Up**, colok lagi |
| HP boot ke Android, bukan mode download | Volume Up dilepas terlalu cepat | tahan sampai `lsusb` menampilkan `1782:4d00` |
| Windows tidak melihat `SPD U2S Diag (COM<n>)` | driver Unisoc belum ada, atau path tool mengandung spasi | pasang `SPD_Driver\Win10\DPInst64.exe`, pindahkan tool ke `C:\mblu21\stock\tools` |
| `Write Part Done` tidak pernah muncul, HP reboot | tombol yang ditahan salah | pakai **Volume Up** saja |
| `LIBUSB_ERROR_IO` di tengah `super` | HP keluar dari mode download | masukkan lagi, ulangi perintah `super` — partisi mungkin sudah sebagian tertulis, jadi penulisan ulang wajib |
| HP berhenti di logo Meizu | `vbmeta`, `vendor_boot`, atau `super` bukan dari build ini | tulis ulang `vbmeta_a` + rantai AVB + `super` dari rilis ini |
| "Your device is corrupt" | `super` setengah tertulis | ulangi penulisan `super` dengan `timeout 600000` sampai `Write Part Done` |
| `custom_exec_no_verify… not found` | tool dijalankan dari folder lain | `cd` ke folder `tools` dulu |
| `w l_runtimenv1` menjawab `Erase Part Done` | memang begitu perilaku tool | jangan tulis partisi itu; lihat §4.4 |

## 7. Kenapa `fastboot` tidak dipakai di sini

Di unit M416H, `fastboot flash` menerima `Sending … OKAY` lalu gagal:

- `fastboot flash vbmeta`, `dtbo`, `vendor_boot` → `Status read failed (No such device)`;
- `fastboot boot` tidak pernah menjalankan image.

Yang berfungsi hanya `fastboot getvar`, `fastboot reboot`, dan `fastboot devices`. Karena itu
seluruh panduan ini memakai mode download Unisoc. BootROM tidak bisa ditimpa, jadi HP yang
tidak mau boot tetap bisa dipulihkan lewat jalur ini.
