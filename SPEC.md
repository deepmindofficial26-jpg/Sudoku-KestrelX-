# SudoEngineProj — Spesifikasi File dan Status Pengembangan

**Jenis dokumen:** Inventaris teknis repositori  
**Status proyek:** **MASIH DALAM PENGEMBANGAN**  
**Dasar penyusunan:** Pemeriksaan isi arsip `SudoEngineProj.zip`.

## 1. Tujuan dan ruang lingkup

SudoEngineProj merupakan ekosistem Sudoku yang menggabungkan tiga kemampuan:
1. ekstraksi data Sudoku dari gambar;
2. pemecahan puzzle dengan engine C++;
3. pengukuran dan perbandingan performa antar executable engine.

Ketiga kemampuan dapat digunakan secara berurutan: gambar diproses menjadi data puzzle, data dimuat oleh engine untuk dihitung solusinya, kemudian executable engine dapat diuji menggunakan sistem benchmark.

## 2. Arsitektur tingkat tinggi

### 2.1 Sudoku Image Extractor

`Capture_Data_From_Image.cpp` memuat gambar, mendeteksi grid, mengoreksi perspektif papan, memproses sel, dan mengenali digit menggunakan Tesseract OCR. Hasil utamanya adalah data teks berisi 81 karakter untuk 9×9 sel. Nilai `0` digunakan untuk sel kosong. Program juga menghasilkan gambar debug agar pembacaan OCR dapat diperiksa.

### 2.2 Sudoku Engine

Versi engine pada `VersionRelease/` merupakan implementasi solver yang menggunakan kandidat berbasis bitmask. Source code memuat validasi petunjuk awal, inisialisasi kandidat, teknik propagasi, pemilihan sel MRV, backtracking, statistik teknik, dan antarmuka perintah interaktif.

Panduan proyek menyebut alur `/load`, `/calculate`, dan `/solve`, dengan perintah tambahan untuk menghitung solusi, memeriksa koordinat, serta berpindah tampilan papan.

### 2.3 Benchmark System

`Benchmark_Test_Engine.cpp` mengorkestrasi executable engine melalui proses terpisah, mengirim perintah interaktif seperti `/calculate` dan `/solve`, membaca keluaran, lalu mengumpulkan metrik waktu dan statistik teknik. Benchmark mendukung beberapa engine dan beberapa iterasi pengujian.

## 3. Spesifikasi setiap file

| File | Spesifikasi dan tanggung jawab |
|---|---|
| `GUIDE.md` | Panduan pengguna berbahasa Indonesia. Menjelaskan prasyarat, contoh kompilasi tiga modul, alur extractor, daftar perintah engine, dan proses benchmark. |
| `STABLE_VER` | File penanda versi stabil. File ini kosong pada arsip yang diterima; tidak ada nomor versi atau metadata stabil yang dapat dibaca. |
| `UtilsHelper/Capture_Data_From_Image.cpp` | Program extractor berbasis OpenCV. Kelas `SudokuExtractor` menangani pemuatan gambar, deteksi grid, koreksi perspektif, visualisasi grid, persiapan gambar sel, deteksi sel kosong, pemanggilan OCR, ekstraksi 81 sel, serta penyimpanan hasil dan gambar debug. |
| `UtilsHelper/Benchmark_Test_Engine.cpp` | Program benchmark. Mendefinisikan snapshot statistik teknik dan solve, membaca Session ID dari file puzzle, memvalidasi file koordinat, berkomunikasi dengan proses engine melalui pipe, menjalankan fase calculate/solve, mem-parsing statistik teknik, serta menghitung ringkasan hasil seperti median dan statistik waktu. |
| `UtilsHelper/setup_dependencies.sh` | Script instalasi untuk Ubuntu/Debian/WSL. Memasang tool build dan library gambar, membangun OpenCV dari branch `5.x`, memasang Tesseract OCR, memverifikasi instalasi, dan menampilkan contoh langkah kompilasi. Script menolak dijalankan langsung sebagai root dan meminta `sudo` saat diperlukan. |
| `VersionRelease/Engine_Ver_10_4.cpp` | Source engine v10.4. Catatan versi menyebut fokus pada *Hot Path Optimization*: kandidat bitmask `uint16_t`, cache posisi/unit, tabel posisi lokal, dan bitmask worklist `dirtyUnits`. Teknik yang tercantum meliputi Naked Single, Hidden Single, Locked Candidates, Naked Subsets, X-Wing, dan backtracking MRV + LCV. |
| `VersionRelease/Engine_Ver_10_5.cpp` | Source engine v10.5. Menambahkan `probeContradiction()` yang menguji kandidat sementara melalui beberapa tahap pemeriksaan, termasuk eliminasi peer, rantai Naked Single, dan validasi unit. Tujuan rancangan adalah menghapus kandidat yang terbukti kontradiktif sebelum percabangan tebakan. |
| `VersionRelease/Engine_Ver_10_6.cpp` | Source engine v10.6. Menambahkan gerbang kelayakan contradiction probe berdasarkan metadata MRV dan kepadatan kandidat, decision cycle yang menjalankan propagasi incremental kembali setelah eliminasi kandidat, serta deduplikasi antrean melalui `queued[81]`. |
| `VersionRelease/VERSIONS.md` | Catatan evolusi tiga versi. Menyebut v10.4 sebagai fokus kecepatan propagasi, v10.5 sebagai heuristic pruning, dan v10.6 sebagai precision decision cycle yang dirancang lebih selektif dan hemat sumber daya. |

