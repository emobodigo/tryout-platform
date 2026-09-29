# Tiket Fase 1 — Backend API

Irisan kerja vertikal dari `docs/SPEC.md`: tiap tiket menembus skema → API → uji, sehingga bisa dibuktikan sendiri tanpa menunggu tiket lain.

**Status:** sudah dipublikasikan sebagai GitHub issue #1–#11 berlabel `ready-for-agent` di `emobodigo/tryout-platform`, lengkap dengan tautan dependensi native (blocked-by/blocking). Implementasi sengaja belum dimulai.

| Tiket | Issue | Tiket | Issue |
| --- | --- | --- | --- |
| T1 | #1 | T7 | #7 |
| T2 | #2 | T8 | #8 |
| T3 | #3 | T9 | #9 |
| T4 | #4 | T10 | #10 |
| T5 | #5 | T11 | #11 |
| T6 | #6 | | |

Kosakata tiket memakai istilah `CONTEXT.md`.

---

## T1 — Fondasi API & database
**Yang dibangun:** repo npm workspaces dengan `apps/api` (NestJS 11) yang hidup, tersambung ke Drizzle ORM + MariaDB lewat driver `mysql2`, punya dokumentasi Swagger, penyedia waktu yang bisa dipalsukan untuk pengujian deadline, dan harness uji terpisah di database `tryout_platform_test`.
**Terhalang oleh:** — (bisa mulai sekarang)
- [ ] `npm run start:dev` menyalakan API, `GET /api/health` menjawab 200
- [ ] Swagger terbuka di `/api/docs`
- [ ] `npm run db:generate` lalu `npm run db:migrate` membuat skema di `tryout_platform`; `.env.example` lengkap
- [ ] `npm run test` dan `npm run test:e2e` hijau di database uji

## T2 — Akun, peran, dan autentikasi
**Yang dibangun:** Peserta bisa mendaftar sendiri dan masuk; Admin bisa mengelola akun Peserta; endpoint terproteksi menolak yang tidak berhak.
**Terhalang oleh:** T1
- [ ] Registrasi + login mengembalikan access token (15 menit) dan refresh token (7 hari, berotasi)
- [ ] Refresh token bisa dicabut lewat logout, dan token lama tidak berlaku lagi
- [ ] `GET /api/auth/me` menampilkan peran dan status `hakPembahasan`
- [ ] Peserta mengganti password sendiri; Admin menyetel ulang password Peserta
- [ ] Endpoint `/api/admin/*` menolak Peserta (403); hanya Superadmin boleh mengelola akun Admin
- [ ] Akun nonaktif ditolak saat login (`ACCOUNT_INACTIVE`)
- [ ] Seeder: 1 Superadmin, 1 Admin, 2 Peserta (satu ber-`Hak Pembahasan`)

## T3 — Master Jenis Tryout & Pengelompokan Soal
**Yang dibangun:** Admin menyusun katalog: `Jenis Tryout` (CPNS/BUMN/OJK) dengan `Pengelompokan Soal` (TWK/TIU/TKP) yang membawa `Bobot`, `Mode Penilaian`, dan `Nilai Ambang`; publik bisa membacanya tanpa login.
**Terhalang oleh:** T2
- [ ] CRUD `Jenis Tryout` (slug unik, urutan, aktif) oleh Admin
- [ ] CRUD `Pengelompokan Soal` di dalam satu `Jenis Tryout`; nama unik per jenis
- [ ] `GET /api/jenis-tryout` dan `/:slug` terbuka tanpa token dan hanya memuat yang aktif
- [ ] Perubahan `Nilai Ambang` tercatat punya efek ke penilaian Attempt (diverifikasi di T7)
- [ ] Data demo: CPNS dengan TWK (utuh, bobot 5, ambang 25), TIU (utuh, 5, 30), TKP (berbobot, 5, 30)

