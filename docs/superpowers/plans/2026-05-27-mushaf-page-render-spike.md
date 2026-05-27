# Mushaf Page Render Spike — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Definition of Done:** Open the app in a browser (`expo start --web`) and see page 5 of the Quran rendered with the DigitalKhatt font on a Skia canvas, with every ayah line justified (filled edge-to-edge via the font's own letter-stretching, not whitespace), closely matching a printed mushaf page to a reader's eye.

**Goal:** Prove the DigitalKhatt + RN-Skia + clean-room justification rendering pipeline by rendering one hardcoded page in a browser.

**Architecture:** Single Expo app, web target via RN-Skia CanvasKit (WASM). One screen renders a hardcoded 15-line page-5 fixture. A `MushafPage` component sizes the canvas and line slots; a `MushafLine` builds one RTL Skia `Paragraph` per line with per-character OpenType `fontFeatures`; a pure `justifier` module computes which features fill each line to its target width. No SQLite, no navigation, no networking.

**Tech Stack:** Expo (SDK 53+), React Native, `@shopify/react-native-skia` (Paragraph API, CanvasKit web backend), TypeScript, DigitalKhattV2.otf (SIL OFL).

**Legal:** Clean-room. Reference Bayaan's *docs* only (algorithm description); never copy Bayaan's AGPL source. DigitalKhatt font is OFL, commercial-safe.

**Spec:** `hq-docs/docs/superpowers/specs/2026-05-27-mushaf-page-render-spike-design.md`

---

## User Journey Walkthrough

1. **User runs `npx expo start --web`** → browser tab opens at the dev server. → No existing component; built in Task 1.
2. **User sees a blank Skia canvas, then a smoke-test glyph** (Task 2 only; removed after) → confirms font features work.
3. **User sees page 5's 15 lines of Arabic** appear on a mushaf-colored canvas, right-aligned, at natural width (ragged left edges). → `MushafPage` + `MushafLine`, Tasks 4–5.
4. **User sees the lines snap to full width**, letters elongated calligraphically so each line reaches both margins. → `justifier`, Task 6.
5. **User compares to a reference page-5 image** in another tab and judges the match. → Task 7 (visual verification).

Every journey step maps to a task below. No step is "TBD."

---

## File Structure

All paths under `hq-react/`.

- `app.json`, `package.json`, `tsconfig.json`, `babel.config.js`, `metro.config.js` — Expo project config (Task 1).
- `App.tsx` — root; mounts the screen, hosts the smoke test (Task 2), then the page (Task 4+).
- `assets/fonts/DigitalKhattV2.otf` — the font (Task 3).
- `src/fixtures/page5.ts` — hardcoded 15-line page-5 data (Task 4).
- `src/render/MushafPage.tsx` — geometry + line slots + orchestration (Task 5).
- `src/render/MushafLine.tsx` — one RTL Paragraph per line, per-char features (Task 5).
- `src/render/justifier.ts` — pure justification function (Task 6).
- `src/render/justifier.test.ts` — unit tests for the justifier (Task 6).

---

## Task 1: Scaffold Expo app with web + Skia

**Files:**
- Create: `hq-react/` (Expo project)
- Modify: `hq-react/package.json`

- [ ] **Step 1: Scaffold a blank TypeScript Expo app into the existing empty `hq-react/` dir**

Run from the workspace root:
```bash
cd /Users/bexgboost/projects/hafiz-quran-app
npx create-expo-app@latest hq-react --template blank-typescript
```
If it refuses because the dir exists/non-empty, scaffold to a temp dir and move files in:
```bash
npx create-expo-app@latest /tmp/hqr --template blank-typescript
cp -R /tmp/hqr/. hq-react/ && rm -rf /tmp/hqr
```

- [ ] **Step 2: Install Skia + web dependencies**

Run:
```bash
cd hq-react
npx expo install @shopify/react-native-skia react-native-web react-dom @expo/metro-runtime
```

- [ ] **Step 3: Initialize git in hq-react (it is its own repo)**

Run:
```bash
cd hq-react && git init && printf "node_modules/\n.expo/\ndist/\nweb-build/\n" > .gitignore
```

- [ ] **Step 4: Verify the app boots on web**

