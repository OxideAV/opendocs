# OxideAV — Open Codec Documentation

**OxideAV** is an effort to ship modern, MIT-licensed codec
implementations in Rust. This repository — `opendocs` — is the public
home for the wire-format specifications and supporting evidence we
produce while documenting each codec.

The implementations live in separate per-codec crates. This repository
is the documentation those implementations are written against.

## Why a separate documentation repository?

Two reasons:

1. **License independence.** Codec specifications are functional
   facts (the bytes a decoder must read), not creative expression. By
   keeping the specs in this repository under a permissive license and
   building the implementations from them, downstream Rust crates can
   ship under MIT without inheriting any other project's licensing
   constraints.
2. **Reusable reference.** A wire-format description is more broadly
   useful than any single decoder. A non-Rust project, an academic
   reader, or a future implementer should be able to read the spec
   and build their own decoder without having to read our Rust code.

## How we work — the clean-room process

Most of the codecs we document are old, niche, or proprietary, and
the only existing decoders are either closed-source vendor binaries
or open-source decoders under copyleft licenses (typically LGPL).
Reusing copyleft decoders would force OxideAV's implementations to
inherit those licenses; reverse-engineering the proprietary binary
directly is the alternative.

We use a **clean-room methodology** to keep the spec and the
implementation legally independent of any existing decoder source.
The high-level shape is the same in every codec we tackle:

### Roles, separated by a wall

The work is split into four roles, each performed by an isolated
agent that has no access to the other roles' working notes:

- **Specifier** — reads only the proprietary binary and a small set
  of public references (vendor change logs, RFCs for public
  algorithms, container-format references like Microsoft's AVI / RIFF
  documentation). Produces the natural-language wire-format
  description in `spec/`.
- **Extractor** — reads only the proprietary binary. Produces the
  small numeric tables under `tables/` with full provenance (binary
  SHA-256, file offsets, extraction method).
- **Auditor** — reads `spec/`, `tables/`, and the public references.
  Cross-checks every Specifier claim against extracted tables and
  externally-verifiable facts. Writes read-only reports in `audit/`;
  never modifies `spec/` or `tables/`.
- **Implementer** — reads only `spec/` and `tables/`. Writes the
  decoder/encoder code. Numeric values are loaded from `tables/` at
  build time, not retyped.

### What each role is forbidden to read

The forbidden-input lists are codec-specific but always include:

- The source code of any existing open-source decoder for the
  target codec (FFmpeg / libav / mplayer / VLC / GStreamer plugins).
- Any prior analysis we ourselves wrote that referenced those
  sources.
- Community summaries that cite those sources as primary.

Roles see only the inputs allow-listed for them. If an agent
encounters forbidden material mid-session, it stops and the
encounter is logged.

### Why this setup matters

A defensible spec must be derivable from the proprietary binary
plus public references — not from someone else's decoder code. The
role isolation is the mechanism: the Specifier can't pattern-match
against an LGPL decoder it never read.

When the resulting `tables/` happen to contain the same numeric
values as an open-source decoder's static tables, the match is
expected — both encode the same wire-format facts — and the match
is defensible because our values trace through provenance to the
binary, not to any model's recall.

### Reproducibility

Every artifact under `spec/` and `tables/` cites either:

- a binary file offset (for static evidence in the proprietary
  binary), with the binary identified by SHA-256, or
- a behavioural-trace identifier (for evidence from running the
  binary against synthetic inputs in a controlled harness), or
- a clause in a public reference (vendor change-log entry, RFC
  section, AVI/RIFF reference clause).

The `audit/` reports cross-check these citations end-to-end. Internal
session logs, harness scripts, and reproducibility artifacts are kept
in the project's working repository and are not part of this public
documentation set, but the SHA-256 hashes in the spec are sufficient
for any reader who has obtained the same vendor binary to reproduce
every cited offset themselves.

## What's published here

| Codec | Status | Path |
| --- | --- | --- |
| **MagicYUV** v7 (2.0+, vendor build 2.4.2) | spec complete; cross-validated against vendor binary on all native FOURCCs | [`codecs/magicyuv/`](codecs/magicyuv/) |

More codecs will land as their clean-room rebuilds finish.

## Reading order

For each codec, the per-codec `README.md` gives a numbered read order
for its spec chapters and tables. Start there.

## License

All documentation in this repository is licensed under the
[**Creative Commons Attribution 4.0 International**](https://creativecommons.org/licenses/by/4.0/)
license (CC BY 4.0). See [`LICENSE`](LICENSE) for the full legal text.

You are free to share and adapt this material for any purpose,
including commercial use, as long as you give appropriate credit and
indicate if changes were made. Suggested attribution:

> *"OxideAV opendocs"* — https://github.com/OxideAV/opendocs — CC BY 4.0.

A decoder built solely from this documentation may ship under any
license its authors choose; CC BY 4.0 applies to the documentation
text, not to independently-written code.

The MagicYUV proprietary binary referenced as evidence in
`codecs/magicyuv/spec/` is **not** redistributed here. References to
`reference/binaries/...` in the spec are evidence anchors keyed to
SHA-256 hashes; readers who have a copy of the vendor binary can
reproduce every cited offset themselves.

## Contributing

Issues and corrections are welcome. If you find a wire-format claim
in `spec/` that disagrees with what a real decoder produces, please
open an issue with a fixture (or pointer to a public stream) that
demonstrates the disagreement — the spec is supposed to match the
binary, and a divergence is a bug we want to know about.

For new codec contributions, please open an issue first. Writing a
clean-room spec to the standard above takes substantial coordination,
and we'd rather plan that with you than have material arrive that
can't be merged because of how it was produced.
