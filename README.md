# Spatial PM2.5 Estimator (Jakarta)

Proyek ini bertujuan untuk mengestimasi konsentrasi PM2.5 di wilayah Daerah Khusus Ibukota (DKI) Jakarta yang tidak memiliki stasiun pemantau kualitas udara. Pendekatan yang digunakan adalah **Random Forest Regression (RFR)** dengan memanfaatkan data cuaca (temperatur, kelembapan, kecepatan/arah angin) serta data Aerosol Optical Depth (AOD) dari citra satelit Himawari.

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
1. Hapus tanda komentar (`#`) pada pustaka berikut di dalam [requirements.txt](file:///c:/Users/LENOVO/Documents/0_Tugas Kuliah/Project/spatial-pm25-estimator/requirements.txt):
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
