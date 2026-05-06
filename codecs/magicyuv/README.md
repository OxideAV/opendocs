# MagicYUV — clean-room wire-format specification

**Codec:** MagicYUV by Pavel Zlatev (`magicyuv.com`), a closed-source
commercial lossless intermediate codec.

**Version targeted:** the **v7** wire layout introduced in MagicYUV
2.0.0rc1 (2016-12-14) and still produced by current vendor builds.
Pre-2.0 wire formats (v0..v6) are out of scope.

This directory holds the natural-language specification and the
small data tables that an MIT-licensed decoder implementation can
consume. Everything here was produced by the clean-room process
described in [`../../README.md`](../../README.md).

## What's in here

| Path | What it is |
| --- | --- |
| `0X-*.md` files in this directory | The wire-format description — frame header, slice table, plane mapping, prediction, entropy coding, AVI carriage. Read in numerical order. |
| [`tables/`](tables/) | Small numeric tables (FOURCC enumeration, predictor IDs) intended to be loaded as build-time data. |

The spec also cites two public vendor-authored pages —
the [change log](https://www.magicyuv.com/change-log/) and
the [SDK page](https://www.magicyuv.com/sdk/) — by URL.
These are not redistributed here.

## Read order

If you are implementing a decoder or just trying to understand the
format, read in this order:

1. [`00-scope.md`](00-scope.md) — pixel-family scope, binary
   identification, chapter outline.
2. [`01-file-header-and-fourccs.md`](01-file-header-and-fourccs.md) —
   the 32-byte frame header and the 17 native FOURCCs.
3. [`02-slice-table.md`](02-slice-table.md) — slice-offset
   table, plane-major preamble, slice payload framing.
4. [`03-pixel-plane-mapping.md`](03-pixel-plane-mapping.md) —
   per-format-byte plane layout (G,B,R / Y,U,V / alpha-last; sample
   container width).
5. [`04-prediction-modes.md`](04-prediction-modes.md) — the
   three on-wire predictors (Left / Gradient / Median) with
   per-bit-depth wrap masks.
6. [`05-entropy-coding.md`](05-entropy-coding.md) — the
   run-length-encoded per-plane Huffman length descriptor and the
   non-RFC-1951 longest-length-first cumulative code construction.
   MSB-first, big-endian-byte. Raw-mode fallback documented.
7. [`06-avi-carriage.md`](06-avi-carriage.md) — RIFF/AVI
   container framing (the 72-byte `strf`, the per-frame `00dc` chunk).
8. [`tables/00-fourcc-table.csv`](tables/00-fourcc-table.csv) — the
   17-row FOURCC ↔ format_byte ↔ pixel-layout table, suitable for
   `include!`-style ingestion.
9. [`tables/01-predictor-table.csv`](tables/01-predictor-table.csv) —
   3-row predictor enumeration with formulae.

## Status

The wire-format specification covers v7 / MagicYUV Ultimate v2.4.2 in
full. The spec was developed against the proprietary vendor binary
and validated end-to-end by building reference decoder and encoder
implementations from it and exercising them against vendor-produced
streams. The validated surface includes:

- All 17 native v7 FOURCCs (8/10/12/14-bit RGB, 8/10-bit YUV, and
  Gray families) — self-round-trip byte-exactly.
- Byte-exact bidirectional interop with the proprietary v2.4.2 codec
  on the 8-bit RGB family; cross-decoder RGB-output equivalence on
  the 8-bit YUV family.
- 10/12/14-bit RGB and 10-bit YUV/Gray families validated via
  cross-codec round-trip.
- AVI carriage: AVI 1.0 single-`RIFF` and OpenDML 2.0 multi-`RIFF`
  (with `indx` super-index back-patching) read and write cleanly.

Two items are intentionally out of scope:

- The vendor's encoder-side **"Dynamic"** predictor strategy is a
  per-slice min-residual heuristic, not a wire-format requirement
  (per [`04-prediction-modes.md`](04-prediction-modes.md) §3). Any
  caller who picks a predictor satisfies the spec contract.
- Per-stream **`ix00`** index chunks are AVI-host-application
  territory (per [`06-avi-carriage.md`](06-avi-carriage.md) §6.1). A
  demuxer that walks the top-level RIFF chain reaches every frame
  without them.

Neither is a blocker for spec-compliant interoperability with
vendor-encoded streams.

## License and intent

The wire-format facts described here are functional necessities — the
bytes a decoder must read to produce the same pixels the encoder
intended. They are not creative expression. The chapter files in this
directory are the project's own natural-language description of
measured wire-format behaviour; the small tables under [`tables/`](tables/)
are factual extractions traceable to the proprietary binary by SHA-256
and file offset.

These artifacts are released under the
[Creative Commons Attribution 4.0 International](https://creativecommons.org/licenses/by/4.0/)
license; see [`../../LICENSE`](../../LICENSE) at the repository root
for the full text. A decoder built solely from this directory may
ship under any license its authors choose; CC BY 4.0 applies to the
documentation, not to independently-written code.

## Reading these files outside the project

Two notes for external readers:

- **References to `reference/binaries/...`** in the spec text are
  evidence anchors — they point to the proprietary MagicYUV binary
  on the staging machine, identified by SHA-256. The binary itself
  is not redistributed here. The hashes are sufficient for any
  reader who has obtained a copy of MagicYUV Ultimate v2.4.2 from
  the vendor to reproduce the offsets.
