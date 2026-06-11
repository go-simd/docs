# Contributing

go-simd follows the same conventions as the author's other Go organizations.

## Hard rules

- **Pure Go, `CGO_ENABLED=0`, stable Go, no `GOEXPERIMENT`.** cgo wrappers of C
  SIMD libraries are faster still but out of scope — this is the pure-Go tier.
- **Drop-in or nothing.** A stdlib-facing package must produce **byte-identical
  output** (and byte-identical *errors*, where the stdlib returns them) for all
  input. The SIMD path handles whole aligned blocks; the tail delegates to the
  standard library so results match exactly.
- **Generated assembly, not hand-written.** Kernels are emitted by
  [go-asmgen](https://github.com/go-asmgen/asmgen); the `*_gen.go` are
  `//go:build ignore` and the resulting `.s` is committed. go-asmgen is a
  build-time tool, never a runtime dependency.
- **100% statement coverage** of the Go code, enforced as a CI gate on every
  arch job. Every dispatch branch (AVX2 / SSE / POPCNT / scalar fallback) is
  driven by a force test on the native runner.
- **Honest numbers.** Report wins, parities, and non-SIMD outcomes alike.
  Headline benchmarks come from native CI, never from emulation; Rosetta is
  never trusted for AVX2.
- **English only** for all repository content (issues, PRs, commits, comments).

## Adding or changing a kernel

1. **Survey the prior art** (see [methodology](methodology.md)) so the
   comparison is against the real state of the art.
2. **Write the generator**, not the `.s`. Drive go-asmgen to emit the kernel for
   each target arch; commit the generated `.s`.
3. **Prove correctness** with a table test (boundaries, every error class) and a
   `Fuzz*` differential target against the stdlib/oracle reference — comparing
   value **and** error. Add a force test that drives each dispatch branch
   directly.
4. **Validate on real hardware** — native amd64 (real AVX2, not Rosetta),
   native arm64, riscv64/loong64 under QEMU.
5. **Model and measure** — `llvm-mca` for the cycle model, then native-CI
   benchmarks for the headline numbers. Update the README's honest results
   table.
6. **Confirm 100% coverage:**
   ```bash
   go test -coverprofile=cover.out ./...
   go tool cover -func=cover.out   # must read 100.0%
   ```

## Regenerating the assembly

Each repo documents its exact regenerate command; the shape is:

```sh
go get github.com/go-asmgen/asmgen@latest
go generate ./...        # or: go run <name>_gen.go
go mod edit -droprequire github.com/go-asmgen/asmgen
go mod tidy
```

## Documentation

This site is built with MkDocs Material and versioned with
[mike](https://github.com/jimporter/mike); see the
[docs repo README](https://github.com/go-simd/docs) for local preview and
release commands. The organization landing page lives in
[go-simd.github.io](https://github.com/go-simd/go-simd.github.io) (Hugo).
