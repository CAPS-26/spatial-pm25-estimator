# Catatan Analisis Geospatial & Machine Learning (Estimasi PM2.5 Jakarta)

Dokumen ini disusun sebagai panduan ilmiah dan teknis untuk mempermudah penyusunan laporan, skripsi, atau tugas kuliah Anda terkait proyek estimasi PM2.5 secara spasial di Jakarta.

---

## 1. Analisis Integritas Data Target (PM2.5) & Duplikasi Grid

### 🔍 Temuan EDA (Exploratory Data Analysis)
Berdasarkan matriks korelasi Pearson yang diuji pada data PM2.5 dari Open-Meteo Air Quality API:
* **Grup 1 (Slipi, Menteng, Kelapa Gading Indah)** memiliki nilai PM2.5 yang identik (korelasi **`1.000`**).
* **Grup 2 (Jagakarsa, Jatinegara)** memiliki nilai PM2.5 yang identik (korelasi **`1.000`**).

### 💡 Penjelasan Ilmiah
Open-Meteo Air Quality API tidak mengambil data dari sensor fisik di tanah Jakarta, melainkan mengambil data dari **CAMS (Copernicus Atmosphere Monitoring Service)**. CAMS adalah model simulasi kimia atmosfer global yang dijalankan oleh superkomputer ECMWF di Eropa.
* **Resolusi Spasial CAMS**: Model CAMS memiliki resolusi spasial yang cukup kasar, yaitu sekitar **0.4° (sekitar 40 km)**.
* **Dampak Snapping**: Karena bentang kota Jakarta kecil (~30 km × 32 km), seluruh area Jakarta hanya masuk ke dalam 2 kotak grid besar milik CAMS. Akibatnya, meskipun koordinat input stasiun berbeda, Open-Meteo melakukan *snapping* koordinat tersebut ke titik tengah grid terdekat yang menghasilkan data PM2.5 yang seragam (duplikat).

---

## 2. Analisis Data Cuaca (ERA5 & Bilinear Interpolation)

### 🔍 Temuan EDA
Berbeda dengan data PM2.5, data cuaca dari Open-Meteo menunjukkan variasi spasial yang lebih kaya:
* Korelasi suhu antar stasiun bernilai sangat tinggi (`0.94` s.d. `0.99`), tetapi **tidak identik (tidak duplikat)**, kecuali untuk Menteng dan Jatinegara yang masih masuk dalam satu grid.
* **Jagakarsa (Jakarta Selatan)** memiliki rata-rata suhu terdingin (`26.73 °C`) karena faktor ketinggian geografis dan banyaknya vegetasi.
* **Kelapa Gading Indah (Jakarta Utara)** memiliki rata-rata kecepatan angin paling kencang (`7.44 km/jam`) karena dekat dengan garis pantai (pengaruh angin laut).

### 💡 Penjelasan Ilmiah
Open-Meteo mengambil data cuaca dari model reanalisis **ERA5** (resolusi ~30 km) dan menerapkan **Bilinear Interpolation** secara otomatis berdasarkan koordinat asli yang kita minta. Hal ini menghasilkan variasi cuaca lokal yang lebih halus (tidak kaku seperti snapping pada model kualitas udara CAMS).

---

## 3. Karakteristik Data Satelit Himawari-9 (AOD/AOT)

### 📡 Struktur Folder & Nama File JAXA FTP
Melalui penelusuran FTP ke server `ftp.ptree.jaxa.jp`, struktur penyimpanan data Himawari-9 Level 2 ARP (Aerosol Retrieval Product) versi `031` adalah:
* **Path Folder**: `/pub/himawari/L2/ARP/031/{YYYYMM}/{DD}/{HH}/`
* **Nama File**: `NC_H09_{YYYYMMDD}_{HHMM}_L2ARP031_FLDK.02401_02401.nc`
* **Keterangan Nama**:
  * `H09`: Satelit Himawari-9 (menggantikan Himawari-8 sejak 13 Desember 2022).
  * `FLDK.02401_02401`: Citra berskala Full Disk dengan dimensi matriks piksel 2401 × 2401.

