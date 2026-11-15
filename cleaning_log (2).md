# Cleaning Log

_Dihasilkan otomatis oleh `notebooks/02_cleaning.ipynb` pada 2026-11-15. Setiap entri dicatat saat langkahnya dikerjakan._

## Ringkasan

- Data mentah: **8600 baris, 18 kolom** (`data/raw/bedtime_screentime_sleep_debt_kotor.csv`, tidak diubah)
- Data bersih: **8500 baris, 18 kolom** (`data/processed/sleep_data_bersih.csv`)
- Baris dibuang: **100** (seluruhnya duplikat persis). Tidak ada baris dibuang karena nilai kosong atau outlier.
- Jumlah entri keputusan: **22**
- SHA-256 `data/raw` (bukti tidak berubah): `0455cb9c2c0f03f1e577aa24aaecdbda8c7d334ca39a4ed175c269ba0c65b03e`

## Tabel ringkas

| # | Langkah | Kolom | Baris terdampak | Keputusan | Oleh |
|---|---|---|---|---|---|
| 1 | L1 Strip spasi | semua kolom teks | 488 | strip() pada seluruh kolom teks, sebelum deduplikasi | kode |
| 2 | L2 Duplikat | seluruh kolom | 100 | drop_duplicates(keep="first") | kode |
| 3 | L3 Tipe teks -> angka | age | 30 | Regex ambil angka, ubah ke float; placeholder tanpa digit menjadi NaN | kode |
| 4 | L3 Tipe teks -> angka | bedtime_phone_minutes | 50 | Regex ambil angka, ubah ke float; placeholder tanpa digit menjadi NaN | kode |
| 5 | L3 Tipe teks -> angka | rem_sleep_pct | 24 | Regex ambil angka, ubah ke float; placeholder tanpa digit menjadi NaN | kode |
| 6 | L4 Kategori | gender | 134 | Normalisasi oleh kode; F -> Female dan M -> Male lewat peta manual | kode + manual |
| 7 | L4 Kategori | sleep_debt_category | 40 | Peta manual eksplisit ke 4 label baku, dilindungi assert | kode + manual |
| 8 | L4 Kategori | occupation_type, chronotype, primary_bedtime_app | 0 | Normalisasi kunci (strip, lowercase, pemisah seragam) lalu petakan ke label baku | kode |
| 9 | L4 Kategori | blue_light_filter_active | 0 | Petakan ke 1/0 (Int64 agar NaN tetap boleh) | kode |
| 10 | L5 Domain | age | 8 | Ubah menjadi NaN; baris dipertahankan | kode |
| 11 | L5 Domain | screen_brightness_pct | 6 | Ubah menjadi NaN; baris dipertahankan | kode |
| 12 | L5 Domain | caffeine_post_5pm_mg | 4 | Ubah menjadi NaN; baris dipertahankan | kode |
| 13 | L5 Domain | sleep_latency_min | 4 | Ubah menjadi NaN; baris dipertahankan | kode |
| 14 | L5 Domain | total_sleep_hours | 8 | Ubah menjadi NaN; baris dipertahankan | kode |
| 15 | L5 Domain | kolom lain (bedtime_phone_minutes, physical_activity_min, deep_sleep_pct, rem_sleep_pct, morning_alarm_snoozes, next_day_fatigue_score) | 0 | Tidak ada tindakan | kode |
| 16 | L6 Missing | sleep_latency_min, deep_sleep_pct | 514 | Biarkan NaN; analisis per pasangan dengan dropna() dan n dilaporkan | kode + manual |
| 17 | L6 Missing | rem_sleep_pct, screen_brightness_pct, caffeine_post_5pm_mg, total_sleep_hours, age | 418 | Biarkan NaN | manual |
| 18 | L6 Missing | occupation_type, chronotype | 297 | Isi label "Unknown" (baris dipertahankan) | manual |
| 19 | L7 Data pribadi | user_id | 8500 | HMAC-SHA256 dengan garam dari SALT_EDA, dipotong 12 heks; kolom asli dihapus | kode |
| 20 | L7 Data pribadi | age | 8500 | Generalisasi menjadi kelompok umur 18-29 / 30-44 / 45-65; kolom age dihapus dari keluaran | kode |
| 21 | L7 Data pribadi | occupation_type, gender, kelompok_umur | 17 | Penyamaran lokal bertahap: baris pada grup < 5 disamarkan pada occupation_type, lalu gender, lalu kelompok_umur bila masih perlu, sampai k >= 5 | kode + manual |
| 22 | L7 Data pribadi | seluruh dataset | 8500 | Verifikasi: tidak ada string "USR-" di keluaran dan kolom user_id/age asli sudah tidak ada | kode |

