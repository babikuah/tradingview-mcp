# Rancangan: EA Scalping XAUUSD dengan Martingale dan Filter Machine Learning (MT5)

Tanggal: 2026-11-27
Status: draf, menunggu persetujuan

## 1. Maksud dan alasan

Pengguna ingin EA MetaTrader 5 yang melakukan scalping XAUUSD secara otomatis. Yang dicari adalah **untung kecil tapi sering, dikumpulkan dari banyak transaksi**: target $0.01 sampai $10 per posisi, banyak posisi terbuka bersamaan (lebih dari 100 diizinkan), buy dan sell boleh terbuka bersamaan, memakai martingale dan compounding. Pengguna juga ingin **AI ikut terlibat** di MT5. Bentuk yang dipilih adalah model machine learning yang ditanam di dalam EA.

Pengguna belum punya strategi sendiri, jadi aturan entri dirancang dari nol (bagian 6).

Kondisi akun pengguna:

- Modal $1,000.
- Leverage 1:5000.
- Rugi maksimal yang bisa diterima: 50%.
- Platform MT5.

## 2. Tujuan, bukan tujuan, dan anti-tujuan

**Tujuan**

- EA MT5 untuk XAUUSD di timeframe M1–M15 yang membuka dan menutup posisi sendiri dengan cepat.
- Banyak posisi terbuka bersamaan, buy dan sell bisa berjalan sekaligus.
- Martingale (averaging dengan lot membesar) dan compounding (lot dasar ikut saldo).
- TP, SL, dan trailing stop per posisi, ditambah TP dan trailing per kelompok.
- Filter sinyal berbasis model machine learning (ONNX) yang bisa dimatikan.
- Batas risiko yang **tidak bisa dilewati** oleh logika lain mana pun di dalam EA.

**Bukan tujuan (tahap ini)**

- Versi MT4. MT4 tidak mendukung ONNX.
- AI percakapan atau LLM (misalnya Claude API lewat WebRequest). Ini bisa menyusul, tapi tidak dibangun sekarang.
- Multi-simbol. Hanya XAUUSD.
- Model yang menentukan lot, arah, atau ukuran martingale.
- Aturan prop firm.
- Jaminan profit.

**Anti-tujuan.** Hal-hal berikut berarti gagal, walaupun EA-nya jalan:

- Ada jalur kode yang membuka posisi **tanpa SL di sisi broker**.
- Ekuitas bisa turun melewati batas 50% karena logika EA. Satu-satunya pengecualian adalah gap atau slippage di luar kendali EA (bagian 7.4).
- Model ML dipakai padahal tidak terbukti lebih baik daripada tanpa model di data yang tidak dipakai saat latihan.
- EA hanya "berhasil" di backtest mode open-prices tapi gagal di mode real ticks.
- Setelah restart terminal atau VPS, EA kehilangan jejak posisinya sendiri.

## 3. Batasan

- **Akun MT5 wajib mode hedging.** Di mode netting, buy dan sell saling meniadakan. EA menolak berjalan (INIT_FAILED dengan pesan jelas) kalau `ACCOUNT_MARGIN_MODE` bukan `ACCOUNT_MARGIN_MODE_RETAIL_HEDGING`.
- **Leverage 1:5000.** Margin praktis tidak lagi menjadi rem, karena 100 posisi 0.01 lot hanya memakai sekitar $73 pada harga $3,650. Yang menahan akun hanyalah batas risiko di dalam EA. EA tidak boleh mengasumsikan leverage. Margin dibaca dari broker (`OrderCalcMargin`), karena banyak broker memberi leverage lebih kecil untuk gold, terutama menjelang berita atau akhir pekan.
- **Spesifikasi simbol dibaca dari broker, tidak di-hardcode.** Ini mencakup ukuran kontrak, nilai tick, volume min/max/step, stop level, freeze level, filling mode, batas jumlah order (`ACCOUNT_LIMIT_ORDERS`), dan nama simbol (bisa `XAUUSD`, `XAUUSDm`, `GOLD`, dan sebagainya; diambil dari chart tempat EA dipasang).
- **Volatilitas saat ini tinggi.** Gold sekitar $3,650 (27 Nov 2026), dengan pergerakan harian $100–170 di bulan terakhir. Contohnya 19→20 Nov turun sekitar $170 dan 25→26 Nov naik sekitar $150.
- **Biaya per posisi.** Spread gold sekitar $0.10–0.50 per ons, ditambah komisi kalau ada. Target bersih sekecil $0.01 per posisi hanya mungkin kalau TP otomatis dipasang di atas biaya (bagian 6.4).
- **Aturan broker.** Banyak broker membatasi jumlah order terbuka, dan sebagian menandai scalping sangat cepat atau modifikasi order berlebihan. Pengguna perlu mengecek aturan brokernya. EA meminimalkan modifikasi dengan langkah trailing (bagian 6.5).
- **Fungsi yang tidak jalan di Strategy Tester:** kalender ekonomi MT5 (filter berita) dan WebRequest. Filter berita otomatis nonaktif di tester, dan ini dicatat di log.
- **Lingkungan pembangun.** Kode ditulis di lingkungan Linux tanpa MetaEditor, jadi kompilasi MQL5 dan Strategy Tester dijalankan oleh pengguna di Windows. Bagian Python bisa diuji penuh di lingkungan pembangun.

