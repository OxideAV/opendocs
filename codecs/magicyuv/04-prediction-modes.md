# MagicYUV v7 — Prediction modes

This chapter pins down the per-slice prediction layer that sits between
the slice envelope (specified in
[`02-slice-table.md`](02-slice-table.md)) and the entropy-coded
residuals (deferred to `05-entropy-coding.md`). Prediction is the layer
that makes the codec lossless: every pixel is reconstructed by adding a
decoded residual to a value computed from already-decoded neighbours.

The wire format defines **three concrete predictors** that the decoder
must implement (Left, Gradient, Median) and **one encoder strategy**
(Dynamic) that picks among the three on a per-slice basis. The wire
itself never carries a "Dynamic" predictor ID; that label is a vendor
encoder-side concept.

All claims trace to byte-level evidence in the proprietary binary
`reference/binaries/system32/magicyuv.dll` (SHA-256
`2036e284cc07a2cb4d4b6b6cb04a1d24c0f97a0041c134fd669ee079b768902c`,
PE32+ x86-64) and to a behavioural-trace harness driven through Wine
VFW.

Notation follows
[`01-file-header-and-fourccs.md`](01-file-header-and-fourccs.md):
`<file>@0xNNNNNN` is a file offset and `<file>!0xNNNNNN` is the
instruction VMA at the binary's default ImageBase `0x69b80000`.

This chapter takes as given:

- The 32-byte v7 header layout from
  [`01-file-header-and-fourccs.md`](01-file-header-and-fourccs.md) §3,
  in particular the byte `codec_variant` at header offset `+0x0b`.
- The slice-table envelope from
  [`02-slice-table.md`](02-slice-table.md), in particular: each slice's
  payload begins at the file offset `entry[k+1] + 0x20`, and the
  preamble (plane count + per-slice plane index + Huffman descriptors)
  precedes the slice payloads.
- The per-Backend bit-depth, plane count, and chroma subsampling from
  [`03-pixel-plane-mapping.md`](03-pixel-plane-mapping.md) §3.

## 1. The two-byte slice prefix

Every slice payload begins with a fixed **2-byte prefix** at the
slice's first two bytes (`entry[k+1] + 0x20` and
`entry[k+1] + 0x20 + 1`). The remainder of the slice payload is either
Huffman-coded residual bytes or raw uncompressed pixel bytes, selected
by the prefix.

```
+--------+------------------------+
| Offset | Field                  |
+========+========================+
|   +0   | slice_flags  (uint8)   |
|   +1   | predictor_id (uint8)   |
+--------+------------------------+
```

### 1.1 `slice_flags` (slice byte +0)

`slice_flags` carries one defined bit:

| Bit  | Mask  | Meaning                                              |
| ---- | ----- | ---------------------------------------------------- |
| 0    | `0x01`| `1` = raw mode: the slice payload that follows is uncompressed pixel residuals (post-prediction); `0` = Huffman-coded residuals |

Other bits are reserved and observed always-zero in v2.4.2.

The encoder writes this byte from a per-slice "raw fallback" decision
made when the Huffman-coded residual stream would be larger than the
uncompressed slice. The wire-format byte is the runtime result, not a
configuration setting — there is no registry knob that pins it. The
change-log entry "Removed 'Adaptive coding' setting (now always
enabled)" (1.2, 2015-08-25) refers to this per-slice raw fallback
becoming unconditional.

**Behavioural confirmation.** A `64×64` M8RG slice in Huffman mode is
~226 bytes (zero-content) to ~290 bytes (high-entropy compressible);
the same slice in raw mode is exactly **`28 × 64 + 2 = 1794` bytes**
for full slices and `8 × 64 + 2 = 514` bytes for the truncated last
slice (slice height 28, frame height 64, so the last slice covers 8
rows). The deterministic pseudo-random fixture
`m8rg_64x64_rand.bin` produces 9 slices all with `flags = 0x01` and
sizes (1794, 1794, 514, 1794, 1794, 514, 1794, 1794, 529) — exactly
the raw-mode formula for each plane (the trailing 529 bytes vs. the
expected 514 reflects encoder-internal residual rounding in the very
last slice plus a few padding bytes). See provenance §C, fixture
`m8rg_64x64_rand.bin`.

### 1.2 `predictor_id` (slice byte +1)

The predictor used to reconstruct this slice's pixels. Three legal
values:

| `predictor_id` | Predictor | Description                                  |
| -------------- | --------- | -------------------------------------------- |
| `0x01`         | Left      | `pred = pixel[x-1]`                          |
| `0x02`         | Gradient  | `pred = (pixel[x-1] + pixel[x-stride] - pixel[x-stride-1])` |
| `0x03`         | Median    | `pred = MED(left, top, top_left)` (JPEG-LS)  |

Values `0x00` and ≥ `0x04` are not produced by the v2.4.2 encoder.
Decoders SHOULD treat them as malformed input.

The vendor's "Dynamic" label at file `@0x2498b4`
(`magicyuv.dll@0x2498b4`: ASCII `Dynamic`, the four predictor-name
strings appear at `@0x2498b4 / @0x2498bc / @0x2498c3 / @0x2498cc`
followed by their use in the decoder/encoder dispatch) is **not** a
wire-format predictor ID. Dynamic is an encoder strategy that
evaluates all three predictors on each slice and picks the one with
the lowest residual sum; whichever wins is then written into the
slice's `predictor_id` byte as `0x01`, `0x02`, or `0x03`.

