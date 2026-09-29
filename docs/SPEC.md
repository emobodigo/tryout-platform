# SPEC — Tryout Platform (Fase 1: Backend API)

Istilah domain dipakai persis seperti `CONTEXT.md`; keputusan besar tercatat di `docs/adr/`.
Lingkup fase 1: **backend saja** — API REST + Swagger + seeder, tanpa frontend.

## 1. Lingkup

Backend API yang membuat satu platform tryout berjalan ujung ke ujung:

- Master `Jenis Tryout` (CPNS, BUMN, OJK) beserta `Pengelompokan Soal` (TWK/TIU/TKP) dan aturan skornya.
- `Bank Soal` tiga bentuk: `pilihan_ganda`, `benar_salah`, `menjodohkan`, lengkap dengan `Pembahasan` dan `Lampiran` gambar.
- `Tryout` (paket) dengan total waktu, `Mode Timer`, dan `Kuota Soal`.
- `Attempt`: undian soal, jawaban bertimer, pengumpulan, skor per `Pengelompokan Soal`, status `Lulus`.
- `Riwayat` serta `Pembahasan` yang dibatasi `Hak Pembahasan`.
- Akun: registrasi mandiri, peran `Peserta`/`Admin`/`Superadmin`.

## 2. Di luar lingkup fase 1

Sengaja ditunda (rancangan tetap dicatat, kode menyusul):

| Ditunda | Alasan |
| --- | --- |
| Modul `Scraping` + layar `Staging`/`Promosi` | belum ada situs target untuk diuji |
| Ekspor CSV hasil | cukup daftar hasil di panel Admin nanti |
| Panel Admin, seluruh frontend SolidJS | fase 2 |
| Berkas deploy (Dockerfile/compose) | fokus dev lokal dulu |
| Email (verifikasi, kirim tautan reset) | lupa password disetel ulang Admin |
| Pembayaran/paket berbayar | katalog bebas diakses |
| Statistik per Soal (persentase benar) | menyusul |

## 3. Model domain

Hierarki: `Jenis Tryout` → (`Pengelompokan Soal` → `Soal`) dan `Jenis Tryout` → `Tryout` → `Attempt` → `Riwayat`.

```
User              id, email (unik), passwordHash, nama, peran, hakPembahasan, status, createdAt, updatedAt
RefreshToken      id, userId, tokenHash, kedaluwarsaPada, dicabutPada, createdAt

JenisTryout       id, slug (unik), nama, deskripsi, urutan, aktif, timestamps
Pengelompokan     id, jenisTryoutId, nama, urutan, bobot, modePenilaian(utuh|berbobot), nilaiAmbang, timestamps
                  unik: (jenisTryoutId, nama)

Soal              id, pengelompokanId, bentuk(pilihan_ganda|benar_salah|menjodohkan), teks, pembahasan (nullable),
                  aktif, timestamps
OpsiJawaban       id, soalId, teks, urutan, kunci(boolean), nilai(int, 0..bobot)      -- pilihan_ganda
Pernyataan        id, soalId, teks, urutan, kunci(boolean), nilai(int, 0..bobot)      -- benar_salah
Pasangan          id, soalId, kiri, kanan, urutan, nilai(int, 0..bobot)               -- menjodohkan
                  jumlah Soal.kunci benar untuk pilihan_ganda tepat 1; benar_salah boleh 1 pernyataan saja

Lampiran          id, soalId, pada(soal|pembahasan), path, mime, ukuranByte, urutan, createdAt

Tryout            id, jenisTryoutId, slug (unik), judul, deskripsi, totalWaktuDetik, modeTimer(global|keras),
                  status(draf|aktif), timestamps
KuotaSoal         id, tryoutId, pengelompokanId, jumlah                                 unik: (tryoutId, pengelompokanId)

Attempt           id, tryoutId, pesertaId, modeTimer, totalWaktuDetik, jumlahSoal, jatahSoalDetik (keras),
                  mulaiPada, deadlinePada, status(berjalan|selesai|selesai_otomatis), dikumpulkanPada, timestamps
AttemptSoal       id, attemptId, soalId, pengelompokanId, urutan, jatahDetik, dibukaPada, ditutupPada, terkunci
                  unik: (attemptId, soalId)
Jawaban           id, attemptSoalId, dijawabPada, updatedAt
JawabanItem       id, jawabanId, itemId, itemJenis(opsi|pernyataan|pasangan), nilaiTeks (nullable), benar(boolean)
SkorPengelompokan id, attemptId, pengelompokanId, skor, nilaiAmbang, lulus
```