## 4. Pendekatan yang dipertimbangkan

### 4.1 Cara kerja EA

| Pendekatan | Ringkasan | Kelebihan | Kekurangan |
|---|---|---|---|
| A. Grid dua arah | Grid buy dan grid sell selalu berjalan. Sisi untung ditutup cepat lalu dibuka lagi, sisi rugi ditambah layer martingale. | Transaksi paling banyak, tidak perlu menebak arah. | Saat trending kuat, sisi rugi menumpuk paling cepat. Spread dibayar di dua sisi. |
| **B. Sinyal + averaging martingale (dipilih)** | Entri dari sinyal tren dan pullback. Kelompok buy dan sell terpisah. Jarak layer mengikuti ATR. | Layer lebih sedikit dan terukur. Jarak menyesuaikan volatilitas, sehingga batas rugi bertahan lebih lama. | Transaksi lebih sedikit dari A. Hasil bergantung pada kualitas sinyal. |
| C. Zone recovery (hedge lock) | Saat harga melawan, buka arah sebaliknya dengan lot lebih besar, bolak-balik. | Siklus cepat selesai di pasar sideways. | Lot naik paling cepat. Dengan $1,000, margin dan batas rugi habis setelah 5–6 langkah. Ditolak. |

Pengguna memilih **B**.

### 4.2 Keterlibatan AI

| Pendekatan | Ringkasan | Kelebihan | Kekurangan |
|---|---|---|---|
| 1. EA bertanya ke LLM lewat WebRequest | Sebelum membuka kelompok, EA mengirim ringkasan kondisi ke API AI dan menerima "boleh buy", "boleh sell", atau "tunggu". | Bisa memberi alasan dalam bahasa manusia. | Tidak bisa di-backtest. Butuh API key, biaya per panggilan, dan PC atau VPS online. |
| 2. Program Python di samping MT5 | Python membaca MT5 lewat paket `MetaTrader5`, bertanya ke AI, lalu memberi perintah ke EA. | Paling fleksibel. | Hanya Windows. Ada lebih banyak bagian yang harus jalan 24 jam. |
| **3. Model ML di dalam EA (ONNX), dipilih** | Model dilatih di Python dari data historis broker, lalu dijalankan di EA. | Cepat, gratis saat berjalan, dan **bisa di-backtest**. | Hanya mengenali pola, tidak bisa diajak bicara. Perlu dilatih ulang berkala. |

Pengguna memilih **3**, dengan peran **penyaring sinyal**, bukan pengganti aturan (bagian 8).

## 5. Gambaran arsitektur

```
             ┌──────────────── EA XauScalperML (MT5) ────────────────┐
 tick/bar →  │ Filter (spread, jam, berita, akhir pekan)              │
             │   ↓                                                    │
             │ Sinyal (EMA M15 + RSI M5) → Filter ML (ONNX, opsional)  │
             │   ↓                                                    │
             │ Gerbang Risiko  ←  wajib dilewati SETIAP order baru     │
             │   ↓                                                    │
             │ Pengelola Kelompok (entri, layer martingale, exit)     │
             │ Trailing (per posisi / per kelompok)                   │
             │ Rem Darurat (harian 10%, global 50%)                   │
             │ State (pulih setelah restart) + Panel di chart          │
             └────────────────────────────────────────────────────────┘

 Python (di luar MT5): data CSV hasil ekspor → fitur → label → latih → ONNX → cek paritas
```