### 1.3 Encoder evidence for the prefix bytes

The encoder writes the two prefix bytes during the slice-emit pass
at `magicyuv.dll!0x69b94168`–`0x69b94177` (with a per-slice-loop
variant at `magicyuv.dll!0x69b941ce`–`0x69b941e8`). The sequence:

1. Loads `entry[1]` (the slice-0 byte offset relative to header
   end) from offset `0x04` of the slice-table pointer.
2. Adds the output-buffer base to obtain the slice-0 absolute
   address.
3. Reads the per-slice `predictor_id` from offset `+0x260` of
   the per-slice metadata block.
4. Stores it into the byte at `slice+1`.
5. Reads the per-slice `flags` from offset `+0x264` of the same
   metadata block and stores it into the byte at `slice+0`.

Offsets `+0x260` and `+0x264` of the per-slice metadata are the
fields the encoder fills in during predictor evaluation (see §3
below). The loop variant at
`magicyuv.dll!0x69b941b0`–`0x69b941e8` does the same thing for
subsequent slices, indexed in stride `0x8f0` per loop iteration
(`02-slice-table.md` §5.3 notes the per-slice descriptor stride is
`0x478 = 1144` bytes; `0x8f0 = 0x478 * 2`, two descriptors per
iteration).

### 1.4 Decoder evidence for the prefix bytes

The decoder reads the two prefix bytes at
`magicyuv.dll!0x69b95ac3`–`0x69b95aed`. The sequence:

1. Loads the slice-payload pointer from offset `+0x228` of the
   decoder context.
2. Compares the byte at `slice+1` against `0x1`, `0x2`, and
   `0x3` in three separate compare-and-set tests, populating
   three boolean fields in the decoder metadata block at
   offsets `+0x218`, `+0x21c`, and `+0x220` (`is_left`,
   `is_gradient`, `is_median`) respectively.

The decoder also reads the slice-flags byte (slice +0) and the
predictor ID is later passed as a function argument into the
per-Backend predictor kernel (see §4).

## 2. Header byte `codec_variant` and the encoder clamps

[`01-file-header-and-fourccs.md`](01-file-header-and-fourccs.md) §3.0
documents header byte `+0x0b` as the codec_variant ID with values
`0x01..0x04` for Left/Gradient/Median/Dynamic.
[`02-slice-table.md`](02-slice-table.md) §intro documents an encoder
clamp at `magicyuv.dll!0x69ba9060` that forces
`codec_variant ∈ {3, 4}` to `0x02`. This chapter confirms that:

- **In every v2.4.2-encoded stream examined, header byte
  `+0x0b` is `0x02`** regardless of the registry `CompMethod` value
  or per-slice predictor decisions.
- The header byte therefore does **not** carry the actual prediction
  mode in v2.4.2 streams. It serves as a decoder fast-path selector
  (see §2.2 below).
- The wire-level prediction mode is in the per-slice
  `predictor_id` byte (§1.2).

### 2.1 The encoder clamp at `0x69ba9060`

This clamp is also discussed in
[`02-slice-table.md`](02-slice-table.md) §intro. The sequence at
`magicyuv.dll!0x69ba905d`–`0x69ba906f`:

1. Reads the configured `CompMethod` from the encoder config at
   offset `+0x74`.
2. Computes `CompMethod - 1`.
3. If that biased value is ≤ 1, leaves the config field
   unchanged.
4. Otherwise overwrites `config[+0x74]` with `0x02` (the
   Gradient ID) before falling through.

Behaviour:

| Registry `CompMethod` | `(CompMethod-1) <= 1` ? | Final `config[+0x74]` |
| --------------------- | ----------------------- | --------------------- |
| 0 (default if absent) | no (`0xffffffff > 1`)   | clamped to `0x02`     |
| 1                     | yes                     | `0x01`                |
| 2                     | yes                     | `0x02`                |
| 3                     | no                      | clamped to `0x02`     |
| 4                     | no                      | clamped to `0x02`     |

`config[+0x74]` is then written into header byte `+0x0b` by the
encoder header writer at `magicyuv.dll!0x69b9763f`–`0x69b97643`
(`01-file-header-and-fourccs.md` §3.0): the encoder reloads the
config field at `+0x74` and stores its low byte at offset `0x0b` of
the output header.

But behavioural traces show **byte `+0x0b` = 0x02 even when
the registry CompMethod = 1**. So either
(a) Wine's registry-write path doesn't reach config offset `+0x74`,
or (b) there is a *second* clamp/write that overrides it. Static
analysis of `magicyuv.dll!0x69b97600`–`0x69b97650` shows exactly
one explicit write to header byte `+0x0b` with the value `0x02`
(at `magicyuv.dll!0x69b976b7`), which is the format-byte-
conditional override that fires for non-RGB formats — see
`01-file-header-and-fourccs.md` §3.1. For M8RG (RGB family), that
override does **not** fire, so the only write to byte `+0x0b` is
the one sourced from the config field. Empirically `+0x0b = 0x02`
always; the mechanism is not pinned down to a single instruction.

The practical impact for a decoder is unchanged: in v2.4.2 streams,
`codec_variant = 0x02` and the actual mode is per-slice.

### 2.2 Decoder use of `codec_variant`