## Rincian keputusan

### 1. L1 Strip spasi: semua kolom teks

- **Masalah:** Spasi di awal/akhir teks memecah kategori dan ID yang sama
- **Keputusan:** strip() pada seluruh kolom teks, sebelum deduplikasi
- **Alasan:** Spasi tidak membawa informasi; efeknya hanya memecah grup
- **Alternatif yang dipertimbangkan:** Strip hanya kolom kunci (ditolak: kolom kategori sama rawannya)
- **Bukti dampak:** 500 sel dirapikan; unik berkurang pada gender, occupation_type, chronotype, primary_bedtime_app
- **Baris terdampak:** 488
- **Diputuskan oleh:** kode

### 2. L2 Duplikat: seluruh kolom

- **Masalah:** Baris duplikat persis menggandakan responden yang sama
- **Keputusan:** drop_duplicates(keep="first")
- **Alasan:** Semua pasangan identik di seluruh kolom, sehingga tidak ada informasi hilang
- **Alternatif yang dipertimbangkan:** Dedup berdasar user_id saja (ditolak; dicek: tidak ada user_id sama dengan isi berbeda)
- **Bukti dampak:** 8600 -> 8500 baris; user_id unik; ID berurutan 1..8500 tanpa celah
- **Baris terdampak:** 100
- **Diputuskan oleh:** kode

### 3. L3 Tipe teks -> angka: age

- **Masalah:** Kolom bertipe teks: 30 nilai bersatuan, 0 placeholder
- **Keputusan:** Regex ambil angka, ubah ke float; placeholder tanpa digit menjadi NaN
- **Alasan:** Satuan hanya penjelas; angkanya valid
- **Alternatif yang dipertimbangkan:** to_numeric(errors="coerce") langsung (ditolak: membuang angka pada teks bersatuan)
- **Bukti dampak:** dtype: teks -> float64; 30 nilai bersatuan diselamatkan
- **Baris terdampak:** 30
- **Diputuskan oleh:** kode

### 4. L3 Tipe teks -> angka: bedtime_phone_minutes

- **Masalah:** Kolom bertipe teks: 50 nilai bersatuan, 0 placeholder
- **Keputusan:** Regex ambil angka, ubah ke float; placeholder tanpa digit menjadi NaN
- **Alasan:** Satuan hanya penjelas; angkanya valid
- **Alternatif yang dipertimbangkan:** to_numeric(errors="coerce") langsung (ditolak: membuang angka pada teks bersatuan)
- **Bukti dampak:** dtype: teks -> float64; 50 nilai bersatuan diselamatkan
- **Baris terdampak:** 50
- **Diputuskan oleh:** kode

### 5. L3 Tipe teks -> angka: rem_sleep_pct

- **Masalah:** Kolom bertipe teks: 0 nilai bersatuan, 24 placeholder
- **Keputusan:** Regex ambil angka, ubah ke float; placeholder tanpa digit menjadi NaN
- **Alasan:** Satuan hanya penjelas; angkanya valid
- **Alternatif yang dipertimbangkan:** to_numeric(errors="coerce") langsung (ditolak: membuang angka pada teks bersatuan)
- **Bukti dampak:** dtype: teks -> float64; 0 nilai bersatuan diselamatkan
- **Baris terdampak:** 24
- **Diputuskan oleh:** kode

### 6. L4 Kategori: gender

