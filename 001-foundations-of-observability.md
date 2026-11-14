# 001 — Foundations of Observability

## Daftar Isi

1. [Evolusi Arsitektur Aplikasi & DevOps](#1-evolusi-arsitektur-aplikasi--devops)
2. [Monitoring & 3 Pertanyaan Utama](#2-monitoring--3-pertanyaan-utama)
3. [Metrik Keberhasilan DevOps: MTTD & MTTR](#3-metrik-keberhasilan-devops-mttd--mttr)
4. [Metode-Metode Monitoring](#4-metode-metode-monitoring)
5. [Observability vs. Monitoring](#5-observability-vs-monitoring)
6. [Metode Pengumpulan Metrik: Push vs. Scrape](#6-metode-pengumpulan-metrik-push-vs-scrape)
7. [Tipe Telemetry Data (MELT)](#7-tipe-telemetry-data-melt)

---

## 1. Evolusi Arsitektur Aplikasi & DevOps

### A. Arsitektur Monolitik vs. Mikroservis

#### Arsitektur Monolitik

![Monolithic Architecture](image/001-monolithic-architecture.png)

- **Penjelasan:** Semua bagian aplikasi (tampilan, logika bisnis, dan akses data) digabung menjadi satu aplikasi besar.
- **Analogi — Kue Lapis Raksasa:** Kalau lapisan paling bawah gosong, kamu tidak bisa mengganti lapisan itu saja; seluruh kue harus dibuat ulang. Begitu pula monolitik: mengubah satu fitur kecil berarti seluruh aplikasi harus di-*deploy* ulang.

#### Arsitektur Mikroservis

![Microservice Architecture](image/002-microservice-architecture.png)

- **Penjelasan:** Aplikasi dipecah menjadi layanan kecil yang berdiri sendiri (login, pembayaran, keranjang), masing-masing dengan database sendiri.
- **Analogi — Balok LEGO:** Kalau satu balok patah, cukup cabut dan ganti balok itu tanpa membongkar seluruh bangunan. Tim juga bisa mengerjakan balok yang berbeda secara bersamaan.

### B. Sudut Pandang DevOps: Waterfall vs. CI/CD

| | Waterfall | CI/CD |
|---|---|---|
| **Cocok dengan** | Monolitik | Mikroservis |
| **Analogi** | Membangun rumah dari nol: desain, fondasi, tembok, atap harus 100% selesai sebelum dihuni | Warung kopi yang terus berinovasi: racik menu baru, jual, minta feedback, perbaiki besok |
| **Ritme** | Besar dan jarang | Kecil, sering, dan cepat |

### C. Mengapa Mikroservis & CI/CD Butuh Observability?

Arsitektur modern ibarat kota yang sangat sibuk:

1. **Banyak layanan yang harus diawasi** — seperti ratusan toko dalam satu kota.
2. **Komunikasi antar-layanan bisa putus** — kurir antar-layanan bisa tersesat atau macet.
3. **Lebih rentan serangan** — banyak pintu masuk yang harus dijaga.
4. **Perubahan kode terjadi setiap saat** — renovasi setiap hari, risiko error baru lebih tinggi.

---

## 2. Monitoring & 3 Pertanyaan Utama

**Monitoring** adalah kegiatan mengumpulkan dan memantau data kesehatan sistem secara rutin. Data ini disebut **Telemetry Data**, ibarat hasil cek laboratorium dokter untuk mencari letak penyakit.

Untuk mengecek kesehatan sistem, ajukan **3 pertanyaan utama**:

1. Apakah layanannya **menyala**?
2. Apakah layanannya **berfungsi** sesuai tugasnya?
3. Apakah layanannya **bekerja dengan baik** (cepat dan sehat)?

### Analogi: Dashboard Mobil

![Car Dashboard](image/003-car-dashboard.png)

| Pertanyaan | Indikator di Mobil |
|---|---|
| Menyala? | Jarum RPM di atas 0 (mesin hidup) |
| Berfungsi? | Mobil melaju 50 km/jam (tugas utamanya terpenuhi) |
| Bekerja dengan baik? | Belum tentu — lampu *Check Engine* menyala, ada masalah tersembunyi meski mobil masih jalan |

### Contoh Nyata: Website *Save the Koala*

![Save the Koala Website](image/004-save-the-koala-website.png)

| Pertanyaan | Cara Cek |
|---|---|
| Menyala? | Kirim HTTP `GET`/`HEAD` ke server. Jika membalas `200 OK`, website menyala. |
| Berfungsi? | Cek jumlah error di log (misal error Python). Di bawah batas wajar (misal < 5 error/menit) berarti normal. |
| Bekerja dengan baik? | Ukur waktu respons. Di bawah 20 ms berarti performanya prima. |

---

## 3. Metrik Keberhasilan DevOps: MTTD & MTTR

Kedua metrik ini mengukur kecepatan tim dalam **mendeteksi** dan **memulihkan** masalah sistem.

### A. Mean Time to Detection (MTTD)

- **Definisi:** Rata-rata waktu sejak sistem mulai rusak hingga tim menyadarinya.
- **Analogi — Alarm Kebakaran:** Asap muncul pukul 14.00, alarm baru berbunyi pukul 14.15. Jeda 15 menit itulah MTTD.

### B. Mean Time to Resolve (MTTR)

- **Definisi:** Rata-rata waktu sejak masalah terdeteksi hingga sistem pulih normal.
- **Analogi — Pemadam Kebakaran:** Dihitung dari penghuni sadar ada kebakaran (14.15) hingga api padam dan dapur aman (14.45).

### Contoh Timeline Insiden

| Waktu | Kejadian |
|---|---|
| 14.00 | Fitur pembayaran error karena gangguan database |
| 14.15 | Sistem monitoring mengirim *alert* ke tim |
| 14.45 | Database diperbaiki, pembayaran normal kembali |

- **MTTD = 15 menit** (14.00 → 14.15)
- **MTTR = 30 menit** (14.15 → 14.45)

> Nilai MTTD/MTTR tim adalah rata-rata dari semua insiden dalam satu periode (misal sebulan).

---

## 4. Metode-Metode Monitoring

![Monitoring Methods by Layer](image/005-monitoring-methods-by-layer.png)

Setiap lapisan sistem punya metode yang paling cocok:

| Lapisan | Metode |
|---|---|
| UI Layer | Core Web Vitals |
| Service Layer | RED Method, Four Golden Signals |
| Infrastructure Layer | USE Method, Four Golden Signals |

### A. RED Method — Fokus pada Layanan

*Analogi: mengamati kasir di restoran cepat saji.*

1. **Rate** — Berapa pelanggan yang datang per menit? *(request per detik)*
2. **Errors** — Berapa pesanan yang gagal atau salah? *(misal HTTP 500)*
3. **Duration** — Berapa lama pelanggan menunggu pesanan? *(response time / latency)*

### B. USE Method — Fokus pada Infrastruktur

*Analogi: mengecek kondisi koki dan peralatan dapur.*

1. **Utilization** — Seberapa sibuk koki dan kompor? *(misal CPU 85%, disk 90%)*
2. **Saturation** — Seberapa panjang tumpukan nota yang belum dikerjakan? *(antrean CPU/jaringan; 0 berarti lancar)*
3. **Errors** — Adakah alat dapur yang rusak? *(misal gagal menulis ke disk)*

### C. Four Golden Signals — Standar Google SRE

1. **Latency** — Seberapa lama pengguna menunggu?
2. **Traffic** — Seberapa ramai permintaan yang masuk?
3. **Errors** — Seberapa sering sistem gagal?
4. **Saturation** — Seberapa dekat sistem ke batas kapasitas maksimal?

### D. Core Web Vitals — Fokus pada Pengalaman Pengguna Web

*Analogi: kesan pertama tamu saat masuk toko.*

1. **LCP (Largest Contentful Paint)** — Seberapa cepat pajangan utama terlihat? *(kecepatan elemen terbesar muncul)*
2. **FID (First Input Delay)** — Saat tamu menekan bel, seberapa cepat pelayan merespons? *(respons klik pertama)*
3. **CLS (Cumulative Layout Shift)** — Apakah rak tiba-tiba bergeser saat tamu berjalan? *(kestabilan tata letak saat halaman dimuat)*

> **Catatan:** Sejak Maret 2024, Google mengganti **FID** dengan **INP (Interaction to Next Paint)**, yang mengukur respons terhadap *semua* interaksi, bukan hanya yang pertama.

---

## 5. Observability vs. Monitoring

### Apa itu Observability?

**Observability (keteramatan)** adalah kemampuan memahami kondisi internal sistem hanya dari data luar yang dihasilkannya (telemetri).

- **Monitoring** seperti **lampu indikator bensin**: memberi tahu satu hal yang sudah kamu tentukan sejak awal — *"bensinmu mau habis."*
- **Observability** seperti **bengkel dengan komputer diagnostik lengkap**: saat mobil mogok secara misterius, mekanik bisa melacak listrik, tekanan bahan bakar, dan suhu mesin sekaligus untuk menemukan *mengapa* dan *di mana* kerusakannya — bahkan untuk masalah yang belum pernah terjadi (*unknown unknowns*).

### Perbandingan

| Aspek | Monitoring | Observability |
|---|---|---|
| **Fokus** | *Apa* dan *kapan* | *Mengapa* dan *di mana* |
| **Cara kerja** | Tentukan dulu apa yang diawasi (misal batas CPU 80°C) | Kumpulkan metrik, log, dan trace secara menyeluruh lalu satukan (*holistic view*) |
| **Kelemahan / Kelebihan** | Komponen yang lupa dipantau tidak akan memberi tanda; aturan manual berlebih memicu **tool sprawl** | Mampu menangani **unknown unknowns** |
| **Cocok untuk** | Monolitik, titik rawannya mudah ditebak | Sistem terdistribusi seperti mikroservis |

---

## 6. Metode Pengumpulan Metrik: Push vs. Scrape

Ada dua cara sistem pengumpul data mendapatkan informasi dari aplikasi.

### A. Push Method (Dorong)

- **Penjelasan:** Aplikasi **aktif mengirim** metrik ke server pusat (*central endpoint*) lewat TCP, UDP, atau HTTP.
- **Analogi — Laporan Harian Pegawai:** Karyawan mengirim laporan ke email bos setiap sore tanpa diminta.
- **Contoh:** Aplikasi mengirim metrik ke **StatsD**, lalu disimpan di **Graphite**.

### B. Scrape Method (Tarik)

- **Penjelasan:** Aplikasi **menyediakan endpoint** (biasanya `/metrics`), lalu server monitoring **datang secara berkala** untuk menarik datanya.
- **Analogi — Inspeksi Kebersihan Kelas:** Guru piket berkeliling memeriksa setiap meja pada jam tertentu, tanpa menunggu murid melapor.
- **Contoh:** **Prometheus** men-*scrape* endpoint metrik aplikasi.

### C. Memilih Push atau Scrape?

#### 1. Tipe Sistem & Aplikasi

*Apakah aplikasi hidup terus di satu tempat, atau hanya sebentar lalu mati?*

- **Scrape — Pegawai Tetap di Kubikel:** Mejanya jelas dan tidak pindah, jadi bos bisa mendatanginya kapan saja. Cocok untuk **long-running services** dengan alamat yang jelas.
- **Push — Kurir Paket Kilat:** Selesai tugas langsung pulang dan HP mati. Kalau tidak melapor sebelum pulang, kantor tidak akan tahu hasil kerjanya. Cocok untuk aplikasi **short-lived/ephemeral** seperti *serverless functions* atau *batch jobs* yang hanya hidup beberapa detik.

#### 2. Skalabilitas

*Apa yang terjadi jika aplikasi membengkak dari 10 menjadi 10.000?*

- **Scrape — Petugas Sensus Keliling:** 10 rumah mudah, 10.000 rumah bikin kewalahan. Server pusat memegang kendali jadwal, tapi seluruh beban ada di sisinya.
- **Push — Grup WhatsApp Keluarga yang Heboh:** 10.000 anggota mengirim chat ke HP Ayah di detik yang sama, HP langsung *lag* atau *down* (*traffic jam* di server pusat).

#### 3. Kompleksitas & Deteksi Layanan Mati

*Bagaimana tahu jika aplikasi tiba-tiba mati?*

- **Scrape — Guru Mengabsen di Kelas:** Nama Budi dipanggil, tidak menjawab, guru **langsung tahu** Budi tidak masuk. Jika scrape gagal, sistem langsung tahu layanan *down*.
- **Push — Murid Mengirim Surat Izin:** Budi pingsan di jalan dan tidak sempat mengirim surat. Guru yang hanya menunggu (*pasif*) tidak tahu Budi bermasalah. Push butuh trik tambahan seperti **heartbeat** untuk membedakan "mati" dari "telat kirim". Selain itu, aplikasi harus tahu alamat server pusat, dan data bisa hilang jika jaringan putus.

### Ringkasan

| Pertimbangan | Scrape | Push |
|---|---|---|
| Tipe aplikasi | Long-running, alamat tetap | Short-lived, ephemeral |
| Skalabilitas | Beban di server pemantau | Risiko lonjakan trafik ke server pusat |
| Deteksi layanan mati | Langsung terdeteksi | Butuh heartbeat |

---

## 7. Tipe Telemetry Data (MELT)

**MELT** adalah 4 jenis data telemetri utama: **Metric, Event, Log, Trace**. Contoh di bawah memakai analogi transaksi pembelian keripik di sebuah toko.

### A. Event (Kejadian)

- **Definisi:** Catatan bahwa **satu tindakan spesifik terjadi pada waktu tertentu**.
- **Fungsi:** Membuktikan aksi benar-benar terjadi (tombol diklik, data tersimpan). Sering ditemui di platform seperti *Kafka*.
- **Analogi — Struk Belanja:** *"1 kantong keripik dibeli pukul 15:20."* Kejadiannya diskrit, terjadi sekali.

### B. Metric (Metrik)

- **Definisi:** **Angka hasil agregasi** dari sekumpulan event dalam kurun waktu tertentu.
- **Fungsi:** Melihat tren dan membandingkan performa dengan masa lalu.
- **Analogi — Grafik Penjualan:** *"Rata-rata 100 bungkus terjual per menit hari ini"* atau *"CPU naik ke 80% jam 12 siang."*

### C. Log (Catatan Rinci)

- **Definisi:** **Rekaman teks detail** tentang sebuah event beserta konteksnya.
- **Fungsi:** Barang bukti utama saat menyelidiki error.
- **Analogi — Nota Super Lengkap:**

  ```text
  Barang       : Keripik Kentang
  Harga        : Rp10.000
  Waktu        : 15:20:05.123 WIB
  ID Kasir     : VEND-SURABAYA-04
  Pembayaran   : QRIS
  Status       : Success (200)
  ```

### D. Trace (Jejak Perjalanan)

- **Definisi:** Rekaman **perjalanan satu request lintas layanan** dari awal sampai akhir.
- **Fungsi:** Menunjukkan bagian mana yang membuat proses lambat atau gagal.
- **Analogi — Lacak Resi Pesanan Online:**

  1. Tekan "Bayar" → **API Gateway** (10 ms)
  2. Potong saldo → **Payment Service** (200 ms)
  3. Cetak pesanan di dapur → **Inventory Service** (50 ms)
  4. Kirim notifikasi → **Notification Service**

  Jika proses macet 5 detik, trace langsung menunjukkan titik macetnya (misal di *Payment Service*).

### Ringkasan MELT

| Tipe | Pertanyaan yang Dijawab | Analogi |
|---|---|---|
| Metric | Bagaimana trennya? | Grafik penjualan |
| Event | Apa yang terjadi? | Struk belanja |
| Log | Apa detailnya? | Nota lengkap |
| Trace | Di mana lambat/gagalnya? | Lacak resi |