## T4 — Bank Soal: tiga Bentuk Soal
**Yang dibangun:** Admin mengisi Bank Soal untuk ketiga bentuk soal lengkap dengan kunci, Pembahasan, dan nilai per item untuk mode `berbobot`; soal bisa dipensiunkan tanpa merusak Attempt lama.
**Terhalang oleh:** T3
- [ ] `pilihan_ganda`: opsi, tepat satu kunci
- [ ] `benar_salah`: satu atau beberapa pernyataan, masing-masing punya kunci
- [ ] `menjodohkan`: pasangan kiri–kanan
- [ ] Nilai per opsi/pernyataan/pasangan hanya dipakai bila `Mode Penilaian` = `berbobot`; totalnya tidak melebihi `Bobot`
- [ ] Pembahasan bersifat opsional; `aktif=false` mengeluarkan Soal dari calon undian
- [ ] Daftar Soal bisa disaring per `Pengelompokan Soal`, bentuk, dan status aktif
- [ ] Validasi menolak Soal tanpa kunci, tanpa opsi, atau pasangan kosong

## T5 — Lampiran gambar
**Yang dibangun:** Soal dan Pembahasan boleh memuat gambar; gambar bisa dibuka publik lewat URL sendiri tanpa membocorkan kunci.
**Terhalang oleh:** T4
- [ ] Unggah `image/jpeg|png|webp` ≤ 2 MB, ditolak 415/413 di luar itu
- [ ] Lampiran menempel pada Soal atau pada Pembahasan, dan punya urutan
- [ ] `GET /api/lampiran/:id` menyajikan berkas
- [ ] Menghapus Lampiran ikut membersihkan berkas di `storage/`
- [ ] Berkas tersimpan di luar repo aplikasi agar tidak ikut ter-commit

## T6 — Tryout & Kuota Soal
**Yang dibangun:** Admin menyusun paket `Tryout` dengan total waktu, `Mode Timer`, dan `Kuota Soal`; publik melihat daftar paket aktif; aktivasi paket yang bank soalnya kurang ditolak jelas.
**Terhalang oleh:** T4
- [ ] CRUD `Tryout` (slug unik, `totalWaktuDetik`, `modeTimer`, status `draf|aktif`)
- [ ] `PUT /api/admin/tryout/:id/kuota` menetapkan jumlah per `Pengelompokan Soal`
- [ ] Mengaktifkan Tryout dengan kuota > Soal aktif ditolak `422 QUOTA_EXCEEDS_BANK` beserta jumlah tersedia
- [ ] `GET /api/tryout` dan `/:slug` hanya memuat paket aktif, dan menyertakan kuota + total waktu
- [ ] Data demo: Paket 1 (global, 45 menit, 10/10/10) dan Paket 2 (keras, 45 menit, 10/10/10)

## T7 — Mesin Attempt: mode `global`
**Yang dibangun:** Satu percobaan utuh lewat API — mulai, undian soal terkunci, jawab, kumpulkan, keluar skor dan status `Lulus`.
**Terhalang oleh:** T6
- [ ] Mulai Attempt mengundi `Kuota Soal` dari Soal aktif, mengurutkan per `Pengelompokan Soal` (acak di dalam kelompok), dan mengunci susunannya
- [ ] `GET /api/attempts/berjalan?tryoutId=` mengembalikan Attempt yang sedang jalan, bukan error
- [ ] Mencoba memulai Attempt kedua untuk Tryout yang sama → `409 ATTEMPT_ALREADY_RUNNING`
- [ ] Penyajian Soal tidak pernah memuat kunci atau Pembahasan
- [ ] Menyimpan Jawaban bersifat idempoten dan boleh berulang untuk Soal yang sama
- [ ] Peserta boleh mengumpulkan lebih awal; Soal kosong bernilai 0
- [ ] Attempt yang deadline-nya lewat otomatis menjadi `selesai_otomatis` dan skornya dihitung
- [ ] Skor per `Pengelompokan Soal` benar untuk mode `utuh` maupun `berbobot`; `Lulus` hanya bila semua ambang tercapai
- [ ] Attempt milik Peserta lain → 403
- [ ] Menutup browser lalu kembali sebelum deadline mempertahankan sisa waktu yang benar (jam dipalsukan di uji)

