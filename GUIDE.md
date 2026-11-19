> **Catatan repository:** Perintah build di bawah menggunakan path sumber yang tersedia di repository ini. Sumber engine terbaru yang disertakan adalah `VersionRelease/Engine_Ver_10_6.cpp`; ini bukan pernyataan bahwa versi tersebut merupakan rilis stabil. Installer `UtilsHelper/setup_dependencies.sh` membangun OpenCV dari branch `5.x`.

# Panduan Penggunaan Ekosistem Sudoku Engine

Sistem ini terdiri dari tiga modul utama yang bekerja secara berurutan maupun mandiri:
1. **Sudoku Image Extractor** (Ekstraksi gambar papan Sudoku menjadi data teks).
2. **Sudoku Engine** (Pemecah utama, kalkulasi koordinat, dan validasi solusi).
3. **Sudoku Engine Benchmark System** (Pengujian performa dan perbandingan antar versi engine).

---

## 1. Persiapan Dependensi & Kompilasi

### 1.1 Jalankan setup dependensi terlebih dahulu (Ubuntu / Debian / WSL)

Sebelum mengompilasi program, jalankan installer dari direktori utama repository:

```bash
bash UtilsHelper/setup_dependencies.sh
```

Installer ini menyiapkan compiler dan tools build, membangun OpenCV dari branch `5.x`, serta memasang Tesseract OCR. Proses build OpenCV dapat memerlukan waktu lama, ruang penyimpanan cukup besar, dan RAM yang memadai.

**Peringatan sebelum menjalankan installer:**
- Script menjalankan `apt-get update/install`, menggunakan `sudo`, menginstal library ke `/usr/local`, menjalankan `ldconfig`, serta membersihkan cache paket. Tinjau isi script dan pastikan Anda menyetujui perubahan sistem tersebut.
- Jalankan sebagai pengguna biasa, **bukan** dengan `sudo bash ...`; script meminta hak sudo ketika diperlukan.
- Target yang dinyatakan script adalah Ubuntu/Debian/WSL. Script ini tidak diuji untuk Android/Termux atau macOS.
- Script membangun OpenCV dari source. Jangan menganggap dependensi sudah terpasang hanya karena script pernah dijalankan; periksa hasil verifikasi dan pesan error pada terminal.
- Nama modul `pkg-config` untuk OpenCV dapat berbeda menurut versi dan cara instalasi. Setelah instalasi, periksa dengan `pkg-config --list-all | grep -i opencv` dan gunakan nama modul yang benar ketika kompilasi.

### 1.2 Kompilasi modul

Lakukan langkah ini setelah installer selesai tanpa error dan compiler/dependensi sudah tersedia.

```bash
# 1. Kompilasi Image Extractor menggunakan header dan library OpenCV 5
g++ UtilsHelper/Capture_Data_From_Image.cpp -std=c++17 -O2 \\
  -I/usr/local/include/opencv5 -L/usr/local/lib \\
  -lopencv_core -lopencv_imgproc -lopencv_imgcodecs -lopencv_calib3d \\
  -o extractor

# 2. Kompilasi Sudoku Engine
g++ VersionRelease/Engine_Ver_10_6.cpp -std=c++17 -O3 -o engine

# 3. Kompilasi Benchmark System
g++ UtilsHelper/Benchmark_Test_Engine.cpp -std=c++17 -O3 -o benchmark
```

Perintah Image Extractor di atas menggunakan konfigurasi OpenCV 5 yang dicantumkan oleh installer: header pada `/usr/local/include/opencv5` dan library pada `/usr/local/lib`. Nama file sumber dan executable disesuaikan dengan struktur repository ini. Jika instalasi OpenCV 5 Anda menggunakan direktori berbeda, sesuaikan path include dan library dengan lokasi instalasi tersebut.
---

## 2. Modul 1: Sudoku Image Extractor (`./extractor`)

