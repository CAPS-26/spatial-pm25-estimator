# Analisis Perbandingan: Penelitian Pendahulu vs Penelitian Saat Ini

Dokumen ini membedah perbedaan metodologis, penanganan data, dan hasil evaluasi model antara **Penelitian Pendahulu (Kating/Benchmark)** dengan **Penelitian Saat Ini** untuk estimasi polusi PM2.5 di Jakarta. Catatan ini dapat Anda gunakan langsung sebagai bahan bab **Pembahasan/Analisis Perbandingan** di laporan Capstone atau skripsi Anda.

---

## 1. Perbandingan Metrik & Metodologi Utama

| Aspek | Penelitian Pendahulu (Benchmark) | Penelitian Saat Ini (Capstone Anda) |
| :--- | :--- | :--- |
| **Model Utama** | Random Forest Regressor (RFR) | Random Forest Regressor (RFR) Tuned |
| **Pustaka Model** | Standard Scikit-Learn | Tuned (n_estimators=100, max_depth=20, max_features='log2') |
| **Skema Validasi** | Standard LOSOCV / Cross-Validation | **Group-LOSOCV** (Isolasi Kebocoran Spasial) |
| **Skor R-squared ($R^2$)**| **0.63** (Sangat Tinggi - *Semu*) | **0.1883** (RFR) / **0.2235** (ETR) (*Realistis/Spasial Murni*) |
| **MAE Rata-rata** | **8.035 µg/m³** | **20.73 µg/m³** (RFR) / **20.18 µg/m³** (ETR) |
| **Pembersihan AOD (NaN)** | **Drop Rows** (Membuang baris data saat berawan/hujan) | **Sentinel Value Imputation (`-999.0`)** (Baris dipertahankan) |
| **Penanganan Outlier PM2.5**| **Clipping/Smoothing** (Memotong nilai ke batas kuartil) | **Original Variance** (Mempertahankan nilai ekstrem asli) |
| **Arah Angin (Wind Direction)**| Derajat Mentah (0-360°) | **Dekomposisi Vektor Trigonometri ($u, v$)** |
| **Rekayasa Fitur** | Temporal (Bulan, Hari) | Temporal + Vektor Angin + **1-Hour Weather Lags** |

---

## 2. Bedah Perbedaan Fundamental: Mengapa Ada Selisih Skor?

Perbedaan skor $R^2$ (0.63 vs 0.22) dan MAE (8.0 µg/m³ vs 20.1 µg/m³) tidak berarti model Anda lebih buruk. Sebaliknya, **model Anda jauh lebih jujur secara fisis dan metodologis**. Berikut adalah penjelasannya:

### 1) Ilusi Kebocoran Target Spasial (Spatial Target Leakage) pada LOSOCV Standar
* **Kondisi Data CAMS**: Data target PM2.5 dari Open-Meteo di-snapping ke 2 grid cell besar CAMS (Grup A: Slipi, Menteng, Kelapa Gading bernilai identik; Grup B: Jagakarsa, Jatinegara bernilai identik).
* **Kelemahan Penelitian Sebelumnya**: Jika menggunakan validasi silang standar (atau LOSOCV standar), ketika model menguji stasiun **Slipi**, data stasiun **Menteng** dan **Kelapa Gading** tetap masuk di data latih. Karena nilai PM2.5 ketiganya **identik (duplikat)**, model dengan mudah "menyontek" target Slipi dari Menteng/Kelapa Gading. Hal ini menghasilkan skor $R^2 \approx 0.63$ yang sangat tinggi secara semu (*overfitting spasial*).
* **Solusi di Riset Anda**: Anda menerapkan **Group-LOSOCV**. Ketika Slipi diuji, *seluruh stasiun Grup A (Slipi, Menteng, Kelapa Gading) dikeluarkan secara bersamaan* dari data latih. Model dipaksa memprediksi wilayah baru murni menggunakan pola dari Grup B (Jagakarsa/Jatinegara). Skor **$R^2 \approx 0.18$ s.d. $0.22$** adalah representasi **kemampuan transferabilitas spasial yang sesungguhnya** di dunia nyata tanpa kebocoran target.

