# PLAN — Data Pipeline & Data Warehouse Mobilitas Manusia DIY

Proyek Rekayasa Data Post-UTS. Sumber: JogjaPulse (Widyawan, 2026), Zenodo DOI `10.5281/zenodo.23157460`.

## 0. Tujuan & Deliverables

| Deliverable | Isi |
|---|---|
| Kode (GitHub) | Notebook pipeline (Extract → Staging → Transform → DW → QC → Analytics → Viz), `README.md` dengan diagram skema |
| Data Warehouse | `db/mobility.duckdb` berisi schema `staging` dan `dw` |
| Analitik & visualisasi | Operasi data cube (roll-up, drill-down, slice/dice, pivot) + grafik |
| Video 5 menit | Latar belakang → arsitektur pipeline → skema DW → QC → insight |

**Pertanyaan analitik utama (usulan):**
> *"Di mana pusat aktivitas siang hari di DIY, dari mana pengunjungnya berasal, dan bagaimana pola hariannya?"*

Pertanyaan turunan:
1. Kapanewon mana yang ramai di siang hari tapi sepi di malam hari (pusat kerja/kuliah/wisata), dan sebaliknya (kawasan permukiman)?
2. Bagaimana siklus harian (24 jam) dan perbedaan hari kerja vs akhir pekan?
3. Kapanewon tujuan mana yang paling banyak menarik pengunjung dari luar wilayahnya, dan dari mana asal mereka?
4. Seberapa besar interaksi lintas kabupaten/kota (misal Sleman ↔ Kota Yogyakarta)?

---

## 1. Koreksi Alur (dari alur sebelumnya)

Alur notebook saat ini: *Extract (Pandas) → Profiling → Transform (Pandas) → (belum ada Load)*. Ada beberapa hal yang perlu dibenarkan:

| # | Masalah | Perbaikan |
|---|---|---|
| 1 | Data mentah tidak pernah disimpan ke DB; langsung transform di Pandas. Tidak sesuai pola slide (`staging` → `dw`). | **Load raw ke `staging` dulu** (apa adanya), lalu transform, lalu bangun `dw` via SQL. |
| 2 | Kode wilayah (`kab_code` = `34.01`) dibaca Pandas sebagai *float*. | Baca CSV dengan `dtype=str` untuk semua kolom kode. |
| 3 | `SUM(n_maids)` dipakai di profiling (cell 1.2 dan 1.4). `n_maids` adalah *distinct count*, **tidak aditif** lintas jam/hari/wilayah. | Ganti dengan `AVG`/`MEDIAN` per jam, atau hanya pakai `SUM(n_pings)`. Beri catatan di notebook. |
| 4 | Tidak ada normalisasi. Jumlah device turun dari ±87 rb/hari (Des) ke ±9 rb/hari (Apr) karena **perubahan sampel**, bukan perubahan mobilitas. | Tambahkan metrik ternormalisasi: `share = n_maids / total n_maids semua kapanewon pada jam yang sama`. Semua perbandingan antar waktu memakai share. |
| 5 | Data OD Mar–Apr `reduced_quality = True` belum diputuskan. | Tetap **simpan semua** di DW (flag `reduced_quality`), tapi analisis kuantitatif OD **hanya Des 2021 – Feb 2022**. |
| 6 | Anomali 8 Maret 2022 (drop satu hari) belum ditangani. | Tambah flag `is_anomaly` di `dim_date`; dikecualikan dari analisis tren. |
| 7 | Baris yang hilang = "< 5 device" (suppressed), bukan nol. | Jangan isi baris hilang dengan 0. Hitung `coverage_pct` per kapanewon sebagai metrik kualitas. |
| 8 | Nama kapanewon `JETIS` ada di Bantul **dan** Kota Yogyakarta. | Selalu join pakai **kode** (`kapanewon_code`), bukan nama. |
| 9 | Path masih Google Colab (`/content/drive/...`). | Buat konfigurasi path yang bisa jalan di Colab maupun lokal (deteksi env, default `./data` & `./db`). |
| 10 | Grain waktu dua fakta berbeda (jam vs bulan). | `dim_date` (harian) untuk presence + `dim_month` untuk OD. Jangan dipaksa satu `dim_time`. |

