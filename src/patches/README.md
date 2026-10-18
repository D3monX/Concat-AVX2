# Patched dependencies

Crates copied from crates.io with one change each, applied through the
workspace's `[patch.crates-io]`. Each is the published version the
`Cargo.lock` names, so an upgrade is: copy the new version in, re-apply
the change, and read it again.

## whisper-rs-sys 0.15.0

The published crate (SHA-256
`6986c0fe081241d391f09b9a071fbcbb59720c3563628c3c829057cf69f2a56f`),
including its whisper.cpp 1.8.3 sources, with only the upstream timestamp
fix [c33c5618b72bb345df029b730b36bc0e369845a3](https://github.com/ggml-org/whisper.cpp/commit/c33c5618b72bb345df029b730b36bc0e369845a3)
backported to `whisper.cpp/src/whisper.cpp`. The exact upstream diff is
kept in `whisper-timestamp-gaps.patch`.

The old segment loop discarded consecutive timestamp tokens and reused
the previous segment's end as the next start. A pause could therefore
appear inside the next caption. The fix advances the start to the last
consecutive timestamp instead. It also includes upstream's special-token
printing and flushing changes; no decoding parameters or model files change.

The workspace's local Cargo override selects this copy. Its lock entry
has the same version and dependencies, with registry source/checksum
removed because this is now a path dependency. Keep the registry checksum
above to verify provenance when refreshing the copy. All other vendored
files match the crate archive byte for byte, including the original licenses.

This patch applies to the common timestamp parser for all Whisper models;
it is not an Italian-only aligner. It does not resolve every boundary error,
including a first caption that starts in leading silence. AVX2 build flags,
downloads, model choices, and application settings remain as before.

Validation completed without compiling: crate checksum, TOML parsing,
exact vendored-file comparison, clean application and reverse application
of the upstream patch, and unchanged AVX2 workflow. Before distributing
the next build, compile with the existing locked workflow and verify actual
transcription after pauses with the same model/audio. In standalone engine
diagnostics the affected old version returned starts of 6 and 14 seconds,
whereas a corrected newer engine returned 12 and 25 seconds. This is
supporting evidence, not a test of this still-unbuilt application backport.

No new model or alignment package is required by this patch.

## i-slint-renderer-skia 1.18.1

`itemrenderer.rs`, `draw_glyph_run`: glyphs are positioned on whole
pixels when the window's scale factor is under 1.5. Slint always asks
Skia for subpixel positioning, which on a 1x Windows screen lands stems
between pixels and the text reads blurred (jub0t/concat#278). On a
high-density screen the subpixel positions stay, where they are worth
their spacing. Nothing else differs from the published crate.
