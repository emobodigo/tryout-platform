# Drizzle ORM, bukan Prisma

Skema database ditulis sebagai TypeScript biasa di `apps/api/src/db/schema/`, migrasi dihasilkan `drizzle-kit generate` menjadi berkas SQL yang ikut di-commit, dan koneksi memakai driver `mysql2` (`drizzle-orm` 0.45, `drizzle-kit` 0.31, `mysql2` 3.24).

Alasan: Drizzle murni JavaScript, jadi `npm install` tidak pernah mengunduh engine biner terpisah — mesin pengembangan ini sudah beberapa kali tersendat pada unduhan biner dari rilis GitHub. Migrasinya berkas SQL yang bisa dibaca dan disunting manusia, dan tidak ada proses `generate client` yang harus diulang tiap kali skema berubah.

Konsekuensi yang harus disadari: tidak ada eager loading bawaan seperti Prisma, jadi setiap query yang menyentuh beberapa tabel harus menulis join-nya sendiri — bentuk join yang keliru tetap lolos tipe, sehingga query lintas tabel perlu uji sendiri. Padanan GUI inspeksi data tidak selengkap Prisma Studio. Berpindah ORM berarti menulis ulang seluruh lapisan akses data.

## Alternatif yang ditolak

- **Prisma**: engine biner saat instalasi dan langkah `generate client` yang harus dijalankan ulang setiap kali skema berubah.
- **TypeORM**: dekorator dan entity tersebar, riwayat migrasinya rawan bentrok pada proyek yang bergerak cepat.
- **Kysely / SQL mentah**: kontrol penuh, tapi kehilangan definisi skema sebagai satu sumber kebenaran untuk tipe dan migrasi.
