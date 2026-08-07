# CLAUDE.md

Repository context for agents. Read fully before any task. This file describes the codebase **as it actually is today**, not as it should be.

---

## Product intent

Alcoia is a Chrome MV3 extension that notices when a reader is struggling with a page and intervenes. It is **not** primarily an eye-tracking product, despite what the current code structure implies.

Signal hierarchy, in order of authority:

1. **Reader responses** — answers to retrieval questions. **Built in P3** (`telemetry/response-signals.js`). The only ground truth in the system, and the engine gives them confidences above anything telemetry can produce, so an answer decides the state. A correct answer resolves to `on_pace` and stops the system pressing; a dismissal asserts nothing at all, because declining to be tested says nothing about comprehension.
2. **Browser telemetry** — reading rate vs. text difficulty and personal baseline, scroll regressions, selection, blur, idle. Precise, always available, no permission needed. **Partly built** — see `comprehension-monitor.js`.
3. **Webcam gaze** — coarse presence and region only. Currently the primary path, which is wrong. ~180 px error; several lines of text.

If a task appears to invert this hierarchy, stop and ask.

---

## Actual repository state

```
alcoia/
├── manifest.json              MV3, v0.2.0
├── background.js              105 lines — service worker
├── src/content/
│   ├── content.js            ~1435 lines — host: UI glue, settings, boot (P6)
│   ├── orchestrator.js         ~360 — detection pipeline, engine + budget (P6)
│   ├── ui-controller.js        ~363 — popups, highlight, toasts, dark mode (P6)
│   ├── question-card.js        ~160 — the retrieval question card (P3)
│   ├── receipt.js              ~290 — reader-owned session record + preview (P4)
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
├── src/popup/                 popup.js ~330, notes.js 197, + 6 HTML pages
├── src/styles/
│   ├── fonts.css              ~105 — @font-face + the three type tokens (UI pass)
│   ├── overlay.css            ~430 — everything drawn on top of a page the reader is reading
│   ├── panel.css              ~175 — shared chrome for the extension's own pages (UI pass)
│   └── popup.css               ~65 — legacy, only upgrade.html still links it
├── src/libs/                  webgazer.min.js (GPLv3), pdfjs, jszip, fonts/ (SIL OFL 1.1)
├── server/index.js            327 lines — Express + Groq proxy
├── server/questions.js        ~185 — question generation + validation (P3, pure)
└── server/receipt-signing.js   ~85 — HMAC tamper-evidence for receipts (P4, pure)
```

**Absent:** any build step, `_locales/`, CI, CONTRIBUTING.

**The product is Alcoia.** Renamed from TL;DR across code, comments, UI, docs, the manifest,
the log prefix and the extension directory (`TL_DR/` → `alcoia/`), plus the CSS tokens
(`--tldr-*` → `--alc-*`) and the icon. Two prefixes were deliberately left alone:

- **`sra_` storage keys.** Renaming them silently discards every existing reader's settings
  unless a migration ships alongside, and a migration is a thing to get right on purpose, not
  a side effect of a rebrand.
- **`sra-` CSS class and element-id prefixes.** ~300 occurrences across CSS, JS, tests and the
  browser check, with no user-visible payoff — nobody reads a class name. Worth doing as its
  own mechanical pass if it ever bothers you; not worth the chance of missing one now.

`mode=tldr` in the summarise request is an API contract with `server/index.js` and is not a
brand string. It was left as is.

**Deleted in the UI pass, both dead:** `src/content/overlay.css` and `src/popup/popup.css`.
Nothing loaded either. The first one mattered — it was where the P3 question card
(`.sra-q-*`) and the P4 receipt (`.sra-receipt*`) rules lived, while `content.js` has only ever
injected `src/styles/overlay.css`. **The primary intervention and the receipt were both
rendering completely unstyled in the browser, and every test passed**, because the tests assert
on structure and behaviour and never on whether a rule applies. Those rules now live in
`src/styles/overlay.css`. Two stylesheets with the same name in different directories is how
this hid; do not recreate the arrangement.

