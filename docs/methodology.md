# Methodology — the validation toolkit

A SIMD kernel is not "done" because it compiles and is fast on one machine. In
go-simd it is done when it is **proven correct** against the reference it
replaces and **measured honestly** on representative hardware. Every repo in the
organization follows the same five-stage pipeline.

## 1. Check existing work first

Before a single instruction is written, the field is surveyed so the comparison
is against the **real state of the art**, not a strawman. Concretely, the prior
pure-Go SIMD implementations each repo benchmarks against:

| repo | prior art benchmarked / surveyed |
| --- | --- |
| base64 | `emmansun/base64` (AVX2), `cristalhq/base64` (scalar); `aklomp/base64` is cgo (excluded) |
| base32 | none — no pre-existing pure-Go SIMD base32 exists |
| hex | `tmthrgd/go-hex` (SSE/AVX, archived Sep 2025) |
| utf8 | `stuartcarnie/go-simd` (2018 Lemire port); `charlievieth/simdutf` is cgo (excluded) |
| adler32 | `mhr3/adler32-simd` (Chromium/zlib transpiled via gocc) |
| popcount | `barakmich/go-popcount` (Tom Thorogood asm) |
| histogram | `vteromero/byte-hist` (CLI, not a library) — gap filled |
| bitpack | `ronanh/intcomp` (head-to-head is a follow-up) |
| matchlen | the compressor in-tree primitives (LZ4/zstd match-finders) |
| strconv | Go's own `strconv` (already very tight, especially `Atoi`'s inlined path) |

This is why the headlines are credible: each is a measured result against a
named competitor, and cgo wrappers (faster, but needing a C toolchain) are
explicitly excluded from the pure-Go tier rather than quietly ignored.

## 2. Generate the assembly with go-asmgen (4 architectures)

No kernel is hand-encoded. The `.s` is emitted by
[go-asmgen](https://github.com/go-asmgen/asmgen) — one builder over a **shared
ABI0 layout** for all four 64-bit Go targets — and `cmd/asm` does the encoding:

| arch | ISA used by go-simd kernels |
| --- | --- |
| amd64 | SSE2 / SSSE3 / SSE4.1 + **AVX2**, runtime-dispatched via `x/sys/cpu` |
| arm64 | NEON |
| loong64 | LSX / LASX |
| riscv64 | RVV |

The `*_gen.go` generators are `//go:build ignore` (go-asmgen is a *build-time*
tool, not a runtime dependency); the resulting `.s` is committed. Constant
tables are emitted via go-asmgen's `emit.File.Data`. Architectures without a
kernel use a portable scalar fallback so the package builds and is correct
everywhere.

## 3. Model the cycles with `llvm-mca`

Before benchmarking, the inner loop is given a **port-level cycle model** with
`llvm-mca` ([go-asmgen `toolkit/mca`](https://github.com/go-asmgen/asmgen/tree/main/asmgen/toolkit/mca)).
This is what turned guesses into wins:

- **base64** — `llvm-mca` showed the kernel was shuffle-port (`Zn3FP1`) bound;
  disassembling `emmansun` revealed a `-4`-offset 32-byte load that spreads with
  a single `VPSHUFB` (no cross-lane `VINSERTI128`). Combining it with the 2×
  unroll was *predicted* at 2.15 cyc/block (vs emmansun's 2.20) and the
  benchmark *confirmed* it — the ~5–6% lead.
- **adler32** — the 4×-unrolled AVX2 kernel models at **17.9 bytes/cycle**
  (≈99% of the vector-ALU port ceiling) on the Zen3 model. The static model puts
  it at parity with `mhr3`; the real out-of-order frontend still leaves mhr3 ~7%
  ahead. The model is a guide, not a verdict — see stage 4.

## 4. Validate on real hardware (Rosetta is never trusted for AVX2)

Correctness is proven by **differential tests + fuzzing** against the
stdlib/oracle reference, and the headline benchmarks come from **native CI**,
not emulation:

- **Real AVX2 x86-64 host** for amd64. **Rosetta hides AVX2**, so it is never
  used to validate or benchmark amd64 kernels; a `--platform amd64` container
  crashes Go. A real x86-64 box (or KVM guest) and GitHub Actions
  `ubuntu-latest` (AMD EPYC, AVX2) are the authoritative amd64 sources.
- **Native arm64** (the Apple-silicon dev box) for the NEON kernels and the
  `!amd64` generic fallback.
- **QEMU** for riscv64 (RVV) and loong64 (LSX) — correctness only; QEMU's TCG
  does not model out-of-order execution, so absolute MB/s under emulation is
  *not* representative and is never quoted as a headline.
- **Fuzzing** runs against the stdlib reference on arbitrary input, comparing
  the returned **value and the full error** (e.g. `strconv`'s `*NumError`,
  `hex`'s `InvalidByteError` offset). Direct kernel fuzz targets exercise the
  SSE and AVX2 paths separately (millions of executions, zero mismatches).

## 5. 100% statement coverage, enforced as a CI gate

Every repo gates **100% Go statement coverage** in CI, on every arch job — the
build fails below 100%. Force tests drive **every dispatch branch** (AVX2 / SSE /
POPCNT / scalar fallback) directly on the native runner, regardless of what the
runtime CPU would otherwise select, so no branch is left unmeasured.

!!! note "What coverage measures"
    The figure is of the **Go code** only. The generated `.s` SIMD kernels are
    not measured by `go test -cover`; they are validated by the differential
    tests against the stdlib/oracle reference plus the fuzz targets. "100%
    coverage" therefore means *every Go statement, including every fallback
    branch* — and the assembly is covered by the correctness suite, not the
    coverage counter.

## Why this matters

The combination — one generator across four ISAs, a cycle model that predicts
before it measures, real-hardware validation that refuses to trust emulation,
and a coverage gate that refuses to ship an unexercised branch — is what lets
go-simd publish **honest** numbers: it beats `emmansun` and `tmthrgd` where it
says it does, and it openly reports the ~7% gap to `mhr3` and the fact that a
byte histogram has no SIMD win at all.