### ☀️ Karakteristik Deteksi AOD
* **Variabel Target**: Di dalam file NetCDF milik JAXA, nama variabel AOD ditulis sebagai **`AOT` (Aerosol Optical Thickness)**, yang secara fisis bermakna sama dengan AOD.
* **Batasan Siang Hari (Daytime Only)**: AOD diukur dengan memanfaatkan pantulan cahaya matahari pada partikel aerosol. Oleh karena itu, data AOD **hanya tersedia pada siang hari** (pukul **00:00 s.d. 09:00 UTC** atau **07:00 s.d. 16:00 WIB**). Pada malam hari, folder jam di FTP JAXA tidak dibuat, dan nilai AOD bernilai `NaN`.
* **Kenyataan Tutupan Awan (Cloud Cover) Jakarta**:
  * Berdasarkan ekstraksi data penuh dari tahun 2023 s.d. 2026 (54.850 jam siang total), **91.31% data AOD bernilai kosong (NaN)**. Hanya **8.69% (4.766 jam)** yang memiliki data valid.
  * Persentase kekosongan per stasiun berkisar antara **87.7%** (Kelapa Gading Indah) hingga **92.8%** (Jagakarsa).
  * Hal ini membuktikan bahwa tutupan awan di Jakarta sangat dominan sepanjang tahun (bahkan di musim kemarau), sehingga metode pengisian AOD spasial murni (Opsi 1) tidak layak digunakan secara mandiri.
* **Korelasi AOD vs PM2.5**:
  * Hasil uji korelasi Pearson antara AOD satelit dan PM2.5 darat (pada jam-jam cerah) menghasilkan nilai **`R = 0.2779`** (korelasi positif yang lemah-sedang).

---

## 4. Solusi Pemodelan Spasial dengan Random Forest Regressor (RFR)

Meskipun data target PM2.5 mengalami duplikasi akibat resolusi grid model global, penelitian ini tetap dapat berjalan dengan akurat menggunakan konsep **Downscaling Spasial (Downscaling Statistik)**:

### 1. Keunikan Fitur Prediktor ($X$)
Model RFR dilatih menggunakan prediktor yang bervariasi secara spasial:
* Koordinat ($lat, lon$) stasiun asli yang unik.
* Data AOD Himawari-9 (resolusi tinggi 1 km) yang unik di setiap koordinat stasiun.
* Fitur cuaca lokal yang bervariasi.

### 2. Penanganan Nilai Kosong (NaN) AOD pada RFR
Karena RFR dalam scikit-learn membutuhkan data tanpa `NaN` (yang banyak terjadi pada malam hari/mendung untuk kolom AOD), kita menerapkan **Imputasi Nilai Sentinel (Sentinel Value Imputation)**:
* Mengisi nilai `NaN` AOD dengan nilai **`-999.0`**.
* RFR (pohon keputusan) secara otomatis akan belajar membagi cabang (*split*):
  * **Cabang AOD < 0 (Mendung/Malam)**: RFR mengabaikan AOD dan mengandalkan fitur cuaca untuk memprediksi PM2.5.
  * **Cabang AOD >= 0 (Cerah)**: RFR menggunakan kombinasi cuaca dan nilai AOD satelit untuk akurasi maksimal.


### 3. Downscaling Spasial pada Tahap Pemetaan (Mapping)
Saat memprediksi PM2.5 di ribuan titik grid 1 km × 1 km Jakarta pada Fase 5, model RFR akan menghasilkan estimasi PM2.5 yang sangat halus, unik, dan detail di setiap grid karena koordinat dan data AOD satelit yang diinputkan unik di setiap piksel 1 km. Ini adalah teknik pemetaan yang valid dan diakui secara akademis.

---

## 5. Efek Penambahan Titik Koordinat Baru pada Model Open-Meteo

Apakah menambahkan lebih banyak titik stasiun akan meningkatkan kualitas prediksi spasial model? Jawabannya bergantung pada skala geografis penambahannya:

