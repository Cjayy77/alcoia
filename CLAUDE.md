# CLAUDE.md

Repository context for agents. Read fully before any task. This file describes the codebase **as it actually is today**, not as it should be.

---

## Product intent

TL;DR is a Chrome MV3 extension that notices when a reader is struggling with a page and intervenes. It is **not** primarily an eye-tracking product, despite what the current code structure implies.

Signal hierarchy, in order of authority:

1. **Reader responses** — answers to retrieval questions. Not yet built. The only ground truth in the system.
2. **Browser telemetry** — reading rate vs. text difficulty and personal baseline, scroll regressions, selection, blur, idle. Precise, always available, no permission needed. **Partly built** — see `comprehension-monitor.js`.
3. **Webcam gaze** — coarse presence and region only. Currently the primary path, which is wrong. ~180 px error; several lines of text.

If a task appears to invert this hierarchy, stop and ask.

---

## Actual repository state

```
TL_DR/
├── manifest.json              MV3, v0.1.0
├── background.js              105 lines — service worker
├── src/content/
│   ├── content.js             ~1700 lines — MONOLITH, see below
│   ├── state-engine.js         ~250 — signal fusion, single state estimate (P1)
│   ├── intervention-policy.js  ~130 — interruption budget, one place (P1)
│   ├── gaze-utils.js           397 — WebGazer smoothing/EMA
│   ├── reading-calibration.js  268 — WPM calibration flow
│   ├── classifier.js           251 — GENERATED decision tree
│   ├── reading-map.js          248 — right sidebar minimap
│   ├── comprehension-monitor.js 218 — telemetry detector (the good one)
│   ├── gaze-features.js        188 — 9-feature extractor
│   ├── webgazer-bootstrap.js   183 — main-world injection
│   ├── dyslexia-utils.js       136
│   ├── idle-overlay.js         133
│   ├── overlay-utils.js        132
│   ├── pdf-handler.js          130
│   ├── focus-ruler.js          129
│   ├── tts-handler.js          126
│   ├── lang-detect.js          125
│   ├── pptx-handler.js          78
│   ├── session-tracker.js       76
│   └── sra-page-bridge.js       41 — postMessage bridge, isolated↔main world
├── src/popup/                 popup.js 357, notes.js 197, + 6 HTML pages
├── src/libs/                  webgazer.min.js, pdfjs, jszip, Merriweather woff2
└── server/index.js            269 lines — Express + Groq proxy
```

**Absent:** any build step, `_locales/`, CI, CONTRIBUTING, ESLint.

**Test suite exists as of P1.** `npm test` (Vitest) at the repo root — 60 tests over the state
engine, the interruption budget, the fusion of both pipelines, and the missing-key trap.
`npm run lint` still does not exist; do not cite it as passing.

**Added in the P0 pass:** `LICENSE` (AGPL-3.0, repo root), `NOTICE.md` (licence scope + bundled
third-party licences), `PRIVACY.md` (**scaffold with TODO markers only — not publishable**).

**Licensing constraint discovered in P0:** `src/libs/webgazer.min.js` is **GPLv3** (LGPLv3 only for
companies valued under $1M). The shipped extension is therefore a combined copyleft work — full
corresponding source must be offered to anyone who receives it. AGPL-3.0 for the client stays
compatible, but the paid-tier plan interacts with WebGazer's $1M threshold. See `NOTICE.md`.

---

## How it currently works

### Two parallel pipelines — fused in P1

**This was the central architectural problem. It is now fixed.** Both pipelines feed
`state-engine.js`, which produces one state, and `intervention-policy.js` decides whether that
state earns an interruption. `content.js` has exactly one subscriber that can put something in
front of the reader. The description below is kept because the two detectors still exist and
still produce their own signals — what changed is that neither can act on its own.

The hierarchy is enforced structurally, not by convention: telemetry may assert any state; gaze
may assert **only** `absent`, and may corroborate a telemetry state to raise confidence. A gaze
label of `confused` with nothing behind it resolves to `unknown` and is dropped. Covered by
`tests/fusion-integration.test.js`.

**Pipeline A — gaze.** `webgazer-bootstrap.js` injects WebGazer into the page's main world; `sra-page-bridge.js` relays samples back to the isolated content script via postMessage. `gaze-features.js` buffers points over a 2500 ms window, runs DBSCAN, computes 9 features. `content.js` (~line 1226) gates on `rawFeatures.gaze_quality < 0.25`, calls `classifyGazeState()`, smooths through a ring buffer (`getSmoothedState`, line 119), then looks up `COGNITIVE_STATE_ACTIONS` and fires `explain` / `simplify` / `nudge`.

