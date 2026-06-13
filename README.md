# Spatial PM2.5 Estimator (Jakarta)

Proyek ini bertujuan untuk mengestimasi konsentrasi PM2.5 di wilayah Daerah Khusus Ibukota (DKI) Jakarta yang tidak memiliki stasiun pemantau kualitas udara. Pendekatan yang digunakan adalah **Random Forest Regression (RFR)** dengan memanfaatkan data cuaca (temperatur, kelembapan, kecepatan/arah angin) serta data Aerosol Optical Depth (AOD) dari citra satelit Himawari.

---

## 📌 Daftar Isi
1. [Struktur Proyek Modular](#-struktur-proyek-modular)
2. [Hasil Akhir Model (Model Performance)](#-hasil-akhir-model-model-performance)
3. [Variabel Prediktor & Rekayasa Fitur (Features & Engineering)](#-variabel-prediktor--rekayasa-fitur-features--engineering)
4. [Panduan Memulai (Setup)](#%EF%B8%8F-panduan-memulai-setup)
5. [Manajemen Dependensi (requirements.txt)](#-manajemen-dependensi-requirementstxt)
6. [Backend Integration & Model Deployment (.pkl)](#-backend-integration--model-deployment-pkl)
7. [Analisis Perbandingan dengan Penelitian Pendahulu](docs/perbandingan_penelitian.md)

---

## 📂 Struktur Proyek Modular

Untuk menjaga efisiensi memori RAM dan menghindari kontaminasi variabel, pipeline proyek ini dibagi menjadi 5 notebook Jupyter terpisah:

1. **`01_Data_Extraction.ipynb`**
   * **Fokus:** Membaca data CSV stasiun, memuat koordinat stasiun, mengekstrak nilai piksel AOD dari raster Himawari (NetCDF/TIFF), dan menggabungkannya dengan data cuaca.
   * **Output:** `raw_merged_data.csv`.

2. **`02_Cleaning_and_Imputation.ipynb`**
   * **Fokus:** Menangani nilai AOD yang hilang (*missing values*) akibat tutupan awan (menggunakan interpolasi temporal/spasial) dan membersihkan data anomali.
   * **Output:** `clean_imputed_data.csv`.

3. **`03_Feature_Engineering_and_EDA.ipynb`**
   * **Fokus:** Ekstraksi fitur waktu (jam, hari, bulan), fitur spasial (latitude/longitude), dan analisis korelasi antar variabel.
   * **Output:** `model_ready_data.csv`.

4. **`04_Modeling_RFR.ipynb`**
   * **Fokus:** Melatih model Random Forest Regressor dengan skema validasi **Leave-One-Station-Out Cross-Validation (LOSOCV)**, hyperparameter tuning, dan analisis *feature importance*.
   * **Output:** Model terlatih (`pm25_rfr_model.pkl` atau `.joblib`) dan metrik performa (R², RMSE, MAE).

5. **`05_Spatial_Prediction_and_Mapping.ipynb`**
   * **Fokus:** Membuat grid spasial 1km x 1km Jakarta, memprediksi PM2.5 pada seluruh titik grid, dan memvisualisasikan peta sebaran polusi menggunakan peta interaktif (*Folium/Matplotlib*).
   * **Output:** Peta polusi spasial Jakarta.

---

## 🏆 Hasil Akhir Model (Model Performance)

Berdasarkan hasil validasi harian pada tanggal netral (**18 Juni 2025**), model **Random Forest Regressor (RFR) Tuned dengan Koordinat + Weather Lags** terpilih sebagai model terbaik dengan performa sebagai berikut:

* **MAE Harian**: **`23.70 µg/m³`**
* **RMSE Harian**: **`26.81 µg/m³`**

### 📊 Kurva Validasi Harian (Diurnal Trend)
Model sangat akurat dalam mendeteksi tren naik-turun konsentrasi harian serta puncak polusi malam hari:

![Tren Harian PM2.5 Kembangan - 18 Juni 2025](results/images/daily_validation_coords_comparison.png)

### 🗺️ Peta Distribusi Spasial PM2.5 Jakarta (Resolusi 1km)
Perbandingan pemetaan spasial Jakarta menunjukkan model RFR mampu melokalisasi polusi dengan membagi wilayah Jakarta Utara (bersih) dan Jakarta Selatan (tinggi) secara spasial:

![Perbandingan Distribusi Spasial PM2.5 Jakarta](results/images/jakarta_pm25_map_comparison.png)

---

## 📊 Variabel Prediktor & Rekayasa Fitur (Features & Engineering)

Dataset siap latih (`data/model_ready_data.csv`) terdiri dari data cuaca primer hasil ekstraksi Open-Meteo, data satelit Himawari-9, serta fitur hasil rekayasa (*feature engineering*) untuk menjaga integritas fisis atmosfer Jakarta:

### 1. Fitur Cuaca Primer (Open-Meteo API)
* **`temperature_2m`**: Suhu udara aktual pada ketinggian 2 meter (°C).
* **`apparent_temperature`**: Suhu semu/apparent temperature (°C) yang dirasakan.
* **`relative_humidity_2m`**: Kelembaban relatif udara pada ketinggian 2 meter (%).
* **`dew_point_2m`**: Titik embun uap air udara pada ketinggian 2 meter (°C).
* **`precipitation`**: Total presipitasi (akumulasi curah hujan cair & padat) (mm).
* **`rain`**: Curah hujan cair (mm).
* **`surface_pressure`**: Tekanan udara di permukaan tanah (hPa).
* **`cloud_cover_total`**: Tutupan awan total (%).

### 2. Fitur Satelit Primer (JAXA Himawari-9 FLDK)
* **`AOD`** (*Aerosol Optical Depth*): Ketebalan optik aerosol di kolom atmosfer.

### 3. Fitur Hasil Rekayasa (*Feature Engineering*)
* **Dekomposisi Vektor Angin (Trigonometris)**:
  * **`u_wind`**: Komponen kecepatan angin Timur-Barat (zonal wind). Hasil dekomposisi kecepatan angin (`wind_speed_10m`) dan arah angin derajat (`wind_direction_10m`) menggunakan fungsi cosinus (cos).
  * **`v_wind`**: Komponen kecepatan angin Utara-Selatan (meridional wind). Hasil dekomposisi menggunakan fungsi sinus (sin).
* **Inersia Atmosfer (1-Hour Weather Lags)**:
  * **`temp_lag1`**: Suhu udara tertunda 1 jam sebelumnya.
  * **`rh_lag1`**: Kelembaban relatif tertunda 1 jam sebelumnya.
  * **`u_wind_lag1`**: Komponen angin Timur-Barat tertunda 1 jam sebelumnya.
  * **`v_wind_lag1`**: Komponen angin Utara-Selatan tertunda 1 jam sebelumnya.
* **Jangkar Spasial (Spatial Anchor)**:
  * **`latitude`** & **`longitude`**: Koordinat lintang dan bujur stasiun asli untuk melokalisasi prediksi spasial di luar stasiun latih.
* **Fitur Temporal (Siklus Waktu)**:
  * **`jam`**: Jam UTC (0-23) untuk menangkap fluktuasi harian batas atmosfer (PBL) dan aktivitas emisi kendaraan.
  * **`bulan`**: Bulan (1-12) untuk mendeteksi variasi musim (kemarau vs hujan).
  * **`hari_dalam_minggu`**: Hari dalam seminggu (0-6, Senin-Minggu) untuk membedakan emisi hari kerja vs akhir pekan.
  * **`is_weekend`**: Flag biner akhir pekan (1 jika Sabtu/Minggu, 0 jika Senin-Jumat).

---

## 🛠️ Panduan Memulai (Setup)

### 1. Mengaktifkan Virtual Environment
Proyek ini menggunakan virtual environment lokal (`.venv`) agar pustaka terisolasi dengan aman.

* **Windows (PowerShell):**
  ```powershell
  .\.venv\Scripts\Activate.ps1
  ```
* **Windows (Command Prompt):**
  ```cmd
  .\.venv\Scripts\activate.bat
  ```

### 2. Memilih Kernel di Jupyter / VS Code
Virtual environment `.venv` sudah didaftarkan sebagai kernel Jupyter dengan nama **`Python (spatial-pm25-estimator)`**.
* **Di VS Code:** Buka notebook apa pun (misal `01_Data_Extraction.ipynb`), klik tombol **Select Kernel** di pojok kanan atas, lalu pilih **Python (spatial-pm25-estimator)**.
* **Di Jupyter Notebook / Lab:** Di menu atas, pilih **Kernel** -> **Change Kernel** -> **Python (spatial-pm25-estimator)**.

---

## 📦 Manajemen Dependensi (requirements.txt)

File `requirements.txt` memisahkan pustaka berdasarkan kebutuhan fase proyek agar mempercepat instalasi awal:

### Fase 1: Ekstraksi Data (Sudah Terinstal)
Pustaka berikut sudah terinstal di dalam `.venv`:
* **Tabular:** `numpy`, `pandas`
* **Geospatial & Citra:** `geopandas`, `shapely`, `rasterio`, `xarray`, `netCDF4`, `rioxarray`
* **Jupyter Kernel:** `ipykernel`

### Fase Berikutnya (Pembersihan, Pemodelan, Visualisasi)
Saat Anda masuk ke notebook `03`, `04`, atau `05`, Anda perlu menginstal pustaka modeling dan visualisasi. Caranya:
1. Hapus tanda komentar (`#`) pada pustaka berikut di dalam [requirements.txt](requirements.txt):
   ```text
   scikit-learn
   matplotlib
   seaborn
   folium
   joblib
   ```
2. Jalankan perintah instalasi berikut di terminal:
   ```powershell
   .\.venv\Scripts\python.exe -m pip install -r requirements.txt
   ```

---

## 🤖 Backend Integration & Model Deployment (.pkl)

Karena berkas model `.pkl` (Pickle) berukuran besar (>500 MB) dan diabaikan oleh Git via `.gitignore`, pengembang backend perlu menyiapkan model tersebut dengan salah satu dari dua cara berikut agar sistem backend/API siap memprediksi PM2.5:

### 1. Cara Memperoleh Berkas Model (`.pkl`)

#### 🔹 Opsi A: Mengunduh dari GitHub Releases (Direkomendasikan untuk Produksi)
1. Buka halaman repositori Git Anda, lalu masuk ke bagian **Releases**.
2. Unduh berkas **`pm25_rfr_model.pkl`** (Random Forest final teroptimasi) dan/atau **`pm25_etr_model.pkl`** (Extra Trees).
3. Buat folder baru bernama `data/` di direktori utama backend Anda jika belum ada.
4. Letakkan berkas `.pkl` yang telah diunduh ke dalam folder `data/` tersebut:
   ```text
   [direktori-utama-backend]/
   └── data/
       ├── pm25_rfr_model.pkl
       └── pm25_etr_model.pkl
   ```

#### 🔹 Opsi B: Membuat Model Sendiri dari Nol (Generate via Notebook)
Jika berkas rilis belum tersedia atau Anda ingin memperbarui model dengan data latihan baru:
1. Buka notebook **`04_Modeling_RFR.ipynb`** menggunakan editor Jupyter (VS Code atau Jupyter Lab).
2. Pastikan berkas dataset latihan **`data/model_ready_data.csv`** sudah ada (diperoleh dari notebook `03`).
3. Jalankan semua cell pada notebook tersebut (*Run All*).
4. Kode akan otomatis melatih model dan menyimpannya ke folder `data/` dengan nama berkas `pm25_rfr_model.pkl` dan `pm25_etr_model.pkl`.

---

### 2. Setup dan Cara Pemuatan Model di Backend (Python)

Sebelum menulis kode API, pengembang backend wajib memahami asal-usul dari 21 fitur input model RFR untuk disiapkan di database/runtime backend:

*   **A. Diambil Langsung dari Open-Meteo Weather API**:
    *   `temperature_2m` (Suhu udara, °C)
    *   `apparent_temperature` (Suhu semu, °C)
    *   `relative_humidity_2m` (Kelembaban relatif, %)
    *   `dew_point_2m` (Titik embun, °C)
    *   `precipitation` (Presipitasi, mm)
    *   `rain` (Curah hujan, mm)
    *   `surface_pressure` (Tekanan permukaan, hPa)
    *   `cloud_cover_total` (Tutupan awan, %)
*   **B. Diambil dari Citra Satelit JAXA Himawari-9**:
    *   `AOD` (*Aerosol Optical Depth*). Jika data AOD kosong akibat malam/mendung, **wajib diisi sentinel value `-999.0`**.
*   **C. Dihitung secara Programmatic di Backend (Feature Engineering)**:
    *   **Koordinat Titik Target**: `latitude` dan `longitude` stasiun atau koordinat piksel grid Jakarta.
    *   **Dekomposisi Vektor Angin**:
        *   `u_wind` = `wind_speed_10m * cos(radian_wind_direction_10m)`
        *   `v_wind` = `wind_speed_10m * sin(radian_wind_direction_10m)`
        *   *Catatan: Radian diperoleh dari `(derajat * pi) / 180`*.
    *   **Inersia Atmosfer (Weather Lags 1 Jam Sebelumnya)**:
        *   `temp_lag1` = `temperature_2m` pada jam T-1
        *   `rh_lag1` = `relative_humidity_2m` pada jam T-1
        *   `u_wind_lag1` = `u_wind` pada jam T-1
        *   `v_wind_lag1` = `v_wind` pada jam T-1
    *   **Fitur Temporal (Ekstraksi Waktu)**:
        *   `jam` (Format 0-23 UTC), `bulan` (1-12), `hari_dalam_minggu` (0-6, di mana 0 = Senin), dan `is_weekend` (1 jika Sabtu/Minggu, 0 jika hari kerja).

Pastikan pustaka `joblib` dan `scikit-learn` (versi `1.6.0` atau yang sesuai saat pelatihan) sudah terinstal di lingkungan backend Anda:
```bash
pip install joblib scikit-learn pandas numpy
```


Berikut adalah contoh skrip Python backend untuk memuat model dan melakukan prediksi:

```python
import os
import joblib
import pandas as pd

# 1. Tentukan path model
MODEL_PATH = os.path.join('data', 'pm25_rfr_model.pkl')

if not os.path.exists(MODEL_PATH):
    raise FileNotFoundError(f"Model pkl tidak ditemukan di: {MODEL_PATH}. Silakan unduh dari GitHub Release.")

# 2. Muat model
model = joblib.load(MODEL_PATH)
print("✓ Model Random Forest Regressor berhasil dimuat.")

# 3. Definisikan 21 Fitur Latihan (Urutan Wajib Sama Persis)
FEATURE_COLUMNS = [
    'latitude', 'longitude', 
    'temperature_2m', 'apparent_temperature', 'relative_humidity_2m',
    'dew_point_2m', 'precipitation', 'rain', 'surface_pressure',
    'cloud_cover_total', 'u_wind', 'v_wind', 'jam', 'bulan',
    'hari_dalam_minggu', 'is_weekend', 'AOD',
    'v_wind_lag1', 'u_wind_lag1', 'temp_lag1', 'rh_lag1'
]

# 4. Contoh data input API (Contoh 1 baris prediksi)
# Backend wajib memastikan semua input fitur ini sudah terisi dan diinterpolasi spasial-temporal
input_data = {
    'latitude': -6.1878,
    'longitude': 106.7264,
    'temperature_2m': 28.5,
    'apparent_temperature': 31.2,
    'relative_humidity_2m': 78.0,
    'dew_point_2m': 24.1,
    'precipitation': 0.0,
    'rain': 0.0,
    'surface_pressure': 1008.2,
    'cloud_cover_total': 45.0,
    'u_wind': 1.2,
    'v_wind': -2.5,
    'jam': 12,                    # Format 24 Jam
    'bulan': 6,                   # Juni
    'hari_dalam_minggu': 2,       # Selasa (0-6)
    'is_weekend': 0,              # False (0)
    'AOD': 0.35,                  # Jika mendung/malam, isi dengan sentinel -999.0
    'v_wind_lag1': -2.1,          # Lag cuaca 1 jam sebelumnya
    'u_wind_lag1': 1.0,
    'temp_lag1': 27.9,
    'rh_lag1': 80.0
}

# Convert ke DataFrame dengan urutan kolom yang benar
df_input = pd.DataFrame([input_data])[FEATURE_COLUMNS]

# 5. Lakukan estimasi
estimated_pm25 = model.predict(df_input)[0]
print(f"Prediksi PM2.5 di koordinat tersebut: {estimated_pm25:.2f} µg/m³")
```

### ⚠️ Catatan Penting untuk Backend Developer:
1. **Urutan Fitur**: Scikit-learn sangat bergantung pada urutan kolom input. DataFrame yang dimasukkan ke `model.predict()` **wajib** diurutkan sesuai urutan dalam array `FEATURE_COLUMNS` di atas.
2. **Sentinel Value AOD**: AOD Himawari-9 kerap bernilai kosong pada malam hari atau saat hujan tebal. Isi dengan nilai sentinel **`-999.0`** jika tidak ada data dari satelit. Model telah dilatih untuk menangani nilai ini.
3. **Penyediaan Lag Cuaca**: Fitur lag 1 jam (`temp_lag1`, dll) merujuk ke parameter cuaca pada jam T-1. Backend wajib mengambil data cuaca dari histori ERA5/Open-Meteo pada jam sebelumnya untuk melengkapi input ini.
