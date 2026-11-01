---
title: "xz"
weight: 70
---

# xz {#xz}

[![CI](https://github.com/go-compressions/xz/actions/workflows/ci.yml/badge.svg)](https://github.com/go-compressions/xz/actions/workflows/ci.yml)
![coverage](https://img.shields.io/badge/coverage-100%25-brightgreen)

Pure-Go (`CGO=0`) **decoder** for `.xz`, raw **LZMA2** (7z, zip method 14) and
`.lzma` ("LZMA alone"), whose memory follows the decoded data and not the sizes
a header declares. It replaces the decode side of `github.com/ulikunitz/xz`:
the API has the same shape, so a consumer swaps the import path.
[Repository →](https://github.com/go-compressions/xz)

## Why {#why}

`ulikunitz/xz` v0.5.17 allocates every LZMA2 dictionary whole, from the block
header: `lzmafilter.go:75-77` takes `max(ReaderConfig.DictCap, header)` and
`lzma/buffer.go:23` does `make([]byte, size+1)` — up to 4 GiB per block from
about a hundred bytes, and no option bounds it. Here a block that declares a
dictionary of 1 GiB, 1.5 GiB or 4 GiB - 1 over 44 bytes of data allocates
22 672 bytes (`ulikunitz/xz`: 1 073 793 080 and 1 610 658 656 bytes for the
first two).

## API {#api}

```go
import (
    "github.com/go-compressions/xz"
    "github.com/go-compressions/xz/lzma"
)

r, err := xz.NewReader(file)                       // .xz, concatenated streams and padding
r, err := xz.ReaderConfig{MaxDictionary: 64 << 20, // ceiling, default 256 MiB
    SingleStream: true}.NewReader(file)

l, err := lzma.NewReader(file)                     // .lzma
l2, err := lzma.Reader2Config{DictCap: 8 << 20}.NewReader2(raw) // raw LZMA2, dictionary from the container
```

All readers stream; output is never buffered whole. Errors are typed and work
with `errors.Is`: `ErrCorrupt` (all malformed input; `ErrChecksum` and
`ErrTruncated` are also `ErrCorrupt`), `ErrTooLarge`, `ErrUnsupported`. Nothing
panics on input.

## The lazy window {#lazy-window}

The LZMA reference decoder (LZMA SDK `LzmaSpec.cpp`, public domain) allocates the
dictionary up front. This port starts the window at 16 KiB and doubles it as the
write position advances, wrapping only once it has reached the dictionary size.
The reference's own distance check is `dist <= Pos || IsFull`: before the window
is full every valid distance points into bytes already written, so the output is
exactly the reference's. Growth is checked only when the write position reaches
the end of the buffer, never per byte, and a growth that would pass
`MaxDictionary` returns `ErrTooLarge` before anything is allocated. The ceiling is
exact: data that fits decodes whatever dictionary the header declares, one byte
more is refused.

Block sizes and Index counts are validated against what arrives and never used to
size a buffer; the Index is checked with a running SHA-256 in constant memory.

## Container and filters {#filters}

Stream header and footer, block headers, Index, Stream Padding, concatenated
streams, checks None / CRC32 / CRC64 / SHA-256 (verified). Filters: LZMA2,
Delta, and the BCJ filters x86, PowerPC, IA-64, ARM, ARM Thumb, SPARC, ARM64 and
RISC-V — every filter squashfs XZ images can use. Unknown filters and reserved
check types are `ErrUnsupported`.

## Verification {#verification}

- Fixtures written by the `xz` command line tool (all checks, presets, wrapping
  window, lc/lp/pb, multi-block, all filters, `.lzma`), the 80 test files of XZ
  Utils (every good file decodes, every bad file is refused), and the output of
  `ulikunitz/xz`'s own writer.
- A symbol-level LZMA encoder in the tests reaches every decoder path, valid or
  not; truncation of every prefix; bit flips; bomb tests under `ulimit -v`.
- `FuzzXZ`, `FuzzLZMA`, `FuzzLZMA2`, differential against `ulikunitz/xz` where both
  decode: 6.83 M, 3.76 M and 5.96 M executions in 100 s each, no crasher.
- **100 % statement coverage**, gated on amd64, arm64, windows, 386 and — through
  test binaries under QEMU — riscv64, ppc64le, s390x and loong64.

## Performance {#performance}

Decode throughput against `ulikunitz/xz` and the C `xz -dc`, single core, is
measured by `go run ./internal/benchxz` on a 64 MiB corpus and refuses to run
above a 1-minute load of 8. The development machine stayed above 30, so **no
figure is claimed yet**.

## License {#license}

BSD-3-Clause. Ported from public-domain (LZMA SDK) and 0BSD (XZ Utils) sources,
credited in the code.
