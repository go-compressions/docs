---
title: "deflate"
weight: 60
---

# deflate {#deflate}

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

## API {#api}

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

// A stream followed by more bytes (zlib's Adler-32): n = input bytes it occupied.
orig, n, err := deflate.InflatePrefix(nil, zlibBody, 64<<20)
```

`NewWriter` accepts `DefaultCompression`, `NoCompression` and the range
`BestSpeed`..`BestCompression` (`-1`, `0`, `1`..`9`), mirroring `compress/flate`.

## How it works {#how-it-works}

**Whole-buffer decoder (v0.2.0).** `Inflate`, `InflateLimit` and `InflatePrefix`
follow libdeflate's design (11-bit literal/length and 8-bit offset tables, a
branchless 64-bit refill, word-copy matches). **Streaming decoder (v0.3.0).**
`NewReader` runs the same table decoder over a 64 KiB input window and a 32 KiB
history plus 256 KiB of output, resuming a block whenever it needs more input or the
output is full. A bare `io.ByteReader` source cannot be read ahead, so it keeps the
bit-serial count/symbol canonical-Huffman decoder (the zlib *puff* method). It handles all three block types — stored, fixed
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
the portable scalar fallback. As with the sibling [`lz4`]({{< relref "lz4.md" >}}), the kernel is
in the encoder's hot path, but end-to-end gains are bounded because encode time
is dominated by match-*finding* (hash chains), not match-*extension*.

## Performance {#performance}

Benchmarks are reproducible from committed, deterministically-generated corpora —
no external data:

```sh
go test -run=^$ -bench=. -benchmem
```

Encode numbers below are from an **Apple M4 Max** (`darwin/arm64`), Go 1.26.4, on
four 512 KiB corpora (2026-07-03). Throughput is MB/s (higher is better); *ratio* is
compressed ÷ original (lower is better).

### Encode (this package vs `compress/flate`) {#encode-this-package-vs-compressflate}

| corpus | level | ours MB/s | `flate` MB/s | ours vs `flate` | ratio ours | ratio `flate` |
|---|---|--:|--:|--:|--:|--:|
| text       | default | 18.2 | 16.9 | **1.08×** | 0.203  | 0.205  |
| text       | best    | 17.2 | 11.5 | **1.49×** | 0.203  | 0.202  |
| json       | default | 103  | 131  | 0.79×     | 0.105  | 0.100  |
| json       | best    | 97.7 | 45.6 | **2.14×** | 0.105  | 0.091  |
| binary     | default | 71.5 | 99.7 | 0.72×     | 0.834  | 0.835  |
| repetitive | default | 690  | 790  | 0.87×     | 0.0034 | 0.0030 |

### Decode (whole-buffer `Inflate`, v0.2.0, 2026-11-22) {#decode}

Apple M4 Max, single core, 128 KiB blocks, MB/s of output, decoding the same
streams (`l3` and `l9` are the levels that compressed them):

| corpus | ours | klauspost | libdeflate | zlib |
|---|--:|--:|--:|--:|
| synth-l3  | 1435 | 522 | 1714 | 1604 |
| synth-l9  | 1596 | 543 | 1414 | 1735 |
| gosrc-l3  | 1579 | 629 | 1692 | 1798 |
| gosrc-l9  | 2076 | 737 | 2120 | 2135 |
| binary-l3 | 665  | 251 | 704  | 700  |
| binary-l9 | 654  | 244 | 679  | 663  |

The streaming `NewReader` runs the same decoder since v0.3.0. Its throughput is not
published: the benchmark host stayed above the load limit. Exact allocation per full
read of a 512 KiB stream (4 MiB for `gosrc`), v0.2.0 against v0.3.0: `text` 390 KB /
39 allocs against 138 KB / 3, `json` 384 KB / 31 against 12 KB / 3, `binary` 373 KB /
19 against 12 KB / 3, `repetitive` 373 KB / 19 against 138 KB / 3, `gosrc` 569 KB /
251 against 138 KB / 3.

## Honest verdict {#honest-verdict}

- **Encode is a mixed picture, not a blanket win.** Faster on `text` (1.08×
  default, **1.49× best**) and `json` at the best level (**2.14×**, because
  `flate`'s `BestCompression` pays for exhaustive lazy matching), at essentially
  equal ratio on `text`. **Slower on `binary` (0.72×), `repetitive` (0.87×) and
  `json` at the default level (0.79×)** — roughly parity-to-slightly-slower on a
  mixed corpus. It also **allocates far more per operation (~390–400 vs ~27)** —
  per-block frequency/code/token buffers, the clearest thing left to optimize.
- **Whole-buffer decode is fast.** Since v0.2.0
  `Inflate`, `InflateLimit` and `InflatePrefix` use a libdeflate-style decoder that
  runs at 2.5 to 3 times klauspost/compress and close to libdeflate and zlib
  (table above). Since v0.3.0 `NewReader` runs the same decoder; its speed is not
  yet measured (see above).

In short: **correctness-first and wire-compatible; competitive encode on text,
libdeflate-class whole-buffer decode.** The streaming reader and the encoder on
`binary` and `json` at the default level are still slower than `compress/flate`;
the value here is a clean, fully-tested, SIMD-`matchlen` DEFLATE that
interoperates with it exactly.

## Untrusted input {#untrusted-input}

As of v0.1.0 `NewReader` buffers a bounded amount whatever the stream says (a
block yields to `Read` every 64 KiB of undelivered output), and `Inflate` stops
at `DefaultMaxOutput` (1 GiB) with `ErrTooLarge`; `InflateLimit` sets another
ceiling. A caller that collects a reader with `io.ReadAll` should still cap it
with `io.LimitReader`: DEFLATE expands up to 1032:1.

## License {#license}

BSD-3-Clause.