Run:
```bash
cd hq-react && npx expo start --web
```
Expected: browser opens, shows the default Expo blank-template text ("Open up App.tsx..."). Stop the server (Ctrl-C) once confirmed.

- [ ] **Step 5: Commit**

```bash
cd hq-react && git add -A && git commit -m "chore: scaffold Expo app with Skia + web support"
```

---

## Task 2: Smoke-test DigitalKhatt OpenType features in the browser (LINCHPIN)

This proves the whole approach: that `fontFeatures` change a DigitalKhatt glyph through RN-Skia's web/CanvasKit shaper. If this fails, STOP and report — the browser-first plan must be reconsidered.

**Files:**
- Add: `hq-react/assets/fonts/DigitalKhattV2.otf`
- Modify: `hq-react/App.tsx`

- [ ] **Step 1: Copy the OFL font into assets**

Run:
```bash
mkdir -p hq-react/assets/fonts
cp "hq-flutter/.claude/worktrees/font-spike/mushaf-render-spike-flutter/assets/fonts/DigitalKhattV2.otf" hq-react/assets/fonts/DigitalKhattV2.otf
```

- [ ] **Step 2: Confirm the exact `fontFeatures` property shape from the INSTALLED types**

Do not trust web docs for this. Read the installed type definition:
```bash
cd hq-react && grep -rn "fontFeatures" node_modules/@shopify/react-native-skia/lib/typescript/ | head
grep -rn "FontFeature" node_modules/@shopify/react-native-skia/lib/typescript/ | head
```
Expected: a type like `fontFeatures?: FontFeature[]` where `FontFeature = { name: string; value: number }`. Use whatever the installed types actually declare in Step 3. If the property is absent from the types entirely, STOP — features are unsupported in this version; report before continuing.

- [ ] **Step 3: Replace `App.tsx` with a two-glyph comparison**

Render one Arabic word twice with DigitalKhatt — once with no features, once with `cv01` enabled — stacked vertically. Use the `FontFeature` shape confirmed in Step 2 (shown here as `{ name, value }`).

```tsx
import React from "react";
import {
  Canvas,
  Paragraph,
  Skia,
  TextDirection,
  useFonts,
} from "@shopify/react-native-skia";

const WORD = "مَـٰلِكِ"; // a word with kashida-eligible joins

export default function App() {
  const fontMgr = useFonts({
    DigitalKhatt: [require("./assets/fonts/DigitalKhattV2.otf")],
  });

  if (!fontMgr) return null;

  const make = (features?: { name: string; value: number }[]) => {
    const b = Skia.ParagraphBuilder.Make(
      { textDirection: TextDirection.RTL },
      fontMgr
    );
    b.pushStyle({
      fontFamilies: ["DigitalKhatt"],
      fontSize: 64,
      color: Skia.Color("black"),
      ...(features ? { fontFeatures: features } : {}),
    });
    b.addText(WORD);
    b.pop();
    const p = b.build();
    p.layout(600);
    return p;
  };

  const plain = make();
  const stretched = make([{ name: "cv01", value: 4 }]);

  return (
    <Canvas style={{ flex: 1, backgroundColor: "#FDF6E3" }}>
      <Paragraph paragraph={plain} x={20} y={40} width={600} />
      <Paragraph paragraph={stretched} x={20} y={160} width={600} />
    </Canvas>
  );
}
```

- [ ] **Step 4: Run on web and visually compare**

Run:
```bash
cd hq-react && npx expo start --web
```
Expected: two renderings of the word. The lower one (with `cv01=4`) shows a **visibly different/wider letter form** than the upper one. If they are identical, features are NOT applying through CanvasKit — STOP and report (try other tags `cv02`/`jalt`/higher values once; if still identical, the linchpin has failed).

- [ ] **Step 5: Commit**

```bash
cd hq-react && git add -A && git commit -m "test: prove DigitalKhatt fontFeatures apply via Skia web (smoke test)"
```

---

## Task 3: Prepare the page-5 fixture (offline, one-time)

**Files:**
- Create: `hq-react/src/fixtures/page5.ts`

Page 5 (QPC v2, 15-line) is mid-Al-Baqarah, 15 ayah lines, 130 words total, with per-line word counts: `[8,9,10,9,11,8,8,9,9,9,7,7,9,9,8]` (from `qpc-v2-15-lines.db`).

