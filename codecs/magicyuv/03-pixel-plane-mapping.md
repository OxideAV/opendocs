# MagicYUV v7 — Pixel plane mapping

This chapter pins down, for every v7 `format_byte` enumerated in
[`01-file-header-and-fourccs.md`](01-file-header-and-fourccs.md) §4.1,
the **wire-format storage layout**: how many planes are emitted, in
what order, what sample type each plane uses, what the per-plane width
and height are relative to the frame's `(width, height)`, and how
high-bit-depth samples (10/12/14-bit) sit in their 2-byte container.

All claims trace to byte-level evidence in the proprietary binary at
`reference/binaries/system32/magicyuv.dll` (SHA-256
`2036e284cc07a2cb4d4b6b6cb04a1d24c0f97a0041c134fd669ee079b768902c`,
PE32+ x86-64). Cross-architecture confirmations against the 32-bit
sibling (`reference/binaries/syswow64/magicyuv.dll`, SHA-256
`9a6586674efc082cb94a1394382720c425c41d835790d1d252d38f8c35fe1545`)
are noted where relevant.

Notation follows
[`01-file-header-and-fourccs.md`](01-file-header-and-fourccs.md): `<file>@0xNNNNNN`
is a file offset and `<file>!0xNNNNNN` is the instruction VMA at the
DLL's default ImageBase (`0x69b80000` for system32). The 64-bit DLL's
`.rdata` lies at file `0x248a00`–`0x269380` (RVA `0x4b000`–`0x6b980`,
VMA `0x69dcb000`–`0x69deb980`); the file→VMA conversion for `.rdata`
is `VMA = 0x69dcb000 + (file_offset − 0x248a00)`.

This chapter takes as given:

- The 32-byte v7 frame header layout from
  [`01-file-header-and-fourccs.md`](01-file-header-and-fourccs.md) §3,
  in particular `format_byte` at `+0x09`, `width` at `+0x10`, and
  `height` at `+0x14`.
- The 17-row native FOURCC table from
  [`01-file-header-and-fourccs.md`](01-file-header-and-fourccs.md) §4.1,
  in particular the `(format_byte, FOURCC, bpp, alpha-flag,
  family-ID)` 5-tuple per native row.

## 1. The Backend roster

The codec distinguishes 14 native pixel layouts. Each layout is
implemented as a C++ class-template instantiation parameterised on
`<sample_type, sample_bits, n_planes, sub_x, sub_y>`, and each
instantiation's Itanium-ABI RTTI typeinfo string appears once in
`.rdata`. The set of typeinfo records is therefore a direct, factual
inventory of the layouts the codec supports.

We refer to each instantiation by our project's own shorthand
`Backend<sample_type, sample_bits, n_planes, sub_x, sub_y>` for the
rest of this chapter. **The label is ours**, modelled on the
parameter shape we observed but not a quotation of the vendor's
class name; the parameter shape itself is functional and is justified
in §2.

Fourteen typeinfo records appear at file offsets `0x256a00` through
`0x256d40`, in 0x40-byte slots, one per layout:

| File offset | Layout (project shorthand)             |
| ----------- | -------------------------------------- |
| `0x256a00`  | `Backend<u8, 8, 1, 1, 1>`              |
| `0x256a40`  | `Backend<u8, 8, 3, 1, 1>`              |
| `0x256a80`  | `Backend<u8, 8, 3, 2, 1>`              |
| `0x256ac0`  | `Backend<u8, 8, 3, 2, 2>`              |
| `0x256b00`  | `Backend<u8, 8, 4, 1, 1>`              |
| `0x256b40`  | `Backend<i16, 10, 1, 1, 1>`            |
| `0x256b80`  | `Backend<i16, 10, 3, 1, 1>`            |
| `0x256bc0`  | `Backend<i16, 10, 3, 2, 1>`            |
| `0x256c00`  | `Backend<i16, 10, 3, 2, 2>`            |
| `0x256c40`  | `Backend<i16, 10, 4, 1, 1>`            |
| `0x256c80`  | `Backend<i16, 12, 3, 1, 1>`            |
| `0x256cc0`  | `Backend<i16, 12, 4, 1, 1>`            |
| `0x256d00`  | `Backend<i16, 14, 3, 1, 1>`            |
| `0x256d40`  | `Backend<i16, 14, 4, 1, 1>`            |