## 4. Rincian modul extractor

Source extractor menunjukkan komponen berikut:
- `loadImage()` — membaca file gambar menggunakan OpenCV.
- `detectGrid()` — mencari area/grid papan Sudoku.
- `correctPerspective()` — melakukan transformasi perspektif pada papan.
- `drawGrid()` dan `saveGridDebug()` — membuat visualisasi untuk pemeriksaan.
- `prepareOcrImage()` — menyiapkan citra sel untuk pengenalan karakter.
- `isEmptyCell()` — menentukan apakah sel dianggap kosong.
- `recognizeFromImage()` — menjalankan pengenalan digit pada citra sel.
- `extractCells()` — memproses sel-sel papan dan membentuk hasil data.
- `saveResultDebug()` — menyimpan visualisasi hasil pembacaan.

Konstanta source menunjukkan grid 9×9 dan ukuran papan kerja 900. Source juga mengizinkan penyimpanan ketika terdapat sel `Uncertain`; oleh sebab itu hasil ekstraksi perlu diperiksa, terutama jika kualitas gambar rendah.

## 5. Rincian engine dan evolusi versi

### 5.1 Fondasi solver yang terlihat di ketiga versi

Ketiga source engine menggunakan struktur `Kandidat` berbasis `uint16_t` dengan operasi:
- memeriksa apakah digit ada dalam mask;
- menambahkan dan menghapus kandidat;
- menghitung jumlah kandidat;
- mendeteksi mask kosong;
- mengambil kandidat digit pertama.

Source juga memuat fungsi-fungsi seperti validasi petunjuk awal, inisialisasi kandidat, undo/trail, pemetaan koordinat Kartesius ke indeks, pemilihan sel MRV, penghitungan dampak eliminasi LCV, validasi koordinat, serta pencetakan statistik dan timing teknik.

### 5.2 v10.4 — Hot Path Optimization

Menurut `VERSIONS.md`, v10.4 mengutamakan kecepatan propagasi murni tanpa fase look-ahead. Rancangan yang dicatat:
- kandidat berbasis bitmask;
- UDP (Unit-Digit Position) cache;
- tabel `LOCAL_POS`;
- bitmask worklist `dirtyUnits`;
- teknik Hidden Single, Locked Candidates, dan X-Wing;
- backtracking MRV + LCV setelah propagasi logika.

Catatan versi menyebut pendekatan ini dapat menggunakan sumber daya lebih besar saat backtracking dalam.

### 5.3 v10.5 — Adaptive MRV Contradiction Probe

v10.5 menambahkan `probeContradiction()`. Berdasarkan catatan versi, probe dilakukan sementara pada sel MRV dengan dua atau tiga kandidat dan memeriksa beberapa tingkat konsekuensi. Kandidat yang terbukti menghasilkan jalan buntu dapat dihapus sebelum percabangan tebakan.

Catatan versi juga menyebut risiko *over-probing* karena belum ada filter kepadatan sel yang selektif.

### 5.4 v10.6 — Precision Decision Cycle

v10.6 menambahkan tiga mekanisme yang dicatat:
- **Probe Eligibility Gate (`layakContradictionProbe`):** membatasi probe pada kondisi MRV/kepadatan tertentu; catatan versi memberi contoh dua kandidat dengan `degree >= 8`, atau tiga kandidat dengan `degree >= 12`.
- **Decision Cycle Loop & Re-Propagation:** menjalankan kembali `propagasiIncremental()` setelah probe berhasil menghapus kandidat.
- **Queue Deduplication (`queued[81]`):** mencegah sel yang sama masuk antrean berkali-kali selama cascade Naked Single di dalam probe.

`VERSIONS.md` menilai v10.6 sebagai rancangan paling optimal dan hemat sumber daya di antara versi yang dicatat. Ini merupakan klaim dalam dokumentasi proyek; hasil benchmark terukur tidak disertakan dalam arsip yang diperiksa.

