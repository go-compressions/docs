---
title: "skein"
weight: 130
---

# skein {#skein}

Pure-Go **Skein-512** (Skein 1.3), with key (MAC) pre-processing and any output
length. It is a port of the OpenZFS implementation (Doug Whiting, public
domain), and exists so that ZFS pool checksums (`checksum=skein`, Skein-512-256
keyed with the pool checksum salt) can be verified in Go.
[Repository →](https://github.com/go-compressions/skein)

## API {#api}

```go
d := skein.New(256, salt) // salt: the 32-byte org.illumos:checksum_salt
d.Write(block)
sum := d.Sum(nil)
```

`New(256, nil)` is plain Skein-512-256. `Clone` copies a keyed state, the
analogue of the ZFS context template.

## Verification {#verification}

`testdata/openzfs-2.3.0-vectors.txt` holds 1092 vectors (unkeyed 256, salted
256, 17-byte key 512, 1000-bit output) produced by compiling the OpenZFS C
sources verbatim behind a small shim; every line is checked by `go test`.

## Performance {#performance}

Single core, measured against the OpenZFS C code compiled with `clang -O2`; the
table is in the [README](https://github.com/go-compressions/skein#performance).