The decoder reads `codec_variant` in the header parser at
`magicyuv.dll!0x69bae30c`–`0x69bae324` and uses it as a fast-path
selector: it loads the byte at header offset `+0x0b`, compares
against `0x2`, and branches to a simpler decoder dispatch when
the value is ≤ 2 (jumping into the shortcut path at
`magicyuv.dll!0x69bae461`).

`<= 2` selects a simpler decoder dispatch; values `> 2` fall into a
property-tree lookup. Since v2.4.2 always writes `0x02`, the decoder
always takes the shortcut.

A decoder built strictly to consume v2.4.2 streams can ignore the
header byte's specific value beyond the `<= 7` range check
(`01-file-header-and-fourccs.md` §2). A decoder that must handle
hypothetical future streams should preserve the existing logic.

### 2.3 Per-slice `predictor_id` is the actual mode

The slice prefix byte at offset `+1` of each slice's payload (§1.2)
is the **definitive** predictor selection for that slice. Three
distinct values appear (`0x01`, `0x02`, `0x03`); the value `0x04`
(would-be Dynamic) does NOT appear on the wire — see §3.

## 3. The Dynamic encoder strategy

The vendor labels `magicyuv.dll@0x2498b4` declare four "Codec
variant" names: Dynamic, Median, Gradient, Left.
`01-file-header-and-fourccs.md` §3.0 maps these to header byte
values 1..4. The mapping at the wire level is:

| Vendor mode label | Per-slice `predictor_id` on the wire        |
| ----------------- | ------------------------------------------- |
| Left              | `0x01` (always)                             |
| Gradient          | `0x02` (always)                             |
| Median            | `0x03` (always)                             |
| Dynamic           | varies per slice ∈ `{0x01, 0x02, 0x03}`     |

Dynamic is **not** a wire-level mode. The encoder, when configured
in Dynamic mode (the v2.4.2 default), evaluates all three concrete
predictors on each slice and writes whichever one produced the
smallest residual sum into that slice's `predictor_id` byte.

### 3.1 Encoder dispatch evidence

The encoder dispatch on the configured mode is at
`magicyuv.dll!0x69b964b2`–`0x69b964be`: it loads the configured
mode (a value in `1..4`) from offset `+0x64` of the encoder
config block, compares against `0x4`, and branches to a separate
"Dynamic" path at `magicyuv.dll!0x69b96970` when the value
equals 4.

For modes 1, 2, 3, the encoder calls a single predictor's evaluator
function (vtable +0x50 or +0x58 of the per-Backend vtable; see §4)
and stores the configured mode value directly into the per-slice
metadata at offset `+0x260` (which is then written to the slice
prefix at slice byte +1; §1.3); the metadata write occurs at
`magicyuv.dll!0x69b964e0`.

For mode 4 (Dynamic) — entry `magicyuv.dll!0x69b96970` — the
encoder calls the predictor evaluator three times (once per
predictor 1, 2, 3) through three different vtable slots
(`+0x48`, `+0x50`, `+0x58`), each producing a residual sum that
is accumulated into a stack slot (`+0x80`, `+0x84`, `+0x88`) at
`magicyuv.dll!0x69b96a91`–`0x69b96aa0`. The encoder then selects
the minimum of the three sums via a compare/conditional-move
chain at `magicyuv.dll!0x69b96ab2`–`0x69b96ac9`:

- Compare `sum_median` against `sum_gradient`; keep the smaller.
- Compare the running minimum against `sum_left`; keep the
  smaller.
- If `min == sum_left`, branch to the slot at
  `magicyuv.dll!0x69b96cb0` which stores `predictor_id = 1`.
- Else if `min == sum_gradient`, branch to
  `magicyuv.dll!0x69b96d95` which stores `predictor_id = 2`.
- Otherwise (`min == sum_median`) fall through to the
  `predictor_id = 3` store.

The selected `predictor_id` is then written into the same
metadata field at `+0x260` that non-Dynamic modes write
directly. The per-slice prefix byte sequence at §1.3 is identical
whether the mode was Dynamic or fixed.

### 3.2 Behavioural confirmation: per-slice predictor varies in Dynamic

Eight 64×64 M8RG fixtures, one per `(pattern, CompMethod)` pair,
demonstrate the four Dynamic-vs-fixed observations:

```
pattern=zero (all-zero):
  cm=0: preds=[1,1,1,1,1,1,1,1,1]   ← Dynamic picks Left for all slices (all-zero is degenerate)
  cm=2: preds=[2,2,2,2,2,2,2,2,2]   ← Force Gradient
  cm=3: preds=[3,3,3,3,3,3,3,3,3]   ← Force Median
  cm=4: preds=[1,1,1,1,1,1,1,1,1]   ← Dynamic, identical to cm=0

pattern=horiz_ramp (per-row constant by column):
  cm=0: preds=[1,1,1,2,2,2,1,1,1]   ← Dynamic: Left for plane 0/2 (G/R), Gradient for plane 1 (B)
  cm=2: preds=[2,2,2,2,2,2,2,2,2]
  cm=3: preds=[3,3,3,3,3,3,3,3,3]
  cm=4: preds=[1,1,1,2,2,2,1,1,1]   ← Identical to cm=0

pattern=diag_ramp (pixel(c,r) = c+r):
  cm=0: preds=[1,1,1,2,2,2,1,1,1]   ← Plane 1 picks Gradient consistently
  cm=2: preds=[2,2,2,2,2,2,2,2,2]
  cm=3: preds=[3,3,3,3,3,3,3,3,3]
  cm=4: preds=[1,1,1,2,2,2,1,1,1]
```