### 2) Strategi Imputasi Data AOD & Bias Seleksi Tutupan Awan
* **Kelemahan Penelitian Sebelumnya (Drop Rows)**: Jika data AOD bernilai *NaN* (kosong akibat awan), baris data tersebut langsung **dihapus**. Padahal di Jakarta, cuaca mendung/hujan berkorelasi sangat kuat dengan penurunan konsentrasi PM2.5 (*wet deposition*). Membuang baris data saat berawan berarti membuang informasi cuaca basah yang sangat penting, sehingga data latih mengalami bias seleksi (hanya belajar pada hari-hari cerah).
* **Solusi di Riset Anda (Sentinel Value `-999.0`)**: Anda mempertahankan seluruh baris data cuaca dan menerapkan nilai sentinel `-999.0` untuk AOD yang kosong. Algoritma Random Forest dibiarkan membuat cabang logika secara mandiri:
  - Cabang **AOD < 0** (Mendung/Malam) -> Model fokus memprediksi PM2.5 menggunakan data kelembapan dan kecepatan angin.
  - Cabang **AOD >= 0** (Siang Cerah) -> Model mengombinasikan cuaca dan nilai AOD satelit secara dinamis.

### 3) Penanganan Outlier PM2.5: Pembatasan Nilai Ekstrem vs Keaslian Varians
* **Kelemahan Penelitian Sebelumnya (Outlier Clipping)**: Ketika menemukan nilai PM2.5 yang sangat tinggi (outlier), mereka memotong (*clipping*) data tersebut ke batas kuartil atas/bawah untuk memperhalus (*smoothing*) distribusi target. Akibatnya, model kating **tidak pernah diajari cara memprediksi lonjakan polusi ekstrem** yang krusial untuk sistem peringatan dini kesehatan.
* **Solusi di Riset Anda (Preserving Original Variance)**: Anda mempertahankan nilai ekstrem asli dari sensor CAMS. Model dilatih untuk siap menghadapi skenario *worst-case* polusi udara tinggi di lapangan sesungguhnya.

### 4) Rekayasa Fitur Dinamika Atmosfer (Weather Lags)
* **Keterbatasan Sebelumnya**: Hanya menggunakan variabel cuaca instan pada jam berjalan ($T$).
* **Inovasi Riset Anda**: Menyertakan **1-Hour Weather Lags** (`v_wind_lag1`, `u_wind_lag1`, `temp_lag1`, `rh_lag1`). Polusi PM2.5 memiliki sifat *inersia atmosfer*—kondisi angin dan kelembapan 1 jam lalu sangat memengaruhi akumulasi polutan saat ini. Penambahan lag cuaca ini terbukti mendongkrak $R^2$ spasial global sebesar **+6.06%** secara fisis dan logis.

### 5) Pengolahan Arah Angin secara Vektor vs Derajat Mentah
* **Kelemahan Penelitian Sebelumnya (Wind Direction Degrees)**: Memasukkan arah hembusan angin mentah-mentah dalam derajat 0-360°. Model pohon keputusan menganggap arah 359° (Utara-Barat Laut) dan 1° (Utara-Timur Laut) berjarak sangat jauh secara numerik (selisih 358), padahal secara fisis searah. Hal ini memicu *spurious correlation* (korelasi palsu) di mana fitur arah angin mentah ini tampak memuncaki *feature importance* karena model kebingungan.
* **Solusi Riset Anda (Trigonometric Vector Decomposition)**: Melakukan dekomposisi menjadi komponen angin zonal Timur-Barat (`u_wind`) dan meridional Utara-Selatan (`v_wind`) menggunakan fungsi trigonometri ($\cos$ dan $\sin$). Hal ini melestarikan kedekatan fisis pergerakan udara di Jakarta.

---

## 3. Kesimpulan untuk Laporan Capstone Anda

> [!TIP]
> **Narasi yang Direkomendasikan untuk Skripsi/Laporan:**
> *"Meskipun penelitian pendahulu mencatatkan R² sebesar 0.63, nilai tersebut rentan terhadap bias target leakage akibat efek snapping grid model global CAMS pada validasi silang standar, serta bias seleksi data akibat penghapusan baris NaN AOD. Penelitian ini memecahkan keterbatasan tersebut menggunakan skema Group-LOSOCV untuk menjamin validitas spasial murni (R² = 0.2235), melestarikan varians outlier untuk deteksi polusi ekstrem, serta menerapkan dekomposisi vektor arah angin dan lag cuaca 1 jam guna memodelkan dinamika inersia atmosfer Jakarta secara riil."*