## 4. Aturan domain

### 4.1 Undian Soal

- Saat Attempt dimulai, ambil `jumlah` dari setiap `KuotaSoal` secara acak dari `Soal` **aktif** pada `Pengelompokan Soal` itu.
- Susunan dikunci ke `AttemptSoal`; urutan = `Pengelompokan Soal` berurutan, lalu acak di dalam kelompok (benih = id Attempt, supaya bisa direproduksi saat investigasi).
- Kurang dari kuota → **422** `QUOTA_EXCEEDS_BANK` (dicek saat Tryout diaktifkan **dan** saat Attempt dimulai).
- Satu `Attempt` berjalan per Peserta per Tryout. Mencoba mulai lagi → **409** `ATTEMPT_ALREADY_RUNNING` berisi id Attempt yang sedang jalan (dipakai frontend untuk melanjutkan).

### 4.2 Mode Timer

Deadline dihitung server. Menutup browser tidak menghentikan apa pun.

- `global`: satu `deadlinePada` = `mulaiPada + totalWaktuDetik`. Peserta bebas berpindah Soal.
- `keras`: `jatahSoalDetik` = ⌊totalWaktuDetik ÷ jumlahSoal⌋; sisa detik pembagian ditambahkan ke Soal terakhir. Jatah per Soal tidak menumpuk: menekan Lanjut lebih awal menghanguskan sisanya. Tidak ada kembali ke Soal sebelumnya. Attempt berakhir setelah Soal terakhir ditutup atau jatahnya habis.
- Kemajuan Soal di mode `keras` dihitung **malas** (lazy) dari waktu server: setiap permintaan menghitung Soal mana yang sedang terbuka; Soal yang jatahnya lewat tanpa jawaban tertutup dengan nilai 0. Tidak ada scheduler/cron.
- Aturan di atas berlaku juga saat Peserta menutup browser: menutup browser tidak menghentikan jatah, dan Soal yang terlewat tertutup bernilai 0.
- Setiap Attempt berakhir otomatis saat deadline lewat: status `selesai_otomatis`, skor dihitung saat itu juga.

### 4.3 Penilaian

- `utuh`: nilai `Bobot` penuh bila seluruh kunci benar; selain itu 0.
- `berbobot`: jumlah `nilai` dari item yang benar (opsi/pernyataan/pasangan), diisi Admin dan totalnya ≤ `Bobot`.
- `Skor` per `Pengelompokan Soal`, dibandingkan `Nilai Ambang` masing-masing.
- `Lulus` = **semua** `Pengelompokan Soal` mencapai ambangnya. Tidak ada ambang total.
- Soal tak dijawab = 0 poin dan tetap tampil di `Riwayat` sebagai kosong.
- Skor dihitung sekali, saat Attempt dikumpulkan atau saat deadline lewat; disimpan di `SkorPengelompokan`.

### 4.4 Pembahasan & Hak Pembahasan

- `Pembahasan` tidak pernah ikut dalam penyajian Soal saat Attempt berjalan.
- Endpoint khusus `GET /api/soal/:id/pembahasan`: hanya `Peserta` ber-`Hak Pembahasan` (dan Admin/Superadmin). Boleh dipanggil kapan saja — termasuk di tengah pengerjaan (lihat ADR-0004).
- Tanpa hak → **403** `FORBIDDEN_PEMBAHASAN_ACCESS`.
- `Hak Pembahasan` diberikan/dicabut Admin lewat editor Peserta, bukan hasil otomatis.

### 4.5 Riwayat

- Peserta melihat daftar `Attempt`-nya sendiri (waktu, skor per pengelompokan, status `Lulus`) dan detail per Soal: teks Soal, Jawaban-nya, kunci benarnya.
- `Pembahasan` pada halaman Riwayat tetap lewat endpoint pembahasan di atas.

### 4.6 Akun & peran

- Registrasi mandiri (email + password), tanpa verifikasi email.
- `Peserta`: mengerjakan Tryout dan melihat Riwayat. `Admin`: mengelola master, Bank Soal, Tryout, dan Peserta. `Superadmin`: semua itu plus mengelola akun Admin.
- Lupa password: Admin menyetel ulang password Peserta. Peserta bisa mengganti password sendiri saat sudah masuk.
- Akun bisa dinonaktifkan Admin; Peserta nonaktif ditolak saat login (**403** `ACCOUNT_INACTIVE`).

## 5. Kontrak API

Prefiks `/api`. Autentikasi `Authorization: Bearer <access token>`. Dokumentasi Swagger di `/api/docs`.

