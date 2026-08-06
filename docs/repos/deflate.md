# deflate

Pure-Go (`CGO=0`) implementation of the **DEFLATE** compressed data format
([RFC 1951](https://www.rfc-editor.org/rfc/rfc1951)) — both the encoder
(*deflate*) and the decoder (*inflate*). The DEFLATE bit stream is the payload
carried inside zlib (RFC 1950) and gzip (RFC 1952), and exactly what the
standard library's `compress/flate` reads and writes.
[Repository →](https://github.com/go-compressions/deflate)

`deflate` is **bidirectionally wire-compatible with `compress/flate`**, verified
by differential fuzzing: `flate.NewReader` decodes every stream our `Writer`
produces, and our `NewReader` decodes every stream `flate.NewWriter` produces —
both back to the original bytes.

## API

```go
import "github.com/go-compressions/deflate"

// Streaming.
var buf bytes.Buffer
w, _ := deflate.NewWriter(&buf, deflate.DefaultCompression)
w.Write(data)
w.Close()

r := deflate.NewReader(&buf)
out, _ := io.ReadAll(r)

// One-shot convenience helpers.
comp := deflate.Deflate(nil, src)          // src -> DEFLATE
orig, _ := deflate.Inflate(nil, comp)      // DEFLATE -> src
```

`NewWriter` accepts `DefaultCompression`, `NoCompression` and the range
`BestSpeed`..`BestCompression` (`-1`, `0`, `1`..`9`), mirroring `compress/flate`.

## How it works

**Decoder.** A bit reader feeds a compact count/symbol canonical-Huffman decoder
(the zlib *puff* method). It handles all three block types — stored, fixed
Huffman and dynamic Huffman (including the code-length run-length alphabet) —
plus LZ77 back-reference copies over a 32 KiB sliding window, with every
malformed-input path (reserved block type, bad stored length, over-subscribed or
truncated codes, out-of-range distances) returning a typed error.

**Encoder.** A hash-chain match-finder (4-byte hash, per-level chain depth)
produces LZ77 tokens; match *extension* runs through the
[`matchlen`](https://github.com/go-compressions/matchlen) SIMD common-prefix
kernel. Each block is emitted as whichever of **stored / fixed / dynamic** is
smallest, with length-limited (≤ 15-bit, ≤ 7-bit for the code-length alphabet)
Huffman codes generated so they are always *complete* — accepted by any RFC 1951
decoder without single-code special cases.

`deflate` pins [`go-compressions/matchlen`](https://github.com/go-compressions/matchlen)
v0.1.1, whose kernel ships real SIMD on **four** of Go's 64-bit targets — amd64
(SSE2), arm64 (NEON), loong64 (LSX) and riscv64 (RVV); other architectures use
the portable scalar fallback. As with the sibling [`lz4`](lz4.md), the kernel is
in the encoder's hot path, but end-to-end gains are bounded because encode time
is dominated by match-*finding* (hash chains), not match-*extension*.

## Performance

Benchmarks are reproducible from committed, deterministically-generated corpora —
no external data:

```sh
go test -run=^$ -bench=. -benchmem
```

Numbers below are from an **Apple M4 Max** (`darwin/arm64`), Go 1.26.4, on four
512 KiB corpora (2026-07-03). Throughput is MB/s (higher is better); *ratio* is
compressed ÷ original (lower is better).

### Encode (this package vs `compress/flate`)

| corpus | level | ours MB/s | `flate` MB/s | ours vs `flate` | ratio ours | ratio `flate` |
|---|---|--:|--:|--:|--:|--:|
| text       | default | 18.2 | 16.9 | **1.08×** | 0.203  | 0.205  |
| text       | best    | 17.2 | 11.5 | **1.49×** | 0.203  | 0.202  |
| json       | default | 103  | 131  | 0.79×     | 0.105  | 0.100  |
| json       | best    | 97.7 | 45.6 | **2.14×** | 0.105  | 0.091  |
| binary     | default | 71.5 | 99.7 | 0.72×     | 0.834  | 0.835  |
| repetitive | default | 690  | 790  | 0.87×     | 0.0034 | 0.0030 |

### Decode (both decoders reading the same `flate`-produced stream)

| corpus | level | ours MB/s | `flate` MB/s | ours vs `flate` |
|---|---|--:|--:|--:|
| text       | default | 253  | 460   | 0.55× |
| json       | default | 343  | 973   | 0.35× |
| binary     | default | 68.7 | 216   | 0.32× |
| repetitive | default | 464  | 10971 | 0.04× |

## Honest verdict

- **Encode is a mixed picture, not a blanket win.** Faster on `text` (1.08×
  default, **1.49× best**) and `json` at the best level (**2.14×**, because
  `flate`'s `BestCompression` pays for exhaustive lazy matching), at essentially
  equal ratio on `text`. **Slower on `binary` (0.72×), `repetitive` (0.87×) and
  `json` at the default level (0.79×)** — roughly parity-to-slightly-slower on a
  mixed corpus. It also **allocates far more per operation (~390–400 vs ~27)** —
  per-block frequency/code/token buffers, the clearest thing left to optimize.
- **`flate` wins decode across the board (≈1.8–3×, ~25× on the highly
  repetitive input).** Ours is a small, correct, bit-serial count/symbol Huffman
  decoder that copies back-references byte-by-byte; `flate`'s is a mature
  table-driven decoder. Decode is inherently bit-serial and not SIMD-amenable, so
  this package targets correctness and wire-compatibility here rather than raw
  speed.

In short: **correctness-first and wire-compatible; competitive encode on text,
slower decode than the standard library.** If raw throughput on arbitrary data
is the priority, `compress/flate` is still the better choice; the value here is a
clean, fully-tested, SIMD-`matchlen` DEFLATE that interoperates with it exactly.

## License

BSD-3-Clause.