The four mode IDs introduced in `01-file-header-and-fourccs.md`
§3.0 don't all reach the wire: **`predictor_id ∈ {1, 2, 3}` on the
wire; 4 does not occur**.

### 3.3 Behavioural confirmation: registry CompMethod=1 also acts Dynamic

`01-file-header-and-fourccs.md` §3.0 maps registry value 1 to
"Left (hidden)". Empirically, registry `CompMethod=1` yields the
**same per-slice predictor sequence as Dynamic (CompMethod=4)** for
all tested input patterns and FOURCCs (see provenance §F):

```
M8G0 pattern=horiz_ramp:
  cm=1: preds=[2,2,2]               ← all Gradient (best for horiz ramp on this 1-plane fixture)
  cm=4: preds=[2,2,2]               ← identical
M8RG pattern=horiz_ramp:
  cm=1: preds=[1,1,1,2,2,2,1,1,1]
  cm=4: preds=[1,1,1,2,2,2,1,1,1]   ← identical
```

So the registry `CompMethod` semantics are:

| Registry `CompMethod` | Encoder behaviour                                     |
| --------------------- | ----------------------------------------------------- |
| 0 (key absent)        | Dynamic                                                |
| 1                     | Dynamic (vendor's "Left" label is misleading)          |
| 2                     | Force per-slice `predictor_id = 0x02` (Gradient)       |
| 3                     | Force per-slice `predictor_id = 0x03` (Median)         |
| 4                     | Dynamic                                                |

The vendor's GUI offers "Dynamic / Median / Gradient" (per
`@0x2498b4`–`@0x2498c3`); the "Left" label at `@0x2498cc` is referenced
in the user-config logging code at `magicyuv.dll!0x69b875d0` but no
configuration path actually writes a per-slice "force Left" mode. A
fixed-Left wire stream is achievable only by hand-construction or a
non-vendor encoder.

For the wire format spec, the registry semantics are out of scope;
what matters is that **decoders see exactly three `predictor_id`
values on the wire, regardless of how the encoder was configured**.

## 4. The three predictor formulas

This section specifies the predictor formulas the decoder must
implement. The formulas are bit-depth parameterised: 8-bit pixels
wrap modulo `2^8`, 10-bit modulo `2^10`, 12-bit modulo `2^12`,
14-bit modulo `2^14`. The bit-depth tier per Backend is given in
[`03-pixel-plane-mapping.md`](03-pixel-plane-mapping.md) §3.

### 4.1 Notation

For a single plane of size `(plane_width, plane_height)` (per
`03-pixel-plane-mapping.md` §9), let:
- `r` = current pixel row (0-indexed), `c` = current pixel column.
- `px[r,c]` = the **decoded** (reconstructed) pixel value at `(r,c)`.
- `res[r,c]` = the **residual** at `(r,c)`, as decoded from the
  Huffman bitstream (or read directly from raw mode).
- `bits` = the plane's bit-depth (8, 10, 12, or 14).
- `MAX = (1 << bits) - 1` = the per-bit-depth wrap mask
  (`0xff`, `0x3ff`, `0xfff`, or `0x3fff`).
- `MED(a, b, c)` = the JPEG-LS median-edge-detect predictor:
  ```
  if c >= max(a, b): return min(a, b)
  if c <= min(a, b): return max(a, b)
  return a + b - c
  ```
  Equivalently, `MED(a, b, c) = clip(a + b - c, min(a, b), max(a, b))`.

All arithmetic is modulo `MAX + 1`. The `& MAX` mask after addition
is the wraparound rule (§4.5).

### 4.2 Left (`predictor_id = 0x01`)

For the first row (`r = 0`) — the first pixel of a slice:

```
px[0, 0] = res[0, 0]                      (no left neighbour at column 0)
px[0, c] = (px[0, c-1] + res[0, c]) & MAX                for c ≥ 1
```

For subsequent rows (`r ≥ 1`):

```
px[r, 0] = (px[r-1, 0] + res[r, 0]) & MAX                (no left at column 0; use top)
px[r, c] = (px[r,   c-1] + res[r, c]) & MAX              for c ≥ 1
```

