# bitset

[![CI](https://github.com/go-simd/bitset/actions/workflows/ci.yml/badge.svg)](https://github.com/go-simd/bitset/actions/workflows/ci.yml)
![coverage](https://img.shields.io/badge/coverage-100%25-brightgreen)

SIMD-accelerated **bulk bit-set operations** over a bit set stored as a slice of
64-bit words (`[]uint64`), in pure Go (`CGO_ENABLED=0`, stable Go, no
`GOEXPERIMENT`). [Repository →](https://github.com/go-simd/bitset)

```go
import "github.com/go-simd/bitset"

bitset.And(dst, a, b)                 // dst[i] = a[i] & b[i]   (alias-safe)
bitset.Or(dst, a, b)                  // dst[i] = a[i] | b[i]
bitset.AndNot(dst, a, b)              // dst[i] = a[i] &^ b[i]
bitset.Xor(dst, a, b)                 // dst[i] = a[i] ^ b[i]

n  := bitset.Count(a)                 // total set bits (popcount)
ni := bitset.IntersectionCount(a, b)  // popcount(a & b),  no allocation
nu := bitset.UnionCount(a, b)         // popcount(a | b)
nd := bitset.DifferenceCount(a, b)    // popcount(a &^ b)
```

Word `w` bit `b` is set bit `w*64+b`. The four logical ops write the first
`min(len(dst), len(a), len(b))` words and may alias (`dst` can be `a` and/or
`b`); the counts read the first `min(len(a), len(b))` words. Nothing is
allocated, and the read-only ops are safe for concurrent use.

## Algorithm

Two clean families — vector logical ops and vector popcount — both available on
every 64-bit SIMD target, so no integer multiply (and therefore no special-casing
of arm64) is needed. The pairwise counts
(`IntersectionCount`/`UnionCount`/`DifferenceCount`) **fuse** the boolean op into
the popcount kernel, so the combined words are never written back to memory. Each
kernel handles whole vector blocks; a scalar `math/bits` tail finishes the
remainder, so every result is bit-identical to the reference.

## Per-arch kernels

| arch | logical ops (`And`/`Or`/`AndNot`/`Xor`) | counts |
|---|---|---|
| **amd64**   | AVX2 `VPAND`/`VPOR`/`VPANDN`/`VPXOR`, 32 B/iter | 4-way `POPCNTQ`, boolean fused in a GPR |
| **arm64**   | NEON `VAND`/`VORR`/`VEOR` (`AndNot` = `a ^ (a&b)`) | NEON `VCNT` + `VUADDLV` |
| **loong64** | LSX `VANDV`/`VORV`/`VXORV`/`VANDNV` | LSX `VPCNTV` per-64-bit lane |
| **ppc64le** | VSX `VAND`/`VOR`/`VXOR`/`VANDC` | VSX `VPOPCNTD` per-doubleword |
| **s390x**   | vector `VN`/`VO`/`VX`/`VNC` | `VPOPCT` + `VSUMB`/`VSUMQF` reduction |
| **riscv64** | portable `math/bits` word loop | portable `math/bits.OnesCount64` |

Five architectures carry native SIMD kernels: amd64, arm64, loong64, ppc64le and
s390x. ppc64le is the POWER8 VSX baseline, s390x the z13 vector-facility baseline,
loong64 LSX is treated as baseline — none needs a runtime feature flag. amd64
gates on `AVX2` (logical ops) and `POPCNT` (counts) at runtime, falling back to
scalar otherwise. riscv64 uses the portable `math/bits` path (the base RVV
profile offers no portable per-element word popcount, and the bandwidth-bound
logical ops gain nothing over scalar there). The hot loops are assembly generated
by [go-asmgen](https://github.com/go-asmgen/asmgen).

## The honest result: it's size-dependent

A scalar `math/bits` word loop already moves about one 64-bit word per cycle, so
for large, out-of-cache sets the bulk logical ops are
**memory-bandwidth-bound** and SIMD can only *tie* scalar — the win is real for
in-cache / medium sets, where the problem is compute-bound. The fused popcount
counts win across the board because they replace several scalar instructions per
word with one vector instruction.

**arm64** (Apple-silicon class, native):

| size  | `And` vs scalar | `Count` vs scalar / bits-and-blooms | `IntersectionCount` vs bits-and-blooms |
|-------|-----------------|-------------------------------------|----------------------------------------|
| 1 KiB | ~1.7× | ~1.8× / ~1.2× | ~1.8× |
| 1 MiB | ~1.7× | ~1.9× / ~1.2× | ~1.9× |
| 16 MiB| ~1.9× | ~1.9× / ~1.2× | ~1.9× |

**amd64** (`POPCNT`+`AVX2`, native x86-64 VM):

| size  | `And` vs scalar | `Count` vs scalar / bits-and-blooms | `IntersectionCount` vs bits-and-blooms |
|-------|-----------------|-------------------------------------|----------------------------------------|
| 1 KiB | ~1.2× | **~3.0×** / ~2.9× | ~2.5× |
| 1 MiB | ~1.4× | **~3.9×** / ~3.5× | ~2.7× |
| 16 MiB| ~1.2× (ties) | ~3.6× / ~2.2× | ~1.1× (ties, bandwidth-bound) |

The `Count`/`IntersectionCount` kernels are the clearest win (hardware popcount
is far denser than a scalar `OnesCount64`); the logical ops are bandwidth-bound
and converge toward scalar as the set leaves cache. **ppc64le / s390x** kernels
are **qemu-validated for correctness; native perf pending** real POWER/Z hardware.

## Where it fits

[`bits-and-blooms/bitset`](https://github.com/bits-and-blooms/bitset) is the
de-facto Go bit set, but its bulk ops are **scalar** `math/bits` loops over its
backing `[]uint64`; this package is its dense-`[]uint64` SIMD complement (operate
on the words directly, via `FromWithLength`).
[`RoaringBitmap/roaring`](https://github.com/RoaringBitmap/roaring) is a
different, **compressed** structure — not directly comparable. This package is
intentionally minimal: it accelerates the bulk word kernels over a caller-owned
`[]uint64` and composes with either.

## Coverage

Every architecture is validated against an independent scalar oracle by a table
test, an exhaustive size sweep across the SIMD-block/tail boundary, and the
`FuzzAnd`/`FuzzOr`/`FuzzAndNot`/`FuzzXor`/`FuzzCount`/`FuzzCounts` fuzzers. CI
runs amd64 and arm64 natively and riscv64/loong64/ppc64le/s390x under QEMU, gated
at **100% coverage** on every arch; the s390x run additionally proves the
big-endian path (`[]uint64` word ops and the popcount reductions are
endian-neutral, the table + fuzz tests are the proof). BSD-3-Clause.
