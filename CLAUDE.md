# CLAUDE.md

Proyek kelompok mata kuliah Rekayasa Data (Post-UTS): **Data Pipeline (ETL) + Data Warehouse** untuk analisis mobilitas manusia di DIY, memakai Pandas + DuckDB. Rencana lengkap, desain skema, daftar QC, dan analisis yang direncanakan ada di `PLAN.md`. Baca itu dulu sebelum menambah section baru.

## Struktur

- `main.ipynb`: notebook utama, satu-satunya kode pipeline. Section: 0 Setup, 1 Extract → staging raw, 2 Profiling, 3 Transform → staging clean, 4 Build DW (belum), lalu QC, Analytics, Visualisasi (belum).
- `data/`: CSV sumber dari Zenodo (JogjaPulse, DOI 10.5281/zenodo.23157460). Jangan diubah; perlakukan sebagai read-only.
- `db/mobility.duckdb`: output pipeline, di-`.gitignore`, selalu bisa dibuat ulang dengan menjalankan notebook dari awal.
- `PLAN.md`: rencana & checklist progres. Centang item ketika selesai.

## Menjalankan

- Lokal: `pip install -r requirements.txt`, lalu buka Jupyter dari dalam repo. Root repo dicari otomatis (folder yang punya `data/` dan `.git`/`PLAN.md`), jadi jangan hardcode path absolut.
- Colab: sel setup otomatis clone repo GitHub dan install requirements.
- Notebook harus bisa dijalankan **dari atas ke bawah tanpa error** (Restart & Run All). Tabel DuckDB dibuat dengan `CREATE OR REPLACE` supaya idempoten.

## Arsitektur data

Schema DuckDB: `staging` (`presence_raw`, `od_raw`, `presence_clean`, `od_clean`) dan `dw` (dimensi & fakta, lihat `PLAN.md` §3: galaxy schema dengan `dim_kapanewon`, `dim_date`, `dim_hour`, `dim_month`, `fact_hourly_presence`, `fact_monthly_od`). Tabel `dw` dibangun dengan SQL dari `staging.*_clean`, bukan langsung dari Pandas.

Dua sumber data:
- `presence_kapanewon_hourly.csv`: grain kapanewon × tanggal × jam (230.418 baris).
- `od_home_visits_kapanewon_monthly.csv`: grain bulan × origin × dest (9.894 baris).

## Aturan data (wajib, sering salah)

- **`n_maids` tidak aditif** (distinct device count). Jangan `SUM` lintas jam, hari, atau kapanewon di presence; pakai `AVG`/`MEDIAN` per slot. Di OD, `SUM` lintas *origin* untuk satu tujuan boleh, lintas *tujuan* tidak boleh.
- **`n_pings` aditif**, jadi boleh di-`SUM`.
- **Baris yang hilang = < 5 device (disuppress), bukan nol.** Jangan isi 0.
- **Ukuran sampel turun drastis** (±87 rb device/hari Des → ±9 rb Apr). Perbandingan antar waktu pakai `maids_share` / `visitor_share_dest`, bukan angka mentah.
- **OD Mar–Apr 2022 `reduced_quality = True`**: disimpan di DW, tapi analisis kuantitatif OD hanya Des 2021 – Feb 2022.
- **2022-03-08 anomali** (`is_anomaly`): dikecualikan dari analisis tren.
- **Join pakai kode wilayah, bukan nama.** Nama `JETIS` ada di Bantul (`34.02.09`) dan Kota Yogyakarta (`34.71.02`).
- Kolom kode (`kab_code`, `kapanewon_code`, `origin_code`, dst.) dibaca sebagai string (`dtype`); tanpa itu `34.01` jadi float.
- Device ≠ orang. Sebut sebagai "device" di narasi/visualisasi.

## Konvensi

- Bahasa notebook & dokumentasi: Indonesia; nama kolom/tabel: snake_case Inggris.
- Setiap section notebook diawali sel markdown yang menjelaskan tujuan tahap itu.
- Gambar untuk README/video disimpan ke `figures/`.
- Pemakaian DuckDB (versi 2.x): hasil `con.execute(...)` harus di-*fetch* habis (`.fetchall()`, atau pakai `con.sql(...).df()`) sebelum query berikutnya, kalau tidak muncul `ResourceInUseException`.
- Sitasi dataset wajib ada di README: Widyawan (2026), JogjaPulse, Zenodo, https://doi.org/10.5281/zenodo.23157460.
