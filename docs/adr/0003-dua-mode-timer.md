# Dua Mode Timer: `global` dan `keras`

Tryout CPNS/BUMN/OJK asli memakai hitungan mundur global dengan kebebasan berpindah Soal, tetapi permintaan awal platform ini adalah jatah waktu per Soal. Diputuskan mendukung keduanya: `global` (satu hitungan mundur untuk seluruh Attempt) dan `keras` (jatah per Soal = total waktu ÷ jumlah Soal, sisa jatah hangus bila Peserta menekan Lanjut lebih awal, tidak bisa kembali ke Soal sebelumnya).

Alasan: satu mode saja memaksa salah satu jenis ujian dipeluk dengan buruk — global kehilangan tekanan waktu per Soal, keras menabrak kebiasaan CAT. Konsekuensi: mesin sesi punya dua jalur deadline (deadline Attempt dan deadline Soal berjalan), keduanya dihitung server agar menutup browser tidak menghentikan waktu. Di kedua mode, Peserta tetap boleh mengumpulkan Attempt lebih awal.
