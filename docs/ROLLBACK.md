# Rollback: kembali ke stok dari LineageOS atau custom ROM

> **MENGHAPUS DATA.** Rollback mengembalikan perangkat lunak ke pabrik, bukan berkas pribadi.
> Selamatkan data dulu (`adb pull /sdcard`), lalu jalankan dengan keputusan pemilik HP.

## 1. Apa yang berubah saat memasang LineageOS 24.0

Menurut catatan rilis LineageOS 24.0 untuk HP ini, yang ditulis saat install adalah:

| Partisi | Ditulis saat install | Kembali ke stok butuh |
|---|---|---|
| `vendor_boot_a` | `vendor_boot.img` LineageOS (berisi Lineage Recovery) | `vendor_boot_a.bin` |
| `super` | `super.img` sparse (partisi logis: system, system_ext, product, vendor, odm, vendor_dlkm, system_dlkm) | `super.bin` |
| `init_boot_a` | `init_boot.img` LineageOS (atau image hasil patch root) | `init_boot_a.bin` |
| `boot`, `dtbo`, `vbmeta*` | **tidak** (sengaja dibiarkan stok, karena bootloader hanya menerima `vbmeta` Meizu) | tidak perlu |

Karena itu rollback = menulis tiga partisi di atas dari rilis ini, lalu menghapus `userdata`
dan `metadata` (format data berbeda: Android 17 → Android 14).

## 2. Langkah

Semua perintah dijalankan dari folder tool, dengan HP di mode download
([FLASH-STOCK.md §3](FLASH-STOCK.md#3-masuk-mode-download-brom)) sebelum **setiap** baris.

```bash
cd ~/mblu21-stock
Q="exec_addr 0x65015f08 fdl fdl1-dl.bin 0x65000800 fdl fdl2-dl.bin 0x9efffe00 exec"

# 1. sistem: super stok (5,56 GB)
cd ../tools-linux
./spd_dump --wait 300 $Q timeout 600000 w super ../mblu21-stock/super.bin reset

# 2. boot chain
./spd_dump --wait 300 $Q w vendor_boot_a ../mblu21-stock/vendor_boot_a.bin reset
./spd_dump --wait 300 $Q w init_boot_a   ../mblu21-stock/init_boot_a.bin   reset
./spd_dump --wait 300 $Q w boot_a        ../mblu21-stock/boot_a.bin        reset
./spd_dump --wait 300 $Q w dtbo_a        ../mblu21-stock/dtbo_a.bin        reset

# 3. AVB + blok kendali boot
./spd_dump --wait 300 $Q w vbmeta_a ../mblu21-stock/vbmeta_a.bin reset
./spd_dump --wait 300 $Q w misc     ../mblu21-stock/misc.bin     reset
```

Windows: sama, tanpa `./`, dan pakai `set Q=...` + `%Q%`.

Catatan bukti: penulisan partisi stok seperti di atas sudah diulang beberapa kali pada HP ini
dan HP selalu boot Android 14 lagi sesudahnya.

## 3. Menghapus data dari sistem sebelumnya

Tiga partisi di atas cukup untuk firmware, tetapi `/data` masih terenkripsi dengan kunci
Android 17. Pilih satu:

**A. Hapus partisi data lewat mode download (paling langsung).**

```bash
./spd_dump --wait 300 $Q e userdata reset
./spd_dump --wait 300 $Q e metadata reset
```

Setelah partisi kosong, Android stok memformatnya sendiri pada boot pertama. Langkah `e`
(erase) memakai perintah `erase_part` dari tool; ini **belum diukur** pada HP ini untuk
rollback lintas versi Android, jadi anggap sebagai langkah yang harus dibuktikan di HP sendiri.

**B. Factory reset dari stock recovery.**

Stock recovery di HP ini terbukti bisa boot (muncul sebagai perangkat `adb` `1782:d001`).
Masuk recovery lalu *Factory reset / Wipe data*: ini yang menghapus data pengguna. Kalau
Android sudah bisa boot, `adb reboot recovery` adalah cara termudah masuk ke sana.

## 4. Verifikasi

```bash
adb shell getprop ro.build.version.release   # 14
adb shell getprop ro.build.display.id        # U01005 ...
adb shell getprop sys.boot_completed         # 1
```

Dengan root, bandingkan partisi:

```bash
for p in super boot_a init_boot_a vendor_boot_a dtbo_a vbmeta_a; do
  printf "%-14s " "$p"
  adb shell su -c "sha256sum /dev/block/by-name/$p"
done
```

`boot_a`, `init_boot_a`, `vendor_boot_a`, `dtbo_a`, `vbmeta_a` harus sama dengan
`SHA256SUMS-full_dump.txt`. `super` dibandingkan sebagai image utuh lewat dump ulang
(`spd_dump r super`), karena daftar hash berisi `super.bin` utuh, bukan isi per partisi logis.

## 5. Kalau masih gagal boot

1. Tulis ulang **semua** 44 partisi dari rilis (bukan hanya tiga di atas) — perbedaan kecil
   di modem/NV atau `splloader`/`uboot` juga bisa menahan boot.
2. Periksa `vbmeta_a` dan rantai `vbmeta_*`: HP ini menolak `vbmeta` apa pun yang bukan dari
   Meizu dan berhenti di logo.
3. HP yang tidak mau boot bukan brick: BootROM selalu bisa dijangkau lewat mode download
   (kombinasi Volume Up + colok USB), jadi rilis ini selalu bisa ditulis ulang.
