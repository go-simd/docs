# histogram

[![CI](https://github.com/go-simd/histogram/actions/workflows/ci.yml/badge.svg)](https://github.com/go-simd/histogram/actions/workflows/ci.yml)
![coverage](https://img.shields.io/badge/coverage-100%25-brightgreen)
[![Go Reference](https://pkg.go.dev/badge/github.com/go-simd/histogram.svg)](https://pkg.go.dev/github.com/go-simd/histogram)

The fastest correct **byte-value histogram** in pure Go (`CGO_ENABLED=0`, stable
Go, **no assembly**): count how many times each of the 256 possible byte values
occurs in a `[]byte`. [Repository →](https://github.com/go-simd/histogram)

## API

```go
c := histogram.Count(data) // c[v] == number of bytes in data equal to v

var acc [256]uint32
histogram.CountInto(&acc, a) // accumulate several slices into one table
histogram.CountInto(&acc, b)
```

`Count` returns a fresh `[256]uint32` (no heap allocation). `CountInto` adds into
a caller-supplied array, folding multiple buffers into one histogram.

## Why there is no SIMD kernel (the honest answer)

A byte histogram is a **scatter-add**: `c[data[i]]++`. Each input byte indexes a
*different, data-dependent* counter and read-modify-writes it. That is precisely
the access pattern SIMD does **not** have:

* AVX2 and ARM NEON have **gather but no scatter**. You cannot vectorise
  `c[idx]++` across a lane vector because two lanes may target the same counter
  (a collision) and the writes would race within the instruction.
* Only **AVX-512** has a true vector scatter — and the dev/CI hardware here is
  Zen3-class with **no AVX-512**. Even where scatter exists, colliding bins
  serialise, so it rarely beats a good scalar loop on real (skewed) data.

So the win does **not** come from SIMD. It comes from **instruction-level
parallelism**: keep several independent `[256]uint32` partial tables, round-robin
consecutive bytes across them, then sum the tables once. Spreading neighbouring
bytes across distinct cells **breaks the store-to-load dependency** that stalls a
single-table loop whenever a byte value repeats — the classic "counting bytes
fast" trick from FSE. `Count` uses **eight partial tables with word-batched
loads** — the fastest of every variant measured (single, 2/4/8 tables, with and
without word-batching).

## Performance

Throughput on a 1 MiB buffer, best-of-6. Two distributions: **uniform** random
bytes, and **skew90** (90% one value — the low-entropy case that wrecks a naive
single-table loop).

### arm64 (indicative — Apple M-series dev box)

| approach | uniform GB/s | skew90 GB/s |
|---|---:|---:|
| single table (naive baseline) | 3.73 | 0.58 |
| 4 partial tables | 3.77 | 2.28 |
| **8 word-batched tables (`Count`)** | 4.52 | 2.83 |

> The authoritative amd64 row is filled by the `bench` workflow on native CI; the
> dev machine is arm64, so amd64 numbers must come from native CI, not Rosetta.

**The finding:** SIMD does not help a byte histogram on AVX2/NEON hardware — it is
a scatter, and there is no scatter instruction. The multi-table scalar loop is the
real winner. On uniform data it edges out the single-table baseline; on skewed
data it is **~5× faster**, because the single-table loop serialises on the
repeated counter while the eight tables keep the pipeline full.

## Correctness & coverage

`Count` is validated against a single-table reference oracle by a table test
(empty, single byte, all-same, every-byte-once, block-boundary sizes),
random-size property tests, a skewed-data test, and a `FuzzCount` differential
fuzz target (run in CI on amd64 and arm64).

The suite covers **100% of the Go statements** (CI fails below that on both arch
jobs). This package is pure Go — there are no generated `.s` kernels — so the
coverage figure reflects the whole implementation.

## Existing work

There was no pure-Go library exposing a fast `Count(data) [256]uint32` API.
`vteromero/byte-hist` is a CLI tool, not a reusable package, and
`valyala/histogram` is a float-quantile sketch, unrelated to byte counting. This
repo fills the gap. BSD-3-Clause.
