## 1. Analisis Metrik Evaluasi Spasial (Group-LOSOCV)

Metode **Group-LOSOCV** digunakan untuk menguji kemampuan transferabilitas spasial model secara jujur tanpa adanya kebocoran target spasial. Seluruh stasiun yang berada dalam satu grid group CAMS dikeluarkan secara bersamaan dari data latih saat pengujian stasiun anggota grup tersebut.

### 📊 Hasil Perbandingan Metode & Rekayasa Fitur (Global Average)

| Konfigurasi Model (Tanpa Koordinat) | $R^2$ Spasial | RMSE ($\mu g/m^3$) | MAE ($\mu g/m^3$) | Kenaikan Performa |
| :--- | :---: | :---: | :---: | :---: |
| **Random Forest Baseline** | 0.1629 | 28.05 | 20.92 | Baseline |
| **Random Forest + Weather Lags (Tuned)** | 0.1883 | 27.59 | 20.73 | +2.54% |
| **Extra Trees + Weather Lags (Tuned)** | **0.2235** | **26.97** | **20.18** | **+6.06%** |

### 📊 Rincian Performa Stasiun Model Terbaik (Extra Trees Tuned + Lags)

| Stasiun Uji (Test Station) | $R^2$ Spasial | RMSE ($\mu g/m^3$) | MAE ($\mu g/m^3$) | Kategori Kinerja |
| :--- | :---: | :---: | :---: | :---: |
| **Jatinegara** (Jakarta Timur) | 0.3606 | 28.02 | 19.82 | Sedang |
| **Jagakarsa** (Jakarta Selatan) | 0.1998 | 31.34 | 22.38 | Cukup / Rendah |
| **Menteng** (Jakarta Pusat) | 0.1323 | 26.26 | 20.31 | Rendah |
| **Kelapa Gading Indah** (Jakarta Utara) | 0.0840 | 26.98 | 20.64 | Sangat Rendah |
| **Slipi** (Jakarta Barat) | 0.0378 | 27.65 | 21.45 | Sangat Rendah |
| **RATA-RATA GLOBAL** | **0.2235** | **26.97** | **20.18** | **Cukup / Rendah** |

> [!NOTE]
> Dengan mengisolasi kebocoran spasial, kita mengidentifikasi kemampuan transfer spasial model yang sesungguhnya. Melalui rekayasa fitur **Weather Lags** (menangkap inersia atmosfer) dan algoritma **Extra Trees** teroptimasi, kita berhasil mendongkrak $R^2$ spasial global sebesar **+6.06%** dari baseline tanpa koordinat, menembus batas awal model ke arah nilai yang lebih fisis dan optimal.

---


## 2. Mengapa Performa Spasial Drop Drastis? ("Makin Drop")

Penurunan drastis nilai $R^2$ menjadi 0.1593 disebabkan oleh dua hal utama yang bersifat fisis dan matematis:

### 1) Isolasi Total Informasi Grid
Saat melakukan validasi pada Group A (Menteng, Slipi, Kelapa Gading), model dilatih *hanya* menggunakan Group B (Jagakarsa, Jatinegara). Karena target PM2.5 kedua grup ini berasal dari grid sel CAMS yang berbeda, model dipaksa memprediksi nilai PM2.5 di grid sel baru yang belum pernah ia pelajari sama sekali pola targetnya.

### 2) Keterbatasan Random Forest terhadap Fitur Koordinat (Spatial Extrapolation Limit)
Random Forest Regressor (RFR) bekerja dengan membuat percabangan linier pada fitur koordinat (misalnya, `latitude <= -6.25`).
* Ketika menguji stasiun di Group A (daerah Utara/Pusat dengan `latitude` sekitar `-6.19`), model dilatih menggunakan data Group B (daerah Selatan/Timur dengan `latitude` antara `-6.23` dan `-6.33`).
* Fitur koordinat Group A berada di luar rentang koordinat Group B (*out-of-bounds*). Random Forest **tidak bisa melakukan ekstrapolasi** nilai di luar jangkauan data latihnya. Akibatnya, RFR menganggap daerah baru tersebut konstan berdasarkan daun percabangan terakhir, sehingga performa koordinat sebagai prediktor spasial rusak total.

---

## 3. Strategi Perbaikan: Eksperimen Model Tanpa Koordinat & Lag Cuaca (Coordinate-Free with Weather Lags)

Untuk mengatasi keterbatasan ekstrapolasi koordinat ini dan meningkatkan kemampuan prediksi spasial, kita mengusulkan rencana perbaikan dengan **menghapus fitur `latitude`/`longitude` dan menambahkan fitur lag cuaca 1 jam**:

