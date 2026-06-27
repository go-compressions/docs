# lzfse

[![CI](https://github.com/go-compressions/lzfse/actions/workflows/ci.yml/badge.svg)](https://github.com/go-compressions/lzfse/actions/workflows/ci.yml)
![coverage](https://img.shields.io/badge/coverage-100%25-brightgreen)

Pure-Go implementation of Apple's **LZFSE** and **LZVN** compression formats.
Byte-compatible with the reference `liblzfse` C implementation: data compressed
by `liblzfse` round-trips through `Decompress`, and data produced by `Compress`
is decoded by `liblzfse` without modification.
[Repository →](https://github.com/go-compressions/lzfse)

## API

```go
import "github.com/go-compressions/lzfse"

compressed, err := lzfse.Compress(payload)
if err != nil { /* ... */ }

decoded, err := lzfse.Decompress(compressed)
if err != nil { /* ... */ }
```

```go
func Compress(src []byte) ([]byte, error)
func Decompress(src []byte) ([]byte, error)
```

`Compress` picks the format automatically: inputs ≤ 4 KiB are emitted as an LZVN
block (one `bvxn` block followed by a `bvx$` end-of-stream); larger inputs are
emitted as LZFSE blocks (V1/V2 headers + FSE-encoded streams). `Decompress`
handles every block magic Apple's reference emits:

| Magic  | Bytes | Meaning                            |
| ------ | ----- | ---------------------------------- |
| `bvx-` | `2D…` | Uncompressed payload (passthrough) |
| `bvx1` | `31…` | LZFSE V1 (uncompressed freq table) |
| `bvx2` | `32…` | LZFSE V2 (variable-length codes)   |
| `bvxn` | `6E…` | LZVN block                         |
| `bvx$` | `24…` | End-of-stream marker               |

LZFSE is LZVN followed by an entropy stage (FSE/Huffman); that entropy stage is
where the codec's time goes, so it stays scalar pure Go — there is no
vectorizable hot loop to accelerate the way an LZ match-finder has.

## Consumers

- [`go-compressions/lzfsec`](lzfsec.md) — CLI wrapper
  (`lzfsec compress|decompress`).
- `github.com/go-filesystems/apfs` — APFS `decmpfs` transparent decompression
  for types 7 / 8 / 11 / 12 (LZVN / LZFSE, inline + resource-fork variants).
- `github.com/go-diskimages/tart-oci` — Tart layer decompression.

## Performance

Benchmarked single-core on Apple M4 Max against Apple's `liblzfse` C reference
built `-O3` (its fastest), timed in-memory on the Silesia corpus.

- **Ratio — at parity.** Compressed size tracks Apple's to within ~1–3% on real
  data (e.g. dickens 0.391 vs 0.379, samba 0.240 vs 0.241, nci 0.099 vs 0.096);
  level on synthetic inputs.
- **Decode — now close.** Two passes of decode-path work (overlap-aware bulk
  copy + word-at-a-time FSE refill, then an index-write LMD loop with fixed
  16-byte short-run copies and reused per-block scratch, 2026-06-24) made decode
  **1.1–1.6× faster** than the prior decoder and shrank the gap to Apple's `-O3`
  C reference from **~2–3×** to **~1.6–2.1×** — real files now decode at ~½–0.6×
  the reference (dickens 611 vs 1281 MB/s).
- **Compression speed — still lags.** Encode runs at roughly ¼–⅓ of the
  reference (dickens 33 vs 124 MB/s); the match-finder is the next target.

!!! note "Cross-compatibility (known limitation)"
    Our V2 FSE bitstream layout is **not yet byte-identical** to Apple's: a
    stream we produce does not decode with Apple's `lzfse` CLI and vice versa,
    despite both using the `bvx2` magic. Round-trip within our own codec is
    correct, and Apple-produced streams round-trip through our `Decompress`. Bit-
    exact interop in the *encode* direction is tracked as an open action item.

## Coverage and safety

`task test` reports **100% statement coverage** (`cover.out`). The corruption /
random-garbage fuzz suites assert **no panic**, so the decoder is safe to call on
adversarial input — bad data returns an error rather than crashing. The package
ships a [Taskfile](https://taskfile.dev) (`task lint|build|test|ci`) used by both
local development and CI; Renovate auto-merges patch/minor `gomod` updates.

## License

BSD-3-Clause.
