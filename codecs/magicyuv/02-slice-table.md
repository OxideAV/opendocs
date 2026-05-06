# MagicYUV v7 — Slice-offset table

This chapter pins down the slice-offset table that immediately follows
the 32-byte v7 frame header described in
[`01-file-header-and-fourccs.md`](01-file-header-and-fourccs.md). The
table is the index that allows a decoder to locate each slice's
compressed payload without scanning. The entropy-coded payload itself
(the per-plane Huffman tables in the post-table preamble, and the
bit-stream of each slice) is the subject of
[`05-entropy-coding.md`](05-entropy-coding.md); this chapter
characterises only the **outer envelope** — the table layout, the
slice count rule, and the field at header offset `+0x1c`.

Two cross-cutting facts established here also affect adjacent
chapters:

- **The field at header offset `+0x1c` is `slice_height`**, not the
  earlier candidate name "height_extra (encoder-context height)".
  Behavioural traces (§3) show the field is the
  constant-per-codec-instance row count that determines how many
  slices each plane is partitioned into. In v2.4.2 with default
  configuration, `slice_height = 28` independent of frame height,
  thread count, codec variant, or pixel family. See §3.

- **The encoder allowlist mask `0xf1903f` polarity.** Reading the
  override block at `magicyuv.dll!0x69b976ab` carefully: the test
  against the mask sets the zero flag when
  `(1 << (format_byte − 0x67)) & mask == 0`, and the conditional
  branch is taken when the bit IS in the mask — jumping **over**
  the override block. So the override fires when
  `(1 << (format_byte − 0x67)) & 0xf1903f == 0`. The format bytes
  for which the override actually fires are the **complement**
  inside `[0x67, 0x7e]`: `{0x6d, 0x6e, 0x6f, 0x70, 0x71, 0x72,
  0x74, 0x75, 0x78, 0x79, 0x7a}` — the RGB-family non-8-bit
  (`M0RG, M0RA, M2RG, M2RA, M4RG, M4RA`) plus four reserved bytes.
  The override does NOT fire for the YUV/Gray family.
  `01-file-header-and-fourccs.md` §3.1 documents the override block
  with this polarity.

  Behavioural trace independently confirms a separate codec-variant
  clamp at `magicyuv.dll!0x69ba9060`–`0x69ba9070`
  (`(codec_variant - 1) > 1 ⇒ codec_variant = 2`); this clamp is
  independent of the `format_byte` mask and forces user
  `CompMethod` registry values of 3 or 4 down to byte 0x0b = 2,
  which matches the trace observation that **all M8G0 / M8RG /
  M8YA / etc. fixtures have byte 0x0b = 0x02 regardless of
  `CompMethod ∈ {1, 2, 3, 4}`**.

This chapter also resolves
`01-file-header-and-fourccs.md` §3.2's open question "Width / height
duplication at `0x18` / `0x1c`. When does `width != width_extra`?":
the answer for `0x1c` is "it isn't `height_extra` at all" — see §3.

## 1. Behavioural-trace evidence base

This is the first chapter of the spec built on behavioural traces.
The static-analysis-only chapters
[`01-file-header-and-fourccs.md`](01-file-header-and-fourccs.md)
and [`03-pixel-plane-mapping.md`](03-pixel-plane-mapping.md) flagged
the slice-table area as needing a behavioural harness; this chapter
uses one.

