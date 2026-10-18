# Word-by-word captions (AVX2 work in progress)

Prepared on 18 October 2026. Build and runtime validation are still pending.

The Text inspector has an opt-in **Word by word** switch (Italian:
**Comparsa parola per parola**). A caption remains one text clip on its existing
track. The switch uses the normal inspector commit/undo mechanism. Turning it
off restores the complete phrase without discarding its word timings.

The complete content is shaped and wrapped before visibility is considered.
Hidden glyphs keep their advance, but paint no fill, stroke, shadow or colour
font marks. The complete background plate remains visible. This also keeps
centring, line breaks and vertical placement fixed when later words appear.

`TextStyle.wordReveal` defaults to false for older documents. Optional
`wordStarts` contains normalized onsets in the text renderer's word order.
Times survive save/load and scale with caption duration. Editing content clears
the old timings; creating another caption or saving a reusable style never
copies the previous caption's timings. A wrong timing count falls back as a
whole to even intervals, from the clip start to 80% of its duration, leaving
the final 20% for reading. A one-word title is visible from the start.

The transcriber now requests token timestamps from the installed Whisper model
with `FullParams::set_token_timestamps(true)`. It preserves ordinary text tokens
as byte spans plus their `t0`/`t1` times (centiseconds converted to seconds).
Special and timestamp tokens are excluded. Bytes are reconstructed and compared
with the complete segment before trimming whitespace: individual tokens may
split a UTF-8 character, and decoding each token as a lossy string would corrupt
the word mapping. A reconstruction failure keeps the text but discards timing.

The caption pane groups these spans with `concat_text::word_ranges`, the same
Unicode line-break segmentation used by the painter. Each word takes the onset
of the token containing its first byte. Subword tokens are not separate words;
a token containing multiple words gives them the same onset rather than
inventing an interpolation. Onsets become caption fractions after correcting
for the source clip's speed and the caption's actual duration, including its
minimum duration. They are saved even when the reveal switch is off.

Missing, invalid, incomplete or backwards word onsets use the regular cadence
as a fallback, as requested by the user. The text inspector identifies saved
word times versus regular cadence. Script captions also keep the fallback.
Existing captions without word timing need to be transcribed again by the
eventual new build; the existing executable and projects are not changed here.

Whisper calls token timestamps **experimental**. These are acoustic timing
estimates from whisper.cpp, not DTW or the separate forced aligner used in the
earlier audio experiment. Early first words, silence and recognition mistakes
can still affect synchronization. No new model, download, language restriction
or speech dependency is introduced. The API was checked against the published
whisper-rs 0.16.0 crate, with its SHA-256 verified against `Cargo.lock`.

API reference: [Whisper segment tokens](https://docs.rs/whisper-rs/0.16.0/whisper_rs/struct.WhisperSegment.html),
[token timestamps](https://docs.rs/whisper-rs/0.16.0/whisper_rs/struct.FullParams.html#method.set_token_timestamps).

The host caches private prefix PNG states at the requested preview/export
resolution. These are generated render cache entries, never project media,
additional clips or tracks. Both playback/prefetch and export select the same
state on the quantized timeline clock, including backward seeks. Export uses
the still reader pool rather than a decoder frozen on the first state. Only
needed states are decoded. The existing title cache sweep also covers these
PNGs. A cache write failure logs a warning and leaves the caption readable as
ordinary text.

The first render costs more than a static title because it paints each prefix.
Profile long titles, font/size dragging and 4K projects before distribution;
this implementation prioritizes exact native glyph/plate rendering over a
rectangle mask that could leave parts of shadows or outlines visible.

## Validation status

Completed without compiling:

- Rust syntax parsing of every changed Rust file with Tree-sitter.
- Exhaustive struct-literal checks for the new fields.
- JSON/TOML parsing and English/Italian label wiring.
- Shared preview/export state selector. The AVX2 build workflow is manual-only;
  its build jobs, AVX2 settings and pinned downloads are unchanged.
- Pinned Whisper API signatures, byte reconstruction, native word segmentation,
  speed/duration conversion and preserved regular-cadence fallback wiring.
- `git diff --check` and verification that the earlier Whisper timestamp
  backport remains the sole change inside the vendored Whisper crate.

Native regression tests are prepared for old-document defaults, timing
round-trips, full-layout invariance, stationary pixels, wrapped/RTL/emoji text,
one clip/one render layer, switch undo, backward seeking and prefetch timing.
Additional tests cover split UTF-8 tokens, subword grouping, shared-token onsets,
clip speed, minimum duration, timing storage and the regular fallback.
They have **not** been compiled or executed. Slint compilation and Rust type
checking have also **not** been run. No claim of in-app visual validation is
made for this source change.

When a build is authorized, first run formatting and the focused tests from
`src/` (`concat-project`, `concat-text`, `concat-host`, `concat-speech`,
`concat-export` and the caption-pane tests in `concat`), then
start the AVX2 workflow manually from Actions > Run workflow. Check playback, scrubbing, export,
save/reopen and switch undo on a fresh project copy at portrait and landscape
resolutions. Also test caption transforms, overlap/fades and editing text.
Regenerate captions from the same `vocal.MP3` with the same Base model, verify
that saved `wordStarts` are populated and not evenly spaced, and listen through
the word onsets. Compare Small separately if already installed. No prior result
from the forced aligner or standalone CLI is a test of this unbuilt integration.
The timestamp correction already prepared in this checkout must remain in
the same eventual build.
