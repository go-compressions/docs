---
title: "edonr"
weight: 120
---

# edonr {#edonr}

A pure-Go implementation of the **Edon-R′ 512** hash function, the function
behind the OpenZFS `checksum=edonr` property, with the keyed (salted) form ZFS
computes for a pool that has a checksum salt. BSD-3-Clause, a **clean-room**
implementation written from the published specification.
[Repository →](https://github.com/go-compressions/edonr)

## API {#api}

```go
sum := edonr.Sum(data)              // [64]byte, Edon-R 512

d := edonr.New()                    // streaming, implements hash.Hash
d.Write(block)
digest := d.Sum(nil)

k := edonr.NewKeyed(poolSalt[:])    // ZFS salted checksum
k.Write(block)
blkCksum := k.Sum(nil)[:32]         // ZFS stores the first 32 bytes
```

`Size` is 64, `BlockSize` is 128.

## Which function this is {#which-function}

The 2008 NIST submission (Edon-R) and the 2009 tweak Edon-R′ differ by a
feedback term. The OpenZFS output is Edon-R′, and the feedback form matches
the OpenZFS oracle on every vector. The keyed form is not part of the
specification; it was found and verified through black-box oracle queries only.

## Provenance {#provenance}

Written from the published Edon-R and Edon-R′ documents only. OpenZFS was used
as a black-box oracle (a binary that prints a digest); no OpenZFS source was
read while writing the code. The README states the two places where the record
is not perfectly clean.

## Performance {#performance}

Plain Go, no allocation. The measured numbers against the OpenZFS C code are in
the [README](https://github.com/go-compressions/edonr#performance), and the
README reports the scalar gap as an open defect. A multi-buffer SIMD kernel and
multi-core hashing of many blocks are being added.