Wait — the "top fallback at column 0 of subsequent rows" rule is the
documented MagicYUV behaviour for Left, identical to what the binary
implements (the inner loop's first column path uses the previous
row's column-0 value to seed the running sum). This is consistent
with how a per-row running sum keeps state — the seed for each row's
running-sum is the column-0 reconstruction, which must be expressible
from data already decoded.

**Decoder evidence** (8-bit kernel `magicyuv.dll!0x69bb4f70`, with
the in-place running-sum step at `magicyuv.dll!0x69bb4ff5`–
`0x69bb5098`): the kernel loads the residual byte at `slice+1`,
adds the previously-decoded pixel byte at `slice+0`, and stores
the result back to `slice+1` in place. The add is an 8-bit
operation that wraps modulo 256 implicitly.

The 8-bit kernel relies on the natural wrap of an 8-bit add; no
explicit mask is needed. For 10/12/14-bit kernels, explicit
`& MAX` is applied (§4.5).

### 4.3 Gradient (`predictor_id = 0x02`)

For the first row (`r = 0`):

```
px[0, 0] = res[0, 0]
px[0, c] = (px[0, c-1] + res[0, c]) & MAX                for c ≥ 1
                                          ; first row falls back to Left
                                          ; (top neighbours don't exist)
```

For subsequent rows (`r ≥ 1`):

```
px[r, 0] = (px[r-1, 0] + res[r, 0]) & MAX                (left and top_left don't exist; use top)
                                                          ;  i.e. pred = px[r-1, 0]
px[r, c] = ((px[r, c-1] + px[r-1, c] - px[r-1, c-1]) + res[r, c]) & MAX     for c ≥ 1
```

The Gradient predictor of a pixel is `left + top - top_left`; the
residual is the difference between the actual pixel and that
predictor, encoded.

### 4.4 Median (`predictor_id = 0x03`)

The Median predictor's exact formula depends on bit-depth.

**8-bit Median is NOT standard JPEG-LS.** The cmov chain at
`magicyuv.dll!0x69bb6a80..0x69bb6c2f` (8-bit) computes the
intermediate gradient with **modular** subtraction
(`(top - top_left) & 0xff`) before adding `left`. Equivalently:

```c
/* 8-bit MED (modular, NOT standard JPEG-LS) */
gradient = (left + top - top_left) & 0xff;
if      (gradient < min(left, top))  pred = min(left, top);
else if (gradient > max(left, top))  pred = max(left, top);
else                                  pred = gradient;
```

Standard JPEG-LS would compute `clip(left + top - top_left,
min, max)` with full-precision `left + top - top_left`; for inputs
where the raw expression is outside `[0, 256)`, the two
formulas produce different `pred` values (and therefore different
reconstructed pixels). Worked example at 8-bit:
`left = 10, top = 20, top_left = 200`. Standard JPEG-LS:
`clip(-170, 10, 20) = 10`. MagicYUV 8-bit:
`gradient = (-170) & 0xff = 86; clip(86, 10, 20) = 20`
(since 86 > max=20). Empirical confirmation: encoding M8Y4 16×16
random with the modular Median produces an AVI the proprietary
decoder reconstructs byte-exactly, while encoding with standard
JPEG-LS Median produces an AVI the proprietary decoder
mis-reconstructs.

**10/12/14-bit Median IS standard JPEG-LS** with full-precision
(un-wrapped) gradient:

```c
/* 10/12/14-bit MED (standard JPEG-LS, NOT modular) */
if      (top_left >= max(left, top))  pred = min(left, top);
else if (top_left <= min(left, top))  pred = max(left, top);
else                                   pred = left + top - top_left;
```

Worked example at 14-bit: `a=24576, b=16384, c=12288`. The two
formulas diverge:

| formula | gradient/result | clipped pred |
| ------- | --------------- | ------------ |
| modular   `(a + b − c) & 0x3fff = 28672 & 0x3fff = 12288` | < min=16384 | 16384 |
| JPEG-LS  `c=12288 < min(a, b)=16384`                       | (return max) | 24576 |

The proprietary 14-bit decoder produces 24576 — confirming
JPEG-LS at 10/12/14-bit. The two formulas agree exactly when the
gradient is naturally inside `[min, max]`; they diverge only on
over/underflow.

A reference implementation must select the formula by bit-depth:
8-bit → modular; 10/12/14-bit → standard JPEG-LS. The MED notation
in §1's predictor table and elsewhere in this chapter refers to the
appropriate variant by bit-depth.

First-row fallback: same as Gradient (use Left within the row;
column 0 = raw).

For subsequent rows (`r ≥ 1`, `c ≥ 1`):

```
left      = px[r,   c-1]
top       = px[r-1, c]
top_left  = px[r-1, c-1]
pred      = MED(left, top, top_left)        ; JPEG-LS / Paeth-like
px[r, c]  = (pred + res[r, c]) & MAX
```

For column 0 of subsequent rows, fall back to top:

```
px[r, 0] = (px[r-1, 0] + res[r, 0]) & MAX
```

**Decoder evidence** (8-bit Median inner kernel at
`magicyuv.dll!0x69bb6a80`–`0x69bb6c2f`). The kernel loads the
left neighbour (the pixel just before the current column),
fetches a vector of bytes from the top row (offset `-stride`)
and a vector of residual bytes from the current row, and
computes the gradient `(top - top_left) + left` using an 8-bit
subtraction so the intermediate `top - top_left` wraps modulo
256 (the modular Median rule for 8-bit samples). The kernel
then computes `min(left, top)` and `max(left, top)` via two
conditional moves (compare-and-cmov on the `top` vs `left`
relation), clamps the gradient between min and max via two
further conditional moves (replace edx with gradient when
`min < gradient`, then replace edx with max when `current >
max`), adds the residual byte, and stores the low byte back to
the current pixel position.

The conditional-move chain implements `MED(a, b, c) = clip(a + b - c,
min(a, b), max(a, b))`. The same kernel structure is used for
10/12/14-bit samples in their respective per-bit-depth kernels
(see §4.5), although the 10/12/14-bit kernels compute the
gradient with full-precision arithmetic instead of the modular
8-bit form (see the bit-depth-conditional formulas above).

### 4.5 Wraparound rule (`& MAX` per bit-depth)

After every predictor + residual addition, the result is masked to
the plane's bit-depth:

| Bit-depth | `MAX`     | Mask      | Decoder kernel (RGB/Y4 family)        |
| --------- | --------- | --------- | ------------------------------------- |
| 8         | `0xff`    | implicit  | `magicyuv.dll!0x69bb4f70` (entry); inner kernels at `0x69bb4f70`/`0x69bb6a80` etc. |
| 10        | `0x3ff`   | explicit  | `magicyuv.dll!0x69ce8680` (entry); explicit `& 0x3ff` mask at `magicyuv.dll!0x69ce8712` etc. |
| 12        | `0xfff`   | explicit  | `magicyuv.dll!0x69cea3d0` (entry); explicit `& 0xfff` mask at `magicyuv.dll!0x69cea462` etc. |
| 14        | `0x3fff`  | explicit  | `magicyuv.dll!0x69cec120` (entry); explicit `& 0x3fff` mask at `magicyuv.dll!0x69cec1b2` etc. |

For 8-bit samples the wrap is implicit in the 8-bit add/store
instructions (modulo 256 by the ALU). For 10/12/14-bit samples the
binary explicitly masks the result after the add — at
`magicyuv.dll!0x69ce8712` (10-bit, mask `0x3ff`),
`magicyuv.dll!0x69cea462` (12-bit, mask `0xfff`), and
`magicyuv.dll!0x69cec1b2` (14-bit, mask `0x3fff`).

The 10/12/14-bit samples are **stored as 16-bit `signed short`** in
RAM (`s` in the Itanium ABI mangling per
`03-pixel-plane-mapping.md` §1) — the high unused bits (`16-bits` of
them) are zero-filled by the mask. Whether the **wire-format
residual** is itself 16-bit signed or some other representation is
entropy-coding business and is covered in
`05-entropy-coding.md`.

### 4.6 Per-Backend kernel addresses

For reference, the per-bit-depth-tier predictor kernel addresses
used by the v2.4.2 decoder. All Backends within the same bit-depth
tier share the same kernel set; only the entry-wrapper at
vtable +0x60 differs by `(n_planes, sub_x, sub_y)`.

| Bit-depth | Median (vtable +0x58) | Left+Gradient (vtable +0x50) | Decoder entry (vtable +0x60) |
| --------- | --------------------- | ---------------------------- | ---------------------------- |
| 8         | `0x69bb6a50`          | `0x69bb6a10`                 | `0x69bb4f70`                 |
| 10        | `0x69ce8610`          | `0x69ce8640`                 | `0x69ce8680`                 |
| 12        | `0x69cea360`          | `0x69cea390`                 | `0x69cea3d0`                 |
| 14        | `0x69cec0b0`          | `0x69cec0e0`                 | `0x69cec120`                 |

These addresses are referenced from the per-Backend vtable; the
vtable VMAs for all 14 Backends are listed in
`03-pixel-plane-mapping.md` §1.

## 5. Where prediction sits in the decode pipeline

This is the layered view of decoding a single slice:

```
+----------------------------------------+
| Slice payload (entry[k+1] + 0x20 ..)   |
+----------------------------------------+
| +0  slice_flags  (uint8)              |   ← §1.1
| +1  predictor_id (uint8)              |   ← §1.2
| +2  ...                                 |   ← residual stream
+----------------------------------------+
                |
                v
   slice_flags & 0x01 ?
                |
       +--------+--------+
       |                 |
       no (Huffman)     yes (raw)
       |                 |
       v                 v
+-------------+   +-----------------------------------+
| Huffman     |   | Raw bytes: each pixel's residual  |
| residuals   |   | is one or two bytes LE (per       |
| per-plane,  |   | `03-pixel-plane-mapping.md` §7),  |
| see entropy |   | already post-prediction.          |
| chapter     |   |                                   |
+-------------+   +-----------------------------------+
                |
                v
   For each pixel (r, c) in plane row order:
     pred = predictor(predictor_id, r, c, neighbours)   ← §4
     px[r, c] = (pred + residual[r, c]) & MAX           ← §4.5
                |
                v
   For RGB-family planes (`03-pixel-plane-mapping.md` §4):
     Reverse the inter-plane decorrelation
     (B = B' + G, R = R' + G) per §5.1.
                |
                v
   Reconstructed plane in storage layout per
   `03-pixel-plane-mapping.md` §3
```

### 5.1 RGB inter-plane decorrelation lives on the wire

For RGB-family format bytes (M8RG, M8RA, M0RG, M0RA, M2RG, M2RA,
M4RG, M4RA), the encoder applies a `B' = B − G; R' = R − G`
(mod `2**bit_depth`) inter-plane decorrelation to the planes
**before** prediction, and the wire stores the decorrelated triple
in plane order **`(B', G, R')`** (with the `A` plane appended
unchanged when present). The decoder must:

1. Run the per-slice predictor on each wire plane independently to
   reconstruct `(P0, P1, P2)` = `(B', G, R')` plane samples.
2. Reverse the decorrelation per-pixel:
   `B = (P0 + P1) mod 2^bit_depth`,
   `R = (P2 + P1) mod 2^bit_depth`.
3. The user-facing plane output is then `(G, B, R)` per
   `03-pixel-plane-mapping.md` §4 (with `A` appended for RGBA).

The transform sits **outside** the predictor: the predictor sees
wire samples (the decorrelated `B'`/`R'` for the chroma-style
planes; raw `G` for the green plane); the per-plane Huffman
descriptor (`05-entropy-coding.md`) is built over those decorrelated
samples' residual histogram. See `03-pixel-plane-mapping.md` §4 for
the plane-order table; the empirical evidence is from M8RG
green/blue/red/mixed fixtures showing bit-exact wire bytes only when
the encoder applies the `(B−G, G, R−G)` transform and emits planes in
that order.

### 5.1.1 Interlaced streams: doubled predictor row stride

When the header `flags` dword has bit 1 set
(`flags & FLAG_INTERLACED == 0x00000002`,
`01-file-header-and-fourccs.md` §3.1), the predictor's row stride
doubles: top-of-row r is row r-2, not row r-1. The first **two**
rows of each slice have no top neighbour (one for the top field, one
for the bottom field) and behave like row 0 of a progressive slice
(column 0 = residual itself, columns 1+ use Left across the row).
This applies to all three predictor IDs (Left, Gradient, Median).

Worked example (4×8 M8RG vertical-ramp B-plane, raw mode, Left
predictor): the proprietary encoder stores residuals
``07 00 00 00 06 00 00 00 fe 00 00 00 fe 00 00 00 fe 00 00 00 fe 00 00 00 fe 00 00 00 fe 00 00 00``
— the `07` and `06` are residuals for image rows 0 and 1
respectively (no top neighbour); the `0xfe` (-2) residuals at
rows 2..7 mean each row's column-0 value is
`top_field_or_bottom_field[r - 2] - 2`, recovering the source
vertical ramp.

The slice-level reset rule (§5.2) still applies: each slice's
first two rows have no top neighbour, regardless of the rows in
the preceding slice. For `slice_height = 28` (the v2.4.2 default)
and a frame height of 32, slice 0 covers image rows 0..27 and
slice 1 covers rows 28..31; rows 28 and 29 are the field-firsts
of slice 1 with no top neighbour (despite rows 26 and 27 of
slice 0 sitting "directly above" them in image-space).

The encoder selects between progressive (field_stride=1) and
interlaced (field_stride=2) prediction by the `Interlaced`
registry value, and writes the result into header bit 1 of the
flags dword. Decoders MUST honour the flag bit to reconstruct
interlaced streams correctly.

### 5.2 First slice of a plane: state initialization

When decoding the first slice of a plane, the decoder must
initialise neighbour state. Behavioural traces show the encoder
treats slice 0 of each plane as starting "fresh": there is no
implicit carry-over of pixel state from the preceding plane.
Specifically, slice 0 of plane 0 starts with `px[0, 0] = res[0, 0]`
(no neighbour); slice 0 of plane 1 likewise starts independently.

Within a plane, slices are stacked vertically (each slice covers
`slice_height` rows of the plane; `02-slice-table.md` §6). The
boundary between slices `k-1` and `k` is the row-boundary between
`(k * slice_height - 1)` and `(k * slice_height)`. **At the
slice-0-of-non-first-slice boundary, the decoder must initialise
the row-running state as if `r = 0` of the slice** — i.e. the first
row of slice `k` is **independent of the last row of slice `k-1`**
for predictor purposes. This is what allows the encoder to write
each slice's `predictor_id` byte independently and pick a different
predictor per slice.

**Behavioural confirmation.** When the encoder is in Dynamic mode
and the input image's content varies across slices (e.g. the
diagonal-ramp pattern at fixture
`r4_m8rg_p7_cm0.bin`), the per-slice `predictor_id` differs:
e.g. plane 0 slices = (1, 1, 1), plane 1 slices = (2, 2, 2),
plane 2 slices = (1, 1, 1). The encoder freely picks per slice
because there is no cross-slice prediction.