## 6. Logika trading (yang sudah diputuskan)

Semua angka di bagian ini adalah **input EA** yang bisa diubah. Angka yang tertulis adalah default.

### 6.1 Sinyal entri

Sinyal dievaluasi **sekali per bar tertutup** di timeframe sinyal. Semua data indikator memakai shift 1 (bar yang sudah tertutup), supaya tidak repaint.

- **Timeframe:** sinyal M5 (input: M1, M5, atau M15), tren M15.
- **Arah tren** dari EMA 50 dan EMA 200 di M15 pada bar tertutup:
  - Naik: EMA50 > EMA200 dan close > EMA200, sehingga EA hanya mencari buy.
  - Turun: EMA50 < EMA200 dan close < EMA200, sehingga EA hanya mencari sell.
  - Selain itu dianggap netral, dan tidak ada kelompok baru.
- **Pemicu** dari RSI 14 di timeframe sinyal:
  - Buy: tren naik, RSI bar sebelumnya < 40, dan RSI bar terakhir ≥ 40 (pullback selesai).
  - Sell: tren turun, RSI bar sebelumnya > 60, dan RSI bar terakhir ≤ 60.
- **Lawan arah.** Kelompok buy dan sell dikelola terpisah. Saat tren berbalik, kelompok arah baru boleh dibuka walau kelompok arah lama masih terbuka.
- **Filter ML** dipakai kalau aktif (bagian 8). Sinyal hanya dieksekusi kalau peluang dari model ≥ `MLThreshold` (default 0.60).

### 6.2 Kelompok (basket) dan banyak posisi

- Satu **kelompok** terdiri dari 1 posisi pembuka ditambah 0..N layer martingale, semuanya satu arah.
- Maksimal `MaxBasketsPerDirection` kelompok per arah (default 3). Sinyal baru searah boleh membuka kelompok baru walau kelompok lama masih terbuka, selama lolos gerbang risiko.
- Batas atas total posisi milik EA: `MaxTotalPositions` (default 150). Ini plafon. Jumlah nyata ditentukan oleh gerbang risiko (6.6).
- **Catatan jujur untuk pengguna:** dengan modal $1,000 dan default risiko di dokumen ini, jumlah posisi terbuka biasanya **jauh di bawah 100**. Bisa jadi hanya puluhan, atau lebih sedikit saat ATR tinggi. 100+ posisi hanya tercapai kalau batas risiko dinaikkan (6.6), dan itu memperbesar peluang menyentuh rem.

### 6.3 Martingale, compounding, dan jarak layer

- **Compounding (lot dasar):** `BaseLot = LotPerStep × floor(Balance / CompoundStepUSD)`, dengan default 0.01 per $1,000 dan minimal satu volume minimum. Dihitung dari **balance** (bukan ekuitas) di saat kelompok dibuka, supaya tidak membengkak karena floating profit. Lot dasar tidak berubah selama kelompok hidup.
- **Lot layer ke-k (k=0 untuk posisi pembuka):** `BaseLot × MartingaleMultiplier^k`, dibulatkan ke volume step broker, dengan default pengali 1.3. Ada pilihan mode tambahan (`+LotIncrement` per layer) kalau pengguna lebih suka kenaikan linear.
- **Jarak layer:** layer baru dibuka kalau harga bergerak melawan sejauh `LayerSpacing` dari layer terakhir. `LayerSpacing = LayerAtrFactor × ATR(14, M15)` dihitung saat layer terakhir dibuka (default faktor 0.5), dengan minimum `MinLayerSpacing` (default $1.00).
- Maksimal layer per kelompok: `MaxLayersPerBasket` (default 15). Ini juga plafon, dan gerbang risiko biasanya berhenti lebih dulu.
- Layer baru juga harus lolos filter spread, berita, dan jam (6.8), plus gerbang risiko.

### 6.4 TP dan SL

- **SL kelompok** dipasang saat posisi pembuka dibuka, di harga pembuka ∓ `BasketSLAtrFactor × ATR(14, M15)` (default 3.0). **Setiap posisi di kelompok itu memakai harga SL yang sama, dipasang di sisi broker.** Dengan begitu perlindungan tetap ada walau EA, terminal, atau VPS mati. SL kelompok tidak pernah dijauhkan.
- **TP per posisi** berlaku saat kelompok masih berisi 1 posisi:
  - Jarak TP = `TPAtrFactor × ATR(14, timeframe sinyal)` (default 1.0).
  - Batas bawah: spread saat ini + komisi (dikonversi ke jarak harga) + `MinNetProfitPer001` (default $0.10 per 0.01 lot, bisa diturunkan sampai $0.01).
  - Batas atas: `MaxTPPer001` (default $10 per 0.01 lot).