- **Masalah:** Singkatan F/M dan variasi huruf/spasi (12 bentuk untuk 3 kategori)
- **Keputusan:** Normalisasi oleh kode; F -> Female dan M -> Male lewat peta manual
- **Alasan:** Satu-satunya tafsiran wajar singkatan tersebut; tidak ada kolom yang bertentangan
- **Alternatif yang dipertimbangkan:** Fuzzy matching (ditolak: tidak transparan); membuang baris F/M (ditolak: kehilangan 134 responden)
- **Bukti dampak:** unik 12 -> 3; distribusi akhir hanya Female/Male/Non-Binary
- **Baris terdampak:** 134
- **Diputuskan oleh:** kode + manual

### 7. L4 Kategori: sleep_debt_category

- **Masalah:** Empat salah ketik: Sever, Moderat, Defisit, Recovry
- **Keputusan:** Peta manual eksplisit ke 4 label baku, dilindungi assert
- **Alasan:** Padanan benar tak ambigu karena hanya ada empat kategori baku
- **Alternatif yang dipertimbangkan:** Fuzzy matching (ditolak); mengubah salah ketik menjadi NaN (ditolak: label hilang padahal jelas maksudnya)
- **Bukti dampak:** unik 8 -> 4; assert tidak ada nilai di luar 4 kategori
- **Baris terdampak:** 40
- **Diputuskan oleh:** kode + manual

### 8. L4 Kategori: occupation_type, chronotype, primary_bedtime_app

- **Masalah:** Variasi huruf, spasi, dan pemisah "_"
- **Keputusan:** Normalisasi kunci (strip, lowercase, pemisah seragam) lalu petakan ke label baku
- **Alasan:** Perbedaan murni penulisan, bukan perbedaan makna
- **Alternatif yang dipertimbangkan:** Membiarkan apa adanya (ditolak: groupby memecah 3 kelompok chronotype menjadi 11)
- **Bukti dampak:** unik: occupation_type 15 -> 5, chronotype 8 -> 3, primary_bedtime_app 18 -> 6
- **Baris terdampak:** 0
- **Diputuskan oleh:** kode

### 9. L4 Kategori: blue_light_filter_active

- **Masalah:** Delapan representasi (1/0, Yes/No, True/False) untuk satu variabel biner
- **Keputusan:** Petakan ke 1/0 (Int64 agar NaN tetap boleh)
- **Alasan:** Semua bentuk bermakna sama; 1/0 paling ringkas untuk analisis
- **Alternatif yang dipertimbangkan:** Membiarkan sebagai teks (ditolak: tidak bisa dirata-ratakan)
- **Bukti dampak:** unik 8 -> 2
- **Baris terdampak:** 0
- **Diputuskan oleh:** kode

### 10. L5 Domain: age

- **Masalah:** Nilai mustahil di luar [10 <= nilai <= 100]: [-5.0, 0.0, 999.0]
- **Keputusan:** Ubah menjadi NaN; baris dipertahankan
- **Alasan:** Umur manusia; batas longgar agar hanya nilai mustahil yang tertangkap. Nilai asli tidak dapat dipulihkan.
- **Alternatif yang dipertimbangkan:** Buang baris (ditolak: membuang kolom valid); clipping ke batas (ditolak: mengarang nilai); IQR (ditolak: memotong nilai sah)
- **Bukti dampak:** min/max kolom kembali ke rentang wajar (lihat tabel sebelum-sesudah)
- **Baris terdampak:** 8
- **Diputuskan oleh:** kode

### 11. L5 Domain: screen_brightness_pct

- **Masalah:** Nilai mustahil di luar [0 <= nilai <= 100]: [-20.0, 130.0, 150.0]
- **Keputusan:** Ubah menjadi NaN; baris dipertahankan
- **Alasan:** Persentase kecerahan layar. Nilai asli tidak dapat dipulihkan.
- **Alternatif yang dipertimbangkan:** Buang baris (ditolak: membuang kolom valid); clipping ke batas (ditolak: mengarang nilai); IQR (ditolak: memotong nilai sah)
- **Bukti dampak:** min/max kolom kembali ke rentang wajar (lihat tabel sebelum-sesudah)
- **Baris terdampak:** 6
- **Diputuskan oleh:** kode

### 12. L5 Domain: caffeine_post_5pm_mg

