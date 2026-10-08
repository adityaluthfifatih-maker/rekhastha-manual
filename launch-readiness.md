# Audit Kesiapan Pemasaran Tahap Awal

**Tanggal audit:** 8 Oktober 2026  
**Cakupan:** source code dan workflow development Rekhastha/Jakarta Gig Finder, konfigurasi artifact, pemindaian keamanan, build/test, serta halaman yang terlihat tanpa login. Ini bukan pentest independen, tinjauan hukum, atau verifikasi produksi.

## Keputusan

**Belum siap dipasarkan secara publik.** Belum ada deployment produksi yang terdaftar. Setelah perubahan kode pada audit ini, blocker yang masih terbuka mencakup kerentanan dependensi runtime serta klaim hukum/privasi dan perlindungan di materi pengguna yang belum dapat dibuktikan oleh implementasi yang diperiksa. Perbaikan akses file hanya berlaku pada source code saat ini; belum diverifikasi pada deployment produksi.

Jangan membuka pendaftaran publik sebelum blocker yang tersisa ditutup. Pilot terbatas baru layak dipertimbangkan setelah dependensi diperbaiki, klaim/pemberitahuan privasi ditinjau, dan operasi transfer manual memiliki penanggung jawab serta prosedur tertulis.

## Blocker sebelum publikasi

### P0 — File identitas dan unggahan dapat diakses tanpa login (diperbaiki pada source code)

Saat audit awal, `artifacts/api-server/src/app.ts` memasang direktori unggahan melalui `express.static` pada `/api/uploads` tanpa autentikasi. Pengujian HEAD tanpa login pada file unggahan menghasilkan HTTP 200. Perubahan saat ini menghapus mount publik dan menggunakan rute privat yang memeriksa kepemilikan KYC/CV atau hak admin/peserta chat. Dokumen lamaran hanya dapat diambil oleh pemilik atau perusahaan pada lamaran terkait.

**Dampak temuan awal:** siapa pun yang memperoleh URL dapat mengambil lampiran; path acak tidak memberikan kerahasiaan. Ini mencakup kategori data yang berpotensi sangat sensitif, seperti foto KTP/selfie dan CV.

**Status:** kontrol akses dan tes untuk akses anonim/lintas akun ditambahkan pada source code. URL chat lama tetap didukung, tetapi aksesnya ditentukan dari percakapan paling awal yang mencatat URL tersebut. Sebelum rilis, jalankan tes dan smoke test pada staging/produksi, pastikan penyimpanan file deployment sama, dan tinjau file lama yang mungkin pernah diakses publik; perubahan kode tidak dapat membatalkan salinan yang sudah diunduh.

### P0 — Kerentanan pada dependensi produksi

`pnpm audit --prod` menemukan **1 kritis, 3 tinggi, 3 sedang, dan 2 rendah** pada dependensi produksi. Temuan paling penting:

- `proxy-addr` 2.0.7 — kritis, CVE-2026-90711; perbaikan tersedia pada 2.0.8 atau lebih baru.
- `multer` 2.2.0 — tiga temuan tinggi (CVE-2026-77078, CVE-2026-77037, CVE-2026-82333); satu advisori sedang tambahan memerlukan 2.4.0 atau lebih baru untuk menutup seluruh isu Multer yang terdeteksi.
- `qs` 6.15.3 — dua temuan sedang; perbaikan tersedia pada 6.16.0 atau lebih baru.

**Sebelum rilis:** perbarui rentang dependensi dan lockfile, lalu ulangi audit produksi, typecheck, build, dan tes. Angka ini hanya mencakup dependensi produksi; pemindaian workspace penuh melaporkan lebih banyak temuan pada seluruh dependency tree.

### P0 — Informasi privasi dan klaim perlindungan belum layak dipublikasikan

Formulir daftar menyebut “Kebijakan Privasi”, tetapi yang ditemukan adalah teks, bukan tautan ke kebijakan tersendiri. Produk mengumpulkan NIK/KTP, selfie, NPWP, dan detail rekening, sehingga pengguna perlu mendapat penjelasan yang jelas tentang tujuan, retensi, akses, pembagian, dan permintaan penghapusan/koreksi.

Syarat & Ketentuan dan UI membuat klaim mengenai KYC biometrik otomatis, aktivasi BPJS/asuransi, arbitrase yang mengikat, dan “imunitas hukum absolut”. Pencarian implementasi tidak menemukan layanan/policy yang membuktikan sebagian klaim tersebut; alur KYC yang tercatat menggunakan verifikasi admin. Jangan menjanjikan manfaat asuransi/BPJS atau hasil hukum sebelum mekanisme dan dasar operasionalnya dikonfirmasi. Minta penasihat hukum Indonesia meninjau alur kerja, penanganan data pribadi, hubungan kerja, transfer/payout, sengketa, dan naskah persetujuan. Temuan ini bukan pendapat hukum.

**Sebelum rilis:** terbitkan kebijakan privasi yang dapat dibuka; revisi setiap klaim yang tidak dapat dibuktikan; minta tinjauan hukum dan konfirmasi tertulis atas status escrow, kustodi dana, perlindungan pekerja, dan pengungkapan data.

