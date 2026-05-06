# MagicYUV v7 — extracted factual tables

These CSVs are factual extractions of content already documented in
the spec chapters one level up
([`..`](../)). They are provided in machine-readable form for an
implementer (e.g. a Rust implementation crate) to consume via
`include_str!` / a CSV reader. Per
[`../README.md`](../README.md) §Methodology, every value here is
justifiable from the proprietary binary or a cited public reference;
nothing is "the model produced it".

## Files

### `00-fourcc-table.csv` — Native FOURCC ↔ format_byte ↔ aux_byte table

17 rows. Each row is one native v7 codec FOURCC and the
header-byte values an encoder/decoder needs to round-trip it.

- **Source**: [`../01-file-header-and-fourccs.md`](../01-file-header-and-fourccs.md)
  §4.1 (the 17-row FOURCC enumeration), with the `aux_byte` column
  cross-checked against `ICCompressGetFormat` strf extradata captures
  for every native FOURCC (per-bit-depth values 12/14/16/18).
- **Family classification** anchored to the per-layout RTTI block
  in `reference/binaries/system32/magicyuv.dll` at file
  `0x256a00..0x256d40` (per
  [`../03-pixel-plane-mapping.md`](../03-pixel-plane-mapping.md)
  §1).

The CSV columns: `format_byte, fourcc, bit_depth, planes, bpp, alpha,
aux_byte, family, subsampling, description`.

A decoder MUST reject a stream whose `aux_byte` does not match the
expected value for the format byte's bit-depth tier (decoder check at
`magicyuv.dll!0x69bae3a8`).

### `01-predictor-table.csv` — Per-slice predictor enumeration

3 rows, one per valid `predictor_id` byte at slice payload offset +1.

- **Source**: [`../04-prediction-modes.md`](../04-prediction-modes.md)
  §1.2 (the predictor ID table) and §4.2..§4.4 (the per-predictor
  formulas). Decoder dispatch at
  `magicyuv.dll!0x69b95acd..0x69b95aed`; Median cmov chain at
  `0x69bb6a80..0x69bb6c2f`.

The CSV columns: `predictor_id, name, formula, decoder_kernel_anchor,
notes`.

The "Dynamic" vendor label is **not** a wire-format predictor ID; it
is an encoder strategy that picks one of {Left, Gradient, Median} per
slice, and the on-wire `predictor_id` byte is always one of `{0x01,
0x02, 0x03}`.

## Extraction provenance

Both CSVs were transcribed by hand from the corresponding spec
chapters. No tool was used to generate them from the binary directly;
the *evidence* for each value already exists in the spec text, and
the CSV is a re-formatting for machine consumption.

If an implementer finds a discrepancy between a CSV value and either
the spec narrative or the binary, the **spec** is authoritative and
the CSV should be updated.

## What is NOT here

- **Static Huffman tables.** The codec is per-frame adaptive (see
  [`../05-entropy-coding.md`](../05-entropy-coding.md) §6.1).
  There is no static residual code book to extract. Per-frame Huffman
  length descriptors are parsed from the wire per
  `05-entropy-coding.md` §1.1, and codes are constructed by the
  algorithm at `05-entropy-coding.md` §2.
- **The 26-entry property descriptor table** referenced by the
  decoder's aux_byte echo-check (the comparison against the
  property descriptor's `+0x24` field at
  `magicyuv.dll!0x69bae3a8`). Indices outside the 17 emitted
  FOURCCs (`0x65..0x7e` minus `{0x6a, 0x68, 0x69, 0x74, 0x75,
  0x77, 0x78, 0x79, 0x7a, 0x7c, 0x7d, 0x7e}` partial gaps) are
  reserved / undocumented; the v2.4.2 encoder does not produce
  them.
- **Per-Backend kernel addresses** for predictor and Huffman code
  paths. These are documented inline in `04-prediction-modes.md` §4.6
  and `05-entropy-coding.md` §1.5 but are decoder-internal RVAs that
  an implementer should not hard-code (the implementer writes their
  own predictor / Huffman decoder; the binary RVAs are evidence
  anchors, not inputs).
