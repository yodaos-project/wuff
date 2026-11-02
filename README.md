# Wuff

[![Crates.io](https://img.shields.io/crates/v/wuff.svg)](https://crates.io/crates/wuff)
[![Docs](https://img.shields.io/docsrs/wuff/latest)](https://docs.rs/wuff)
[![Crates.io License](https://img.shields.io/crates/l/wuff)](#license)

A lightweight, pure-Rust decoder for [WOFF](https://www.w3.org/TR/WOFF/) and [WOFF2](https://www.w3.org/TR/WOFF2/) web fonts.

This is the YodaOS fork of [nicoburns/wuff](https://github.com/nicoburns/wuff).
It adds stricter WOFF2 memory limits for Ink; see
[WOFF2 limits in this fork](#woff2-limits-in-this-fork). The crates.io and
docs.rs badges above refer to the upstream published crate, not this fork.

Wuff takes a compressed WOFF or WOFF2 file and decodes it back into a plain
[OpenType/TrueType (sfnt)](https://learn.microsoft.com/en-us/typography/opentype/spec/otff)
font that can be parsed by standard font tooling.

Upstream is hand-ported from Google's [woff2](https://github.com/google/woff2)
C++ library and reports byte-identical results for the fonts in
[google/fonts](https://github.com/google/fonts). This fork's stricter limits can
reject fonts accepted by upstream.

## Why Wuff?

- **Pure Rust** — no C/C++ toolchain or bindings required.
- **Support both WOFF and WOFF2 formats**
- **Lightweight** — minimal dependencies, with the compression backends behind
  optional features.
- **Bring your own decompressor** — use the bundled backends, or plug in your
  own Brotli/zlib implementation.

## Encoding

Wuff currently only includes the decoder and does not yet have a port of the encoder. For encoding, consider:
  - https://github.com/bearcove/woofwoof (Rust bindings to the C++ woff2 library)
  - https://github.com/0x6b/ttf2woff2 (AI-assisted port of the C++ woff2 encoder to Rust)

## Usage

Decode a WOFF2 file into an OpenType/TrueType font:

```rust
let woff2_bytes = std::fs::read("font.woff2")?;
let otf_bytes = wuff::decompress_woff2(&woff2_bytes)?;
```

Decode a WOFF (version 1) file:

```rust
let woff_bytes = std::fs::read("font.woff")?;
let otf_bytes = wuff::decompress_woff1(&woff_bytes)?;
```

### Custom decompressors

If you'd rather not pull in the bundled compression crates (for example to share
a Brotli implementation you already depend on), disable the default features and
supply your own decompression closure:

```rust
use wuff::decompress_woff2_with_custom_brotli;

let otf_bytes = decompress_woff2_with_custom_brotli(&woff2_bytes, &mut |input, hint| {
    // Decompress `input`, optionally using `hint` as a size hint for the output buffer.
    my_brotli_decompress(input, hint)
})?;
```

A matching `decompress_woff1_with_custom_z` is available for WOFF1.

### WOFF2 limits in this fork

Ink processes fonts on memory-constrained devices. Compared with upstream
wuff 0.2.9, this fork:

- Limits compressed WOFF2 input, the decompressed table stream, and the
  reconstructed font to 16 MiB each, instead of upstream's 128 MiB limit on
  decompressed and reconstructed data.
- Rejects WOFF2 files with more than 256 tables, collections with more than
  128 faces, and collection entries with more than 256 table references.
- Checks the caller's remaining budget while reconstructing tables, including
  `glyf` and `loca`, rather than relying only on the font's declared size or
  checking the result after reconstruction.
- Adds `decompress_woff2_with_limit` for callers that need a lower per-call
  limit. The limit is capped at 16 MiB. It applies to the decompressed stream
  and reconstructed font; the compressed input still has the 16 MiB cap.

With the `brotli` or `brotli-c` feature enabled:

```rust
let woff2_bytes = std::fs::read("font.woff2")?;
let remaining_budget = 8 * 1024 * 1024;
let otf_bytes = wuff::decompress_woff2_with_limit(&woff2_bytes, remaining_budget)?;
```

The existing `decompress_woff2` and `decompress_woff2_with_custom_brotli`
entry points also use the 16 MiB limit. The custom Brotli callback is
responsible for limiting its own allocations. These checks do not cap every
temporary allocation or guarantee a 16 MiB total process-memory peak. They
apply to WOFF2; they do not introduce a new WOFF1 budget. Ink separately
manages its application font-registry budget.

To depend on this fork from Cargo, pin a reviewed Git commit, for example:

```toml
wuff = { git = "https://github.com/yodaos-project/wuff.git", rev = "779357934149ccd8ffa4a4eb01da0592f342dfba", default-features = false, features = ["brotli"] }
```

Inputs exceeding these limits return an error, including otherwise valid large
fonts. See [INK-CHANGES.md](./INK-CHANGES.md) and run the regression tests with
`cargo test -p wuff --test limits`.

## Conformance testing

The `conformance` crate is a test harness which verifies that three decoders
produce byte-identical output for every font it tests:

1. Google's C++ woff2 reference decoder (the `woff2_decompress` binary)
2. The wuff Rust decoder (this crate)
3. The `wuff-capi` C++ wrapper around the wuff decoder

Requirements: a C++ toolchain, plus `cmake` and `brotli` to build the C++ woff2 tools.

Its inputs are:

- The WOFF2 files from the `css/WOFF2` section of the [web-platform-tests](https://github.com/web-platform-tests/wpt)
  committed to this repository under `conformance/fonts/wpt/`. This test suite contains deliberately
  invalid WOFF2 files, so consistent rejection by all three decoders is considered a test pass.
- Every `ttf`/`otf`/`ttc` font in the [google/fonts](https://github.com/google/fonts)
  repository, encoded to WOFF2 with the C++ reference encoder (`woff2_compress`).

Run it with:

```sh
cargo run -rp conformance
cargo run -rp conformance --refresh-fonts  # re-download google/fonts and rebuild the cache
```

The first run downloads the google/fonts repository (~1.5GB) and encodes every
font at maximum Brotli quality, which takes a while. The encoded WOFF2
files (~820MB) are cached in `.data/encoded`. Use `--refresh-fonts` to a force a cache refresh.

## Repository layout

This repository contains both the published crate and the reference material
used to develop it:

- `wuff/` - the published crate: an idiomatic Rust rewrite of the decoder.
- `wuff-capi/` - a C API for the wuff decoder, usable as a drop-in replacement for the woff2 C++ library's decoding API.
- `conformance/` - the conformance test harness described above.
- `woff2/` - a copy of Google's [woff2](https://github.com/google/woff2/) C++
  library, used as the reference implementation.

## License

Licensed under the [MIT License](./LICENSE).
