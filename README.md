<p align="center">
  <img src="docs/assets/kestrelx-logo.png" alt="KestrelX logo" width="156">
</p>

<h1 align="center">KestrelX</h1>

<p align="center">
  <strong>A C++ Sudoku-solving engine with image extraction and performance benchmarking tools.</strong><br>
  From a Sudoku image or text puzzle to candidate analysis, solving, and version comparison.
</p>

<p align="center">
  <img src="https://img.shields.io/badge/License-MIT-green.svg" alt="License: MIT">
  <img src="https://img.shields.io/badge/C%2B%2B-17%2B-blue.svg" alt="C++17 or newer">
  <img src="https://img.shields.io/badge/Engine-v10.6-informational.svg" alt="Latest documented engine source: v10.6">
  <img src="https://img.shields.io/badge/Status-Under%20Development-orange.svg" alt="Under development">
  <img src="https://img.shields.io/badge/OCR-Tesseract%20%2B%20OpenCV-8a2be2.svg" alt="Tesseract and OpenCV">
</p>

---

**KestrelX** is the Sudoku-solving engine in the `Sudoku-KestrelX-` repository. The repository also contains two supporting utilities: an image-to-puzzle extractor and a benchmark runner for comparing engine executables.

The documented engine line currently includes **v10.4**, **v10.5**, and **v10.6**. The v10.6 source is the newest version represented in this repository; this does **not** mean it has been declared a stable production release. The project is still under active development.

## Project overview

| Component | Responsibility |
|---|---|
| **KestrelX Sudoku Engine** | Loads puzzle text, validates givens, calculates candidate/coordinate data, solves Sudoku, counts solutions, and supports cell inspection. |
| **Sudoku Image Extractor** | Uses OpenCV image processing and Tesseract OCR to detect a Sudoku board and produce an 81-cell text puzzle, alongside visual review images. |
| **Benchmark System** | Runs one or more engine executables against puzzle files and reports timing and solver-workload statistics. |

## Architecture

The diagram below shows the intended workflow and the separate benchmark path. The image-extraction stage is optional when a puzzle is already available as text.

```mermaid
flowchart TD
    U["Puzzle input"] --> SRC{"Input format?"}
    SRC -->|"Image / screenshot"| IMG["Sudoku Image Extractor"]
    IMG --> OCR["Board detection + perspective correction"]
    OCR --> TESS["Tesseract OCR"]
    TESS --> REVIEW["Review uncertain OCR cells"]
    REVIEW --> TXT["81-cell puzzle text file"]
    SRC -->|"Existing text puzzle"| TXT

    TXT --> BUILD["Compile selected KestrelX version"]
    BUILD --> LOAD["/load puzzle.txt"]
    LOAD --> CALC["/calculate candidates + coordinates"]
    CALC --> SOLVE["/solve"]
    SOLVE --> RESULT["Solved board + solve data"]
    CALC --> TOOLS["/count · /hitungxy · /seesolve · /back"]

    subgraph BENCH["Benchmark workflow"]
      VERS["Engine executables: v10.4 / v10.5 / v10.6"] --> RUN["Benchmark runner"]
      TXT --> RUN
      RUN --> METRICS["Timing + workload statistics"]
    end
```

### How the workflow operates

1. **Provide a puzzle.** Start with an image or a text file containing the puzzle data.
2. **Extract, if needed.** The image extractor detects the board, corrects perspective, and reads the cells with OCR. Inspect the generated review image when a cell is marked uncertain or the extracted digits do not match the source.
3. **Build an engine executable.** Choose one of the versioned C++ source files.
4. **Load and calculate.** Use `/load` followed by `/calculate` to load the puzzle and prepare coordinate/candidate data.
5. **Solve and inspect.** Run `/solve`; use the other interactive commands to inspect or validate the result.
6. **Benchmark versions.** Run the benchmark utility with the same puzzle data and selected engine executables to compare their reported measurements.

## Quick start

### 1. Requirements

- A C++17-compatible compiler such as `g++`.
- Linux, Ubuntu/Debian, or WSL is the documented setup target for the helper installer and benchmark runner.
- OpenCV and Tesseract OCR for the image extractor.
- `pkg-config` for the convenient OpenCV build command, where the installed OpenCV package provides a matching `.pc` file.

The dependency script in this repository builds OpenCV from its `5.x` branch and installs Tesseract. OpenCV package names and `pkg-config` metadata can vary by installation; check the installed environment before compiling.

### 2. Install image-extractor dependencies (optional)

Run from the repository root:

```bash
bash UtilsHelper/setup_dependencies.sh
```

The script targets Ubuntu/Debian/WSL and uses `sudo` internally when required. Read the script before running it and confirm that its package and system changes are appropriate for your environment.

### 3. Compile KestrelX

For the newest source included in this repository:

```bash
g++ -std=c++17 -O3 VersionRelease/Engine_Ver_10_6.cpp -o kestrelx
```

To build another documented version, replace the source filename:

```bash
g++ -std=c++17 -O3 VersionRelease/Engine_Ver_10_5.cpp -o kestrelx-10.5
g++ -std=c++17 -O3 VersionRelease/Engine_Ver_10_4.cpp -o kestrelx-10.4
```

