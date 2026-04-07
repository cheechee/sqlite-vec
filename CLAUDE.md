# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

sqlite-vec is a pure C SQLite extension for vector search with no external dependencies. It implements a `vec0` virtual table module supporting float32, int8, and binary vectors with multiple indexing strategies: full-scan (brute force), Rescore (quantization + re-ranking), IVF (inverted file index, experimental), and DiskANN (graph-based ANN). Currently pre-v1.0 (v0.1.10-alpha.3).

## Build Commands

```bash
make loadable       # Build loadable SQLite extension (.so/.dylib/.dll) → dist/vec0
make static         # Build static library (libsqlite_vec0.a)
make cli            # Build standalone SQLite CLI with vec0 compiled in
make all            # Build all three
make wasm           # Build WebAssembly version
make amalgamation   # Create single-file amalgamation
```

Override defaults with `CC=clang make` or `AR=... make`.

SIMD optimizations (AVX2 on x86_64, NEON on ARM64) are detected and enabled automatically.

## Test Commands

```bash
make test                           # Basic SQL tests via test.sql
make test-loadable                  # Full Python pytest suite (requires uv)
make test-unit                      # C unit tests
make test-loadable-snapshot-update  # Update pytest snapshot golden files
make test-loadable-watch            # Watch mode (requires watchexec)
make fuzz-quick                     # Fuzz for 30 seconds per target
make fuzz-long                      # Fuzz for 5 minutes per target
```

The Python tests load `dist/vec0` (the loadable extension). Build it first with `make loadable`. Tests use uv-managed Python and pytest. The `tests/conftest.py` auto-skips tests for features not compiled in (e.g., IVF, DiskANN).

To run a single test file:
```bash
uv run pytest tests/test-diskann.py -v
uv run pytest tests/test-loadable.py::TestClassName::test_method_name -v
```

## Code Quality

```bash
make format   # Format C with clang-format, Python with black
make lint     # Check C formatting (diff against clang-format)
```

## Architecture

### Core Source Files

- **`sqlite-vec.c`** (~10,700 lines) — Main implementation. Contains: vector type parser, distance functions (L2, L1, Cosine, Hamming), the `vec0` virtual table (`vec0Create/Connect`, `vec0BestIndex`, `vec0Filter`, `vec0Update`), query planner, metadata/auxiliary column handling, and full-scan/KNN/point query execution.
- **`sqlite-vec-diskann.c`** — DiskANN graph-based ANN index; `#include`d into `sqlite-vec.c`.
- **`sqlite-vec-ivf.c`** — IVF clustering index (experimental); `#include`d into `sqlite-vec.c`.
- **`sqlite-vec-rescore.c`** — Quantization + re-scoring index; `#include`d into `sqlite-vec.c`.
- **`sqlite-vec-ivf-kmeans.c`** — K-means for IVF centroids; `#include`d into `sqlite-vec.c`.
- **`sqlite-vec.h.tmpl`** — Header template; substituted at build time to produce `sqlite-vec.h`.

### Virtual Table Internals

`vec0` is a SQLite virtual table using shadow tables for storage:
- `_chunks` — chunk metadata
- `_rowids` — mapping between user rowids and chunk positions
- `_vector_chunksNN` — packed vector data per chunk
- `_metadatachunksNN` / `_metadatatextNN` — metadata columns
- `_auxiliary` — auxiliary (non-indexed) columns

Query planning encodes query type and constraint information in `idxStr` using a header byte + 4-byte blocks per constraint. Query types: `FULLSCAN`, `POINT`, `KNN`. See `ARCHITECTURE.md` for the full `idxStr` encoding spec.

### Test Layout

```
tests/
  conftest.py          # Pytest fixtures and auto-skip logic for optional features
  helpers.py / utils.py
  test-unit.c          # C-level unit tests (compiled separately)
  test-loadable.py     # Primary comprehensive test suite
  test-diskann.py      # DiskANN-specific tests
  test-ivf.py          # IVF-specific tests
  test-rescore.py      # Rescore index tests
  test-*.py            # Other focused test modules
  __snapshots__/       # Snapshot golden files
  fuzz/                # Fuzzing harness and corpus
```

### Bindings and Examples

Language bindings live in `bindings/` (Python, Rust, Go). Usage examples for 14+ languages/runtimes are in `examples/`. Benchmark infrastructure is in `benchmarks-ann/` (unified ANN benchmark suite) and legacy `benchmarks/`.