The sample-type and sample-bits values are recovered from the
Itanium-ABI mangling embedded in each typeinfo string (see
<https://itanium-cxx-abi.github.io/cxx-abi/abi.html#mangling>):

- `I…E` brackets a template-argument list.
- `h` = `unsigned char`, `s` = `short` (signed 16-bit).
- `Li<n>E` is a literal `int` template parameter with value `<n>`.

Each typeinfo string is referenced exactly once in `.rdata` (verified
by full-binary scan).
The reference is the standard `__type_info::name` field of the class
typeinfo record (the typeinfo records themselves cluster at VMA
`0x69dd5730`–`0x69dd57a0`). Each typeinfo record is in turn referenced
exactly once by a virtual-table at VMA `0x69ddf118`–`0x69ddf798`. So
the 14 layouts correspond to 14 distinct vtables in the DLL — one per
concrete instantiation.

The 32-bit DLL contains the same 14 typeinfo records at file offsets
`0x24df60`–`0x24e2a0` (0x40-byte slots, identical demangled
parameters). The wire format is architecture-independent; this matches
[`01-file-header-and-fourccs.md`](01-file-header-and-fourccs.md) §5.

A 15th class for 12-bit Bayer CFA input exists in `.rdata` at file
`0x25a100`, paired with three input-side frontend instantiations at
file `0x25a620` / `0x25a660` / `0x25a6a0` that handle the
12-bit-packed Bayer mosaic. No FOURCC or `format_byte` in
[`01-file-header-and-fourccs.md`](01-file-header-and-fourccs.md) §4.1
reaches this layout; treat it as an internal-only path. Listed as
open question 3 in §10 below.

## 2. The Backend template parameters

Template parameter semantics, derived from cross-referencing parameter
positions against bit-depth claims in the v2.4.2 vendor change log
([the vendor change log](https://www.magicyuv.com/change-log/))
and the FOURCC enumeration in
[`01-file-header-and-fourccs.md`](01-file-header-and-fourccs.md) §4.1:

```
Backend<sample_type, sample_bits, n_planes, sub_x, sub_y>
            ↑           ↑            ↑       ↑       ↑
          T1 (h/s)    T2 (8/10/12/14) T3   T4    T5
```

| Parameter      | Range observed       | Wire-format meaning                                                        |
| -------------- | -------------------- | -------------------------------------------------------------------------- |
| `sample_type`  | `h` (u8), `s` (i16)  | C++ container element type for a single sample                             |
| `sample_bits`  | 8, 10, 12, 14        | Significant bits of one sample (≤ container bit-width)                     |
| `n_planes`     | 1, 3, 4              | Number of planar components written to the wire                            |
| `sub_x`        | 1, 2                 | Horizontal subsampling divisor for non-luma planes                         |
| `sub_y`        | 1, 2                 | Vertical subsampling divisor for non-luma planes                           |

**Pinning each parameter to evidence:**

1. `sample_type` ↔ `sample_bits` covariation. Every Backend in §1 with
   `sample_bits ≤ 8` uses `h` (u8) and every Backend with `sample_bits
   ≥ 10` uses `s` (i16). Never observed: `h` with `sample_bits > 8`,
   or `s` with `sample_bits = 8`. So `sample_type` is determined by
   `sample_bits`: ≤ 8 ⇒ 1-byte container, ≥ 10 ⇒ 2-byte container.
   No 16-bit `sample_bits` Backend exists; the maximum on the wire is
   14-bit.

2. `n_planes` co-varies with the FOURCC's "alpha-flag" column from
   [`01-file-header-and-fourccs.md`](01-file-header-and-fourccs.md) §4.1:
   `n_planes = 1` for Gray, `n_planes = 3` for non-alpha non-gray,
   `n_planes = 4` for alpha. The set `{1, 3, 4}` exactly matches the
   set of plane counts implied by §4.1's families (Gray / RGB or YUV /
   RGBA or YUVA). `n_planes = 2` is **never observed** — there is no
   semi-planar (NV12-style Y+UV) Backend on the wire side. Semi-planar
   *input* formats (e.g. `P010`, `P210` per
   [`01-file-header-and-fourccs.md`](01-file-header-and-fourccs.md) §4.2)
   exist as input-only frontends and are converted to a 3-plane
   Backend before encoding.

3. `(sub_x, sub_y)` ∈ {(1,1), (2,1), (2,2)}. The combination (1,2) is
   absent (would correspond to vertical-only chroma subsampling, which
   isn't a published 4:x:y layout). The vendor change log
   ([the vendor change log](https://www.magicyuv.com/change-log/))
   confirms 4:0:0 / 4:2:0 / 4:2:2 / 4:4:4 as the supported set, mapping
   exactly to `n_planes=1` / `(2,2)` / `(2,1)` / `(1,1)` for the YUV
   family.

   The subsampling parameters apply only to non-luma planes; the
   luma plane (plane 0) always has full resolution. Evidence for
   this is the encoder allowlist mask at
   [`01-file-header-and-fourccs.md`](01-file-header-and-fourccs.md) §3.1
   that overrides codec-variant to Gradient when the format byte is in
   the YUV/Gray subset, plus the per-plane-iterating decoder slice
   loop at `magicyuv.dll!0x69b93ec0`–`0x69b94060` which advances the
   plane buffer by `(coded_width >> shift_x) * (coded_height >>
   shift_y)` between planes.

The 14 Backend instantiations partition into 4 bit-depth tiers, each
with its own subset of `(n_planes, sub_x, sub_y)` triples:

| `sample_bits` | `(n_planes, sub_x, sub_y)` triples present                                            |
| ------------- | ------------------------------------------------------------------------------------- |
| 8             | (1,1,1)   (3,1,1)   (3,2,1)   (3,2,2)   (4,1,1)                                       |
| 10            | (1,1,1)   (3,1,1)   (3,2,1)   (3,2,2)   (4,1,1)                                       |
| 12            | (3,1,1)   (4,1,1)                                                                     |
| 14            | (3,1,1)   (4,1,1)                                                                     |

The 8-bit and 10-bit tiers each have all 5 layouts; the 12-bit and
14-bit tiers only have the 4:4:4 RGB / RGBA layouts. This matches the
table in [`00-scope.md`](00-scope.md) §"In scope".

## 3. Format byte → Backend mapping

Cross-referencing the FOURCC table in
[`01-file-header-and-fourccs.md`](01-file-header-and-fourccs.md) §4.1
against §1's Backend roster gives a unique Backend per `format_byte`
in `0x65..0x7e`. The mapping is fully determined by the FOURCC's `bpp`
column (which encodes bit-depth × planar tier — see §3.1) and its
`alpha-flag` column.

| `format_byte` | FOURCC  | bpp | α | Family description (§4.1)         | Backend                                |
| ------------- | ------- | --- | - | --------------------------------- | -------------------------------------- |
| `0x65`        | `M8RG`  | 24  | 0 | RGB, 8-bit                        | `Backend<u8, 8, 3, 1, 1>`              |
| `0x66`        | `M8RA`  | 32  | 1 | RGBA, 8-bit                       | `Backend<u8, 8, 4, 1, 1>`              |
| `0x67`        | `M8Y4`  | 24  | 0 | YUV 4:4:4, 8-bit                  | `Backend<u8, 8, 3, 1, 1>`              |
| `0x68`        | `M8Y2`  | 24  | 0 | YUV 4:2:2, 8-bit                  | `Backend<u8, 8, 3, 2, 1>`              |
| `0x69`        | `M8Y0`  | 24  | 0 | YUV 4:2:0, 8-bit                  | `Backend<u8, 8, 3, 2, 2>`              |
| `0x6a`        | `M8YA`  | 32  | 1 | YUVA 4:4:4:4, 8-bit               | `Backend<u8, 8, 4, 1, 1>`              |
| `0x6b`        | `M8G0`  | 24  | 0 | Gray (4:0:0), 8-bit               | `Backend<u8, 8, 1, 1, 1>`              |
| `0x6c`        | `M0Y2`  | 20  | 0 | YUV 4:2:2, 10-bit                 | `Backend<i16, 10, 3, 2, 1>`            |
| `0x6d`        | `M0RG`  | 30  | 0 | RGB, 10-bit                       | `Backend<i16, 10, 3, 1, 1>`            |
| `0x6e`        | `M0RA`  | 40  | 1 | RGBA, 10-bit                      | `Backend<i16, 10, 4, 1, 1>`            |
| `0x6f`        | `M2RG`  | 36  | 0 | RGB, 12-bit                       | `Backend<i16, 12, 3, 1, 1>`            |
| `0x70`        | `M2RA`  | 48  | 1 | RGBA, 12-bit                      | `Backend<i16, 12, 4, 1, 1>`            |
| `0x71`        | `M4RG`  | 42  | 0 | RGB, 14-bit                       | `Backend<i16, 14, 3, 1, 1>`            |
| `0x72`        | `M4RA`  | 56  | 1 | RGBA, 14-bit                      | `Backend<i16, 14, 4, 1, 1>`            |
| `0x73`        | `M0G0`  | 10  | 0 | Gray (4:0:0), 10-bit              | `Backend<i16, 10, 1, 1, 1>`            |
| `0x76`        | `M0Y4`  | 30  | 0 | YUV 4:4:4, 10-bit                 | `Backend<i16, 10, 3, 1, 1>`            |
| `0x7b`        | `M0Y0`  | 15  | 0 | YUV 4:2:0, 10-bit                 | `Backend<i16, 10, 3, 2, 2>`            |

Two non-trivial points fall out of this table:

- **Same Backend for "RGB" and "YUV 4:4:4"**. `0x65` (M8RG) and `0x67`
  (M8Y4) both resolve to `Backend<u8, 8, 3, 1, 1>`; same for the
  alpha pair `0x66`/`0x6a` and the 10-bit pair `0x6d`/`0x76`. The
  Backend stores three full-resolution planes either way; the
  `format_byte` is what distinguishes "the three planes are G/B/R"
  (RGB family, see §4) from "the three planes are Y/U/V" (YUV
  family, see §5). The semantics are different even when the
  storage layout is identical.

- **No 12/14-bit YUV format on the wire.** The 12-bit and 14-bit tiers
  only have RGB and RGBA layouts. The
  [`00-scope.md`](00-scope.md) §"In scope" table reflects this: YUV
  on the wire goes up to 10-bit only. The change-log lower bound on
  YUV 12/14-bit was speculative; this binary does not realise it.

The unmapped `format_byte` values in `0x65..0x7e` — `0x74`, `0x75`,
`0x77`, `0x78`, `0x79`, `0x7a`, `0x7c`, `0x7d`, `0x7e` — are absent
from §4.1. Of these, the encoder allowlist (per
[`01-file-header-and-fourccs.md`](01-file-header-and-fourccs.md) §3.1
mask `0xf1903f` rebased at `0x67`) admits `0x77`, `0x7c`, `0x7d`,
`0x7e`; this is open-question 4 in §6 of that chapter and is not
resolved here.

## 4. Plane order and semantics — RGB family

For RGB-family `format_byte` values (`0x65 M8RG`, `0x66 M8RA`,
`0x6d M0RG`, `0x6e M0RA`, `0x6f M2RG`, `0x70 M2RA`, `0x71 M4RG`,
`0x72 M4RA`), the **user-facing plane order** is **G, B, R** (and
**A** when present). The **on-wire plane order** in v2.4.2 streams
is different: the encoder applies a per-pixel inter-plane
decorrelation `B' = B − G` and `R' = R − G` (mod `2**bit_depth`)
**before** prediction, and writes the planes in the order
**`(B', G, R')`** (with **A** appended when present). A decoder
thus reads:

```
wire_plane[0] = B' = (B − G) mod 2^bit_depth
wire_plane[1] = G
wire_plane[2] = R' = (R − G) mod 2^bit_depth
wire_plane[3] = A    (when present, no decorrelation)
```

and reverses with `B = (wire_plane[0] + wire_plane[1]) mod 2^bit_depth`,
`R = (wire_plane[2] + wire_plane[1]) mod 2^bit_depth`. Empirical
evidence: validating a Python reference encoder against the
proprietary decoder on solid-blue / solid-green / mixed RGB inputs
at `M8RG 16×16` shows **bit-exact** wire bytes when the encoder
applies the `(B−G, G, R−G)` transform and emits planes in that
order, and only then; without the transform the proprietary decoder
produces garbage / silent zero output.

The per-slice Huffman descriptor and predictor (per
[`04-prediction-modes.md`](04-prediction-modes.md) and
[`05-entropy-coding.md`](05-entropy-coding.md)) apply per-plane to
the **decorrelated** plane values. The plane-index mapping table at
the end of this section gives the user-facing GBR order; the wire's
column-1 / column-2 entries carry the decorrelated values.

Evidence:

1. **Vendor user-facing names** at `magicyuv.dll@0x24a8d8`–`@0x24a9b8`
   advertise the RGB native layouts as **GBR planar**:

   ```
   @0x24a8d8: "RGB 10-bit - (r210,R10k,G3[0][10],GBAL)"
   @0x24a900: "RGBA 10-bit - (G4[0][10])"
   @0x24a91a: "RGB 12-bit - (G3[0][12])"
   @0x24a933: "RGBA 12-bit - (G4[0][12])"
   @0x24a94d: "RGB 14-bit - (G3[0][14])"
   @0x24a966: "RGBA 14-bit - (G4[0][14])"
   ```

   The bracketed strings are FOURCCs — `G3\x00\x0a` ≡ "GBR planar 3
   planes 10-bit"; `G4\x00\x0a` ≡ "GBR planar 4 planes 10-bit"
   (alpha). These FOURCCs also appear in
   [`01-file-header-and-fourccs.md`](01-file-header-and-fourccs.md) §4.2
   at `format_id` `0x24` (GBAL) through `0x2b`, with vendor labels
   "RGB 10-bit planar GBR" through "RGBA 16-bit planar GBR". The
   string `GBAL` decodes from "GBR-A-Limited" or "Avid Green-Blue-Red
   Aligned" depending on documentation set; either way, the channel
   prefix order is **G B (A) R**.

2. **GBR-input frontend family in `.rdata`**. The 18 input-side
   typeinfo records at file `0x259c80`–`0x25a0c0` belong to the
   GBR-input frontend family (a class template parameterised on
   `<input_bits, output_bits, has_alpha>`). The family's RTTI name
   itself, and the parameter shape, mark the input convention as
   **GBR**, matching the vendor user-facing strings cited above.
   The frontend converts a planar GBR input into the Backend's
   internal storage. The byte-level conversion preserves plane
   order: the family's parameter shape contains no permutation
   parameter — only `(input_bits, output_bits, has_alpha)` — so
   no per-channel reordering is possible.

3. **Encoder allowlist mask** (per
   [`01-file-header-and-fourccs.md`](01-file-header-and-fourccs.md) §3.1).
   The mask `0xf1903f` rebased at `0x67` matches `{0x67, 0x68, 0x69,
   0x6a, 0x6b, 0x6c, 0x73, 0x76, 0x77, 0x7b, 0x7c, 0x7d, 0x7e}` —
   exactly the YUV/Gray family. The RGB family (`0x65, 0x66, 0x6d,
   0x6e, 0x6f, 0x70, 0x71, 0x72`) is the complement, and is present
   in the decoder fast-path mask `0x19bf03` (rebased at `0x65`). The
   two masks taken together establish that the codec's binary
   distinguishes the RGB family from the YUV family at the wire-format
   level.

The plane order is therefore:

| Plane index | RGB (no α) | RGBA (with α) |
| ----------- | ---------- | ------------- |
| 0           | G          | G             |
| 1           | B          | B             |
| 2           | R          | R             |
| 3           | (n/a)      | A             |

All RGB-family Backends use `(sub_x, sub_y) = (1, 1)`. Per-plane
dimensions are `(width, height)` for every plane.

## 5. Plane order and semantics — YUV family

For YUV-family `format_byte` values (`0x67 M8Y4`, `0x68 M8Y2`, `0x69
M8Y0`, `0x6a M8YA`, `0x6c M0Y2`, `0x76 M0Y4`, `0x7b M0Y0`), the
wire-format plane order is **Y, U, V** (then **A** when present, at
`0x6a M8YA`).

Evidence:

1. **Planar-YUV-input frontend family in `.rdata`**. The 21
   typeinfo records at file `0x2595c0`–`0x259ac0` belong to the
   planar-YUV-input frontend family (a class template parameterised
   on `<input_bits, output_bits, n_planes, sub_x, sub_y, b1, b0>`).
   The family's RTTI name marks the input convention as **YUV**, not
   **YVU**. The codec maintains separate dedicated handlers for the
   two ordering conventions on the input side — one set covering
   I444/I420-style Y-U-V inputs and another covering YV24/YV12-style
   Y-V-U inputs (their symbol strings appear at file
   `0x2593c0`–`0x259420`) — and presumably converts both to the
   planar-YUV family's canonical order before passing to the Backend.

2. **Channel-permutation evidence on the packed-input side**. For the
   v308 packed-YUV-4:4:4 8-bit input format (per
   [`01-file-header-and-fourccs.md`](01-file-header-and-fourccs.md) §4.2,
   `format_id 0x2c`), the input handler is the 24-bit-input frontend
   template instantiated with three planes, an inter-plane
   decorrelation helper carrying axis indices `(1, 2, 0)`, and a
   YUV-family decorrelation policy. Its typeinfo record sits at file
   `0x2572e0`. The decorrelation helper specifies the permutation
   `output_plane[k] = input_channel[j_k]`, where `(j_0, j_1, j_2) =
   (1, 2, 0)`. v308's in-memory byte order per Apple QuickTime /
   Microsoft DirectX documentation
   (<https://learn.microsoft.com/en-us/windows/win32/medfound/recommended-8-bit-yuv-formats-for-video-rendering#packed-yuv-formats>)
   is `(V, Y, U)` per pixel — input channel 0 = V, channel 1 = Y,
   channel 2 = U. Applying axis indices `(1, 2, 0)`:

   - output plane 0 ← input channel 1 = Y
   - output plane 1 ← input channel 2 = U
   - output plane 2 ← input channel 0 = V

   This pins the wire plane order at **(Y, U, V)** for the YUV family.

3. **Alpha-channel ordering from 32-bit packed YUVA input**. The
   packed input formats v408 (YUVA 4:4:4:4 8-bit, `format_id 0x2d` /
   `0x2e` per
   [`01-file-header-and-fourccs.md`](01-file-header-and-fourccs.md) §4.2)
   and AYUV (`format_id 0x08`) are handled by 32-bit-input frontend
   instantiations that pair a 4-axis inter-plane decorrelation helper
   with the YUV-family decorrelation policy; their typeinfo records
   span file `0x257660`–`0x2588c0`. Three distinct axis-index tuples
   appear in this 32-bit-input + YUV combination:

   - axes `(1, 0, 2, 3)` at file `0x257660`, `0x257d80`, `0x2585c0`.
   - axes `(2, 1, 0, 3)` at file `0x257840`, `0x258020`, `0x2588c0`.
   - axes `(1, 2, 3, 0)` at file `0x258680`.

   Each maps a different 32-bit packed-pixel byte-order convention
   to the same canonical plane order:

   - **v408** byte order per Microsoft documentation
     (<https://learn.microsoft.com/en-us/windows/win32/medfound/recommended-8-bit-yuv-formats-for-video-rendering#packed-yuv-formats>)
     is `(U, Y, V, A)` — channel 0 = U, channel 1 = Y, channel 2 = V,
     channel 3 = A. Applying axes `(1, 0, 2, 3)`:

     - output plane 0 ← input channel 1 = Y
     - output plane 1 ← input channel 0 = U
     - output plane 2 ← input channel 2 = V
     - output plane 3 ← input channel 3 = A

   - **AYUV** byte order per the same Microsoft documentation is
     `(V, U, Y, A)` (low-to-high in memory; the LE 32-bit word is
     `0xAAYYUUVV`). Channel 0 = V, channel 1 = U, channel 2 = Y,
     channel 3 = A. Applying axes `(2, 1, 0, 3)`:

     - output plane 0 ← input channel 2 = Y
     - output plane 1 ← input channel 1 = U
     - output plane 2 ← input channel 0 = V
     - output plane 3 ← input channel 3 = A

   - **Axes `(1, 2, 3, 0)`** (file `0x258680`) correspond to a
     third byte-order convention `(A, Y, U, V)` (channel 0 = A, 1 = Y,
     2 = U, 3 = V). This produces planes (Y, U, V, A) by the same
     logic. The specific input format that takes this permutation is
     not pinned down from RTTI alone, but the output plane order is
     consistent.

   All three permutations produce wire plane order **(Y, U, V, A)**.
   This is consistent with §5.2's per-format-byte argument and pins
   the YUVA wire plane order at **(Y, U, V, A)** independently of the
   packed-input byte-order convention.

The plane order is therefore:

| Plane index | YUV (no α) | YUVA (with α) | Per-plane width                   | Per-plane height                  |
| ----------- | ---------- | ------------- | --------------------------------- | --------------------------------- |
| 0           | Y          | Y             | `width`                           | `height`                          |
| 1           | U          | U             | `width  / sub_x` (rounded — §7)   | `height / sub_y` (rounded — §7)   |
| 2           | V          | V             | `width  / sub_x` (rounded — §7)   | `height / sub_y` (rounded — §7)   |
| 3           | (n/a)      | A             | `width`                           | `height`                          |

The alpha plane (when present) is at full resolution regardless of
the chroma subsampling — i.e. there is no `Backend<u8, 8, 4, 2, 1>`
or similar in the §1 roster, so YUVA only exists at 4:4:4:4
(`(sub_x, sub_y) = (1, 1)`). The vendor names also confirm this
("YUVA 4:4:4:4 - (AYUV,v408)" at `magicyuv.dll@0x24a771`); there is
no published "YUVA 4:2:2:4" or similar in v7.

## 6. Plane order and semantics — Gray family

For Gray-family `format_byte` values (`0x6b M8G0`, `0x73 M0G0`), the
single wire plane is the luma plane:

| Plane index | Gray | Per-plane width | Per-plane height |
| ----------- | ---- | --------------- | ---------------- |
| 0           | Y    | `width`         | `height`         |

`Backend<u8, 8, 1, 1, 1>` and `Backend<i16, 10, 1, 1, 1>` are the only
1-plane Backends. There is no 12/14-bit Gray on the wire (consistent
with §3's table — those tiers only have 4:4:4 RGB / RGBA).

## 7. Per-plane sample storage

### 7.1  8-bit family

For every Backend with `sample_bits = 8` (i.e. `format_byte` in
{`0x65`, `0x66`, `0x67`, `0x68`, `0x69`, `0x6a`, `0x6b`}), each
sample occupies exactly one byte (`unsigned char`). The wire-format
sample is the raw 8-bit value; no padding or alignment.

The sample type `h` (= `unsigned char` in Itanium ABI mangling) is
direct binary evidence: every 8-bit Backend uses `h` (file offsets
`0x256a00`–`0x256b00`).

Per-plane byte count = `(width / sub_x) × (height / sub_y)` (with
`sub_x = sub_y = 1` except for `0x68` 4:2:2 and `0x69` 4:2:0).

### 7.2  10/12/14-bit families

For every Backend with `sample_bits ∈ {10, 12, 14}`, each sample
occupies exactly two bytes (`signed short`, mangled letter `s`). The
sample is **not packed**: no 10-bit-in-2-bytes-with-residual-bits
shared across pixels; each sample is its own 16-bit slot. So the
on-the-wire bytes-per-plane-pixel are 2, regardless of whether
`sample_bits` is 10, 12, or 14.

Evidence:

- **All `s`-typed Backends in §1**. The use of `signed short`
  exclusively for sample storage in 10/12/14-bit Backends is
  established by the RTTI strings at file `0x256b40`–`0x256d40`.
  No `t` (= `unsigned short`) or other 2-byte container type is
  observed.

- **No "packed" Backend variants**. The only family-backend
  templates in `.rdata` are the `<sample_type, sample_bits, n_planes,
  sub_x, sub_y>`-shaped one rostered in §1 and the unrelated 12-bit
  Bayer-CFA layout class also noted in §1; neither has a "packed"
  parameter. The "packed" v210 / r210 / R10k / V210 / V410 input
  representations and the packed 10-bit-YUV input family (per
  [`01-file-header-and-fourccs.md`](01-file-header-and-fourccs.md) §4.2)
  are **input-side only**; the corresponding frontend handlers for
  V210, V410, R210/R10K, and packed 10-bit YUV input (their typeinfo
  records span file `0x259540`–`0x25a220`) unpack each input pixel
  into one 16-bit per-channel sample before the Backend sees it.

- **Width-padding evidence**. The encoder bitset-test instruction at
  `magicyuv.dll!0x69b85780` (file `@0x4d80`) loads the immediate
  `0x2000100252000200` and tests a per-format-id bit; the test pads
  width to a multiple of 4 only for the format IDs in that mask (set
  bits at positions 9, 25, 28, 30, 33, 44, 61 — the input-side
  "packed" formats Y8 / b48r / RGB0 / `Y1\x00\x0a` / BGR0 / v308 /
  `Y1\x00\x10`). MagicYUV-native `format_byte` values do **not**
  appear in this mask. So the wire format does not impose its own
  alignment beyond the per-plane sub-sampling.

**The unused MSBs** (bits 10–15 for 10-bit, bits 12–15 for 12-bit,
bits 14–15 for 14-bit) of the 16-bit slot have not been pinned to a
specific value by static analysis alone. The RTTI uses `s` (signed)
container which permits negative values; pixel values are nominally
non-negative but residuals (after prediction) are signed. Whether the
**stored sample** is the predicted residual or the reconstructed pixel
value, and whether sign-extension of negative residuals fills bits ≥
`sample_bits` with the sign bit, is deferred to the entropy-coding
chapter ([`05-entropy-coding.md`](05-entropy-coding.md)) where the
question of "what is encoded" is more naturally answered. The
container size (2 bytes) and endianness (LE — see §7.3) are
nonetheless pinned in this chapter.

### 7.3  Endianness

All multi-byte integer fields in the v7 wire format are little-endian
per [`00-scope.md`](00-scope.md) §"Conventions". This applies to the
2-byte sample slots in §7.2: the low byte is at the lower file
position, the high byte at the higher file position.

The 32-bit `width` and `height` fields at header offsets `0x10` /
`0x14` are little-endian by the same convention. The encoder
stores them at `magicyuv.dll!0x69b9767d` (see
[`01-file-header-and-fourccs.md`](01-file-header-and-fourccs.md) §3.2);
on x86-64 this is a native LE store.

## 8. Per-plane dimensions and rounding

### 8.1  Luma and alpha planes

The luma plane (and the alpha plane, when present) always has
dimensions `(width, height)` exactly equal to the header fields at
`+0x10` / `+0x14`. No subsampling, no padding.

### 8.2  Chroma planes

The U and V planes have dimensions `(ceil(width / sub_x),
ceil(height / sub_y))`.

Rounding behaviour (ceiling vs. floor) at odd resolutions is
unverified by static analysis alone. The 1.1 change log entry "Fixed
compatibility issues with odd resolutions" implies non-trivial
handling, and the 2.0.0 entry "Fixed YUV 4:0:0 → Y8 decoding for odd
resolutions" implies this was further refined. The encoder's slice
loop at `magicyuv.dll!0x69b93ec0` performs per-plane offset arithmetic
that hints at an arithmetic shift `>> shift` of width/height (where
`shift` is `log2(sub_x)` or `log2(sub_y)`), which is a floor
operation. A floor operation can lose the last column / row of
chroma if `width`/`height` is odd; the encoder may compensate by
upsampling odd dimensions up to the nearest even number first.
Behavioural-trace work running the codec on `1×1`,
`3×1`, `1×3`, `3×3` and similar odd-size frames would pin this down.

For even `width` and `height` (the common case), `ceil = floor` and
the per-plane dimensions are simply `(width / sub_x, height / sub_y)`.

## 9. Summary table: format byte → wire plane sizes

For convenience, here is the per-plane wire size (in bytes,
assuming even `width` and `height`) for every native `format_byte`
in §3:

| `format_byte` | Plane 0                                  | Plane 1                                            | Plane 2                                            | Plane 3                                  |
| ------------- | ---------------------------------------- | -------------------------------------------------- | -------------------------------------------------- | ---------------------------------------- |
| `0x65 M8RG`   | G: `W × H` × 1 B                         | B: `W × H` × 1 B                                   | R: `W × H` × 1 B                                   | —                                        |
| `0x66 M8RA`   | G: `W × H` × 1 B                         | B: `W × H` × 1 B                                   | R: `W × H` × 1 B                                   | A: `W × H` × 1 B                         |
| `0x67 M8Y4`   | Y: `W × H` × 1 B                         | U: `W × H` × 1 B                                   | V: `W × H` × 1 B                                   | —                                        |
| `0x68 M8Y2`   | Y: `W × H` × 1 B                         | U: `(W/2) × H` × 1 B                               | V: `(W/2) × H` × 1 B                               | —                                        |
| `0x69 M8Y0`   | Y: `W × H` × 1 B                         | U: `(W/2) × (H/2)` × 1 B                           | V: `(W/2) × (H/2)` × 1 B                           | —                                        |
| `0x6a M8YA`   | Y: `W × H` × 1 B                         | U: `W × H` × 1 B                                   | V: `W × H` × 1 B                                   | A: `W × H` × 1 B                         |
| `0x6b M8G0`   | Y: `W × H` × 1 B                         | —                                                  | —                                                  | —                                        |
| `0x6c M0Y2`   | Y: `W × H` × 2 B                         | U: `(W/2) × H` × 2 B                               | V: `(W/2) × H` × 2 B                               | —                                        |
| `0x6d M0RG`   | G: `W × H` × 2 B                         | B: `W × H` × 2 B                                   | R: `W × H` × 2 B                                   | —                                        |
| `0x6e M0RA`   | G: `W × H` × 2 B                         | B: `W × H` × 2 B                                   | R: `W × H` × 2 B                                   | A: `W × H` × 2 B                         |
| `0x6f M2RG`   | G: `W × H` × 2 B                         | B: `W × H` × 2 B                                   | R: `W × H` × 2 B                                   | —                                        |
| `0x70 M2RA`   | G: `W × H` × 2 B                         | B: `W × H` × 2 B                                   | R: `W × H` × 2 B                                   | A: `W × H` × 2 B                         |
| `0x71 M4RG`   | G: `W × H` × 2 B                         | B: `W × H` × 2 B                                   | R: `W × H` × 2 B                                   | —                                        |
| `0x72 M4RA`   | G: `W × H` × 2 B                         | B: `W × H` × 2 B                                   | R: `W × H` × 2 B                                   | A: `W × H` × 2 B                         |
| `0x73 M0G0`   | Y: `W × H` × 2 B                         | —                                                  | —                                                  | —                                        |
| `0x76 M0Y4`   | Y: `W × H` × 2 B                         | U: `W × H` × 2 B                                   | V: `W × H` × 2 B                                   | —                                        |
| `0x7b M0Y0`   | Y: `W × H` × 2 B                         | U: `(W/2) × (H/2)` × 2 B                           | V: `(W/2) × (H/2)` × 2 B                           | —                                        |

`W` = `width` (header `+0x10`). `H` = `height` (header `+0x14`).
The byte count is the **uncompressed** plane size; the compressed
representation lives inside the slice payload and is the subject of
[`05-entropy-coding.md`](05-entropy-coding.md).

## 10. Open questions

1. **Sample storage of high-bit-depth values: residual or raw?**
   The 16-bit slots in 10/12/14-bit Backends are `signed short`,
   indicating residuals (which can be negative) are a possibility.
   Whether the Backend stores raw pixel values in `[0, 2^bits − 1]`
   and prediction is done in-place (overwriting the buffer with a
   residual at encode time, then re-overwriting with the
   reconstructed value at decode time), or whether the Backend's
   buffer holds residuals at the wire-format-payload level, is not
   pinned down by the RTTI alone. This is properly answered by the
   entropy-coding chapter
   ([`05-entropy-coding.md`](05-entropy-coding.md)).

2. **Odd-resolution subsampling rounding.** §8.2: do `1×1`, `3×3`
   etc. coded sizes round chroma plane dimensions up or down? The
   1.1 vendor change log
   ([the vendor change log](https://www.magicyuv.com/change-log/))
   mentions "Fixed compatibility issues with odd resolutions" and
   the 2.0.0 entry mentions "Fixed YUV 4:0:0 → Y8 decoding for odd
   resolutions"; the v2.4.2 binary's actual wire-format rule needs a
   behavioural trace.

3. **Bayer-CFA path.** The 12-bit Bayer-CFA layout class noted at
   the end of §1 (file `0x25a100`) exists in the binary but no
   `format_byte` in §4.1 reaches it. Is this an internal-only
   conversion target (e.g. ARRIRAW → demosaiced internal RGB), a
   future-reserved wire-format path, or a plugin-only feature?
   A binary review tracing where the paired 12-bit-packed Bayer
   input frontends at file `0x25a620`/`0x25a660`/`0x25a6a0`
   become reachable would close this.

4. **Explicit luma-plane numbering for YUV 4:0:0.** In §6 the
   single plane is labelled "Y". The vendor uses `Y` (luminance)
   in user-facing strings (e.g. "YUV 4:0:0 - (Y8,Y800,GREY)" at
   `magicyuv.dll@0x24a73c`) but this could equally well be called
   "L" or "K". The chapter's labelling is by codec convention,
   not a behavioural test.

5. **Format bytes `0x77`, `0x7c`, `0x7d`, `0x7e`** are present in
   the encoder allowlist
   ([`01-file-header-and-fourccs.md`](01-file-header-and-fourccs.md) §3.1
   mask `0xf1903f`) but not in §4.1's published FOURCC enumeration.
   This is also flagged as an open question in
   [`01-file-header-and-fourccs.md`](01-file-header-and-fourccs.md) §6.
   They might correspond to additional YUV/Gray Backend
   instantiations (e.g. `Backend<i16, 12, 3, 2, 1>`, which would be a
   YUV 4:2:2 12-bit). However, no such Backend RTTI string exists in
   the binary's §1 roster, so the more likely explanation is that
   these `format_byte` values are reserved for non-AVI carriage paths
   (e.g. internal pre-conversion) and never appear in MAGY frames on
   disk.