### 4. Compile the image extractor (optional)

If `pkg-config` can locate your OpenCV installation:

```bash
g++ -std=c++17 -O3 UtilsHelper/Capture_Data_From_Image.cpp \
  -o kestrelx-extractor \
  $(pkg-config --cflags --libs opencv4)
```

If your OpenCV installation exposes a different package name, use the corresponding package name reported by `pkg-config --list-all` and the include/library paths for that installation. The repository's setup script builds OpenCV 5, while the exact `pkg-config` module name may differ by packaging; this command may need adjustment for your environment.

### 5. Compile the benchmark runner

```bash
g++ -std=c++17 -O3 UtilsHelper/Benchmark_Test_Engine.cpp -o kestrelx-benchmark
```

The benchmark implementation uses POSIX process APIs, so the documented helper is intended for Unix-like environments.

## Using the engine

Start the executable:

```bash
./kestrelx
```

Then use the interactive commands:

| Command | Purpose |
|---|---|
| `/load <file>` | Load a text puzzle file and initialize its session data. |
| `/calculate` | Build coordinate and candidate data. Run this before solving if it has not already been prepared. |
| `/solve` | Run the solving process and save the solve output. |
| `/count` | Count possible solutions, up to the engine's configured limit. |
| `/hitungxy` | Inspect a cell using the engine's X/Y coordinate convention. |
| `/seesolve` | Display the solved board after a successful solve. |
| `/back` | Return the display to the original puzzle. |
| `/quit` | Exit the engine. |

See [`GUIDE.md`](GUIDE.md) for the detailed interactive workflow and expected output files.

## Image extraction

Build and run the extractor after installing OpenCV:

```bash
./kestrelx-extractor
```

The program prompts for an image path and an output filename. According to the project guide, it produces:

- a text puzzle file containing 81 cell values (`0` represents an empty cell);
- a `*_grid.png` review image showing board/grid detection and perspective correction;
- a `*_hasil.png` review image showing OCR results for individual cells.

OCR is not guaranteed to be correct for every image. Check the generated review image and compare the extracted puzzle with the original image before solving, especially when the source is small, blurry, tilted, or poorly lit.

## Benchmarking

Build and run the benchmark utility:

```bash
./kestrelx-benchmark
```

The interactive runner asks for engine executable paths, puzzle files, warm-up count, and benchmark run count. It performs a pre-flight check for the session/coordinate data and reports measurements such as minimum, median, mean, maximum, and standard deviation, along with solver-workload statistics described in `GUIDE.md`.

For meaningful comparisons, use the same puzzle inputs and record the machine, compiler, build flags, warm-up count, and run count. Benchmark output can vary with hardware, compiler, build configuration, and system load.

## Version history

| Version | Codename | Main documented focus |
|---|---|---|
| **v10.4** | Hot Path Optimization | Bitmask-based candidate handling and incremental propagation-oriented performance work. |
| **v10.5** | Adaptive MRV Contradiction Probe | Probe candidate choices to detect contradictions before branching. |
| **v10.6** | Precision Decision Cycle | Selective probe eligibility, re-propagation after successful pruning, and queue deduplication. |

See [`VersionRelease/VERSIONS.md`](VersionRelease/VERSIONS.md) for the source-level version notes. These descriptions summarize the project's documented design; they are not independent performance guarantees.

## Repository layout

```text
.
├── README.md
├── GUIDE.md
├── SPEC.md
├── LICENSE
├── .gitignore
├── STABLE_VER
├── docs/
│   └── assets/
│       └── kestrelx-logo.png
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

## Development status

<details>
<summary><strong>Under development — current limitations and next steps</strong></summary>

KestrelX is an active development project. The presence of versioned source files and helper utilities does not establish that the complete image-to-solution pipeline has passed a comprehensive validation process.

Items to verify and improve include:

- document the supported compiler, C++ standard, OpenCV version, and Tesseract version consistently;
- align build commands with the actual installed OpenCV package metadata;
- add repeatable end-to-end tests from image extraction through solving;
- evaluate OCR accuracy across image sizes, lighting, perspective, and board styles;
- validate solver correctness on a documented set of puzzles with known solutions;
- store raw benchmark results with environment and build details;
- define explicit release criteria and update `STABLE_VER` only when a stable version is formally selected.

`STABLE_VER` is empty in the supplied project archive. Therefore, this README does not declare v10.6 (or any other version) as a stable release.

</details>

## Contributing

Contributions and issue reports are welcome. When proposing a change:

1. Describe the affected component (engine, image extractor, or benchmark system).
2. Include a minimal reproducible puzzle or image example when possible.
3. Record compiler, dependency versions, and build flags.
4. Include before/after benchmark data for performance-related changes.
5. Avoid committing generated binaries, private data, or unrelated local files.

## License

This project is licensed under the **MIT License**. See [`LICENSE`](LICENSE) for the full text.

---

<p align="center">
  <sub>Developed as part of the DeepMindTeam project ecosystem.</sub>
</p>