- [ ] **Step 1: Extract the page-5 Uthmani word sequence**

The verses on page 5 come from `hq-data/data/raw/verses/002-Al-Baqarah.json` (`text_uthmani` per verse). Identify the verse range whose words fill exactly 130 words for page 5, take each verse's `text_uthmani`, concatenate in order, and split on spaces into a flat 130-word array. Use this throwaway script (run from workspace root) to print candidate words; adjust the verse slice until the count is 130:

```bash
python3 - <<'PY'
import json
d = json.load(open("hq-data/data/raw/verses/002-Al-Baqarah.json"))
# Page 5 of QPC v2 is mid-Al-Baqarah; print verse_key + word count to locate the 130-word span.
for v in d["verses"]:
    t = v.get("text_uthmani","").strip()
    print(v["verse_key"], len(t.split()), t)
PY
```
Read the output, find the contiguous verse span summing to 130 words, and record those words in order. Cross-check the span against a reference page-5 image (quran.com page 5) so the first/last words match the printed page.

- [ ] **Step 2: Slice the 130 words into 15 lines by the QUL counts and write the fixture**

Create `hq-react/src/fixtures/page5.ts`. Shape: an array of 15 lines, each line an array of word strings. Fill it with the actual words from Step 1, sliced using counts `[8,9,10,9,11,8,8,9,9,9,7,7,9,9,8]`.

```ts
// Page 5 of the QPC v2 15-line mushaf (Al-Baqarah). Hardcoded spike fixture.
// Each inner array is one line; each string is one Uthmani word.
// Word counts per line: [8,9,10,9,11,8,8,9,9,9,7,7,9,9,8] (from qpc-v2-15-lines.db).
export const PAGE5_LINES: string[][] = [
  // line 1 (8 words) — fill with real words from Step 1
  ["<w1>", "<w2>", "<w3>", "<w4>", "<w5>", "<w6>", "<w7>", "<w8>"],
  // ... lines 2-15, filled with the real sliced words ...
];
```
Replace ALL `<wN>` placeholders with the real words before finishing this task. The placeholders above are a template — the committed file must contain real Arabic words and no `<` characters.

- [ ] **Step 3: Verify the fixture shape**

Run:
```bash
cd hq-react && npx tsx -e "import {PAGE5_LINES} from './src/fixtures/page5'; const c=PAGE5_LINES.map(l=>l.length); console.log(c, c.reduce((a,b)=>a+b,0));"
```
Expected: `[8,9,10,9,11,8,8,9,9,9,7,7,9,9,8] 130`. (If `tsx` is unavailable, eyeball the file: 15 lines, counts matching, no `<` placeholders.)

- [ ] **Step 4: Commit**

```bash
cd hq-react && git add src/fixtures/page5.ts && git commit -m "data: add hardcoded page-5 Uthmani fixture"
```

---

## Task 4: Render page 5 at natural width (no justification yet)

**Files:**
- Create: `hq-react/src/render/MushafLine.tsx`
- Create: `hq-react/src/render/MushafPage.tsx`
- Modify: `hq-react/App.tsx`

- [ ] **Step 1: Create `MushafLine.tsx` — one RTL paragraph per line**

```tsx
import React, { useMemo } from "react";
import {
  Paragraph,
  Skia,
  SkTypefaceFontProvider,
  TextDirection,
} from "@shopify/react-native-skia";

type Props = {
  words: string[];
  fontMgr: SkTypefaceFontProvider;
  fontSize: number;
  x: number;
  y: number;
  width: number;
};

export function MushafLine({ words, fontMgr, fontSize, x, y, width }: Props) {
  const paragraph = useMemo(() => {
    const b = Skia.ParagraphBuilder.Make(
      { textDirection: TextDirection.RTL },
      fontMgr
    );
    b.pushStyle({
      fontFamilies: ["DigitalKhatt"],
      fontSize,
      color: Skia.Color("black"),
    });
    b.addText(words.join(" "));
    b.pop();
    const p = b.build();
    p.layout(width);
    return p;
  }, [words, fontMgr, fontSize, width]);

  return <Paragraph paragraph={paragraph} x={x} y={y} width={width} />;
}
```

- [ ] **Step 2: Create `MushafPage.tsx` — geometry + 15 line slots**

