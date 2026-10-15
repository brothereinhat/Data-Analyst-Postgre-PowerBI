# Analisis Data Toko Fashion Online: PostgreSQL & Power BI

Proyek analisis data end-to-end pada dataset toko fashion online Indonesia. Alurnya mencakup pembersihan dan validasi data di **PostgreSQL**, analisis bisnis dengan **SQL**, lalu visualisasi interaktif di **Power BI**.

## Daftar Isi

- [Tujuan Proyek](#tujuan-proyek)
- [Tools](#tools)
- [Dataset](#dataset)
- [Alur Kerja](#alur-kerja)
- [1. Data Cleaning](#1-data-cleaning)
- [2. Data Validation](#2-data-validation)
- [3. Analisis Bisnis](#3-analisis-bisnis)
- [4. Dashboard Power BI](#4-dashboard-power-bi)
- [Insight Utama](#insight-utama)
- [Keterbatasan](#keterbatasan)
- [Cara Menjalankan Ulang](#cara-menjalankan-ulang)

---

## Tujuan Proyek

1. Memastikan kualitas data (NULL, duplikat, format, konsistensi antar tabel) sebelum dianalisis.
2. Menjawab pertanyaan bisnis: kurir favorit, efektivitas promo, segmentasi pelanggan, produk laris yang kehabisan stok, dan produk yang menumpuk.
3. Menyajikan hasilnya dalam dashboard interaktif.

## Tools

| Tool | Kegunaan |
|---|---|
| PostgreSQL 13 | Database dan analisis SQL |
| VS Code (extension PostgreSQL) | Menulis dan menjalankan query |
| Power BI Desktop | Visualisasi dan dashboard |

## Dataset

Lima tabel dalam format CSV, dengan data transaksi sekitar tahun 2022 sampai 2025.

| Tabel | Jumlah Baris | Isi |
|---|---|---|
| `customers` | 800 | Data pelanggan (kota, provinsi, gender, kelompok usia, tanggal registrasi) |
| `products` | 300 | Data produk (kategori, brand, harga, stok, rating) |
| `orders` | 3.000 | Data pesanan (tanggal, total, status, promo, kurir, metode bayar) |
| `order_items` | 4.986 | Detail item per pesanan |
| `reviews` | 1.500 | Ulasan dan rating produk |

Relasi antar tabel (foreign key):

```
customers ──< orders ──< order_items >── products
    │            │                          │
    └────────────┴────────< reviews >───────┘
```

## Alur Kerja

```
CSV  →  PostgreSQL (CREATE TABLE + COPY)  →  Cleaning  →  Validasi  →  Analisis SQL  →  Power BI
```

---

## 1. Data Cleaning

| Masalah | Temuan | Penanganan |
|---|---|---|
| NULL di `customers.phone` | 40 baris | Diisi placeholder `'Unknown'` (nomor telepon tidak bisa ditebak) |
| NULL di `orders.discount_amount_idr` dan `orders.promo_code` | Masing-masing 1.786 baris | Diskon diisi `0`; promo diisi `'No Promo'` (NULL berarti tidak memakai promo) |
| NULL di `reviews.review_text` | 120 baris | **Dibiarkan NULL**, karena berarti customer memberi rating tanpa komentar |
| Email duplikat di `customers` | 3 email dipakai 2 customer berbeda | Didokumentasikan sebagai catatan kualitas data (nama, telepon, dan kota berbeda) |
| Telepon duplikat | Tidak ada | - |
| Spasi berlebih di kolom teks | Dicek dengan `TRIM` dan `REGEXP_REPLACE` | Tabel `orders` sudah bersih |

Contoh query pengecekan duplikat dan NULL:

```sql
-- Email duplikat
SELECT email, COUNT(*)
FROM customers
GROUP BY email
HAVING COUNT(*) > 1;

-- Jumlah NULL di semua kolom sekaligus
SELECT
    COUNT(*) AS total_rows,
    COUNT(*) FILTER (WHERE phone IS NULL) AS null_phone,
    COUNT(*) FILTER (WHERE email IS NULL) AS null_email
FROM customers;
```

## 2. Data Validation

| Jenis Validasi | Cara | Hasil |
|---|---|---|
| Referensial (antar tabel) | `NOT EXISTS` pada foreign key | Foreign key aktif dan data konsisten |
| Rentang nilai | Cek harga, quantity, dan stok negatif | Stok `0` dianggap valid (barang habis), bukan error |
| Urutan waktu | Order vs tanggal registrasi customer | **795 dari 3.000 order (sekitar 26%)** tercatat sebelum tanggal registrasi customer |
| Konsistensi perhitungan | `total_amount_idr` vs `order_items` | Dari sampel baris, `total_amount_idr = SUM(subtotal_idr) - discount_amount_idr`; ongkir (`shipping_cost_idr`) dicatat terpisah |

Catatan: 795 order yang tanggalnya mendahului registrasi kemungkinan berasal dari proses pembuatan data yang tidak memastikan urutan tanggal. Temuan ini didokumentasikan, tidak diperbaiki.

```sql
-- Order yang tanggalnya lebih awal dari registrasi customer
SELECT o.order_id, o.order_date, c.customer_id, c.registration_date
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
WHERE o.order_date < c.registration_date;
```

## 3. Analisis Bisnis

### 3.1 Efektivitas Promo

Hanya order berstatus `delivered` yang dihitung.

| Kelompok | Jumlah Order | Rata-rata Transaksi (Rp) | Total Revenue (Rp) | Total Diskon (Rp) |
|---|---|---|---|---|
| Tanpa Promo | 1.059 | 510.122 | 540.219.000 | 0 |
| Pakai Promo | 717 | 470.757 | 337.533.049 | 22.184.951 |

Rata-rata transaksi dengan promo sekitar 7,7% lebih rendah daripada tanpa promo. Ini temuan deskriptif, belum membuktikan sebab-akibat.

### 3.2 Promo: Customer Baru vs Lama

Pemakaian promo hampir sama di kedua kelompok.

| Kategori | Jumlah Order | Pakai Promo | Persentase |
|---|---|---|---|
| Order pertama | 787 | 313 | 39,8% |
| Order berikutnya | 2.213 | 901 | 40,7% |

### 3.3 Segmentasi Pelanggan (RFM)

- **Recency**: selisih hari antara order terakhir dalam dataset dan order terakhir customer
- **Frequency**: jumlah order (status selain `cancelled` dan `returned`)
- **Monetary**: total nilai belanja
- Skor 1 sampai 5 dengan `NTILE(5)`, lalu dikelompokkan dengan `CASE WHEN`

| Segmen | Jumlah Customer | Persentase* |
|---|---|---|
| At Risk | 206 | 26,8% |
| Loyal Customers | 198 | 25,8% |
| Champions | 133 | 17,3% |
| New Customers | 129 | 16,8% |
| Lost | 102 | 13,3% |
| No Valid Order | 32 | - |

\* Persentase dihitung dari 768 customer yang punya order valid. Sebanyak 32 customer lain tidak punya order selain `cancelled`/`returned`, sehingga dimasukkan ke segmen `No Valid Order`. Total keseluruhan 800 customer.

<details>
<summary>Query RFM lengkap</summary>

```sql
WITH rfm_base AS (
    SELECT c.customer_id,
           (SELECT MAX(order_date) FROM orders
            WHERE order_status NOT IN ('returned', 'cancelled')) - MAX(o.order_date) AS recency_hari,
           COUNT(DISTINCT o.order_id) AS frequency,
           SUM(o.total_amount_idr) AS monetary
    FROM customers c
    LEFT JOIN orders o ON c.customer_id = o.customer_id
        AND o.order_status NOT IN ('returned', 'cancelled')
    GROUP BY c.customer_id
),
rfm_score AS (
    SELECT customer_id, recency_hari, frequency, monetary,
           CASE WHEN frequency > 0 THEN NTILE(5) OVER (PARTITION BY (frequency > 0) ORDER BY recency_hari DESC) END AS r_score,
           CASE WHEN frequency > 0 THEN NTILE(5) OVER (PARTITION BY (frequency > 0) ORDER BY frequency ASC) END AS f_score,
           CASE WHEN frequency > 0 THEN NTILE(5) OVER (PARTITION BY (frequency > 0) ORDER BY monetary ASC) END AS m_score
    FROM rfm_base
)
SELECT customer_id, recency_hari, frequency, monetary,
       r_score, f_score, m_score,
       CASE
           WHEN frequency IS NULL OR frequency = 0 THEN 'No Valid Order'
           WHEN r_score >= 4 AND f_score >= 4 AND m_score >= 4 THEN 'Champions'
           WHEN r_score >= 3 AND f_score >= 3 THEN 'Loyal Customers'
           WHEN r_score >= 3 AND f_score <= 2 THEN 'New Customers'
           WHEN r_score <= 2 AND f_score >= 2 THEN 'At Risk'
           WHEN r_score <= 2 AND f_score <= 1 THEN 'Lost'
           ELSE 'Others'
       END AS segmen
FROM rfm_score
ORDER BY monetary DESC;
```

</details>

### 3.4 Produk Laris yang Stoknya Habis (Potensi Lost Sales)

Produk dengan `stock = 0` yang punya riwayat penjualan, diurutkan berdasarkan total pendapatan historis. Ini hanya estimasi (proxy), karena dataset tidak mencatat kapan stok menjadi 0.

```sql
SELECT p.product_id, p.brand, p.avg_rating,
       SUM(oi.quantity) AS total_terjual_historis,
       SUM(oi.subtotal_idr) AS total_pendapatan_historis,
       COUNT(DISTINCT oi.order_id) AS jumlah_order
FROM products p
JOIN order_items oi ON p.product_id = oi.product_id
WHERE p.stock = 0
GROUP BY p.product_id, p.brand, p.avg_rating
ORDER BY total_pendapatan_historis DESC;
```

### 3.5 Produk Menumpuk (Dead Stock)

Diukur dengan rasio stok terhadap total terjual. Rasio lebih informatif daripada angka penjualan saja, karena membedakan stok 407 dengan penjualan 11 dari stok 11 dengan penjualan 10.

```sql
SELECT p.product_id, p.brand, p.stock,
       COALESCE(SUM(oi.quantity), 0) AS total_terjual,
       ROUND(p.stock::NUMERIC / NULLIF(COALESCE(SUM(oi.quantity), 0), 0), 2) AS rasio_stok_vs_terjual
FROM products p
LEFT JOIN order_items oi ON p.product_id = oi.product_id
GROUP BY p.product_id, p.brand, p.stock
HAVING p.stock > 0
ORDER BY rasio_stok_vs_terjual DESC;
```

---

## 4. Dashboard Power BI

Data diambil langsung dari PostgreSQL (mode Import). Tabel hasil query kompleks (`rfm_detail`, `lost_sales`, `dead_stock`) ditambahkan lewat custom SQL.

| Halaman | Isi |
|---|---|
| **Overview** | Card (total revenue, total order, customer aktif), tren revenue, revenue per provinsi, slicer provinsi |
| **RFM** | Donut chart distribusi segmen, bar chart revenue per segmen, tabel detail customer |
| **Product Analysis** | Lost sales, dead stock, produk terlaris, slicer dari tabel `products` |

Interaktivitas:

- Cross-filtering antar visual di setiap halaman
- Slicer di tiap halaman
- Relationship antar tabel di Model view. Tabel `lost_sales` dan `dead_stock` dihubungkan ke `products` lewat `product_id` agar ikut tersaring slicer

<!-- Tambahkan screenshot dashboard di sini, contoh:
![Overview](images/overview.png)
![RFM](images/rfm.png)
![Product Analysis](images/product_analysis.png)
-->

---

## Insight Utama

1. **Promo belum terbukti mengangkat nilai transaksi.** Rata-rata order dengan promo lebih rendah, dan promo dipakai merata oleh customer baru maupun lama (sekitar 40% di keduanya). Ini mengindikasikan promo belum terarah pada target tertentu.
2. **Segmen At Risk dan Loyal Customers paling besar** (sekitar 27% dan 26% dari customer yang punya order valid). Kampanye retensi untuk At Risk layak diprioritaskan.
3. **Ada produk yang stoknya menumpuk** dengan rasio stok terhadap penjualan tinggi (contoh: PRD137 dengan stok 407 dan terjual 11 unit). Kandidat untuk diskon atau bundling.
4. **Ada produk laris yang stoknya kosong** (contoh: PRD001 dengan pendapatan historis sekitar 15 juta). Kandidat utama untuk restock.
5. **Catatan kualitas data**: sekitar 26% order tercatat sebelum tanggal registrasi customer, dan 3 email dipakai dua customer berbeda.

## Keterbatasan

- Lost sales hanya estimasi berdasarkan penjualan historis, karena tidak ada data tanggal stok habis.
- "Order pertama" berarti order pertama **di dalam dataset**, bukan pembelian pertama sepanjang riwayat customer.
- Recency dihitung dari tanggal order terakhir di dataset, bukan dari tanggal hari ini.
- Batas segmen RFM bersifat heuristik dan bisa disesuaikan dengan kebutuhan bisnis.
- Perbandingan promo bersifat deskriptif. Perbedaan rata-rata transaksi belum diuji secara statistik.
- Dataset tampak seperti data latihan (misalnya tanggal order sebelum registrasi), sehingga temuan sebaiknya dibaca sebagai latihan analisis.

## Cara Menjalankan Ulang

1. Install PostgreSQL dan buat tabel (`CREATE TABLE`) sesuai struktur di atas, lengkap dengan foreign key.
2. Letakkan file CSV di folder yang bisa dibaca service PostgreSQL (misalnya `C:/Dataset/`). Folder di profil user seperti Desktop bisa menyebabkan error *Permission denied*.
3. Import dengan `COPY` **sesuai urutan relasi**: `customers` → `products` → `orders` → `order_items` → `reviews`.

   ```sql
   COPY customers FROM 'C:/Dataset/customers.csv' DELIMITER ',' CSV HEADER;
   ```

4. Jalankan query pembersihan, validasi, dan analisis di atas.
5. Di Power BI Desktop: **Get Data → PostgreSQL database**, pilih tabel, lalu tambahkan query RFM, lost sales, dan dead stock lewat **Advanced options**. Hilangkan tanda `;` di akhir query karena Power BI membungkus query di dalam subquery.
6. Atur relationship di Model view dan susun visual.

---

Latihan ini dibuat sebagai analisis latihan data end-to-end menggunakan PostgreSQL dan PowerBI.
