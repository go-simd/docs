# levenshtein

[![CI](https://github.com/go-simd/levenshtein/actions/workflows/ci.yml/badge.svg)](https://github.com/go-simd/levenshtein/actions/workflows/ci.yml)
![coverage](https://img.shields.io/badge/coverage-100%25-brightgreen)

Pure-Go **Levenshtein edit distance** using **Myers' bit-parallel algorithm** —
tens of times faster than scalar dynamic-programming implementations, with
byte-for-byte identical results. [Repository →](https://github.com/go-simd/levenshtein)

```go
import "github.com/go-simd/levenshtein"

levenshtein.Distance([]byte("kitten"), []byte("sitting")) // 3
levenshtein.DistanceString("kitten", "sitting")           // 3
```

## API

```go
func Distance(a, b []byte) int
func DistanceString(a, b string) int
```

Both compare element by element (byte by byte / UTF-8 code unit by code unit) and
are symmetric. To compare Unicode text by *rune* rather than by byte, decode to
`[]rune` first.

## Honest scope: pure-Go bit-parallel, not vector-SIMD

This repo is in the go-simd suite as a **primitive**, but unlike the other
packages it ships **no `.s` SIMD kernel** — and the page says so plainly. The
single-word path (patterns ≤ 64 bytes) is already optimal scalar `uint64` bitops:
there is nothing for a vector unit to parallelise *within* one machine word. The
multi-word blocked path has a **strict top-to-bottom data dependency** — each
64-bit block consumes the horizontal carry produced by the block above it — so
blocks within a single text symbol cannot be advanced independently across SIMD
lanes without a more elaborate carry-lookahead scheme. Accordingly this release
ships a **portable pure-Go bit-parallel implementation on all of Go's 64-bit
targets** (amd64, arm64, riscv64, loong64, ppc64le, s390x). It is endian-clean
(validated on big-endian s390x) because Go defines `uint64` shift/add semantics
independently of machine byte order.

**Future work:** lane-parallel processing of *multiple independent string pairs*
(SIMD across pairs, not across blocks) is the clean fit for vector hardware and
is the most promising path to true SIMD acceleration here.

## Algorithm

Instead of the textbook `O(m·n)` DP recurrence, this package uses **Myers'
bit-vector algorithm** (G. Myers, *A fast bit-vector algorithm for approximate
string matching based on dynamic programming*, J. ACM 46(3), 1999), in Hyyrö's
clean reformulation. An entire DP column is packed into machine words and
advanced with a handful of bitwise operations:

```
xv = Eq | Mv
xh = (((Eq & Pv) + Pv) ^ Pv) | Eq
Ph = Mv | ~(xh | Pv)
Mh = Pv & xh
... shift, fold back into Pv/Mv, update the running score ...
```

This is `O(⌈m/w⌉·n)` with word size `w = 64`:

* **Patterns ≤ 64 bytes** use a single-word routine (`myers64`): one `uint64`
  holds the whole column, so the inner loop is a few register-resident bitops per
  text byte and runs in `O(n)` word operations with **zero allocations**.
* **Longer patterns** are processed in blocks of 64 rows (`myersBlocked`),
  threading the horizontal carry top-to-bottom between adjacent 64-bit words.

The shorter input is always packed into the bit-vectors (the "pattern") and the
longer one streamed, minimising the number of blocks.

## Performance

Apple M-series (`go test -bench`), random 26-letter strings with ~10% edits, vs
the scalar two-row DP and [`agnivade/levenshtein`](https://github.com/agnivade/levenshtein)
(the popular scalar DP library):

| length | this (bit-parallel) | scalar DP | agnivade | vs DP | vs agnivade |
|-------:|--------------------:|----------:|---------:|------:|------------:|
|     16 |   70 ns | 284 ns  |   86 ns |  4.0× |  1.2× |
|     64 |  219 ns | 4.9 µs  |  1.5 µs |   22× |  6.9× |
|    256 |  2.2 µs | 86 µs   |  138 µs |   38× | **62×** |
|   1024 |   33 µs | 1.40 ms | 4.64 ms |   43× | **142×** |

The advantage grows with length because the bit-parallel column update collapses
64 DP cells into one word operation. For short inputs (≤16) all three are within a
couple of register operations of each other; the algorithmic win shows up once the
strings exceed a word. Numbers are machine-dependent; reproduce with
`go test -bench=. -benchmem`.

## Correctness

`Distance` is checked against an intentionally trivial scalar DP oracle:

* an exhaustive **differential test** across lengths straddling every 64-bit
  block boundary (0, 1, …, 63, 64, 65, …, 257, 300) and alphabets of size
  1/2/4/26;
* a `FuzzDistance` differential + symmetry fuzzer (millions of executions);
* the full suite runs **natively on amd64/arm64 and under QEMU on riscv64,
  loong64, ppc64le, and big-endian s390x** in CI.

Test coverage is gated at **100%** on every architecture. BSD-3-Clause.
