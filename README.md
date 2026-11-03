# Meizu mBlu 21 (M416H) — image firmware stok

Image partisi stok yang dibaca langsung dari HP fisik lewat Unisoc BootROM (mode download),
lalu disimpan di sini supaya HP bisa dikembalikan ke kondisi pabrik tanpa harus mencari paket
vendor yang tidak pernah dirilis Meizu.

| | |
|---|---|
| Model | Meizu mBlu 21, M416H (board `ums9230_6h10`) |
| SoC | Unisoc UMS9230 / Tiger T606 |
| Penyimpanan | UFS, A/B, partisi dinamis (`super`) |
| Build stok | Android 14, `U01005` (`1745906188`), patch keamanan 2025-04-05 |
| Sumber | dump 44 partisi dari HP ini pada 2026-10-31, di-hash saat dump |
| Rilis | [`mblu21-stock-U01005-20261031`](../../releases/tag/mblu21-stock-U01005-20261031) |

## Dokumen

| Berkas | Isi |
|---|---|
| [docs/FLASH-STOCK.md](docs/FLASH-STOCK.md) | Pulih total ke stok: prasyarat host, masuk mode download, perintah `spd_dump` per partisi di Windows dan Linux, urutan tulis, verifikasi, troubleshooting |
| [docs/UNLOCK-RELOCK.md](docs/UNLOCK-RELOCK.md) | Unlock dan relock bootloader: status faktual, tool komunitas yang dipakai, syarat relock yang tidak membuat HP mati |
| [docs/ROLLBACK.md](docs/ROLLBACK.md) | Kembali ke stok dari LineageOS atau custom ROM lain |
| [docs/CUSTOM-ROM.md](docs/CUSTOM-ROM.md) | Memasang LineageOS 24.0 dan recovery (OrangeFox, SHRP): apa yang sudah jalan, apa yang belum |

## Isi rilis

Semua image adalah aset rilis GitHub, bukan berkas di dalam git (satu aset dibatasi 2 GiB,
jadi `super.bin` dipecah tiga bagian).

| Grup | Berkas |
|---|---|
| Boot chain | `boot_a.bin`, `init_boot_a.bin`, `vendor_boot_a.bin`, `dtbo_a.bin`, `dtb_a.bin`, `vbmeta_a.bin`, `vbmeta_system_a.bin`, `vbmeta_system_ext_a.bin`, `vbmeta_vendor_a.bin`, `vbmeta_product_a.bin`, `vbmeta_odm_a.bin` |
| Sistem | `super.bin` (dipecah `super.bin.part0`–`part2`): `system`, `system_ext`, `product`, `vendor`, `odm`, `vendor_dlkm`, `system_dlkm` |
| Bootloader | `splloader.bin`, `uboot_a.bin`, `uboot_log.bin`, `sml_a.bin`, `trustos_a.bin`, `teecfg_a.bin`, `pm_sys_a.bin` |
| Modem / DSP / NV | `l_modem_a.bin`, `l_ldsp_a.bin`, `l_gdsp_a.bin`, `l_agdsp_a.bin`, `l_deltanv_a.bin`, `l_fixnv1_a.bin`, `l_fixnv2_a.bin`, `l_runtimenv1.bin`, `l_runtimenv2.bin`, `prodnv.bin` |
| Kalibrasi / sisa | `persist.bin`, `calinv.bin`, `common_rs1_a.bin`, `common_rs2_a.bin`, `hypervsior_a.bin`, `avbmeta_rs_a.bin`, `sysdumpdb.bin`, `logo.bin`, `fbootlogo.bin`, `misc.bin`, `miscdata.bin`, `metadata.bin`, `pgpt.bin` |
| Image logis hasil `lpunpack` | `system_a.img`, `vendor_a.img`, `product_a.img`, `system_ext_a.img`, `odm_a.img`, `vendor_dlkm_a.img`, `system_dlkm_a.img` |
| Tabel & ringkasan | `partition_1793459407.xml` (tabel partisi dari HP), `dump_summary.json` |
| Tool | `tools-windows.zip` (`spd_dump.exe`, `gen_spl-unlock.exe`, `chsize.exe`, FDL loader, driver Unisoc), `tools-linux.zip` (biner Linux yang sama + aturan udev) |
| Checksum | `SHA256SUMS-full_dump.txt` (hash dump apa adanya, `super.bin` utuh), `SHA256SUMS-logical.txt` (hash image logis) |

`userdata` tidak ada di rilis ini: partisi itu berisi data pribadi dan memang tidak pernah
di-dump.

Nama berkas sama dengan nama partisi di `/dev/block/by-name` (mis. `boot_a.bin` →
partisi `boot_a`), kecuali `custom_exec_no_verify_65015f08.bin` (payload untuk tool, bukan
partisi) dan `partition_1793459407.xml` (tabel partisi).

## Verifikasi sebelum menulis apa pun

Linux:

```bash
gh release download mblu21-stock-U01005-20261031 -R tickernelz/android_stock_meizu_mblu21
cat super.bin.part0 super.bin.part1 super.bin.part2 > super.bin
sha256sum -c SHA256SUMS-full_dump.txt
sha256sum -c SHA256SUMS-logical.txt
```

Windows PowerShell:

```powershell
Get-FileHash .\boot_a.bin -Algorithm SHA256
Select-String -Path .\SHA256SUMS-full_dump.txt -Pattern boot_a
```

## Peringatan

- **`fastboot flash` tidak bisa menulis ke HP ini.** Di unit M416H, `fastboot flash` dan
  `fastboot boot` dijawab `Sending ... OKAY` lalu gagal di tahap `Writing`
  (`Status read failed (No such device)`) atau tidak pernah jalan. Semua penulisan di panduan
  ini lewat mode download Unisoc (`spd_dump`). `fastboot` di sini hanya berguna untuk
  `getvar` dan `reboot`.
- **Menulis partisi stok menghapus data.** Mengembalikan firmware tidak mengembalikan berkas
  pribadi; `userdata` tidak ada di rilis ini dan tidak bisa dipulihkan dari sini.
- **Jangan menulis partisi khas perangkat ke HP lain.** `prodnv`, `l_fixnv1_a`, `l_fixnv2_a`,
  `l_runtimenv1`, `l_runtimenv2`, `persist`, dan `miscdata` menyimpan data produksi HP ini —
  `l_runtimenv1.bin` antara lain memuat nomor 15 digit bergaya IMEI. Menulisnya ke perangkat
  lain ilegal di banyak negara dan merusak perangkat itu.
- Image ini bukan milik proyek ini: haknya ada pada Meizu dan Unisoc, dipublikasikan ulang
  hanya agar pemilik perangkat bisa memulihkan perangkatnya sendiri. Tanpa jaminan apa pun.