**The browser check now runs anywhere.** `tests/browser/smoke.mjs` had absolute Linux paths for
the extension directory and the Chromium binary, so it could only run on one machine; both are
derived now (`EXT` / `CHROME` env vars still override). Ran clean on Windows.

**Test suite exists as of P1.** `npm test` (Vitest) at the repo root — 216 tests over the state
engine, the interruption budget, pipeline fusion, the P2 detectors, question generation and
validation, response signals, session recall, the receipt and its signing, the UI controller,
and the missing-key trap.
`npm run test:browser` loads the extension unpacked in Chromium and runs the verification
checklist below. **`npm run lint` exists as of P6** (ESLint, flat config) and exits 0 — it is a
defect linter, not a style linter, so warnings in untouched files are left visible on purpose.

**Verified in a real browser (P1):** content script injects with no page errors; telemetry-only
detection reaches the reader with the camera off (`State: struggling (conf 0.60, camera 0.00)`);
zero `getUserMedia` calls when the camera is off; no image or video data in any request.
**Not verified:** the gaze path end to end — WebGazer fetches its face detector from `tfhub.dev`,
which this environment blocks, so `webgazer.begin()` never reaches the camera here.

**Added in the P0 pass:** `LICENSE` (AGPL-3.0, repo root), `NOTICE.md` (licence scope + bundled
third-party licences), `PRIVACY.md` (**scaffold with TODO markers only — not publishable**).

**Licensing constraint discovered in P0:** `src/libs/webgazer.min.js` is **GPLv3**. The shipped
extension is therefore a combined copyleft work — full corresponding source must be offered to
anyone who receives it, and the client can never be closed-source while WebGazer ships in the
package. AGPL-3.0 stays compatible. ~~The paid-tier plan interacts with WebGazer's $1M
threshold.~~ **It does not** — see the licensing section at the end of this file. `NOTICE.md` has
the detail.

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

`POST /api/summarize` with modes, plus `/demo` and `/health`. Uses `llama-3.1-8b-instant`,
escalating to `llama-3.3-70b-versatile` for some requests. Rate-limited.

~~There is **no question-generation endpoint** — that is the main gap.~~ **`POST /api/questions`
added in P3.** The logic lives in `server/questions.js` as a pure CommonJS module with no express
dependency, so it is testable without standing a server up (`tests/questions.test.js`).

The hard requirement is `span`: every question must cite, **verbatim**, the sentence in the
passage containing its answer. Spans that do not appear in the passage are rejected outright — a
model that cannot point at its evidence invented the question, and an invented question asked of
a struggling reader is worse than none. When nothing survives validation the endpoint returns 422
and the client falls back to an explanation. Responses are cached by content hash (LRU, 24h) so
the same paragraph costs one generation, and questions get their own tighter rate-limit bucket
(10/min) since they cost more than summaries.

Two prompts previously asserted "the reader's eye movements indicate they are confused". With the
camera off by default that is usually false, and it is a detection claim embedded in a prompt.
Both now describe the observation (slowed down, went back) rather than the sensor.

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

~~1708 lines in one IIFE.~~ **~1435 as of P6**, after two extractions:

- **`orchestrator.js` decides.** It owns the telemetry detectors, the state engine, the
  interruption budget, the one engine subscriber and the gaze classify loop. It does not render:
  when an interruption is allowed it calls `host.onIntervention()` and takes a boolean back
  saying whether anything reached the screen. The budget is spent only on a yes.
- **`ui-controller.js` renders.** It owns `openPopups` — nothing else may mutate that map,
  because the dedup and the `MAX_POPUPS` eviction both depend on it being the only record of
  what is on screen. Create cards through `reservePopup()` / `showPopup()`, never by hand.

`content.js` is now the host: module loading, settings and their storage listener, the AI fetch,
text highlighting, word lookup, selection handling, keyboard shortcuts, WebGazer bootstrap, SPA
navigation, session continuity, and the render callbacks the orchestrator calls.

Settings still live in `content.js` as loose `let`s, so both modules read them through accessors
(`settings()` / `getSettings()`) rather than capturing copies — the storage listener reassigns
them at runtime and a captured copy goes stale silently. Extracting a real settings module is
the obvious next cleanup.

