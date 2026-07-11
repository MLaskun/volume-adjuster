# volume-adjuster — research notes & plan

Goal: a Rust tool that scans a chosen directory for MP3s and adjusts
their volume/gain so a playlist plays back at consistent loudness,
**without adding any distortion**. Cross-platform, developed on
macOS, primary target Windows. Volume is feature 1; more processing
steps may follow later.

## Stack

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
