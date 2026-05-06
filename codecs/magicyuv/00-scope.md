# MagicYUV — spec scope and roadmap

This document is the scope chapter of the MagicYUV v7 wire-format
specification. See [`../../README.md`](../../README.md) for the
project's clean-room methodology overview.

## In scope

The **version-7** wire format — the layout that landed with codec
release **2.0.0rc1** (2016-12-14) per
[the vendor change log](https://www.magicyuv.com/change-log/)
("Codec variant now top-level selection with new FOURCC codes"). The
v2.4.2 binary studied here only produces v7 streams (the version field
is hard-coded to `0x07` in the encoder header writer; the decoder
accepts `version <= 7`; see §1 chapter for citations).

Pixel families to cover, lower-bounded by the change log and confirmed
by static evidence in the binary's FOURCC enumerator:

| Family       | Bit depths   | Subsamplings                        | Alpha | Notes |
| ------------ | ------------ | ----------------------------------- | ----- | ----- |
| YUV          | 8 / 10       | 4:0:0 / 4:2:0 / 4:2:2 / 4:4:4       | yes (4:4:4 only) | "YUVA 4:4:4:4" only; no alpha for sub-sampled YUV |
| RGB          | 8 / 10 / 12 / 14 | planar GBR                      | yes (RGBA) | 4 bit-depth tiers |
| Gray         | 8 / 10       | (n/a)                               | n/a   | Y8 / Y0 families |

12-bit and 14-bit RGB families *are* present in the binary's FOURCC
table (`M2RG`/`M2RA` for 12-bit, `M4RG`/`M4RA` for 14-bit). The
change-log lower bound for YUV at 12/14-bit was speculative; in this
binary, YUV only has 8-bit and 10-bit codec FOURCCs (`M8Y*`/`M0Y*`).
RGB has 8/10/12/14-bit codec FOURCCs.

Color-matrix selection (Rec.601, Rec.709, full-range) is encoded in
the file header but the bit assignment is only partially pinned
down — see [`01-file-header-and-fourccs.md`](01-file-header-and-fourccs.md)
§3.1 ("Flags dword").

## Out of scope

- Pre-2.0 wire format (v0..v6). Out of scope per
  `../../README.md` §Scope; no fixtures available.
- Any color-space decoder applied *after* lossless decoding (Rec.601
  vs. Rec.709 conversion). MagicYUV is mathematically lossless on
  the wire; conversions are an application-level affair.
- The VLC-decoder side plugin (`libmagicyuv_plugin.dll`); decode-only,
  out of scope for this analysis.
- The Vegas-NLE plugin; thin wrapper.

## Source of truth

This specification draws on the following allow-listed inputs only:

1. **The proprietary MagicYUV Ultimate v2.4.2 binaries** at
   `../reference/binaries/`. The 64-bit codec at
   `system32/magicyuv.dll`
   (SHA-256 `2036e284cc07a2cb4d4b6b6cb04a1d24c0f97a0041c134fd669ee079b768902c`)
   is the primary subject; the 32-bit `syswow64/magicyuv.dll`
   (SHA-256 `9a6586674efc082cb94a1394382720c425c41d835790d1d252d38f8c35fe1545`)
   is consulted for cross-architecture sanity checks. Both are PE
   DLLs stripped to external PDB, exporting the standard VFW driver
   trio (`Configure`, `ConfigureAdobe`, `DriverProc`) confirmed via
   `objdump -p`.
2. **Vendor-authored documentation**: the change log at
   <https://www.magicyuv.com/change-log/> and the SDK page at
   <https://www.magicyuv.com/sdk/>.
3. **Public AVI / Video-for-Windows references**: Microsoft's
   `vfw.h` documentation on the MSDN-style documentation set
   (`ICCompress`, `BITMAPINFOHEADER` semantics, `biCompression`
   FOURCC convention). RFC 1951 §3.2.2 (canonical-Huffman recipe)
   is referenced for comparison in
   [`05-entropy-coding.md`](05-entropy-coding.md) §2.

Evidence anchored to (1) is cited as `<binary path>@<file offset>`
or `@<RVA>` in the format `0xXXXXXXXX`; the system32 64-bit DLL has
RVA→file offset = `RVA - 0x1000` for `.text` and
`RVA - 0xb000 + 0x247a00` for `.rdata` (computed from
`objdump -h`). Cross-references between RVA and instruction VMA
follow `objdump`'s default ImageBase of `0x69b80000`.

## Chapter outline

The chapters of this specification, in suggested read order:

1. [`01-file-header-and-fourccs.md`](01-file-header-and-fourccs.md)
   — the 32-byte v7 frame header layout and the 17 native FOURCCs
   accepted by the codec.

2. [`02-slice-table.md`](02-slice-table.md) — the slice-offset
   table layout immediately after the file header. The encoder
   unconditionally computes a slice partition; per-slice byte
   offsets are written contiguously starting at offset `0x20`
   (= header_size). The slice count is a function of the codec
   configuration (per-codec thread count / variant / interlace
   flag).

3. [`03-pixel-plane-mapping.md`](03-pixel-plane-mapping.md) — for
   every format byte in the v7 enumeration, what planes are emitted
   (Y/U/V/A; G/B/R/A), in what order, and whether the per-plane
   width/height inherits the frame width/height or sub-samples. The
   8-bit FOURCC bytes are well-formed (M8Y4/M8Y2/M8Y0/M8G0 ↔
   4:4:4/4:2:2/4:2:0/4:0:0 YUV; M8RG/M8RA = RGB/RGBA); the precise
   plane order and the 2-byte LE convention for 10/12/14-bit packed
   planes are pinned down here.

4. [`04-prediction-modes.md`](04-prediction-modes.md) — the
   per-slice predictor selection. The change log advertises
   "Gradient / Median / Dynamic"; the encoder allows a 4th internal
   mode "Left" (selectable via the registry's `CompMethod` value
   but not exposed in the GUI). The header byte at offset `0x0B`
   carries the prediction-mode ID (see
   [`01-file-header-and-fourccs.md`](01-file-header-and-fourccs.md)
   §3.0); the on-wire predictor selection is per-slice.

5. [`05-entropy-coding.md`](05-entropy-coding.md) — Huffman /
   canonical Huffman table representation for the per-symbol
   residuals. The change log entry "Removed 'Adaptive coding'
   setting (now always enabled)" (1.2, 2015-08-25) implies the
   wire format always carries a per-frame Huffman table; the
   table's representation (run-length-encoded length array; not
   RFC 1951 §3.2.2 canonical construction) is documented here.