## 6. Rincian benchmark

Source benchmark mendefinisikan enam kategori teknik yang dipantau:
- Naked Single;
- Hidden Single;
- Locked Candidates;
- Naked Subsets;
- X-Wing;
- Contradiction Probe.

Snapshot per teknik mencatat waktu, jumlah pemanggilan, jumlah pola, kandidat yang dihapus, dan sel yang terisi. Snapshot solve mencatat waktu, propagasi, eliminasi, tebakan, deadend, retry, serta validitas hasil.

Panduan menyebut metrik ringkasan Minimum, Median, Mean, Maximum, dan Standard Deviation. Median disebut sebagai indikator utama kestabilan. Benchmark juga menampilkan dampak teknik melalui statistik pemanggilan dan beban kerja.

## 7. Perintah engine yang didokumentasikan

| Perintah | Fungsi menurut `GUIDE.md` |
|---|---|
| `/load <file>` | Memuat data puzzle, membaca 81 digit pertama, memvalidasi petunjuk, dan membuat/membaca Session ID. |
| `/calculate` | Membangun cache koordinat Kartesius `(-4..4)` dan kandidat awal. |
| `/solve` | Menjalankan propagasi logika dan backtracking serta menyimpan hasil solve. |
| `/count` | Menghitung kemungkinan solusi hingga batas 9999 untuk memeriksa keunikan. |
| `/hitungxy` | Memeriksa sel pada koordinat X dan Y dari -4 sampai 4. |
| `/seesolve` | Menampilkan papan hasil setelah solve berhasil. |
| `/back` | Mengembalikan tampilan ke papan puzzle awal. |
| `/quit` | Menutup sesi engine. |

## 8. Status pengembangan saat ini

**SudoEngineProj masih dalam pengembangan.**

Dasar penetapan status ini:
1. Arsip berisi tiga versi source engine dan catatan evolusi, tetapi tidak berisi laporan hasil pengujian menyeluruh yang dapat diverifikasi.
2. `STABLE_VER` kosong, sehingga tidak ada versi stabil resmi yang dinyatakan melalui file penanda tersebut.
3. Script setup menargetkan Ubuntu/Debian/WSL dan membangun OpenCV dari branch `5.x`, sedangkan contoh kompilasi di `GUIDE.md` menggunakan nama file generik dan `pkg-config opencv4`. Kesesuaian instruksi tersebut perlu diselaraskan dan diuji pada lingkungan target.
4. Panduan extractor mengakui kemungkinan hasil OCR tidak sesuai dan menyarankan gambar papan yang besar serta pemeriksaan hasil.
5. Benchmark dirancang untuk mengukur performa, tetapi arsip tidak menyertakan hasil benchmark terukur yang membuktikan klaim performa lintas puzzle dan lingkungan.

Dengan demikian, v10.6 dapat disebut sebagai versi terbaru yang terdapat dalam arsip dan versi yang dinilai paling optimal dalam `VERSIONS.md`, tetapi **belum boleh disebut rilis stabil/produksi resmi hanya berdasarkan arsip ini**.

## 9. Rekomendasi pekerjaan lanjutan

1. **Build reproducibility:** tetapkan versi compiler, OpenCV, Tesseract, dan instruksi kompilasi per file dengan path yang benar.
2. **OCR test set:** uji beragam ukuran gambar, pencahayaan, perspektif, ketebalan garis, dan font digit; catat akurasi per sel serta jumlah `Uncertain`.
3. **Solver correctness:** siapkan puzzle dengan solusi diketahui, puzzle tidak valid, puzzle tanpa solusi, dan puzzle dengan banyak solusi.
4. **Benchmark protocol:** dokumentasikan CPU, OS, compiler flags, puzzle input, warm-up, jumlah run, serta hasil mentah.
5. **Version comparison:** bandingkan v10.4, v10.5, dan v10.6 pada dataset identik; jangan menyimpulkan versi tercepat hanya dari label/catatan desain.
6. **Stable release criteria:** definisikan syarat kelulusan dan isi `STABLE_VER` hanya setelah syarat tersebut terpenuhi.
7. **Documentation alignment:** pastikan `GUIDE.md`, script setup, nama binary, versi OpenCV, dan perintah build konsisten.

## 10. Batasan dokumen

Dokumen ini adalah inventaris berbasis source dan dokumentasi yang tersedia di arsip. Dokumen ini tidak menjalankan seluruh program, tidak menyatakan seluruh file berhasil dikompilasi, dan tidak mengesahkan akurasi OCR, kebenaran solver untuk semua puzzle, atau klaim performa. Klaim desain dari `VERSIONS.md` dibedakan dari hasil yang sudah dibuktikan melalui benchmark.