- **Begitu kelompok punya ≥ 2 posisi**, TP per posisi dihapus dan exit dikelola di tingkat kelompok (6.5). SL tetap terpasang.
- **TP kelompok:** semua posisi ditutup kalau untung bersih kelompok ≥ `BasketTPPer001 × (BaseLot/0.01)`, dengan default $5.

### 6.5 Trailing stop

**Per posisi** berlaku saat kelompok berisi 1 posisi. Angkanya sama dengan contoh yang sudah disetujui pengguna:

- Aktif saat untung ≥ `TrailStart` (default $2.00 jarak harga).
- Saat aktif, SL pindah ke harga masuk ± `TrailLock` (default $0.50). Kunci ini harus ≥ komisi plus untung bersih minimal. EA menaikkan nilainya otomatis kalau terlalu kecil.
- Setelah itu SL mengikuti harga terbaik dengan jarak `TrailDistance` (default $1.50).
- SL hanya dimodifikasi kalau bergeser ≥ `TrailStep` (default $0.50), supaya tidak membanjiri broker dengan perintah modifikasi.
- SL tidak pernah digeser menjauh, dan selalu mematuhi stop level dan freeze level.

**Per kelompok** berlaku saat kelompok berisi ≥ 2 posisi:

- Aktif saat untung bersih kelompok ≥ `BasketTrailStartPer001 × (BaseLot/0.01)` (default $3).
- Puncak untung kelompok dicatat. Semua posisi ditutup kalau untung turun ≥ `BasketTrailDropPct` (default 30%) dari puncak.

### 6.6 Gerbang risiko (inti keamanan)

**Setiap** order baru (posisi pembuka maupun layer) wajib lolos gerbang ini. Tidak ada jalur yang melewatinya.

Definisi:

- `ProjEquityAtSL` = balance + jumlah P/L semua posisi EA **seandainya semuanya ditutup di SL masing-masing**, termasuk komisi dan swap berjalan, **ditambah posisi baru yang diusulkan**, dengan buffer slippage `SLSlippageBuffer` (default $0.30 jarak harga per posisi).
- `DayFloor` = ekuitas saat awal hari server × (1 − `DailyLossPct`), default 10%.
- `GlobalFloor` = HWM × (1 − `MaxDrawdownPct`), default 50%. HWM adalah balance tertinggi yang pernah dicapai, minimal deposit awal.
- `BasketWorstLoss` = P/L kelompok itu kalau semua posisinya kena SL, termasuk posisi baru.

Order hanya boleh dikirim kalau **semua** syarat ini benar:

1. `ProjEquityAtSL ≥ max(DayFloor, GlobalFloor)`.
2. `BasketWorstLoss ≥ −BasketRiskPct × Balance`, default 5%.
3. Margin bebas cukup menurut `OrderCalcMargin`, dengan cadangan margin level ≥ `MinMarginLevelPct` (default 300%).
4. Jumlah posisi < `MaxTotalPositions` dan < batas order broker.

Setiap penolakan dicatat di log beserta alasannya. Begitu trailing menggeser SL ke arah untung, `ProjEquityAtSL` naik dan ruang untuk posisi baru bertambah.

Ilustrasi kenapa posisi biasanya sedikit (dengan ATR M15 sekitar $8, lot dasar 0.01, pengali 1.3, jarak layer 0.5×ATR sekitar $4, SL 3×ATR sekitar $24): kerugian terburuk layer 1–3 di SL sekitar $24, +$20, +$32. Batas per kelompok 5% ($50) sudah habis setelah 2 posisi. Ini memang disengaja, dan angkanya bisa diubah.

### 6.7 Rem darurat

Rem ini berjalan setiap tick, independen dari gerbang. Gunanya menangkap gap atau slippage yang membuat ekuitas turun melewati batas walaupun ada SL.

