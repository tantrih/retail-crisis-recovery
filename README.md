# Retail Crisis & Recovery — Hidden Growth Product Detection

> **DQLab x UjiKompetensi Hackathon** | Python · Pandas · Matplotlib · Mlxtend  
> Kompetisi: `HACK-2026-PYTHON-01`

---

## 📌 Ringkasan Proyek

Sebuah minimart (DQFresh Mart) mengalami penurunan total penjualan selama beberapa bulan. Manajemen awalnya menyimpulkan kondisi ini disebabkan oleh pelemahan ekonomi, lalu merespons dengan strategi defensif: mempertahankan produk bestseller lama dan mengurangi eksperimen produk baru.

Proyek ini mereplikasi analisis investigatif yang dilakukan Sophia (manajer toko) untuk **membantah asumsi tersebut secara data-driven** — menemukan produk yang tumbuh konsisten namun tidak terlihat di agregasi tradisional, dan mengidentifikasi pola pembelian bersama untuk strategi bundling.

---
1. Apakah penurunan penjualan sepenuhnya disebabkan faktor eksternal, atau ada sinyal pertumbuhan tersembunyi yang terlewat? |
2.Produk mana yang menunjukkan tren kenaikan konsisten dalam 30 hari terakhir meski kontribusi revenue-nya kecil? |
3.Apakah produk-produk yang tumbuh tersebut memiliki pola pembelian bersama yang bisa dimanfaatkan untuk strategi bundling? |
4.Bagaimana kecepatan pertumbuhan produk tersembunyi dibandingkan dengan produk terlaris secara keseluruhan? |

---

## 🔍 Masalah yang Diselesaikan

### Bias Analitik dalam Agregasi Tradisional
Dashboard toko hanya menampilkan **Top N produk berdasarkan total revenue** — metode ini secara sistematis menyembunyikan produk baru yang tumbuh cepat karena nilai absolut revenue-nya masih kecil.

**Contoh nyata dari dataset ini:**
- Produk `Kaos Kaki (3 Pasang)` → revenue **Rp 310 juta** → masuk Top 3
- Produk `Minyak Goreng Refill 1L` → revenue **Rp 44 juta** → tidak terlihat di Top 10
- Namun MA Minyak Goreng tumbuh **+712%** dalam sesi tren terpanjangnya (14 hari berturut-turut)

Ini adalah kasus klasik **survivorship bias dalam analitik retail**: sistem hanya "menyelamatkan" produk yang sudah besar, dan menghukum produk yang baru mulai naik.

---

## Dataset

| Atribut | Detail |
|---------|--------|
| **File** | `data_penjualan.xlsx` |
| **Periode** | 1 Februari 2025 – 4 Maret 2025 (32 hari) |
| **Total baris** | 42.446 baris transaksi |
| **Total invoice** | 9.403 struk unik |
| **Total produk** | 58 SKU |
| **Total revenue** | Rp 3,44 miliar |

### Struktur Kolom

| Kolom | Tipe | Deskripsi |
|-------|------|-----------|
| `nomor_struk` | string | ID invoice transaksi |
| `tgl_transaksi` | datetime | Tanggal transaksi |
| `kode_produk` | string | Kode SKU produk |
| `nama_produk` | string | Nama produk |
| `jumlah_terjual` | int | Quantity terjual |
| `harga` | int | Harga satuan (Rp) |
| `total_nilai` | int | Total nilai = harga × qty |

---

##  Data Quality & Cleaning

Dataset ini **tidak memerlukan cleaning** (dinyatakan eksplisit dalam spesifikasi soal), namun validasi tetap dilakukan:

```
Missing values : 0 di semua kolom ✓
Duplikasi baris: tidak ada ✓
Konsistensi    : total_nilai = harga × jumlah_terjual ✓
Tipe data      : tgl_transaksi sudah datetime64, numerik sudah int64 ✓
```

**Satu keputusan desain penting terkait data:**

Saat melakukan visualisasi, hari-hari di mana produk tertentu tidak memiliki transaksi (missing days) **diisi dengan 0**, bukan dengan interpolasi linear. Alasannya: hari tanpa penjualan berarti penjualan memang nol — bukan nilai di antara dua titik. Menggunakan interpolasi linear akan menciptakan data fiktif.

---

## Proses Analisis

### Tahap 1 — Rising Star Detection

```
Data Harian per Produk
        ↓
Moving Average 3 Hari (min_periods=3)
        ↓
Identifikasi "Sesi Tren Naik": MA[hari ini] > MA[hari sebelumnya]
        ↓
Hitung Consecutive Rising Days per sesi
        ↓
Filter: max streak ≥ 12 hari berturut-turut
        ↓
Hitung Growth % = (MA akhir tren / MA awal tren − 1) × 100
```

**Kenapa Moving Average 3 hari?**  
Penjualan harian retail sangat volatile — hari libur, akhir bulan, dan efek promosi harian menciptakan spike yang misleading. Window 3 hari meredam noise tanpa kehilangan sinyal tren jangka pendek yang bermakna.

