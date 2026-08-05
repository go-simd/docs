# crc32

[![CI](https://github.com/go-simd/crc32/actions/workflows/ci.yml/badge.svg)](https://github.com/go-simd/crc32/actions/workflows/ci.yml)
![coverage](https://img.shields.io/badge/coverage-100%25-brightgreen)

A pure-Go, SIMD-accelerated **drop-in replacement for `hash/crc32`**
(`CGO_ENABLED=0`, stable Go, no `GOEXPERIMENT`). It produces **bit-identical**
CRC-32 checksums — for the `IEEE`, `Castagnoli` and `Koopman` polynomials and
for any custom polynomial. [Repository →](https://github.com/go-simd/crc32)

## Why arm64 only

Unlike the rest of the org, this package does **not** target all six 64-bit
arches. The standard library's `hash/crc32` IEEE fast path is already an
excellent PCLMULQDQ fold on amd64 and hardware-assisted on ppc64le and s390x —
there is nothing to beat there. The one 64-bit target whose stdlib IEEE path
is a latency-bound serial `CRC32X` is **arm64**; this package folds it with an
eight-lane `PMULL`/`PMULL2` kernel and defers to the standard library
everywhere else (including every non-IEEE polynomial, on every arch). The
result is always exactly `hash/crc32`.

| Arch    | IEEE bulk path                              | Gate                                   |
|---------|-----------------------------------------------|-----------------------------------------|
| arm64   | `PMULL` / `PMULL2` fold-by-eight (this pkg)  | `cpu.ARM64.HasPMULL`, always on darwin |
| amd64   | stdlib `PCLMULQDQ` / AVX-512 fold             | standard library                        |
| ppc64le | stdlib `VPMSUMD`                              | standard library                        |
| s390x   | stdlib vector-galois                          | standard library                        |
| riscv64 | stdlib scalar table                           | standard library                        |
| loong64 | stdlib scalar table                           | standard library                        |

## API

The API matches `hash/crc32` exactly — change only the import path:

```go
import "github.com/go-simd/crc32" // was "hash/crc32"

sum := crc32.ChecksumIEEE(data)

tab := crc32.MakeTable(crc32.IEEE)
sum = crc32.Checksum(data, tab)
crc := crc32.Update(0, tab, data)

h := crc32.NewIEEE()
h.Write(data)
_ = h.Sum32()
```

`Checksum`, `ChecksumIEEE`, `Update`, `New`, `NewIEEE`, `MakeTable`,
`IEEETable`, the `IEEE`/`Castagnoli`/`Koopman` constants, `Size`, the `Table`
type (aliased to the stdlib type, so tables are interchangeable), and the
`hash.Hash32` returned by `New` (including
`BinaryMarshaler`/`BinaryUnmarshaler`/`AppendBinary`) are all present.

## Algorithm

For the IEEE polynomial and inputs at or above `minBulk` (512 B), the data is
folded 16 bytes at a time into eight independent 128-bit reflected
accumulators using carryless multiplication — eight dependency chains hide the
`PMULL` latency, which is what lifts it past arm64's serial hardware
`CRC32X`. The lanes are then collapsed to one and reduced to the 32-bit CRC.
The fold constants are derived from the IEEE polynomial itself
(`reflect(x^(d+63) mod P)` and `reflect(x^(d-1) mod P)`) — no copied magic
numbers. The short tail (< 16 bytes) and every non-IEEE polynomial reuse the
standard library, so the result is guaranteed identical to `hash/crc32`.

## Coverage

`FuzzChecksum` compares against `hash/crc32` for both IEEE and Castagnoli on
arbitrary inputs, plus exhaustive length sweeps across every block boundary
for IEEE, Castagnoli and Koopman. CI runs on native amd64/arm64 (with `-race`)
and under QEMU for riscv64, loong64, ppc64le (power9) and s390x (big-endian),
with a **100% statement-coverage** gate on every architecture.

!!! note "No cross-arch benchmark table here"
    Unlike the other repos in this org, `crc32` ships no benchmark suite as of
    this writing — since only arm64 gets a real kernel and every other arch
    already runs a hardware-assisted stdlib path, there is no honest
    cross-arch speedup number to report yet. See the
    [repository](https://github.com/go-simd/crc32) for the current state.

BSD-3-Clause.
