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

## 2. Generate the assembly with go-asmgen (6 architectures)

No kernel is hand-encoded. The `.s` is emitted by
[go-asmgen](https://github.com/go-asmgen/asmgen) `v0.5.0` — one builder over a
**shared ABI0 layout** for **all six of Go's 64-bit SIMD targets** — and
`cmd/asm` does the encoding:

| arch | ISA used by go-simd kernels |
| --- | --- |
| amd64 | SSE2 / SSSE3 / SSE4.1 + **AVX2**, runtime-dispatched via `x/sys/cpu` |
| arm64 | NEON |
| loong64 | LSX / LASX |
| riscv64 | RVV |
| **ppc64le** | **VSX / AltiVec** (POWER8 baseline, no runtime dispatch) |
| **s390x** | **vector facility** (z13 baseline) — **big-endian** |

This is the **go-asmgen v0.5.0 six-arch milestone**: one generator now emits
correct kernels for every 64-bit target Go can produce SIMD for. The ppc64le
(VSX) and s390x (vector facility) backends are the newest; since VSX is baseline
on POWER8+ and the vector facility on z13+, those kernels need no runtime feature
dispatch — the SIMD path is simply the arch's only path (build-tagged), like
riscv64 and loong64.

**s390x is big-endian.** Every kernel is ported and validated on a big-endian
target, which exercises the control-vector lane numbering of `VPERM` shuffles and
the byte order of vector loads/stores (`VL`/`VST`) — a real cross-endian
correctness check, not just a recompile. Where amd64's 16-bit windows are
themselves big-endian (e.g. base32, base64), the s390x layout matches directly.

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
- **QEMU** for riscv64 (RVV), loong64 (LSX), **ppc64le (VSX)** and **s390x
  (vector facility)** — correctness only; QEMU's TCG does not model
  out-of-order execution, so absolute MB/s under emulation is *not*
  representative and is never quoted as a headline. The cross-built binaries run
  under `qemu-user` in a **debian:trixie** container with `QEMU_CPU=power9` for
  ppc64le and `QEMU_CPU=qemu` for s390x, exercising the VSX and z/Architecture
  vector instructions the kernels emit.
- **ppc64le is now natively measured on real silicon.** Beyond the qemu
  correctness run, the VSX kernels are benchmarked on a **POWER10 host on the
  [GCC Compile Farm](https://portal.cfarm.net/)** (VSX, Go 1.26.4, June 2026),
  confirming they are fast as well as correct: matchlen **6.3× scalar**,
  streamvbyte decode **11.9×**, hex encode **7.6×**, utf8 validate **7×**, crc64
  **5.7×**. These ppc64le headline numbers are real native measurements, not
  emulated.
- **s390x is qemu-validated for correctness, native perf pending.** It runs the
  official test vectors and the byte-identical differential fuzz suites (and on
  **big-endian s390x** the output is proven bit-exact), but there is **no
  GitHub-hosted IBM Z runner**, so native throughput numbers are not yet measured
  — and are deliberately *not* invented from the emulated runs.
- **A seventh architecture, ppc64 (big-endian), is build- and test-validated on
  real POWER9 silicon** (GCC Compile Farm). It has no VSX build tag, so it runs
  the **portable generic fallback path** — proven bit-exact on a big-endian target
  *distinct from* s390x's own vector kernel, adding cross-endian correctness
  coverage. SIMD acceleration itself stays on the six SIMD targets: **six SIMD
  targets, validated on seven architectures.**
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

The combination — one generator across **six ISAs**, a cycle model that predicts
before it measures, real-hardware validation that refuses to trust emulation (and
qemu-correctness validation where no native runner exists), and a coverage gate
that refuses to ship an unexercised branch — is what lets go-simd publish
**honest** numbers: it beats `emmansun` and `tmthrgd` where it says it does, it
openly reports the ~7% gap to `mhr3` and the fact that a byte histogram has no
SIMD win at all, and — now that **ppc64le is measured on a real POWER10 host** —
it surfaces those native VSX speedups while still labelling **s390x** as
correctness-validated with native perf pending (no IBM Z runner) rather than
quoting emulated throughput as if it were a headline. Correctness now spans a
**seventh architecture, ppc64 big-endian** (real POWER9, generic fallback path),
even though SIMD acceleration stays on six targets. A standout of the six-arch
port: **base32 gets real SIMD on ppc64le
(`VSRH`, register-variable vector shift) and s390x (`VMLHH`, integer vector
multiply-high) precisely where Go's arm64 assembler lacks those two
primitives** — so POWER and IBM Z run the full spread-extract kernel that NEON
could not express.