The harness is a 200-line C source built with mingw-w64
(`x86_64-w64-mingw32-gcc -O2 -static -lvfw32`) that calls
`ICOpen / ICCompressGetFormat / ICCompressBegin / ICCompress` on the
proprietary `magicyuv.dll` running in a Wine prefix. The codec is
registered as a VfW driver via the entries under
`HKLM\System\CurrentControlSet\Control\MediaResources\icm\`
(`VIDC.M8RG = magicyuv.dll`, etc.) and accepts standard `BI_RGB`
input frames; the 32-byte MAGY header plus its post-header payload
emerges in `lpOutput` of `ICCompress`. The harness lives in a
transient build directory and is not committed.

This chapter cites trace **fixtures** by filename (e.g.
`m8rg_64x64_zero.bin`). Each fixture is a single AVI-style MAGY
frame produced by feeding a synthetic input image into the codec at
a known `(format_byte, width, height, pattern)` tuple.

## 2. The slice table sits at header offset 0x20

Per `01-file-header-and-fourccs.md` §2 the v7 header is exactly 32
bytes; the byte at file offset `0x20` is the start of the
post-header payload. From every trace fixture (≥ 30 fixtures, native
FOURCCs `M8RG / M8RA / M8Y4 / M8Y2 / M8Y0 / M8YA / M8G0`,
dimensions `16×16` through `1024×1024` covering edge cases at the
slice-count boundaries), the post-header payload begins with a
contiguous run of **little-endian uint32 values that grow
monotonically**.

The encoder code that writes the first entry of this run is at
`magicyuv.dll!0x69b93f5c` (file `@0x12f5c`). The instruction
sequence at `magicyuv.dll!0x69b93f4d`–`0x69b93f5c` does the
following:

1. Reads `header_size` (= `0x20`) from offset `0x04` of the
   encoder context.
2. Adds it to the output-buffer base pointer to obtain the
   address of the first table slot at `output_buf + 0x20`.
3. Computes `(slice_count + 1) * 4` — the size of the slice
   table itself in bytes.
4. Stores that value as the initial `entry[0]` at the first
   table slot.

The slice-count value was computed earlier (at
`magicyuv.dll!0x69b93f01`–`0x69b93f1e`) by dividing the encoder's
per-slice metadata `std::vector` byte length by `0x478` (the
inverse-multiply constant `0xfaa11e6f` is the modular inverse of
`0x8f` = 143; the slice-descriptor item size is `8 * 0x8f =
0x478` bytes). So the first write to the table installs
`(N + 1) * 4` — the size of the slice table itself — as a
tentative initial value, and a later store at
`magicyuv.dll!0x69b93fae` updates the same slot to its final
running-cumulative value (matching `entry[1]`). The semantics of
the **final** entry values is what this chapter pins down; the
encoder's bookkeeping is not part of the wire format.

## 3. The field at header `+0x1c` is `slice_height`

The encoder writes the dword at header offset `+0x1c` from a value
loaded from offset `0x24` within the encoder context (the store
occurs at `magicyuv.dll!0x69b97699`). This field is a **constant per
encoder instance** that determines the number of slices the encoder
emits. We confirm this with the following behavioural trace.

For every fixture cited in this chapter, the dword at file offset
`+0x1c` of the MAGY header is **`0x0000001c` = 28**:

```
fixture                    bytes 0x18..0x1f
m8rg_64x64_zero.bin        40 00 00 00 1c 00 00 00
m8rg_256x256_zero.bin      00 01 00 00 1c 00 00 00
m8g0_64x64_zero.bin        40 00 00 00 1c 00 00 00
m8ya_64x64.bin             40 00 00 00 1c 00 00 00
m8y0_64x32.bin             40 00 00 00 1c 00 00 00
m8g0_64x16.bin             40 00 00 00 1c 00 00 00
```

The value at `+0x1c` does not vary with frame height, frame width,
codec variant, native FOURCC, registry `NumEncodeThreads`,
registry `AutoEncodeThreads`, or registry `CompMethod`. (The matrix
of registry overrides was tested with values `1, 4, 16, 64` for
threads and `1, 2, 3, 4` for `CompMethod`.)

The **slice count per plane** observed in the slice table (§4) is
exactly `ceil(height / slice_height)` for every fixture. The
following table is a verbatim summary of fixtures:

| Frame `h` | `slice_height` (header `+0x1c`) | Observed `slices_per_plane` | `ceil(h/28)` |
| --------- | -------------------------------: | --------------------------: | -----------: |
| 16        | 28                               | 1                           | 1            |
| 28        | 28                               | 1                           | 1            |
| 30        | 28                               | 2                           | 2            |
| 32        | 28                               | 2                           | 2            |
| 56        | 28                               | 2                           | 2            |
| 64        | 28                               | 3                           | 3            |
| 80        | 28                               | 3                           | 3            |
| 84        | 28                               | 3                           | 3            |
| 100       | 28                               | 4                           | 4            |
| 112       | 28                               | 4                           | 4            |
| 128       | 28                               | 5                           | 5            |
| 160       | 28                               | 6                           | 6            |
| 192       | 28                               | 7                           | 7            |
| 224       | 28                               | 8                           | 8            |
| 256       | 28                               | 10                          | 10           |
| 280       | 28                               | 10                          | 10           |
| 282       | 28                               | 11                          | 11           |
| 308       | 28                               | 11                          | 11           |
| 336       | 28                               | 12                          | 12           |
| 560       | 28                               | 20                          | 20           |

The boundary at `h = 280 → 282` is exact: `280 / 28 = 10` (no
remainder, slices stays at 10), `282 / 28 = 10.07` (ceiling jumps
to 11). This confirms `slice_height = 28` to its precise integer
value:

- `slice_height = 27` is rejected: it would predict
  `slices_per_plane = ceil(28/27) = 2` at `h = 28`, contradicting
  the observed `slices_per_plane = 1`.
- `slice_height = 29` is rejected: it would predict
  `slices_per_plane = ceil(282/29) = 10` at `h = 282`,
  contradicting the observed `slices_per_plane = 11`.
- `slice_height = 28` is the unique integer in `[27, 29]`
  consistent with both observations and matches the value the
  encoder writes at header `+0x1c`.

That `slice_height` is encoded *in* the header (rather than
inferable from constants in the binary) means a future codec
revision could change the value without changing version 7's outer
envelope. The decoder MUST read this field and not assume 28; in
v2.4.2, however, the encoder always writes `0x0000001c`.

An earlier candidate description of this field as "encoder-coded
vs. display height" is rejected by these traces.

## 4. Slice count

The total number of slices in a v7 frame is

```
total_slices = num_planes × slices_per_plane
slices_per_plane = ceil(height / slice_height)
```

where `num_planes` comes from the `format_byte` → backend mapping
established in
[`03-pixel-plane-mapping.md`](03-pixel-plane-mapping.md) §3 (1 for
Gray, 3 for RGB / YUV non-alpha, 4 for RGBA / YUVA).

The chroma planes of subsampled YUV formats (`M8Y2 / M8Y0 / M0Y2 /
M0Y0`) carry the **same number of slices as the luma plane**, even
though their pixel-row count is `height / sub_y`. Trace fixture
`m8y0_64x32_zero.bin` (M8Y0, 4:2:0, height 32, chroma height 16):
`slices_per_plane = 2` for all three planes (total slices = 6); the
chroma slice count is *not* `ceil(16 / 28) = 1`. This is observed
directly: the slice table has 7 entries (= 1 + total_slices for 6
slices = 7), and the per-plane huffman descriptors in the preamble
section repeat 3 times.

Cross-check across all FOURCCs at `64×64` produces
`slices_per_plane = 3`:

| FOURCC | `num_planes` | total slices observed | slice-table entries observed |
| ------ | -----------: | --------------------: | ---------------------------: |
| M8G0   | 1            | 3                     | 4                            |
| M8RG   | 3            | 9                     | 10                           |
| M8RA   | 4            | 12                    | 13                           |
| M8Y4   | 3            | 9                     | 10                           |
| M8Y2   | 3            | 9                     | 10                           |
| M8Y0   | 3            | 9                     | 10                           |
| M8YA   | 4            | 12                    | 13                           |

The relation `slice_table_entries = total_slices + 1` holds in every
fixture. See §5 for what the extra entry means.

## 5. The slice table layout

After the 32-byte header, the next `4 × (total_slices + 1)` bytes
are little-endian uint32 entries. Each entry is a **byte offset
relative to the byte after the header** (i.e. the absolute file
offset of the entry's referenced byte is `entry + 0x20`).

The table is followed immediately by a **per-frame preamble** (the
plane count, optional plane-assignment pad bytes, and per-plane
Huffman table descriptors; specified in `05-entropy-coding.md`),
and then the slice payloads.

### 5.1 Decoder rule

Let `N = total_slices`. The decoder reads `N + 1` entries
`entry[0]..entry[N]`. Then:

- Slice `k` (for `k = 0..N-2`) occupies the byte range
  `[entry[k+1] + 0x20, entry[k+2] + 0x20)` in the file.
- Slice `N-1` (the last slice) occupies the byte range
  `[entry[N] + 0x20, file_size)`.

Equivalently: slice `k` starts at `entry[k+1] + 0x20`, and ends at
`entry[k+2] + 0x20` for non-last slices or at `file_size` for the
last slice.

`entry[0]` is **ignored by this rule**. In every v2.4.2-produced
fixture surveyed, `entry[0] == entry[1]` (the encoder
writes the same value into both — see §5.3). A spec-compliant
decoder need not validate this equality; if entry[0] ≠ entry[1] in
some future encoder build, the rule above (which reads entry[1]
onward) gives the same result.

The byte range `[0x20 + 4*(N+1), entry[1] + 0x20)` is the **per-frame
preamble**: a small region that holds the plane count, optional
per-slice plane-assignment pad bytes, and per-plane Huffman
descriptors. Its layout is entropy-coding business (`05-entropy-
coding.md`); for the slice-table chapter it suffices that the
decoder can compute its bounds from the slice table itself:

```
preamble_start = 0x20 + 4 * (total_slices + 1)
preamble_end   = 0x20 + entry[1]
preamble_size  = entry[1] - 4*(total_slices + 1)
```

### 5.2 Worked example: `m8rg_64x64_zero.bin`

This is the canonical worked example for this chapter. The frame is
a `64×64` solid-zero RGB frame produced by passing an
all-zero `BI_RGB` 24-bit image through the codec keyed on
`M8RG`. Total file size = `1670` bytes (`0x686`). `format_byte =
0x65` (M8RG), `slice_height = 28`, `num_planes = 3`,
`slices_per_plane = ceil(64/28) = 3`, `total_slices = 9`.

Header (32 B):

```
0000:  4d 41 47 59 20 00 00 00  07 65 0c 02 00 00 20 00      MAGY ....e.... .
0010:  40 00 00 00 40 00 00 00  40 00 00 00 1c 00 00 00      @...@...@.......
```

Slice table: 10 uint32 LE entries at `0x20..0x47`:

```
0020:  44 00 00 00  44 00 00 00  26 01 00 00  08 02 00 00     entry[0]=0x44 entry[1]=0x44 entry[2]=0x126 entry[3]=0x208
0030:  4a 02 00 00  2c 03 00 00  0e 04 00 00  50 04 00 00     entry[4]=0x24a entry[5]=0x32c entry[6]=0x40e entry[7]=0x450
0040:  32 05 00 00  14 06 00 00                                entry[8]=0x532 entry[9]=0x614
```

Preamble: bytes `0x48..0x63` (28 B):

```
0048:  03 00 00 00 01 01 01 02 02 02 01 89 5d 08 89 9f  01 89 5d 08 89 9f 01 89 5d 08 89 9f
       \pc/\—— per-slice plane idx ——/\  plane 0 huff  /\  plane 1 huff  /\  plane 2 huff  /
