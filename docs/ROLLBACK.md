# Rollback: kembali ke stok dari LineageOS atau custom ROM

> **MENGHAPUS DATA.** Rollback mengembalikan perangkat lunak ke pabrik, bukan berkas pribadi.
> Selamatkan data dulu (`adb pull /sdcard`), lalu jalankan dengan keputusan pemilik HP.

Tata letak folder sama dengan [FLASH-STOCK.md §2](FLASH-STOCK.md#2-ambil-rilis-susun-folder-verifikasi):
folder image berisi subfolder `tools`.

## 1. Apa yang berubah saat memasang LineageOS 24.0

Menurut catatan rilis LineageOS 24.0 untuk HP ini, yang ditulis saat install adalah:

| Partisi | Ditulis saat install | Kembali ke stok butuh |
|---|---|---|
| `vendor_boot_a` | `vendor_boot.img` LineageOS (berisi Lineage Recovery) | `vendor_boot_a.bin` |
| `super` | `super.img` sparse (partisi logis: system, system_ext, product, vendor, odm, vendor_dlkm, system_dlkm) | `super.bin` |
| `init_boot_a` | `init_boot.img` LineageOS (atau image hasil patch root) | `init_boot_a.bin` |
| `boot`, `dtbo`, `vbmeta*` | tidak diubah saat install biasa | tidak perlu, kecuali pernah ditimpa |

## 2. Langkah

Semua perintah dijalankan dari folder tool, dengan HP di mode download
([FLASH-STOCK.md §3](FLASH-STOCK.md#3-masuk-mode-download-brom)) sebelum **setiap** baris.

```bash
cd ~/mblu21-stock/tools
Q="exec_addr 0x65015f08 fdl fdl1-dl.bin 0x65000800 fdl fdl2-dl.bin 0x9efffe00 exec"

# 1. sistem: super stok (5,56 GB) — ulangi sampai "Write Part Done"
./spd_dump --wait 300 $Q timeout 600000 w super ../super.bin reset

# 2. boot chain
./spd_dump --wait 300 $Q w vendor_boot_a ../vendor_boot_a.bin reset
./spd_dump --wait 300 $Q w init_boot_a   ../init_boot_a.bin   reset
./spd_dump --wait 300 $Q w boot_a        ../boot_a.bin        reset
./spd_dump --wait 300 $Q w dtbo_a        ../dtbo_a.bin        reset

# 3. AVB + blok kendali boot
./spd_dump --wait 300 $Q w vbmeta_a ../vbmeta_a.bin reset
./spd_dump --wait 300 $Q w misc     ../misc.bin     reset
```

Windows: sama, tanpa `./`, dengan `set Q=...` + `%Q%` dan `..\` sebagai ganti `../`.

Catatan bukti: penulisan partisi stok seperti di atas sudah diulang beberapa kali pada HP ini
dan HP selalu boot Android 14 lagi sesudahnya.

## 3. Menghapus data dari sistem sebelumnya

Tiga partisi di atas cukup untuk firmware, tetapi `/data` masih terenkripsi dengan kunci
Android 17. Jalur yang dipakai: **factory reset dari stock recovery**, setelah
`vendor_boot_a` kembali stok.

1. Kalau Android stok belum bisa boot karena `/data` berasal dari Android 17, HP berhenti di
   layar logo atau meminta reset — itu tanda data harus diformat.
2. Masuk stock recovery lewat `adb reboot recovery` (adb di recovery terdeteksi sebagai
   perangkat `1782:d001`). Kombinasi tombol hardware untuk recovery pabrik belum diukur di
   panduan ini, jadi jangan mengandalkannya.
3. Pilih **Factory reset / Wipe data** → konfirmasi → **Reboot system**.
4. Tunggu boot pertama 2–4 menit.

Stock recovery di HP ini terbukti bisa boot (muncul sebagai perangkat `adb` `1782:d001`).

**Jangan pakai `spd_dump e userdata` untuk ini.** Perintah itu bukan menghapus `/data`: tool
menulis blok kendali `boot-recovery / --wipe_data` ke `misc` lalu **mengosongkan partisi
`persist`** (`common.c:1144-1154`), sehingga kalibrasi khas perangkat hilang dan HP diarahkan
ke recovery — bukan `/data` yang dibersihkan. Kalau `persist` sudah telanjur kosong, tulis
kembali `persist.bin` dari rilis ini (partisi ini ditulis mentah, byte-per-byte).

## 4. Verifikasi

```bash
adb reboot                      # keluar dulu dari recovery
adb shell getprop ro.build.version.release   # 14
adb shell getprop ro.build.display.id        # U01005 ...
adb shell getprop sys.boot_completed         # 1
```

Setelah stok tidak ada root, jadi partisi dibandingkan dengan membacanya kembali lewat mode
download (bukan `adb shell su`):

```bash
mkdir -p ~/mblu21-stock/readback && cd ~/mblu21-stock/readback
Q="exec_addr 0x65015f08 fdl fdl1-dl.bin 0x65000800 fdl fdl2-dl.bin 0x9efffe00 exec"
../tools/spd_dump --wait 300 $Q r boot_a r init_boot_a r vendor_boot_a r dtbo_a r vbmeta_a reset
sha256sum boot_a.bin init_boot_a.bin vendor_boot_a.bin dtbo_a.bin vbmeta_a.bin
```

Kelima nilai itu harus sama dengan baris berkas yang sama di `SHA256SUMS-full_dump.txt`.
`super` tidak dibandingkan per partisi logis; bandingkan dengan cara membaca ulang `super`
(`r super`, 5,56 GB) lalu mencocokkan hash-nya.

## 5. Kalau masih gagal boot

1. Tulis ulang **semua** 42 image partisi dari rilis, bukan hanya tiga di atas — perbedaan di
   modem/NV, `splloader`/`uboot`, atau rantai `vbmeta` juga menahan boot.
2. Periksa `vbmeta_a` dan rantai `vbmeta_*`: HP ini menolak `vbmeta` apa pun yang bukan dari
   Meizu dan berhenti di logo.
3. HP yang tidak mau boot bukan brick: BootROM selalu bisa dijangkau lewat mode download
   (Volume Up + colok USB), jadi rilis ini selalu bisa ditulis ulang.
