# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Purpose

Student assessment starter repo for **Assignment 2 — Mandelbrot two-variant** (`parallel_for` + `tasks`, compared via `CHOICE.md`) in the "Shared Memory Programming with OpenMP" course (2026 refresh). 30 marks. Iterates only the upper half (j ∈ [0, NPOINTS/2)) and doubles by symmetry around the real axis.

Sibling repos for the other two assignments (released on a staged schedule):

- `ppp-openmp-assessment-1` — numerical integration (parallel-for + reduction + schedule choice)
- `ppp-openmp-assessment-3` — 3D Jacobi stencil (core + chosen extension)

**Public** companion: lectures repo `~/projects/ppp-openmp` (slides, snippets, brief, handouts).
**Private** companion: `~/projects/ppp-openmp-grader` (instructor only — reference solutions, MCQ keys, grader script).

Grading is summative on a CX3 login node via `grade_cohort.sh` in the private grader repo. Formative CI runs on every student push on GH-hosted Ubuntu (no secrets, no self-hosted runner). **Students modify only the designated C++ source files** — do not rename files, add new headers/libraries, or modify `.github/workflows/` (overwritten at grading time).

## Build (laptop)

```bash
cmake -B build -S . -DCMAKE_CXX_COMPILER=clang++-18    # or equivalent Clang ≥ 16
cmake --build build -j
```

Produces `build/mandelbrot_for` and `build/mandelbrot_tasks`.

## Local correctness check

Both binaries print a single reproducible line to stdout (the correctness token). Timing is measured by the hyperfine harness into JSON, NOT printed to stdout.

```bash
OMP_NUM_THREADS=4 ./build/mandelbrot_for > out.txt
python3 bin/smart_diff.py out.txt expected_output.txt
```

Reference correctness token (post-symmetry refactor):

```
outside = 20807388
area = 7.490660
```

## What students submit

- `mandelbrot_for.cpp`, `mandelbrot_tasks.cpp` — student implementations.
- `questions.md` — 15 MCQ (read-only; answers auto-graded).
- `answers.csv` — student fills in `qid,answer` (A/B/C/D per row).
- `REFLECTION.md` — fixed-format reflection. CI checks header presence + ≥ 50 words per required section.
- `tables.csv` — measured times + speedup + efficiency. CI checks per-row internal consistency only.
- `CHOICE.md` — YAML-header block declaring the recommended variant. Graded against the **student's own** committed `perf-results-a2.json`, not against canonical times.

## CI workflow (`.github/workflows/ci.yml` — formative only, GH-hosted Ubuntu)

A single workflow with five parallel jobs:

- **`lint`** — clang-format-20 + clang-tidy-20 + cppcheck against `.clang-format` and `.clang-tidy`.
- **`correctness`** — Clang-18 + TSan + Archer OMPT. Builds **both** variants and exercises each at `{1, 2, 4, 8, 16}` threads. Any `^(WARNING|ERROR): ThreadSanitizer` line fails the run.
- **`reflection-format`** — formative format-only check on `REFLECTION.md` headers + word counts.
- **`language-check`** — non-English content detector on `*.md` and C++ comments.
- **`generated-files-check`** — fails if `.o`, executables, `build/`, or other artifacts are committed.

There is **no** `assessment.yml` and **no self-hosted runner**. Summative grading is run by the instructor on CX3 from the private grader repo.

## Naming note: `mandelbrot_for` vs `parallel_for`

The CMake target / binary name is `mandelbrot_for` (short for "parallel for"). The semantic label used in `CHOICE.md`'s `recommended:` field is `parallel_for`. So:

- Binary: `build/mandelbrot_for` ←→ CHOICE.md value: `parallel_for`
- Binary: `build/mandelbrot_tasks` ←→ CHOICE.md value: `tasks`

This dual naming is inherited from the source repo and intentionally documented; do not rename the targets.

## Rome perf harness (`evaluate.pbs`)

Run at canonical thread ladder **`{1, 16, 64, 128}`** on Rome (dual-socket EPYC 7742, 128 physical cores, 8 NUMA domains). Times **both** variants. Produces `perf-results-a2.json`.

Timer: `hyperfine --warmup 1 --min-runs 3 --export-json`. Build flags: `-O3 -march=znver2 -mavx2 -fopenmp`.

Students may run this on their own CX3 accounts to populate `tables.csv` and the evidence base for `CHOICE.md`; this is encouraged but not required. The summative score uses the instructor's canonical re-run.

## Shared lint config

`.clang-format` and `.clang-tidy` are kept in lockstep with the lectures repo and the other two assessment repos. When editing, keep them identical across all four (students pre-commit on the assessment repos, grader enforces on all).

## Hardware facts (canonical constants — see lectures-repo `docs/rome-inventory.md`)

- 128 physical cores per node (2 sockets × 64 cores), SMT off
- 8 NUMA domains per node (NPS=4 per socket, 16 cores per domain)
- Peak DP 4608 GFLOPs (AVX2 FMA, theoretical); HPL-achievable 2896 GFLOPs ≈ 63 % (foss/2024a + OpenBLAS, measured 2026-04-26)
- STREAM triad ceiling 246.2 GB/s (one thread per CCX); 231.5 GB/s at 128 threads full-node; 116.0 GB/s on one socket (measured 2026-04-26 v3)
- Intra-socket node distance 12, cross-socket 32
- Canonical thread ladder `{1, 16, 64, 128}`