- **Harian:** kalau ekuitas ≤ `DayFloor`, semua posisi EA ditutup dan EA berhenti membuka posisi sampai hari server berikutnya.
- **Global:** kalau ekuitas ≤ `GlobalFloor`, semua posisi EA ditutup dan EA **berhenti permanen**. EA baru jalan lagi kalau pengguna mengubah input `ResetGlobalStop = true` secara sadar. Statusnya disimpan supaya tidak hilang saat restart.

### 6.8 Filter

- **Spread:** tidak ada order baru kalau spread > `MaxSpreadPoints` (default 40 point, yaitu $0.40 pada broker 2 digit).
- **Jam rollover:** tidak ada order baru antara `NoTradeStart` dan `NoTradeEnd` jam server (default 23:30–01:30).
- **Berita:** memakai kalender ekonomi bawaan MT5, yaitu event mata uang USD dengan tingkat penting **tinggi** (NFP, CPI, FOMC, dan sejenisnya). Tidak ada kelompok baru maupun layer baru dari `NewsMinutesBefore` sampai `NewsMinutesAfter` (default 30/30 menit). Di Strategy Tester filter ini nonaktif, dan EA mencatatnya di log.
- **Akhir pekan:** tidak ada kelompok baru setelah Jumat `FridayNoNewHour` (default 20:00 server). Kalau `CloseBeforeWeekend = true` (default), semua posisi ditutup Jumat `FridayCloseHour` (default 22:00 server), karena gap Senin bisa melompati SL.

### 6.9 Eksekusi dan ketahanan

- Magic number unik per instance EA. EA hanya mengelola posisinya sendiri.
- Deteksi filling mode otomatis. Retry terbatas untuk requote atau harga berubah. Setiap order selalu dikirim dengan SL.
- **Pemulihan state setelah restart:** keanggotaan kelompok (ticket → id kelompok, indeks layer, lot dasar, harga SL kelompok, puncak trailing) disimpan di file state (MQL5\Files). Komentar order hanya cadangan, karena sebagian broker menimpa komentar. Day-start equity, HWM, dan status rem disimpan di GlobalVariables terminal.
- Kalau ada posisi magic EA yang tidak dikenali di state, posisi itu dimasukkan sebagai kelompok sendiri dengan SL yang sudah ada. Kalau tidak punya SL, EA memasang SL darurat sesuai aturan 6.4.

### 6.10 Panel di chart

Panel teks (Comment) menampilkan: balance, ekuitas, HWM, P/L hari ini, sisa jatah rugi harian dan global, `ProjEquityAtSL`, ringkasan kelompok (arah, jumlah layer, P/L, SL), peluang ML terakhir, dan status setiap filter atau rem.

## 7. Input EA (ringkasan default)

| Grup | Input | Default |
|---|---|---|
| Umum | `MagicNumber` | 20261127 |
| Sinyal | `SignalTF` / `TrendTF` | M5 / M15 |
| Sinyal | `EmaFast` / `EmaSlow` | 50 / 200 |
| Sinyal | `RsiPeriod` / `RsiBuyLevel` / `RsiSellLevel` | 14 / 40 / 60 |
| Lot | `LotPerStep` / `CompoundStepUSD` | 0.01 / 1000 |
| Martingale | `MartingaleMode` / `MartingaleMultiplier` / `LotIncrement` | Multiply / 1.3 / 0.01 |
| Martingale | `LayerAtrFactor` / `MinLayerSpacing` / `MaxLayersPerBasket` | 0.5 / $1.00 / 15 |
| Kelompok | `MaxBasketsPerDirection` / `MaxTotalPositions` | 3 / 150 |
| TP/SL | `BasketSLAtrFactor` / `TPAtrFactor` | 3.0 / 1.0 |
| TP/SL | `MinNetProfitPer001` / `MaxTPPer001` / `BasketTPPer001` | $0.10 / $10 / $5 |
| Trailing | `TrailStart` / `TrailLock` / `TrailDistance` / `TrailStep` | $2.00 / $0.50 / $1.50 / $0.50 |
| Trailing | `BasketTrailStartPer001` / `BasketTrailDropPct` | $3 / 30% |
| Risiko | `DailyLossPct` / `MaxDrawdownPct` / `BasketRiskPct` | 10% / 50% / 5% |
| Risiko | `SLSlippageBuffer` / `MinMarginLevelPct` / `ResetGlobalStop` | $0.30 / 300% / false |
| Filter | `MaxSpreadPoints` / `NoTradeStart` / `NoTradeEnd` | 40 / 23:30 / 01:30 |
| Filter | `NewsFilter` / `NewsMinutesBefore` / `NewsMinutesAfter` | true / 30 / 30 |
| Filter | `FridayNoNewHour` / `CloseBeforeWeekend` / `FridayCloseHour` | 20 / true / 22 |
| ML | `MLEnabled` / `MLThreshold` / `MLModelFile` | true / 0.60 / `xau_filter.onnx` |
| Debug | `DebugFeatureLog` | false |