Add new logic to the new modules, never to `content.js`.

**Fixed in the UI pass**, all in `content.js` unless noted:

- The Google Fonts `<link>` injected into every page is gone; `src/styles/fonts.css` is injected
  instead, ahead of `overlay.css` so `@font-face` is registered before anything asks for it.
- `sra_eye` defaults to `false` (also `popup.js` and `background.js`) — see the camera note under
  *Decisions already made*.
- The simulate path (`Alt+1`–`5` and the `simulateState` message) speaks the engine's vocabulary.
  Both used to send `confused` / `overloaded` / `zoning_out` straight into
  `COGNITIVE_STATE_ACTIONS`; they now go through one `runSimulatedState()` helper that translates
  to the classifier's older action keys in a single place. `Alt+2` still reaches the `simplify`
  renderer, which nothing else exercises.
- The image-dwell branch tested `['confused','overloaded'].includes(lastCogState)`, which no
  longer matched anything the engine emits — it had been silently dead since the state rename.
  Now `lastCogState === 'struggling'`.
- `reading-map.js` `EVENT_COLOR` was keyed on the same dead names, so map markers had no colour.

**Fixed in the telemetry pass** — three features that were quietly camera-only:

- **`focus-ruler.js` followed gaze Y and nothing else.** With the camera off — the default —
  `update()` was never called, so the band sat wherever it was last left. It now resolves its
  position from the freshest available source: gaze (< 2s old) → cursor (< 4s old, and only
  when the pointer is inside body text) → the reading line at 0.4 × viewport height, the same
  anchor `telemetry/paragraph-tracker.js` uses. The reading line is always available and is the
  correct behaviour on touch, so it is the floor rather than the failure case. `BAND_BY_STATE`
  moved to the engine's state names.
- **Reading calibration bailed out without WebGazer.** The gaze flow exists to generate
  training points, but the number the feature is *for* is the reader's WPM baseline, which
  needs no sensor: show a passage of known length, let them read it, ask when they are done.
  `runSelfPacedCalibration()` in `reading-calibration.js` does that, and implausible results
  (< 70 or > 1200 wpm) are rejected with a reason rather than seeding a baseline that would
  mis-judge every later paragraph. **Related bug:** `content.js` wrote `sra_baseline_wpm` only
  inside the gaze-baseline branch, so with the camera off the measured number never survived
  the page unload. It is persisted on its own now.
- **Reading modes no longer set `sra_eye`.** Switching to "Study" used to turn the webcam on.

**Card placement.** `placePopup()` used to drop the card on top of the passage whenever no side
had room at full width — so on a wide-column page the question covered the paragraph it was
asking about. It now tries narrowing into the larger side margin first (down to 262px) and only
overlaps as a genuine last resort. Below a 560px viewport it stops dodging altogether and
becomes a bottom sheet (`.sra-sheet`); on a phone there is no margin to dodge into, and a
predictable position beats a clever one.

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

- **Camera off by default.** Requesting webcam at install destroys conversion. **This is now
  true in code as well as in principle** — `sra_eye` defaulted to `true` in `content.js`, in the
  popup and in the service worker's guard, so a fresh profile started WebGazer on the first page
  it saw. All three now default to `false`, and reading modes no longer set the key at all:
  switching to "Study" must never start a webcam.