### Alur yang benar

```
[Zenodo CSV]
    │  1. EXTRACT     pd.read_csv(dtype=str untuk kode)
    ▼
[staging.presence_raw, staging.od_raw]       ← raw, apa adanya
    │  2. PROFILE     audit null, duplikat, coverage, konsistensi kode
    │  3. TRANSFORM   standarisasi, parsing tanggal, fitur waktu, flag kualitas, normalisasi
    ▼
[staging.presence_clean, staging.od_clean]
    │  4. LOAD DW     CREATE TABLE dw.dim_* / dw.fact_* AS SELECT ... (SQL)
    ▼
[dw: dim_kapanewon, dim_date, dim_hour, dim_month, fact_hourly_presence, fact_monthly_od]
    │  5. QC          rowcount, null key, orphan FK, duplikat grain, range nilai
    │  6. ANALYTICS   query cube (roll-up, drill-down, slice/dice, pivot)
    ▼
[DataFrame → Matplotlib/Seaborn]  7. VISUALIZE
```

---

## 2. Data Sumber

| File | Grain | Baris | Measures |
|---|---|---|---|
| `presence_kapanewon_hourly.csv` | kapanewon × tanggal × jam | 230.418 | `n_maids` (non-aditif), `n_pings` (aditif) |
| `od_home_visits_kapanewon_monthly.csv` | bulan × origin × dest | 9.894 | `n_maids` (visitor distinct, non-aditif lintas dest) |

- 78 kapanewon, 5 kab/kota (`34.01` Kulon Progo, `34.02` Bantul, `34.03` Gunungkidul, `34.04` Sleman, `34.71` Kota Yogyakarta).
- Periode 2021-12-01 s.d. 2022-04-30 (151 hari), waktu WIB.
- Baris OD per bulan: Des 2.595, Jan 2.624, Feb 2.653, Mar 1.202, Apr 820 (turun drastis karena suppression).

**Aturan agregasi (wajib dipatuhi):**
- `n_pings` → boleh `SUM` ke mana saja.
- `n_maids` presence → `AVG`/`MEDIAN`/`MAX` per jam; **tidak boleh** `SUM` lintas jam, hari, atau kapanewon.
- `n_maids` OD → untuk satu tujuan, `SUM` lintas origin **boleh** (tiap device punya satu home per bulan). `SUM` lintas tujuan **tidak boleh** (satu device bisa mengunjungi banyak tujuan).

---

## 3. Desain Data Warehouse (Fact Constellation / Galaxy Schema)

```
                 dim_date ──┐           ┌── dim_month
                 dim_hour ──┤           │
                            ▼           ▼
dim_kapanewon ──► fact_hourly_presence   fact_monthly_od ◄── dim_kapanewon (origin)
                                              ▲
                                              └───────────── dim_kapanewon (dest)
```

### Dimensi

**`dw.dim_kapanewon`**: `kapanewon_code` (PK), `kapanewon_name`, `kab_code`, `kab_kota`, `is_city` (Kota Yogyakarta = true)

