# MagicYUV v7 — File header and FOURCC enumeration

This chapter describes the 32-byte MagicYUV v7 frame header and the
binary's enumeration of accepted AVI / VfW FOURCC codes. All claims
trace to byte-level evidence in the proprietary binary at
`reference/binaries/system32/magicyuv.dll` (SHA-256
`2036e284cc07a2cb4d4b6b6cb04a1d24c0f97a0041c134fd669ee079b768902c`,
PE32+ x86-64). Where useful, the corresponding code in
`reference/binaries/syswow64/magicyuv.dll` (i386) is cited as a
cross-architecture sanity check.

Notation: `<file>@0xNNNNNN` is a file offset; `<file>!0xNNNNNN` is
the instruction VMA as printed by `objdump -d` with the binary's
default ImageBase `0x69b80000` (system32) or `0x6be80000`
(syswow64). The system32 DLL has `.text` at file `0x400` (RVA
`0x1000`) and `.rdata` at file `0x248a00` (RVA `0x4b000`) per
`objdump -h`.

## 1. The MAGY magic and acceptance test

Every v7 frame starts with the 4-byte ASCII signature `M A G Y`
(`0x4d 0x41 0x47 0x59`). When loaded as a little-endian 32-bit word
it equals `0x5947414d`; the binary uses exactly this immediate in
every magic-check site. Twelve such sites exist in the 64-bit DLL;
representative ones:

- `magicyuv.dll!0x69badfd2` (file `@0x2cfd2`): decoder
  buffer-validation entry — compares the 32-bit word at the
  buffer base against the magic constant `0x5947414d`.
- `magicyuv.dll!0x69bae1e4` (file `@0x2d1e4`): buffer-with-skip
  variant of the same magic comparison.
- `magicyuv.dll!0x69b97507` (file `@0x16507`): encoder writes the
  magic constant `0x5947414d` into the first 4 bytes of the
  output buffer.
- `magicyuv.dll!0x69b9773c` (file `@0x1673c`): second encoder
  header writer (allocator path) — also stores the same
  little-endian magic at the buffer base.

The 32-bit DLL uses the same immediate
(`syswow64/magicyuv.dll!0x6be81d00` etc.); the wire format is
architecture-agnostic.

The decoder rejects a packet shorter than `0x20` bytes before even
checking the magic. At `magicyuv.dll!0x69badfcc` the decoder
compares the packet length against `0x1f` (31) and branches to
the rejection path when length ≤ 31; only then does the magic
comparison at `0x69badfd2` execute. The `0x1f` threshold is
correct for "must be at least 32 bytes" because the test is
`length <= 31 => reject`. This proves the v7 header is **at
least** 32 bytes; §3 below shows it is exactly 32 bytes.

## 2. The header is 32 bytes

The encoder header writer at `magicyuv.dll!0x69b97507`–`0x69b97540`
writes exactly the following sequence into the output buffer (the
buffer base is the encoder's output pointer; the encoder also
clears all 32 bytes explicitly):

| Site (VMA)                | Field                                                | Value written |
| ------------------------- | ---------------------------------------------------- | ------------- |
| `0x69b97507`              | magic at offset `0x00` (4 bytes)                     | `0x5947414d` ('MAGY') |
| `0x69b9750d`              | header_size at offset `0x04` (uint32)                | `0x20` |
| `0x69b97514`              | version at offset `0x08` (uint8)                     | `0x07` |
| `0x69b97518`–`0x69b97520` | three single-byte fields at offsets `0x09..0x0b`     | cleared to `0x00` |
| `0x69b97524`              | flags dword at offset `0x0c` (uint32)                | cleared to `0x00` |
| `0x69b97532`–`0x69b97540` | four 32-bit fields at offsets `0x10/0x14/0x18/0x1c`  | cleared to `0x00` |

So the header is **exactly 32 bytes**, and the next byte (`@0x20`)
is the start of the post-header payload (the slice-offset table; see
the roadmap in `00-scope.md`).

The decoder uses the 32-bit word at `0x04` as the **declared**
header size and rejects any packet whose total length differs.
At `magicyuv.dll!0x69bae4e0` (file `@0x2d4e4`) the decoder
compares the packet length against the header's declared size at
offset `0x04` and rejects on mismatch. The encoder always writes
`0x20` here, so the declared header size = total packet size for
v7. Any future format revision could grow the header and the
decoder would still reject mismatched sizes.