## 8. Filter machine learning

### 8.1 Peran

Model **hanya menyaring** sinyal dari aturan 6.1. Model menghitung peluang (0–1) bahwa setelah masuk, harga **mencapai jarak TP sebelum mencapai jarak layer pertama** yang melawan. Dengan kata lain, peluang kelompok selesai tanpa perlu averaging. Model tidak menentukan lot, arah sendiri, martingale, atau exit. Alasannya, kerugian besar EA martingale datang dari kelompok yang terus ditambah layer, sedangkan kalau model salah menentukan lot atau arah, tidak ada aturan lain yang menahannya.

### 8.2 Data

- Skrip MQL5 `ExportXauData.mq5` mengekspor bar M1, M5, dan M15 XAUUSD dari **broker pengguna sendiri** ke CSV: waktu, OHLC, tick volume, dan spread per bar. Targetnya data sebanyak yang tersedia, minimal 12 bulan, idealnya ≥ 2 tahun.
- Pengguna menjalankan skrip itu di MT5-nya dan mengirim CSV-nya. Pelatihan berjalan di Python.

### 8.3 Fitur

Semua fitur dihitung dari **bar tertutup** dan harus identik di Python dan MQL5:

- RSI(14) sinyal dan perubahannya dari bar sebelumnya.
- (close − EMA50) / ATR, dan (EMA50 − EMA200) / ATR di M15.
- Kemiringan EMA50 M15 selama 3 bar, dinormalisasi ATR.
- Rasio ATR sinyal / ATR M15, dan ATR M15 / harga.
- Return 1, 3, dan 6 bar terakhir di timeframe sinyal, dinormalisasi ATR.
- Posisi close dalam range high-low bar terakhir.
- Spread bar terakhir dalam satuan ATR.
- Jam server (sin dan cos) dan hari dalam minggu.
- Arah sinyal (+1 buy, −1 sell).

**Peringatan paritas:** indikator Python harus meniru implementasi MT5 persis, termasuk cara inisialisasi EMA, smoothing RSI, dan cara MT5 menghitung ATR. Pembangun memverifikasinya terhadap keluaran MT5 dengan cek paritas di 8.6, bukan dengan asumsi.

### 8.4 Label

Untuk setiap sinyal aturan dalam histori:

1. Entri dihitung di open bar berikutnya. Buy di ask = bid + spread bar.
2. Jarak TP dan jarak layer pertama dihitung dengan rumus EA yang sama (6.3, 6.4).
3. Label 1 kalau dalam `H` bar (default 48 bar M5, sekitar 4 jam) harga mencapai TP lebih dulu daripada jarak layer pertama yang melawan.
4. Label 0 kalau jarak layer pertama tersentuh lebih dulu atau waktunya habis.
5. Urutan di dalam bar diselesaikan dengan data M1. Kalau masih ambigu, labelnya 0 (konservatif).

### 8.5 Pelatihan dan validasi

- Model: gradient boosting (LightGBM atau setara) yang bisa dikonversi ke ONNX dengan andal. Pilihan library diserahkan ke pembangun, dengan syarat lulus cek kesetaraan ONNX.
- Pembagian **berdasarkan waktu**: kira-kira 70% data tertua untuk latihan, 15% berikutnya untuk validasi (memilih parameter dan threshold), dan 15% terbaru untuk uji. Ada jeda (purge) sepanjang `H` bar di antara setiap bagian.
- **Kriteria lulus** di periode uji, dibandingkan dengan semua sinyal aturan tanpa filter:
  1. Persentase sinyal yang menyentuh layer pertama turun **≥ 5 poin persentase**.
  2. Sinyal yang tersisa ≥ 30% dari total.
  3. Di Strategy Tester MT5 (real ticks) pada periode uji yang sama: drawdown maksimal dengan ML ≤ drawdown tanpa ML, dan profit bersih dengan ML ≥ tanpa ML.