**Pipeline B — telemetry.** `comprehension-monitor.js` runs independently. On paragraph exit (`content.js` ~line 1100) it emits `speed_mismatch` or `backtrack` signals, which fire their *own* popups via a separate handler (~line 1118).

~~Both can interrupt the same reader with no coordination.~~ Both now route through the engine and
share one budget. `handleComprehensionSignal()` is a **renderer only** — never call it from a
detector. New detectors call `stateEngine.update({ telemetry: signal })` and nothing else.

### `comprehension-monitor.js` — the asset

Already implements: Flesch-Kincaid per paragraph, expected reading time from word count × difficulty, a **personal WPM baseline** as a running median persisted in `chrome.storage` and seedable from `reading-calibration.js`, speed-mismatch detection in both directions, and scroll backtrack.

This is the correct primary sensor. It needs promoting, not rewriting. Note it correctly skips FK on non-English pages — which means **non-English pages currently have almost no primary signal.**

### Server

Single endpoint `POST /api/summarize` with modes, plus `/demo` and `/health`. Uses `llama-3.1-8b-instant`, escalating to `llama-3.3-70b-versatile` for some requests (line ~180). Rate-limited. There is **no question-generation endpoint** — that is the main gap.

---

## Known defects — verified by reading the code

### `gaze-features.js`

- **Line ~104:** fixation/saccade boundary is `dist > 30` px. WebGazer's error is ~180 px, so the threshold sits entirely inside the noise floor. Nearly every consecutive sample pair registers as a saccade. `saccade_length`, `saccade_std`, `velocity_mean` and `regression_rate` are measuring tracker jitter, not eye movement. No threshold tuning fixes this; the information is not in the signal.
- **Line ~84:** `lineBand = Math.round(pt.y / 20)` uses raw viewport `y` with no scroll offset. Scrolling changes the band with zero eye movement. `line_reread_count` does not count re-reads.
- **Fallback constants** (`200`, `50`, `80`, `30`, `0.1`, `200`) are all focused-looking. When data is sparse the extractor fabricates plausible focused-reading features instead of abstaining.
- **DBSCAN `eps: 80`** is below the tracker error, so it clusters noise. The documented fallback comment — "a noisy classification is better than no classification" — is backwards for a system that acts on its output.
- **Keep:** `gaze_drift_px`. Dispersion is the one gaze measure with consistent support in the literature.

### `classifier.js`

- Header records `Test accuracy: 0.851`, now carrying the full synthetic-data qualifier (P0 pass). Trained on **synthetic** data: 2500 clean rows (500/class), each duplicated with Gaussian noise → 5000 rows, 4000 train / 1000 test. That number measures recovery of the generator's rules, not reading behaviour. Several leaves have `confidence: 1.000` — an overfitting signature on synthetic data.
- **The 0.851 is additionally inflated by train/test leakage.** The augmented rows are noise-perturbed duplicates of the clean rows and the split is random, so a row's noisy twin can sit in test while the original sits in train. Any honest retrain must split *before* augmenting.
- The **root split is `scroll_delta_px`** — the one feature that isn't gaze. The model is already telling you telemetry carries the information.

### `manifest.json`

- ~~`<all_urls>` + `all_frames: true` + `match_about_blank` will draw manual Web Store review.~~ **Fixed in P0** — `all_frames` is now `false` and `match_about_blank` is removed. `<all_urls>` remains (the extension has to read arbitrary pages) and still invites review.
- `web_accessible_resources` exposes `src/content/**` to `<all_urls>`, letting any page enumerate the extension's modules. Worth narrowing; not yet done.

### `content.js`

1708 lines in one IIFE (`__sra_main`). Contains: constants, runtime state, highlight persistence, state smoothing, module loader, settings, dark mode, AI fetch, popup positioning and rendering, keyboard shortcuts, the classify loop, the comprehension handler, and scroll listeners. Split into `orchestrator.js`, `ui-controller.js`, `state-engine.js`, `intervention-policy.js`. Add new logic to the new modules, never to `content.js`.

---

## THE TRAP — read this before touching features

`classifier.js` branches on `f.saccade_length`, `f.saccade_std`, `f.velocity_mean`.

Remove those keys from the extractor without retraining and every comparison becomes `undefined <= X`, which evaluates to `false` **without throwing**. The classifier will not crash. It will silently route down one branch forever and keep emitting confident labels.

Verified empirically — `classifyGazeState({})`, with **no features at all**, returns
`{ label: 'skimming', confidence: 0.722 }`. Dropping the three saccade/velocity keys from a
focused-reading sample flips `focused (0.993)` to `skimming (1.000)`. Confident labels, zero data.

