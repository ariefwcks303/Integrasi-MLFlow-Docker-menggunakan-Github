# Automated MLOps Pipeline: Integrasi MLflow, Docker, dan GitHub Actions untuk Deployment Credit Scoring ke GCP


![Python](https://img.shields.io/badge/python-3670A0?style=for-the-badge&logo=python&logoColor=ffdd54) ![MLflow](https://img.shields.io/badge/mlflow-%23d94111?style=for-the-badge&logo=mlflow&logoColor=white) ![Docker](https://img.shields.io/badge/docker-%230db7ed.svg?style=for-the-badge&logo=docker&logoColor=white) ![GitHub Actions](https://img.shields.io/badge/github%20actions-%232088FF.svg?style=for-the-badge&logo=githubactions&logoColor=white) ![GCP](https://img.shields.io/badge/Google%20Cloud-%234285F4.svg?style=for-the-badge&logo=google-cloud&logoColor=white)

Proyek ini mendemonstrasikan implementasi siklus **MLOps (Machine Learning Operations)** ujung-ke-ujung (*end-to-end*). Sistem ini mengotomatiskan proses dari pembaruan kode di repositori lokal, pelacakan eksperimen model, manajemen kontainer, hingga penyediaan layanan prediksi (*model serving*) sebagai REST API publik di infrastruktur **Google Cloud Platform (GCP)** menggunakan metode **CI/CD (Continuous Integration & Continuous Deployment)**.

---

## 🗺️ Arsitektur Sistem (System Architecture)

Sistem ini dirancang agar data inferensi dari klien dapat diproses secara real-time oleh model di peladen cloud melalui alur kerja berikut:

```
[Laptop Lokal / Terminal Git Bash] 
         │ (Mengirim data nasabah dalam format JSON via Port 8080)
         ▼
[GCP VPC Firewall Rule: allow-mlflow-8080]
         │ (Meloloskan lalu lintas data dari jaringan publik)
         ▼
[Compute Engine VM Instance (RAM 8GB Custom)]
         │ (Menyediakan lingkungan komputasi berbasis Linux)
         ▼
[Docker Container (Container-Optimized OS)]
         │ (Menjalankan server REST API terisolasi secara portable)
         ▼
[MLflow Model Serving API] ──> (Memvalidasi skema data & mengembalikan hasil prediksi)
```

---

## 🛠️ Detail Implementasi Komponen Teknis

### 1. Otomasi CI/CD dengan GitHub Actions (`main.yml`)
Setiap kali ada pembaruan pada kode sumber model atau skrip prapemrosesan, alur kerja otomasi di `.github/workflows/main.yml` akan terpicu secara otomatis. Pipa (*pipeline*) ini menjalankan beberapa instruksi penting:
* Melakukan pemeriksaan lingkungan kerja (*environment checking*).
* Menggunakan skrip Python internal untuk memanggil API MLflow guna mendapatkan `RUN_ID` eksperimen terbaru secara dinamis (`mlflow.search_runs`).
* Membangun Docker Image model secara otomatis via perintah `mlflow models build-docker`.
* Menggunakan *GitHub Secrets* (`DOCKER_HUB_USERNAME` dan `DOCKER_HUB_ACCESS_TOKEN`) untuk proses otentikasi aman, melakukan *tagging*, lalu mendorong (*push*) *image* teranyar ke pusat registri.

> **Bukti Workflow Sukses:**
> ![GitHub Actions Build Success](Asset/6.%20Build%20Docker%20Image.png)

---

### 2. Manajemen Registri Kontainer (Docker Hub)
Untuk memastikan model dapat berjalan di server mana pun tanpa kendala perbedaan versi pustaka (*library dependency error*), model dikemas secara terisolasi ke dalam unit kontainer dengan nama repositori `ariefwcksdevs/cc:latest`. 
* **Portabilitas Tinggi:** Berbasis arsitektur `linux/amd64` dengan ukuran terkompresi sebesar 1.54 GB, siap ditarik (*pull*) kapan saja oleh peladen produksi.

> **Bukti Registri Gambar:**
> ![Docker Hub Registry](Asset/7.%20Docker%20Image.png)

---

### 3. Penyediaan Infrastruktur Cloud (GCP Compute Engine)
Kontainer model dideploy menggunakan Virtual Machine (VM) di Google Cloud Platform regional Jakarta (`asia-southeast2-a`).
* **Optimasi Memori (Troubleshooting OOM):** Untuk mencegah eror *Out of Memory* (OOM) yang sempat menghentikan kontainer saat inisialisasi awal, spesifikasi mesin VM ditingkatkan secara kustom menggunakan alokasi **RAM 8 GB**.
* **Keamanan Jaringan (Networking & Firewall):** Aturan kustom `allow-mlflow-8080` diterapkan pada protokol `tcp:8080` dengan target pemetaan jaringan luar `0.0.0.0/0` agar REST API dapat diakses secara publik.

> **Spesifikasi VM & Aturan Jaringan:**
> ![GCP VM Running](Asset/1.%20GCP%20VM%20Instance.png)
> ![GCP Firewall Configuration](Asset/2.%20GCP%20FireWallConfiguration.png)

* **Observabilitas Sistem (Observability):** Grafik pemantauan (*monitoring graphs*) menunjukkan metrik kesehatan server seperti *CPU Utilization*, *Network Traffic*, dan *Disk Throughput* yang bekerja stabil dan merekam lonjakan kecil yang wajar saat memproses data inferensi secara simultan.

> **Metrik Kinerja Peladen:**
> ![GCP VM Metrics](Asset/4.%20GCP%20VM%20Instance%20Log.png)

---

## 🧪 Validasi Pengujian REST API (Inference Verification)

Server MLflow menerapkan aturan ketat skema data (*Schema Enforcement*). Input data wajib mengirimkan format tipe data yang sesuai dengan dataset pelatihan, yang mencakup 11 parameter spesifik (fitur demografi nasabah serta komponen reduksi dimensi PCA): `Age`, `Credit_Mix`, `Payment_of_Min_Amount`, `Payment_Behaviour`, `pc1_1`, `pc1_2`, `pc1_3`, `pc1_4`, `pc1_5`, `pc2_1`, dan `pc2_2`.

Pengujian REST API dilakukan langsung dari terminal Git Bash lokal dengan menembak IP Publik server GCP (`http://34.101.107.71:8080/invocations`).

### Hasil Validasi Terminal:
Peladen berhasil memproses berbagai bentuk skenario beban data, mulai dari pengujian profil tunggal hingga pengujian massal secara paralel (*Batch Inference*):

![Terminal Testing Log](Asset/9.%20Testing%20GitBash%201-2-10%20Sample.png)

1. **Skenario 1 Nasabah (Single Request):**
   * Sukses mengembalikan satu hasil evaluasi risiko kredit. Output: `{"predictions": [2]}`
2. **Skenario 2 Nasabah Simultan:**
   * Sukses mengembalikan array evaluasi berurutan. Output: `{"predictions": [1, 1]}`
3. **Skenario 10 Nasabah Sekaligus (Batch Inference):**
   * Menembak 10 baris matriks array data dummy nasabah yang berbeda dalam satu ketukan perintah. Server memprosesnya secara paralel dalam hitungan milidetik dan mengembalikan respon utuh: `{"predictions": [0, 2, 1, 0, 1, 2, 0, 2, 1, 1]}`

---
---

## 📈 Kesimpulan & Pengembangan ke Depan (Future Roadmap)

### Hasil Kerja Saat ini
Proyek ini membuktikan keberhasilan rancangan arsitektur sistem ML (*Machine Learning Operations*) yang andal di tingkat *back-end* dan infrastruktur awan. Penggunaan otomatisasi **CI/CD via GitHub Actions**, isolasi lingkungan menggunakan **Docker**, serta manajemen server **GCP Compute Engine** terbukti mampu menyajikan model *Machine Learning* lokal menuju fase produksi berskala luas menjadi REST API yang stabil.

### 🔮 Rencana Pengembangan Selanjutnya (Next Improvements)
Sistem ini dirancang dengan arsitektur terbuka, sehingga sangat siap untuk diintegrasikan dengan komponen antarmuka di masa mendatang. Rencana pengembangan berikutnya meliputi:
1. **Integrasi Antarmuka Pengguna (UI/UX Interface):** Proyek ini masih dalam tahap pengembangan aktif untuk menyajikan visualisasi prediksi yang lebih interaktif dan ramah pengguna (*user-friendly*). Saat ini sistem berfokus penuh pada keandalan pipa data (*data pipeline*) dan API backend.
2. **Kolaborasi Pengembangan Frontend:** Terbuka untuk pengembangan bersama (kolaborasi) menggunakan framework antarmuka seperti **Streamlit**, **Gradio**, atau aplikasi web modern guna menjembatani hasil prediksi data JSON agar dapat dikonsumsi dengan mudah oleh tim bisnis maupun pengguna awam.
Proyek ini membuktikan keberhasilan rancangan arsitektur sistem ML yang andal. Penggunaan otomatisasi **CI/CD**, isolasi lingkungan menggunakan **Docker**, serta manajemen server **GCP** yang terukur terbukti mampu menyajikan model *Machine Learning* lokal menuju fase produksi berskala luas (*scalable production-ready API*).
