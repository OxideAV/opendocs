# MagicYUV v7 — AVI / VfW carriage

This is the final chapter of the MagicYUV v7 wire-format specification.
It pins down how a v7 frame — the `MAGY`-prefixed packet defined by
[`01-file-header-and-fourccs.md`](01-file-header-and-fourccs.md) §3
through [`05-entropy-coding.md`](05-entropy-coding.md) — is wrapped in
a Microsoft AVI / Video-for-Windows (VfW) container, and what the
encoder publishes via the `BITMAPINFOHEADER` and any extradata block.

All claims trace to byte-level evidence in the proprietary binary
`reference/binaries/system32/magicyuv.dll` (SHA-256
`2036e284cc07a2cb4d4b6b6cb04a1d24c0f97a0041c134fd669ee079b768902c`,
PE32+ x86-64), to behavioural-trace fixtures generated through the
Wine VFW harness, and to the public Microsoft AVI / VfW reference
documentation cited inline.

Notation follows
[`01-file-header-and-fourccs.md`](01-file-header-and-fourccs.md):
`<file>@0xNNNNNN` is a file offset and `<file>!0xNNNNNN` is the
instruction VMA at the binary's default ImageBase `0x69b80000`.

This chapter takes as given:

- The 32-byte v7 frame header layout from
  [`01-file-header-and-fourccs.md`](01-file-header-and-fourccs.md) §3.
- The 17 native FOURCCs from
  [`01-file-header-and-fourccs.md`](01-file-header-and-fourccs.md)
  §4.1 (M8RG, M8RA, M8Y4, M8Y2, M8Y0, M8YA, M8G0, M0Y2, M0RG, M0RA,
  M2RG, M2RA, M4RG, M4RA, M0G0, M0Y4, M0Y0).
- The codec's status as a VfW driver: the DLL exports
  `Configure`, `ConfigureAdobe`, and `DriverProc` (verified via
  `objdump -p` — three exports, ordinals 1/2/3 at RVAs `0xde90`,
  `0x7f00`, `0xe120`).

## 1. AVI container layout overview

