# Tryout Platform

Platform publik untuk mengerjakan tryout ujian (CPNS, BUMN, OJK, dan sejenisnya): katalog tryout, bank soal, sesi pengerjaan bertimer, dan pembahasan yang dibatasi hak akses.

## Language

**Tryout**:
Satu paket yang dikerjakan Peserta — kumpulan Soal berurutan dengan total waktu, Mode Timer, dan Kuota Soal miliknya sendiri; bernaung di bawah satu Jenis Tryout.
_Avoid_: kuis, ujian online, test

**Jenis Tryout**:
Klasifikasi katalog (mis. CPNS, BUMN, OJK) yang menaungi sekumpulan Tryout sekaligus memegang Pengelompokan Soal beserta Bobot, Mode Penilaian, dan Nilai Ambang-nya.
_Avoid_: kategori tryout, tipe ujian, ujian

**Pengelompokan Soal**:
Belahan materi di dalam satu Jenis Tryout (mis. TWK, TIU, TKP) yang menaungi Soal sekaligus pemegang Bobot, Mode Penilaian, dan Nilai Ambang.
_Avoid_: kategori soal, subtes, subjek

**Bentuk Soal**:
Wujud pertanyaan: `single_choice` (tepat satu jawaban benar), `true_false` (satu pernyataan saja atau beberapa pernyataan sekaligus), atau `matching` (memasangkan dua lajur). Dipilih Admin saat menyusun Soal.
_Avoid_: tipe soal, jenis soal, question type

**Mode Penilaian**:
Cara sebuah Pengelompokan Soal memberi poin. `all_or_nothing`: semua kunci harus benar baru keluar poin (gaya TWK/TIU). `weighted`: setiap opsi jawaban membawa nilainya sendiri (gaya TKP).
_Avoid_: scoring mode, penilaian parsial

**Bobot**:
Poin satu Soal bila dinilai `utuh`, atau poin maksimum satu Soal bila dinilai `berbobot` — seragam untuk seluruh Soal dalam satu Pengelompokan.
_Avoid_: nilai soal, poin, mark

**Nilai Ambang**:
Skor minimum sebuah Pengelompokan Soal agar Attempt dinyatakan lulus. Kelulusan menuntut **semua** Pengelompokan melewati ambangnya masing-masing; tidak ada ambang total.
_Avoid_: passing grade, KKM, batas lulus

**Skor**:
Poin yang dikumpulkan sebuah Attempt pada satu Pengelompokan Soal; dibandingkan dengan Nilai Ambang untuk menentukan Lulus.
_Avoid_: nilai, point, grade

**Lulus**:
Status sebuah Attempt ketika semua Pengelompokan Soalnya mencapai Nilai Ambang-nya masing-masing.
_Avoid_: passed, kelulusan, naik

**Soal**:
Satu satuan uji beserta kunci jawabannya, wajib bernaung di bawah tepat satu Pengelompokan Soal, dengan Pembahasan yang bersifat opsional. Soal bisa dipensiunkan (nonaktif) tanpa mengubah Attempt lama yang sudah memakainya.
_Avoid_: pertanyaan, item, butir

**Bank Soal**:
Kumpulan Soal aktif milik satu Jenis Tryout yang menjadi bahan Undian Soal.
_Avoid_: repositori soal, pool soal

**Staging**:
Ruang penampungan Soal hasil Scraping yang belum diterima sebagai bagian Bank Soal.
_Avoid_: draft, inbox, antrean soal

**Scraping**:
Pengambilan Soal dan Pembahasan dari halaman web atas URL yang ditempel Admin — bukan crawler terjadwal.
_Avoid_: crawling, harvesting, spider

**Promosi**:
Tindakan Admin mengangkat Soal dari Staging menjadi Soal Bank Soal setelah disunting dan disetujui.
_Avoid_: publish, approve, konversi

**Pembahasan**:
Penjelasan atas kunci sebuah Soal; hanya terbaca oleh Peserta yang memiliki Hak Pembahasan, dan boleh dibuka kapan saja — termasuk saat Attempt masih berjalan.
_Avoid_: penjelasan, kunci, solusi, discussion

**Lampiran**:
Gambar yang menempel pada Soal atau Pembahasan; diunggah Admin dan disimpan di server sendiri.
_Avoid_: file, media, upload

**Hak Pembahasan**:
Penanda pada akun Peserta yang membuka akses Pembahasan; diberikan oleh Admin, tidak muncul otomatis dari perilaku Peserta.
_Avoid_: premium, langganan, subscription

**Kuota Soal**:
Jumlah Soal yang harus diambil dari sebuah Pengelompokan Soal untuk satu Tryout (mis. TWK 30, TIU 35, TKP 35); ditetapkan Admin.
_Avoid_: jumlah soal, jumlah pertanyaan

**Undian Soal**:
Hasil pengambilan Kuota Soal dari Bank Soal untuk sebuah Attempt; dikunci saat Attempt dimulai sehingga susunannya tidak berubah sampai Attempt selesai.
_Avoid_: acak soal, random pick, sample

**Attempt**:
Satu kesempatan seorang Peserta mengerjakan sebuah Tryout; memegang Undian Soal, Jawaban sementara, status pengerjaan, dan hasil nilainya. Deadline-nya dipegang server sehingga Attempt tetap berjalan meski browser ditutup, dan jumlah Attempt per Peserta tidak dibatasi.
_Avoid_: sesi, ujian, run, percobaan

**Riwayat**:
Daftar Attempt lama milik seorang Peserta beserta Skor dan status Lulus-nya; Tryout menampilkan Skor terbaik dari daftar itu.
_Avoid_: history, log pengerjaan

**Jawaban**:
Pilihan Peserta atas satu Soal di dalam sebuah Attempt; satu Soal hanya punya satu Jawaban per Attempt.
_Avoid_: response, answer sheet

**Mode Timer**:
Aturan waktu sebuah Tryout. `global`: satu hitungan mundur untuk seluruh Attempt dan Peserta bebas berpindah Soal. `strict`: setiap Soal punya jatah sendiri (total waktu ÷ jumlah Soal) dan Peserta berpindah paksa saat jatahnya habis tanpa bisa kembali; menekan Lanjut lebih awal menghanguskan sisa jatah Soal itu, dan Attempt berakhir saat Soal terakhir selesai atau jatahnya habis. Mode dipilih Admin per Tryout.
_Avoid_: timer type, mode waktu, timer per soal

**Peserta**:
Pengguna yang mengerjakan Tryout.
_Avoid_: user, siswa, member, murid

**Admin**:
Pengguna yang mengelola Jenis Tryout, Soal, dan Peserta.
_Avoid_: operator, pengelola

**Superadmin**:
Admin dengan kuasa penuh, termasuk mengelola akun Admin.
_Avoid_: root, owner