```tsx
import React from "react";
import { useWindowDimensions } from "react-native";
import { Canvas, SkTypefaceFontProvider } from "@shopify/react-native-skia";
import { PAGE5_LINES } from "../fixtures/page5";
import { MushafLine } from "./MushafLine";

const PAGE_ASPECT = 1080 / 1747; // QPC page width/height ratio

export function MushafPage({ fontMgr }: { fontMgr: SkTypefaceFontProvider }) {
  const { height: winH } = useWindowDimensions();
  const pageH = winH - 32;
  const pageW = pageH * PAGE_ASPECT;
  const marginX = pageW * 0.06;
  const contentW = pageW - 2 * marginX;
  const contentTop = pageH * 0.05;
  const contentH = pageH * 0.9;
  const interline = contentH / PAGE5_LINES.length;
  const fontSize = contentW / 13; // initial guess; justifier refines later

  return (
    <Canvas style={{ width: pageW, height: pageH, backgroundColor: "#FDF6E3" }}>
      {PAGE5_LINES.map((words, i) => (
        <MushafLine
          key={i}
          words={words}
          fontMgr={fontMgr}
          fontSize={fontSize}
          x={marginX}
          y={contentTop + i * interline}
          width={contentW}
        />
      ))}
    </Canvas>
  );
}
```

- [ ] **Step 3: Wire `App.tsx` to load the font and mount `MushafPage`**

```tsx
import React from "react";
import { View } from "react-native";
import { useFonts } from "@shopify/react-native-skia";
import { MushafPage } from "./src/render/MushafPage";

export default function App() {
  const fontMgr = useFonts({
    DigitalKhatt: [require("./assets/fonts/DigitalKhattV2.otf")],
  });
  if (!fontMgr) return null;
  return (
    <View style={{ flex: 1, alignItems: "center", backgroundColor: "#333" }}>
      <MushafPage fontMgr={fontMgr} />
    </View>
  );
}
```

- [ ] **Step 4: Run on web and verify**

Run:
```bash
cd hq-react && npx expo start --web
```
Expected: a cream-colored page with 15 right-aligned lines of Arabic in DigitalKhatt. Left edges will be **ragged** (un-justified) — that is correct for this task. All 15 lines visible and legible.

- [ ] **Step 5: Commit**

```bash
cd hq-react && git add -A && git commit -m "feat: render page 5 at natural width with DigitalKhatt"
```

---

## Task 5: Define the justifier contract with a stub (TDD setup)

Establish the justifier's interface and tests before the real algorithm, so Task 6 has a target.

**Files:**
- Create: `hq-react/src/render/justifier.ts`
- Create: `hq-react/src/render/justifier.test.ts`

- [ ] **Step 1: Install a test runner**

Run:
```bash
cd hq-react && npx expo install jest jest-expo @types/jest && npm pkg set scripts.test="jest" && npm pkg set jest.preset="jest-expo"
```

- [ ] **Step 2: Define the justifier interface + a failing test**

The justifier is pure: given the line text, a target width, and a function to measure the line's width under a given per-character feature map, it returns per-character features and a font-size ratio. For testability we inject measurement so no Skia is needed in unit tests.

```ts
// justifier.test.ts
import { justifyLine } from "./justifier";

// fake measure: each char is 10 wide; each applied feature adds its value to width.
const fakeMeasure = (
  chars: string,
  features: Map<number, { name: string; value: number }[]>
) => {
  let w = Array.from(chars).length * 10;
  features.forEach((fs) => fs.forEach((f) => (w += f.value)));
  return w;
};

test("short line gets features applied to reach target width", () => {
  const r = justifyLine({ text: "ابت", targetWidth: 60, measure: fakeMeasure });
  // natural width = 3*10 = 30; target 60 → needs +30 from features, none overshooting
  const total = [...r.fontFeaturesByCharIndex.values()]
    .flat()
    .reduce((a, f) => a + f.value, 0);
  expect(30 + total).toBeLessThanOrEqual(60);
  expect(r.fontSizeRatio).toBe(1);
});

test("long line shrinks via fontSizeRatio, no features", () => {
  const r = justifyLine({
    text: "ابتثجحخدذرزسش",
    targetWidth: 60,
    measure: fakeMeasure,
  });
  // natural = 13*10 = 130 > 60 → ratio = 60/130, no features
  expect(r.fontFeaturesByCharIndex.size).toBe(0);
  expect(r.fontSizeRatio).toBeCloseTo(60 / 130);
});
```