```

The preamble decomposes as:
- **Byte 0** (`0x48`): `03` — plane count = 3.
- **Bytes 1..9** (`0x49..0x51`): `00 00 00 01 01 01 02 02 02` — per-slice plane index, one byte per slice. The 3 zero bytes are for plane 0's 3 slices; the 3 bytes valued 1 are for plane 1's 3 slices; the 3 bytes valued 2 are for plane 2's 3 slices.
- **Bytes 10..27** (`0x52..0x63`): three 6-byte per-plane Huffman descriptors.

Slice payloads:

```
slice[0]: 0x64..0x146  (226 B)   ; entry[1]+0x20 .. entry[2]+0x20
slice[1]: 0x146..0x228 (226 B)
slice[2]: 0x228..0x26a (66 B)
slice[3]: 0x26a..0x34c (226 B)
slice[4]: 0x34c..0x42e (226 B)
slice[5]: 0x42e..0x470 (66 B)
slice[6]: 0x470..0x552 (226 B)
slice[7]: 0x552..0x634 (226 B)
slice[8]: 0x634..0x686 (82 B)    ; entry[9]+0x20 .. file_end
```

The 226 / 226 / 66 size pattern repeats 3 times (once per plane).
The within-plane sizes correspond to slices that cover non-equal
heights: with `64` rows and `slice_height = 28`, the encoder
partitions as `[0..27], [28..55], [56..63]` — slice 0 and slice 1
cover 28 rows each, slice 2 covers the remaining 8 rows. The
compressed-size pattern (~226 B for full slices vs. ~66 B for the
truncated last slice) is consistent with this partition under any
fixed-rate predictor; see §6 for the row-partitioning rule.

### 5.3 The first-entry duplication (`entry[0] == entry[1]`)

Every fixture surveyed has `entry[0] == entry[1]`. The
two entries take on the same value because the encoder makes two
distinct stores to the entry[0] address during the slice-table
emit pass at `magicyuv.dll!0x69b93f5c`–`0x69b941b0`:

- The first store writes `4 * (N+1)` to entry[0]
  (at `magicyuv.dll!0x69b93f5c`).
- A later store updates entry[0] to its final value
  `4 * (N+1) + preamble_size` (the call into
  `magicyuv.dll!0x69bcdbb8` from `magicyuv.dll!0x69b94198` repacks
  the table; entry[0] ends with the same value as entry[1]).

Whether this is intentional ("entry[0] is an alias of entry[1]
giving the slice-0 boundary twice") or a vestigial structure
inherited from a pre-v7 layout where entry[0] meant something else
is not pinned down by static analysis alone. The vendor change-log
entry for 2.0.0rc1 ("Codec variant now top-level selection with
new FOURCC codes") is the v6 → v7 boundary; the old format
might have had different first-entry semantics. The recommendation
is: **decoders treat `entry[0]` as redundant and use
`entry[1..N]` to bound slices** (§5.1).

## 6. Per-plane row partitioning

For an image with height `H` and `slice_height = 28`, the planes
are partitioned into `slices_per_plane = ceil(H / 28)` slices. The
specific row ranges are:

- Slice `s` (for `s = 0..slices_per_plane - 2`) covers rows
  `[s × slice_height, (s + 1) × slice_height)`.
- The last slice (`s = slices_per_plane - 1`) covers rows
  `[(slices_per_plane - 1) × slice_height, H)`.

The last slice is shorter than 28 rows when `H` is not a multiple
of 28. For `H ≤ 28`, there is a single slice covering all rows.

For chroma planes of subsampled YUV formats, the **luma row count**
`H` is used (not the chroma row count `H / sub_y`). The chroma plane
is divided into the same number of slices, each covering rows
`[s × slice_height / sub_y, (s + 1) × slice_height / sub_y)` of the
chroma plane. For `M8Y0` (4:2:0, `sub_y = 2`), `slice_height` of
the chroma plane is effectively `14` rows.

This row-partitioning rule is consistent with the observed
compressed-size pattern in §5.2 (even-sized slices for slices
0..N-2, smaller slice for slice N-1) and with the trace observations
across the height matrix in §3. The rule is **not** derived
directly from binary static analysis (the per-slice row-range
arithmetic is buried in the encoder's worker-thread dispatch,
which is not fully decoded in this chapter); it is the simplest
partition consistent with `slice_count = ceil(H / slice_height)`
and is the only one that produces the observed compressed-size
pattern under a fixed-rate predictor. A definitive confirmation
would require either (a) running the codec on a non-zero input
pattern with known per-row delta and inspecting where the
per-slice bitstream "resets" the predictor state, or (b) decoding
the trace fixtures back to pixels using a Huffman-table extractor
and verifying the per-row content. The rule above is the
**operational hypothesis** for this chapter. Open question 7 in
§10 records this.

## 7. Preamble structure (slice-table-adjacent)

The bytes that immediately follow the slice table and precede
slice 0's payload form the **preamble**. Its full byte-level
layout is entropy-coding territory and properly belongs to
`05-entropy-coding.md`, but the preamble's first
`1 + total_slices` bytes are **slice-table-adjacent** and need
to be specified here for a decoder to be able to walk slices in
plane order.

### 7.1 Preamble byte layout

The preamble is structured as:

```
+----------------------+----------------------------------------------+
| Bytes                | Field                                        |
+----------------------+----------------------------------------------+
| 0                    | plane_count (uint8) = num_planes             |
| 1 .. total_slices    | per_slice_plane_index (uint8 × total_slices) |
| total_slices+1 .. ?  | per-plane Huffman descriptors (variable)     |
+----------------------+----------------------------------------------+
```

The total preamble size = `entry[1] - 4 × (total_slices + 1)`
(per §5.1). The portion `[total_slices+1 .. preamble_end)` carries
the per-plane Huffman descriptors and is entropy-coding business;
this chapter only specifies the first `1 + total_slices` bytes.

### 7.2 plane_count

The first byte of the preamble is `plane_count = num_planes`. For
the v7 native FOURCCs (per `03-pixel-plane-mapping.md` §3): 1 for
Gray (M8G0, M0G0), 3 for RGB / YUV non-alpha, 4 for alpha-bearing
FOURCCs.

This byte's value is redundant with the `format_byte` at header
`+0x09` — given the format_byte and the FOURCC enumeration, the
decoder already knows num_planes. Whether `plane_count` could
ever differ from what's implied by the format_byte is unverified;
a decoder probably should validate that `plane_count` matches the
format_byte's family-implied count.

### 7.3 per_slice_plane_index

The next `total_slices` bytes give, for each slice index
`s ∈ [0, total_slices)`, the **plane index** that slice `s`
belongs to.

Across all v2.4.2-encoder fixtures surveyed, the per-slice
plane-index sequence is **plane-major**:

```
per_slice_plane_index[s] = s / slices_per_plane    (integer division)
```

Equivalently, the slice indices are partitioned as:
`[0..slices_per_plane-1]` for plane 0,
`[slices_per_plane..2×slices_per_plane-1]` for plane 1, and so on.

Concrete examples:

| Fixture       | total_slices | per_slice_plane_index bytes                         |
| ------------- | -----------: | --------------------------------------------------- |
| M8G0 64×64    | 3            | `00 00 00`                                          |
| M8RG 64×16    | 3            | `00 01 02`                                          |
| M8RG 64×64    | 9            | `00 00 00 01 01 01 02 02 02`                        |
| M8RG 64×128   | 15           | `00 00 00 00 00 01 01 01 01 01 02 02 02 02 02`      |
| M8RA 64×64    | 12           | `00 00 00 01 01 01 02 02 02 03 03 03`               |
| M8YA 64×64    | 12           | `00 00 00 01 01 01 02 02 02 03 03 03`               |

The "plane-major" ordering is the only ordering observed; an
encoder could conceivably interleave planes (for parallelism),
but the v2.4.2 encoder does not. A spec-compliant decoder MUST
read `per_slice_plane_index` from the preamble (not assume the
plane-major ordering); the encoder's freedom to interleave is
preserved by the table format.

### 7.4 What plane index 0 means

Plane index 0 is the **first plane** in the format-byte's family
order (per `03-pixel-plane-mapping.md` §4..§6):
- For RGB family: G (green).
- For YUV family: Y (luma).
- For Gray family: Y (the single plane).

Plane indices then increment in the family's plane order:
RGB → G, B, R (then A); YUV → Y, U, V (then A); Gray → Y only.

## 8. Slice payload byte alignment

Each slice's compressed payload begins at the offset
`entry[k+1] + 0x20` and is a byte-aligned bitstream. From the
fixtures, the first two bytes of every slice (zero-content fixture)
are `00 01` followed by a run of `0xff` (or `0x7f` for some 10-bit
formats). The semantics of these prefix bytes are entropy-coding
business (`05-entropy-coding.md`), not slice-table.

The total file size is **always even** in v2.4.2: every fixture
examined had `file_size & 1 == 0`. This is consistent
with the change-log entry "Encoded frames are now padded to be an
even number of bytes" (1.2-rev1, 2016-10-13). The padding is
applied to the trailing byte of the last slice; in `M8RG 64×64
zero` the last slice's bitstream ends partway through byte
`0x67d` and bytes `0x67e..0x685` are zero-filled to preserve the
even-length invariant. The decoder MUST ignore any trailing
zero padding within the last slice's payload region; it MUST
not include the padding bytes in any per-slice byte count.

## 9. The full v7 frame layout (recap)

For a v7 MAGY frame with header values `H = height`,
`slice_height = 28` (constant in v2.4.2), `format_byte`
determining `num_planes`, and
`N = num_planes × ceil(H / slice_height)`:

```
+------------+---------------------------------------------------+
| File range | Field                                             |
+============+===================================================+
| 0x00..0x1f | Header (32 B; per `01-file-header-and-fourccs.md` §3) |
| 0x20..     | Slice table (N+1 uint32 LE entries; 4*(N+1) B)    |
| ...        | Preamble (per-plane Huffman, per `05-entropy-coding.md`; size |
|            |   = entry[1] - 4*(N+1))                           |
| ...        | Slice payloads (N slices, each at entry[k+1]+0x20)|
+------------+---------------------------------------------------+
```

Total size:

```
file_size =
  0x20                                            (header)
  + 4 * (N + 1)                                   (slice table)
  + (entry[1] - 4 * (N + 1))                      (preamble)
  + ∑(slice sizes)                                (slice payloads)
  + (1 if total is odd, else 0)                   (even-pad)
