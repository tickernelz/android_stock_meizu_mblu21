# Unlock dan relock bootloader — Meizu mBlu 21 (M416H)

> **MENGHAPUS DATA / MERUSAK BOOT.** Semua perintah di halaman ini menyentuh bootloader.
> Kesalahan di sini bisa membuat HP tidak mau boot. Jangan jalankan tanpa keputusan pemilik HP
> dan tanpa dump 42 image partisi yang sudah diverifikasi ([README](../README.md)).

## 1. Status faktual

- **Meizu tidak menyediakan jalur unlock resmi untuk mBlu 21.** Tidak ada halaman permintaan
  kode, dan ROM bawaan HP ini tidak merutekan unlock lewat protokol fastboot standar
  (`fastboot flashing unlock` bukan jalur yang tersedia di unit ini).
- **Satu-satunya jalur yang ada adalah tool komunitas** karya TomKing062 untuk
  CVE-2022-38694, memakai celah di Unisoc BootROM. Panduan halaman ini hanya memakai tool itu;
  detail eksploitasinya tidak disalin di sini — ikuti sumber aslinya:
  - Repositori: <https://github.com/TomKing062/CVE-2022-38694_unlock_bootloader>
  - Rilis (biner + paket `ums9230_universal_unlock_UFS`): <https://github.com/TomKing062/CVE-2022-38694_unlock_bootloader/releases>
  - Wiki langkah resmi: <https://github.com/TomKing062/CVE-2022-38694_unlock_bootloader/wiki>
  - Tool tulis partisi yang dipakai: <https://github.com/TomKing062/spreadtrum_flash>
- HP ini **UFS**, bukan eMMC: pakai paket `ums9230_universal_unlock_UFS`, bukan varian
  eMMC.

## 2. Apa yang berubah setelah unlock (terukur di unit ini, 2026-11-02)

| Sebelum | Sesudah |
|---|---|
| `ro.boot.flash.locked=1`, `verifiedbootstate=green` | `ro.boot.flash.locked=0`, `verifiedbootstate=orange`, splash "LOCK FLAG IS: UNLOCK" |
| bootloader terkunci | flag unlock tersimpan di `miscdata` |

**Data tidak terhapus oleh jalur ini.** Terukur pada HP ini: setelah unlock, `/data` masih
berisi 2,9 GB, 255 paket terpasang, dan FBE masih aktif. Metode CVE-2022-38694 hanya mengubah
flag di `miscdata`; ia tidak menjalankan wipe seperti `fastboot flashing unlock` di perangkat
lain.

Yang hilang: **Play Integrity** lolos (bootloader terbuka), dan
splash menampilkan peringatan oranye. Sebagian aplikasi perbankan menolak jalan.

## 3. Unlock — urutan yang dipakai di unit ini

