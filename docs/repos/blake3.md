# blake3

[![CI](https://github.com/go-compressions/blake3/actions/workflows/ci.yml/badge.svg)](https://github.com/go-compressions/blake3/actions/workflows/ci.yml)
![coverage](https://img.shields.io/badge/coverage-100%25-brightgreen)

Pure-Go, cgo-free implementation of the **BLAKE3** cryptographic hash (default,
unkeyed mode), verified against the official BLAKE3 test vectors. A single
dependency-free package with 100% test coverage.
[Repository →](https://github.com/go-compressions/blake3)

## API

```go
import "github.com/go-compressions/blake3"

sum := blake3.Sum256(data)            // [32]byte
sum512 := blake3.Sum512(data)         // [64]byte

h := blake3.New()                     // streaming (hash.Hash-style)
h.Write(part1); h.Write(part2)
digest := h.Sum(nil)                  // 32-byte digest

var xof [131]byte                     // extendable output (XOF)
h.Digest(xof[:])
```

- `Sum256(data) [32]byte`, `Sum512(data) [64]byte`
- `New() *Hasher` with `Write`, `Sum(b) []byte`, `Reset`, `Size`, `BlockSize`,
  and `Digest(out []byte)` for arbitrary-length extendable output.

Default (unkeyed) hash mode only; keyed and derive-key modes are not implemented.

## Pure-Go optimisations (no assembly)

Two optimisations apply with zero assembly, on every `GOARCH`:

- **Precomputed message schedule** — the per-round message-word permutation is
  folded into a `[7][16]` lookup table at init, so the compression function
  indexes the original message directly instead of physically permuting a
  16-word array six times per block.
- **Multi-core one-shot hashing** — BLAKE3's chunk tree makes per-chunk chaining
  values independent, so `Sum256`/`Sum512` compute them across all CPU cores
  (`runtime.NumCPU()`); the cheap `O(chunks)` tree merge stays sequential.
  Inputs below 16 chunks (16 KiB) are hashed inline to avoid goroutine overhead.
  The result is byte-identical to the streaming `Hasher`.

Representative throughput on a 16-core machine (`Sum256`):

| Input | This package | Notes |
| --- | --- | --- |
| 2 KiB | ~0.35 GB/s | single chunk — scalar path only |
| 64 KiB | ~0.9 GB/s | parallel kicks in |
| 1 MiB | ~1.8 GB/s | |
| 16 MiB | ~2.1 GB/s | scales with cores |

The streaming `Hasher` (`Write`/`Sum`/`Digest`) is single-threaded by design;
use `Sum256`/`Sum512` for one-shot hashing of large buffers to get the parallel
speed-up.

## Default SIMD path (all six arches, no GOEXPERIMENT)

As of **v0.5.0** the **default build is SIMD-accelerated on all six** 64-bit
targets — cgo-free, no `GOEXPERIMENT` — using
[go-asmgen](https://github.com/go-asmgen/asmgen)-generated assembly for the
`mix4` round function: NEON (arm64), SSE2 (amd64), LSX (loong64), RVV (riscv64),
VSX (ppc64le) and the vector facility (s390x, **big-endian**). It is
bit-identical to the scalar path (verified against the official BLAKE3 vectors)
and falls back to scalar for small inputs. **ppc64le is now natively measured on
real POWER10 silicon** ([GCC Compile Farm](https://portal.cfarm.net/), VSX,
Go 1.26.4): `mix4` runs **4.5× scalar**. **riscv64 is now natively measured too**
on a SpacemiT X60 (RVV 1.0, a low-power in-order core — the only widely-available
RVV silicon; GCC Compile Farm, Go 1.26.4): `mix4` runs **2.9× scalar** (3024 vs
8782 ns) with `FillChunk` ~2.0× (275 vs 137 MB/s); an out-of-order RVV core would
likely do better. The s390x path stays qemu-validated
(correct + bit-identical), with native throughput pending an IBM Z runner.

Beyond the six SIMD targets, `blake3` also **builds and passes its tests
bit-exact on a seventh architecture, ppc64 (big-endian)**, on real POWER9 silicon
via the portable fallback path — six SIMD targets, validated on seven
architectures.

## Experimental archsimd path (`GOEXPERIMENT=simd`)

A second, opt-in SIMD path uses Go's experimental
[`simd/archsimd`](https://pkg.go.dev/simd/archsimd) intrinsics (still pure Go —
no `.s`). It compiles only under `GOEXPERIMENT=simd`:

- **amd64 AVX2** (`blake3_simd_amd64.go`, Go 1.26+) hashes **8 full chunks at
  once**, one chunk per 256-bit lane.
- **arm64 NEON** (`blake3_simd_arm64.go`, Go 1.27+) hashes **4 chunks per lane**
  (`Uint32x4`).

```sh
GOEXPERIMENT=simd go test ./...      # requires Go 1.26+ (amd64) / 1.27+ (arm64)
```

Both are bit-identical to scalar against the official vectors. Measured natively
on an Apple M4 Max (NEON):

| input | scalar | archsimd NEON | Δ |
| --- | --- | --- | --- |
| 2 KiB | 337 MB/s | 335 MB/s | tie (single chunk — no SIMD) |
| 64 KiB | 841 MB/s | 841 MB/s | tie (scalar fallback) |
| 1 MiB | 1791 MB/s | 2452 MB/s | **+37%** |
| 16 MiB | 2350 MB/s | 3297 MB/s | **+40%** |

The kernel only switches to SIMD once there are at least `NumCPU` chunk batches
(measured crossover: scalar wins at 64 KiB, NEON wins from 80 KiB — right at
batches ≈ NumCPU), so the SIMD build is never slower than scalar at any size.

## Honest verdict

A hand-written AVX2 implementation such as
[`lukechampine.com/blake3`](https://github.com/lukechampine/blake3) is still
several times faster on a single core. The default build deliberately trades that
for being portable to every `GOARCH` and trivially auditable, while the
go-asmgen and archsimd paths recover much of the gap without leaving pure Go.

## License

BSD-3-Clause.
