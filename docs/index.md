# go-compressions documentation

**Pure-Go byte-stream transformation primitives** — lossless compression codecs
and content-addressable hash functions, sitting at the same layer of the stack:
allocation-aware, zero-copy-friendly transforms that turn a byte stream into
another, smaller or fingerprinted, byte stream. **Five repositories** in all.

Everything here is **pure Go, `CGO_ENABLED=0`, multi-arch**. Codecs are
fuzz-tested against the upstream reference implementations they are
wire-compatible with; hashes are cross-checked against the official spec test
vectors. **100% statement coverage** is the bar on every module.

## Two concerns, one layer

Despite the name, this org houses the broader family of byte-stream
transformation primitives — both lossless compression codecs **and**
content-addressable hash functions. They live together because they sit at the
same place in the stack: a content-addressable storage layer hashes its blobs
and compresses its bodies side by side, so it is natural to ship those
primitives side by side too. Different math, same engineering bar.

## The honesty policy

The numbers on these pages are benchmarked, not hand-waved. Every page reports
the **honest headline** — the wins **and** the cases where this code trails the
state of the art. [`lz4`](repos/lz4.md) **beats `pierrec/lz4` on compression
ratio** on every corpus, but it is **slower** on speed, and that is reported as
such. [`blake3`](repos/blake3.md) is several times slower than a hand-written
AVX2 implementation on a single core in its pure-Go default, and openly says so.
The credibility is the honesty.

## Compression codecs

| Package | Format | Honest headline |
| --- | --- | --- |
| [`lz4`](repos/lz4.md) | LZ4 block format | **beats `pierrec/lz4` on ratio** (text ≈4.6% smaller, binary ≈2.2%); **trails on speed** (~0.67–0.72× native arm64) — the gap is match-*finding*, not the SIMD match-extension kernel |
| [`lzfse`](repos/lzfse.md) | Apple LZFSE + LZVN | byte-compatible with Apple's `liblzfse` (round-trips both ways); auto-picks LZVN ≤ 4 KiB, LZFSE above; 100% coverage with no-panic fuzz on adversarial input |
| [`lzfsec`](repos/lzfsec.md) | LZFSE/LZVN CLI | cobra CLI over `lzfse` — `compress`/`decompress`, stdin/stdout by default, optional timing+ratio summary; pipe-safe |

## Content-addressable hashes

| Package | Algorithm | Honest headline |
| --- | --- | --- |
| [`blake3`](repos/blake3.md) | BLAKE3 (unkeyed) | pure-Go, verified vs official vectors; **multi-core one-shot** + precomputed message schedule; **default build is SIMD on all 6 arches** (`mix4` via [go-asmgen](https://github.com/go-asmgen/asmgen)); optional `archsimd` path. Honest: hand-written AVX2 (`lukechampine`) still faster on a single core |
| [`b3sum`](repos/b3sum.md) | BLAKE3 checksum CLI | reference-compatible `b3sum` (`<hex>  <name>`); `--check`, `--length`, `--no-names`; rides blake3's six-arch SIMD; ~1.5× end-to-end on a 1 GiB file vs the scalar build |

The `matchlen` SIMD common-prefix primitive that powers [`lz4`](repos/lz4.md)'s
match extension lives in the [go-simd](https://github.com/go-simd) family
([matchlen](https://github.com/go-compressions/matchlen) is mirrored here) — its
kernel ships real SIMD on all six of Go's 64-bit SIMD targets (amd64, arm64,
riscv64, loong64, ppc64le, s390x). **ppc64le is now natively measured on real
POWER10 silicon** ([GCC Compile Farm](https://portal.cfarm.net/), VSX, Go 1.26.4):
`lz4` encode runs **1.8× scalar** (1174 vs 644 MB/s) and **beats `pierrec/lz4`**
(1174 vs 1012 MB/s), with `blake3` `mix4` at **4.5× scalar**. The s390x path stays
**qemu-validated for correctness** (bit-identical to scalar); its native throughput
is **pending an IBM Z runner** and is never quoted as a headline.

Beyond the six SIMD targets, every library (`lz4`, `lzfse`, `blake3` / `b3sum`)
also **builds and passes its tests bit-exact on a seventh architecture, ppc64
(big-endian)**, on real POWER9 silicon via the portable fallback path — proving
big-endian correctness distinct from the s390x vector kernel. **Six SIMD targets,
validated on seven architectures.**

Read the [methodology](methodology.md) for the wire-compat → fuzz →
real-hardware → 100%-coverage pipeline every repo follows.

Source lives under
[github.com/go-compressions](https://github.com/go-compressions).