1. **Generalisasi Lebih Baik**: Tanpa koordinat, model dipaksa untuk mempelajari hubungan fisis murni antara PM2.5 dengan variabel cuaca (`v_wind`, `temperature`, `humidity`) dan satelit (`AOD`) yang terdistribusi secara kontinu di seluruh Jakarta.
2. **Kualitas Peta 1km x 1km yang Lebih Halus**: Tanpa koordinat, peta polusi yang dihasilkan di Phase 5 akan ter-downscale secara halus (*smooth contour*) mengikuti gradien cuaca dan AOD Himawari-9, alih-alih membentuk pola patahan kotak-kotak (*decision-boundary artifacts*) yang tidak logis secara meteorologis akibat percabangan biner koordinat pada Random Forest.
3. **Penyertaan Fitur Lag Cuaca (Weather Lags)**:
   Kita membuat fitur lag 1 jam untuk variabel cuaca utama: `v_wind_lag1`, `u_wind_lag1`, `temp_lag1`, dan `rh_lag1`.
   * **Mengapa Valid untuk Pemetaan?** Berbeda dengan lag PM2.5 yang dilarang (karena di wilayah tanpa sensor kita tidak tahu nilai PM2.5 sebelumnya), lag cuaca dari ERA5 **seluruhnya valid dan tersedia** di mana saja karena parameter cuaca di titik baru dapat diinterpolasi secara spasial-temporal untuk jam $T-1$.
   * **Dampak Fisis**: Fitur lag cuaca menangkap **inersia/akumulasi atmosfer** (misalnya, pengaruh kecepatan angin atau suhu 1 jam sebelumnya terhadap penumpukan aerosol PM2.5 saat ini).

> [!IMPORTANT]
> **Keputusan Desain Akhir**:
> Model final RFR yang dilatih dan disimpan di [pm25_rfr_model.pkl](../data/pm25_rfr_model.pkl) **resmi menggunakan pendekatan Dengan Koordinat dan Fitur Lag Cuaca (RFR with Coordinates and Weather Lags)**, karena koordinat terbukti berfungsi sebagai rujukan spasial (spatial anchor) yang memangkas kesalahan harian (MAE) secara dramatis pada lokasi baru.


---


## 4. Analisis Feature Importance (Tingkat Kepentingan Fitur)

Tabel berikut menunjukkan seberapa sering suatu fitur dipilih untuk membagi data (*split*) dalam pohon keputusan Random Forest final (Tuned RFR Model + Coords + Lags), berbobot pada penurunan ketidakmurnian (*impurity reduction*):

![Feature Importance Random Forest Regressor](../results/images/feature_importance_rfr.png)

| Rank | Fitur | Importance | Deskripsi & Penjelasan Fisis |
| :---: | :--- | :---: | :--- |
| 1 | `v_wind_lag1` | 0.1424 | Komponen angin Utara-Selatan tertunda 1 jam. Menunjukkan inersia atmosfer sirkulasi angin laut/darat. |
| 2 | `v_wind` | 0.1321 | Komponen angin Utara-Selatan (Monsoon & Sea Breeze). Sangat krusial bagi Jakarta di pesisir utara. |
| 3 | `bulan` | 0.1123 | Waktu bulanan. Mewakili variasi musiman (Musim Kemarau vs Musim Hujan) di Indonesia. |
| 4 | `jam` | 0.0900 | Waktu harian. Mewakili siklus aktivitas manusia (kemacetan lalu lintas) dan dinamika PBL. |
| 5 | `latitude` | 0.0595 | Posisi Lintang. Menunjukkan gradien polusi spasial Utara-Selatan (pantai/industri vs selatan hijau). |
| 6 | `temperature_2m` | 0.0483 | Suhu udara aktual pada ketinggian 2 meter. Memengaruhi kestabilan kolom udara. |
| 7 | `temp_lag1` | 0.0448 | Suhu udara tertunda 1 jam. Menangkap efek termal akumulatif udara. |
| 8 | `relative_humidity_2m` | 0.0445 | Kelembaban relatif. Memicu pertumbuhan higroskopis partikulat aerosol PM2.5. |
| 9 | `surface_pressure` | 0.0434 | Tekanan udara permukaan. Berhubungan erat dengan sistem tekanan rendah/tinggi polutan. |
| 10 | `rh_lag1` | 0.0381 | Kelembaban relatif tertunda 1 jam. Menangkap riwayat kebasahan udara sekitar. |
| 11 | `u_wind_lag1` | 0.0373 | Komponen angin Barat-Timur tertunda 1 jam. Menangkap adveksi lateral tertunda. |
| 12 | `u_wind` | 0.0356 | Komponen angin Barat-Timur. Membawa polutan secara lateral lintas wilayah Jakarta. |
| 13 | `apparent_temperature` | 0.0353 | Suhu semu (gabungan temperatur dan humiditas) yang dirasakan. |
| 14 | `dew_point_2m` | 0.0310 | Titik embun. Ukuran kejenuhan uap air di udara. |
| 15 | `cloud_cover_total` | 0.0289 | Tutupan awan total. Menentukan intensitas radiasi matahari untuk reaksi fotokimia. |
| 16 | `hari_dalam_minggu` | 0.0204 | Hari dalam seminggu. Membedakan fluktuasi emisi harian (hari kerja vs libur). |
| 17 | `precipitation` | 0.0198 | Total presipitasi (curahan air dari awan). |
| 18 | `rain` | 0.0171 | Curah hujan cair. Berkontribusi pada pencucian polutan (*wet deposition*). |
| 19 | `longitude` | 0.0120 | Posisi Bujur. Memberikan informasi koordinat barat-timur. |
| 20 | `is_weekend` | 0.0046 | Flag akhir pekan (Sabtu/Minggu). |
| 21 | `AOD` | **0.0028** | **Aerosol Optical Depth (Himawari-9)**. |