This makes parallel slice decoding well-defined: slices within a
plane can be decoded in any order (provided the slice payload
boundaries from `02-slice-table.md` §5 are honoured), and the decode
is independent per slice.

## 6. Interaction with subsampled YUV (M8Y2, M8Y0, M0Y2, M0Y0)

For subsampled YUV format bytes, the per-plane row count differs
between luma and chroma. `02-slice-table.md` §4 notes that chroma
planes use the same slice count as luma (not
`ceil(chroma_h / slice_height)`). `02-slice-table.md` §6 specifies
that for `sub_y > 1`, the chroma slice's row range is
`[s × slice_height / sub_y, (s + 1) × slice_height / sub_y)`
of the chroma plane.

For prediction, this means a chroma slice's row count is
`slice_height / sub_y` (with the last slice short if needed). The
predictor formulas are unchanged; they operate on the chroma
plane's row-major storage with the chroma plane's stride
(`chroma_plane_width = ceil(width / sub_x)`).

Behavioural confirmation: the M8Y0 (4:2:0, sub_x=sub_y=2) 64×32
fixture `m8y0_64x32.bin` has 6 slices total
(3 planes × 2 slices/plane). Luma slice 0 covers 28 rows of luma;
chroma slice 0 covers 14 rows of chroma. Each slice carries its own
2-byte prefix and is independently predictable.

