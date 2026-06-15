# lz4c

[![CI](https://github.com/go-compressions/lz4c/actions/workflows/ci.yml/badge.svg)](https://github.com/go-compressions/lz4c/actions/workflows/ci.yml)
![coverage](https://img.shields.io/badge/coverage-100%25-brightgreen)

A pure-Go CLI for [`go-compressions/lz4`](lz4.md) — the **LZ4 block format** on
the command line, as a single CGO-free static binary.
[Repository →](https://github.com/go-compressions/lz4c)

## Usage

```sh
lz4c [-d] [-i input] [-o output] [-v]
```

- Compresses by default; `-d` / `--decompress` decompresses.
- `-i` / `--input` defaults to stdin.
- `-o` / `--output` defaults to stdout.
- `-v` / `--verbose` prints a summary line to stderr with byte counts,
  compression ratio (compress only), and elapsed time:

  ```text
  compressed 65536 → 12345 bytes (18.8%) in 4.231ms
  decompressed 12345 → 65536 bytes in 1.872ms
  ```

  Without `-v`, lz4c stays silent so the binary output on stdout is safe to
  pipe.

## Examples

```sh
# Compress a file to disk.
lz4c -i big.bin -o big.bin.lz4

# Round-trip through a pipe.
cat big.bin | lz4c | lz4c -d > restored.bin

# Show timing + ratio.
lz4c -v -i big.bin -o big.bin.lz4
```

## Container format

The `lz4` library exposes the LZ4 *block* codec only, which does not record the
decompressed length. lz4c frames each block with a fixed 12-byte header so a
stream is self-describing:

```text
magic   [4]byte  // "LZ4C"
rawLen  uint64   // little-endian length of the original data
...block...      // a standard LZ4 block, byte-for-byte
```

The payload after the header is an unmodified, standard LZ4 block and stays
wire-compatible with [`pierrec/lz4`](https://github.com/pierrec/lz4): strip the
12-byte header and any LZ4 block decoder consumes the remainder.

## Build

```sh
go build ./cmd/lz4c        # or: task build
```

## Coverage

`task test` reports **100% statement coverage** across all three packages:

| Package                    | Role                                  |
| -------------------------- | ------------------------------------- |
| `cmd/lz4c`                 | `main` and the cobra root command     |
| `cmd/lz4c/lz4io`           | container framing over the LZ4 blocks |
| `cmd/lz4c/internal/cmdio`  | shared stdin/stdout/file IO helpers   |

The package ships a [Taskfile](https://taskfile.dev) (`task lint|build|test|ci`)
used by both local development and CI; Renovate auto-merges patch/minor `gomod`
updates.

## License

BSD-3-Clause.