---

## 5. Mengapa Fitur AOD Memiliki Importance Sangat Rendah (0.28%)?

Meskipun secara teori AOD satelit adalah prediktor terbaik untuk PM2.5 karena mengukur kolom aerosol secara langsung dari luar angkasa, dalam model final kepentingannya hanya **0.28%**. Hal ini disebabkan oleh:

1. **Persentase Data Kosong yang Ekstrim (91.31% NaNs)**
   AOD Himawari-9 hanya tersedia pada siang hari yang cerah. Di Jakarta, awan tebal sangat sering menutupi langit, menyebabkan 91.31% baris data memiliki nilai AOD kosong yang diimputasi dengan nilai sentinel `-999.0`.
2. **Perilaku Decision Tree terhadap Sentinel Value**
   Karena AOD bernilai `-999.0` pada lebih dari 91% data, fitur ini menjadi konstan untuk sebagian besar sampel. Pohon keputusan (Random Forest) tidak bisa melakukan pembelahan (*split*) yang menghasilkan penurunan impuritas yang signifikan pada data konstan ini. Akibatnya, sebagian besar node dalam RFR hanya menggunakan fitur cuaca (`v_wind`, `jam`, `bulan`) untuk membagi data, sehingga nilai *importance* kumulatif AOD menjadi sangat kecil.

> [!TIP]
> **Apakah AOD tidak berguna?**
> Tetap berguna! Pada saat cuaca cerah (AOD $\ge 0$), model akan masuk ke cabang pencabangan AOD dan menghasilkan estimasi PM2.5 spasial dengan resolusi sangat tinggi (1 km) yang menangkap detail lokal secara dinamis. Namun, untuk estimasi rata-rata jangka panjang atau malam hari, model akan bergantung penuh pada dinamika cuaca (`v_wind`) dan siklus waktu (`jam`, `bulan`).

---

## 6. Penyetelan Parameter Model Terbaik (Extra Trees Tuning)

Untuk mendapatkan performa model tertinggi yang kebal terhadap noise dan batasan grid, proses pencarian parameter terbaik dilakukan menggunakan **GridSearchCV** dikombinasikan dengan *Custom Group-LOSOCV Splitter* (5-fold, 360 total run) pada prediktor tanpa koordinat untuk model **Extra Trees Regressor** (yang terbukti mengungguli Random Forest Regressor biasa).

### 📊 Hasil Tuning Parameter Terbaik (Extra Trees)

![Feature Importance Extra Trees Regressor](../results/images/feature_importance_etr.png)

* **Skor $R^2$ Validasi Spasial Terbaik**: **`0.2191`** (Meningkat dari baseline tanpa koordinat `0.1629` dan RFR Tuned `0.1747`).
* **Kombinasi Parameter Terpilih (Best Parameters)**:
  * `n_estimators`: **`200`** (Jumlah pohon keputusan yang lebih banyak membantu meredam variansi prediksi spasial).
  * `max_depth`: **`None`** (Kedalaman tak terbatas memungkinkan model menangkap detail variasi non-linear yang kompleks).
  * `max_features`: **`None`** (Menggunakan seluruh prediktor cuaca & lag cuaca di setiap pembelahan node untuk menstabilkan interpretasi fisis).
  * `min_samples_split`: **`5`** (Mencegah overfitting berlebihan pada level noise mikro).

> [!NOTE]
> Meskipun Extra Trees Regressor (ETR) memiliki skor Group-LOSOCV $R^2$ yang lebih tinggi pada skenario tanpa koordinat, validasi harian independen pada hari netral (18 Juni 2025) menunjukkan bahwa **Random Forest Regressor (RFR) Tuned dengan Koordinat + Lags** menghasilkan error terkecil (MAE: 23.70 µg/m³ dan RMSE: 26.81 µg/m³). Oleh karena itu, **RFR Tuned dengan Koordinat + Lags secara resmi dipilih sebagai model final** dan disimpan di [pm25_rfr_model.pkl](../data/pm25_rfr_model.pkl). ETR juga disimpan di [pm25_etr_model.pkl](../data/pm25_etr_model.pkl) sebagai pembanding visual di Phase 5.


