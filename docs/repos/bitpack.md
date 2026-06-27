# bitpack

[![CI](https://github.com/go-simd/bitpack/actions/workflows/ci.yml/badge.svg)](https://github.com/go-simd/bitpack/actions/workflows/ci.yml)
![coverage](https://img.shields.io/badge/coverage-100%25-brightgreen)

The **FastPFOR / simdcomp** bit-packing primitive in pure Go (`CGO_ENABLED=0`,
stable Go, no `GOEXPERIMENT`): pack blocks of **128 `uint32`** values, each using
exactly `bits` bits (1..32), into a tight little-endian bitstream — and the
inverse. [Repository →](https://github.com/go-simd/bitpack)

## API

```go
n := bitpack.Pack(dst, src, bits)   // src len a multiple of 128; n = 16*bits per block
bitpack.Unpack(out, dst, bits)
```

Output is **byte-identical to Lemire's simdcomp** (`SIMD_fastpackwithoutmask`):
the 128 integers are stored as 4 interleaved 32-int lanes. On amd64 the bulk runs
a generated **SSE2 / AVX2** kernel (one per bit-width, dispatched via
`x/sys/cpu`); **ppc64le** runs a **VSX** kernel (`LXVW4X`/`STXVW4X`, `VSLW`/`VSRW`,
`VAND`/`VOR`) and **s390x** a **vector-facility** kernel (`VL`/`VST`,
`VESLF`/`VESRLF`, `VN`/`VO`, `VPERM`), both build-tagged (VSX is POWER8 baseline,
the vector facility z13 baseline). Other arches use the scalar reference. ppc64le
is little-endian so words load/store directly; **s390x is big-endian** — the
`uint32` arrays are native big-endian and the vector facility is big-endian too,
so no fix-up is needed, and the big-endian port produces the same bytes as
scalar/simdcomp. Every path is byte-exact — verified by table tests, a
scalar-equality test, and `FuzzPack`/`FuzzUnpack`. **ppc64le is now measured on
real POWER9** (see below); **s390x stays qemu-validated for correctness only**,
native perf pending.

## Performance

Native amd64 (ubuntu-latest CI, AVX2, `-count=6`), throughput over the input
`uint32` stream, **vs the package's own scalar reference**:

| op | bits=8 | bits=16 | bits=24 |
|---|---|---|---|
| **Pack** SIMD | ~62.0 GB/s | ~57.5 GB/s | ~46.8 GB/s |
| Pack scalar | ~2.31 GB/s | ~1.92 GB/s | ~1.46 GB/s |
| **Unpack** SIMD | ~44.5 GB/s | ~42.5 GB/s | ~40.4 GB/s |
| Unpack scalar | ~2.08 GB/s | ~1.85 GB/s | ~1.38 GB/s |

That's **~21–32×** over scalar — bit-packing is a textbook SIMD-friendly problem
(shifts + masks + ORs, no data-dependent control flow). go-asmgen generates the
per-width kernels.

**ppc64le — measured on real POWER9** (VSX, GCC Compile Farm, Go 1.26.4,
2026-06-26): `Pack` ~7,200 vs ~1,020 MB/s scalar (**~7.0×**), `Unpack` ~8,630 vs
~760 MB/s (**~11.3×**). This **overturns the earlier llvm-mca cycle-model
estimate of ~1.1×** — on real silicon the VSX path is a decisive win. s390x is
still an llvm-mca estimate (~4.7× on z14), native perf pending real IBM Z silicon.

> Competitor note: `ronanh/intcomp` is the prior pure-Go SIMD-ish integer codec
> for this space; a head-to-head benchmark against it is a follow-up.

## Coverage

100% of the Go code, gated on the native amd64 and arm64 jobs and the **ppc64le /
s390x QEMU jobs** (the build fails below 100% on every one). Both amd64 kernel
paths — the AVX2 block-pair kernel and the SSE finisher — are driven directly by
the force test on the native AVX2 runner. The `.s` kernels are validated by
differential tests against the scalar reference plus the two fuzz targets (the VSX
and big-endian vector-facility kernels under `qemu-user`). BSD-3-Clause.
