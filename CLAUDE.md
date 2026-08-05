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
│   ├── state-engine.js         ~330 — signal fusion, single state estimate (P1/P2)
│   ├── intervention-policy.js  ~130 — interruption budget, one place (P1)
│   ├── telemetry/              P2 detectors — each exports { update(), signal() }
│   │   ├── paragraph-tracker.js   ~110 — viewport-driven paragraph timing
│   │   ├── scroll-regression.js    ~75 — paragraph-index returns + latency signature
│   │   ├── interaction-signals.js  ~95 — selection, copy, blur/return
│   │   ├── scroll-dynamics.js      ~60 — scroll jerk
│   │   ├── text-difficulty.js     ~135 — FK + syntactic load, works non-English
│   │   ├── residual-distribution.js ~55 — per-reader pace thresholds
│   │   ├── cursor-tracking.js      ~95 — mouse as reading pointer when it is one
│   │   └── progression-entropy.js  ~80 — session shape (for the P4 receipt)
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
`npm run test:browser` loads the extension unpacked in Chromium and runs the verification
checklist below. `npm run lint` still does not exist; do not cite it as passing.

**Verified in a real browser (P1):** content script injects with no page errors; telemetry-only
detection reaches the reader with the camera off (`State: struggling (conf 0.60, camera 0.00)`);
zero `getUserMedia` calls when the camera is off; no image or video data in any request.
**Not verified:** the gaze path end to end — WebGazer fetches its face detector from `tfhub.dev`,
which this environment blocks, so `webgazer.begin()` never reaches the camera here.

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

Already implements: difficulty per paragraph, expected reading time from word count × difficulty, a **personal WPM baseline** as a running median persisted in `chrome.storage` and seedable from `reading-calibration.js`, speed-mismatch detection in both directions, and scroll backtrack.

This is the correct primary sensor. It was promoted in P2, not rewritten. Two changes:

- **Difficulty now comes from `telemetry/text-difficulty.js`** — Flesch-Kincaid weighted 0.6 with
  syntactic structure (clause length, sentence length, subordination, passives) at 0.4. On
  non-English pages FK is skipped and structure carries the whole score. ~~Non-English pages have
  almost no primary signal.~~ They now get one; previously every FK-based check was skipped and
  those pages produced nothing but scroll backtrack.
- **Thresholds are per-reader.** After ~8 paragraphs, `telemetry/residual-distribution.js` judges
  "too fast"/"too slow" by z-score against the reader's own spread of reading-rate residuals
  instead of the fixed 0.30/0.50 ratios, which fall back only until there is enough history. A
  reader who consistently runs at 0.6x the model is not struggling on every paragraph, and the
  fixed cutoff said they were.

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
- **Added in P2:** `on_page_fraction` (share of samples inside the padded text-column rect) and
  `face_present`, plus a `presence()` method that reports even below `MIN_POINTS` — `computeFeatures()`
  returns null there, so absence was previously unobservable from the extractor. Both abstain rather
  than defaulting: `on_page_fraction` is `null` when there is no content rect. These are the only
  gaze questions a ~180px tracker can answer honestly, and they are what the engine consumes.
- **Still not done — blocked.** Deleting `saccade_length`, `saccade_std` and `velocity_mean` requires
  retraining first (see THE TRAP), and retraining replaces the classifier, which needs human
  approval. `line_reread_count` is superseded by `telemetry/scroll-regression.js` for decisions, but
  the feature still exists and still feeds the classifier — removing it is part of the same blocked
  deletion.

### `classifier.js`

- Header records `Test accuracy: 0.851`, now carrying the full synthetic-data qualifier (P0 pass). Trained on **synthetic** data: 2500 clean rows (500/class), each duplicated with Gaussian noise → 5000 rows, 4000 train / 1000 test. That number measures recovery of the generator's rules, not reading behaviour. Several leaves have `confidence: 1.000` — an overfitting signature on synthetic data.
- **The 0.851 is additionally inflated by train/test leakage.** The augmented rows are noise-perturbed duplicates of the clean rows and the split is random, so a row's noisy twin can sit in test while the original sits in train. Any honest retrain must split *before* augmenting.
- The **root split is `scroll_delta_px`** — the one feature that isn't gaze. The model is already telling you telemetry carries the information.

### `manifest.json`

- ~~`<all_urls>` + `all_frames: true` + `match_about_blank` will draw manual Web Store review.~~ **Fixed in P0** — `all_frames` is now `false` and `match_about_blank` is removed. `<all_urls>` remains (the extension has to read arbitrary pages) and still invites review.
- `web_accessible_resources` exposes `src/content/**` to `<all_urls>`, letting any page enumerate the extension's modules. Worth narrowing; not yet done.

### `content.js`