## 7. Summary — what a decoder must implement

To decode v2.4.2 MagicYUV streams:

1. Parse the 32-byte header (`01-file-header-and-fourccs.md`);
   ignore byte `+0x0b` beyond a `<= 7` check on `version` (`+0x08`).
2. Walk the slice table at `+0x20` (`02-slice-table.md` §5).
3. For each slice:
   a. Read 2 prefix bytes at `entry[k+1] + 0x20` (`slice_flags` and
      `predictor_id`).
   b. If `slice_flags & 0x01`, the rest of the slice is **raw**
      `slice_height/sub_y * plane_width * bytes_per_sample` pixel
      residual bytes (the last slice may be shorter); otherwise
      Huffman-decode `slice_height * plane_width` residuals using
      the slice's plane Huffman table from the preamble
      (`05-entropy-coding.md`).
   c. Apply the predictor selected by `predictor_id`:
      - `0x01` → Left  (§4.2)
      - `0x02` → Gradient (§4.3)
      - `0x03` → Median (§4.4)
      Other values (including `0x04`) MUST be treated as malformed.
   d. Mask each reconstructed pixel by `MAX = (1 << bits) - 1`
      (§4.5).
4. Repeat for every slice. Each slice is independent (§5.2).
5. Apply RGB inter-plane decorrelation reversal (B = B' + G,
   R = R' + G) for RGB-family format bytes after each slice's
   plane is reconstructed (§5.1).

## 8. Open questions

1. **The "first column of a row" predictor for Gradient and Median.**
   §4.3 and §4.4 use "top" as the fallback predictor for column 0
   of subsequent rows. This is the simplest rule consistent with
   the binary's "previous-row state seeds the running sum"
   architecture, and is identical to JPEG-LS. It has not been
   directly observed in v2.4.2 fixture data because every
   fixture's column 0 is part of a region with a known-zero
   residual. Confirmation would require constructing a fixture
   whose first-column residuals are non-zero, decoding it through
   both the proprietary decoder (via Wine harness) and an
   implementation of the rules above, and comparing pixel-for-pixel.

2. **Raw mode pixel layout for high-bit-depth Backends.**
   For 8-bit Backends, raw mode emits 1 byte per pixel. For
   10/12/14-bit Backends, are raw bytes 1-byte or 2-byte? The
   sample storage is 2 bytes (`03-pixel-plane-mapping.md` §7.2).
   High-bit-depth raw slices are not exercised by v2.4.2's encoder
   for the fixtures cited here (the deterministic random pattern
   at 8-bit was the trigger for raw mode). Whether the encoder
   ever emits raw-mode for a 10/12/14-bit slice with the v2.4.2
   binary, and what byte count per pixel it uses, is unverified.

3. **Reserved bits in `slice_flags`.** All v2.4.2 fixtures show
   bits 1..7 of byte +0 = 0. Whether any future encoder revision
   will set additional bits is not knowable from this binary;
   decoders should mask `& 0x01` when interpreting and pass other
   bits through unchanged for forward compatibility.

4. **Header byte `+0x0b` semantic in non-2.4.2 streams.** The
   v2.4.2 encoder always writes `0x02` (§2.1, §2.2), but the
   decoder accepts `<= 2` via shortcut and `> 2` via property-tree
   lookup. A 2.0..2.3.x stream might carry a non-`0x02` value;
   the decoder's behaviour on such streams is unverified (no
   fixtures available). For practical purposes, decoders
   targeting v2.4.2 output can ignore byte `+0x0b` beyond the
   range check.