- **Kalau tidak lulus**, default `MLEnabled` diubah menjadi `false` dan hasilnya dilaporkan apa adanya ke pengguna.

### 8.6 ONNX di EA dan cek paritas

- Model dimuat dari file (`MLModelFile`). Ada `#property tester_file` supaya model ikut terbawa di Strategy Tester. Dengan cara ini pengguna bisa mengganti model hasil latih ulang tanpa mengubah kode. Kalau pembangun memilih cara lain, syarat ini tetap harus dipenuhi.
- Kalau `MLEnabled = true` tapi model gagal dimuat atau bentuk input dan outputnya tidak cocok, EA **tidak membuka kelompok baru** (fail-safe) dan menampilkan galat di panel dan log. Posisi yang sudah terbuka tetap dikelola.
- **Cek kesetaraan ONNX:** keluaran model asli dan keluaran ONNX (lewat onnxruntime) harus sama untuk set uji, dengan selisih ≤ 1e-5.
- **Cek paritas fitur:** dengan `DebugFeatureLog = true`, EA menulis vektor fitur setiap sinyal ke CSV. Skrip `parity_check.py` membandingkannya dengan fitur Python untuk bar yang sama. Toleransinya 1e-4, dan wajib lulus sebelum model dipakai.

### 8.7 Latih ulang

Prosedur bulanan didokumentasikan di README: ekspor data, latih, cek kelulusan, salin `.onnx` ke folder MT5, lalu restart EA.

## 9. Pengujian

- **Python:** unit test untuk indikator, fitur, label (kasus buatan dengan jawaban yang sudah diketahui), pembagian waktu (tidak ada kebocoran), dan kesetaraan ONNX. Semuanya dijalankan di lingkungan pembangun.
- **MQL5:** pembangun tidak bisa mengompilasi di Linux, jadi kodenya harus ditulis konservatif dan disertai checklist kompilasi. Pengguna mengompilasi di MetaEditor dan melaporkan galat, lalu pembangun memperbaikinya.
- **Strategy Tester** (dijalankan pengguna, mode "Every tick based on real ticks", periode ≥ 6 bulan termasuk November 2026). Hal yang harus dicek di log:
  - Tidak ada satu pun order tanpa SL.
  - Gerbang risiko tidak pernah meloloskan order yang membuat `ProjEquityAtSL` di bawah batas. Setiap keputusan gerbang dicatat dalam satu baris log yang mudah dicari.
  - Rem harian dan global bekerja. Uji dengan menurunkan `DailyLossPct` dan `MaxDrawdownPct` supaya rem pasti tersentuh.
  - State pulih setelah tester dihentikan dan dilanjutkan, atau setelah EA dilepas dan dipasang lagi di akun demo.
- **Forward test di akun demo** minimal 2 minggu sebelum dipakai di akun live.

## 10. Struktur file

Disimpan di repositori ini pada branch `ccr-4e15e918-gs6r4i`, dalam folder terpisah dari kode TradingView MCP (bisa dipindah ke repositori sendiri nanti):

```
xauusd-ea/
  README.md                      (bahasa Indonesia: pasang, kompilasi, backtest, latih ulang)
  mql5/Experts/XauScalperML.mq5
  mql5/Include/XauScalper/*.mqh  (Signals, Baskets, RiskGate, Trailing, Filters, Features, MLFilter, State, Panel)
  mql5/Scripts/ExportXauData.mq5
  python/                        (features, labels, train, export_onnx, parity_check, tests, requirements.txt)
  models/                        (keluaran .onnx + laporan validasi)
```

## 11. Urutan pengerjaan

1. **EA inti** (bagian 6 dan 7), dengan filter ML sudah ada tempatnya tapi bisa dimatikan. Ini bisa langsung di-backtest pengguna.
2. **Skrip ekspor data dan pipeline Python** (bagian 8.2–8.5).
3. **Integrasi ONNX dan cek paritas** (bagian 8.6), lalu evaluasi kelulusan.

## 12. Diserahkan ke pembangun

- Struktur kode, nama kelas atau fungsi, dan pembagian file `.mqh`.
- Library ML dan konverter ONNX, selama lulus cek kesetaraan.
- Format file state dan log.
- Tampilan panel.
- Detail retry order dan penanganan kode galat broker.
- Default tambahan yang tidak disebut di sini, dengan syarat dicatat di README.