- **Masalah:** Nilai mustahil di luar [0 <= nilai <= 600]: [-50.0, 2500.0]
- **Keputusan:** Ubah menjadi NaN; baris dipertahankan
- **Alasan:** Tidak negatif; batas atas jauh di atas konsumsi harian wajar (~400 mg). Nilai asli tidak dapat dipulihkan.
- **Alternatif yang dipertimbangkan:** Buang baris (ditolak: membuang kolom valid); clipping ke batas (ditolak: mengarang nilai); IQR (ditolak: memotong nilai sah)
- **Bukti dampak:** min/max kolom kembali ke rentang wajar (lihat tabel sebelum-sesudah)
- **Baris terdampak:** 4
- **Diputuskan oleh:** kode

### 13. L5 Domain: sleep_latency_min

- **Masalah:** Nilai mustahil di luar [0 <= nilai <= 300]: [-10.0, 999.0]
- **Keputusan:** Ubah menjadi NaN; baris dipertahankan
- **Alasan:** Tidak negatif; wajar di bawah 5 jam. Nilai asli tidak dapat dipulihkan.
- **Alternatif yang dipertimbangkan:** Buang baris (ditolak: membuang kolom valid); clipping ke batas (ditolak: mengarang nilai); IQR (ditolak: memotong nilai sah)
- **Bukti dampak:** min/max kolom kembali ke rentang wajar (lihat tabel sebelum-sesudah)
- **Baris terdampak:** 4
- **Diputuskan oleh:** kode

### 14. L5 Domain: total_sleep_hours

- **Masalah:** Nilai mustahil di luar [0.01 <= nilai <= 14]: [-3.0, 0.0, 25.0]
- **Keputusan:** Ubah menjadi NaN; baris dipertahankan
- **Alasan:** Harus > 0 jam dan tidak mungkin mendekati 24 jam. Nilai asli tidak dapat dipulihkan.
- **Alternatif yang dipertimbangkan:** Buang baris (ditolak: membuang kolom valid); clipping ke batas (ditolak: mengarang nilai); IQR (ditolak: memotong nilai sah)
- **Bukti dampak:** min/max kolom kembali ke rentang wajar (lihat tabel sebelum-sesudah)
- **Baris terdampak:** 8
- **Diputuskan oleh:** kode

### 15. L5 Domain: kolom lain (bedtime_phone_minutes, physical_activity_min, deep_sleep_pct, rem_sleep_pct, morning_alarm_snoozes, next_day_fatigue_score)

- **Masalah:** Diperiksa terhadap aturan domain
- **Keputusan:** Tidak ada tindakan
- **Alasan:** Seluruh nilai berada dalam rentang aturan
- **Alternatif yang dipertimbangkan:** -
- **Bukti dampak:** 6 kolom lolos tanpa pelanggaran
- **Baris terdampak:** 0
- **Diputuskan oleh:** kode

### 16. L6 Missing: sleep_latency_min, deep_sleep_pct

- **Masalah:** Kosong ~3% (MCAR menurut uji Bonferroni; waspada MNAR)
- **Keputusan:** Biarkan NaN; analisis per pasangan dengan dropna() dan n dilaporkan
- **Alasan:** Imputasi sedikit mengubah variasi dan hubungan pada kolom inti pertanyaan dan menambah n semu; mekanisme MNAR tidak dapat dibuktikan
- **Alternatif yang dipertimbangkan:** Median global; median per kategori; dropna() total
- **Bukti dampak:** Std latency: NaN=17.407, median global=17.148; eta2 kategori: NaN=0.3607, per kategori=0.3692; dropna total membuang 1161 baris (13.7%)
- **Baris terdampak:** 514
- **Diputuskan oleh:** kode + manual

### 17. L6 Missing: rem_sleep_pct, screen_brightness_pct, caffeine_post_5pm_mg, total_sleep_hours, age