**Kenapa threshold 12 hari?**  
12 hari dari 30 hari data = 40% dari periode observasi. Ini memastikan tren yang terdeteksi bukan kebetulan jangka pendek, melainkan pola yang cukup konsisten untuk dijadikan keputusan bisnis.

### Tahap 2 — Potential Packaging (Market Basket Analysis)

```
Basket Matrix: baris = invoice, kolom = produk, nilai = True/False
        ↓
Apriori Algorithm (mlxtend, min_support = 0.01)
        ↓
Association Rules (metric = lift, min_threshold = 1.0)
        ↓
Filter: salah satu sisi harus Rising Star + lift ≥ 2
        ↓
Sort: Lift → Support → Confidence (descending)
```

**Kenapa lift ≥ 2 sebagai threshold?**  
Lift = 1 berarti dua produk dibeli bersama secara independen (tidak ada asosiasi). Lift ≥ 2 berarti produk B **dua kali lebih mungkin dibeli** ketika produk A dibeli — sinyal yang cukup kuat untuk dijadikan dasar bundling.

### Tahap 3 — Visualisasi

Dua jenis grafik dihasilkan:
1. **Index Chart (Base 100)**: semua produk dinormalisasi ke titik awal 100 — perbandingan *kecepatan* tumbuh, bukan ukuran absolut
2. **Actual Value Chart**: nilai penjualan asli (MA) untuk konteks skala

---

## Insight yang Ditemukan

### Rising Star Products (4 produk)

| Kode | Produk | Max Streak | Growth % | Total Revenue |
|------|--------|-----------|---------|--------------|
| MGR1L | Minyak Goreng Refill 1L | 14 hari | **+712%** | Rp 44,9 juta |
| BRS5KG | Beras Premium 5kg | 18 hari | **+700%** | Rp 44,1 juta |
| SCC15L | Sabun Cuci Cair 1.5L | 22 hari | **+552%** | Rp 95,9 juta |
| WJANEM | Wajan Enamel Anti Lengket | 22 hari | **+537%** | Rp 44,2 juta |

**Key findings:**
- Semua 4 rising star **tidak ada satupun** yang masuk Top 10 berdasarkan total revenue
- `Sabun Cuci Cair 1.5L` memiliki streak terpanjang: **22 hari berturut-turut naik** — hampir sepanjang seluruh periode data
- Kontribusi gabungan 4 produk ini terhadap total revenue hanya **6,7%** — alasan sistematis kenapa dashboard tradisional tidak pernah menyorotnya
- Growth rate 500–700% dalam ~3 minggu adalah sinyal demand yang kuat, bukan fluktuasi acak

### Implikasi Bisnis

> Strategi defensif manajemen (mempertahankan bestseller lama) justru melewatkan peluang nyata. Keempat produk ini tumbuh **meski toko secara total sedang turun** — artinya ada pergeseran preferensi konsumen yang belum ditangkap.

**Rekomendasi:**
1. **Tambah stok** keempat produk tersebut — terutama yang sering stockout (kasir melaporkan sering habis)
2. **Buat paket bundling** berdasarkan hasil market basket analysis
3. **Pasang di lokasi lebih visible** di toko (end-cap atau area checkout)

---

## Output yang Dihasilkan

```
python solusi-retail.py
```

| File | Deskripsi |
|------|-----------|
| `retail_insight.xlsx` | Sheet "Rising Star" + "Potential Packaging" |
| `rising_star_index.png` | Line chart pertumbuhan relatif (Base 100) |
| `rising_star_actual.png` | Line chart nilai penjualan asli |

---

## Tech Stack & Versi

| Library | Versi | Fungsi |
|---------|-------|--------|
| Python | 3.10–3.14 | — |
| pandas | 2.3.1 | Data manipulation, rolling MA, groupby |
| matplotlib | 3.10.7 | Visualisasi line chart |
| mlxtend | 0.23.4 | Apriori algorithm, association rules |
| openpyxl | 3.1.5 | Export Excel dengan formatting |

---

## Keputusan Teknis & Trade-offs

| Keputusan | Alternatif | Alasan Dipilih |
|-----------|-----------|----------------|
| `min_periods=3` untuk MA | `min_periods=1` | Sesuai spesifikasi soal; MA valid hanya setelah ada 3 data point |
| `fillna(0)` untuk missing day | `interpolate(linear)` | Hari tanpa penjualan = 0, bukan nilai interpolasi — ini data retail, bukan sensor fisik |
| Loop Python di `calc_max_streak` | `iterrows()` | Menghindari overhead pembuatan Series per baris; ~50–100x lebih cepat pada dataset besar |
| mlxtend `apriori` + `association_rules` | Implementasi manual | Wajib oleh spesifikasi soal; juga lebih maintainable dan teruji |

---
