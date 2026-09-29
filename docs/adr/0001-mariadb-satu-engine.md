# MariaDB sebagai satu-satunya database

Mesin pengembangan (macOS 12) tidak punya Docker dan tidak punya PostgreSQL; MariaDB 10.4 sudah tersedia lewat XAMPP dan sudah dipakai proyek lain di mesin yang sama. Diputuskan memakai MariaDB untuk dev maupun produksi supaya dialect dan perilaku query identik di kedua sisi, dengan konsekuensi: dialect MySQL/MariaDB lewat driver `mysql2` (lihat ADR-0005), dan fitur khas PostgreSQL (JSONB, partial index, array) tidak tersedia. Pindah engine di kemudian hari berarti migrasi penuh, jadi keputusan ini sadar diambil meski PostgreSQL secara umum lebih kaya fitur.

## Alternatif yang ditolak

- **PostgreSQL lokal**: butuh build dari source di mesin ini (tanpa Docker, bottle Homebrew tidak tersedia untuk macOS 12) — biaya setup tidak sebanding untuk fase 1.
- **SQLite dev → PostgreSQL produksi**: dua engine berarti dialect, tipe kolom, dan perilaku constraint berbeda antara apa yang diuji dan apa yang dipakai Peserta.