- [ ] **Step 3: Write a minimal stub and confirm tests FAIL meaningfully**

```ts
// justifier.ts
export type Feature = { name: string; value: number };
export type JustifyInput = {
  text: string;
  targetWidth: number;
  measure: (chars: string, features: Map<number, Feature[]>) => number;
};
export type JustifyResult = {
  fontFeaturesByCharIndex: Map<number, Feature[]>;
  fontSizeRatio: number;
};

export function justifyLine(_in: JustifyInput): JustifyResult {
  return { fontFeaturesByCharIndex: new Map(), fontSizeRatio: 1 };
}
```

Run:
```bash
cd hq-react && npx jest src/render/justifier.test.ts
```
Expected: the "long line shrinks" test FAILS (ratio is 1, expected ~0.46). This confirms the test harness runs and the contract is real.

- [ ] **Step 4: Commit**

```bash
cd hq-react && git add src/render/justifier.ts src/render/justifier.test.ts package.json && git commit -m "test: define justifier contract with failing tests"
```

---

## Task 6: Implement the clean-room justification algorithm

Reimplement the measure → stretch → re-measure loop from the spec (sourced from Bayaan's `justification-engine.md` *description* and DigitalKhatt's feature tags). NOT copied from Bayaan source.

**Files:**
- Modify: `hq-react/src/render/justifier.ts`
- Modify: `hq-react/src/render/MushafLine.tsx`

- [ ] **Step 1: Implement the two-branch core (shrink vs. stretch)**

```ts
export type Feature = { name: string; value: number };
export type JustifyInput = {
  text: string;
  targetWidth: number;
  measure: (chars: string, features: Map<number, Feature[]>) => number;
};
export type JustifyResult = {
  fontFeaturesByCharIndex: Map<number, Feature[]>;
  fontSizeRatio: number;
};

// DigitalKhatt stretch features, applied in escalating priority.
// Each eligible char may receive a widening variant; we raise values until the
// line fits. This is a simplified clean-room version of the lookup loop
// described in Bayaan's justification-engine.md (not copied from its source).
const STRETCH_TAGS = ["cv01", "cv02", "cv03"];
const MAX_VALUE = 8;

export function justifyLine(input: JustifyInput): JustifyResult {
  const { text, targetWidth, measure } = input;
  const chars = Array.from(text);
  const features = new Map<number, Feature[]>();

  const natural = measure(text, features);

  if (natural >= targetWidth) {
    // Line too long: shrink, no kashida.
    return {
      fontFeaturesByCharIndex: new Map(),
      fontSizeRatio: targetWidth / natural,
    };
  }

  // Line too short: greedily apply widening features to eligible chars
  // (skip spaces) until width reaches targetWidth or features exhausted.
  for (const tag of STRETCH_TAGS) {
    for (let value = 1; value <= MAX_VALUE; value++) {
      for (let i = 0; i < chars.length; i++) {
        if (chars[i] === " ") continue;
        if (measure(text, features) >= targetWidth) {
          return { fontFeaturesByCharIndex: features, fontSizeRatio: 1 };
        }
        const trial = new Map(features);
        trial.set(i, [{ name: tag, value }]);
        if (measure(text, trial) <= targetWidth) {
          features.set(i, [{ name: tag, value }]);
        }
      }
    }
  }
  return { fontFeaturesByCharIndex: features, fontSizeRatio: 1 };
}
```

- [ ] **Step 2: Run the unit tests — expect PASS**

Run:
```bash
cd hq-react && npx jest src/render/justifier.test.ts
```
Expected: both tests PASS (short line reaches ≤ target via features; long line returns ratio ~0.46, no features).

- [ ] **Step 3: Wire real Skia measurement into `MushafLine`**

Replace `MushafLine` so it (a) builds a paragraph applying per-char features and returns `getLongestLine()` as the measurement, (b) calls `justifyLine` with that measure, (c) rebuilds the paragraph applying the returned per-character features (push/add-one-char/pop per char) and `fontSizeRatio`.

