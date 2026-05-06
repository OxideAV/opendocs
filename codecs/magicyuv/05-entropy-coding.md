# MagicYUV v7 — Entropy coding (Huffman + residuals)

This chapter pins down the entropy stage that sits between the
prediction layer ([`04-prediction-modes.md`](04-prediction-modes.md))
and the per-plane row data inside each slice payload. It specifies:

1. The per-plane Huffman length descriptor encoded in the per-frame
   preamble (immediately after the per-slice plane-index bytes per
   [`02-slice-table.md`](02-slice-table.md) §7).
2. Canonical-Huffman code construction from those lengths. The codec
   does **NOT** use [RFC 1951 §3.2.2](https://www.rfc-editor.org/rfc/rfc1951#section-3.2.2);
   it uses a related-but-different canonical algorithm (longest-length
   first, right-shifting accumulator). See §2 for the binary-derived
   recipe.
3. The per-slice bitstream layout — bit ordering, byte ordering, end
   conditions.
4. The raw-mode fallback exact byte layout.
5. The relationship between the 1.2 (2015-08-25) change-log entry
   "Removed 'Adaptive coding' setting (now always enabled)" and the
   per-frame Huffman tables.

All claims trace to byte-level evidence in the proprietary binary
`reference/binaries/system32/magicyuv.dll` (SHA-256
`2036e284cc07a2cb4d4b6b6cb04a1d24c0f97a0041c134fd669ee079b768902c`,
PE32+ x86-64) and to behavioural-trace fixtures driven through the
Wine VFW harness.

Notation follows
[`01-file-header-and-fourccs.md`](01-file-header-and-fourccs.md):
`<file>@0xNNNNNN` is a file offset and `<file>!0xNNNNNN` is the
instruction VMA at the binary's default ImageBase `0x69b80000`.

This chapter takes as given:

- The per-frame preamble layout from
  [`02-slice-table.md`](02-slice-table.md) §7. The preamble's first
  `1 + total_slices` bytes carry `plane_count` and the per-slice
  plane-index sequence; the byte range
  `[0x20 + 4*(N+1) + 1 + total_slices, 0x20 + entry[1])` is the
  **per-plane Huffman descriptor region**.
- The per-slice 2-byte prefix from
  [`04-prediction-modes.md`](04-prediction-modes.md) §1: byte `+0` is
  `slice_flags`, byte `+1` is `predictor_id`. Bit 0 of `slice_flags`
  selects raw mode (1) or Huffman mode (0).
- The plane semantics from
  [`03-pixel-plane-mapping.md`](03-pixel-plane-mapping.md) §3..§7:
  per-Backend bit-depth (`bits ∈ {8, 10, 12, 14}`), plane count, and
  per-plane (width, height).

## 1. The per-plane Huffman descriptor region

Each plane carries one length descriptor. The descriptors are
written **plane-major in the preamble** in plane index order
(plane 0, plane 1, ..., plane (plane_count - 1)). Per
[`02-slice-table.md`](02-slice-table.md) §7.4, plane index 0 is the
first plane in the format-byte's family order
(G for RGB, Y for YUV / Gray; see `03-pixel-plane-mapping.md` §4..§6).

### 1.1 Descriptor encoding (run-length form)

Each descriptor is a sequence of bytes that decodes to **exactly**
`N = 1 << bits` code lengths, one per residual symbol. Each symbol
position `s ∈ [0, N)` has an associated code length
`L[s] ∈ [0, max_length]` where:

| Bit-depth | Symbol count `N` | `max_length` |
| --------- | ---------------: | -----------: |
| 8         | 256              | 12           |
| 10        | 1024             | 14           |
| 12        | 4096             | 16           |
| 14        | 16384            | 18           |

`max_length` is taken from the third numeric parameter of the
matching Huffman-coder specialisation in the binary at file
`0x2568f0..0x256960` (see §1.5 below). Throughout this spec we use
the project shorthand `HuffCoder<symbol_type, accumulator_type,
bit_depth, max_length, default_max_length>` to label each
specialisation by the parameter shape recovered from the binary; the
labels are ours and do not quote vendor class names. The four
specialisations are:

- 8-bit:  HuffCoder<u8, u16, 8, 12, 12>
- 10-bit: HuffCoder<u16, u32, 10, 14, 11>
- 12-bit: HuffCoder<u16, u32, 12, 16, 12>
- 14-bit: HuffCoder<u16, u32, 14, 18, 12>

The byte stream emits length values with a **two-state run-length**
scheme:

```
Loop: repeat until N lengths produced.
  Read one byte B.
  If (B & 0x80) == 0:
      emit length B          (append a single L = B to the lengths array).
  Else:
      Read one more byte C.
      Let v = B & 0x7f       (length value, in [0, max_length]).
      Emit length v repeated (1 + C) times.
End loop.
```

A literal byte appends one length; a high-bit-flagged byte is
followed by a count byte and appends `1 + C` repetitions of the
masked length. Counts are 0..255, so a single run encodes between 2
and 256 consecutive equal-length symbols. (`C = 0` ⇒ run of length
2; `C = 0xff` ⇒ run of length 256.) A run of length 1 is **always**
written as a literal — the encoder never produces a `(0x80|v, 0)`
pair (the encoder's RLE-emit path at
`magicyuv.dll!0x69b945c8`–`0x69b94830` only emits the two-byte form
when its match counter is non-zero, i.e. at least one match was
found, see §1.4).

The byte `B = 0x80` represents `length = 0` (symbol unused). Its run
form `0x80 C` writes `1 + C` zero-length symbols (= `1 + C`
unused symbols).

### 1.2 Worked example: 8-bit all-zero-residual descriptor

For an all-zero 8-bit plane (only one symbol active in the residual
distribution), the v2.4.2 encoder emits the 6-byte descriptor

```
01 89 5d 08 89 9f
```

Decoding per §1.1:

| Byte | Action                 | Lengths produced           | Cumulative count |
| ---- | ---------------------- | -------------------------- | ---------------: |
| `01` | literal length 1       | `[1]`                      | 1                |
| `89` | run, value 9 …         |                            |                  |
| `5d` | … repeat (1 + 93) = 94 | 94 × `[9]`                 | 95               |
| `08` | literal length 8       | `[8]`                      | 96               |
| `89` | run, value 9 …         |                            |                  |
| `9f` | … repeat (1 + 159) = 160 | 160 × `[9]`              | 256              |

Total: 256 lengths. **Kraft inequality check**:

```
Σ 2^-L = 1·2^-1 + 1·2^-8 + 254·2^-9
       = 0.5 + 1/256 + 254/512
       = 1.0
```

The Kraft sum is exactly 1, so this is a valid complete canonical
Huffman code for 256 symbols. The most-frequent symbol gets length 1
(code = `0` per §2 below), all other symbols get longer codes.

### 1.3 Cross-check: every fixture parses to a valid Huffman code

This was verified on every fixture cited below
(`r4_*.bin`, `m8rg_*.bin`, `m8g0_*.bin`, `m8y0_*.bin` — ≥ 30 fixtures
spanning patterns 0/1/3/4/5/6/7/8 × CompMethod 0/1/2/3/4 × FOURCCs
M8RG/M8G0/M8Y0/M8RA/M8YA): in every case, the descriptor decodes to
**exactly `N` lengths** and the Kraft sum is **exactly 1.0**. No
fixture observed produces an over-full or under-full code book.

A representative sample:

| Fixture                        | Plane | Descriptor bytes | Kraft |
| ------------------------------ | -----:| ----------------:| -----:|
| `r4_m8rg_p0_cm0` (zero) plane 0 | 0    | 6                | 1.0   |
| `r4_m8g0_p1_cm0` (h_ramp)       | 0    | 16               | 1.0   |
| `r4_m8g0_p7_cm0` (diag_ramp)    | 0    | 18               | 1.0   |
| `r4_m8rg_p4_cm0` plane 1 (B)    | 1    | 9                | 1.0   |
| `r4_m8rg_p7_cm0` plane 1        | 1    | 20               | 1.0   |
| `r4_m8y0_p3_cm0` plane 0 (Y)    | 0    | 97               | 1.0   |
| `r4_m8y0_p3_cm0` plane 1 (U)    | 1    | 124              | 1.0   |
| `r4_m8y0_p3_cm0` plane 2 (V)    | 2    | 110              | 1.0   |
| `r4_rand_cm0` plane 0           | 0    | 20               | 1.0   |

Across all fixtures the maximum length observed in 8-bit plane
descriptors is **12**, matching the `HuffCoder<u8, u16, 8, 12, 12>`
specialisation's third numeric parameter.

### 1.4 Decoder evidence: the descriptor parser

The decoder's per-plane Huffman descriptor parser is at
`magicyuv.dll!0x69baf6c0`–`0x69baf80a` (file
`@0x2f6c0`–`@0x2f80a`). The core loop:

1. Loads the next descriptor byte `B` from the input pointer
   and provisionally records `bytes_consumed = 1`.
2. Tests the high bit of `B`. If clear, treats `B` as a
   literal length value and falls through to the emit step.
3. Otherwise: bounds-checks that a second byte is available,
   loads the count byte `C`, sets `bytes_consumed = 2`, and
   masks `B` to its low 7 bits (the length value).
4. Writes the (now-decoded) length to the next output slot.
5. If a count was read (run case), enters a replication
   sub-loop that decrements `C` and writes the same length to
   subsequent output slots until the count reaches zero or
   the output buffer is full.

The loop is unrolled 4-way for performance; the key facts —
`& 0x80` test, two-byte form, `(1 + count)` repetitions of the
masked length — are unambiguous from static analysis.

The function called at the entry (just before the parser) is the
per-Backend bit-depth getter at vtable `+0x40` (per `04-prediction-modes.md` §4.6
mappings). For 8-bit Backends this is `magicyuv.dll!0x69ce85b0`,
which returns the literal value 8 to its caller. The 10-bit
getter at `magicyuv.dll!0x69ce85d0` returns 10 (`0xa`); the
12-bit getter at `magicyuv.dll!0x69cea320` returns 12 (`0xc`);
the 14-bit getter at `magicyuv.dll!0x69cec070` returns 14
(`0xe`). The decoder then computes `N = 1 << bits` at
`magicyuv.dll!0x69baf602` (the seed value `1` is loaded at
`magicyuv.dll!0x69baf5ec`).

### 1.5 Encoder evidence: the descriptor writer

The encoder's matching descriptor writer is at
`magicyuv.dll!0x69b945c8`–`0x69b9484f` (file
`@0x145c8`–`@0x1484f`). It reads the per-plane lengths (stored as
4-byte ints in a `std::vector<int>` whose data pointers live at
offsets `+0x20..+0x28` of the Huffman coder object) and emits
the run-length-encoded byte stream:

- At `magicyuv.dll!0x69b945a1` it short-circuits when the input
  has fewer than 2 entries (jumps to a tail path).
- At `magicyuv.dll!0x69b945c8` the main loop reads the next
  length value and compares it against the running "current"
  value. If they differ (`magicyuv.dll!0x69b945cf`), the loop
  branches to the literal-emit path at
  `magicyuv.dll!0x69b94830`. If they match, the run counter is
  set to 1 and the loop advances.
- At `magicyuv.dll!0x69b94600` the run count is capped at `0xfe`
  (254); when the counter exceeds the cap the loop branches to
  the run-emit path at `magicyuv.dll!0x69b94810` to flush the
  current run before continuing.
- The two-byte run-emit path at
  `magicyuv.dll!0x69b946d6`–`0x69b946f2` writes byte 0 as
  `(value | 0x80)`, writes byte 1 as the run count, and resets
  the run-count accumulator to 0.

The emit-as-literal branch at `magicyuv.dll!0x69b94830` writes a
single byte equal to the length value with no high-bit flag.

At end-of-input (run-count finalisation), the encoder writes the
remaining literal at
`magicyuv.dll!0x69b94705`–`0x69b9470b`. The encoder caps counts
at 254 to ensure the count byte fits in 8 bits with one unit of
headroom (the value `0xff` is reserved); per behavioural trace,
no fixture surveyed produced a `0xff` count byte.

The Huffman-coder class hierarchy in the binary consists of a base
class and four bit-depth specialisations, each backed by a body class
parameterised on `<symbol_type, accumulator_type, bit_depth,
max_length, default_max_length>`. We reference each by our project
shorthand `HuffCoder<...>` introduced in §1.1. The records are at:

| Layout (project shorthand)                  | File offset    |
| ------------------------------------------- | -------------- |
| Base class                                  | `@0x25a6e0`    |
| 8-bit specialisation (derived)              | `@0x2568e0`    |
| 10-bit specialisation                       | `@0x256980`    |
| 12-bit specialisation                       | `@0x256990`    |
| 14-bit specialisation                       | `@0x2569a0`    |
| `HuffCoder<u8, u16, 8, 12, 12>` (8-bit body) | `@0x2568f0`   |
| `HuffCoder<u16, u32, 10, 14, 11>`           | `@0x256920`    |
| `HuffCoder<u16, u32, 12, 16, 12>`           | `@0x256940`    |
| `HuffCoder<u16, u32, 14, 18, 12>`           | `@0x256960`    |

Vtables are at VMA `0x69ddee68`, `0x69ddeeb8`, `0x69ddef08`,
`0x69ddef58`, `0x69ddefa8`, `0x69ddeff8`, `0x69ddf048`, `0x69ddf098`
(found by reverse-lookup of typeinfo records at file
`0x252ff0..0x2530e0`; the full mapping is straightforward to
reproduce from the typeinfo records).

The shorthand parameter triple `(bits, max_length, default_max_length)`
follows the convention:

- 1st = `bits` (significant bits / log2(symbol count))
- 2nd = `max_length` (the value used by the canonical-Huffman length
  limiter — see §2)
- 3rd = a related hint or a different bit-reader chunk size; not
  used by the wire format and treated as opaque by this spec.

## 2. Canonical-Huffman code construction (length-descending cumulative)

The codec does not use RFC 1951 §3.2.2's canonical-code recipe. The
code-value assignment at `magicyuv.dll!0x69bb2190..0x69bb2790` is a
different canonical algorithm — same notion of "the codes are
determined entirely by the length array", but a different bit-level
assignment. The codec walks length tiers **longest-first**, not
shortest-first, and uses an accumulator that **right-shifts** between
tiers, not left-shifts as RFC 1951 does. The two algorithms produce
different per-symbol code bit-patterns from the same length array.

Given the length array `L[0..N)` from §1, the codec constructs the
codes by walking lengths from `max_len_used` down to `1` and
incrementally assigning codes within each length tier from a
right-shifting accumulator.

### 2.0 Algorithm (binary-derived)

Inputs:

- `L[0..N)` — per-symbol code lengths (`L[s] = 0` means "symbol
  unused").
- `max_len_used = max(L[s] for s in 0..N if L[s] > 0)`.
- `bl_count[len]` for `len in 0..max_len_used+1` —
  `bl_count[len] = #{s : L[s] == len}`.

Output: `code[s]` for each `s` with `L[s] > 0`, the bit pattern of
length `L[s]` (right-justified into a `≥ L[s]`-bit machine word). The
encoder packs each `(code[s], L[s])` MSB-first per §2.2 below.

Procedure (direct simulation of the body at
`magicyuv.dll!0x69bb2680`–`0x69bb277b`):

```
acc = 0xffffffff             # = -1 mod 2^32   (the running accumulator)
sym_counter = 0              # symbol-position counter
for len in range(max_len_used, 0, -1):
    if bl_count[len] > 0:
        # Within this length tier, assign codes to symbols in
        # ascending symbol-index order. The first code emitted is
        # acc+1, the next is acc+2, and so on, up to acc+bl_count[len].
        for i, s in enumerate(symbols_with_length(len)):
            code[s] = (acc + 1 + i) & 0xffffffff
        acc = (acc + bl_count[len]) & 0xffffffff
    # Validity check: the accumulator must not exceed the codespace
    # for the current length. (1 << len) is 2^len, the number of
    # distinct codes representable at this length.
    if (1 << len) <= acc:
        raise ValueError("malformed canonical Huffman descriptor")
    acc = acc >> 1            # logical right shift, unsigned
```

`symbols_with_length(len)` enumerates the symbols `s` with `L[s] ==
len`, in ascending `s` order. The right-shift is unconditional —
applied even when `bl_count[len] == 0`.

**Equivalent recursive formulation.** Let `start[len]` be the smallest
code value at length `len`. Then:

```
start[max_len_used]      = 0
start[len]               = ceil((start[len+1] + bl_count[len+1]) / 2)   for len < max_len_used
```

(The `+1` and `>>= 1` together implement `ceil(a/2)` when `a` is
odd; when `a` is even, both forms agree.)

### 2.0.1 Worked example (8-bit all-zero residual descriptor)

For descriptor `01 89 5d 08 89 9f` from §1.2 (lengths: `L[0] = 1`;
`L[s] = 9` for `s in [1..94] ∪ [96..255]`; `L[95] = 8`):

| `len` | `bl_count[len]` | `acc` after assignment | `(1 << len)` check | `acc` after `>>= 1` | codes assigned                                    |
| -----:|:---------------:|:----------------------:|:------------------:|:-------------------:|--------------------------------------------------:|
| 9     | 254             | -1+254 = 253           | 512 > 253 ✓        | 126                 | symbols 1..94, 96..255 ← codes 0..253             |
| 8     | 1               | 126+1 = 127            | 256 > 127 ✓        | 63                  | symbol 95 ← code 127 (`0b01111111`)               |
| 7     | 0               | 63                     | 128 > 63 ✓         | 31                  | (none)                                             |
| 6     | 0               | 31                     | 64 > 31 ✓          | 15                  | (none)                                             |
| 5     | 0               | 15                     | 32 > 15 ✓          | 7                   | (none)                                             |
| 4     | 0               | 7                      | 16 > 7 ✓           | 3                   | (none)                                             |
| 3     | 0               | 3                      | 8 > 3 ✓            | 1                   | (none)                                             |
| 2     | 0               | 1                      | 4 > 1 ✓            | 0                   | (none)                                             |
| 1     | 1               | 0+1 = 1                | 2 > 1 ✓            | 0                   | symbol 0 ← code 1 (`0b1`)                          |

Result: symbol 0 (the most-frequent symbol in an all-zero residual
plane) gets code `1` of length 1. A bitstream of all-symbol-0 codes
emits `1` repeatedly → bytes `0xff 0xff 0xff …` (MSB-first packing
per §2.2 below). **This matches the observed `m8rg_64x64_zero.bin`
slice 0 bitstream byte-for-byte** (bytes at file
`@0x66+` = `ff ff ff ff …`).

### 2.0.2 Relation to RFC 1951

Both RFC 1951 §3.2.2 and the MagicYUV algorithm produce **canonical**
codes (codes uniquely determined by the length array), but the
per-symbol bit patterns differ. The differences:

| Aspect                       | RFC 1951 §3.2.2          | MagicYUV (this codec)            |
| ---------------------------- | ------------------------ | -------------------------------- |
| Length-tier traversal        | shortest-first (`1..max`) | longest-first (`max..1`)         |
| Accumulator initial value    | `0`                      | `0xffffffff` (= `-1 mod 2^32`)    |
| Per-tier shift               | `<<= 1` (left)            | `>>= 1` (right)                   |
| Symbol order within tier     | ascending `s`            | ascending `s` (same)              |

For the worked example, the two algorithms diverge:

| Symbol | `L[s]` | RFC 1951 code | MagicYUV code |
| ------:| ------:| -------------:| --------------:|
| 0      | 1      | `0`            | `1`            |
| 1      | 9      | `100000010`    | `0`            |
| 95     | 8      | `01111111`     | `01111111` (= 127) |
| 255    | 9      | `011111111`    | `011111101` (= 253) |

A naive XOR-with-`(1<<L)-1` ("bit-inverted RFC 1951") agrees with
MagicYUV for the most-frequent shortest-length symbol but not for
the longer-length tiers. The orientation choice is load-bearing on
the per-tier offsets, and only the precise algorithm above
reproduces every observed bit pattern.

### 2.0.3 Implementation evidence (binary anchors)

The decoder's canonical-code constructor is at
`magicyuv.dll!0x69bb2190`–`0x69bb2790` (file
`@0x12190`–`@0x12790`). The function enforces a max-length bound
at `magicyuv.dll!0x69bb21a2` — it rejects when the caller-supplied
max-length value exceeds `0x1f` (31). The HuffCoder max_length
values (12, 14, 16, 18) are all well within this bound.

**Phase 1 — count `bl_count[len]`**
(`magicyuv.dll!0x69bb21ac`–`0x69bb22e3`). A stack region (256 bytes
of `bl_count` slots) is zeroed and then incremented once for each
input length value as the array is scanned. The running
max-length-seen is tracked in a register that the later phases
re-read as `max_len_used`.

**Phase 2 — build cumulative offset table**
(`magicyuv.dll!0x69bb22e5`–`0x69bb2473`). A second stack region
(another 256 bytes of `start[len]` slots) is zeroed and filled
top-down. The loop initialises `start[max_len_used] = 0` (the
write occurs at `magicyuv.dll!0x69bb2311`) and accumulates from
there: each iteration writes `start[len-1] = eax` and updates
`eax += bl_count[len-1]`. So `start[len]` ends up as the count
of all symbols at lengths **strictly greater** than `len` — i.e.
the position where length tier `len` begins in the symbol-output
buffer (sorted longest-first).

**Phase 3 — write `(symbol, length, ?)` triples to the output
buffer** (`magicyuv.dll!0x69bb24af`–`0x69bb2660`). For each
symbol index `s = 0, 1, ..., N-1`, look up `L[s]` from the input,
then look up `start[L[s]]` from the offset table, write
`(s, L[s], 0)` at `output[start[L[s]]]`, and increment
`start[L[s]]` so the next symbol of the same length goes at the
next position. Stride is 12 bytes per entry (the multiply by 12
is implemented as a base-plus-`*2` LEA folded into the
`*4`-scaled output addressing). After this phase the output
buffer is sorted **longest-length first, ascending symbol-index
second**.

**Phase 4 — assign code values**
(`magicyuv.dll!0x69bb2669`–`0x69bb277b`). This is the algorithm
of §2.0; in narrative form:

1. At `magicyuv.dll!0x69bb2669`–`0x69bb266c` the running
   accumulator is initialised to `0xffffffff` (i.e. `-1` mod
   `2^32`) and the symbol counter to 0.
2. At `magicyuv.dll!0x69bb2677` the max-length-seen value is
   tested against zero; if zero, the loop is skipped entirely.
3. The outer loop at `magicyuv.dll!0x69bb2680` walks lengths
   from `max_len_used` down to 1. For each length it loads
   `bl_count[len]` from the Phase-1 stack region.
4. When `bl_count[len] > 0`, the inner loop assigns codes to
   the `bl_count[len]` consecutive output entries starting at
   the symbol counter, writing `acc + 1 + i` into each entry's
   code field. Then the accumulator is incremented by
   `bl_count[len]` (at `magicyuv.dll!0x69bb2761`) and the
   symbol counter advances (at `magicyuv.dll!0x69bb2764`).
5. At `magicyuv.dll!0x69bb2767`–`0x69bb276f` the loop computes
   `1 << len` and rejects (jumps to the prologue's reject path
   at `magicyuv.dll!0x69bb2260`) if the accumulator is greater
   than or equal to that codespace bound.
6. At `magicyuv.dll!0x69bb2775`–`0x69bb277b` the accumulator is
   logically right-shifted by 1, the length counter is
   decremented, and the loop continues until length reaches 0.
7. On exit at `magicyuv.dll!0x69bb2781` the function returns
   success.

The compare-and-branch at step 5 uses an unsigned comparison
("less-than-or-equal") between `(1 << len)` and the accumulator;
the right-shift at step 6 is logical, so the accumulator is
treated as unsigned and stays non-negative.

The output buffer is a `std::vector` of 12-byte Huffman code-entry
structs; the wire format does not see the internal layout. Only the
reconstructed `(symbol, length, code)` triples reach the decoder's
flat-table lookup.

### 2.1 Validity of the parsed lengths

The decoder verifies that the parsed lengths array yields a valid
canonical code via the in-loop check at
`magicyuv.dll!0x69bb276c`–`0x69bb276f`: if
`(1 << len) <= acc` at any length tier, the function jumps to
`magicyuv.dll!0x69bb2260` (the reject path at the function's
prologue ladder), returns 0, and the caller propagates decoder
error code `-3` (the constant `0xfffffffd` is loaded into the
return register at `magicyuv.dll!0x69baece0`). So the decoder
rejects malformed (over-full) Huffman descriptors.

The behaviour for **under-full** code books (Σ 2^-L < 1) is not
exercised by the v2.4.2 encoder (every fixture has Kraft = 1.0
exactly per §1.3) and is left as
[open question 1](#10-open-questions) for future
investigation. (Mechanically, an under-full descriptor leaves
`acc != 0` after the final right-shift at len=1; the binary does
not check this directly — the constructor returns success — but
the resulting code book has unused codespace and the decoder's
flat lookup table will have invalid-symbol entries that may or
may not be hit depending on the residual stream.)

### 2.2 Bit ordering

The codec serialises each symbol's code **MSB-first** (code's most
significant bit emitted first). The encoder kernel at
`magicyuv.dll!0x69d44ff0` (HuffCoder8 vtable +0x10, idx 2) packs
codes into a 64-bit accumulator (variable-shift-left of the
incoming code, OR'd against the running accumulator) and emits
8-byte chunks via a byte-swap-on-store at
`magicyuv.dll!0x69d4506b` (the store byte-reverses to
**big-endian** byte order). The decoder kernel at
`magicyuv.dll!0x69d44bc0` reads bytes one-at-a-time into the
accumulator (variable shift on the incoming byte, OR'd into the
running accumulator) at `magicyuv.dll!0x69d44d10`; the shift
amount decreases as bits are consumed, equivalent to
left-aligned MSB-first parsing.

So the bitstream within a slice is **MSB-first** within each byte:
the first symbol's code's MSB is at bit 7 of slice byte +2, then bit
6, …, bit 0; then bit 7 of slice byte +3, etc. This is the same
convention as DEFLATE's literal/length codes (RFC 1951 §3.1.1) and
JPEG (ITU-T T.81 §F.1.2.1.3), which is the standard MSB-first
big-endian bitstream.

## 3. Per-slice residual bitstream layout

### 3.1 Slice payload header recap

Per [`04-prediction-modes.md`](04-prediction-modes.md) §1, every
slice begins with a 2-byte prefix:

| Slice byte | Field         | Description                                |
| ----------:| ------------- | ------------------------------------------ |
|   +0       | `slice_flags` | bit 0 = raw mode (1) / Huffman mode (0)    |
|   +1       | `predictor_id`| `0x01` Left, `0x02` Gradient, `0x03` Median |

The bytes that follow (slice byte +2 onward) are:

- **Huffman mode (`slice_flags & 0x01 == 0`):** an MSB-first
  bitstream of canonical-Huffman codes.
- **Raw mode (`slice_flags & 0x01 == 1`):** uncompressed residual
  bytes (§4).

### 3.2 Huffman mode bitstream order

The residual symbols within a slice are emitted in **plane row-major
order**: first row 0 column 0, then row 0 column 1, …, then row 1
column 0, etc. Each symbol is one residual value (the difference
between the actual pixel and the predicted pixel modulo `MAX + 1`,
per [`04-prediction-modes.md`](04-prediction-modes.md) §4).

The slice covers `slice_height` rows of the plane (or fewer for the
last slice; per `02-slice-table.md` §6); the row width is the
per-plane width (`width / sub_x` for chroma planes per
`03-pixel-plane-mapping.md` §8). For 8-bit
samples the symbol alphabet is `[0, 256)`, so the residual is itself
an 8-bit byte. For 10/12/14-bit samples the symbol alphabet is
`[0, 1024)` / `[0, 4096)` / `[0, 16384)`. Residuals are not
sign-extended — they are stored as **unsigned** values modulo
`2^bits`, so a "negative" residual `-1` is encoded as
`(1 << bits) - 1`. (The wraparound rule from
`04-prediction-modes.md` §4.5 ensures
this representation is the natural arithmetic.)

### 3.3 Bit accumulator and end-of-slice

The decoder reads the slice payload starting at slice byte +2,
shifting bytes into a 64-bit MSB-aligned accumulator. After each
symbol is decoded, the corresponding number of bits is consumed and
the accumulator is refilled.

There is **no explicit end-of-stream symbol** in the alphabet — the
decoder knows the symbol count from the plane geometry
(`slice_height_for_this_slice × plane_width` = total residual count
for a slice). Once all residuals are produced, any remaining bits in
the slice payload are ignored. (The encoder packs the bitstream into
whole bytes; trailing bits in the last partial byte are zero-filled
or undefined by the encoder, but the decoder MUST NOT read past the
required residual count.)

### 3.4 Slice payload size relations

For a Huffman-mode slice with row count `h_slice`, plane width
`w`, and bit-depth `bits`:

```
huffman_bytes_per_slice = 2 (prefix) + ceil(Σ codes_emitted / 8)
```

where `Σ codes_emitted` is the total bits required to encode
`h_slice * w` residual symbols using their canonical-Huffman codes.
For all-zero residuals (most-frequent symbol with code length 1),
`Σ = h_slice * w * 1`, e.g. for slice_height=28, w=64, 8-bit:
`Σ = 28 * 64 = 1792` bits = 224 bytes; total slice = 226 bytes.
This matches the observed slice size for `r4_m8rg_p0_cm0`'s plane 0
slices 0 and 1 (per
[`02-slice-table.md`](02-slice-table.md) §5.2).

### 3.5 Even-byte padding

Per [`02-slice-table.md`](02-slice-table.md) §8, every v2.4.2 frame
has total byte count even. The padding byte (0x00 when needed) is
written into the trailing bytes of the **last slice**. The decoder
MUST stop reading slice content when it has decoded the expected
number of residuals; the trailing pad bytes are not part of the
bitstream.

## 4. Raw mode (slice_flags bit 0 = 1)

When the encoder cannot Huffman-compress the slice into fewer bytes
than the raw representation, it falls back to raw mode. The encoder
threshold is at `magicyuv.dll!0x69b9551d`–`0x69b955f5`. The
sequence:

1. Reads the encoded Huffman byte count for this slice from the
   per-slice metadata at offset `+0x250` of the slice context
   (load at `magicyuv.dll!0x69b95521`).
2. At `magicyuv.dll!0x69b95527`–`0x69b95538` it computes the raw
   byte count as `(total_pixels * bits + 7) / 8` — multiplying
   total pixel count by the Backend's bit-depth (returned by
   vtable `+0x40`), adjusting for negative-zero rounding, and
   shifting right by 3 (`/ 8`).
3. At `magicyuv.dll!0x69b9553f`–`0x69b95542` it compares the raw
   byte count against the Huffman byte count and branches to
   keep Huffman when raw is greater than or equal.
4. The raw fallback path begins at `magicyuv.dll!0x69b95560` and
   ends with `magicyuv.dll!0x69b955f5`, which sets bit 0 of the
   per-slice flags field at offset `+0x264` (i.e.
   `slice_flags |= 0x01`).

So **raw mode is selected per-slice when the raw representation is
strictly smaller than the Huffman representation**. This is the
"always-on adaptive coding" mechanism (see §6).

### 4.1 Raw mode byte layout

In raw mode, the slice payload is:

```
+-----+------+-----------+
| +0  |  +1  |  +2..end  |
+-----+------+-----------+
| F   |  P   | residuals |
+-----+------+-----------+

F = slice_flags  (bit 0 = 1)
P = predictor_id (0x01, 0x02, 0x03)
residuals = uncompressed per-pixel residual values, in plane row-major
```

For 8-bit Backends (`bits = 8`), each residual is **one byte**. For
10/12/14-bit Backends (`bits ∈ {10, 12, 14}`), each residual is
**packed into a `bits`-bit field** within a continuous bit-stream
identical in bit ordering to the Huffman bitstream (§2.2 / §3.2):
MSB-first, big-endian byte ordering. The encoder's threshold formula
`(total_pixels * bits + 7) / 8` confirms this packing exactly.

**Behavioural confirmation.** The
`m8rg_64x64_rand.bin` fixture (a 64×64 deterministic pseudo-random
M8RG frame, 8-bit) produces 9 slices all with `slice_flags = 0x01`.
The slice sizes are `(1794, 1794, 514, 1794, 1794, 514, 1794, 1794,
529)` — for `slice_height = 28`, `w = 64`, `bits = 8`:

- Full slices: `28 * 64 * 8 / 8 = 1792 bytes + 2 prefix = 1794`. ✓
- Last slice: `8 * 64 * 8 / 8 = 512 bytes + 2 prefix = 514` (slices
  2, 5). ✓
- Last-of-frame: 529 = 514 + 15 bytes of even-byte trailing padding
  (per [`02-slice-table.md`](02-slice-table.md) §8) — the bitstream
  itself is still 512 bytes.

For 10/12/14-bit raw slices, the formulas predict:

| `bits` | `slice_height = 28`, `w = 64` raw size | last-slice (8 rows) raw size |
| ------ | --------------------------------------: | ---------------------------: |
| 8      | 28·64·8/8 + 2 = 1794                    | 8·64·8/8 + 2 = 514           |
| 10     | 28·64·10/8 + 2 = 2242                   | 8·64·10/8 + 2 = 642          |
| 12     | 28·64·12/8 + 2 = 2690                   | 8·64·12/8 + 2 = 770          |
| 14     | 28·64·14/8 + 2 = 3138                   | 8·64·14/8 + 2 = 898          |

The 10/12/14-bit cases are **not behaviourally exercised** by the
fixtures cited here — see
[open question 2](#10-open-questions).

### 4.2 Raw residuals are post-prediction

The raw bytes/bits are **post-prediction residuals**, not raw pixel
values. The slice's `predictor_id` byte still applies: the decoder
reads each `bits`-bit residual, then adds the predictor's prediction
to reconstruct the pixel (per `04-prediction-modes.md` §4 and §5).
This is why even
raw mode carries the `predictor_id` byte — the decoder needs it to
reconstruct.

Evidence: the encoder's pre-Huffman pipeline applies prediction
**before** the Huffman/raw-fallback split. The call at
`magicyuv.dll!0x69b954ac` (which targets the predictor
evaluator/applier at `magicyuv.dll!0x69bb1b20`) runs **before**
the per-slice byte-count decision at
`magicyuv.dll!0x69b95542`. The residual buffer that gets fed to
either the Huffman-encode call (a virtual call through vtable
`+0x10` at `magicyuv.dll!0x69b954ec`) or to the raw-mode emit
call (a virtual call through vtable `+0x20` at
`magicyuv.dll!0x69b955dd`) is the **same buffer** — the
post-prediction residuals. Raw mode just bypasses the Huffman
compression of those residuals.

## 5. Decoder pipeline summary

For each slice in the slice table:

```
1. Read 2-byte prefix (`04-prediction-modes.md` §1).
2. If slice_flags & 0x01:
       Read raw residuals: total_pixels * bits bits, MSB-first.
   Else:
       Initialise canonical-Huffman decoder for this plane (using
       per-plane lengths parsed from the preamble per §1, codes built
       per §2).
       Read total_pixels Huffman-coded residuals from slice byte +2.
3. For each residual in plane row-major order:
       Compute prediction (Left / Gradient / Median per predictor_id;
       `04-prediction-modes.md` §4).
       reconstructed_pixel[r,c] = (prediction + residual) & ((1<<bits)-1)
4. Continue until all rows of all planes are reconstructed.
```

The Huffman decoder state is **independent per slice**: each slice
has its own `predictor_id` and reuses the per-plane Huffman table
(set up once per plane from the preamble). There is no per-slice
Huffman table — only a per-plane one. (See §6 for the
adaptive-coding interpretation.)

## 6. The "Adaptive coding" change-log entry

The vendor change log entry for v1.2 (2015-08-25) reads:

> Removed "Adaptive coding" setting (now always enabled).

In the v2.4.2 wire format, "adaptive coding" manifests as **two
distinct mechanisms**:

### 6.1 Per-frame Huffman tables (per-plane)

Every frame carries its own per-plane Huffman length descriptor
(§1). The encoder builds the Huffman code from the residual histogram
**of the current frame**, plane-by-plane. Different frames have
different residual distributions (depending on content), so the
Huffman codes vary frame-to-frame.

Behavioural evidence: the descriptor bytes vary per pattern even at
identical FOURCC × CompMethod (fixtures
`r4_m8rg_p0..p8_cm0`), e.g.:

```
pattern 0 (zero):       01 89 5d 08 89 9f                    (6 B)
pattern 4 (h_ramp p1):  01 02 8a 4e 09 0a 09 8a ab           (9 B)
pattern 7 (diag p1):    01 03 8c 04 06 8c 1a 06 8c 1a 05 8c 10 0b 0c 8b 04 8c a6 02 (20 B)
```

This per-frame adaptation is the always-on form: there is no
"non-adaptive" mode that would emit a fixed table.

### 6.2 Per-slice raw fallback

Per §4, every slice independently selects raw mode (`slice_flags &
0x01 = 1`) or Huffman mode based on which produces fewer bytes. The
encoder threshold is purely byte-count-based and applies on a
per-slice basis. This is the second always-on adaptive mechanism.

The pre-1.2 "Adaptive coding" toggle, judging by the position in the
change log (between performance fixes and a `Improved encoding speed
by 10–20 %` line) and the fact that v1.2 was released six months
after v1.1 (which mentioned "I444 support" and "odd resolutions
fixes"), most plausibly enabled the per-slice raw fallback (the
per-frame Huffman table itself would always have been required —
without it, the bitstream would not be self-contained). Either way,
in v2 / v7-format streams **both** mechanisms are unconditional
properties of the wire format.

### 6.3 No across-frame Huffman state

The decoder **does not retain Huffman tables across frames**. Each
frame's preamble carries the complete per-plane length descriptors;
no frame inherits state from a previous one. This is consistent with
the **lossless intra-only** nature of the codec — every frame is a
key-frame and can be decoded standalone.

## 7. End-to-end worked example

Combining the slice-table envelope (`02-slice-table.md`), the
prediction layer (`04-prediction-modes.md`), and this chapter's
entropy stage, the decode of fixture
`m8rg_64x64_zero.bin` proceeds as:

1. Parse 32-byte header (`01-file-header-and-fourccs.md`):
   `format_byte = 0x65 (M8RG)`, `width = 64`, `height = 64`,
   `slice_height = 28`. Backend is `<u8, 8, 3, 1, 1>`
   (`03-pixel-plane-mapping.md` §1). `num_planes = 3` (G, B, R).
2. `slices_per_plane = ceil(64/28) = 3`. Total slices = 9.
3. Slice table at offset 0x20: 10 little-endian uint32 entries
   `[0x44, 0x44, 0x126, 0x208, 0x24a, 0x32c, 0x40e, 0x450, 0x532,
   0x614]`.
4. Preamble at file offset `0x48..0x63` (28 bytes):
   - `0x48`: `03` = plane_count
   - `0x49..0x51`: `00 00 00 01 01 01 02 02 02` = per-slice plane
     index (plane-major)
   - `0x52..0x57`: `01 89 5d 08 89 9f` = plane 0 (G) Huffman
     descriptor → 256 lengths: 1×len-1, 1×len-8, 254×len-9.
   - `0x58..0x5d`: `01 89 5d 08 89 9f` = plane 1 (B) descriptor.
   - `0x5e..0x63`: `01 89 5d 08 89 9f` = plane 2 (R) descriptor.
5. Build canonical-Huffman codes from each plane's lengths per the
   §2 algorithm (longest-length-first cumulative; **not** RFC 1951).
   For plane 0/1/2, symbol 0 (length 1) gets code `1`, symbol 95
   (length 8) gets code `127`, and symbols 1..94, 96..255 (length 9)
   get codes `0..253` in symbol-index order. (See §2.0.1 for the
   per-tier table.)
6. Slice 0 (plane 0, rows 0..27) starts at file offset
   `0x44 + 0x20 = 0x64`. Read 2-byte prefix: `slice_flags = 0x00`,
   `predictor_id = 0x01` (Left). Huffman bitstream begins at file
   offset `0x66`. The first slice has 226 bytes → 224 bytes of
   bitstream → 1792 bits. With the most-frequent symbol (symbol 0)
   having length-1 code `1`, expect 28×64 = 1792 zero residuals
   decoded → all-`1` bits → bytes `ff ff ff …`. Verified against
   the actual fixture bytes at `@0x66+`.

### 7.1 Why all-zero residual encodes as `0xff` bytes

The `m8rg_64x64_zero.bin` slice 0 bitstream begins
`ff ff ff ff ff ff ...`. Under the §2
algorithm (worked example in §2.0.1), symbol 0 (the most-frequent
symbol in an all-zero residual plane) gets length-1 code `1` —
emitting `1` bits → bytes `0xff 0xff 0xff …`. Match.

If the codec had used standard RFC 1951 §3.2.2 instead, symbol 0
would get code `0` of length 1 (the smallest length-1 code under
that recipe), and an all-symbol-0 bitstream would emit `00 00 00 …`
bytes. The observed all-`1` bytes rule out RFC 1951; the §2
algorithm reproduces the observation exactly.

The decoder kernel at `magicyuv.dll!0x69d44bc0` reads bits MSB-first
and walks a flat lookup table populated from the
`(symbol, length, code)` triples produced in Phase 4 of §2.0.3
above. The flat-table lookup uses the top `max_length` bits of the
bit accumulator to index directly into a preallocated decoder table.

## 8. Summary — what a decoder must implement

To fully decode v2.4.2 MagicYUV slice payloads:

1. Parse the 32-byte header (`01-file-header-and-fourccs.md`).
2. Walk the slice table at offset 0x20
   (`02-slice-table.md` §5).
3. Parse the preamble plane_count + per_slice_plane_index bytes
   (`02-slice-table.md` §7).
4. For each plane `p ∈ [0, plane_count)`:
   a. Parse the plane's Huffman descriptor per §1 → `N` length values
      where `N = 1 << bits`.
   b. Construct canonical-Huffman codes per §2 (the
      longest-length-first cumulative algorithm at
      `magicyuv.dll!0x69bb2190..0x69bb2790`, **not** RFC 1951
      §3.2.2 — see §2.0 procedure and §2.0.2 contrast table).
5. For each slice `s ∈ [0, total_slices)`:
   a. Locate the slice payload at `entry[s+1] + 0x20`
      (`02-slice-table.md` §5).
   b. Read 2-byte prefix `(slice_flags, predictor_id)`
      (`04-prediction-modes.md` §1).
   c. If `slice_flags & 0x01`:
      Read `slice_pixels * bits` bits MSB-first (§4.1) for the
      residual stream.
      Else:
      Decode `slice_pixels` Huffman-coded residuals MSB-first using
      the per-plane Huffman code (§3).
   d. Apply the predictor `predictor_id`
      (`04-prediction-modes.md` §4) plane-row-major
      to reconstruct pixels. Mask each result by `(1 << bits) - 1`.
6. Reverse RGB inter-plane decorrelation if format is RGB family
   (`04-prediction-modes.md` §5.1).

`slice_pixels = slice_height_for_this_slice * plane_width`, computed
per `02-slice-table.md` §6 and `03-pixel-plane-mapping.md` §8.

## 9. Cross-reference index for implementers

| Topic                                | Source             |
| ------------------------------------ | ------------------ |
| Header layout (32 B)                 | `01-file-header-and-fourccs.md` §2..§3 |
| FOURCC enumeration                   | `01-file-header-and-fourccs.md` §4 |
| Slice table layout                   | `02-slice-table.md` §5 |
| Preamble plane_count + per_slice_idx | `02-slice-table.md` §7 |
| Plane order (G/B/R or Y/U/V)         | `03-pixel-plane-mapping.md` §4..§6 |
| Per-plane bit-depth                  | `03-pixel-plane-mapping.md` §3 / §7 |
| Per-slice prefix bytes               | `04-prediction-modes.md` §1 |
| Predictor formulas                   | `04-prediction-modes.md` §4 |
| Wraparound mask `& MAX`              | `04-prediction-modes.md` §4.5 |
| Per-plane Huffman descriptor         | §1 |
| Canonical Huffman code construction  | §2 (longest-length-first; **not** RFC 1951) |
| MSB-first bitstream                  | §2.2 / §3 |
| Raw-mode byte layout                 | §4 |
| Adaptive coding meaning              | §6 |

## 10. Open questions

1. **Under-full Huffman code books.** No fixture surveyed produces
   `Σ 2^-L < 1` (every fixture has Kraft = 1.0). Whether the encoder
   ever produces an under-full book and how the decoder treats it
   (does it accept and zero-fill the unused code positions, or
   reject?) is not pinned down. The decoder's allocation at
   `magicyuv.dll!0x69baf650` (the malloc call for the lengths
   buffer) and the subsequent code-construction at
   `magicyuv.dll!0x69bb2190` should be traced on a hand-crafted
   under-full descriptor in a behavioural harness.

2. **Raw mode for 10/12/14-bit.** §4.1's formula
   `(slice_pixels * bits + 7) / 8` predicts raw-mode slice byte
   counts but is unverified for non-8-bit Backends. A high-entropy
   10/12/14-bit fixture (the v2.4.2 encoder at default settings
   does not produce 10/12/14-bit raw slices on the trace patterns
   surveyed) is needed to confirm the bit packing. The encoder's
   pixel-count multiplication at
   `magicyuv.dll!0x69b95527` (where the bit-depth value returned
   by the Backend's vtable getter is multiplied into the running
   total-pixel count) strongly implies the formula.

3. **Encoder maximum run length cap (`0xfe = 254`)**. The encoder
   caps the run count at 254 (the comparison/branch is at
   `magicyuv.dll!0x69b94600`). This means a maximum single run of
   `1 + 254 = 255` repeated lengths. For `N = 1024..16384`
   (10/12/14-bit), an all-unused-after-symbol-0 distribution would
   need multiple `(0x80, 0xfe)` pairs to fill — e.g. for 10-bit,
   1024 symbols: a typical sparse case would emit
   `01 89 fe 89 fe 89 fe 89 92` or similar (4 full runs of 255 plus
   a small final run). This is purely a function of the encoder's
   chosen byte-count limit and does not affect the wire format
   semantics. 10-bit fixtures would confirm.

4. **Beyond-12 max code length for 10-bit.** The HuffCoder
   parameter triple is `(10, 14, 11)` for 10-bit — the
   max code length is **14**, but the third parameter is **11**.
   This 3rd parameter may be a "fast-path" decoder cap (codes ≤ 11
   bits use a flat lookup table; codes 12..14 use a fallback
   tree-walk). The 8-bit case has `(8, 12, 12)` so `12 == 12` —
   no fallback path. For 10/12/14-bit (`14, 11`), `(16, 12)`,
   `(18, 12)` the fallback path exists. The exact decoder
   semantics need behavioural confirmation but the wire format is
   unaffected: the descriptor's max length value is determined
   solely by the second parameter. Documented for implementer
   awareness.

5. **No decoder check that descriptor consumption matches preamble
   size.** The preamble's total size is determined by `entry[1]`
   (per `02-slice-table.md` §7.1). The decoder's descriptor parser
   consumes exactly the bytes needed to produce N lengths per plane,
   no matter what `entry[1]` says. If a stream had extra bytes
   between the last plane's last descriptor byte and `entry[1] +
   0x20`, the decoder would silently advance to the slice payload.
   v2.4.2 produces no such trailing bytes (every preamble's parsed
   length exactly equals `entry[1] - (4*(N+1) + 1 + total_slices)`),
   but future or third-party encoders could insert padding without
   breaking decoding.

6. **RGB inter-plane decorrelation pipeline placement.** Per
   `04-prediction-modes.md` §5.1, the RGB family applies a
   `B' = B - G; R' = R - G` inter-plane decorrelation between input
   and Backend storage. Whether this is applied before or after
   Huffman coding (the wire residual is the post-decorrelation
   post-prediction residual either way) is left open. The
   decoder MUST reverse the decorrelation to obtain final RGB
   samples; the placement relative to Huffman/prediction is an
   implementation choice that does not affect bit-faithful round-trip.

7. **HuffCoder parameter triples for non-power-of-2
   alphabets.** The current Backends only have `bits ∈ {8, 10, 12,
   14}` (alphabet sizes 256, 1024, 4096, 16384). If a future codec
   revision introduced a non-power-of-2 alphabet (e.g. 9-bit or
   13-bit), the descriptor parser's `N = 1 << bits` formula would
   still work as long as the bit-depth getter returns the right
   value. The wire format is robust against such extensions; this
   is observational, not a requirement.