- **Masalah:** Kosong kecil (<3%), MCAR; sebagian berasal dari placeholder dan nilai mustahil
- **Keputusan:** Biarkan NaN
- **Alasan:** Mengisi hanya mengarang nilai; kehilangan kecil dan acak
- **Alternatif yang dipertimbangkan:** Median global/per kelompok (ditolak: lihat uji 6.2)
- **Bukti dampak:** Lihat 6.2 dan 6.3
- **Baris terdampak:** 418
- **Diputuskan oleh:** manual

### 18. L6 Missing: occupation_type, chronotype

- **Masalah:** Kosong pada kolom kategorikal (~1,5-2%)
- **Keputusan:** Isi label "Unknown" (baris dipertahankan)
- **Alasan:** Kategorikal: "tidak diketahui" jujur sebagai kelompok sendiri dan baris tetap ikut analisis lain
- **Alternatif yang dipertimbangkan:** Modus (ditolak: mengarang); drop baris (ditolak: membuang kolom lain yang valid)
- **Bukti dampak:** 170 occupation_type dan 127 chronotype berlabel Unknown
- **Baris terdampak:** 297
- **Diputuskan oleh:** manual

### 19. L7 Data pribadi: user_id

- **Masalah:** Pengenal langsung (ID responden) terekspos di data
- **Keputusan:** HMAC-SHA256 dengan garam dari SALT_EDA, dipotong 12 heks; kolom asli dihapus
- **Alasan:** Hash konsisten sehingga tabel tetap dapat ditautkan; garam rahasia mencegah dibalik karena ID berurutan
- **Alternatif yang dipertimbangkan:** Hapus kolom saja (ditolak: kehilangan kunci penaut); hash tanpa garam (ditolak: dapat dibalik dengan 8.500 percobaan)
- **Bukti dampak:** Konsisten & berbeda terbukti; 8500 hash unik tanpa tabrakan
- **Baris terdampak:** 8500
- **Diputuskan oleh:** kode

### 20. L7 Data pribadi: age

- **Masalah:** Kuasi-identifier: umur persis + gender + pekerjaan dapat mempersempit ke individu (k awal = 1)
- **Keputusan:** Generalisasi menjadi kelompok umur 18-29 / 30-44 / 45-65; kolom age dihapus dari keluaran
- **Alasan:** Tiga kelompok cukup memberi daya analitis untuk pertanyaan 3 (kelompok berisiko) sambil menekan risiko
- **Alternatif yang dipertimbangkan:** Lima kelompok (ditolak: k tetap 1); membuang age sepenuhnya (ditolak: kehilangan analisis kelompok umur)
- **Bukti dampak:** k: 1 -> 1 setelah generalisasi
- **Baris terdampak:** 8500
- **Diputuskan oleh:** kode

### 21. L7 Data pribadi: occupation_type, gender, kelompok_umur

- **Masalah:** Setelah generalisasi k masih 1 (10 grup < 5 baris)
- **Keputusan:** Penyamaran lokal bertahap: baris pada grup < 5 disamarkan pada occupation_type, lalu gender, lalu kelompok_umur bila masih perlu, sampai k >= 5
- **Alasan:** Hanya baris pada kombinasi langka yang disamarkan; mayoritas data tidak berubah. Urutan dari kolom yang paling kecil dampaknya pada analisis
- **Alternatif yang dipertimbangkan:** Menghapus occupation_type seluruhnya (ditolak: kehilangan variabel analisis); memperlebar umur jadi 2 kelompok (ditolak: lebih kasar dan tetap tidak cukup)
- **Bukti dampak:** k akhir = 5; 17 baris (0.20%) disamarkan; chronotype tidak termasuk QI (bila dimasukkan k = 1)
- **Baris terdampak:** 17
- **Diputuskan oleh:** kode + manual

### 22. L7 Data pribadi: seluruh dataset

- **Masalah:** Dataset akan diunggah ke repositori
- **Keputusan:** Verifikasi: tidak ada string "USR-" di keluaran dan kolom user_id/age asli sudah tidak ada
- **Alasan:** Anonimisasi harus diverifikasi, bukan sekadar diklaim
- **Alternatif yang dipertimbangkan:** -
- **Bukti dampak:** Lihat Langkah 9 (assert pada berkas yang tersimpan)
- **Baris terdampak:** 8500
- **Diputuskan oleh:** kode
