# 002 — Prometheus

## Daftar Isi

1. [Instalasi Prometheus di Ubuntu](#1-instalasi-prometheus-di-ubuntu)
2. [Cara Mengumpulkan Metrik](#2-cara-mengumpulkan-metrik)
3. [Node Exporter](#3-node-exporter)
4. [Praktik: Memantau App Server dengan Node Exporter](#4-praktik-memantau-app-server-dengan-node-exporter)
5. [Data Model](#5-data-model)
6. [Tipe Data PromQL](#6-tipe-data-promql)
7. [Operator Aritmatika](#7-operator-aritmatika)
8. [Operator Perbandingan](#8-operator-perbandingan)
9. [Operator Himpunan (Set)](#9-operator-himpunan-set)
10. [Label Selectors](#10-label-selectors)
11. [Operator Agregasi](#11-operator-agregasi)
12. [Time Offset](#12-time-offset)
13. [Fungsi-Fungsi PromQL](#13-fungsi-fungsi-promql)
14. [Aggregation Over Time](#14-aggregation-over-time)

---

## 1. Instalasi Prometheus di Ubuntu

Langkah instalasi **Prometheus v3.x** di server Linux (Ubuntu) sebagai layanan *systemd*.

### Langkah 1 — Persiapan & Pembaruan Sistem

Cek arsitektur prosesor agar tidak salah mengunduh file rilis.

```bash
# x86_64 = amd64, aarch64 = arm64
uname -m

sudo apt update && sudo apt upgrade -y
sudo apt install wget -y
```

### Langkah 2 — Membuat User & Direktori

Buat *service account* tanpa akses login, serta direktori data dan konfigurasi.

```bash
sudo groupadd --system prometheus
sudo useradd -s /sbin/nologin --system -g prometheus prometheus

sudo mkdir -p /var/lib/prometheus
sudo mkdir -p /etc/prometheus/rules
sudo mkdir -p /etc/prometheus/rules.d
sudo mkdir -p /etc/prometheus/files_sd
```

### Langkah 3 — Unduh, Ekstrak, dan Pindahkan Biner

```bash
wget https://github.com/prometheus/prometheus/releases/download/v3.16.0/prometheus-3.16.0.linux-amd64.tar.gz
tar xvf prometheus-3.16.0.linux-amd64.tar.gz
cd prometheus-3.16.0.linux-amd64

# Pindahkan biner ke PATH sistem
sudo mv prometheus promtool /usr/local/bin/

# Salin konfigurasi utama
sudo cp prometheus.yml /etc/prometheus/

# Verifikasi
prometheus --version
```

### Langkah 4 — Atur Kepemilikan & Hak Akses

```bash
sudo chown -R prometheus:prometheus /etc/prometheus
sudo chmod -R 775 /etc/prometheus
sudo chown -R prometheus:prometheus /var/lib/prometheus
```

### Langkah 5 — Konfigurasi Layanan Systemd

Agar Prometheus berjalan di *background* dan otomatis menyala saat *reboot*.

```bash
sudo nano /etc/systemd/system/prometheus.service
```

```ini
[Unit]
Description=Prometheus
Documentation=https://prometheus.io/docs/introduction/overview/
Wants=network-online.target
After=network-online.target

[Service]
Type=simple
User=prometheus
Group=prometheus
ExecReload=/bin/kill -HUP $MAINPID
ExecStart=/usr/local/bin/prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/var/lib/prometheus/ \
  --web.listen-address=0.0.0.0:9090 \
  --web.external-url=

SyslogIdentifier=prometheus
Restart=always

[Install]
WantedBy=multi-user.target
```

Simpan dengan `Ctrl + O` → `Enter`, keluar dengan `Ctrl + X`.

> **Catatan Prometheus 3.x:** Flag `--web.console.templates` dan `--web.console.libraries` sudah tidak dipakai karena aset konsol web sudah tertanam di dalam biner.

### Langkah 6 — Jalankan & Verifikasi

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now prometheus
sudo systemctl status prometheus
```

### Langkah 7 — Akses Web UI

Setelah status **`active (running)`**, buka:

```text
http://<IP_SERVER>:9090
```

> Pastikan port **9090/TCP** diizinkan di firewall (UFW) maupun Security Group cloud provider.

---

## 2. Cara Mengumpulkan Metrik

![Prometheus Data Collection](image/006-prometheus-data-collection.png)

Setelah Prometheus terinstal, ada **3 skenario utama** untuk membawa metrik masuk.

### Skenario 1 — Punya Source Code Aplikasi → *Client Library*

- **Penjelasan:** Untuk aplikasi buatan sendiri (Python, Go, Java, Ruby, .NET), sisipkan **library resmi Prometheus** ke kode. Library ini membuka endpoint `/metrics` untuk di-*scrape*.
- **Analogi — Mobil Rakitan Sendiri:** Karena kamu yang merakit, kamu bebas memasang sensor suhu dan kecepatan langsung di mesinnya.

### Skenario 2 — Tidak Punya Source Code → *Exporter*

- **Masalah:** Sistem pihak ketiga yang kodenya tidak bisa diubah, misalnya:
  - Database: **MySQL**, **PostgreSQL**
  - Load balancer: **HAProxy**
  - Cloud: **AWS CloudWatch**
  - Sistem operasi: **Linux / Windows Server**
  - Perangkat **IoT** (sensor lampu lalu lintas, sensor suhu)
- **Cara yang buruk:** Skrip manual (Bash/PowerShell + Cron Job) yang mendorong data ke Prometheus. Tidak *scalable*, dan jika jutaan perangkat IoT mendorong data bersamaan, Prometheus bisa *down*.
- **Solusi — Exporter:**
  - Program kecil yang dipasang di sebelah sistem target, membaca kondisinya, lalu **menerjemahkannya ke format Prometheus**.
  - Secara *default*, Prometheus men-*scrape* exporter **setiap 15 detik**.
- **Analogi — Penerjemah Bahasa:** Prometheus hanya mengerti Bahasa Indonesia, MySQL hanya bicara Bahasa Jepang. Kamu menyewa **penerjemah (exporter)** yang duduk di sebelah MySQL. Setiap 15 detik Prometheus bertanya *"Gimana kondisi MySQL?"*, dan penerjemah menjawab dalam Bahasa Indonesia.

### Skenario 3 — Aplikasi Berumur Singkat → *Pushgateway*

- **Masalah:** Prometheus **selalu Pull**, tidak menerima Push secara langsung. Bagaimana dengan *batch job* atau *serverless* yang hanya hidup 3 detik lalu mati?
- **Solusi — Pushgateway:** Komponen resmi Prometheus yang menjadi **tempat penitipan sementara**.
  1. Aplikasi singkat mem-*push* metriknya ke Pushgateway sebelum mati.
  2. Prometheus tetap men-*scrape* Pushgateway setiap 15 detik seperti biasa.
- **Analogi — Kotak Penitipan Paket:** Kurir kilat menaruh paket di kotak depan pagar lalu pergi. Saat pemilik rumah pulang, ia tinggal mengambilnya dari kotak.

> **Penting:** Pushgateway **tidak** mengubah Prometheus menjadi sistem Push. Prometheus tetap menarik data, hanya saja dari Pushgateway.

### Ringkasan

| Kondisi | Gunakan |
|---|---|
| Punya kode aplikasinya | **Client Library** |
| Sistem pihak ketiga / server / database | **Exporter** |
| Aplikasi kilat yang langsung mati | **Pushgateway** |

---

## 3. Node Exporter

### Apa itu Node Exporter?

- **Bukan Node.js** — tidak ada hubungannya dengan JavaScript.
- **"Node"** di Prometheus berarti mesin/server berbasis kernel Unix (Linux, Ubuntu).
- **Definisi:** *Official exporter* dari tim Prometheus untuk mengumpulkan **metrik tingkat sistem (kernel-level)** dari mesin Unix/Linux.

### Metrik yang Dikumpulkan (Default)

1. **CPU Usage** — seberapa sibuk prosesor
2. **Memory Usage** — penggunaan RAM
3. **Disk Usage** — kapasitas penyimpanan
4. **Network I/O** — lalu lintas data jaringan

Bisa diperluas dengan **pluggable metric collectors**.

### Analogi — Alat Rekam Medis Atlet

- **Server** = tubuh seorang atlet.
- **Node Exporter** = alat rekam medis yang menempel di tubuh, terus membaca detak jantung, suhu, dan tekanan darah.
- **Prometheus** = dokter yang datang setiap 15 detik untuk membaca laporan alat tersebut.

---

## 4. Praktik: Memantau App Server dengan Node Exporter

### Topologi

| Peran | IP | Port |
|---|---|---|
| Prometheus Server | `10.30.0.119` | `9090` |
| App Server (target) | `10.30.0.120` | `9100` |

### Langkah 1 — Install Node Exporter di App Server (`10.30.0.120`)

1. **Buat system user:**

   ```bash
   sudo groupadd --system node_exporter
   sudo useradd -s /sbin/nologin --system -g node_exporter node_exporter
   ```

2. **Unduh dan ekstrak:**

   ```bash
   wget https://github.com/prometheus/node_exporter/releases/download/v1.12.1/node_exporter-1.12.1.linux-amd64.tar.gz
   tar xvf node_exporter-1.12.1.linux-amd64.tar.gz
   cd node_exporter-1.12.1.linux-amd64
   ```

3. **Pindahkan biner & verifikasi:**

   ```bash
   sudo mv node_exporter /usr/local/bin/
   node_exporter --version
   ```

### Langkah 2 — Amankan Akses (Hanya untuk Prometheus)

*Bind* Node Exporter ke IP privat App Server dan batasi dengan firewall.

1. **Buat file service:**

   ```bash
   sudo nano /etc/systemd/system/node_exporter.service
   ```

2. **Isi konfigurasi** (ganti `10.30.0.120` dengan IP privat App Server):

   ```ini
   [Unit]
   Description=Node Exporter
   After=network.target

   [Service]
   User=node_exporter
   Group=node_exporter
   Type=simple
   ExecStart=/usr/local/bin/node_exporter \
     --web.listen-address=10.30.0.120:9100

   [Install]
   WantedBy=multi-user.target
   ```

3. **Jalankan layanan:**

   ```bash
   sudo systemctl daemon-reload
   sudo systemctl enable --now node_exporter
   sudo systemctl status node_exporter
   ```

4. **(Opsional) Batasi dengan UFW** agar port `9100` hanya bisa diakses Prometheus:

   ```bash
   sudo ufw allow from 10.30.0.119 to any port 9100 proto tcp
   ```

### Langkah 3 — Daftarkan App Server di Prometheus (`10.30.0.119`)

1. **Buka konfigurasi:**

   ```bash
   sudo nano /etc/prometheus/prometheus.yml
   ```

2. **Tambahkan scrape job:**

   ```yaml
   scrape_configs:
     # ... job bawaan prometheus

     - job_name: 'app-server'
       static_configs:
         - targets: ['10.30.0.120:9100']
   ```

3. **Validasi & restart:**

   ```bash
   promtool check config /etc/prometheus/prometheus.yml
   sudo systemctl restart prometheus
   ```

### Langkah 4 — Verifikasi di Web UI

1. Buka `http://10.30.0.119:9090`.
2. Klik **Status → Targets**.
3. Pastikan target `app-server` (`10.30.0.120:9100`) berstatus **UP** (hijau).

---

## 5. Data Model

- **Time series:** Setiap data metrik disimpan bersama *timestamp* (Unix time).
- **Identitas unik:** Setiap time series diidentifikasi oleh **nama metrik + kumpulan label**.
- **Label:** Pasangan *key-value* opsional, ditulis dalam kurung kurawal dan dipisah koma.

```promql
<metric_name>{<label_key>="<label_value>", ...}
```

**Contoh:**

```promql
auth_api_hit{count="1", time_taken="800"}
```

- `auth_api_hit` — nama metrik (jumlah hit API autentikasi)
- `count`, `time_taken` — label (`time_taken` dalam milidetik)

---

## 6. Tipe Data PromQL

### A. Scalar

Nilai tunggal tanpa deret waktu.

- **Float:** Angka bulat (`1`) maupun desimal (`1.5`) dianggap *float*.
- **String:** Teks dalam kutip (`"..."` atau `'...'`).

> **Contoh:** Pada `code="200"`, nilai `200` adalah **string**. Karena itu kamu bisa memakai regex seperti `code=~"2.*"`. Menulis `code=200` tanpa kutip tidak akan berhasil.

### B. Instant Vector

Kumpulan time series yang masing-masing hanya punya **satu sampel terbaru**.

```promql
node_network_transmit_errs_total
node_network_transmit_errs_total{device="eth0"}
```

Hasilnya: satu nilai terbaru untuk setiap kombinasi label unik.

### C. Range Vector

Seperti instant vector, tapi menarik **sekumpulan sampel historis** dalam rentang waktu, ditandai kurung siku `[...]`.

```promql
node_network_transmit_errs_total[5m]
```

Artinya: data 5 menit ke belakang. Jumlah sampel bergantung pada durasi dan *scrape interval* (default 15 detik → sekitar 20 sampel per 5 menit).

**Satuan waktu** (*case-sensitive*):

| Satuan | Arti |
|---|---|
| `ms` | milidetik |
| `s` | detik |
| `m` | menit |
| `h` | jam |
| `d` | hari (24 jam) |
| `w` | minggu (7 hari) |
| `y` | tahun (365 hari) |

> Tidak ada satuan bulan.

---

## 7. Operator Aritmatika

| Operator | Arti |
|---|---|
| `+` | Penjumlahan |
| `-` | Pengurangan |
| `*` | Perkalian |
| `/` | Pembagian |
| `%` | Modulo (sisa bagi) |
| `^` | Pangkat |

Hasil operasi selalu **vektor/nilai baru**; data asli di Prometheus tidak berubah.

### Aturan Penerapan

| Kombinasi | Perilaku | Contoh |
|---|---|---|
| Scalar ⊕ Scalar | Menghasilkan scalar baru | `2 + 2` → `4` |
| Vector ⊕ Scalar | Scalar diterapkan ke **setiap** elemen vektor | `550` dan `333` dikurangi `10` → `540` dan `323` |
| Vector ⊕ Vector | Elemen dicocokkan berdasarkan **set label yang persis sama**; yang tidak punya pasangan dibuang | Lihat ilustrasi di bawah |

> Pada operasi aritmatika antar-vektor, **nama metrik diabaikan** saat pencocokan dan hasilnya tidak membawa nama metrik. Yang harus sama adalah set labelnya.

### Ilustrasi Vector + Vector

**Vektor A** — `api_errors_total`:

| Label | Nilai |
|---|---|
| `{endpoint="/login"}` | 10 |
| `{endpoint="/register"}` | 5 |
| `{endpoint="/checkout"}` | 2 |

**Vektor B** — `api_retries_total`:

| Label | Nilai |
|---|---|
| `{endpoint="/login"}` | 3 |
| `{endpoint="/register"}` | 1 |

**Hasil `A + B`:**

| Label | Perhitungan | Hasil |
|---|---|---|
| `/login` | 10 + 3 | **13** ✅ |
| `/register` | 5 + 1 | **6** ✅ |
| `/checkout` | tidak ada pasangan | **dibuang** ❌ |

**Intinya:** Hanya elemen yang **berpasangan** (label klop) yang diproses; yang "jomblo" dibuang.

---

## 8. Operator Perbandingan

### Nilai Boolean

Prometheus memakai angka: **`1` = True**, **`0` = False**.

### Daftar Operator

| Operator | Arti |
|---|---|
| `==` | Sama dengan |
| `!=` | Tidak sama dengan |
| `>` | Lebih besar |
| `<` | Lebih kecil |
| `>=` | Lebih besar atau sama |
| `<=` | Lebih kecil atau sama |

### Cara Kerja

| Kombinasi | Perilaku | Contoh |
|---|---|---|
| Scalar vs Scalar | Hasil `1` atau `0` (wajib pakai `bool`) | `5 == bool 5` → `1` |
| Vector vs Scalar | **Filter**: hanya elemen yang memenuhi syarat yang muncul | Vektor bernilai `10` dan `4`, query `== 10` → hanya `10` |
| Vector vs Vector | Cocokkan pasangan berlabel sama, lalu filter yang memenuhi syarat | — |

> Tambahkan modifier `bool` (misal `metric > bool 5`) jika ingin hasil `0`/`1` alih-alih filter.

---

## 9. Operator Himpunan (Set)

### Ketentuan Umum

- **Hanya untuk instant vector** (tidak untuk scalar atau range vector).
- **Huruf kecil:** `and`, `or`, `unless`.

### Tiga Operator

| Operator | Nama | Hasil |
|---|---|---|
| `and` | Irisan (*intersection*) | Elemen kiri yang **punya pasangan label sama** di kanan (nilai diambil dari kiri) |
| `or` | Gabungan (*union*) | Semua elemen kiri, ditambah elemen kanan yang labelnya tidak ada di kiri |
| `unless` | Pengecualian (*difference*) | Elemen kiri yang **tidak punya pasangan label** di kanan |

> Pencocokan berdasarkan **set label**, bukan nilainya.

---

## 10. Label Selectors

### Konsep Dasar

- Query dimulai dari nama metrik, lalu dipersempit dengan filter label di dalam `{}`.
- **Koma (`,`) = AND:** semua filter harus terpenuhi bersamaan.

### Operator

| Operator | Arti |
|---|---|
| `=` | Sama persis |
| `!=` | Tidak sama |
| `=~` | Cocok dengan regex |
| `!~` | Tidak cocok dengan regex |

```promql
prometheus_http_requests_total{code=~"2.*", handler!="/metrics"}
```

### Catatan Penting

- **Regex:** `.*` cocok dengan karakter apa saja (misal `2.*` = diawali angka 2). Hindari regex yang bisa cocok dengan string kosong.
- **Tipe data:** Nilai label selalu **string**. Menulis `1000` tanpa kutip (bukan `"1000"`) akan error karena tidak ada konversi otomatis.

---

## 11. Operator Agregasi

### Konsep Dasar

- Meringkas elemen **instant vector** menjadi vektor baru dengan elemen lebih sedikit.
- Sintaks mirip fungsi: `operator(instant_vector)`.

### Daftar Operator

| Operator | Fungsi |
|---|---|
| `sum` | Total penjumlahan |
| `min` | Nilai terkecil |
| `max` | Nilai terbesar |
| `avg` | Rata-rata |
| `count` | Jumlah elemen |
| `group` | Mengelompokkan; nilai hasil selalu `1` |
| `count_values` | Jumlah elemen per nilai tertentu |
| `topk(n, v)` | *n* elemen dengan nilai **terbesar** |
| `bottomk(n, v)` | *n* elemen dengan nilai **terkecil** |
| `stddev` | Standar deviasi populasi |
| `stdvar` | Varians populasi |

### Modifier `by` dan `without`

- **`by (label)`** — kelompokkan **hanya** berdasarkan label yang disebut.

  ```promql
  sum(node_cpu_seconds_total) by (mode)
  ```

- **`without (label)`** — buang label yang disebut, pertahankan sisanya.

  ```promql
  sum(node_cpu_seconds_total) without (mode)
  ```

---

## 12. Time Offset

### Konsep Dasar

- Default-nya, instant vector mengambil nilai **terbaru**.
- **`offset`** menarik data dari titik waktu di masa lalu, memakai satuan durasi standar (`ms`, `s`, `m`, `h`, `d`, `w`, `y`).

```promql
prometheus_http_requests_total offset 10m
```

### Posisi `offset`

`offset` **harus diletakkan tepat setelah selector metrik**, bukan di akhir query.

✅ **Benar:**

```promql
avg(prometheus_http_requests_total offset 8h) by (code)
```

❌ **Salah:**

```promql
avg(prometheus_http_requests_total) by (code) offset 8h
```

---

## 13. Fungsi-Fungsi PromQL

### A. Ketersediaan Data

Berguna untuk *alerting* saat metrik hilang.

| Fungsi | Input | Perilaku |
|---|---|---|
| `absent(v)` | Instant vector | Ada data → hasil **kosong**. Tidak ada data → vektor 1 elemen bernilai **`1`** |
| `absent_over_time(v[d])` | Range vector | Sama seperti `absent`, untuk rentang waktu |

### B. Matematika Dasar

| Fungsi | Arti | Contoh |
|---|---|---|
| `abs()` | Nilai mutlak | `-5` → `5` |
| `ceil()` | Bulatkan ke atas | `1.6` → `2` |
| `floor()` | Bulatkan ke bawah | `1.6` → `1` |

### C. Clamping (Pembatasan Nilai)

Berguna untuk merapikan grafik di dashboard (misal Grafana).

| Fungsi | Arti |
|---|---|
| `clamp_min(v, min)` | Nilai di bawah `min` dinaikkan ke `min` |
| `clamp_max(v, max)` | Nilai di atas `max` diturunkan ke `max` |
| `clamp(v, min, max)` | Batas bawah dan atas sekaligus |

### D. Kalender

Mengekstrak informasi hari dari timestamp (UTC):

| Fungsi | Hasil |
|---|---|
| `day_of_month()` | `1`–`31` |
| `day_of_week()` | `0`–`6`, dengan **`0` = Minggu**, `1` = Senin, ..., `6` = Sabtu |

### E. Perubahan Nilai

> Hanya untuk **gauge** (nilai naik-turun, misal suhu, memori), **bukan counter**.

| Fungsi | Arti |
|---|---|
| `delta(v[d])` | Selisih nilai awal dan akhir dalam rentang |
| `idelta(v[d])` | Selisih **dua sampel terakhir** dalam rentang |

```promql
delta(node_cpu_temperature[2h])
```

### F. Logaritma

| Fungsi | Basis | Contoh |
|---|---|---|
| `log2(v)` | 2 | `2` → `1` |
| `log10(v)` | 10 | `10` → `1` |
| `ln(v)` | *e* (natural) | `e` → `1` |

### G. Pengurutan

| Fungsi | Urutan |
|---|---|
| `sort(v)` | Menaik (kecil → besar) |
| `sort_desc(v)` | Menurun (besar → kecil) |

### H. Waktu & Timestamp

| Fungsi | Hasil |
|---|---|
| `time()` | Unix timestamp saat ini (detik), sebagai scalar |
| `timestamp(v)` | Timestamp saat sampel di-*scrape*. Jika dipakai dengan `offset`, timestamp ikut bergeser ke masa lalu |

---

## 14. Aggregation Over Time

### Konsep Dasar

- Operator agregasi biasa (`sum`, `avg`, dll.) hanya bekerja pada **instant vector**; memakai range vector (`[2h]`) akan error.
- Untuk range vector, gunakan fungsi berakhiran **`_over_time`**, yang meringkas sampel historis **per time series**.

### Daftar Fungsi

| Fungsi | Hasil dalam rentang waktu |
|---|---|
| `avg_over_time(v[d])` | Rata-rata |
| `sum_over_time(v[d])` | Total |
| `min_over_time(v[d])` | Nilai terkecil |
| `max_over_time(v[d])` | Nilai terbesar |
| `count_over_time(v[d])` | Jumlah sampel |
| `stddev_over_time(v[d])` | Standar deviasi |
| `stdvar_over_time(v[d])` | Varians |

```promql
avg_over_time(node_load1[1h])
```