Error selalu berbentuk kode Inggris + parameter (teks Indonesia ada di berkas locale frontend):

```json
{ "error": { "code": "QUOTA_EXCEEDS_BANK", "params": { "pengelompokanId": 7, "butuh": 30, "tersedia": 20 } } }
```

| Area | Endpoint |
| --- | --- |
| Auth | `POST /auth/register`, `POST /auth/login`, `POST /auth/refresh`, `POST /auth/logout`, `GET /auth/me`, `PATCH /auth/me/password` |
| Publik | `GET /jenis-tryout`, `GET /jenis-tryout/:slug`, `GET /tryout`, `GET /tryout/:slug`, `GET /health` |
| Admin master | `CRUD /admin/jenis-tryout`, `CRUD /admin/jenis-tryout/:id/pengelompokan` |
| Admin soal | `CRUD /admin/soal` (filter pengelompokan/bentuk/aktif), `POST /admin/soal/:id/lampiran`, `DELETE /admin/lampiran/:id` |
| Lampiran | `GET /lampiran/:id` (berkas publik, tanpa kunci jawaban) |
| Admin tryout | `CRUD /admin/tryout`, `PUT /admin/tryout/:id/kuota` |
| Admin peserta | `GET /admin/peserta`, `PATCH /admin/peserta/:id` (status, hakPembahasan), `PATCH /admin/peserta/:id/password` |
| Admin hasil | `GET /admin/tryout/:id/attempts` |
| Peserta — Attempt | `POST /tryout/:id/attempts` (mulai), `GET /attempts/berjalan?tryoutId=`, `GET /attempts/:id` (Soal + sisa waktu), `PUT /attempts/:id/jawaban`, `POST /attempts/:id/lanjut` (mode keras), `POST /attempts/:id/kumpulkan` |
| Riwayat | `GET /me/attempts`, `GET /attempts/:id/hasil` |
| Pembahasan | `GET /soal/:id/pembahasan` |

`GET /attempts/:id` mengembalikan daftar Soal **tanpa kunci dan tanpa pembahasan**, plus `sisaDetik` (mode `global`) atau `sisaJatahDetik` + `nomorSoalAktif` (mode `keras`).

## 6. Teknis

- Monorepo npm workspaces: `apps/api` (NestJS 11 + TypeScript). `apps/web` (SolidJS) menyusul di fase 2.
- Drizzle ORM (`drizzle-orm` 0.45 + `drizzle-kit` 0.31, driver `mysql2` 3.24), database `tryout_platform` di MariaDB XAMPP (`127.0.0.1:3306`, user `root` tanpa password) — lihat ADR-0001 dan ADR-0005. Skema ditulis sebagai TypeScript di `apps/api/src/db/schema/`, migrasinya berkas SQL hasil `drizzle-kit generate` yang ikut di-commit. Database uji terpisah: `tryout_platform_test`.
- `@nestjs/config` + `.env` (disertai `.env.example`), `class-validator` + `ValidationPipe` global (whitelist + transform).
- Unggah: multer disk storage ke `storage/lampiran/`, batas 2 MB, mime `image/jpeg|png|webp`; disajikan `GET /api/lampiran/:id`.
- Token: access 15 menit, refresh 7 hari dengan rotasi; refresh disimpan sebagai hash sehingga bisa dicabut (logout).
- Waktu: semua timestamp UTC di database, durasi selalu dalam detik. Zona tampil (WIB) urusan frontend.
- Jam disuntik lewat penyedia waktu sendiri agar deadline bisa diuji tanpa menunggu waktu nyata.
- Pengujian: Jest unit + e2e (supertest) di atas `tryout_platform_test`. Seeder menyiapkan data demo.

## 7. Rencana kerja

Rincian tiket ada di `docs/TICKETS.md` (11 irisan, urut ketergantungan). Setelah daftar itu disetujui, tiket dipublikasikan sebagai GitHub issue dengan label `ready-for-agent`.

## 8. Keputusan yang sudah ditutup

1. **Mode `keras` saat browser ditutup** — jatah tetap berjalan, Soal yang terlewat tertutup bernilai 0 (lihat §4.2). Ditegaskan sadar: konsisten dengan "sisa jatah hangus" dan tidak bisa dicurangi dengan menutup browser.
2. **Kredensial database** — `root` tanpa password bawaan XAMPP untuk dev lokal; user khusus menyusul saat deploy.
3. **`benar_salah` satu pernyataan** — Soal dengan tepat satu `Pernyataan`, bukan bentuk tersendiri.