Modul ini mengubah foto atau tangkapan layar papan Sudoku menjadi string 81 digit yang siap diproses oleh Engine.

### Langkah Penggunaan
1. Jalankan program:
   ```bash
   ./extractor
   ```
2. Masukkan lokasi file gambar saat diminta:
   ```text
   Input image path: soal_sudoku.jpg
   ```
3. Masukkan nama file output (ekstensi `.txt` akan ditambahkan otomatis jika tidak ditulis):
   ```text
   Output file name: puzzle1.txt
   ```

### Output Yang Dihasilkan
* `puzzle1.txt`: Berisi 81 deret angka (`0` untuk kotak kosong, `1-9` untuk angka terisi).
* `puzzle1_grid.png`: Gambar visualisasi deteksi garis kotak dan koreksi perspektif.
* `puzzle1_hasil.png`: Gambar hasil pembacaan OCR pada setiap sel untuk memudahkan pengecekan manual jika ada sel yang gagal terbaca (`Uncertain`).

### Peringatan akurasi OCR dan pemeriksaan statistik

**Hasil ekstraksi OCR tidak otomatis dijamin benar.** Setelah proses selesai, bandingkan `puzzle1.txt` dan gambar review `puzzle1_hasil.png` dengan gambar Sudoku asli. Pastikan setiap angka berada di baris dan kolom yang benar.

Jika satu atau beberapa angka hasil ekstraksi tidak sesuai dengan gambar sumber:

1. Jangan langsung menggunakan file hasil tersebut untuk menyelesaikan Sudoku.
2. Periksa `puzzle1_grid.png` untuk memastikan papan terdeteksi dan dipotong dengan benar.
3. Periksa `puzzle1_hasil.png` untuk mencocokkan pembacaan setiap sel dengan gambar asli.
4. Gunakan gambar sumber yang lebih jelas dan dekat: perbesar/crop bagian papan Sudoku agar memenuhi area gambar, pastikan seluruh 9×9 kotak tetap terlihat, tidak terpotong, tidak miring berlebihan, dan memiliki pencahayaan yang cukup.
5. Jalankan kembali extractor dengan gambar yang sudah diperbaiki. Jika masih ada perbedaan, koreksi data puzzle secara manual sebelum memuatnya ke Engine.

#### Cara membaca Extraction Report

Laporan menampilkan `Total cells`, `Empty cells`, `Recognized`, dan `Uncertain`. Untuk satu papan Sudoku, `Total cells` seharusnya **81**, dan secara hitungan `Empty + Recognized + Uncertain` harus sama dengan 81.

Perhatikan nilai yang tidak wajar, misalnya:
- `Total cells` bukan 81;
- `Uncertain` tinggi atau ada banyak sel yang ditandai perlu koreksi;
- `Recognized` = 0 padahal gambar jelas berisi banyak angka;
- hampir semua sel terbaca kosong (`Empty` mendekati 81) padahal papan sumber terisi;
- jumlah `Recognized` atau `Empty` sangat berbeda dari perkiraan saat melihat gambar asli.

Angka statistik tersebut hanya ringkasan jumlah sel, **bukan bukti bahwa OCR benar**. Bahkan jika `Uncertain` = 0, OCR masih mungkin salah mengenali angka. Selalu lakukan pemeriksaan visual dan validasi puzzle sebelum menjalankan solver.

---

## 3. Modul 2: Sudoku Engine (`./engine`)

Modul interaktif untuk memuat puzzle, memetakan kandidat dalam sistem koordinat Kartesius, memecahkan puzzle, dan mengecek keunikan solusi.

### Alur Kerja Dasar
1. Jalankan engine:
   ```bash
   ./engine
   ```
2. Muat file data puzzle:
   ```text
   /load puzzle1.txt
   ```
   *Engine akan memvalidasi keabsahan petunjuk awal, membuat `Session ID`, dan mencatat memori persisten (`.mem`) untuk mendeteksi perubahan file di masa mendatang.*