6. [`06-avi-carriage.md`](06-avi-carriage.md) — how a v7 frame is
   wrapped in an AVI `00dc` chunk plus the `BITMAPINFOHEADER`
   `biCompression` convention. The static FOURCC table extracted
   in §4 of `01-file-header-and-fourccs.md` is the anchor;
   carriage details (whether biSize is fixed, whether any
   AVI-stream-header extension is required) are confirmed
   behaviourally.

## Conventions

- All multi-byte integer fields are **little-endian** unless noted.
  This is consistent with the magic-check sites in the binary, which
  compare against the immediate `0x5947414d` — the bytes
  `'M','A','G','Y'` loaded as a 32-bit little-endian word.
- "Frame" = one ICCompress output buffer = one AVI `00dc` payload =
  one MAGY-prefixed byte sequence.
- **Format byte** (§1 §3.0) is the 1-byte selector at offset `0x09`
  of the header. Range 0x65..0x7E in v7. Not the same thing as the
  FOURCC of the AVI stream (the FOURCC and format byte are linked
  through a binary-internal table; see §1 §2).
- "Codec variant" / "codec method" (§1 §3.0) = the prediction mode
  ID at offset `0x0B`. Range 0x01..0x04. The change log calls these
  "compression methods"; the binary calls them "codec variants" in
  user-facing strings (`248eae [%s] Codec variant: %s`).