```tsx
import React, { useMemo } from "react";
import {
  Paragraph,
  Skia,
  SkTypefaceFontProvider,
  TextDirection,
} from "@shopify/react-native-skia";
import { justifyLine, Feature } from "./justifier";

type Props = {
  words: string[];
  fontMgr: SkTypefaceFontProvider;
  fontSize: number;
  x: number;
  y: number;
  width: number;
};

export function MushafLine({ words, fontMgr, fontSize, x, y, width }: Props) {
  const paragraph = useMemo(() => {
    const text = words.join(" ");
    const chars = Array.from(text);

    const buildWith = (features: Map<number, Feature[]>, size: number) => {
      const b = Skia.ParagraphBuilder.Make(
        { textDirection: TextDirection.RTL },
        fontMgr
      );
      chars.forEach((ch, i) => {
        const f = features.get(i);
        b.pushStyle({
          fontFamilies: ["DigitalKhatt"],
          fontSize: size,
          color: Skia.Color("black"),
          ...(f ? { fontFeatures: f } : {}),
        });
        b.addText(ch);
        b.pop();
      });
      const p = b.build();
      p.layout(width * 2); // wide box; RTL right-aligns within
      return p;
    };

    const measure = (_chars: string, features: Map<number, Feature[]>) =>
      buildWith(features, fontSize).getLongestLine();

    const { fontFeaturesByCharIndex, fontSizeRatio } = justifyLine({
      text,
      targetWidth: width,
      measure,
    });

    return buildWith(fontFeaturesByCharIndex, fontSize * fontSizeRatio);
  }, [words, fontMgr, fontSize, width]);

  return <Paragraph paragraph={paragraph} x={x} y={y} width={width * 2} />;
}
```

- [ ] **Step 4: Run on web and verify justification**

Run:
```bash
cd hq-react && npx expo start --web
```
Expected: the 15 lines now **fill the content width** — left edges aligned to the margin, achieved by letter-widening (look for elongated letter forms), not by big gaps between words. Compare line fullness against Task 4's ragged output.

- [ ] **Step 5: Commit**

```bash
cd hq-react && git add -A && git commit -m "feat: justify mushaf lines via clean-room DigitalKhatt feature stretching"
```

---

## Task 7: Visual verification against a reference page

**Files:** none (verification only)

- [ ] **Step 1: Obtain a reference image of page 5**

Open a known-good QPC v2 page 5 in a browser tab (e.g. quran.com page 5, or a printed-mushaf scan). Keep it side-by-side with the running app.

- [ ] **Step 2: Compare line-by-line**

Checklist (judge by eye):
- Same 15 lines, same line breaks (each line ends on the same word).
- Each line fills both margins.
- Letter stretching looks calligraphic, not distorted (no broken joins, no fake gaps).
- Overall page shape recognizable as the mushaf page 5.

- [ ] **Step 3: Record the outcome**

If it matches: the spike's Definition of Done is met — update the spec file's Status (`Proposed` → `Validated`) and stop. If specific lines are off, note which (the spec's "What to Inspect for Visual Misalignment" list — fontSize scale, interline, justifier acceptance conditions — is the debugging starting point), and iterate on Task 6's algorithm. Do not declare done until the visual match holds.

---

## Self-Review Notes

- **Spec coverage:** goal, scope, legal constraint, architecture (MushafPage/MushafLine/Justifier), hardcoded fixture, data flow, the linchpin risk (Task 2), build order (Tasks 2/4/6/7) — all mapped to tasks.
- **Linchpin first:** the riskiest unknown (do features apply on web) is Task 2, before any engine work, per the spec.
- **Type consistency:** `justifyLine`, `JustifyInput`, `JustifyResult`, `Feature {name,value}`, `fontFeaturesByCharIndex`, `fontSizeRatio` are used identically across Tasks 5 and 6 and consumed by `MushafLine` in Task 6.
- **Justifier simplification (intentional, not a placeholder):** Task 6 is a *simplified* clean-room version (escalating cv01–cv03 values). The spec calls the full Bayaan lookup ordering "intricate"; for a one-page spike we start simple and escalate only if Task 7's visual match demands it.
- **Fixture (Task 3):** the only task requiring offline judgment (locating the 130-word verse span, cross-checked against a reference image). Its template placeholders MUST be replaced with real words before its commit — flagged explicitly in the task.