3. Hitung database koordinat & kandidat awal:
   ```text
   /calculate
   ```
   *Langkah ini wajib dilakukan sebelum menjalankan `/solve` (jika belum ada, engine akan menawarkan pembuatan otomatis).*
4. Pecahkan puzzle:
   ```text
   /solve
   ```

### Daftar Perintah Lengkap

| Perintah | Fungsi | Keterangan |
| :--- | :--- | :--- |
| `/load <file>` | Memuat data puzzle | Membaca 81 digit pertama dari file teks. Otomatis membuat atau membaca `[id_session]`. |
| `/calculate` | Membangun cache koordinat | Menyimpan pemetaan titik Kartesius `(-4..4)` dan kandidat awal ke `data_koordinat_<id>.txt`. |
| `/solve` | Memecahkan Sudoku | Menjalankan propagasi logika & backtracking. Menyimpan hasil lengkap ke `data_solve_<id>.txt`. |
| `/count` | Validasi keunikan solusi | Menghitung seluruh kemungkinan solusi (maksimal 9999) untuk memastikan puzzle valid (1 solusi unik). |
| `/hitungxy` | Inspeksi sel spesifik | Meminta input koordinat `X` dan `Y` (`-4` hingga `4`), lalu menampilkan isi baris, kolom, dan kandidatnya. |
| `/seesolve` | Tampilkan papan solusi | Mengubah tampilan papan di layar menjadi hasil akhir setelah `/solve` berhasil. |
| `/back` | Kembali ke papan awal | Mengembalikan tampilan papan di layar ke kondisi soal asli. |
| `/quit` | Keluar dari program | Menutup sesi engine. |

---

## 4. Modul 3: Benchmark System (`./benchmark`)

Modul otomatis untuk mengukur kecepatan eksekusi (dalam milidetik & mikrodetik), beban kerja solver, serta dampak tiap teknik logika pada satu atau beberapa versi engine sekaligus.

### Langkah Penggunaan
1. Jalankan program benchmark:
   ```bash
   ./benchmark
   ```
2. Daftarkan executable engine yang ingin diuji (bisa lebih dari satu untuk perbandingan):
   ```text
   Masukkan file executable engine contoh ./main ./engine, atau ./main_ver #1: ./engine_A
   Tambahkan engine lain? (y/n): y
   Masukkan file executable engine contoh ./main ./engine, atau ./main_ver #2: ./engine_B
   Tambahkan engine lain? (y/n): n
   ```
3. Masukkan file data puzzle untuk masing-masing engine:
   ```text
   Masukkan data puzzle untuk Engine #1 (./engine_A): puzzle1.txt
   Masukkan data puzzle untuk Engine #2 (./engine_B): puzzle1.txt
   ```
4. Tentukan jumlah iterasi pemanasan (*warm-up*) dan pengujian utama (*runs*):
   ```text
   Jumlah warm-up: 5
   Jumlah benchmark runs: 100
   ```
5. Ketik `continue` untuk memulai proses pengujian otomatis:
   ```text
   Mulai benchmark? (ketik 'continue'): continue
   ```

### Mekanisme & Hasil Analisis
* **Pre-Flight Check:** Sistem otomatis memeriksa keberadaan `Session ID` dan file `data_koordinat_<id>.txt`. Jika belum tersedia, benchmark akan menjalankan perintah `/calculate` secara otomatis sebelum memulai pengukuran waktu.
* **Metrik Utama:** Menampilkan waktu *Minimum*, *Median* (indikator utama kestabilan), *Mean*, *Maximum*, dan *Standard Deviation*.
* **Core Workload & Technique Impact:** Menampilkan nilai median untuk jumlah propagasi, eliminasi kandidat, tebakan (*backtracking*), *deadend*, serta waktu eksekusi dan jumlah panggilan per teknik logika.
