# int8dot

[![CI](https://github.com/go-simd/int8dot/actions/workflows/ci.yml/badge.svg)](https://github.com/go-simd/int8dot/actions/workflows/ci.yml)
![coverage](https://img.shields.io/badge/coverage-100%25-brightgreen)

SIMD-accelerated **integer dot products over byte slices** — the
multiply-accumulate at the heart of **quantized machine-learning inference**,
where embeddings and weights are stored as 8-bit integers to cut memory bandwidth
and exploit per-byte vector throughput. The signed `int8`, unsigned `uint8`, and
mixed `uint8×int8` contractions, each summed exactly into a 32-bit accumulator. It
is the integer counterpart to [`floats`](floats.md).
[Repository →](https://github.com/go-simd/int8dot)

Pure Go: `CGO_ENABLED=0`, stable Go, no `GOEXPERIMENT`. The hot MAC loops are real
integer SIMD on five of the six 64-bit Go targets (see below), assembly generated
by [go-asmgen](https://github.com/go-asmgen/asmgen).

```go
import "github.com/go-simd/int8dot"

d  := int8dot.Dot(a, b)        // []int8  · []int8   -> int32
du := int8dot.DotUint8(a, b)   // []uint8 · []uint8  -> uint32
ds := int8dot.DotU8S8(a, b)    // []uint8 · []int8   -> int32  (quantized GEMM layout)
```

## API

| function | operands | result |
|---|---|---|
| `Dot(a, b []int8)`             | int8 × int8   | `int32`  |
| `DotUint8(a, b []uint8)`       | uint8 × uint8 | `uint32` |
| `DotU8S8(a []uint8, b []int8)` | uint8 × int8  | `int32`  |

All three require `len(a) == len(b)` and panic otherwise. INT8 quantization
schemes differ in the sign of each operand: symmetric weights are `int8`,
ReLU-style activations quantize to `uint8`, and the **`uint8 × int8`** layout
(`DotU8S8`) is what hardware INT8 GEMM paths (x86 VNNI, ARM dot-product) are built
around — so it gets its own entry point.

### Saturation safety

Each byte×byte product fits in 16 bits and the sum accumulates in a 32-bit
integer, so there is no clamping: the result is **exact**. With worst-case
magnitudes `int32` does not overflow until ~131k elements — embedding vectors
(256–4096 dims of small quantized values) stay comfortably inside. Every SIMD
kernel is required to be **bit-for-bit identical** to the scalar reference on
every architecture (differential tests + `FuzzDot`).

## Per-arch kernels

| arch | int8 / uint8 kernel | u8×s8 kernel | notes |
|---|---|---|---|
| **amd64**   | `VPMOVSXBW`/`VPMOVZXBW` + `VPMADDWD` (AVX2) | AVX2 | runtime-gated on AVX2; VNNI deferred |
| **arm64**   | `VSMULL`/`VUMULL` widening (NEON, **Go 1.27+**) | `VUXTL`+`VSXTL`+`VSMULL` (NEON, **Go 1.27+**) | scalar on stable Go ≤ 1.26 |
| **ppc64le** | `VMULESB`/`VMULOSB` even/odd widening (VMX) | *scalar* | no mixed-sign byte multiply on Power |
| **s390x**   | `VMEB`/`VMOB` even/odd widening (vector facility) | *scalar* | big-endian; no mixed-sign byte multiply |
| **riscv64** | `VWMULVV` + `VWREDSUMVS` (RVV) | `VWMULSUVV` (RVV) | runtime-gated on RVV |
| **loong64** | `VMULWEVHB`/`VMULWODHB` even/odd widening (LSX) | *scalar* | no mixed-sign byte multiply on LoongArch |

Integer multiply-accumulate is associative and exact, so — unlike the
floating-point sibling — there is no lane-order or ULP subtlety; each kernel
widens byte products to 16 bits, accumulates into several 32-bit lanes (for ILP),
and horizontally sums at the end, with a scalar tail for the remainder.

### Honest architecture notes

- **arm64 — NEON SIMD on Go 1.27+, scalar on stable Go ≤ 1.26.** All three dot
  products have a NEON widening multiply-accumulate kernel, but the
  integer-multiply mnemonics (`VSMULL`/`VUMULL`) are only exposed by the Go arm64
  assembler from **Go 1.27** (1.26 has only `VPMULL`, polynomial multiply, useless
  for integer arithmetic), so the kernel is `//go:build arm64 && go1.27` and
  **stable Go ≤ 1.26 falls back to the scalar reference**. `DotU8S8` is full SIMD
  here too (unlike ppc64le/s390x/loong64). The ARMv8.2 dot-product instructions
  (`SDOT`/`UDOT`/`USDOT`) would be the one-instruction kernel, but as of the Go
  1.27 dev tree the assembler exposes them only in their **SVE** form — and SVE is
  not implemented by Apple Silicon, where this package is natively tested.
- **amd64 — AVX2, not VNNI (yet).** `VPDPBUSD` (AVX-512-VNNI) is the ideal
  `uint8×int8` instruction, but qemu's TCG implements no AVX-512, so neither the
  differential tests nor the 100% gate can exercise it. Rather than ship an
  unvalidated kernel, amd64 uses the AVX2 path (validated bit-exact on a native
  x86-64 VM). A `VPDPBUSD` fast path is a clean follow-up once a VNNI-capable,
  test-covered runner exists.
- **ppc64le / s390x / loong64 — `DotU8S8` is scalar.** None of these ISAs has a
  mixed unsigned×signed byte multiply primitive; `Dot` and `DotUint8` are full
  SIMD on all three.

## Performance — honest

Measured ratio of the AVX2 kernel against the scalar reference (`go test -bench`,
x86-64 under qemu TCG — absolute throughput is depressed by emulation, but the
**ratio** is representative; native silicon is ~4×):

| dim | scalar | AVX2 | speedup |
|----:|-------:|-----:|--------:|
| 256 | 254 MB/s | 836 MB/s | **~3.3×** |

**ppc64le:** the kernel is now **measured on real POWER10 silicon** (GCC Compile
Farm, VSX, Go 1.26.4, June 2026) — **~4.1× scalar**. **riscv64:** the RVV widening
multiply-reduce kernel (`VWMULVV` + `VWREDSUMVS`) is now **measured on real SpacemiT
X60 silicon** (GCC Compile Farm, RVV 1.0, Go 1.26.4, June 2026) — **~9.1× scalar**,
the largest riscv64 win in the suite (this is an arithmetic-bound kernel, exactly
what an in-order RVV core does well). **s390x:** **QEMU-validated for correctness;
native perf pending** a GitHub-hosted IBM Z runner.

## Coverage

`Dot`/`DotUint8`/`DotU8S8` are checked against a straight-line scalar loop by
table tests, extreme-value tests, and `FuzzDot`, with **100% statement coverage**
gated on every architecture: amd64 and arm64 natively, riscv64/loong64/ppc64le/
s390x under QEMU. Dispatch branches (AVX2 / RVV / scalar) are exercised directly.
BSD-3-Clause.