- **Telemetry is the primary path.** Gaze is a secondary sensor contributing presence and coarse region only.
- **Questions, not summaries, are the primary intervention.** Explanation is the fallback after a wrong answer. Follows D'Mello et al. (2016), whose RCT found just-in-time questioning recovered comprehension losses (d = 0.47). Summarising removes the desirable difficulty that produces retention. **Implemented in P3** — `STATE_ACTIONS` maps `struggling` and `skimming` to `ask`, and the explanation card is reached only after a wrong answer or when no question could be generated.
- **State names describe observations:** `on_pace`, `skimming`, `struggling`, `drifting`, `absent`, `unknown`. Do not reintroduce `confused` or `overloaded` — they are unmeasurable internal states.
- **AGPL-3.0 for the client; `server/` moves to a separate private repo.**
- Fonts: **Source Serif 4** (reading voice) + **Plus Jakarta Sans** (UI voice), both bundled in
  `src/libs/fonts/` under SIL OFL 1.1. Fraunces is retired; nothing is fetched from Google any
  more. ~~Literata (body) + Inter (UI).~~ Superseded by the owner's direction: something in
  Merriweather's family for reading, Jakarta for the secondary voice. Times New Roman was
  considered for the reading voice and rejected — small x-height and thin strokes at 13px, and,
  being the browser's default serif, it reads as a page whose stylesheet failed to load. Source
  Serif 4 is the screen-drawn face closest to what was asked for. **Both faces are reached only
  through `--alc-serif` / `--alc-ui` in `src/styles/fonts.css`; nothing hard-codes a family, so
  changing the pairing is a two-line edit.**

---

## Requires human approval — scaffold only, do not author

- `PRIVACY.md`, terms, DPA, refund policy. Generate structure and TODO markers only. Generated legal text will be plausible and wrong about actual data flows.
- Any pricing figure.
- Any change to the invariants above.
- Deleting or replacing the trained classifier.
- Anything touching payments or user PII.

Stop and ask rather than proceeding.

---

## The receipt (P4)

`receipt.js` builds a reader-owned record and shows it in a preview panel. Alt+I, or the
message API. Nothing runs on a timer and nothing leaves the machine without a click.

**What a signature proves, exactly.** A valid signature means the receipt is byte-for-byte what
the server issued. It proves nothing about whether the reading happened — the figures come from
the reader's own browser, and a modified extension could send anything. Wording shown to anyone
must say "unaltered since issued", never "verified" or "authentic". The caveat is a field of the
artifact itself (`receipt.caveat`) so it travels with the file. Anything stronger would require
measuring somewhere the reader does not control, which is the covert monitoring this product
refuses to build.

**The recall block is the substance.** `receiptIsSubstantive()` is false when nothing was
answered, and the panel says so in those words: coverage alone records that pages were scrolled
through, which is not evidence of reading and is trivial to fake.

**`auditReceipt()`** is a backstop for invariant 6. It walks the artifact and rejects anything
that looks like raw sensor data or a full URL, and the panel refuses to present a receipt that
fails. Covered by `tests/receipt.test.js`; the browser check asserts no URL and no gaze keys
reach the panel.

Signing is HMAC-SHA256 with `RECEIPT_SECRET`, so verification goes back through the server and
the key never leaves it. Asymmetric signing would let third parties verify offline; that is a
human decision, not a default to slip in. `POST /api/receipt/sign` and `/api/receipt/verify`
store nothing.

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

## Platforms — desktop Chrome today, Safari on iOS/iPadOS intended

The extension is built to run on a phone, and the architecture already suits it: telemetry is
the primary path, it needs no permission, and scroll and pace signals work exactly the same on
a touch screen. What has been done, and what has not:

**Done — the UI survives a phone.**

- The popup is `width: min(400px, 100vw)`, and below 400px the two-column grids collapse to one.
- `@media (pointer: coarse)` raises every tappable target to 44px, the smallest Apple's guidance
  treats as reliable. The desktop sizes are 20–34px, which is a mis-tap every time.
- Hover styling is neutralised on touch, where it latches after a tap and reads as a selection
  the reader never made.
- In-page cards become a bottom sheet under 560px; the word bubble, which needs hover, is hidden.
- Camera controls are removed entirely when the platform cannot deliver a camera. `popup.js`
  `cameraIsAvailable()` treats iOS and iPadOS as false — `navigator.mediaDevices` exists there
  and does not work, so feature detection alone reports a capability that is not real.

**Not done, and not doable from this repository.**

- **A Safari build needs an Xcode wrapper.** `xcrun safari-web-extension-converter alcoia/`
  generates an Xcode project; shipping it needs a paid Apple Developer account, an App Store
  submission, and a native container app. There is no build step in this repo and adding one is
  a separate decision.
