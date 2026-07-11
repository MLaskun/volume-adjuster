# volume-adjuster — research notes & plan

Goal: a Rust tool that scans a chosen directory for MP3s and adjusts
their volume/gain so a playlist plays back at consistent loudness,
**without adding any distortion**. Cross-platform, developed on
macOS, primary target Windows. Volume is feature 1; more processing
steps may follow later.

## Feasibility: yes

The classic `mp3gain` trick modifies the 8-bit `global_gain` field
already present in every MP3 frame's side info. That field is
applied during decode-time dequantization, so changing it shifts
playback loudness in ~1.5 dB steps **without ever decoding/
re-encoding the audio samples** — i.e. genuinely zero added
distortion (unlike a decode → scale → re-encode pipeline, which
always loses a generation of quality). This is a solved,
well-understood technique, not an open question.

## Key finding: don't reimplement it — `mp3rgain` already exists

[`mp3rgain`](https://github.com/M-Igashi/mp3rgain) is an actively
maintained Rust crate (MIT license, v2.9.6 published 2026-07-10,
verified against the crates.io and GitHub APIs) that implements
exactly this lossless bitstream-gain technique for MP3 (and extends
it to AAC/M4A as a modern `aacgain` replacement — the originals have
been unmaintained since 2009/2015). It ships as CLI, native GUI, and
a **library crate**, and already exposes:

- `collect_audio_files()` — directory scanning
- `analyze()` / `find_max_amplitude()` — pre-adjustment measurement
- `apply_gain_db()` / `apply_gain_to_peak()` /
  `apply_gain_db_auto()` — the lossless gain application
- `would_clip()` — predicts clipping before applying, so a bad
  adjustment can be refused instead of distorting the file
- `undo_gain()` — fully reversible, since original samples are
  never touched
- ReplayGain / ID3v2 / APEv2 tag helpers

Recommendation: depend on `mp3rgain` as a library instead of
hand-rolling MPEG frame parsing. Our code becomes "walk a directory
→ decide target gain per file → call into mp3rgain → report," which
keeps essentially all distortion risk out of our own code.

## Recommended stack

- **Lossless gain engine**: [`mp3rgain`]
- **Directory traversal**: `mp3rgain::collect_audio_files()`
  (fallback: [`walkdir`](https://crates.io/crates/walkdir))
- **CLI args**: [`clap`](https://crates.io/crates/clap)
- **GUI (later)**: [`rfd`](https://crates.io/crates/rfd) for native
  folder picker + [`egui`/`eframe`](https://github.com/emilk/egui)
  for the window
- **Windows installer (later)**: [`cargo-wix`]

## Plan

- [ ] Add `mp3rgain` as a dependency; confirm it builds and
      cross-compiles cleanly to `x86_64-pc-windows-gnu` from macOS
- [ ] CLI skeleton (`clap`) that takes a directory path and lists
      files found via `collect_audio_files()`
- [ ] Core loop: analyze each file → decide target gain (start with
      track-level normalization to a fixed reference loudness) →
      `would_clip()` check → apply gain → log results
- [ ] Dry-run mode; rely on `undo_gain()` for reversibility instead
      of hand-rolled backups
- [ ] Validate end-to-end against a real folder of MP3s, confirm no
      audible/measurable quality change
- [ ] Decide loudness reference target (ReplayGain 1.0 ~89 dB vs
      EBU R128 -18 LUFS) — policy choice, `mp3rgain` speaks
      ReplayGain 1.0 by default
- [ ] GUI: wrap CLI core with `rfd` folder picker + `egui`
      progress/results window
- [ ] Windows packaging: `cargo-wix` MSI once there's a release
      candidate
- [ ] Stretch: extend beyond MP3 gain (e.g. more formats, batch
      tagging, other processing steps)

## Open items to sanity-check once code starts

- Re-check `mp3rgain`'s LICENSE file directly before distribution
  (MIT, should be a non-issue)
- Confirm the crate's AAC/M4A path is as mature as its MP3 path if
  that ends up mattering
- MSVC-ABI Windows builds are more friction to cross-compile from
  macOS than GNU-ABI; likely easiest to build MSVC target via CI
  (GitHub Actions `windows-latest`) rather than fighting it locally
  — not blocking early development, which can happen via plain
  `cargo build`/`cargo run` on macOS