### 1. Penambahan Jarak Dekat (Di Dalam Jakarta)
Jika menambahkan titik-titik stasiun baru yang lokasinya berdekatan (misal sesama Jakarta Pusat atau Jakarta Barat):
* **Efisiensi Rendah (Diminishing Returns)**: Titik-titik ini kemungkinan besar akan ter-*snap* ke grid model CAMS (Open-Meteo) yang sama. Hal ini mengakibatkan data target PM2.5 ($Y$) dan cuaca ($X$) menjadi identik (duplikat), yang secara matematis hanya meningkatkan bobot data tersebut di dalam Random Forest tanpa menambahkan informasi pola baru yang mendasar.
* **Manfaat Minor**: Meskipun data PM2.5-nya identik, karena koordinat asli dan data AOD satelit Himawari-9 ($X_{AOD}$) tetap unik di setiap titik tersebut, model RFR masih mendapatkan sedikit manfaat dalam mempelajari smoothing spasial AOD lokal.

### 2. Penambahan Jarak Jauh/Regional (Jabodetabek)
Jika menambahkan titik-titik stasiun di kota penyangga sekitar Jakarta (misal: **Depok, Bogor, Tangerang, Bekasi, atau Kepulauan Seribu**):
* **Manfaat Sangat Tinggi (High Benefit)**: Titik-titik ini akan jatuh ke **sel grid CAMS (Open-Meteo) yang berbeda**.
* Hal ini memberikan variasi data target PM2.5 ($Y$) yang riil dan unik (misalnya, udara Kepulauan Seribu jauh lebih bersih, sedangkan Bekasi lebih berpolusi karena faktor industri).
* Model Random Forest akan belajar mendeteksi gradien dan pola dispersi polusi lintas kota secara nyata, yang secara drastis meningkatkan kemampuan model untuk melakukan prediksi di wilayah baru (*generalization*).

---

## 6. Desain Fitur Machine Learning (Feature Engineering)

Saat mempersiapkan dataset untuk Random Forest Regressor (RFR), terdapat dua aturan desain penting yang diterapkan dalam proyek ini:

### 1. Mengapa Kita Tidak Menggunakan Fitur Lag PM2.5?
Dalam pemodelan data runtun waktu (*time-series forecasting*), nilai PM2.5 di jam sebelumnya ($T-1$, $T-2$) adalah prediktor yang sangat kuat. Namun, dalam konteks **estimasi spasial (spatial mapping/downscaling)**:
* **Tujuan Akhir**: Memprediksi polusi PM2.5 di wilayah baru yang **tidak memiliki sensor pemantau sama sekali** (Phase 5).
* **Kendala**: Di wilayah baru tersebut, kita tidak memiliki sensor darat untuk memberi tahu model berapa nilai PM2.5 pada $T-1$.
* **Keputusan**: Kita **wajib mengabaikan** fitur lag PM2.5 dari model agar model dapat melakukan prediksi murni menggunakan data satelit AOD dan cuaca di wilayah tanpa stasiun.

### 2. Mengapa Arah Angin Harus Didekomposisi?
Arah angin diukur dalam derajat lingkaran (0° hingga 360°), yang merupakan fitur sirkular.
* **Masalah pada Decision Trees (Random Forest)**: Model pohon keputusan membagi data berdasarkan ambang batas linier ($X_i \le \theta$). Arah angin 359° (Utara-Barat Laut) secara fisik sangat dekat dengan 1° (Utara-Timur Laut). Namun, Random Forest akan membacanya secara numerik sebagai dua nilai yang sangat berjauhan (satu mendekati maksimum, satu mendekati minimum).
* **Solusi (Dekomposisi Vektor)**: Kita mengubah kecepatan angin ($ws$) dan arah angin dalam radian ($\theta_{rad}$) menjadi komponen vektor angin Timur-Barat ($u$) dan Utara-Selatan ($v$):
  $$u = ws \times \cos(\theta_{rad})$$
  $$v = ws \times \sin(\theta_{rad})$$
* **Manfaat**: Metode ini melestarikan kedekatan fisis arah angin (misalnya, arah 359° dan 1° akan memiliki nilai $u$ dan $v$ yang sangat dekat), sehingga Random Forest dapat memproses arah angin secara akurat untuk mendeteksi adveksi (pergerakan) polutan.