- **Keyboard shortcuts do not exist on iPhone.** Alt+S/T/F/M/R/I are reachable on an iPad with a
  keyboard and nowhere else. Every one of them needs a touch equivalent before the mobile
  experience is complete — most already have a button in the popup; the focus ruler, reading map
  and word lookup do not.
- **`Ctrl+hover` word lookup has no touch path at all.** Long-press is the obvious candidate and
  collides with the system selection menu; this needs design, not just an event listener.
- **WebGazer is dead weight on mobile** — no usable camera, and it is the GPLv3 dependency. If
  Safari becomes a real target, dropping it is worth more there than anywhere else.

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

**Fonts.** ~~No font is bundled.~~ **Two are, as of the UI pass:** Source Serif 4 and Plus
Jakarta Sans, latin + latin-ext, roman + italic, variable weight, ~436 KB total in
`src/libs/fonts/`. Both are SIL OFL 1.1 and both licence files ship beside the binaries
(`OFL-SourceSerif4.txt`, `OFL-PlusJakartaSans.txt`). **Any future font binary must arrive with
its licence in the same directory** — that is the whole obligation, and it is easy to forget.
~~Fraunces is fetched from `fonts.googleapis.com` on every page the reader visits.~~ **Removed.**
No request to `fonts.googleapis.com` is made anywhere any more. See `NOTICE.md`.

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

## Known gaps in test coverage — read before trusting a green run

The P6 refactor silently deleted the entire keyboard-shortcut handler and **every test passed**.
ESLint caught it, not the suite. The lesson is not only "lint before refactor" — it is that the
suite's *shape* has holes, and the keyboard handler was simply the first one to become visible.

Closed since:

- **Keyboard shortcuts** are now pressed in `npm run test:browser` — Alt+1/T/F/M/S/I/R and Esc.
- **The classifier feature contract** (`tests/classifier-feature-contract.test.js`) compares the
  keys the tree branches on against the keys the extractor emits, by reading both files. This is
  the guard CLAUDE.md asked for. It cannot be caught by running the classifier, because the broken
  case returns confident labels — that is the failure mode.

A third hole became visible in the UI pass, and it is the same shape as the first two:

- **Nothing asserts that a stylesheet rule reaches an element.** The question card and the
  receipt shipped their CSS in a file no code loaded, and 216 unit tests plus a browser smoke
  check all passed while the primary intervention rendered as unstyled default HTML. The browser
  check clicks `.sra-q-option` and reads `.sra-q-text`; both work perfectly on an unstyled
  button. **Closed:** `npm run test:browser` now reads computed styles — that `.sra-popup`
  resolves `position: fixed` with a non-zero radius, that `.sra-q-option` is styled, and that
  `Source Serif 4` is the family actually resolving — and it counts third-party requests, which
  must stay at zero. Presence of an element is not evidence that its rules loaded.

Still open, and worth knowing when a green run tempts you:

- No test drives a **full reading session** end to end with clock advancement — scroll, dwell,
  regress, answer, across minutes. The budget is unit-tested against a fake clock and the browser
  check runs ~20 seconds.
- The **camera-on path is not a distinct case** in any test. `tfhub.dev` is blocked here, so
  `webgazer.begin()` has never reached a camera in any run.
- **Question quality is untested.** See below.

---

## Question quality — the untested thing that matters most

The question pipeline is verified mechanically: `/api/questions` is hit, `/api/summarize` is not,
an explanation follows only a wrong answer. All of that was against a canned endpoint.

**Whether the questions are any good has never been checked.** A well-plumbed pipeline serving
mediocre questions fails in a way that looks like "readers don't want interruptions" rather than
"the questions were bad", and those are indistinguishable in retention metrics while having
opposite fixes.

`npm run questions:review -- --server http://localhost:3000 --file <path> --url <url>` runs real
passages through the real endpoint and lays the output out for reading. It applies the checks that
can be mechanised — span really in the passage, question/span lexical overlap (the word-matching
tell), giveaway distractors, conspicuous option lengths — and flags rather than scores. Judging
whether a question tests understanding is a human job. Do twenty varied pages before shipping.

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