A MagicYUV v2.4.2 stream lives inside a standard Microsoft AVI
RIFF container as defined in the [AVI RIFF File
Reference](https://learn.microsoft.com/en-us/windows/win32/directshow/avi-riff-file-reference)
(Microsoft Learn). The codec itself does **not** write AVI files; it
exposes the standard VfW driver entry points (`ICCompressGetFormat`,
`ICCompressBegin`, `ICCompress`, etc.) and lets the host application's
AVI muxer (typically Microsoft's `avifil32.dll` / `AVIFile*` API)
assemble the chunks. From the codec's perspective, each call to
`ICCompress` produces **one MAGY frame** (the
`'MAGY'`-prefixed byte sequence specified in chapters 01..05); the muxer
packages it into a `00dc` chunk verbatim.

The expanded RIFF form per the Microsoft reference is:

```
RIFF ('AVI '
      LIST ('hdrl'
            'avih' ( <AVIMAINHEADER, 56 B> )
            LIST ('strl'
                  'strh' ( <AVISTREAMHEADER, 64 B> )
                  'strf' ( <BITMAPINFOHEADER, 40 B>
                           <extradata = MAGY frame header, 32 B> )
                  [ 'strn' ( <stream name> ) ]   ; optional
                  [ 'strd' ( <stream-handler-private data> ) ]   ; not used by MagicYUV
                 )
           )
      LIST ('movi'
            '00dc' ( <MAGY frame> )      ; one chunk per encoded frame
            '00dc' ( <MAGY frame> )
            ...
           )
      [ 'idx1' ( <AVIOLDINDEX entries> ) ]    ; AVI 1.0 index
      [ 'indx' ( <super-index> ) ]            ; OpenDML 2.0; muxer-side, see §6
     )
```

The bracketed elements are optional. MagicYUV streams in practice
always include `idx1` (AVIF_HASINDEX is set). OpenDML 2.0 indexing
is muxer-side and is briefly addressed in §6.

## 2. The `00dc` per-frame chunk

### 2.1 Chunk identity

Per [Microsoft Learn — AVI RIFF File
Reference](https://learn.microsoft.com/en-us/windows/win32/directshow/avi-riff-file-reference)
("Stream Data ('movi' List)"), each video frame chunk has a FOURCC
of the form `<sn>dc`, where `<sn>` is the two-digit zero-padded
stream index in ASCII and `dc` (= ASCII bytes `0x64 0x63`) marks
"compressed video frame". For a single-stream MagicYUV file, the
chunk ID is `00dc` (= LE DWORD `0x63643030`).

The codec's response to `ICM_COMPRESS` (= 0x4000 + 0x000e =
`0x400e`; handler at `magicyuv.dll!0x69b86a50` reachable via the
`DriverProc` dispatch at `magicyuv.dll!0x69b8e120`–`0x69b8e4d6`) sets
`*lpckid` to the constant `0x63646364` ("dcdc" in LE byte order;
the encoder writes this 32-bit immediate at
`magicyuv.dll!0x69b85762`) and `*lpdwflags` to `0x10` =
`AVIIF_KEYFRAME` (the corresponding 32-bit immediate write occurs
at `magicyuv.dll!0x69b8576c`). The literal "dcdc" is a
placeholder the codec emits; standard AVI muxers recompute the
final chunk-ID with the correct stream index (`00dc` for stream 0),
and the muxer's value is what reaches the file. Every `ICCompress`
output is flagged AVIIF_KEYFRAME because MagicYUV is a lossless
intra-only codec — every frame is independently decodable
(`05-entropy-coding.md` §6.3).

### 2.2 Chunk payload is one MAGY frame

The `ckData` of each `00dc` chunk is exactly the byte sequence
defined by chapters 01..05: the 32-byte MAGY frame header
(`01-file-header-and-fourccs.md` §3), followed by the slice-offset table (`02-slice-table.md` §5),
followed by the per-frame preamble (`02-slice-table.md` §7 + `05-entropy-coding.md` §1),
followed by the slice payloads
(`04-prediction-modes.md` and `05-entropy-coding.md` §3..§4).

Behavioural confirmation: every fixture surveyed has its
`00dc` payload begin with the bytes `4d 41 47 59` (= `'M' 'A' 'G' 'Y'`,
the `01-file-header-and-fourccs.md` §1 magic) followed by the 32-byte
v7 header. A representative trace from fixture `m8rg_64x64.avi`
(constructed from a 64×64 zero-content M8RG encode):

```
file offset  bytes                                    interpretation
0x108..0x10b 30 30 64 63                              "00dc" chunk ID
0x10c..0x10f 86 06 00 00                              chunk size = 0x686 = 1670 bytes
0x110..0x113 4d 41 47 59                              MAGY magic (`01-file-header-and-fourccs.md` §1)
0x114..0x117 20 00 00 00                              header_size = 0x20 (`01-file-header-and-fourccs.md` §2)
0x118        07                                       version = 7
0x119        65                                       format_byte = 0x65 (M8RG; `01-file-header-and-fourccs.md` §3.0)
...
```

The remaining 1670 − 32 = 1638 bytes of the chunk payload are the
slice table + preamble + slice payloads per chapters 02..05. See
[`02-slice-table.md`](02-slice-table.md) §5.2 for the worked
example using the same fixture content.

### 2.3 Word-alignment padding

Per the [Microsoft Learn — AVI RIFF File
Reference](https://learn.microsoft.com/en-us/windows/win32/directshow/avi-riff-file-reference)
("RIFF File Format"), every RIFF chunk is padded to a 2-byte boundary
in the file. The chunk's `ckSize` field gives the **payload byte
count without padding**; if that count is odd, one zero byte is
appended after the payload before the next chunk header.

Per [`02-slice-table.md`](02-slice-table.md) §8, the v2.4.2 encoder
always emits an even-byte-count MAGY frame ("Encoded frames are now
padded to be an even number of bytes" — vendor change-log entry
1.2-rev1, 2016-10-13). So the `00dc` chunk payload itself is
already even-aligned and no
RIFF-level pad byte is required. A correct reader MUST nonetheless
honour the AVI even-pad rule: a non-MagicYUV encoder writing
odd-length MAGY-format frames into AVI would still produce
spec-compliant `00dc` chunks with one trailing pad byte, and
decoders MUST skip that byte before reading the next chunk header.

## 3. The `strf` chunk: BITMAPINFOHEADER + extradata

The stream format (`strf`) chunk for a MagicYUV video stream is
**72 bytes** of payload: a 40-byte
[`BITMAPINFOHEADER`](https://learn.microsoft.com/en-us/previous-versions/dd183376(v=vs.85))
followed by a 32-byte **extradata** block. The codec produces this
exact layout in its response to `ICM_COMPRESS_GET_FORMAT`
(= `0x4004`; handler at `magicyuv.dll!0x69b84bf0` reachable via the
`DriverProc` dispatch).

### 3.1 Encoder evidence: `ICCompressGetFormat` builds the strf

The handler at `magicyuv.dll!0x69b84bf0`:

1. Calls a helper at `magicyuv.dll!0x69b97700` (the call site is
   at `magicyuv.dll!0x69b84ce6`) that allocates a 1 MB working
   buffer and writes the codec's per-format-byte 32-byte MAGY
   header pattern into its first 32 bytes; returns the value
   `0x20` to its caller (the header_size field at offset 4 of
   the MAGY header, loaded from `+0x04` of the buffer at
   `magicyuv.dll!0x69b978f5`).
2. At `magicyuv.dll!0x69b84cf3` it computes `0x20 + 0x28 = 0x48
   = 72` — the biSize value that includes 40 bytes of standard
   BITMAPINFOHEADER plus 32 bytes of extradata.
3. If the caller's `lpbiOutput` is NULL — i.e. the caller is using
   [`ICCompressGetFormatSize`](https://learn.microsoft.com/en-us/windows/win32/api/vfw/nf-vfw-iccompressgetformatsize)
   to query just the size — the function returns the constant
   `0x48` directly (the immediate is loaded into the return
   register at `magicyuv.dll!0x69b84e78`).
4. Otherwise, copies the input BITMAPINFOHEADER's `biWidth`,
   `biHeight`, `biCompression`, `biSizeImage`, `biXPelsPerMeter`,
   `biYPelsPerMeter` into the output starting at offset 8 of
   `lpbiOutput`
   (`magicyuv.dll!0x69b84d00`–`0x69b84d2c`), writes `biSize =
   0x48` at offset 0, writes `biPlanes = 1` at offset 12 (the
   constant `1` is set up at `magicyuv.dll!0x69b84d03` and stored
   into the output buffer's `+0x0c` slot), then computes
   `biCompression`, `biBitCount`, and `biSizeImage` per the
   per-format-byte property record (the family record's `+0x24`
   field for biBitCount is the
   `aux_byte / max_huffman_code_length` value documented in
   `01-file-header-and-fourccs.md` §3.0).
5. Calls the **MAGY header writer** at
   `magicyuv.dll!0x69b974f0` with the second argument set to
   `lpbiOutput + 0x28` (the argument is set up and the call is
   issued at `magicyuv.dll!0x69b84d74`). This is the same writer
   documented in `01-file-header-and-fourccs.md` §2: it emits the
   32-byte MAGY header into the pointed-to buffer, populating the
   `format_byte`, `aux_byte`, `codec_variant`, `flags`, `width`,
   `height`, `width_extra`, `slice_height` fields per the
   configured codec parameters. **The bytes it writes into the
   strf extradata are exactly the same bytes that will appear at
   the start of every `00dc` chunk.**

### 3.2 BITMAPINFOHEADER fields the codec writes

Behavioural trace: `avi_probe.exe M8RG 64 64`
(provenance §B). The 72-byte strf payload returned is:

```
0x00..0x03   48 00 00 00          biSize = 72
0x04..0x07   40 00 00 00          biWidth = 64 (LONG, signed)
0x08..0x0b   40 00 00 00          biHeight = 64 (LONG, positive — see §3.4)
0x0c..0x0d   01 00                biPlanes = 1 (WORD)
0x0e..0x0f   18 00                biBitCount = 24 (WORD; for M8RG)
0x10..0x13   4d 38 52 47          biCompression = FOURCC 'M8RG' = 0x4752384d
0x14..0x17   04 33 04 00          biSizeImage = 275204 (codec's max-output-size estimate)
0x18..0x1b   00 00 00 00          biXPelsPerMeter = 0
0x1c..0x1f   00 00 00 00          biYPelsPerMeter = 0
0x20..0x23   00 00 00 00          biClrUsed = 0
0x24..0x27   00 00 00 00          biClrImportant = 0
0x28..0x47   <32 B extradata = MAGY header; see §4>
```

Field-by-field semantics (per [Microsoft Learn —
BITMAPINFOHEADER](https://learn.microsoft.com/en-us/previous-versions/dd183376(v=vs.85))):

| Field             | Codec value                          | Decoder validation rule (§3.5)        |
| ----------------- | ------------------------------------ | ------------------------------------- |
| `biSize`          | `72` (= 0x48) — fixed for v2.4.2     | MUST be ≥ 40 (sizeof BITMAPINFOHEADER); MUST be ≥ 72 to carry the 32-B extradata, but see §3.3 |
| `biWidth`         | from caller's input format           | passed to MAGY header `width` (`01-file-header-and-fourccs.md` §3.2) |
| `biHeight`        | from caller's input format (positive) | passed to MAGY header `height`        |
| `biPlanes`        | `1` (always)                         | per [BITMAPINFOHEADER docs] "must be set to 1" |
| `biBitCount`      | per format byte (see §3.6)           | informational; the codec recomputes from extradata |
| `biCompression`   | the native FOURCC (`M8RG`, `M0Y0`, …) | MUST equal one of the 17 from `01-file-header-and-fourccs.md` §4.1 |
| `biSizeImage`     | codec's worst-case compressed-output size estimate (~270 KB for 64×64) | informational; not equal to the actual `00dc` chunk size |
| `biXPelsPerMeter` | `0`                                  | ignored                              |
| `biYPelsPerMeter` | `0`                                  | ignored                              |
| `biClrUsed`       | `0`                                  | ignored (not a paletted format)      |
| `biClrImportant`  | `0`                                  | ignored                              |

**Critical fields** for a decoder are `biCompression` (the FOURCC
selects the codec), `biWidth`, and `biHeight`. These three carry the
information necessary to set up the decoder. The MAGY-header
extradata (§4) carries the same width/height plus the
`format_byte`, `flags`, and `aux_byte`/`codec_variant` discriminators
that the in-band `00dc`-chunk header will repeat verbatim per frame.

### 3.3 Why the extradata exists

The codec's `ICDecompressBegin` handler (`ICM_DECOMPRESS_BEGIN`,
= `0x400d`; handler at `magicyuv.dll!0x69b86290`) reads the
**input** BITMAPINFOHEADER (the strf bytes, accessible via
`AVIStreamReadFormat` per the [AVI RIFF File
Reference](https://learn.microsoft.com/en-us/windows/win32/directshow/avi-riff-file-reference))
and validates the extradata before accepting the stream.

Specifically, at
`magicyuv.dll!0x69b8636c`–`0x69b8638d` the handler:

1. Loads `bi_in.biSize` from offset `0` of the input
   BITMAPINFOHEADER pointer.
2. Computes the extradata pointer as `bi_in + 40` (i.e. just
   past the 40-byte standard BITMAPINFOHEADER).
3. Computes the extradata length as `biSize - 40`.
4. Calls the decoder header parser at `magicyuv.dll!0x69badfb0`
   with the extradata pointer and length as arguments.

The parser at `magicyuv.dll!0x69badfb0` is the same one `01-file-header-and-fourccs.md` §1
documented for in-frame validation:

- At `magicyuv.dll!0x69badfcc`–`0x69badfd0` it rejects when the
  extradata length is `<= 31`.
- At `magicyuv.dll!0x69badfd2`–`0x69badfd8` it compares the
  first 4 bytes of the extradata against the magic constant
  `0x5947414d` (`'MAGY'`) and accepts on match.
- At `magicyuv.dll!0x69badff0`–`0x69badff4` it requires the
  declared header_size at offset `+0x04` to equal the input
  length.
- At `magicyuv.dll!0x69badff6`–`0x69badffa` it rejects when the
  version byte at offset `+0x08` exceeds `0x7`.

So the decoder requires the extradata to be a syntactically valid
v7 MAGY header: starts with `MAGY` magic, has its own
`header_size` (offset 4) equal to its declared length (= 32 in
v2.4.2), and has version ≤ 7. Streams where `biSize == 40` (no
extradata) reach the parser with the length argument set to 0
and are rejected by the `length <= 31` guard at
`magicyuv.dll!0x69badfcc`.

In practice the strf extradata and the per-frame `00dc` MAGY header
are identical in v2.4.2: both carry the same configured
`(format_byte, aux_byte, codec_variant, flags, width, height,
width_extra, slice_height)` tuple, written by the same encoder-side
function (`magicyuv.dll!0x69b974f0`).

### 3.4 Top-down vs bottom-up: `biHeight` is positive

Per the [BITMAPINFOHEADER MSDN
reference](https://learn.microsoft.com/en-us/previous-versions/dd183376(v=vs.85))
("biHeight" member): a positive `biHeight` declares a bottom-up
DIB; a negative `biHeight` declares a top-down DIB. The same
reference notes:

> If **biHeight** is negative, indicating a top-down DIB,
> **biCompression** must be either BI_RGB or BI_BITFIELDS. Top-down
> DIBs cannot be compressed.

Behavioural trace: in every fixture surveyed, the
codec's `ICCompressGetFormat` writes `biHeight` **positive**:

```
fixture                  biHeight value
M8RG 64×64               64
M8RA 64×64               64
M8G0 64×64               64
M8Y4 128×96              96
```

(Confirmed via `avi_probe.exe`; see provenance §C.)

This is consistent with the AVI/DIB rule: compressed video streams
declare positive biHeight. The actual storage **inside** the
compressed payload uses top-down per-plane row order (`03-pixel-plane-mapping.md` §1
plus `04-prediction-modes.md` §4 — predictors run row 0 → row N-1 within each
slice, so plane row 0 is the **top** of the plane). The
biHeight = positive convention does NOT mean MagicYUV-decoded
pixels are bottom-up; it is a property of the AVI/DIB declaration
that doesn't transfer to the codec's internal pixel orientation.

For decoders writing reconstructed frames out into a top-down
display surface or a top-down container, no row-flip is needed:
the codec's natural output is top-down. For decoders feeding into
a legacy bottom-up DIB consumer (e.g. `BitBlt` on a raw RGB DIB
with positive biHeight), a vertical flip of each plane is
required at the application layer.

### 3.5 `biCompression` MUST be a native MagicYUV FOURCC

The 17 native FOURCCs from `01-file-header-and-fourccs.md` §4.1 (`M8RG`, `M8RA`, `M8Y4`,
`M8Y2`, `M8Y0`, `M8YA`, `M8G0`, `M0Y2`, `M0RG`, `M0RA`, `M2RG`,
`M2RA`, `M4RG`, `M4RA`, `M0G0`, `M0Y4`, `M0Y0`) are the legal
biCompression values. Each FOURCC selects:

- The pixel family and bit depth (`01-file-header-and-fourccs.md` §4.1; `03-pixel-plane-mapping.md` §3).
- The internal Backend used by the codec to lay out planes
  (`03-pixel-plane-mapping.md` §1 — 14 distinct Backends, with M8RG/M8Y4 sharing
  `Backend<u8, 8, 3, 1, 1>` etc.).

A decoder's responsibility on receiving a strf with one of these
FOURCCs is to:

1. Verify `biSize >= 72` (the extradata is mandatory in v2.4.2;
   §3.3 above).
2. Verify the extradata starts with the `MAGY` magic and is a
   well-formed 32-byte v7 header (`01-file-header-and-fourccs.md` §3).
3. Optionally cross-check that the extradata's `format_byte`
   maps to the same FOURCC as `biCompression` per `01-file-header-and-fourccs.md` §4.1.
   In v2.4.2 this is always the case (the encoder writes both
   from the same property record), but a hostile / malformed
   stream could mismatch them; the decoder's defence is to use
   the extradata's `format_byte` (and the per-frame MAGY-header
   `format_byte`) as the source of truth, since downstream
   plane/Backend dispatch (`03-pixel-plane-mapping.md`) is keyed on `format_byte`,
   not on the FOURCC.
4. Use `biWidth` and `biHeight` (positive) as the frame
   dimensions for buffer-allocation purposes; cross-check
   against the extradata's `width` / `height` fields (header
   `+0x10` / `+0x14` per `01-file-header-and-fourccs.md` §3.2).

### 3.6 `biBitCount` per FOURCC

The codec's response to `ICCompressGetFormat` writes `biBitCount`
as a per-format-byte value pulled from the property record's
`+0x24` field. Behavioural trace gives:

| FOURCC | biBitCount (observed) | Comment                          |
| ------ | --------------------: | -------------------------------- |
| M8RG   | 24                    | RGB 8-bit, 3 planes × 8 bits each |
| M8RA   | 32                    | RGBA 8-bit, 4 × 8                |
| M8Y4   | 24                    | YUV 4:4:4 8-bit, 3 × 8 packed |
| M8G0   | 24                    | Gray 8-bit (declared as 24 — informational only; per `03-pixel-plane-mapping.md` §3 only one plane is emitted) |

This matches the `bpp` column from `01-file-header-and-fourccs.md` §4.1 §4.1, except for
M8G0 (4:0:0 single-plane gray) where the published `bpp = 24` is
not directly meaningful. `biBitCount` is **informational** for
MagicYUV — the actual per-plane sample size is determined by the
FOURCC + format_byte, not by `biBitCount`.

For 10/12/14-bit FOURCCs, the `01-file-header-and-fourccs.md` §4.1 `bpp` column gives the
codec-declared values: M0Y2 = 20, M0Y4 = 30, M0Y0 = 15, M0RG = 30,
M2RG = 36, M4RG = 42, M0RA = 40, M2RA = 48, M4RA = 56, M0G0 = 10.
These are also bits-per-pixel sums across all (sub-sampled) planes,
not container bit counts. A decoder MUST NOT rely on `biBitCount`
to determine sample storage; that comes from the format_byte
→ Backend mapping in `03-pixel-plane-mapping.md` §3.

## 4. Extradata = the 32-byte MAGY frame header

The 32 bytes that follow the BITMAPINFOHEADER inside the `strf`
chunk are the **same** structure as the in-frame MAGY header
defined by [`01-file-header-and-fourccs.md`](01-file-header-and-fourccs.md) §3.

Layout (per `01-file-header-and-fourccs.md` §3, repeated here for convenience):

```
+--------+-------------------------------------------------------+
| Offset | Field                                                 |
+========+=======================================================+
|  0x00  | magic         (4 B, fixed = 'MAGY' = 0x5947414d LE)   |
|  0x04  | header_size   (uint32 LE, fixed = 0x00000020 in v7)   |
|  0x08  | version       (uint8,    fixed = 0x07 in v7)          |
|  0x09  | format_byte   (uint8;    `01-file-header-and-fourccs.md` §3.0)                |
|  0x0a  | aux_byte      (uint8;    `01-file-header-and-fourccs.md` §3.0)                |
|  0x0b  | codec_variant (uint8;    `01-file-header-and-fourccs.md` §3.0)                |
|  0x0c  | flags         (uint32 LE; `01-file-header-and-fourccs.md` §3.1)               |
|  0x10  | width         (uint32 LE)                             |
|  0x14  | height        (uint32 LE)                             |
|  0x18  | width_extra   (uint32 LE; usually = width)            |
|  0x1c  | slice_height  (uint32 LE; `02-slice-table.md` §3 — 28 in v2.4.2)  |
+--------+-------------------------------------------------------+
```

(The `0x1c` field is `slice_height`, per `02-slice-table.md` §3.)

### 4.1 Behavioural confirmation: extradata equals frame header

For fixture `m8rg_64x64.avi` (avi_probe.exe; provenance §C), the
strf extradata bytes (offsets 0x28..0x47 of the strf payload) are:

```
4d 41 47 59 20 00 00 00 07 65 0c 02 00 00 20 00
40 00 00 00 40 00 00 00 40 00 00 00 1c 00 00 00
```

The first `00dc` chunk's payload begins (file offsets 0x110..0x12f
in the same AVI):

```
4d 41 47 59 20 00 00 00 07 65 0c 02 00 00 20 00
40 00 00 00 40 00 00 00 40 00 00 00 1c 00 00 00
```

**Byte-identical**. The codec's encoder-side helper at
`magicyuv.dll!0x69b974f0` emits both, with the same configured
parameters. The same equality holds for every (FOURCC, dimension)
combination tested:

| FOURCC | format_byte (header +0x09) | aux_byte (+0x0a) | codec_variant (+0x0b) |
| ------ | --------------------------:|------------------:|---------------------:|
| M8RG   | 0x65                       | 0x0c              | 0x02                 |
| M8RA   | 0x66                       | 0x0c              | 0x02                 |
| M8Y4 (128×96) | 0x67                | 0x0c              | 0x02                 |
| M8G0   | 0x6b                       | 0x0c              | 0x02                 |

`format_byte` matches the `01-file-header-and-fourccs.md` §4.1
mapping for the FOURCC. `aux_byte = 0x0c = 12` matches the
`HuffCoder<u8, u16, 8, 12, 12>` specialisation's
max_huffman_code_length parameter (per `05-entropy-coding.md` §1.5).
`codec_variant = 0x02` (Gradient) is the per `04-prediction-modes.md`
§2 always-emitted-by-v2.4.2 value.

### 4.2 What's redundant between strf and per-frame header

For a v2.4.2 stream, the strf extradata and every `00dc` chunk's
leading 32 bytes carry the **same configured constants**:
`format_byte`, `aux_byte`, `codec_variant`, `flags`, `width`,
`height`, `width_extra`, `slice_height`. The fields that are
strictly per-frame (slice table, preamble, slice payloads) start
at file offset 32 of each `00dc` payload (`02-slice-table.md` §5).

Decoders MAY use the strf extradata to set up per-stream state
(the Huffman max-length getter, the Backend dispatch, the
plane-order convention) and then validate each per-frame
`00dc` chunk's leading 32 bytes against the stream's extradata.
v2.4.2 streams will produce byte-identical matches; a future
codec revision could in principle vary those constants per
frame, but the v7 wire format does not constrain this either way.

The strictly necessary information from the extradata for a
decoder is:

1. The MAGY magic + version, to confirm wire-format compatibility.
2. The `format_byte`, to dispatch to the right Backend
   (`03-pixel-plane-mapping.md` §3) before the first `00dc` arrives. Without
   extradata, a decoder would have to defer Backend selection
   until it sees the first per-frame header — which works for
   plain in-band parsing but breaks streaming demuxers that need
   to allocate buffers up front.
3. The `slice_height`, to know how many slices to expect per
   plane (`02-slice-table.md` §4). Without extradata, slice-table walking
   still works (it's self-contained per `02-slice-table.md` §5), so this is
   secondary.

## 5. The `strh` chunk (AVISTREAMHEADER)

Per [Microsoft Learn —
AVISTREAMHEADER](https://learn.microsoft.com/en-us/previous-versions/windows/desktop/api/avifmt/ns-avifmt-avistreamheader),
the `strh` chunk's payload is a 64-byte AVISTREAMHEADER (12 fixed
fields totalling 48 bytes followed by a `RECT rcFrame` of 4 LONGs =
16 bytes). The codec does not write this chunk directly (it's the
muxer's responsibility), but a decoder must know the field
expectations:

| Field                | Required value for MagicYUV         | Source                              |
| -------------------- | ----------------------------------- | ----------------------------------- |
| `fccType`            | `'vids'` (= 0x73646976)             | per [AVISTREAMHEADER docs] (video stream marker) |
| `fccHandler`         | One of the 17 native FOURCCs        | `01-file-header-and-fourccs.md` §4.1 — `'M8RG'`, `'M0Y0'`, etc. |
| `dwFlags`            | typically `0`                       | AVISF_VIDEO_PALCHANGES is not set (no palette) |
| `wPriority`          | typically `0`                       | informational                      |
| `wLanguage`          | typically `0`                       | informational                      |
| `dwInitialFrames`    | `0` for non-interleaved             | matches dwInitialFrames in avih    |
| `dwScale` / `dwRate` | `dwRate / dwScale` = frame rate     | application-defined; common = 30/1 |
| `dwStart`            | typically `0`                       | per-stream start offset            |
| `dwLength`           | total frame count                   | matches avih's dwTotalFrames       |
| `dwSuggestedBufferSize` | ≥ max compressed-frame size      | matches avih's dwSuggestedBufferSize |
| `dwQuality`          | `-1` (default) or 0..10000          | informational                      |
| `dwSampleSize`       | `0`                                  | "each video frame in a separate chunk" per AVISTREAMHEADER docs |
| `rcFrame`            | `(0, 0, biWidth, biHeight)`         | display rectangle                  |

`fccHandler` MUST equal the FOURCC declared in the `strf`'s
`biCompression` field. A decoder MAY use either value
interchangeably; the codec dispatcher (`AVIFile`) keys plugin
selection off `fccHandler`.

## 6. The `avih` chunk (AVIMAINHEADER) and OpenDML 2.0

Per [Microsoft Learn —
AVIMAINHEADER](https://learn.microsoft.com/en-us/previous-versions/windows/desktop/api/aviriff/ns-aviriff-avimainheader),
the `avih` chunk's payload is a 56-byte AVIMAINHEADER (the AVI
1.0 main header structure). MagicYUV-relevant field values:

| Field                | Expected value                       |
| -------------------- | ------------------------------------ |
| `dwMicroSecPerFrame` | `1000000 / fps`                      |
| `dwMaxBytesPerSec`   | bandwidth budget (informational)     |
| `dwPaddingGranularity` | typically `0`                      |
| `dwFlags`            | `AVIF_HASINDEX` (0x10) when idx1 present; `AVIF_MUSTUSEINDEX` not set; `AVIF_ISINTERLEAVED` only for multi-stream |
| `dwTotalFrames`      | frame count                          |
| `dwInitialFrames`    | `0` for video-only or non-interleaved |
| `dwStreams`          | `1` for video-only                   |
| `dwSuggestedBufferSize` | ≥ max frame size                  |
| `dwWidth`, `dwHeight`| match `biWidth` and positive `biHeight` |
| `dwReserved[4]`      | `0`                                  |

### 6.1 OpenDML 2.0 super-index

For AVI files larger than 1 GB, AVI 1.0's `idx1` chunk (which
uses 32-bit offsets) becomes inadequate, and the
[OpenDML 2.0 AVI File Format
Extensions](https://web.archive.org/web/20070228232302/http://www.the-labs.com/Video/odmlff2-avidef.pdf)
define a hierarchical super-index scheme (`indx` + `ix00` chunks)
that supports >2 GB streams and ≥4 GB files.

The vendor change-log entry for v2.4.2
(<https://www.magicyuv.com/change-log/>) states:

> Fixed reading very large (multiple TB) AVI files from OBS with
> index corruption.

This implies the codec — or, more precisely, the application that
hosts the codec via VfW (which delegates demuxing to
`avifil32.dll` or similar) — supports OpenDML 2.0 indexing. The
codec itself does not parse RIFF chunks: it only accepts a
single per-frame buffer via `ICDecompress`. So OpenDML compliance
is **muxer/demuxer side**: the host application (OBS, VirtualDub,
AviSynth, ffmpeg's AVI reader) builds OpenDML structures, and the
codec consumes the sequence of MAGY-prefixed packets one at a
time.

The wire-format-relevant takeaway: an OpenDML AVI file containing
MagicYUV is identical at the per-frame chunk level (`00dc` chunks
in `movi` LIST(s); MAGY-prefixed payload) to a vanilla AVI 1.0
file. The differences are confined to how the file's overall
RIFF structure is laid out (multiple `RIFF AVIX` continuations
plus an `indx` super-index chunk), which is outside the
MagicYUV-specific spec scope.

A decoder reading MagicYUV from an AVI file should therefore use
a standard OpenDML-aware demuxer (the AVIFile API does this
transparently) to walk the `00dc` chunks; once it has each
chunk's payload, the decoding rules of chapters 01..05 apply
unchanged.

## 7. The `idx1` chunk (AVI 1.0 index)

Per the [AVI RIFF File
Reference](https://learn.microsoft.com/en-us/windows/win32/directshow/avi-riff-file-reference)
("AVI Index Entries — AVI 1.0 index"), an `idx1` chunk contains
an [`AVIOLDINDEX`](https://learn.microsoft.com/en-us/previous-versions/windows/desktop/api/aviriff/ns-aviriff-avioldindex)
structure: a sequence of 16-byte entries, one per data chunk in
the `movi` list, each consisting of:

```
DWORD dwChunkId;     // e.g. '00dc'
DWORD dwFlags;       // AVIIF_KEYFRAME (0x10) for MagicYUV
DWORD dwOffset;      // offset of the chunk header
DWORD dwSize;        // chunk payload size (= the `00dc` ckSize)
```

Each MagicYUV `00dc` chunk produces one such entry. `dwFlags`
SHOULD be `AVIIF_KEYFRAME` (= 0x10) for every entry, since
MagicYUV is intra-only (every frame is independently decodable;
`05-entropy-coding.md` §6.3) — the codec itself flags every `ICCompress`
output with `AVIIF_KEYFRAME` (the encoder writes the 32-bit
immediate `0x10` to the flags field at
`magicyuv.dll!0x69b8576c`).

`dwOffset` is conventionally measured from the start of the
`movi` LIST chunk (i.e. the file offset of the bytes
`'L' 'I' 'S' 'T'` of the LIST header) plus 8 (skipping the
LIST header itself). Some applications instead measure from the
start of the file or from the start of the `movi` chunk's
payload (i.e. after the `'movi'` four-cc); the AVI 1.0
specification is famously unclear on this point and the AVIFile
API tolerates both conventions. Decoders that walk `idx1` should
attempt the standard convention (offset relative to start of
`movi` LIST + 8) first and fall back if the resulting position
does not contain a valid chunk header.

For AVI files that exceed the 4 GB representable in a 32-bit
DWORD, the AVI 1.0 `idx1` is replaced or supplemented by OpenDML
2.0 indexing per §6.1.

## 8. End-to-end worked example: `m8rg_64x64.avi`

Captured by the harness `avi_probe.exe M8RG 64 64
m8rg_64x64.avi` (provenance §C); 1966 bytes total. Annotated hex:

```
0x0000  52 49 46 46 a6 07 00 00 41 56 49 20             RIFF<size>"AVI "
0x000c  4c 49 53 54 e8 00 00 00 68 64 72 6c             LIST<size>"hdrl"
0x0018  61 76 69 68 38 00 00 00                         "avih"<56=0x38>
        (56 B AVIMAINHEADER: dwMicroSecPerFrame=33333,
         dwFlags=0x10 (AVIF_HASINDEX), dwTotalFrames=1,
         dwStreams=1, dwSuggestedBufferSize=0x686=1670,
         dwWidth=64, dwHeight=64)
0x0058  4c 49 53 54 9c 00 00 00 73 74 72 6c             LIST<size>"strl"
0x0064  73 74 72 68 40 00 00 00                         "strh"<64=0x40>
        (64 B AVISTREAMHEADER: fccType='vids',
         fccHandler='M8RG', dwScale=1, dwRate=30,
         dwLength=1, dwSuggestedBufferSize=1670,
         dwQuality=0xffffffff (-1), dwSampleSize=0,
         rcFrame=(0,0,64,64))
0x00ac  73 74 72 66 48 00 00 00                         "strf"<72=0x48>
        (72 B BITMAPINFOHEADER + extradata; see §3.2)
0x00b4  48 00 00 00 40 00 00 00 40 00 00 00             biSize=72 biWidth=64 biHeight=64
0x00c0  01 00 18 00 4d 38 52 47                         biPlanes=1 biBitCount=24 biCompression='M8RG'
0x00c8  86 06 00 00 00 00 00 00                         biSizeImage=0x686 biXPelsPerMeter=0
0x00d0  00 00 00 00 00 00 00 00 00 00 00 00             biYPelsPerMeter=0 biClrUsed=0 biClrImportant=0
0x00dc  4d 41 47 59 20 00 00 00 07 65 0c 02 00 00 20 00 (extradata: MAGY ver=7 fmt=0x65 aux=0x0c
0x00ec  40 00 00 00 40 00 00 00 40 00 00 00 1c 00 00 00  cv=0x02 flags=0x200000 w=64 h=64 we=64 sh=28)
0x00fc  4c 49 53 54 92 06 00 00 6d 6f 76 69             LIST<size>"movi"
0x0108  30 30 64 63 86 06 00 00                         "00dc"<size=1670>
0x0110  4d 41 47 59 20 00 00 00 07 65 0c 02 00 00 20 00 (per-frame MAGY header — IDENTICAL
0x0120  40 00 00 00 40 00 00 00 40 00 00 00 1c 00 00 00  to extradata at 0x00dc..0x00fb)
0x0130  ...                                              (slice table + preamble + payload per
                                                          `02-slice-table.md` §5.2)
0x0796  69 64 78 31 10 00 00 00                         "idx1"<16>
0x079e  30 30 64 63 10 00 00 00 fc 00 00 00 86 06 00 00 (one entry: '00dc' AVIIF_KEYFRAME
                                                          dwOffset=0xfc dwSize=1670)
```

The byte-identity between the strf extradata at file offset
0x00dc..0x00fb and the per-frame `00dc` MAGY header at
0x0110..0x012f is the central observation of this chapter (§4.1).

## 9. Cross-reference for implementers

| Topic                                | Primary source        | Cross-ref                            |
| ------------------------------------ | --------------------- | ------------------------------------ |
| RIFF / AVI overall file structure    | AVI RIFF File Ref [^1]| §1                                   |
| `00dc` chunk per frame               | AVI RIFF File Ref [^1]| §2                                   |
| `00dc` payload = MAGY frame          | chapters 01..05 + this chapter §2.2 | `01-file-header-and-fourccs.md` §3 for MAGY header  |
| BITMAPINFOHEADER fields              | MSDN BITMAPINFOHEADER [^2] | §3                              |
| biCompression FOURCC list            | `01-file-header-and-fourccs.md` §4.1          | §3.5                                 |
| 32-byte extradata = MAGY header      | this chapter §4 (codec-side static + behavioural) | `01-file-header-and-fourccs.md` §3 |
| AVISTREAMHEADER fields               | MSDN AVISTREAMHEADER [^3]  | §5                              |
| AVIMAINHEADER fields                 | MSDN AVIMAINHEADER [^4]    | §6                              |
| OpenDML 2.0 super-index              | OpenDML 2.0 spec [^5] | §6.1                                 |
| `idx1` chunk (AVI 1.0 index)         | AVIOLDINDEX MSDN      | §7                                   |
| Top-down vs bottom-up                | MSDN BITMAPINFOHEADER `biHeight` member [^2] | §3.4                |

[^1]: <https://learn.microsoft.com/en-us/windows/win32/directshow/avi-riff-file-reference>
[^2]: <https://learn.microsoft.com/en-us/previous-versions/dd183376(v=vs.85)>
[^3]: <https://learn.microsoft.com/en-us/previous-versions/windows/desktop/api/avifmt/ns-avifmt-avistreamheader>
[^4]: <https://learn.microsoft.com/en-us/previous-versions/windows/desktop/api/aviriff/ns-aviriff-avimainheader>
[^5]: OpenDML AVI File Format Extensions, OpenDML AVI M-JPEG File Format Subcommittee, Sept 1996. Archive: <https://web.archive.org/web/20070228232302/http://www.the-labs.com/Video/odmlff2-avidef.pdf>

## 10. Open questions

1. **`biSize == 40` (no extradata) acceptance.** The decoder's
   strf parser at `magicyuv.dll!0x69b86290`–`0x69b86388` rejects
   when extradata length ≤ 31 (the length-vs-`0x1f` guard at
   `magicyuv.dll!0x69badfcc`). A strf chunk with `biSize == 40`
   (the bare minimum BITMAPINFOHEADER, no extradata) reaches
   the parser with the length argument set to 0 and is rejected.
   So **streams produced without extradata are not decodable by
   v2.4.2**, even though the FOURCC alone uniquely identifies
   the codec. Whether earlier MagicYUV versions (v1, v0..v6)
   accepted bare `biSize == 40` is unverified.

2. **`aux_byte = max_huffman_code_length` for non-8-bit FOURCCs.**
   8-bit FOURCCs (M8RG, M8RA, M8Y4, M8G0) confirm
   `aux_byte = 0x0c = 12`, matching the
   `HuffCoder<u8, u16, 8, 12, 12>` specialisation's
   max_huffman_code_length parameter. 10/12/14-bit FOURCCs (which
   require 10-bit RGB or YUV input the harness here did not
   provide) are unverified: `aux_byte` should be 14, 16,
   18 for M0RG/M0Y4/M0G0/M0Y2/M0Y0/M0RA, M2RG/M2RA, M4RG/M4RA
   respectively if the same per-bit-depth rule holds.

3. **Per-frame MAGY header equality with strf extradata.** Every
   tested fixture has byte-identity between the strf extradata
   and the leading 32 bytes of the first `00dc` chunk. The wire
   format does not formally require this; an encoder could vary
   the per-frame header (e.g. change codec_variant per frame).
   v2.4.2 doesn't, but future-format streams might. Decoders
   SHOULD validate the per-frame header on its own merits
   (`01-file-header-and-fourccs.md` §1) and not assume
   strf-extradata equality.

4. **`idx1` `dwOffset` convention.** §7 notes the historical
   ambiguity (offset from RIFF start vs. offset from movi start
   vs. offset from movi+8). v2.4.2 does not write AVI files
   itself, so the question is moot for codec-output verification;
   the AVIFile API the harness used picks the right convention
   for whichever container format it sees. Implementers building
   a from-scratch AVI parser must pick a convention and may need
   to fall back if the file uses the other.

5. **OpenDML 2.0 `indx` chunk parsing.** §6.1 notes that
   OpenDML support is muxer-side; the codec does not parse
   `indx`/`ix00`/`AVIX` chunks. A from-scratch implementer-side
   demuxer that reads AVI files (rather than relying on a
   pre-existing AVI-aware demuxer like `riff::Reader`) needs
   OpenDML 2.0 support to handle the >2 GB streams the v2.4.2
   change log calls out. This is demuxer territory, not
   codec territory; flagged here for cross-reference.

6. **Behavioural exercise of 10-bit / YUV / interlaced fixtures.**
   The harness used here only exercised 8-bit RGB-input AVI files
   (M8RG, M8RA, M8G0, M8Y4 with non-square dimensions). The
   strf format-size = 72 invariant, the extradata-equals-frame-
   header invariant, and biHeight-positive convention have been
   confirmed across these. Extending the harness to 10-bit
   inputs (which require non-trivial frontend conversion or
   YUV input pipelines) would strengthen the cross-FOURCC
   confidence.
