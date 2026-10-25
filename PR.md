# Add compressed EROFS image reading

## Motivation

`go-erofs` currently rejects every image whose superblock advertises
compression (`sb.compr_algs != 0` or `EROFS_FEATURE_INCOMPAT_LZ4_0PADDING`):

```
unsupported compressed filesystem (FeatureIncompat=0x…, ComprAlgs=0x…): not implemented
```

`mkfs.erofs -z` is the default for many distributions and container-layer
producers, so a large share of real-world EROFS images cannot be opened at all.
This PR implements the read side of compressed EROFS for the four codecs defined
by the on-disk format.

## Implementation

New files:

- `compress.go` — codec layer for one physical cluster: LZ4 (raw block,
  including `EROFS_FEATURE_INCOMPAT_LZ4_0PADDING` leading padding), LZMA
  (microLZMA stream with the dictionary size taken from `COMPR_CFGS`), DEFLATE
  (raw stream) and ZSTD (standard frame). Also implements the partial-prefix
  decode required by `Z_EROFS_LI_PARTIAL_REF` (`lz4`/`lzma`).
- `compr_cfgs.go` — parser for the post-superblock `COMPR_CFGS` blob
  (`z_erofs_lz4_cfgs`, `z_erofs_lzma_cfgs`, `z_erofs_deflate_cfgs`,
  `z_erofs_zstd_cfgs`), including 4-byte record alignment and validation.
- `zmap.go` — cluster-map ("zmap") decoding for both on-disk index formats:
  - legacy/uncompacted `z_erofs_lcluster_index` (`EROFS_INODE_COMPRESSED_FULL`);
  - compacted index packs (`EROFS_INODE_COMPRESSED_COMPACT`), covering the
    4-byte initial region, 2-byte middle packs and the 4-byte trailer;
  - head/non-head chains, `BIG_PCLUSTER` / `CBLKCNT`, holes, partial
    references, deduplication and `ztailpacking`/fragment tails.
- `compressed_read.go` — per-inode zmap cache, extent mapping, decompressed
  physical-cluster LRU cache and the compressed read path.

Changed files:

- `erofs.go`: removes the blanket rejection, parses `COMPR_CFGS` at open time,
  routes `LayoutCompressedFull`/`LayoutCompressedCompact` inodes to the new
  loader, and adds `ReadAt` to image files for random access across cluster
  boundaries.
- `internal/disk/types.go`: new superblock feature bits and codec IDs.
- `erofs_test.go`: the old `lz4-unimplemented` case now asserts that a
  compressed image is readable.

The decoding logic is a Go port of the MIT-licensed
[`am-fs-erofs`](https://github.com/antimatter-studios/rust-fs-erofs) crate; the
license attribution is kept in the file headers.

### Scope / limitations

- **Read only.** Writing compressed images (`mkfs`) remains out of scope.
- LZMA configuration supports the dictionary size and the defined format (0).
  `lc`/`lp`/`pb` keep the kernel defaults used by erofs-utils.
- Chunk-based inodes are unchanged (already supported for uncompressed data).
- Fragments (`-E fragments`) are followed through the packed inode for tails;
  fragment data clusters themselves are read through the normal data path.

## Testing

New tests:

- `compress_test.go` — codec round trips for all four algorithms (including
  microLZMA stream construction), leading-zero padding removal, algorithm-ID
  mapping, partial-prefix decoding and error paths.
- `compr_cfgs_test.go` — record framing/alignment and per-codec config
  validation (dictionary size, unsupported formats, invalid window bits).
- `zmap_test.go` — compact-entry bit decoding for 2B/4B packs, pack geometry,
  compacted region offsets and entry classification.
- `compressed_read_test.go` — end-to-end images built with `mkfs.erofs` for
  `lz4`, `lzma`, `deflate` and `zstd` combined with: default index,
  `-E legacy-compress`, `-E ztailpacking`, `-E dedupe`, `-E fragments`,
  `-E dedupe,fragments`, `-C 16384` and `-C 65536`. Each image is checked for
  file contents (multi-cluster, small, empty, zero-filled, nested), hardlinks,
  symlinks and `ReadAt` at offsets crossing cluster boundaries. Truncated
  images must fail cleanly.
  Unsupported `mkfs.erofs` combinations are skipped, and `-short` skips the
  matrix.

Commands:

```sh
gofmt -l .
go vet ./...
go build ./...
go test ./...          # full matrix
go test -short ./...   # fast run
```

### Known pre-existing failures

On the development machine (erofs-utils 1.8.6) the unmodified `HEAD` already
fails the following tests; they are unrelated to this change and are not
modified by this PR:

- `TestErofs/*/chunk-index` — `mkfs.erofs --chunksize … --blobdev` fixture fails
  in `erofstest`.
- `TestCreateFSBlockSize/1024` — `fsck.erofs` rejects the generated image
  (`invalid sb_extslots 0`).

With this branch the failure set is identical to `HEAD`.

## Checklist

- [x] Compression codecs: LZ4, LZMA, DEFLATE, ZSTD
- [x] Legacy and compacted cluster maps
- [x] Partial references / deduplication / tail-packing / fragments
- [x] `ReadAt` random access
- [x] Unit tests and mkfs-based integration tests
- [x] `gofmt`, `go vet`, `go build`, `go test` clean (except pre-existing failures)
- [x] Module dependencies recorded in `go.mod`/`go.sum`
- [ ] Compression writing (separate PR)

## How to submit

This work lives on the `compressed-read` branch of a local clone. To open the
PR:

```sh
git remote add fork git@github.com:<your-user>/go-erofs.git
git push fork compressed-read
```

Then open a pull request from `<your-user>:compressed-read` against
`erofs/go-erofs:main` with the title and description above.
