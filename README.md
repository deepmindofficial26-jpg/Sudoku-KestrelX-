# SudoEngineProj — KestrelX Sudoku Engine Ecosystem

> **Nama engine: KestrelX**  
> KestrelX adalah nama engine pemecah Sudoku utama di dalam proyek ini. SudoEngineProj adalah nama repositori/ekosistem yang menaungi engine, image extractor, dan benchmark system.

> **Status proyek: MASIH DALAM PENGEMBANGAN**
>
> Repositori ini berisi ekosistem pemrosesan Sudoku: ekstraksi papan dari gambar, pemecahan puzzle, dan benchmark untuk membandingkan versi engine. Versi v10.6 dideskripsikan dalam `VersionRelease/VERSIONS.md` sebagai versi dengan keputusan yang lebih selektif dan efisien, tetapi repositori ini belum memberikan bukti rilis produksi atau hasil benchmark menyeluruh yang dapat dianggap sebagai sertifikasi stabil.

## Ringkasan

SudoEngineProj terdiri dari tiga bagian utama:

1. **Sudoku Image Extractor** — mendeteksi papan Sudoku dari gambar, melakukan koreksi perspektif, membaca digit dengan Tesseract OCR, lalu menghasilkan data teks 81 sel.
2. **KestrelX Sudoku Engine** — memvalidasi petunjuk, menghitung kandidat, menjalankan teknik propagasi logika dan backtracking, serta menyediakan inspeksi koordinat dan penghitungan solusi.
3. **Benchmark System** — menjalankan satu atau beberapa executable engine pada puzzle, mengumpulkan waktu dan statistik teknik, lalu menghitung metrik perbandingan.

## Struktur repositori

```text
SudoEngineProj/
├── GUIDE.md
├── STABLE_VER
├── UtilsHelper/
│   ├── Benchmark_Test_Engine.cpp
│   ├── Capture_Data_From_Image.cpp
│   └── setup_dependencies.sh
└── VersionRelease/
    ├── Engine_Ver_10_4.cpp
    ├── Engine_Ver_10_5.cpp
    ├── Engine_Ver_10_6.cpp
    └── VERSIONS.md
```

## Spesifikasi singkat file

| File | Fungsi |
|---|---|
| `GUIDE.md` | Panduan kompilasi dan penggunaan tiga modul. |
| `STABLE_VER` | File penanda versi stabil; pada arsip yang diperiksa file ini kosong, sehingga tidak menyatakan versi stabil tertentu. |
| `UtilsHelper/Capture_Data_From_Image.cpp` | Ekstraksi papan Sudoku dari gambar menggunakan OpenCV dan Tesseract OCR. |
| `UtilsHelper/Benchmark_Test_Engine.cpp` | Menjalankan benchmark pada executable engine dan membaca statistik pemecahan. |
| `UtilsHelper/setup_dependencies.sh` | Otomatisasi pemasangan tool dan dependensi untuk Ubuntu/Debian/WSL. |
| `VersionRelease/Engine_Ver_10_4.cpp` | Implementasi engine v10.4, berfokus pada optimasi jalur propagasi. |
| `VersionRelease/Engine_Ver_10_5.cpp` | Implementasi v10.5 dengan contradiction probe untuk menyaring kandidat yang berujung kontradiksi. |
| `VersionRelease/Engine_Ver_10_6.cpp` | Implementasi v10.6 dengan gerbang kelayakan probe, siklus keputusan dan propagasi ulang, serta deduplikasi antrean. |
| `VersionRelease/VERSIONS.md` | Catatan evolusi dan perbandingan rancangan v10.4–v10.6. |

## Alur penggunaan

1. Siapkan gambar papan Sudoku atau data puzzle teks.
2. Jika sumbernya gambar, jalankan extractor untuk menghasilkan data puzzle.
3. Kompilasi salah satu versi engine dan jalankan program.
4. Muat puzzle menggunakan `/load <file>`.
5. Jalankan `/calculate` untuk membangun cache koordinat dan kandidat.
6. Jalankan `/solve` untuk menyelesaikan puzzle.
7. Gunakan `/count` untuk menghitung kemungkinan solusi, atau `/hitungxy` untuk memeriksa sel berdasarkan koordinat.
8. Gunakan benchmark untuk membandingkan executable versi engine pada puzzle yang sama.

Detail perintah dan contoh interaksi terdapat di `GUIDE.md`.

## Status pengembangan dan batasan

Proyek ini **masih dalam pengembangan**. File versi dan catatan fitur menunjukkan perkembangan implementasi, tetapi tidak cukup untuk menyimpulkan bahwa seluruh sistem sudah tervalidasi untuk penggunaan produksi.

Hal-hal yang masih perlu diverifikasi atau dilengkapi:
- konsistensi instruksi kompilasi dengan versi OpenCV dan Tesseract yang benar-benar digunakan;
- ketersediaan dependensi dan kompatibilitas lintas distribusi/OS;
- pengujian end-to-end dari gambar sampai solusi;
- akurasi OCR pada variasi kualitas gambar, sudut, pencahayaan, dan jenis papan;
- validasi hasil solver pada kumpulan puzzle yang representatif;
- benchmark berulang dengan lingkungan dan dataset yang terdokumentasi;
- penetapan versi stabil secara eksplisit.

Catatan khusus: `STABLE_VER` kosong dalam arsip yang diperiksa. Karena itu README ini tidak menetapkan v10.6 sebagai rilis stabil resmi. `VERSIONS.md` memang menyebut v10.6 sebagai rancangan yang paling optimal dan hemat sumber daya di antara tiga versi yang didokumentasikan, tetapi klaim tersebut tetap perlu didukung pengujian yang dapat direproduksi.

## Rekomendasi langkah pengembangan

1. Tetapkan compiler, standar C++ (panduan menyebut C++17), versi OpenCV, dan versi Tesseract yang didukung.
2. Perbarui perintah build agar sesuai dengan nama file dan lokasi direktori aktual.
3. Buat kumpulan puzzle uji dengan tingkat kesulitan dan solusi yang diketahui.
4. Simpan hasil benchmark mentah beserta konfigurasi mesin, jumlah warm-up, jumlah run, dan versi executable.
5. Tambahkan pengujian untuk OCR yang hasilnya `Uncertain` dan alur pemeriksaan manual.
6. Setelah seluruh kriteria rilis dipenuhi, isi `STABLE_VER` dan dokumentasikan proses rilis.

## Catatan

Dokumen ini merangkum struktur dan fungsi yang tampak pada arsip sumber. Keberadaan fitur atau file bukan bukti bahwa semua skenario telah lulus pengujian. Gunakan `GUIDE.md` sebagai panduan operasional yang disediakan proyek dan `SUDOENGINE_SPECIFICATION.md` untuk rincian file.