```

`entry[1]` in this formula is itself a function of all per-plane
Huffman tables; the slice-table chapter does not characterise
it numerically but the relation `entry[1] >= 4 * (N + 1)` is
invariant.

## 10. Open questions

1. **Encoder treatment of `slice_height` other than 28.**
   `slice_height` is a header field, not a constant. Whether the
   v2.4.2 encoder ever writes a different value (for a
   non-default GUI configuration not exercised by the
   harness) is unverified. The decoder, however, MUST read this
   field; assuming 28 would be a liveness hazard.

2. **Pre-v7 slice-table format.** The decoder accepts
   `version <= 7` (per `01-file-header-and-fourccs.md` §2). Whether
   v0..v6 used the same or different slice-table layout is out of
   scope; no v6-and-prior fixtures are available.

3. **Slice-payload internal structure.** Each slice begins with
   a 2-byte `00 01` prefix in the all-zero fixture but the
   semantic of the prefix is entropy-coding territory. See
   `05-entropy-coding.md`.

4. **Why `entry[0] == entry[1]`.** Whether the duplication is
   semantically meaningful (a separate "preamble end" boundary
   that happens to equal "slice 0 start") or is a vestigial v6
   layout artifact is not pinned down. Decoders that follow §5.1
   are robust to either interpretation.

5. **Variable `slice_height` per-codec-variant.** It is not tested
   whether `slice_height` differs between predictor modes
   (Left / Gradient / Median / Dynamic) for non-RGB formats
   where the override rule (`01-file-header-and-fourccs.md` §3.1)
   is bypassed. All fixtures surveyed had byte `0x0b = 0x02`
   (Gradient) due to the override.

6. **Encoder `CompMethod` vs. byte 0x0b.** Registry
   `CompMethod = 3` (Median) on RGB family produces a different
   compressed output than `CompMethod = 1, 2, 4`, but the wire
   byte at `0x0b` always reads `0x02`. The internal codec-variant
   clamp at `magicyuv.dll!0x69ba9060` forces
   `(codec_variant - 1) > 1 ⇒ codec_variant = 2`. The actual
   prediction path used by the encoder may diverge from what byte
   `0x0b` encodes. This is a `04-prediction-modes.md` concern;
   flagged here for cross-reference.

7. **Per-slice row-range partition (§6).** The
   "slices 0..N−2 cover slice_height rows; slice N−1 covers the
   remainder" rule is the operational hypothesis but is not
   directly derived from binary static analysis. Confirmation would
   require either decoding the trace fixtures back to pixel rows
   (using extracted Huffman tables) or examining the encoder's
   worker-thread dispatch to find the per-thread row range
   computation.

8. **Plane-major slice ordering normativity (§7.3).** All v2.4.2
   fixtures use `per_slice_plane_index[s] = s / slices_per_plane`.
   Whether the wire format normatively requires plane-major slice
   ordering, or whether the v2.4.2 encoder happens to choose this
   ordering and the decoder accepts arbitrary orderings, is
   unverified. A future encoder revision (or a competing encoder)
   could interleave planes for parallelism without changing the
   wire format. Decoders MUST read `per_slice_plane_index` and
   not assume the plane-major partition.