**`dw.dim_date`**: `date_key` (PK, int `YYYYMMDD`), `date`, `day`, `day_of_week`, `day_name`, `is_weekend`, `week_of_year`, `month_key` (FK ke `dim_month`), `is_holiday` (25 Des, 1 Jan, 1 Feb Imlek, 28 Feb Isra Mi'raj, 3 Mar Nyepi — cek ulang), `is_anomaly` (2022-03-08)

**`dw.dim_hour`**: `hour` (PK 0–23), `time_slot` (Dini Hari 00–05, Pagi 06–09, Siang 10–14, Sore 15–18, Malam 19–23), `is_daytime` (08–17, sesuai definisi OD), `is_night` (00–05, sesuai validasi dataset)

**`dw.dim_month`**: `month_key` (PK int `YYYYMM`), `month` (`YYYY-MM`), `month_name`, `year`, `reduced_quality` (Mar–Apr = true)

### Fakta

**`dw.fact_hourly_presence`**: grain 1 baris = 1 kapanewon × 1 tanggal × 1 jam
- FK: `kapanewon_code`, `date_key`, `hour`
- Measures: `n_maids`, `n_pings`, `pings_per_maid`, `maids_share` (= `n_maids / SUM(n_maids) OVER (PARTITION BY date_key, hour)`)

**`dw.fact_monthly_od`**: grain 1 baris = 1 bulan × 1 origin × 1 dest
- FK: `month_key`, `origin_code`, `dest_code`
- Atribut: `flow_scope` (Intra-Kapanewon / Antar-Kapanewon dalam Kab / Lintas Kab), `reduced_quality`
- Measures: `n_maids`, `visitor_share_dest` (= `n_maids / SUM(n_maids) per (month, dest)`)

> Catatan untuk laporan: `maids_share` adalah share *device-presence*, bukan populasi. Gunakan sebagai indikator relatif.

---

## 4. Quality Check (QC) — wajib tampil di notebook & video

| Cek | Ekspektasi |
|---|---|
| Rowcount `staging` vs CSV vs `dw.fact_*` | presence 230.418, OD 9.894 (sama di tiap tahap) |
| Null pada semua FK | 0 |
| Orphan FK (fakta tanpa pasangan di dimensi) | 0 |
| Duplikat grain (`kapanewon_code, date_key, hour`) / (`month_key, origin, dest`) | 0 |
| Range: `hour` 0–23, `n_maids >= 5`, `n_pings >= n_maids` | semua lolos |
| Jumlah kapanewon di dimensi | 78 |
| Coverage per kapanewon (% slot jam terpublikasi) | dilaporkan; tandai yang < 50% |
| Kode vs nama konsisten (1 kode = 1 nama) | lolos; `JETIS` muncul 2× dengan kode berbeda |

Hasil QC dikumpulkan dalam satu tabel ringkasan (PASS/FAIL).

---

## 5. Analitik (Operasi Data Cube)

Semua query SQL di DuckDB → `.df()` → visualisasi.

| # | Operasi | Analisis | Visualisasi |
|---|---|---|---|
| A1 | **Drill-down** (hari → jam) | Profil 24 jam rata-rata `maids_share` per kab/kota; weekday vs weekend | Line chart (5 garis kab/kota, panel weekday/weekend) |
| A2 | **Slice** (jam siang vs malam) | Rasio siang/malam per kapanewon = `AVG share 10–14` / `AVG share 00–05` → pusat aktivitas vs permukiman | Bar chart horizontal top/bottom 10 |
| A3 | **Dice** (Sleman + Kota, hari kerja) | Heatmap kapanewon × jam untuk area urban | Heatmap |
| A4 | **Roll-up** (jam → hari → minggu) | Tren harian total `n_pings` & jumlah device (untuk *menunjukkan* bias sampel & alasan normalisasi) | Line chart + anotasi 1 Jan, 16 Feb, 8 Mar |
| A5 | **Pivot** | Matriks OD kab/kota asal × kab/kota tujuan (Des–Feb) | Heatmap matriks 5×5 |
| A6 | Slice (tujuan tertentu) | Top 10 kapanewon tujuan berdasarkan pengunjung dari luar wilayah (inter-zonal), plus asal pengunjung untuk 2–3 tujuan (mis. Depok, Gondokusuman, Gedongtengen) | Bar chart; opsional Sankey |
| A7 | Roll-up (event) | Perbandingan share malam tahun baru / libur vs hari biasa di kapanewon wisata (Kraton, Gedongtengen, Gunungkidul pesisir) | Bar chart berpasangan |

Opsional (nilai tambah): clustering profil 24 jam kapanewon (KMeans pada vektor share 24 jam) → label "residential / commercial / mixed"; peta choropleth jika ada shapefile kapanewon (Geoportal DIY).

---

## 6. Struktur Repo (target)

```
rekdat-postuts/
├── README.md              # deskripsi, cara menjalankan, diagram skema, insight utama, sitasi dataset
├── PLAN.md
├── main.ipynb  # notebook utama (urutan section = alur di §1)
├── requirements.txt       # pandas, duckdb, matplotlib, seaborn
├── data/                  # CSV dari Zenodo (±13 MB, aman di-push)
├── db/mobility.duckdb     # dihasilkan pipeline (bisa di-.gitignore & di-generate ulang)
└── figures/               # PNG hasil visualisasi untuk README & video
```

Section notebook:
1. Setup & konfigurasi path
2. Extract → Load `staging` raw
3. Data Understanding & Profiling *(sudah ada; perbaiki poin #2, #3)*
4. Transform *(sudah ada; tambah normalisasi, flag anomali/holiday)*
5. Build DW (DDL + `INSERT ... SELECT`)
6. Quality Check
7. Analytics & Visualization (A1–A7)
8. Kesimpulan & keterbatasan data

---

## 7. Pembagian Tugas (8 anggota)

| Peran | Orang | Tugas |
|---|---|---|
| Data Engineer (2) | | Extract, staging, transform, perbaikan alur #1–#10 |
| DW Modeler (1) | | DDL dimensi & fakta, diagram skema untuk README/video |
| QC (1) | | Semua query QC + tabel ringkasan |
| Analyst (2) | | Query A1–A7 + visualisasi |
| Dokumentasi (1) | | README, requirements, rapikan repo |
| Video (1) | | Naskah, slide, rekaman, editing (semua anggota kebagian bicara) |

---

## 8. Urutan Kerja / Checklist

- [x] Perbaiki konfigurasi path (Colab + lokal) & baca CSV dengan `dtype=str` untuk kode
- [x] Load raw ke `staging.presence_raw`, `staging.od_raw`
- [x] Revisi profiling: hapus `SUM(n_maids)`, tambah coverage per kapanewon
- [x] Transform: fitur waktu, flag holiday/anomali, `maids_share`, `flow_scope`, `visitor_share_dest`
- [x] Simpan hasil transform ke `staging.*_clean`
- [x] Buat `dw.dim_kapanewon`, `dim_date`, `dim_hour`, `dim_month`
- [x] Buat `dw.fact_hourly_presence`, `dw.fact_monthly_od`
- [x] Jalankan QC → semua PASS
- [x] Analitik A1–A7 + simpan PNG ke `figures/`
- [x] Tulis kesimpulan + keterbatasan (devices ≠ people, suppression, bias sampel, OD Mar–Apr)
- [x] README + diagram skema + `requirements.txt`
- [ ] `git pull --allow-unrelated-histories` lalu push ke GitHub
- [ ] Rekam video 5 menit

### Rundown video (±5 menit)
| Menit | Isi |
|---|---|
| 0:00–0:40 | Latar belakang & pertanyaan analitik |
| 0:40–1:30 | Dataset (2 sumber) & keterbatasannya |
| 1:30–2:30 | Arsitektur pipeline ETL + skema DW (galaxy) |
| 2:30–3:00 | Quality check |
| 3:00–4:30 | Insight dari visualisasi (A1, A2, A5, A6) |
| 4:30–5:00 | Kesimpulan & keterbatasan |

---

## 9. Sitasi

Widyawan (2026). *JogjaPulse: An Aggregated Spatio-Temporal Mobility Dataset for Yogyakarta, Indonesia.* Zenodo. https://doi.org/10.5281/zenodo.23157460