The decoder's version-acceptance test at `magicyuv.dll!0x69bae4ea`
reads the byte at header offset `0x08` and rejects when its
value exceeds `0x7` — i.e. the decoder accepts versions 0..7.
This binary's encoder only emits 7. Versions 0..6 may exist in
older streams; this chapter does not characterise them.

## 3. Header field layout

Combining the encoder-write sequence (§2) and the decoder-read
sequence (`0x69bae306`–`0x69bae31b`, `0x69bae3a4`, `0x69bae42`–
`0x69bae4ca`), the v7 header is:

```
+--------+-------------------------------------------------------+
| Offset | Field                                                 |
+========+=======================================================+
|  0x00  | magic         (4 B, fixed = 'MAGY' = 0x5947414d LE)   |
|  0x04  | header_size   (uint32 LE, fixed = 0x00000020 in v7)   |
|  0x08  | version       (uint8,    fixed = 0x07 in v7)          |
|  0x09  | format_byte   (uint8;    see §3.0 and §4)             |
|  0x0a  | aux_byte      (uint8;    see §3.0)                    |
|  0x0b  | codec_variant (uint8;    see §3.0)                    |
|  0x0c  | flags         (uint32 LE; see §3.1)                   |
|  0x10  | width         (uint32 LE; ICCompress lpbiInput width) |
|  0x14  | height        (uint32 LE)                             |
|  0x18  | width_extra   (uint32 LE; usually = width)            |
|  0x1c  | slice_height  (uint32 LE; = 28 in v2.4.2)             |
+--------+-------------------------------------------------------+
```

The field at `+0x1c` is `slice_height` — the constant the encoder
uses to partition each plane into slices (= 28 in v2.4.2 for every
fixture observed). The slices-per-plane count is
`ceil(height / slice_height)`. A decoder MUST read this field and
not assume 28. See `02-slice-table.md` §3 for the behavioural
evidence.

### 3.0  The three format/variant bytes at `0x09..0x0b`

The encoder writes these three bytes in three separate
single-byte stores:

| Encoder site (VMA) | Header offset | Field             |
| ------------------ | ------------- | ----------------- |
| `0x69b975b8`       | `0x09`        | format_byte (lookup output) |
| `0x69b9763c`       | `0x0a`        | aux_byte (property-descriptor field at `+0x24`) |
| `0x69b97643`       | `0x0b`        | codec_variant (config) |

**format_byte (`0x09`)** — selects the codec's internal pixel
family. Its values are anchored in the FOURCC table (§4); the
range emitted by this encoder is `0x65..0x7e` (12 of the 26
possible values are valid). The decoder accepts the same range
plus a fall-through to a property-table lookup; see the encoder
allowlist mask in §3.2.

**aux_byte (`0x0a`)** — the encoder fetches this from the
per-pixel-family property-tree record at offset `+0x24` of the
record (load at `magicyuv.dll!0x69b97634`). It is **not** a format
byte alias; it is a field of the property descriptor that travels
with the format byte. The decoder confirms the round-trip by
comparing the stream's aux_byte against the property descriptor's
`+0x24` field and rejecting on mismatch, at
`magicyuv.dll!0x69bae1a5` and `magicyuv.dll!0x69bae3a8`.

**`aux_byte = max_huffman_code_length`** for the format_byte's
bit-depth tier, matching the third numeric parameter of the
corresponding HuffCoder specialisation (per `05-entropy-coding.md`
§1.5):

| bit-depth | aux_byte (= 0x0a) | format_byte set                          |
| --------- | ----------------- | ---------------------------------------- |
| 8         | `0x0c` (= 12)     | `{0x65, 0x66, 0x67, 0x68, 0x69, 0x6a, 0x6b}` |
| 10        | `0x0e` (= 14)     | `{0x6c, 0x6d, 0x6e, 0x73, 0x76, 0x7b}`   |
| 12        | `0x10` (= 16)     | `{0x6f, 0x70}`                            |
| 14        | `0x12` (= 18)     | `{0x71, 0x72}`                            |

