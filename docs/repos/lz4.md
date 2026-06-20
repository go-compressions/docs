# lz4

A clean pure-Go implementation of the **LZ4 block format** —
`CompressBlock` / `DecompressBlock` — wire-compatible with the reference and
cross-checked against `pierrec/lz4`. Blocks are mutually decodable with
`pierrec` in **both** directions, verified on arm64 and amd64.
[Repository →](https://github.com/go-compressions/lz4)

## API

```go
c := lz4.CompressBlock(src)
out, _ := lz4.DecompressBlock(c, len(src))
```

The compressor delegates LZ4's hot "count the matching bytes" inner loop
(`LZ4_count`) to [matchlen](https://github.com/go-compressions/matchlen), whose
SIMD common-prefix kernel makes match extension fast. As of `matchlen`
**v0.3.0** that kernel ships SIMD on **all six** of Go's 64-bit SIMD targets —
amd64 (SSE2), arm64 (NEON), riscv64 (RVV), loong64 (LSX), ppc64le (VSX) and
s390x (vector facility). lz4 needs **no code change** to benefit: `MatchLen`
dispatches per-arch, so the bulk match runs vectorized on ppc64le and s390x too.
**ppc64le is now natively measured on real POWER10 silicon**
([GCC Compile Farm](https://portal.cfarm.net/), VSX, Go 1.26.4): lz4 encode runs
**1.8× scalar** (1174 vs 644 MB/s) and **beats `pierrec/lz4`** (1174 vs 1012 MB/s).
The s390x path stays qemu-validated — correct and bit-identical to scalar — with
native throughput pending an IBM Z runner.

Beyond the six SIMD targets, `lz4` also **builds and passes its tests bit-exact
on a seventh architecture, ppc64 (big-endian)**, on real POWER9 silicon via the
portable fallback path — six SIMD targets, validated on seven architectures.

## The compressor

The parse is a single-cell hash table keyed on a **6-byte sequence** (the
reference LZ4 / `pierrec` fast-mode hash, far better dispersed than a 4-byte
key). Each step probes three adjacent positions (`ip`, `ip+1`, `ip+2`) from one
8-byte load, inserting every position so later matches see more candidates, and
ramps its skip distance on incompressible spans. On a hit it applies **lazy
matching** — it peeks one byte ahead and, if `ip+1` yields a strictly longer
match, defers the current one — capped to short matches so the lookahead cost is
only paid where it can help. Match-length extension is delegated to `matchlen`'s
SIMD kernel.

LZ4 is the ideal `matchlen` consumer because it has **no entropy-coding stage** —
encode time is dominated by match-finding and extension, so a faster `MatchLen`
shows up end-to-end (unlike codecs whose time goes to FSE/Huffman).

## Performance

Encoded as single LZ4 blocks, three representative corpora — two text (Project
Gutenberg `pg1661` Mark Twain, and the Twain corpus) and one binary (a kernel
`bzImage`) — `-count=8` medians, **native arm64 (Apple Silicon)**:

| corpus | this package | `pierrec/lz4` | speed vs `pierrec` | our size | `pierrec` size | size vs `pierrec` |
|---|---:|---:|---:|---:|---:|---:|
| text `pg1661` | 208 MB/s | 301 MB/s | 0.69× | **0.528** | 0.553 | **−4.6%** |
| text Twain | 205 MB/s | 285 MB/s | 0.72× | **0.548** | 0.575 | **−4.6%** |
| binary `bzImage` | 249 MB/s | 370 MB/s | 0.67× | **0.638** | 0.653 | **−2.2%** |

On the QEMU x86_64 lima VM (TCG, so absolutes are low and noisy) the encoder
lands at ≈0.5–0.7× `pierrec`'s throughput with the **same** size advantage — the
parse is deterministic, so sizes match arm64 exactly, and the compressed output
is byte-identical and decodes both ways with `pierrec`.

## Honest verdict

This pass **beats `pierrec` on compression ratio** on every corpus (text ≈4.6%
smaller, binary ≈2.2% smaller) — the 6-byte hash, 3-position probe and lazy
matching are real wins, and they fixed the prior parse, which was actually 7–8%
*worse* than `pierrec` on text. It does **not** beat `pierrec` on **speed**:
`pierrec` stays ahead at ≈1.4× (we are at ~0.67–0.72× native arm64). That gap is
the parse/table, not the kernel — `pierrec` uses a half-the-size 16-bit position
table (better cache footprint) and skips lazy matching in fast mode, trading a
little ratio for speed; we make the opposite trade. The SIMD `matchlen`
extension is correct and in use, but match extension is only ~10% of encode time
here — the bottleneck is match-*finding*.

## License

BSD-3-Clause.