## T8 — Mesin Attempt: mode `keras`
**Yang dibangun:** Percobaan berjatah per Soal: waktu habis berpindah paksa, tidak bisa kembali, sisa jatah hangus bila Lanjut lebih awal.
**Terhalang oleh:** T7
- [ ] Jatah per Soal = ⌊total ÷ jumlah Soal⌋ detik, sisa pembagian ditambahkan ke Soal terakhir
- [ ] Soal terbuka dihitung dari waktu server; jatah habis menutup Soal itu dengan nilai 0 tanpa perlu scheduler
- [ ] Menekan Lanjut lebih awal membuka Soal berikutnya dengan jatah penuh (sisa yang hangus tidak menumpuk)
- [ ] Permintaan menyentuh Soal yang sudah tertutup → `423 SOAL_LOCKED`
- [ ] Attempt berakhir setelah Soal terakhir ditutup atau jatahnya habis, lalu skor dihitung
- [ ] Angka pada uji memakai kasus 2700 detik ÷ 30 Soal = 90 detik, dan kasus tak bulat (mis. 5390 ÷ 110 → sisa ditambahkan ke Soal terakhir)

## T9 — Riwayat & hasil
**Yang dibangun:** Peserta melihat Attempt lamanya beserta rincian per Soal dan skor terbaiknya; Admin melihat daftar hasil sebuah Tryout.
**Terhalang oleh:** T7
- [ ] `GET /api/me/attempts` memuat waktu, skor per `Pengelompokan Soal`, dan status `Lulus`
- [ ] `GET /api/attempts/:id/hasil` memuat teks Soal, Jawaban Peserta, dan kunci benarnya
- [ ] Skor terbaik per Tryout ditandai pada daftar Riwayat
- [ ] `GET /api/admin/tryout/:id/attempts` memuat daftar Attempt semua Peserta
- [ ] Riwayat Attempt milik Peserta lain → 403 (Admin boleh)

## T10 — Pembahasan & Hak Pembahasan
**Yang dibangun:** Pembahasan bisa dibuka kapan saja oleh Peserta berhak — termasuk saat Attempt masih berjalan — dan tertutup bagi yang tidak berhak (ADR-0004).
**Terhalang oleh:** T7
- [ ] `GET /api/soal/:id/pembahasan` mengembalikan Pembahasan + lampirannya untuk Peserta ber-`Hak Pembahasan`
- [ ] Tanpa hak → `403 FORBIDDEN_PEMBAHASAN_ACCESS`
- [ ] Soal tanpa Pembahasan → `404 QUESTION_EXPLANATION_NOT_FOUND`
- [ ] Admin dan Superadmin selalu boleh
- [ ] Bisa dipanggil di tengah Attempt yang masih berjalan
- [ ] Mencabut `Hak Pembahasan` langsung menutup akses pada permintaan berikutnya

## T11 — Seeder demo, verifikasi ujung ke ujung, README
**Yang dibangun:** Satu perintah yang mengubah database kosong jadi platform siap dicoba, plus bukti dua mode timer benar-benar berjalan dari luar.
**Terhalang oleh:** T8, T9, T10
- [ ] `npm run seed` idempoten: CPNS (TWK/TIU/TKP), 30 Soal berpembahasan (10/10/10), Paket 1 & Paket 2, 1 Superadmin, 1 Admin, 2 Peserta
- [ ] `npm run demo` menjalankan satu Attempt penuh di mode `global` dan satu di mode `keras` lewat HTTP, lalu mencetak skor per `Pengelompokan Soal` dan status `Lulus`
- [ ] README: langkah dari nol (nyalakan MySQL XAMPP, migrasi, seed, jalankan, buka Swagger) + daftar variabel env
- [ ] `npm run test` dan `npm run test:e2e` hijau dari database kosong