This is cross-checked against `ICCompressGetFormat` extradata for
every native FOURCC. The decoder check at
`magicyuv.dll!0x69bae3a8` validates the stream's aux_byte
against the per-format-byte property descriptor's `+0x24`
field and rejects on mismatch — i.e. a decoder hard-coding
`aux_byte = 12` will reject valid 10/12/14-bit streams.

**codec_variant (`0x0b`)** — the prediction-mode (compression
method). The encoder reads it from its config field at offset
`0x74` within the encoder context (load at
`magicyuv.dll!0x69b9763f`). User-facing labels confirmed at
`magicyuv.dll@0x2498b4`:
"Dynamic", "Median", "Gradient", and a fourth string "Left" at
`@0x2498cc` referenced only via the registry value `CompMethod`
(`magicyuv.dll!0x69b875dc`–`0x69b87c25`):

| Header byte at `+0x0b` | Mode label   | Notes                          |
| ---------------------- | ------------ | ------------------------------ |
| `0x01`                 | "Left"       | not exposed in the GUI         |
| `0x02`                 | "Gradient"   | "faster decode"                |
| `0x03`                 | "Median"     | "fast encode, good compression"|
| `0x04`                 | "Dynamic"    | "max. compression"             |

This table lists the **encoder-side configured-mode** values, NOT
the on-wire byte at header `+0x0b` in v2.4.2 streams. The encoder's
clamp at `magicyuv.dll!0x69ba9060` forces
`(CompMethod − 1) > 1 ⇒ CompMethod = 2` *before* writing
`+0x0b`, so **byte `+0x0b` is always `0x02` in v2.4.2 output**.
The actual on-wire predictor selection is the **per-slice
`predictor_id` byte** (see `04-prediction-modes.md` §1.2) with
values `{0x01, 0x02, 0x03}` only; value `0x04` (would-be Dynamic)
never reaches the wire.

A decoder targeting v2.4.2 streams may ignore byte `+0x0b`
beyond the `version <= 7` range check on `+0x08`. A decoder
built defensively for hypothetical future streams should
preserve the existing `<= 2` shortcut at `0x69bae320` plus the
property-tree fallback for `> 2`.

The change log entry for 2.0.0rc1 (2016-12-14) listed these as
"compression methods"; the binary's user-facing string at
`@0x248eae` is "Codec variant: %s". Both names refer to the same
field. The decoder accepts `codec_variant <= 2` only via the
"shortcut" path: at `magicyuv.dll!0x69bae320` the decoder
compares the codec_variant byte against `0x2` and branches to
the skip-lookup fast path when the value is ≤ 2.

Values 3 and 4 (Median, Dynamic) take the property-tree lookup
path. Value 0 is rejected by the property-tree path's range
check at `magicyuv.dll!0x69bae1af`, which rejects when the
header byte at `+0x0b` exceeds `0x2`. Whether the decoder also
reads `0x01`/`0x02` correctly via the shortcut is not separately
confirmed; the encoder's allowlist (§3.2) suggests all of
1..4 are valid in produced streams.

### 3.1  Flags dword at `0x0c..0x0f`

The encoder writes this field via OR-accumulation at
`magicyuv.dll!0x69b97647`–`0x69b9767a`. The sequence does the
following:

- Read the encoder's `ColorMatrix` config (offset `0x68` within
  the encoder context). If it equals 1, skip the matrix
  contribution; otherwise mask to the low nibble, shift left by
  20 (so bits 20..23 hold the matrix nibble), and OR into the
  flags accumulator already present at offset `0x0c` of the
  output header.
- Read the `Interlaced` config (offset `0x70` within the encoder
  context). If non-zero, OR `0x2` into the accumulator (bit 1
  set when interlaced).
- Read the `FullRangeYUV` config (offset `0x78` within the
  encoder context). If non-zero, OR `0x4` into the accumulator
  (bit 2 set when full-range).

So at minimum the flags dword has these documented bits:

| Bit  | Mask         | Meaning (vendor name)   | Encoder source (encoder context offset)  |
| ---- | ------------ | ----------------------- | ---------------------------------------- |
| 1    | `0x00000002` | Interlaced              | `+0x70` (`Interlaced` registry value)    |
| 2    | `0x00000004` | Full-range YUV (0..255) | `+0x78` (`FullRangeYUV` registry value)  |
| 20-23| `0x00f00000` | ColorMatrix nibble      | `+0x68` (`ColorMatrix` registry value), low nibble |

