# Mushaf Page Render Spike — Design

**Date:** 2026-05-27
**Status:** Proposed
**Scope:** First milestone of the `hq-react` rebuild — prove the Quran-page rendering pipeline.

## Goal

A browser tab shows **page 5 of the Quran**, rendered as live text with the
DigitalKhatt font on a Skia canvas, with each line **justified** (letters
stretched via the font's OpenType variant features) so the page visually
matches a printed mushaf.

This is a throwaway-quality spike to de-risk the single hardest part of the app.
Code quality and reusability are secondary to proving the pipeline works.

## Success Criteria

Open `expo start --web` in a browser → see page 5 → lines fill edge-to-edge with
correct calligraphic letter-stretching (not whitespace padding), matching a
reference mushaf page image closely enough that a reader recognizes it.

## Out of Scope (deferred, not forgotten)

- SQLite, the `hq-data` JSON seam, seeding. Page-5 data is **hardcoded**.
- Other pages, navigation, page-turning.
- Tajweed coloring, word tap/selection, surah-name headers.
- The persistent layout cache (Bayaan uses MMKV; not needed for one page).
- Native iOS/Android simulators — **web only** for this milestone.
- Any Bayaan source code. See Legal Constraint.

## Legal Constraint (governs all implementation)

The app may go commercial. Bayaan (`github.com/thebayaan/Bayaan`), the reference
React Native DigitalKhatt implementation, is **AGPL-3.0** — we must **not copy
its source code**. We reimplement the justification algorithm clean-room, using:

- Bayaan's published **documentation** (`docs/features/digital-khatt/*.md`) —
  algorithm descriptions are not copyrightable.
- The **DigitalKhatt font's own** OpenType feature interface (`cv01`–`cv18`,
  `jalt`, etc.), published by the DigitalKhatt project (Amine Anane).

The **DigitalKhattV2.otf** font is **SIL OFL 1.1** — commercial-safe to bundle.
No Bayaan `.ts` files are read, copied, or adapted. Docs only.

## Architecture

Single Expo app targeting web via React Native Skia's CanvasKit (WASM) backend.

```
App (one screen)
 └─ MushafPage          reads hardcoded page-5 data, sizes the canvas,
                        lays out 15 line slots, runs justification per line
     └─ MushafLine      builds one Skia Paragraph (RTL) per line, attaches
                        per-character OpenType fontFeatures, draws it
         └─ Justifier   pure function: given line text + target width + font
                        manager, returns the per-character feature map that
                        makes the line fill the width
```

### Components

- **`MushafPage`** — owns page geometry. Derives content area from window size,
  computes `scale`, `lineWidth`, `fontSize`, and 15 evenly-spaced vertical line
  slots. For each line, calls the Justifier, then renders a `MushafLine`.

- **`MushafLine`** — builds one Skia `Paragraph` with `TextDirection.RTL`, base
  DigitalKhatt style. Walks the line's characters; for each, attaches the
  OpenType `fontFeatures` the Justifier assigned to that character index. Draws
  the paragraph at its vertical slot, right-aligned.

- **`Justifier`** (pure module, the core of the spike) — clean-room
  reimplementation of the measure → stretch → re-measure loop:
  1. Measure the natural width of the line.
  2. If wider than target: shrink font ratio (no kashida).
  3. If narrower: apply OpenType letter-variant features in a strict priority
     order (`cv*` lookups described in Bayaan's `justification-engine.md`),
     re-measuring after each candidate, accepting only features that increase
     width without overshooting the target.
  4. Distribute any small remainder into inter-word spacing.
  Returns `{ fontFeaturesByCharIndex, fontSizeRatio, spacing }` for the line.

### Data (hardcoded for this milestone)

Page 5 is 15 ayah lines (word IDs 358–487 in the QPC v2 15-line layout). The
hardcoded fixture is a TS file: an array of 15 lines, each an array of words,
each word the Uthmani character string. Prepared once, offline, then committed
into the app.

**Fixture prep (offline, one-time):** The QUL `qpc-v2-15-lines.db` gives, per
line, the `first_word_id..last_word_id` range — i.e. the **word count** per
line. `hq-data` provides per-verse Uthmani text for the verses on page 5. There
is no local global-word-ID → text table, so we do not rely on the word IDs as
keys. Instead: take the page-5 verses' Uthmani text in order, split on spaces to
get a flat word sequence, then slice that sequence into the 15 lines using the
per-line word counts from the QUL layout. This produces the 15-line fixture
without needing a word-ID lookup. (If the flat-split word count and the QUL
counts disagree at a boundary, it is corrected by hand during prep — this is a
one-page, one-time fixture.) The app itself has zero data-loading logic.

## Data Flow (page 5 render)

1. `MushafPage` mounts, loads `DigitalKhattV2.otf` into a Skia font provider
   (`useFonts`), waits until ready.
2. Reads the hardcoded 15-line page-5 fixture.
3. Computes geometry (content width/height, font size, line slots).
4. For each line: `Justifier(lineText, targetWidth, fontMgr)` → feature map.
5. Each `MushafLine` builds its RTL Paragraph with per-char features, draws it.
6. Browser shows the justified page.

## Key Technical Risk (proven in build step 1, not assumed)

The whole approach depends on DigitalKhatt's OpenType variant features
(`cv01`–`cv18`) actually taking effect through RN-Skia's **CanvasKit/web**
shaper. CanvasKit shapes text with HarfBuzz (WASM), which honors OpenType
feature tags — so this is expected to work, but it is unverified.

**First implementation step is a smoke test:** render a single DigitalKhatt word
twice in the browser, once with `cv01` off and once on, and confirm the glyph
visibly changes. If it fails, the browser-first plan is reconsidered before any
engine work. Cheap to check, fatal if ignored.

## Build Order

1. **Smoke test** — Expo web + RN-Skia + DigitalKhatt; prove `fontFeatures`
   change a glyph in the browser. (De-risks the linchpin.)
2. **Natural-width render** — draw all 15 lines of page 5 RTL at natural width.
   Confirms font + fixture + Skia pipe. Lines will be ragged — that's expected.
3. **Justification** — implement the Justifier; lines fill to width. Milestone
   "perfect" is reached here.
4. **Visual check** — compare against a reference page-5 image in the browser.

## Verification

- Build steps 1–2 are self-verifying (glyph changes; Arabic appears).
- Step 3/4: side-by-side with a known-good page-5 mushaf image. The spike is
  done when the rendered page matches to a reader's eye.
- No automated tests for this throwaway spike beyond the smoke test; the
  artifact is a visual one and is verified visually.

## What This Spike Decides For the Real App (later, not now)

If it succeeds: confirms DigitalKhatt + RN-Skia + a clean-room justifier is the
rendering foundation, and the real app's data seam (SQLite from `hq-data`),
caching, and interactivity get built on top. If the smoke test fails: forces an
earlier architecture rethink (native-only, or a different shaper path).
