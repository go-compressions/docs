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

## Coverage and safety

`task test` reports **100% statement coverage** (`cover.out`). The corruption /
random-garbage fuzz suites assert **no panic**, so the decoder is safe to call on
adversarial input — bad data returns an error rather than crashing. The package
ships a [Taskfile](https://taskfile.dev) (`task lint|build|test|ci`) used by both
local development and CI; Renovate auto-merges patch/minor `gomod` updates.

## License

BSD-3-Clause.
