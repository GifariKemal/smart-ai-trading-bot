# **Blueprint Sistem Trading Algoritmik (Forex/Gold) \- Edisi 2026**

Dokumen ini adalah cetak biru teknis (technical blueprint) untuk membangun sistem trading otomatis end-to-end yang menggabungkan logika institusional (SMC) dengan validasi statistik (Machine Learning).

## ---

**1\. Workflow (Alur Kerja Sistem)**

Sistem ini dirancang menggunakan arsitektur **Asynchronous Event-Driven**. Robot tidak bekerja secara linear (menunggu), tetapi bereaksi terhadap *event* (perubahan harga) secara real-time.

1. **Data Ingestion (Penyedot Data):**  
   * Koneksi ke **MetaTrader 5 (MT5)** via Python API.  
   * Streaming data *tick* atau *candle* (M1/M15) secara real-time.  
2. **Preprocessing & Feature Engineering:**  
   * Pembersihan data (hapus *bad tick*).  
   * Kalkulasi indikator teknikal & Deteksi Pola SMC (FVG, Order Block) menggunakan library polars (pengganti Pandas).  
3. **Market Regime Detection (Filter Pasar):**  
   * **Algoritma:** HMM (Hidden Markov Model).  
   * **Fungsi:** Menentukan apakah pasar sedang *Trending*, *Ranging*, atau *High Volatility/Crisis*.  
   * **Output:** Jika *Crisis*, robot masuk mode "Sleep".  
4. **Signal Generation (Otak AI):**  
   * **Algoritma:** XGBoost / LightGBM (Ensemble).  
   * **Logika:** Jika HMM \= "Aman", data masuk ke model prediksi.  
   * **Output:** Probabilitas arah harga (Buy/Sell/Hold).  
5. **Risk Engine (Polisi Risiko):**  
   * Cek saldo & Margin Level.  
   * Hitung lot size dinamis (Risk-Constrained Kelly Criterion).  
   * Cek batas kerugian harian (*Daily Loss Limit*).  
6. **Execution (Eksekusi):**  
   * Kirim order ke broker via MT5 (Order Send).  
   * Set *Hard Stop Loss* & *Take Profit* di server broker.  
7. **Monitoring & Logging:**  
   * Simpan setiap keputusan (Signal, Risk, Result) ke database untuk audit & retraining.

## ---

**2\. Tech Stack (Tumpukan Teknologi)**

Kami memilih teknologi yang standar digunakan di industri *Quantitative Finance* pada tahun 2026 untuk kecepatan dan stabilitas.

* **Bahasa Pemrograman:** Python 3.11+ (Wajib mendukung asyncio).  
* **Database:**  
  * **Redis (Hot Storage):** Untuk menyimpan data harga real-time dan status order aktif (in-memory, sangat cepat).  
  * **TimescaleDB / PostgreSQL (Cold Storage):** Untuk menyimpan data historis bertahun-tahun dan jurnal trading.  
* **Infrastructure:**  
  * **VPS:** Wajib lokasi **Singapura (SG1)** atau **London (LD4)** tergantung lokasi server broker Anda (Target Latency: \< 5ms).  
  * **Container:** **Docker** (Agar lingkungan development di laptop sama persis dengan di VPS).

## ---

**3\. Library Python (Wajib Install)**

| Kategori | Library | Fungsi Utama |
| :---- | :---- | :---- |
| **Koneksi Broker** | MetaTrader5 | Library resmi untuk kontrol terminal MT5. |
| **Data Processing** | polars | Pengganti pandas. 50x lebih cepat untuk memproses data time-series besar. |
| **Analisis Teknikal** | pandas-ta, smartmoneyconcepts | smartmoneyconcepts untuk deteksi FVG/Order Block otomatis. |
| **Machine Learning** | xgboost, scikit-learn, joblib | Algoritma prediksi utama (Gradient Boosting). |
| **Regime Detection** | hmmlearn | Mendeteksi fase pasar (Hidden Markov Model). |
| **Backtesting** | vectorbt | Backtesting performa tinggi berbasis vektor (bukan looping). |
| **Asynchronous** | asyncio | Manajemen proses paralel (non-blocking). |

## ---

**4\. Algoritma (The Hybrid Brain)**

Sistem ini tidak menggunakan satu otak, melainkan sistem **Ensemble** (Gabungan):

1. **Gatekeeper (HMM \- Hidden Markov Model):**  
   * *Tugas:* Menjawab "Apakah pasar kondusif?"  
   * *Input:* Volatilitas (ATR), Return Distribution.  
   * *Keputusan:* Jika pasar *High Volatility* (misal: saat berita NFP), HMM akan memblokir semua sinyal trading.  
2. **Signal Generator (XGBoost/LightGBM):**  
   * *Tugas:* Menjawab "Beli atau Jual?"  
   * *Input:* Pola SMC (jarak ke Order Block), RSI, Moving Average, Volume.  
   * *Kenapa XGBoost?* Lebih ringan dan akurat untuk data tabular (harga) dibandingkan Deep Learning yang berat.  
3. **Optimizer (Walk-Forward):**  
   * *Tugas:* Melatih ulang model (*Retraining*) setiap minggu/bulan agar robot tidak "kadaluarsa" (Data Drift).

## ---

**5\. Strategi (SMC \+ AI Filter)**

Strategi murni SMC sering terjebak *fakeout* (jebakan likuiditas). Kita gunakan AI untuk memfilternya.