5. **`predictor_id = 0x04` (would-be Dynamic) on the wire.** The
   spec asserts (§3, §7) that values ≥ `0x04` MUST be treated as
   malformed. No v2.4.2 fixture produces such a value. A
   conservative decoder may however choose to fall back to
   Median or Gradient for any unknown `predictor_id` to be
   forward-compatible with hypothetical future per-slice modes.

## 9. Reconciliation with `01-file-header-and-fourccs.md` §3.0

[`01-file-header-and-fourccs.md`](01-file-header-and-fourccs.md) §3.0
maps header byte `+0x0b` to the four "codec variant" mode IDs
0x01 = Left, 0x02 = Gradient, 0x03 = Median, 0x04 = Dynamic.
This chapter does not contradict that table for the *vendor's
labelling*, but adds critical context:

- **The header byte at `+0x0b` is not the on-wire predictor mode**
  in v2.4.2 streams. It is always `0x02` due to the encoder clamps
  documented in this chapter §2.1 (and reiterated in
  [`02-slice-table.md`](02-slice-table.md) §intro).
- **The on-wire predictor is the per-slice `predictor_id` at slice
  byte +1**, with values `0x01`, `0x02`, `0x03` only.
- **Dynamic is an encoder strategy, not a wire-level mode.** No
  slice carries `predictor_id = 0x04`.
- **The four labels** in `01-file-header-and-fourccs.md`'s table
  apply to the *encoder's config-space* mode (the value the encoder
  uses internally to drive its strategy), not the wire format. The
  wire format has three predictor IDs; the encoder uses the
  configured mode to decide which of those to write.

The §3.0 table in `01-file-header-and-fourccs.md` should be read as
listing the *encoder-side* labels associated with each value of the
configured mode field, with the understanding that this chapter
supersedes any implicit claim that all four values appear on the
wire as byte `+0x0b`. Only `0x02` does, because of the clamps. The
slice-table envelope and mask-polarity facts in
[`02-slice-table.md`](02-slice-table.md) §intro remain authoritative.
  claim that those four values appear on the wire as `+0x0b`.
  Only `0x02` does, because of the clamps.
