# Memasang LineageOS, OrangeFox, dan SHRP

Halaman ini menjelaskan **status sebenarnya** per 2026-11-03 dan menunjuk ke rilis proyek
masing-masing. Kalau sebuah proyek belum merilis image, jangan pakai instruksi generik dari
internet: instruksi untuk perangkat Unisoc lain bisa membuat HP ini tidak boot.

> **MENGHAPUS DATA.** Memasang sistem operasi lain memformat `/data`. Pastikan dump stok
> sudah diunduh dan diverifikasi ([FLASH-STOCK.md](FLASH-STOCK.md)) sebelum mulai.

## 1. Aturan bootloader yang berlaku di HP ini (terukur)

| Fakta | Akibat |
|---|---|
| `vbmeta` hanya diterima kalau ditandatangani Meizu | `vbmeta` LineageOS/AOSP test key → HP berhenti di logo Meizu |
| `boot` dan `vendor_boot` diverifikasi AVB | `vendor_boot` asing (mis. recovery SHRP) ditolak |
| `init_boot` **tidak** diverifikasi | bisa ditambal untuk root |
| `fastboot flash` tidak bisa menulis | semua penulisan lewat mode download (`spd_dump`) atau recovery |

Karena itu: `boot`, `dtbo`, dan semua `vbmeta*` harus tetap stok.

## 2. LineageOS 24.0 (Android 17, unofficial)

Status per 2026-11-03: **boot dan dipakai harian di satu unit M416H** dengan matriks pengujian
sebagian. Rilis `lineage-24.0-20261103-UNOFFICIAL-mblu21` masih SELinux permissive dan
dm-verity mati untuk partisi logis. Build lokal dengan commit device tree `79d312d` dan
`0d92969` sudah diuji di HP pada 2026-11-03: SELinux enforcing, telepon dan SMS, serta
Volume Down tidak lagi me-reset HP; build itu belum dirilis tersendiri. Belum diuji:
Bluetooth pairing, VoLTE, GPS di luar ruangan, deep sleep, OTA.

- Rilis dan langkah install resmi proyek:
  <https://github.com/tickernelz/lineage_build_mblu21/releases>
- Sumber: [android_device_meizu_mblu21](https://github.com/tickernelz/android_device_meizu_mblu21),
  [android_vendor_meizu_mblu21](https://github.com/tickernelz/android_vendor_meizu_mblu21),
  [android_kernel_meizu_mblu21](https://github.com/tickernelz/android_kernel_meizu_mblu21),
  [lineage_build_mblu21](https://github.com/tickernelz/lineage_build_mblu21)
- Panduan flash mentah (sebelum AVB dipahami, jadi masih memuat jalur `fastboot` yang tidak
  berfungsi di HP ini): [FLASH.md](https://github.com/tickernelz/lineage_build_mblu21/blob/main/FLASH.md)

Ringkas jalur install yang dipakai proyek itu (detail dan urutan pastinya ada di catatan
rilis, ikuti itu):

1. Unlock bootloader — [UNLOCK-RELOCK.md](UNLOCK-RELOCK.md), dan ambil dump 42 image partisi.
2. Tulis `vendor_boot.img` ke `vendor_boot_a` dari mode download (`spd_dump`).
3. Boot ke recovery (Lineage Recovery ada di `vendor_boot`).
4. Gabung bagian `super.img`, tulis `super.img` (sparse) + `init_boot.img` dari recovery
   dengan `sparse_flash.py` yang disertakan rilis.
5. Format data sebelum boot pertama kalau datang dari stok.

## 3. Recovery: OrangeFox dan SHRP

Status per 2026-11-03: OrangeFox yang dibangun lokal dari branch `fix/fox-legacy-drm`
(`b3dbb0c`) dan di-reseal dengan blob AVB stok Meizu sudah diuji di satu unit M416H: UI dan
layar sentuh jalan, PIN dari LineageOS membuka `/data`, dan Android tetap boot normal. Rilis
OrangeFox yang sudah terbit dibangun sebelum perbaikan itu dan belum diuji. SHRP-Reborn belum
diuji. `vendor_boot` di HP ini diverifikasi AVB, jadi image recovery hanya termuat bila
di-reseal dengan blob AVB stok Meizu. Jangan mengandalkan recovery yang belum diuji untuk
menyelamatkan HP.

- Rilis: <https://github.com/tickernelz/android_device_meizu_mblu21-recovery/releases>
- Repo: <https://github.com/tickernelz/android_device_meizu_mblu21-recovery>

Kalau rilis di sana menyatakan belum diuji atau belum bisa boot, **jangan dipasang**; pakai
Lineage Recovery yang menyertai build LineageOS, atau stock recovery.

## 4. ZIP yang belum ada

Proyek ini tidak menyediakan paket ZIP resmi LineageOS/OrangeFox/SHRP untuk mBlu 21. Kalau
kelak ada rilis ZIP resmi, langkahnya akan ditambahkan di halaman ini **setelah** teruji di
perangkat. Sampai itu terjadi, satu-satunya jalur yang terdokumentasi adalah:

- pulih ke stok — [FLASH-STOCK.md](FLASH-STOCK.md)
- kembali dari LineageOS — [ROLLBACK.md](ROLLBACK.md)
- pasang LineageOS 24.0 — tautan rilis di §2

## 5. Kembali ke stok

Kalau percobaan gagal, ikuti [ROLLBACK.md](ROLLBACK.md). BootROM tidak bisa ditimpa, jadi
mode download selalu bisa dipakai untuk menulis firmware stok lagi.