Semua perintah di halaman ini dijalankan dari folder tool, `~/mblu21-stock/tools` (tata
letaknya sama dengan [FLASH-STOCK.md §2](FLASH-STOCK.md#2-ambil-rilis-susun-folder-verifikasi)),
karena `spd_dump` membuka payload dan FDL relatif terhadap folder kerja.

Prasyarat: tools dari rilis CVE-2022-38694 di satu folder (`spd_dump`, `gen_spl-unlock`,
`chsize`, `fdl1-dl.bin`, `fdl2-dl.bin`, `fdl2-cboot.bin`, `misc-wipe.bin`,
`spl-unlock.bin`, `custom_exec_no_verify_65015f08.bin`), HP di mode download
([FLASH-STOCK.md §3](FLASH-STOCK.md#3-masuk-mode-download-brom)), dan dump partisi yang sudah
diverifikasi.

Rangkaian perintahnya (Linux; Windows sama tanpa `./`, dan paket tool menyertakan
`unlock_autopatch_9230.bat` yang menjalankan urutan ini dengan jeda):

```bash
# 1. baca SPL + U-Boot, lalu kosongkan splloader (backup otomatis ke splloader_bak)
./spd_dump --wait 300 exec_addr 0x65015f08 fdl fdl1-dl.bin 0x65000800 \
    fdl fdl2-dl.bin 0x9efffe00 exec r splloader r uboot e splloader e splloader_bak reset

# 2. tanda tangani ulang SPL dengan patch unlock, siapkan U-Boot dari dump
./gen_spl-unlock splloader.bin
mv splloader.bin u-boot-spl-16k-sign.bin
./chsize uboot.bin
mv uboot.bin uboot_bak.bin

# 3. jalankan payload unlock lewat U-Boot
./spd_dump --wait 300 exec_addr 0x65015f08 fdl fdl1-dl.bin 0x65000800 \
    fdl fdl2-dl.bin 0x9efffe00 exec w uboot fdl2-cboot.bin reset
sleep 10
./spd_dump exec_addr 0x65015f08 fdl spl-unlock.bin 0x65000800

# 4. periksa flag: 64 byte nol = masih terkunci; string perangkat + dua hash = terbuka
./spd_dump exec_addr 0x65015f08 fdl fdl1-dl.bin 0x65000800 fdl fdl2-dl.bin 0x9efffe00 exec \
    verbose 2 read_part miscdata 8192 64 m.bin reset
xxd m.bin | head

# 5. kembalikan bootloader asli dan bersihkan blok kendali boot
./spd_dump exec_addr 0x65015f08 fdl fdl1-dl.bin 0x65000800 fdl fdl2-dl.bin 0x9efffe00 exec \
    r boot w splloader u-boot-spl-16k-sign.bin w uboot uboot_bak.bin w misc misc-wipe.bin reset
```

Kalau langkah 4 masih menunjukkan 64 byte nol, ulangi langkah 3 (jalankan dua kali seperti
dokumentasi upstream), lalu periksa lagi.

Nyatakan berhasil kalau HP boot normal ke Android dan:

```bash
adb shell getprop ro.boot.flash.locked     # 0
adb shell getprop ro.boot.verifiedbootstate # orange
```

## 4. Relock — hanya setelah semuanya stok

Relock **bukan** sekadar mengunci ulang: bootloader memverifikasi `vbmeta` dengan kunci Meizu,
jadi mengunci HP yang masih berisi image tidak bertanda tangan Meizu membuat HP **tidak bisa
boot sama sekali** (terukur: `vbmeta` asing — termasuk `vbmeta` bertanda tangan kunci uji
AOSP — membuat HP berhenti di logo Meizu; hanya `vbmeta` asli Meizu yang lolos).

### 4.1 Prasyarat wajib

1. Semua partisi sudah stok dari rilis ini: `super`, `boot_a`, `init_boot_a`,
   `vendor_boot_a`, `dtbo_a`, `dtb_a`, seluruh `vbmeta*_a`, `splloader`, `uboot_a`,
   `misc`.
2. Hash setiap partisi cocok dengan `SHA256SUMS-full_dump.txt` — periksa dengan root sebelum
   mengunci:
   ```bash
   for p in boot_a init_boot_a vendor_boot_a dtbo_a vbmeta_a super misc; do
     printf "%-14s " "$p"
     adb shell su -c "sha256sum /dev/block/by-name/$p"
   done
   ```
3. Tidak ada modul root yang menambal `init_boot_a`. Kalau pernah di-root, tulis ulang
   `init_boot_a.bin` stok dulu.

### 4.2 Jalur A — perintah fastboot

```bash
fastboot flashing lock
# atau, pada bootloader yang memakai perintah OEM lama:
fastboot oem lock
```

Di unit ini `fastboot` **tidak bisa menulis partisi** dan ROM-nya tidak merutekan unlock lewat
fastboot, sehingga perintah di atas bisa dijawab `FAILED (remote: 'unknown command')` atau
tidak berefek. Jalankan hanya setelah §4.1 terpenuhi; kalau ditolak, pakai jalur B.

### 4.3 Jalur B — memulihkan flag lewat mode download

Dump di rilis ini diambil saat HP **masih terkunci**, jadi `miscdata.bin` di rilis adalah
keadaan terkunci: 64 byte di offset 8192 bernilai nol (bisa diperiksa sendiri:
`dd if=miscdata.bin bs=1 skip=8192 count=64 status=none | xxd`). Menulisnya kembali
mengembalikan flag ke kondisi terkunci.

```bash
Q="exec_addr 0x65015f08 fdl fdl1-dl.bin 0x65000800 fdl fdl2-dl.bin 0x9efffe00 exec"
./spd_dump --wait 300 $Q w splloader ../splloader.bin reset      # bootloader asli
./spd_dump --wait 300 $Q w uboot     ../uboot_a.bin  reset
./spd_dump --wait 300 $Q w miscdata  ../miscdata.bin reset
./spd_dump --wait 300 $Q w misc      ../misc.bin     reset
```

Setelah reboot, periksa:

```bash
adb shell getprop ro.boot.flash.locked       # 1
adb shell getprop ro.boot.verifiedbootstate  # green
```

Jalur B ini **belum dieksekusi di unit ini**; ia diturunkan dari fakta yang terukur (isi
`miscdata` saat terkunci, dan bahwa jalur unlock mengubah flag di sana). Perlakukan sebagai
prosedur yang harus dibuktikan di HP sendiri, bukan resep yang sudah terbukti.

### 4.4 Peringatan

- Relock dengan image tidak bertanda tangan Meizu → HP berhenti di logo, tidak ada Android.
  Pemulihan hanya lewat mode download (BootROM selalu bisa dijangkau), jadi simpan rilis ini
  dan kabel yang terbukti jalan.
- Relock pada HP yang pernah di-root tanpa mengembalikan `init_boot_a` stok dapat membuat
  boot gagal; kembalikan dulu.
- Relock tidak mengembalikan Play Integrity kalau ada jejak lain, dan
  tidak mengembalikan data yang sudah dihapus.
