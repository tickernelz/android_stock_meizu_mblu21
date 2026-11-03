# Memasang LineageOS, OrangeFox, dan SHRP

Halaman ini menjelaskan **status sebenarnya** per 2026-11-03 dan menunjuk ke rilis proyek
masing-masing. Kalau sebuah proyek belum merilis image, jangan pakai instruksi generik dari
internet: instruksi untuk perangkat Unisoc lain bisa membuat HP ini tidak boot.

> **MENGHAPUS DATA.** Memasang sistem operasi lain memformat `/data`. Pastikan dump stok
> sudah diunduh dan diverifikasi ([FLASH-STOCK.md](FLASH-STOCK.md)) sebelum mulai.

## 1. Aturan bootloader yang berlaku di HP ini (terukur)

| Fakta | Akibat |
|---|---|
| `vbmeta` hanya diterima kalau ditandatangani Meizu | `vbmeta` hasil build sendiri atau bertanda tangan kunci uji → HP berhenti di logo Meizu; build LineageOS dan recovery karena itu **mempertahankan** `vbmeta` stok |
| `vendor_boot` harus membawa blob AVB Meizu | image dengan blob dari dump stok termuat; image tanpa blob itu berhenti di logo Meizu (terukur 2026-11-02 pada SHRP/OrangeFox sebelum build di-reseal) |
| `boot` diverifikasi | `boot` dibiarkan stok |
| `init_boot` **tidak** diverifikasi | bisa ditambal untuk root |
| `fastboot flash` tidak bisa menulis | semua penulisan lewat mode download (`spd_dump`) atau dari dalam recovery |
| Bootloader tidak terkunci tidak menghapus syarat di atas | yang berubah hanya status (`verifiedbootstate=orange`), bukan aturan verifikasi image |

Karena itu: `vbmeta*` dan `boot` tetap stok, dan setiap `vendor_boot` yang ditulis harus
membawa blob AVB Meizu. Build LineageOS 24.0 dan build recovery di repo ini melakukannya
(blob diambil dari dump stok); image buatan sendiri yang tidak menempelkan blob itu akan
berhenti di logo.

## 2. LineageOS 24.0 (Android 17, unofficial)

Status per 2026-11-03: **boot dan dipakai harian di satu unit M416H** dengan matriks pengujian
sebagian — SELinux permissive, dm-verity mati untuk partisi logis, sebagian komponen
(Bluetooth pairing, VoLTE, GPS di luar ruangan, deep sleep, OTA) belum diuji.

- Rilis dan langkah install resmi proyek:
  <https://github.com/tickernelz/lineage_build_mblu21/releases>
- Sumber: [android_device_meizu_mblu21](https://github.com/tickernelz/android_device_meizu_mblu21),
  [android_vendor_meizu_mblu21](https://github.com/tickernelz/android_vendor_meizu_mblu21),
  [android_kernel_meizu_mblu21](https://github.com/tickernelz/android_kernel_meizu_mblu21),
  [lineage_build_mblu21](https://github.com/tickernelz/lineage_build_mblu21)
- Panduan flash mentah (ditulis sebelum AVB dipahami, jadi masih memuat jalur `fastboot` yang
  tidak berfungsi di HP ini): [FLASH.md](https://github.com/tickernelz/lineage_build_mblu21/blob/main/FLASH.md)

Ringkas jalur install yang dipakai proyek itu — detail dan urutan pastinya ada di catatan
rilis, ikuti itu:

1. Unlock bootloader — [UNLOCK-RELOCK.md](UNLOCK-RELOCK.md) — dan ambil dump 44 partisi.
2. Tulis `vendor_boot.img` LineageOS ke `vendor_boot_a` dari mode download (`spd_dump`).
   Image ini membawa blob AVB Meizu, jadi bootloader menerimanya.
3. Boot ke recovery (Lineage Recovery ada di `vendor_boot` ramdisk).
4. Gabung bagian `super.img`, tulis `super.img` (sparse) + `init_boot.img` dari recovery
   dengan `sparse_flash.py` yang disertakan rilis.
5. Format data sebelum boot pertama kalau datang dari stok.

## 3. Recovery: OrangeFox dan SHRP

Status per 2026-11-03: rilis terbaru masih berlabel **UNTESTED** di catatan rilisnya. Secara
teknis, recovery ini tinggal di `vendor_boot`, dan itu sebabnya build-nya menempelkan blob
AVB Meizu dari dump stok ke `vendor_boot` yang dihasilkan. Image lama tanpa blob itu tidak
termuat (berhenti di logo Meizu) — itu penyebab kegagalan yang tercatat pada 2026-11-02.

- Rilis: <https://github.com/tickernelz/android_device_meizu_mblu21-recovery/releases>
- Repo: <https://github.com/tickernelz/android_device_meizu_mblu21-recovery>

Sebelum memasang: baca catatan rilis. Kalau masih "UNTESTED", yang sesuai untuk pemulihan
adalah Lineage Recovery yang menyertai build LineageOS, atau stock recovery.

Hal yang perlu diketahui saat memakai recovery ini:

- Hanya satu recovery yang muat: Lineage Recovery, stock recovery, OrangeFox, dan SHRP saling
  menggantikan di `vendor_boot`.
- Update ROM/OTA menimpa `vendor_boot`, jadi recovery ikut hilang dan harus dipasang ulang.
- Recovery ini memakai `vendor_boot` yang sama untuk boot normal, jadi jangan menulis
  `vendor_boot` recovery ke HP yang masih dipakai tanpa cadangan `vendor_boot_a.bin`.

## 4. ZIP yang belum ada

Proyek ini tidak menyediakan paket ZIP resmi LineageOS/OrangeFox/SHRP untuk mBlu 21. Kalau
kelak ada rilis ZIP resmi, langkahnya ditambahkan di halaman ini **setelah** teruji di
perangkat. Sampai itu terjadi, satu-satunya jalur terdokumentasi adalah:

- pulih ke stok — [FLASH-STOCK.md](FLASH-STOCK.md)
- kembali dari LineageOS — [ROLLBACK.md](ROLLBACK.md)
- pasang LineageOS 24.0 — tautan rilis di §2

## 5. Kembali ke stok

Kalau percobaan gagal, ikuti [ROLLBACK.md](ROLLBACK.md). BootROM tidak bisa ditimpa, jadi mode
download selalu bisa dipakai untuk menulis firmware stok lagi.