~~Paragraph tracking is gaze-driven.~~ **Fixed in P2.** `telemetry/paragraph-tracker.js` picks the
paragraph crossing the reading line (0.4 of viewport height) and `syncParagraph()` in `content.js`
drives `enterParagraph`/`leaveParagraph` from scroll, focus and a 5s tick. `onGaze()` no longer owns
paragraph timing — it only refines which element is under the reader. Verified camera-off in the
browser: `speed_mismatch` and `regression` both fire now, where previously only scroll backtrack did.

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
  Register the signal type in `state-engine.js`: assertable types get a branch in `fromTelemetry()`
  with a confidence and an evidence sentence; corroboration-only types go in `CORROBORATING_TYPES`
  plus `CORROBORATION`, and a `CORROBORATION_GUARD` if the signal is only meaningful for one
  subtype. An unregistered type is silently ignored — `fromTelemetry()` returns null for it.
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

---

## Third-party licensing — read before adding, removing or replacing dependencies

**`src/libs/webgazer.min.js` is GPLv3.** Companies have an LGPLv3 option only while valuation is
under $1,000,000.

Consequences, in order of importance:

1. **While WebGazer ships inside the extension package, the client can never be closed-source.**
   The shipped extension is a combined copyleft work. This is permanent and is the only
   consequence that constrains future decisions.
2. **AGPL-3.0 is compatible** — §13 of GPLv3 and §13 of AGPLv3 each permit the combination. The
   current LICENSE is correct and does not need changing.
3. **The $1M threshold is irrelevant to this project.** The LGPL option exists to allow
   proprietary linking. This project is copyleft by choice, so crossing the threshold simply
   leaves it under GPLv3, which is where it already is. Do not treat the threshold as a blocker
   on billing, pricing or incorporation.
4. **The server is unaffected.** It is a separate program communicating over a network API in a
   separate process. WebGazer's copyleft does not reach it. Keeping it in a private repo is
   unaffected by this.
5. **AGPL's network clause does almost nothing here.** A browser extension runs on the user's
   machine; the user already receives the code, and nobody deploys an extension as a network
   service. The protection comes from ordinary distribution copyleft. Do not cite the network
   clause as a deterrent in any documentation.

**WebGazer is unmaintained.** Official maintenance ended 24 February 2026. It remains functional;
updates are not guaranteed.

**Exit path, logged but not scheduled.** Gaze is now demoted to presence and coarse region. A small
MediaPipe FaceMesh implementation (Apache 2.0) would cover what remains, remove the GPL obligation,
and drop a dead dependency. Do not undertake this without asking.

**Fonts.** ~~Merriweather (`src/libs/`) is SIL OFL 1.1 and needs `OFL.txt` shipped alongside.~~
**No font is bundled.** The two `Merriweather-*.woff2` files were 82-byte text placeholders, not
fonts, and nothing referenced Merriweather anywhere in the codebase; they have been removed. There
is nothing to license yet — add `OFL.txt` alongside the first real font binary. Fraunces is
currently fetched from `fonts.googleapis.com` on every page the reader visits. See `NOTICE.md`.

---

## Accuracy figures — corrected count

At least eight figures exist across the project, not four:

| Figure | Location |
|---|---|
| 0.851 | `src/content/classifier.js` (shipped) |
| 0.909 | `tldr classifier/classifier.js` |
| 88% | notebooks |
| 91.5% | notebooks |
| ~65–70% | notebooks |
| 0.835 / 0.884 / 0.897 | notebook outputs |
| 92% | report |
| "75–82% real-world" | marketing drafts |

**The shipped figure is inflated by construction, not merely synthetic.** The training notebook
duplicates every row with Gaussian noise and then performs a random 80/20 split, so a row's noisy
twin sits in test while the original sits in train. This is train/test leakage. 0.851 is therefore
not a valid measure even of how well the tree recovers its own generator's rules. The qualifier in
`classifier.js` says so.

---

## The trap — now empirically confirmed

```
classifyGazeState({})                   → { label: 'skimming', confidence: 0.722 }
focused sample, 3 saccade keys removed  → focused (0.993) becomes skimming (1.000)
```

Confident labels from zero features, no exception thrown. Any work that touches the feature set
must land the missing-key guard first. Pinned by `tests/classifier-missing-keys.test.js`.

---

## Revised phase order

`package.json`, Vitest and the missing-key guard test come **before** P1, not with P6 — the
Verification section of this file was unexecutable until they existed. Done as of P1.

Order: P0 (done) → test harness (done) → P1 (done) → P2 (done) → P3 → P6 split → ship → P7 → P4 → P5.

---

## Working agreement with the repository owner

- Open a PR for each phase. Reviewing a diff is how the owner retains control of a codebase an
  agent is writing. Do not push directly to main.
- One phase per session. If a diff cannot be read in one sitting, the phase was scoped too large —
  split it.
- Continue reporting what was verified versus what was assumed, and continue flagging where the
  brief conflicts with the code. That behaviour is correct and should not be moderated.