**Required sequence:** retrain in `tldr classifier/tldr_classifier_training(1).ipynb` against the reduced feature set, regenerate `classifier.js`, *then* delete from the extractor. Or add a guard that throws on any missing key. Do not delete first. Write a test for this exact failure mode.

⚠️ **Use the right notebook.** The tree currently shipping was exported from
`tldr_classifier_training(1).ipynb` (noise-augmented, 5000 rows, 4000 train). It is *not* the
export from `tldr_classifier_v2.ipynb`, which produces a different tree. Retraining in the wrong
notebook silently replaces the model. Confirmed by diffing the shipped file against both exports:
the body matches `tldr classifier/classifier(1).js`.

---

## Hard invariants — never violate, never make configurable

1. **No covert monitoring.** No hidden mode, no silent-collection flag, no admin override that observes a reader without their action. Not behind a feature flag, not for a paying institution, not for testing.
2. **Video never leaves the device.** No frames, no raw camera data, no base64 images in any network request.
3. **No accuracy claims** in code, comments, UI, or docs. See below.
4. **Receipts are reader-generated only.** No background submission. The reader sees the contents before sharing.
5. **`unknown` is a valid, correct, common state.** Never interrupt on it. Never substitute plausible defaults for missing data.
6. **No raw gaze coordinates** in any persisted or transmitted artifact. Aggregates only.

---

## Claims discipline

Four different accuracy figures exist across this project's materials: 0.851 (`classifier.js`), 88% (notebooks), 92% (report), and "75–82% real-world" (marketing drafts). All derive from synthetic data or from nowhere. **No real-participant evaluation has been performed.**

- Never introduce, restore, or repeat an accuracy percentage.
- Where one exists, it must carry the qualifier that it measures a synthetic generator, not reading.
- If asked for copy implying detection reliability, write mechanism descriptions and flag the request.

---

## Decisions already made — do not re-litigate

- **Camera off by default.** Requesting webcam at install destroys conversion.
- **Telemetry is the primary path.** Gaze is a secondary sensor contributing presence and coarse region only.
- **Questions, not summaries, are the primary intervention.** Explanation is the fallback after a wrong answer. Follows D'Mello et al. (2016), whose RCT found just-in-time questioning recovered comprehension losses (d = 0.47). Summarising removes the desirable difficulty that produces retention.
- **State names describe observations:** `on_pace`, `skimming`, `struggling`, `drifting`, `absent`, `unknown`. Do not reintroduce `confused` or `overloaded` — they are unmeasurable internal states.
- **AGPL-3.0 for the client; `server/` moves to a separate private repo.**
- Fonts: **Literata** (body) + **Inter** (UI). Fraunces and Merriweather are being retired.

---

## Requires human approval — scaffold only, do not author

- `PRIVACY.md`, terms, DPA, refund policy. Generate structure and TODO markers only. Generated legal text will be plausible and wrong about actual data flows.
- Any pricing figure.
- Any change to the invariants above.
- Deleting or replacing the trained classifier.
- Anything touching payments or user PII.

Stop and ask rather than proceeding.

---

## Interruption policy

Reader attention is the scarcest resource here. A wrong interruption is worse than a missed one.

- Max one per 3 minutes; max five per session; never twice on the same paragraph; never on `unknown`.
- Every interruption carries user-visible evidence — "You slowed down a lot here" — converting an inference into an observation.

**Enforced in `intervention-policy.js` as of P1**, with tests. Two notes for anyone changing it:

- `evaluate()` decides, `record()` spends. Call `record()` only once the interruption is actually
  on screen, or a decision dropped downstream silently burns the budget.
- Skimming interrupts only on `difficult` / `very_difficult` text. Skimming easy prose is a choice,
  not a problem.

---

## Conventions

- ES modules; no bundler-specific syntax in content scripts. Modules loaded dynamically via `loadModule()` in `content.js`.
- New telemetry detectors go in `src/content/telemetry/`, each exporting `{ update(), signal() }`.
- Keep modules under ~300 lines.
- Tests in Vitest. Priority: feature extractors against fixtures, state engine against synthetic sequences, interruption budget, and the missing-key guard above.
- No new runtime dependencies without asking.

---

## Verification

After any change:

```bash
npm run lint && npm test
```

Then load unpacked and confirm manually:
- Loads with no console errors on a plain article page
- Reading detection runs with the camera **off**
- No `getUserMedia` call unless the user explicitly enabled the camera
- No network request contains image or video data

Report what you verified, not what you believe should work.