* **Timeframe:** Eksekusi di **M15**, Tren dilihat di **H4**.  
* **Logic Entry (SMC):**  
  * Cari **FVG (Fair Value Gap)** yang searah dengan tren besar (H4).  
  * Tunggu harga masuk kembali (*mitigation*) ke area **Order Block**.  
  * Validasi adanya **BOS (Break of Structure)** kecil di M15.  
* **Logic Filter (AI Validation):**  
  * Saat setup SMC muncul, kirim data ke AI: "Kondisi sekarang seperti ini, peluang menang berapa?"  
  * Jika AI Score \> **65%** ![][image1] **EKSEKUSI**.  
  * Jika AI Score \< 65% ![][image1] **ABAIKAN** (Meskipun chart terlihat bagus, statistik historis mengatakan risikonya tinggi).

## ---

**6\. Risk Management (Jantung Sistem)**

Tanpa ini, strategi terbaik pun akan bangkrut.

1. **Position Sizing:** Gunakan **Risk-Constrained Kelly Criterion**.  
   * Rumus ini menghitung lot optimal agar akun tumbuh maksimal, tapi membatasi *drawdown* agar tidak agresif.  
   * *Rule of Thumb:* Jangan pernah trade lebih dari **1% \- 2%** risiko per transaksi.  
2. **Circuit Breaker (Sekring Otomatis):**  
   * **Daily Loss Limit:** Jika rugi hari ini \> 3% modal, Robot **STOP** trading sampai besok.  
   * **Flash Crash Guard:** Jika harga bergerak \> 1% dalam 1 menit (anomali), tutup semua posisi segera.  
3. **Hard Stop Loss:** Wajib dipasang di server broker (bukan hanya di memori Python) untuk jaga-jaga jika koneksi internet putus.

## ---

**7\. Data & Training**

* **Sumber:** Data *Tick* asli dari broker yang Anda pakai (JANGAN pakai data Yahoo Finance untuk Forex, karena beda server beda harga/spread).  
* **Training Method:** **Walk-Forward Optimization**.  
  * *Salah:* Train data 2020-2024, Test 2025\.  
  * *Benar (Rolling):* Train Jan-Mar, Test Apr. Train Feb-Apr, Test Mei. Ini mensimulasikan kondisi real-time yang terus berubah.

## ---

**8\. Skenario Modal: $5,000 vs $50,000**

Strategi harus disesuaikan dengan ukuran modal.

### **Skenario A: Modal Kecil ($5,000)**

* **Tujuan:** Pertumbuhan Akun (*Growth*).  
* **Strategi:** Sedikit agresif (Scalping/Day Trading M15).  
* **Aset:** Fokus 1-2 Pair likuid (XAUUSD, GBPUSD).  
* **Risiko per Trade:** 1.5% ($75).  
* **Leverage:** 1:100 (Dibutuhkan untuk margin).  
* **Infrastruktur:** VPS Shared ($10-$15/bulan).

### **Skenario B: Modal Menengah ($50,000)**

* **Tujuan:** Keamanan & Konsistensi (*Wealth Preservation*).  
* **Strategi:** Konservatif (Swing Trading H1/H4) & Portfolio.  
* **Aset:** Diversifikasi 5-10 Aset (Forex Major, Gold, Oil, Indeks) agar risiko tersebar.  
* **Risiko per Trade:** 0.5% \- 1% ($250 \- $500).  
* **Leverage:** 1:30 atau 1:50 (Lebih aman).  
* **Infrastruktur:** VPS Dedicated / Bare Metal ($50+/bulan) untuk stabilitas maksimal.

## ---

**9\. Broker & Infrastruktur (Konteks Indonesia)**

**Opsi A: Broker Lokal (Regulasi Bappebti) \- Aman Secara Hukum**

* **Rekomendasi:** **Dupoin**, **MIFX (Monex)**, atau **Moneta Markets**.  
* **Kenapa:** Dana aman dijamin KBI (Kliring Berjangka Indonesia). Mendukung MT5.  
* **Setup:** VPS Windows \-\> Install MT5 \-\> Install Python \-\> Robot jalan di VPS.

**Opsi B: Broker Luar (Offshore) \- Teknologi Terbaik**

* **Rekomendasi:** **Interactive Brokers (IBKR)** atau **IC Markets**.  
* **Kenapa:** Spread sangat tipis (Raw ECN), API kelas dunia.  
* **Tantangan:** Deposit/WD lebih kompleks, website sering diblokir (butuh VPN/DoH).

## **10\. Konsekuensi & Realita**

1. **Biaya Operasional:** Siapkan budget rutin untuk VPS ($15-$50/bln) dan Data Feed (jika perlu).  
2. **Maintenance:** Robot perlu "di-servis" (Retraining model) minimal sebulan sekali.  
3. **Psikologi:** Tantangan terberat adalah **membiarkan robot bekerja saat sedang rugi (drawdown)**. Jangan intervensi manual kecuali *Circuit Breaker* jebol.

[image1]: <data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAABMAAAAYCAYAAAAYl8YPAAAAfUlEQVR4XmNgGAWjYHACWSDuBmIOdAlyQTkUUwWIAfF+IDZDlyAXgAw6AsQq6BI8QCxJBg4G4kdAzMmABCqggqTiZ0D8H4jjGSgE3EC8EIj70CVIBa5AvJoBzXvkABYGiIs80CXIAdJAvBmIRdAlyAGsQCwExIzoEqNggAEAkekYp+CjMnEAAAAASUVORK5CYII=>