The decoder reads bit 2 (`0x4`) explicitly: at
`magicyuv.dll!0x69bae311` (file `@0x2d311`) it shifts the flags
dword right by 2 and isolates the low bit, treating the result
as a boolean ("full-range YUV"). Bit 1 (Interlaced) is also read
by the encoder allowlist code at `magicyuv.dll!0x69b976ab` and
used by the post-validation path (see §3.2). Bits 20..23 (the
ColorMatrix nibble) are used at the application/conversion
layer, not the lossless codec layer; the exact 16 possible
values aren't all populated by the GUI, which only exposes
Rec.601 and Rec.709 (per
[the vendor change log](https://www.magicyuv.com/change-log/)
v0.9.2-beta).

The encoder also clears bit 2 in a corner case where the format
byte is in a specific subset. At
`magicyuv.dll!0x69b9769c`–`0x69b976bb` the encoder:

1. Computes the biased index `format_byte - 0x67`. If it
   exceeds `0x17`, the override block is reached unconditionally
   (the range-guard at `magicyuv.dll!0x69b9769c` falls through
   to the override path).
2. Otherwise builds the bit `1 << (format_byte - 0x67)` and
   tests it against the mask `0xf1903f`.
3. When the bit IS in the mask, branches over the override
   block.
4. When the bit is NOT in the mask (or when the range guard
   fell through), clears bit 2 of the flags dword, forces
   the codec_variant byte at header `+0x0b` to `0x02`
   (Gradient), and writes the updated flags back to header
   `+0x0c`.

So the override fires when
`(1 << (format_byte − 0x67)) & 0xf1903f == 0` for in-range
format bytes, and unconditionally for out-of-range format bytes.

The mask `0xf1903f` (per the formula `bit b → format byte 0x67 + b`)
decodes to format bytes
`{0x67, 0x68, 0x69, 0x6a, 0x6b, 0x6c, 0x73, 0x76, 0x77, 0x7b, 0x7c, 0x7d, 0x7e}`
— the M8\* / M0\* family **YUV/Gray** codes. These are the format
bytes for which the override does **NOT** fire; the override fires
for the **RGB family** (the complement
`{0x6d, 0x6e, 0x6f, 0x70, 0x71, 0x72, 0x74, 0x75, 0x78, 0x79, 0x7a}`
inside `[0x67, 0x7e]`), and also for `0x65 = M8RG` and
`0x66 = M8RA` via the out-of-range fallthrough — meaning
**8-bit RGB also takes the override path**.

In v2.4.2 streams the byte at `+0x0b` is **always `0x02`** anyway,
due to the separate format-independent
`codec_variant − 1 > 1 ⇒ 2` clamp at `magicyuv.dll!0x69ba9060`
(see the §3.0 codec_variant note above). The mask's polarity
matters only for determining whether **flags bit 2** also gets
cleared at the same time. RGB-family streams have flags bit 2
cleared by this override; YUV/Gray-family streams retain whatever
flags bit 2 was set to by the encoder's earlier OR-accumulation.

### 3.2  Width/height fields at `0x10..0x1f`

The encoder writes the four 32-bit fields at header offsets
`0x10/0x14/0x18/0x1c` from `magicyuv.dll!0x69b9767d`–`0x69b97699`:

- `0x10` ← width (the ICCompress argument width).
- `0x18` ← the same width value again.
- `0x14` ← height (the ICCompress argument height).
- `0x1c` ← a value loaded from offset `0x24` of the encoder
  context (an "extra" field).

So `0x10` and `0x18` carry the same width value and `0x14` carries
the height. The value at `0x18` is "width again" and the value at
`0x1c` is `slice_height` (per `02-slice-table.md` §3); they are
sourced from different places in the encoder context. The exact
semantic of `0x18` (likely display/coded distinction or
stride-related) would need a synthetic non-square-dimension frame
to disambiguate.

## 4. The FOURCC enumeration

The encoder publishes a list of 81 (format-id, FOURCC, bpp,
flag, extra) records via the VFW `ICCompressGetFormat` /
`ICDecompressQuery` callbacks. The list is built at
`magicyuv.dll!0x69b88820`–`0x69b8a000` (file `@0x07820`–`@0x09000`)
into a 0x1400-byte buffer (room for 0x1400 / 0x14 = 256 entries
of 20 bytes each; 81 used).

Each encoder-side staging block writes the same 5 fields onto a
stack record (offsets within the staging frame) before pushing:

| Stack offset | Field         | Contents                                                       |
| ------------ | ------------- | -------------------------------------------------------------- |
| `+0x30`      | `format_id`   | one of the format bytes `0x65..0x7e` (or non-MAGY internal IDs)|
| `+0x34`      | `fourcc`      | little-endian 4-char code                                      |
| `+0x38`      | `bpp`         | bits per pixel                                                 |
| `+0x3c`      | `flag`        | 0 = no alpha, 1 = with alpha                                   |
| `+0x40`      | `extra`       | family ID or `0xff`                                            |

Extracted in source-code order from the 64-bit DLL (decimal
values for fmt_id where they correspond to header format bytes
in §3.0; "—" means an internal-only ID used for VFW negotiation
with non-MAGY input formats):

### 4.1  MagicYUV-native FOURCCs (header `format_byte`)

These are the FOURCCs used in the AVI stream's
`BITMAPINFOHEADER.biCompression` field when the codec encodes
into its own format. The first byte of each FOURCC is the
ASCII letter `M`. The second byte encodes the bit depth tier
(`8` = 8-bit, `0` = 10-bit, `2` = 12-bit, `4` = 14-bit). The
last two bytes encode the family.

The per-FOURCC `aux_byte` (header `+0x0a`) value is included as a
column in the table below (confirmed for every native FOURCC).

| `format_byte` | FOURCC  | bpp | alpha-flag | aux_byte | family-ID `extra` | Description (vendor) |
| ------------- | ------- | --- | ---------- | -------- | ----------------- | -------------------- |
| `0x65`        | `M8RG`  | 24  | 0          | `0x0c`   | `0x00` (RGB 8)    | RGB, 8-bit           |
| `0x66`        | `M8RA`  | 32  | 1          | `0x0c`   | `0x00` (RGBA 8)   | RGBA, 8-bit          |
| `0x67`        | `M8Y4`  | 24  | 0          | `0x0c`   | `0x07`            | YUV 4:4:4, 8-bit     |
| `0x68`        | `M8Y2`  | 24  | 0          | `0x0c`   | `0x04`            | YUV 4:2:2, 8-bit     |
| `0x69`        | `M8Y0`  | 24  | 0          | `0x0c`   | `0x06`            | YUV 4:2:0, 8-bit     |
| `0x6a`        | `M8YA`  | 32  | 1          | `0x0c`   | `0x08`            | YUVA 4:4:4:4, 8-bit  |
| `0x6b`        | `M8G0`  | 24  | 0          | `0x0c`   | `0x09`            | Gray (4:0:0), 8-bit  |
| `0x6c`        | `M0Y2`  | 20  | 0          | `0x0e`   | `0x0b`            | YUV 4:2:2, 10-bit    |
| `0x6d`        | `M0RG`  | 30  | 0          | `0x0e`   | `0x21`            | RGB, 10-bit          |
| `0x6e`        | `M0RA`  | 40  | 1          | `0x0e`   | `0x1f`            | RGBA, 10-bit         |
| `0x6f`        | `M2RG`  | 36  | 0          | `0x10`   | `0x21`            | RGB, 12-bit          |
| `0x70`        | `M2RA`  | 48  | 1          | `0x10`   | `0x1f`            | RGBA, 12-bit         |
| `0x71`        | `M4RG`  | 42  | 0          | `0x12`   | `0x21`            | RGB, 14-bit          |
| `0x72`        | `M4RA`  | 56  | 1          | `0x12`   | `0x1f`            | RGBA, 14-bit         |
| `0x73`        | `M0G0`  | 10  | 0          | `0x0e`   | `0x0b`            | Gray (4:0:0), 10-bit |
| `0x76`        | `M0Y4`  | 30  | 0          | `0x0e`   | `0x2f`            | YUV 4:4:4, 10-bit    |
| `0x7b`        | `M0Y0`  | 15  | 0          | `0x0e`   | `0x0b`            | YUV 4:2:0, 10-bit    |

(Format byte `0x6a` for `M8YA` is written explicitly at
`magicyuv.dll!0x69b888f1` into the staging record's `format_id`
slot, and the FOURCC immediate `0x4159384d` written at
`magicyuv.dll!0x69b88900` is the little-endian encoding of
`M8YA`.)

The decoder's format-byte allowlist is encoded as a bitmask. At
`magicyuv.dll!0x69bae461`–`0x69bae480` the decoder:

1. Computes the biased index `format_byte - 0x65`.
2. Falls through to a fallback path when the index exceeds
   `0x14`.
3. Otherwise builds the bit `1 << index` and tests it against
   the mask `0x19bf03`. When the bit IS in the mask the
   decoder branches to the property-tree lookup path;
   otherwise it falls into the fast RGB-style decoder path.

Decoding mask `0x19bf03` (bit `b` ↔ format byte `0x65 + b`) gives
`{0x65, 0x66, 0x6d, 0x6e, 0x6f, 0x70, 0x71, 0x72, 0x74, 0x75, 0x78, 0x79}`
— the 12 RGB/RGBA/12bit/14bit format bytes. Format bytes outside
this mask but in `0x65..0x79` (the YUV/Gray ones) take a
**different** decoder path (the property-tree lookup), which means
this bitmask describes formats that bypass the tree lookup and
go straight to the simpler RGB-style decoder. This is consistent
with packed-RGB layouts having a direct decoder fast path.

The encoder allowlist (§3.1) uses a different mask `0xf1903f` and
a different bias (`format_byte - 0x67`); decoding that gives
`{0x67 0x68 0x69 0x6a 0x6b 0x6c 0x73 0x76 0x77 0x7b 0x7c 0x7d 0x7e}`
— all-YUV/Gray formats. Format bytes `0x77`, `0x7c`, `0x7d`,
`0x7e` are present in the encoder allowlist but **not** present
in the published FOURCC enumeration of §4.1; they may correspond
to MagicYUV internal modes used for color-space conversion only
(no AVI carriage). See open question 4 in §6.

### 4.2  Non-MagicYUV FOURCCs (input-only; for VFW negotiation)

When the codec is asked to compress a frame whose source FOURCC
is not MagicYUV-native, it accepts the following inputs (these
also appear in the GUI's "supported formats" listing at
`@0x24a6d0`–`@0x24a9b8`):

| Internal `format_id` | FOURCC   | bpp | alpha-flag | Description                      |
| -------------------- | -------- | --- | ---------- | -------------------------------- |
| `0x01`               | `DIB ` (= 0x20424944) | 32 | 2 | RGB packed in DIB        |
| `0x01`               | (no fourcc, 32-bit)  | 32 | 2 | "RGB32"                  |
| `0x02`               | `DIB `   | 32  | 3          | RGB packed in DIB (variant)      |
| `0x02`               | (no fourcc, 32-bit)  | 32 | 3 | "RGB32 BI_BITFIELDS"     |
| `0x03`               | `DIB `   | 24  | 2          | RGB24 packed in DIB              |
| `0x03`               | (no fourcc, 24-bit)  | 24 | 2 | "RGB24"                  |
| `0x04`               | `YUY2`, `YUYV` | 16 | 0   | YUV 4:2:2 packed                 |
| `0x05`               | `UYVY`, `2vuy`, `2Vuy`, `HDYC` | 16 | 0 | YUV 4:2:2 (UYVY ordering) |
| `0x06`               | `YV12`   | 12  | 0          | YUV 4:2:0 planar                 |
| `0x07`               | `YV24`   | 24  | 0          | YUV 4:4:4 planar                 |
| `0x08`               | `AYUV`   | 32  | 1          | YUVA 4:4:4:4                     |
| `0x09`               | `Y8  `, `Y800`, `GREY` | 8 | 0 | Gray (4:0:0)                     |
| `0x0a`               | `I420`, `IYUV` | 12 | 0   | YUV 4:2:0 planar                 |
| `0x0b`               | `v210`, `V210` | 20 | 0   | YUV 4:2:2 packed 10-bit          |
| `0x0c`               | `r210`   | 30  | 0          | RGB 10-bit packed                |
| `0x0d`               | `R10k`   | 30  | 0          | RGB 10-bit packed (BE variant)   |
| `0x17`               | `b64a`   | 64  | 1          | BGRA 16-bit (BE)                 |
| `0x18`               | `b64a`   | 64  | 0          | BGR 16-bit (BE)                  |
| `0x19`               | `b48r`   | 48  | 0          | BGR 16-bit                       |
| `0x1a`               | `RBA@` (= 0x40414252) | 64 | 1 | RGBA 16-bit (LE)         |
| `0x1b`               | `RBA@`   | 64  | 0          | RGB 16-bit (LE)                  |
| `0x1c`               | `RGB0` (= 0x30424752) | 48 | 0 | RGB 16-bit (DPX-style)   |
| `0x1d`               | `I444`   | 24  | 0          | YUV 4:4:4 planar (alt of YV24)   |
| `0x1e`               | `Y1\x00\x0a` (= 0x0a003159) | 10 | 0 | Gray 10-bit               |
| `0x1f`               | `BRA@` (= 0x40415242) | 64 | 1 | BGRA 16-bit (LE)         |
| `0x20`               | `BRA@`   | 64  | 0          | BGR 16-bit (LE)                  |
| `0x21`               | `BGR0` (= 0x30524742) | 48 | 0 | BGR 16-bit (DPX-style)   |
| `0x22`               | `P210`   | 20  | 0          | YUV 4:2:2 10-bit semi-planar     |
| `0x23`               | `I2AL`, `Y3\x0a\x0a` | 20 | 0 | YUV 4:2:2 10-bit planar          |
| `0x24`               | `GBAL`, `G3\x00\x0a` | 30 | 0 | RGB 10-bit planar GBR            |
| `0x25`               | `G3\x00\x0c` | 36 | 0          | RGB 12-bit planar GBR            |
| `0x26`               | `G3\x00\x0e` | 42 | 0          | RGB 14-bit planar GBR            |
| `0x27`               | `GBFL`, `G3\x00\x10` | 48 | 0 | RGB 16-bit planar GBR            |
| `0x28`               | `G4\x00\x0a` | 40 | 1          | RGBA 10-bit planar GBR           |
| `0x29`               | `G4\x00\x0c` | 48 | 1          | RGBA 12-bit planar GBR           |
| `0x2a`               | `G4\x00\x0e` | 56 | 1          | RGBA 14-bit planar GBR           |
| `0x2b`               | `G4\x00\x10` | 64 | 1          | RGBA 16-bit planar GBR           |
| `0x2c`               | `v308`   | 24  | 0          | YUV 4:4:4 packed 8-bit           |
| `0x2d`               | `v408`   | 32  | 0          | YUVA 4:4:4:4 packed 8-bit        |
| `0x2e`               | `v408`   | 32  | 1          | YUVA 4:4:4:4 (alpha)             |
| `0x2f`               | `v410`   | 30  | 0          | YUV 4:4:4 packed 10-bit          |
| `0x30`               | `I4AL`, `Y3\x00\x0a` | 30 | 0 | YUV 4:4:4 10-bit planar          |
| `0x33`               | `I4FL`, `Y3\x00\x10` | 48 | 0 | YUV 4:4:4 16-bit planar          |
| `0x39`               | `P010`   | 15  | 0          | YUV 4:2:0 10-bit semi-planar     |
| `0x3a`               | `I0AL`, `Y3\x0b\x0a` | 15 | 0 | YUV 4:2:0 10-bit planar          |
| `0x3b`               | `I2FL`, `Y3\x0a\x10` | 32 | 0 | YUV 4:2:2 16-bit planar          |
| `0x3c`               | `I0FL`, `Y3\x0b\x10` | 24 | 0 | YUV 4:2:0 16-bit planar          |
| `0x3d`               | `Y1\x00\x10` (= 0x10003159) | 16 | 0 | Gray 16-bit               |

These are encoder-input candidates only — when the codec receives
an AVI/DirectShow input frame in one of these formats, it
internally converts to the matching MagicYUV format byte (per the
GUI explanation at `@0x24a6d0`: "YUV 4:4:4 - (YV24,I444,v408,v308)"
collapses to `M*Y4`).

Vendor user-facing name strings (with file offsets) that document
this collapsing are:

- `@0x24a6bb`: `RGB - (RGB32,RGB24)`
- `@0x24a6d0`: `YUV 4:4:4 - (YV24,I444,v408,v308)`
- `@0x24a6f8`: `YUV 4:2:2 - (YUY2,YUYV,UYVY,2vuy,HDYC)`
- `@0x24a71f`: `YUV 4:2:0 - (YV12,I420,IYUV)`
- `@0x24a73c`: `YUV 4:0:0 - (Y8,Y800,GREY)`
- `@0x24a757`: `RGBA - (RGB32 with alpha)`
- `@0x24a771`: `YUVA 4:4:4:4 - (AYUV,v408)`
- `@0x24a790`: `YUV 10-bit 4:4:4 - (v410,I4AL,Y3[00][10])`
- `@0x24a7c0`: `YUV 10-bit 4:2:2 - (v210,P210,I2AL,Y3[10][10])`
- `@0x24a7f0`: `YUV 10-bit 4:2:0 - (P010,I0AL,Y3[11][10])`
- `@0x24a820`: `YUV 10-bit 4:0:0 - (Y1[00][10])`
- `@0x24a840`: `YUV 16-bit 4:4:4 - (I4FL,Y3[00][16])`
- `@0x24a868`: `YUV 16-bit 4:2:2 - (I2FL,Y3[10][16])`
- `@0x24a890`: `YUV 16-bit 4:2:0 - (I0FL,Y3[11][16])`
- `@0x24a8b8`: `YUV 16-bit 4:0:0 - (Y1[00][16])`
- `@0x24a8d8`: `RGB 10-bit - (r210,R10k,G3[0][10],GBAL)`
- `@0x24a900`: `RGBA 10-bit - (G4[0][10])`
- `@0x24a91a`: `RGB 12-bit - (G3[0][12])`
- `@0x24a933`: `RGBA 12-bit - (G4[0][12])`
- `@0x24a94d`: `RGB 14-bit - (G3[0][14])`
- `@0x24a966`: `RGBA 14-bit - (G4[0][14])`
- `@0x24a980`: `RGB 16-bit - (b48r,b64a,BGR[48],BRA[64],G3[0][16],GBFL)`
- `@0x24a9b8`: `RGBA 16-bit - (b64a,G4[0][16])`

The 16-bit YUV / RGB families *appear* in this naming surface but
none of the per-format Mxxx FOURCCs in §4.1 cover bit-depth = 16.
The encoder must convert any 16-bit input down to one of the
14-bit family members (`M4RG`/`M4RA`) or up to a 16-bit-padded
14-bit storage; **this chapter does not pin down the conversion
choice**, but the published `format_byte` set (4.1) is the entire
set of bit-depths emitted on the wire.

## 5. Cross-architecture sanity check

The 32-bit DLL `syswow64/magicyuv.dll` exhibits the same magic
constant (compared against `0x5947414d` at
`syswow64/magicyuv.dll!0x6be81d00`), the same FOURCC strings (e.g. `M8RG`
at file `@0x8bae`, `M0G0` at `@0x8ec7`, `M4RA` at `@0x9101`), and
the same `MAGY 10b` / `MAGY 12b` / `MAGY 14b` user-facing labels
in the codec-info dialog text. The wire format is independent of
the architecture of the codec implementation.

## 6. Open questions

1. **Width / height duplication at `0x18` / `0x1c`.** When does
   `width != width_extra` at `0x18`? Likely encoder-coded vs.
   display distinction (16-pel-aligned coded width vs. true display
   width), but unverified. (The field at `0x1c` is `slice_height`,
   per `02-slice-table.md` §3.)
2. **Mask `0x19bf03` decoder fast path.** A behavioural trace would
   confirm which format bytes actually go through the simpler RGB
   path vs. the full property-tree lookup, and why.
3. **Format bytes `0x77` `0x7c` `0x7d` `0x7e`.** Present in the
   encoder format-byte allowlist but absent from the published
   FOURCC enumeration. Internal-only conversion targets, or
   alternative wire-format codes not exercised by the v2.4.2
   encoder?
4. **Pre-v7 stream format.** Decoder accepts versions 0..7; the
   v2.4.2 binary only writes 7. Can the binary decode older
   streams? No fixtures available to test.
