# Catatan pembicara

Disalin dari catatan di tiap slide , untuk berjaga kalau Canva tidak membawa catatan saat impor.

## 1. Cover

Selamat siang. Kami dari kelompok Rekayasa Data. Proyek kami membangun data pipeline dan data warehouse untuk melihat pergerakan device di Daerah Istimewa Yogyakarta, memakai Pandas dan DuckDB.

## 2. Pertanyaan

Pertanyaan utama kami: di mana pusat aktivitas siang hari di DIY, dari mana pengunjungnya berasal, dan bagaimana pola hariannya. Dari situ kami turunkan empat pertanyaan. Kapanewon mana yang ramai siang tapi sepi dini hari. Seperti apa siklus 24 jam, hari kerja dibanding akhir pekan. Tujuan mana yang menarik pengunjung dari luar wilayahnya. Dan seberapa besar perpindahan antar kabupaten atau kota.

## 3. Dataset

Datanya dari JogjaPulse, dataset agregat dari lokasi device, dengan periode 1 Desember 2021 sampai 30 April 2022. Ada dua file. Presence per jam punya 230.418 baris, dan asal-tujuan bulanan punya 9.894 baris. Ada tiga aturan yang kami pegang. Jumlah device unik tidak boleh dijumlahkan lintas jam, hari, atau wilayah. Baris yang hilang berarti kurang dari lima device, bukan nol. Dan yang dihitung adalah device, bukan orang.

## 4. Pipeline

Pipeline kami ada lima tahap dalam satu notebook. Pertama, CSV dibaca apa adanya dan disimpan ke staging raw. Kedua, data diprofil dan ditransformasi, hasilnya ke staging clean. Ketiga, dimensi dan fakta dibangun dengan SQL. Keempat, quality check. Kelima, analitik. Jumlah baris harus sama di setiap tahap, dan notebook bisa dijalankan ulang dari awal.

## 5. Skema DW

Skema warehouse-nya berbentuk galaxy. Ada dua tabel fakta, presence per jam dan asal-tujuan per bulan. Keduanya berbagi dimensi kapanewon, dan di tabel asal-tujuan dimensi itu dipakai dua kali, sebagai asal dan sebagai tujuan. Grain waktunya beda, jadi dim_date dan dim_hour melayani presence, sedangkan dim_month melayani asal-tujuan. Primary key hanya ada di dimensi. Integritas referensial kami cek di quality check.

## 6. Quality check

Sebelum dipakai untuk analitik, isi warehouse kami periksa. Rowcount sama di setiap tahap. Tidak ada null di kolom kunci, tidak ada orphan, dan tidak ada duplikat grain. Nilai berada dalam range yang wajar, total share per slot sama dengan satu, dan kode wilayah konsisten. Semua lolos. Satu catatan: tujuh kapanewon di Gunungkidul punya coverage di bawah lima puluh persen, jadi hasilnya kami tafsirkan hati-hati.

## 7. Pola harian

Untuk pola harian, bentuk siklus 24 jam hari kerja dan akhir pekan hampir sama. Dengan share, median rasio siang dibanding dini hari sekitar satu koma nol tiga, jadi tampak tidak ada kontras. Tapi kalau memakai rata-rata device, seluruh tujuh puluh satu kapanewon lebih ramai di siang hari, mediannya dua koma tujuh kali. Jadi pilihan metrik mengubah kesimpulan, dan itu kami tulis sebagai keterbatasan.

## 8. Asal dan tujuan

Untuk asal pengunjung, tiap kabupaten atau kota paling banyak dikunjungi orang dari wilayahnya sendiri. Di Sleman angkanya enam puluh tiga koma tujuh persen. Sleman juga jadi asal luar terbesar bagi kabupaten dan kota lain. Untuk tujuan, Depok menarik sekitar sembilan koma dua ribu device luar per bulan, paling banyak. Enam dari sepuluh tujuan teratas ada di Sleman. Analisis ini hanya memakai Desember sampai Februari.

## 9. Sampel dan event

Satu hal yang harus diingat: jumlah device yang teramati turun dari sekitar delapan koma tujuh ribu per jam di Desember menjadi sekitar seribu di April. Ini perubahan sampel, bukan perubahan mobilitas, jadi semua perbandingan antar waktu kami lakukan dengan share. Di kapanewon wisata, malam tahun baru dan libur satu Januari tidak menaikkan share secara konsisten.

## 10. Kesimpulan dan keterbatasan

Kesimpulannya, Depok menarik pengunjung luar terbanyak, siklus harian hari kerja dan akhir pekan mirip, dan pengunjung paling banyak datang dari wilayah sendiri, dengan Sleman sebagai asal luar terbesar. Keterbatasannya: yang dihitung adalah device, data di bawah lima device tidak dipublikasikan, ukuran sampel berubah antar waktu, asal-tujuan Maret sampai April berkualitas rendah, dan share bersifat zero-sum.

## 11. Penutup

Satu kalimat untuk dibawa pulang: yang dihitung di sini adalah device, bukan orang. Datanya dari JogjaPulse, Widyawan tahun dua ribu dua puluh enam, di Zenodo. Terima kasih.