## Siap secara teknis, dengan syarat

### Halaman publik dan autentikasi

- Pengunjung tanpa login langsung melihat formulir masuk/daftar; belum ada landing page publik yang menjelaskan nilai produk, cara kerja, dukungan, dan biaya.
- Metadata masih generik (“Jakarta Gig Finder”) dan deskripsi masih berupa placeholder “built on Replit”. `/sitemap.xml` membalas HTML aplikasi (status 200), bukan sitemap.
- UI menampilkan klaim “15.000+ Pekerja · 3.000+ Perusahaan”; audit ini tidak dapat memverifikasi angka tersebut terhadap data produksi. Validasi sumbernya sebelum menggunakannya sebagai bukti sosial.
- Tombol “Lanjutkan dengan Google” tampil, tetapi `GOOGLE_CLIENT_ID` dan `GOOGLE_CLIENT_SECRET` tidak tersedia. Kode mengarahkan kembali dengan status `not_configured`. Konfigurasikan dan uji OAuth, atau sembunyikan tombol sampai aktif.
- Rute login/pendaftaran pengguna belum menunjukkan rate limit; validasi password menerima minimal enam karakter. Terapkan proteksi percobaan login dan kebijakan kredensial yang memadai.

### Pembayaran, escrow, dan operasi

Alur posting saat ini menggunakan transfer manual dan review admin. API pembayaran/payout Midtrans mengembalikan HTTP 410; pembayaran otomatis tidak aktif. Pengguna diarahkan mengirim bukti lewat WhatsApp, lalu admin memverifikasi.

Ini bisa dipakai untuk pilot ber-volume rendah **hanya** bila ada petugas yang memeriksa pembayaran, SLA aktivasi dan payout, rekonsiliasi, proses pengembalian/sengketa, kanal bantuan bisnis yang terkonfirmasi, serta catatan transaksi. Kode status “escrow” tidak membuktikan dana ditahan pada rekening kustodian atau skema escrow yang sah; pastikan hal itu dengan penasihat hukum/keuangan sebelum menjanjikan “dana terjamin”.

## Hasil verifikasi

- **Status deployment:** tidak ada deployment produksi aktif pada saat audit.
- **Typecheck frontend web dan API:** lulus secara terpisah.
- **Tes API setelah perbaikan kontrol file:** 22 lulus pada 3 file tes (`admin.test.ts`, `pricing.test.ts`, `uploads.test.ts`). Empat tes rute unggahan mencakup akses anonim, pemilik/admin KYC, pemilik/pemberi kerja CV, peserta chat, URL chat lama, dan penolakan lintas percakapan.
- **Smoke test file privat:** permintaan HEAD anonim pada rute unggahan menghasilkan HTTP 401 setelah API dibangun ulang dan workflow dijalankan. Ini memverifikasi development, bukan deployment produksi.
- **Build API:** lulus.
- **Build web dan film promosi:** lulus setelah menyediakan `PORT` dan `BASE_PATH` seperti yang diwajibkan oleh konfigurasi artifact.
- **Typecheck seluruh workspace:** gagal pada artefak film promosi (antara lain tipe DOM dan Framer Motion). Ini terpisah dari typecheck web/API yang lulus, tetapi membuat pemeriksaan monorepo belum bersih.
- **Build frontend:** berhasil, namun bundle utama sekitar 1,77 MB sebelum gzip dan memunculkan peringatan chunk lebih dari 500 kB; optimasi pemuatan akan membantu perangkat/jaringan lambat.
- **Pemeriksaan escrow dan konsistensi biaya:** workflow terkait selesai; tes escrow yang tercatat lulus.
- **SAST:** satu temuan medium tentang redirect ditinjau manual; tujuan redirect dibentuk ke URL Google yang tetap sehingga tampak sebagai false positive. **Pemindaian rahasia/privacy:** tidak ada temuan.

## Urutan kerja yang disarankan

1. Verifikasi di staging bahwa file KTP, selfie, CV, dan lampiran privat hanya dapat diakses oleh pihak berwenang; tinjau paparan file lama.
2. Tambal dependensi runtime, lalu jalankan ulang audit keamanan produksi.
3. Selesaikan kebijakan privasi, tinjauan hukum, dan semua klaim escrow/asuransi/KYC yang belum didukung.
4. Siapkan landing page Indonesia, metadata/sitemap, angka pengguna yang bersumber, tautan dukungan, serta tampilkan hanya opsi login yang berfungsi.
5. Dokumentasikan operasi transfer/payout/sengketa dan pastikan petugas pilot tersedia; tambah tes end-to-end untuk daftar/login, lamar, pembayaran, review admin, dan akses file.
6. Rapikan typecheck workspace dan lakukan smoke test deployment/staging sebelum pemasaran publik.

**Perubahan pada audit ini:** kontrol akses privat untuk unggahan dan tes API terkait ditambahkan, serta dokumen audit dan threat model diperbarui. Tidak ada deployment atau perubahan data produksi.
