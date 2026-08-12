# CLAUDE.md

Repository context for agents. **Read fully before any task.** This file describes the codebase
**as it actually is today**, not as it should be. Where it is wrong, fix it as part of your change —
a stale context document is worse than none, because it is trusted.

**This repository is the extension only.** It contains no roadmap. Build instructions arrive per
task; this file tells you what exists and what you must not break.

---

## Product intent

alcoia is a browser extension (Chrome MV3, Firefox via build target) that notices when a reader is
struggling with a page and intervenes with a **retrieval question** about what they just read.

### The optimisation target is inverted, and it binds code

alcoia's purpose is **to make the reader need it less.**

| Normal product | alcoia |
|---|---|
| More engagement is success | Declining intervention need is success |
| Retention = daily active use | Retention = the reader still reads well, with fewer prompts |
| Help more when the user struggles | Help less on material they have demonstrated competence in |
| Explain thoroughly | Explain **only** on failure, then stop |

- **A correct answer ends the interaction.** Confirmation only — no explanation, no elaboration, no
  praise. Explaining a correct answer adds load at the moment of consolidation and trains the
  reader to expect the system to do the closing work.
- **The explanation path is the failure path.** Reached after a wrong answer, or when no question
  could be generated. Never the default.
- **The cheapest path through the system must be the successful one.**

**The measurement trap.** "Fewer interruptions because the reader improved" and "fewer
interruptions because the reader trained the system into silence" produce **identical telemetry**.
Any adaptation that reduces interruption frequency must be gated on demonstrated competence —
answer accuracy at escalating difficulty — never on interruption count or dismissal rate. A system
that optimises for its own silence will achieve it and report success.

### Signal hierarchy, in order of authority

1. **Reader responses** (`telemetry/response-signals.js`) — the only ground truth. The engine gives
   them confidence above anything telemetry can produce, so an answer decides the state. A correct
   answer resolves to `on_pace`. **A dismissal asserts nothing at all** — declining to be tested
   says nothing about comprehension, and must never be read as success or as struggle.
2. **Browser telemetry** — reading rate vs. text difficulty and personal baseline, scroll
   regressions, selection, copy, blur, idle. Precise, always available, no permission needed.
   **The primary path and the default.**
3. **Webcam gaze** — scheduled for removal. See *Known defects*.

If a task appears to invert this hierarchy, stop and ask.

---

## Scope

**In this repository:** content scripts, telemetry detectors, state engine, interruption policy,
question card, quiz, receipt, popup and extension pages, styles, build.

**Not in this repository:**

| Thing | Where | Note |
|---|---|---|
| API server | Separate private repo | See migration status below |
| Accounts, entitlements, install tokens, assist counter | Server | Client displays; server decides |
| Question generation and validation | Server | The client consumes questions; it never authors them |
| Aggregate class analytics | Server | Cohorts cannot be aggregated on one client |
| Educator / team portals | Separate web app | The extension never sees the admin console |
| Pricing, tiers, legal text | Human decision | Scaffold only, escalate |

⚠️ **Migration in progress.** `alcoia/server/` still exists in this tree and is **legacy**. It is
excluded from the shipped package by `build.mjs`, and `tests/questions.test.js` (31 tests) and
`tests/receipt.test.js` (24 tests) import from it. Removing it is a sequenced task; until it lands,
treat `alcoia/server/` as **read-only reference**. Do not extend it. If a task requires a server
change, say so and stop.

---

## Statefulness — decided: level B

The server holds **accounts, entitlements and a counter. Nothing more.**

| Level | Server holds | Status |
|---|---|---|
| A — stateless | Nothing | Superseded |
| **B — account + counter** | **Email, plan, entitlements, install tokens, assist counts** | **Current** |
| C — account + reading history | Above, plus concepts, answers, calibration state | **Not approved** |

**Consequence for this repository:** nothing in the extension may transmit reading *content* —
paragraph text, question history, answers, concepts — to the server for storage. Settings, the WPM
baseline and UI preferences are not reading content and already persist locally.

**Local persistence is permitted and is not level C.** Level B constrains what *the company* holds.
Data that never leaves the device creates no legal surface and no sync obligation. The quiz record
lives here.

### The C ledger — deferred, blocked on one decision

Every row is blocked on the same question: **does the server hold a record of what individuals
read?** Recorded here so these stop being reintroduced inside feature lists.

| Item | Why it is C |
|---|---|
| Cross-document concept linking / concept map | Requires history across documents |
| Per-concept epistemic state (knowledge / recall / transfer / confidence / calibration) | Persistent per-reader model |
| **Connect** rung of the difficulty ladder | "How does this relate to yesterday?" needs yesterday |
| Longitudinal interruption quieting | Competence per reader per domain over months |
| Spaced retrieval / next-day recall | Yesterday's paragraphs must survive today |
| Personal knowledge graph | Explicitly rejected once already as "a different company" |
| Making forgetting visible | Requires a months-long baseline |
| Surprise retrieval across documents | Same |
| Cross-device quiz sync | Quizzes are local-only by decision |
| Independence trajectory metric | Unaided-vs-explained ratio over time |

**One approved feature straddles the line.** Aggregate class analytics — anonymous, cohort-level —
is approved but has two builds and they are not equivalent:

- **Client-side aggregation, unlinkable submission.** The extension computes per-document outcomes
  and submits them detached from the account. Genuinely anonymous. **Stays within B.**
- **Server-side aggregation from identified submissions, anonymised on read.** Easier, standard, and
  the server now holds per-reader reading outcomes. **That is C**, regardless of what the dashboard
  displays.

The claim that the aggregate view is "structurally robust to individual fraud" holds only for the
first build. Open architecture decision.

---

## Access control — decided

**No account is required to use the extension.** Free install, no signup, no email.

**AI calls are gated by an install token.**

- On first run the extension requests an opaque token from the server and stores it locally.
- Every AI call carries it. **No token, no response.**
- The server counts against the token and enforces the ceiling.

**The token is issued, not derived.** It is not computed from device or network characteristics.
The reader holds it, can delete it, and deleting it works — they get a fresh one. It identifies an
install, not a person or a machine. It is never attached to passage content in logs.

**Explicitly refused: any device or network fingerprinting.** No IP-derived identifier, no browser
fingerprint, no canvas/font/audio entropy, no attempt to re-identify a reader who cleared their
data. This is invariant 1 in a different costume: a mechanism whose only purpose is to defeat an
action the reader deliberately took is covert monitoring. (MAC address is not obtainable from a
browser at all.)

**The reinstall leak is accepted.** A reader who reinstalls to reset their count is one person,
manually, monthly. That is not the threat the token exists to stop. The threat is an
unauthenticated endpoint being scripted as a free LLM proxy by someone who never installs the
extension.

**No visible quota on the free tier.** No counter in the reading UI, no "N remaining", no upgrade
nag. The ceiling bounds automated abuse; it does not meter readers. The **diagnostics page** is the
one place the number is visible, for support.

**Metering:** a generated quiz costs **one assist** and must therefore be **one server call** — one
passage batch in, N questions out. Not one call per question.

**The client's count is a display convenience. The server is the gate.**

---

## Hard invariants — never violate, never make configurable

1. **No covert monitoring.** No hidden mode, no silent-collection flag, no admin override observing
   a reader without their action, no fingerprinting. Not behind a flag, not for a paying
   institution, not for testing.
2. **Video never leaves the device.** No frames, no raw camera data, no base64 images in any request.
3. **No accuracy claims** in code, comments, UI or docs.
4. **Receipts are reader-generated only.** No background submission. The reader sees the contents
   before sharing.
5. **`unknown` is a valid, correct, common state.** Never interrupt on it. Never substitute
   plausible defaults for missing data.
6. **No raw gaze coordinates** in any persisted or transmitted artifact.
7. **Never accuse.** A receipt reports *unaltered since issued* or nothing. Never *verified*, never
   *authentic*, never *suspected*. Atypical behaviour has innocent causes: motor impairment,
   assistive technology, trackpad, phone.
8. **Never test a reader who did not read.** A drifting reader, or one below the coverage
   threshold, is guaranteed to fail — which is punishment. Re-read prompt first; question after.
   **This applies to quizzes at page scale exactly as it applies to single interventions.**
9. **Every failure degrades to silence.** Server unreachable, 422, extraction failure, hostile page,
   model timeout, malformed model output — every one resolves to `unknown`, and `unknown` never
   interrupts. A wrong intervention is worse than a missed one.
10. **No reading content is transmitted for storage.**

### Interruption budget

Reader attention is the scarcest resource in this product.

- **One interruption per 3 minutes.**
- **Cap scales with content read, not with "session."** A textbook chapter earns more than a news
  article because it is more. Derived from tracked paragraphs and measured reading time
  (`paragraph-tracker`, `progression-entropy`).
- **Absolute per-session ceiling ~20–25** so pathological loops terminate.
- **Dismissal-aware backoff.** Consecutive dismissals raise the bar. Three in a row is the reader
  telling you to stop, and it is more reliable than any inference.
- **Never twice on the same paragraph.** Never on `unknown`. Minimum confidence 0.5.
- **Skimming interrupts only on `difficult` / `very_difficult` text.** Skimming easy prose is a
  choice. This is load-bearing for extraction readers (a lawyer scanning for a clause, an engineer
  looking up a signature) and must not be "fixed."
- **Every interruption carries user-visible evidence** — "You slowed down a lot here" — converting
  an inference into an observation.
- **Reader-initiated actions spend no budget.** A quiz the reader asked for is not an interruption.

> **Superseded:** the flat five-per-session cap. It used the same number for a two-minute article
> and a three-hour chapter, and failed readers of long or difficult material. The denominator was
> the defect, not the existence of a cap.

Enforced in `intervention-policy.js`. Two notes:

- `evaluate()` decides, `record()` spends. Call `record()` only once the interruption is on screen,
  or a decision dropped downstream silently burns budget.
- The orchestrator calls `host.onIntervention()` and takes a **boolean** back saying whether
  anything reached the screen. **Preserve that seam** — it makes the invariant structural rather
  than remembered.

---

## The quiz — decided

- **Offered at end of reading, and triggerable from the popup.** Both read the **same coverage
  threshold from one function**. If they diverge, the overlay says ready while the button says no.
- **Gated on accumulated coverage for the document, not for the visit.** Session continuity already
  survives leaving and returning — the same store drives this. Read half today, half tomorrow, and
  it unlocks tomorrow. Coverage keys must be robust to query strings and SPA navigation; a
  `?utm_source=` must not reset progress.
- **Below threshold the popup button is disabled with a stated reason** — "not enough reading
  tracked on this page yet" — never a silent no-op.
- **Reader-initiated, so it spends no interruption budget.** The end-of-page *offer* is shown at
  most once per document and is dismissible.
- **Selection is weighted, not ranked** — reuse `session-recall.js`. Struggle raises weight
  substantially without guaranteeing a slot; every genuinely-read paragraph keeps a real chance.
  Asking only about failures turns the quiz into a list of the reader's mistakes.
- **5–8 questions.** Never one per paragraph.
- **One server call per quiz.** See metering above.
- **Local-only, persisted, IndexedDB.** Never transmitted.
- **No retake.** This is why passage text is **not** stored — only questions, the reader's answers
  and confidence ratings, and verdicts. Review works; regeneration does not.
- **Deletable**, per document and all at once, and deletion must actually delete.
- **This is the first feature that writes reading content to disk.** Privacy copy must be updated in
  the same change, not later.

---

## Confidence calibration — decided shape

**Confidence is captured at commit time**, not as a post-answer probe: one card, answer and
confidence submitted together, graded afterward.

This is structural rather than remembered. A post-hoc probe leaks the result — if "are you sure?"
appears more often after wrong answers, readers learn to read the interface instead of the passage
and the mechanism inverts into a hint system. Commit-time capture cannot leak, needs no
randomisation, no one-in-three cap, no "never twice consecutively", and measures calibration rather
than hindsight.

| Answer | Confidence | Response |
|---|---|---|
| Correct | High | Correct, and appropriately confident |
| Correct | Low | Correct — you knew more than you thought |
| Wrong | High | The important case. Confidently wrong is where learning is most needed |
| Wrong | Low | Wrong, and you knew you were unsure. That is good calibration |

- **Skippable.** A reader who answers without rating gets the bare-✓ correct path.
- **Wrong + high confidence gets no harsher tone than wrong + low.** Never write copy implying the
  reader should feel worse for having been confident.
- **Do not ship "correct answer, but your reasoning doesn't support it."** It requires grading
  reasoning quality — a much harder model task with a high false-positive rate — and telling a
  reader who reasoned correctly that they did not is exactly the manufactured-doubt failure the UX
  constraint exists to prevent.

**Elaboration prompts are a separate mechanic** ("why do you think that?", "what would change your
mind?", "give me one reason you could be wrong"). Rare, and **keyed on self-reported confidence,
never on correctness** — the reader already knows their own confidence, so that leaks nothing.
Keying on correctness is what leaks.

---

## Claims discipline

At least eight accuracy figures exist across this project: `0.851` (`src/content/classifier.js`),
`0.909` (`tldr classifier/classifier.js`), 88%, 91.5%, ~65–70%, 0.835 / 0.884 / 0.897 (notebook
outputs), 92% (report), "75–82% real-world" (marketing drafts).

**All derive from synthetic data. No real-participant evaluation has ever been performed.**

The shipped figure is inflated **by construction**: the training notebook duplicates every row with
Gaussian noise and *then* does a random 80/20 split, so a row's noisy twin sits in test while the
original sits in train. That is train/test leakage. `0.851` is not valid even as a measure of how
well the tree recovers its own generator's rules.

- Never introduce, restore or repeat an accuracy percentage.
- Where one exists in a file header, it must carry the full qualifier.
- If asked for copy implying detection reliability, write a mechanism description and flag it.

---

## THE TRAP — read before touching the feature set

`classifier.js` branches on `f.saccade_length`, `f.saccade_std`, `f.velocity_mean`.

Remove those keys from the extractor without retraining and every comparison becomes
`undefined <= X`, which evaluates to `false` **without throwing.** The classifier does not crash. It
routes down one branch forever and keeps emitting confident labels.

```
classifyGazeState({})                   → { label: 'skimming', confidence: 0.722 }
focused sample, 3 saccade keys removed  → focused (0.993) becomes skimming (1.000)
```

Pinned by `tests/classifier-missing-keys.test.js` and `tests/classifier-feature-contract.test.js`.
**Do not disable either.**

**This does not block removing the gaze path wholesale.** `orchestrator.js` gates the classify loop
on `!cfg.eyeTrackingEnabled || !host.getLastGazePoint()`. With no sensor there is no gaze point, the
loop returns early, and `classifyGazeState` is never reached. The trap applies to deleting *feature
keys while the classifier still runs* — a different and more dangerous operation.

---

## Actual repository state

Verified by reading the tree. Line counts current as of this writing.

```
.
├── LICENSE                    AGPL-3.0 — repo root only, see DEFECT
├── build.mjs                  per-target package build; excludes server/
├── eslint.config.js           flat config, defect linter
├── manifests/
│   ├── base.json              shared manifest — THE SOURCE OF TRUTH
│   ├── chrome.json            service_worker
│   └── firefox.json           background.scripts + gecko.id
├── tests/                     16 files, 250 tests (Vitest) + browser smoke
├── tools/question-quality.mjs question review harness
├── tldr classifier/           notebooks + synthetic CSVs — historical
└── alcoia/
    ├── manifest.json          GENERATED from manifests/. Do not hand-edit
    ├── background.js          service worker
    ├── server/                LEGACY — excluded from build, migration pending
    ├── src/content/
    │   ├── content.js           1754 — host: modules, settings, fetch, highlight, word lookup,
    │   │                               selection, keyboard, SPA nav, render callbacks
    │   ├── ui-controller.js      466 — popups, highlight, toasts, dark mode. Owns openPopups
    │   ├── reading-calibration.js 443 — WPM calibration (self-paced path is the live one)
    │   ├── gaze-utils.js         414 — WebGazer smoothing/EMA        [removal candidate]
    │   ├── state-engine.js       402 — signal fusion, one state estimate
    │   ├── orchestrator.js       389 — detectors, engine, budget, the one subscriber
    │   ├── comprehension-monitor.js 300 — pace vs. difficulty vs. personal baseline
    │   ├── receipt.js            282 — reader-owned session record + preview
    │   ├── classifier.js         274 — GENERATED decision tree       [removal candidate]
    │   ├── reading-map.js        255 — sidebar minimap
    │   ├── focus-ruler.js        239 — reading band
    │   ├── gaze-features.js      232 — feature extractor             [removal candidate]
    │   ├── webgazer-bootstrap.js 202 — main-world injection          [removal candidate]
    │   ├── question-card.js      150 — the retrieval question card
    │   ├── dyslexia-utils.js     136
    │   ├── idle-overlay.js       133
    │   ├── overlay-utils.js      132
    │   ├── pdf-handler.js        130 — partially wired, see defects
    │   ├── tts-handler.js        126
    │   ├── lang-detect.js        125
    │   ├── intervention-policy.js 125 — the interruption budget, one place
    │   ├── session-tracker.js     91
    │   ├── pptx-handler.js        78 — partially wired, see defects
    │   ├── sra-page-bridge.js     41 — postMessage bridge            [removal candidate]
    │   └── telemetry/
    │       ├── segmentation.js       225 — NOT a detector. Word/sentence counting, Intl.Segmenter
    │       ├── text-difficulty.js    185 — NOT a detector. FK + syntactic load
    │       ├── response-signals.js   118 — TIER 1, not telemetry. Placement is historical
    │       ├── session-recall.js     117 — recall pool (in-memory)
    │       ├── paragraph-tracker.js  158 — which paragraph is being read; figures/tables/pre are
    │       │                             tracked landmarks with media: true, not measured as prose
    │       ├── interaction-signals.js 100 — selection, copy, blur/return
    │       ├── cursor-tracking.js     93 — pointer-Y. Its signal() output is DEAD, see defects
    │       ├── scroll-regression.js   76 — paragraph-index returns
    │       ├── progression-entropy.js 76 — session shape
    │       ├── scroll-dynamics.js     59 — scroll jerk
    │       └── residual-distribution.js 53 — NOT a detector. Per-reader pace thresholds
    ├── src/popup/             popup.js, notes.js, 6 HTML pages
    ├── src/styles/            fonts.css, overlay.css, panel.css, popup.css (legacy)
    └── src/libs/              webgazer.min.js (GPLv3, 1.7 MB), pdfjs, jszip, fonts/ (OFL 1.1)
```

**Detector count, precisely.** Eleven files in `telemetry/`, but only **eight** export a `create*`
factory; of those, `response-signals.js` is tier 1 rather than telemetry and `cursor-tracking.js`'s
signal is not consumed. So **six live telemetry detectors** feed the engine, plus
`comprehension-monitor.js`, the primary sensor, which lives outside `telemetry/`. Do not describe
this as "eleven detectors."

**A build step exists.** `build.mjs` copies the source tree verbatim per target — not a bundler,
nothing transpiled — and writes a per-target `manifest.json` from `manifests/base.json` +
`manifests/<target>.json` (shallow merge, deliberately: deep-merging `background` would give Firefox
both a service worker and a scripts array).

⚠️ **`alcoia/manifest.json` is generated and committed.** `tests/manifest.test.js` runs the build as
a side effect, so editing the generated file and running `npm test` **silently reverts your
change.** Edit `manifests/base.json`.

**`sra_` prefixes are deliberate.** `sra_` storage keys and `sra-` CSS/element-id prefixes (~300
occurrences) survive the rebrand. Renaming storage keys discards every existing reader's settings
without a migration. `mode=tldr` in the summarise request is an API contract, not a brand string.

---

## Known defects — verified by reading the code

### Dead code that reads as wired

**`cursor_reading` is dead twice over.** `telemetry/cursor-tracking.js` emits it. `state-engine.js`
lists it in `CORROBORATING_TYPES` — excluding it from asserting — but there is **no entry in
`CORROBORATION`**, so the loop skips it unconditionally. And `orchestrator.js` never calls
`cursorTracker.signal()`; it uses only `getPointerY()`. `tests/cursor-and-progression.test.js`
passes against the detector in isolation and asserts nothing about the engine.

### Configuration

- **`localhost:3000` is hardcoded in four code sites**, not two: `content.js` (`BACKEND_DEFAULT`),
  `background.js` (fallback URL), `popup.js` (`sra_backend_url` default — the one that reaches
  storage), `popup.html` (placeholder). Plus `http://localhost/*` and `http://127.0.0.1/*` in
  `host_permissions` in **`manifests/base.json`**.
- **All four icon sizes point at one PNG.** `assets/alcoia-mark-lilac.png` serves 16/32/48/128 in
  both `action.default_icon` and `icons`.
- **The shipped package contains no LICENSE.** `build.mjs` copies from `alcoia/`; `LICENSE` sits at
  the repo root. `dist/*/` ships 1.7 MB of GPLv3 WebGazer with no licence text. Resolves itself when
  the gaze path goes; until then it is a defect. Font licences are fine.

### `gaze-features.js` — unfixable by tuning

- **`dist > 30`** as the fixation/saccade boundary against ~180 px of tracker error — entirely
  inside the noise floor. `saccade_length`, `saccade_std`, `velocity_mean` and `regression_rate`
  measure tracker jitter, not eye movement.
- **`lineBand = Math.round(pt.y / 20)`** uses raw viewport `y` with no scroll offset. Scrolling
  changes the band with zero eye movement.
- **Fallback constants are all focused-looking.** When data is sparse the extractor fabricates
  plausible focused-reading features instead of abstaining. Violates invariant 5.
- **DBSCAN `eps: 80`** is below tracker error, so it clusters noise.

`gaze_drift_px`, `on_page_fraction` and `face_present` are the only honest outputs.

### `pdf-handler.js` / `pptx-handler.js` — partially wired

Both carry `no-unused-vars` warnings for `backendUrl`, `fetchSummary` and `renderPopup`, suggesting
they accept dependencies they never use. Extraction, paragraph matching and the question path for
PDF and PPTX have not been verified end to end. **Verify before building file import on top.**

### Convention drift

`content.js` is **1754 lines** and growing (was ~1435 at the last refactor). Six files exceed the
~300-line convention. Settings live in `content.js` as loose `let`s read through accessors
(`settings()` / `getSettings()`) because the storage listener reassigns them and a captured copy
goes stale silently. **Add new logic to the new modules, never to `content.js`.**

---

## Known gaps in test coverage — read before trusting a green run

250 tests pass. `npm run lint` exits 0 with 15 warnings (all `no-unused-vars` in untouched files).

**The suite's failure mode is absence, not error.** Four instances so far:

1. A refactor silently deleted the entire keyboard-shortcut handler. **133 tests passed.** ESLint
   caught it, because no test pressed a key.
2. The question card and receipt shipped CSS in a file no code loaded. 216 tests plus a browser
   smoke check passed while the primary intervention rendered as unstyled HTML — the check clicked
   `.sra-q-option` and read `.sra-q-text`, both of which work perfectly on an unstyled button.
3. Every word count was `text.split(/\s+/)`, so a 600-character Chinese paragraph counted as **one
   word** and fell under every threshold. The extension loaded, ran and did nothing on a large
   fraction of the web. Nothing threw. Every test passed.
4. `cursor_reading` — currently live in the tree.

**Still open:**

- **`comprehension-monitor.js` has no unit test file.** The primary sensor is untested — the
  running-median WPM baseline, its `chrome.storage` persistence, the speed-mismatch thresholds, and
  **the reader-to-self comparison that stops the system quizzing a slow reader for reading slowly.**
  That is the fairness safeguard, covered only incidentally.
- **No end-to-end reading session** with clock advancement across minutes.
- **The camera-on path is not a distinct case** in any test.
- **Question quality is untested.** `npm run questions:review` mechanises what can be mechanised —
  span really in the passage, question/span lexical overlap, giveaway distractors, conspicuous
  option lengths — and **flags rather than scores.** Judging whether a question tests understanding
  is a human job. Do twenty varied pages before shipping.
- **No test asserts a failure path reaches `unknown`.** Invariant 9 is stated but unguarded.

---

## Decisions already made — do not re-litigate

- **Camera off by default.** `sra_eye` defaults to `false` in `content.js`, `popup.js` and the
  service worker guard. Reading modes must never set it.
- **Telemetry is the primary path.**
- **Questions, not summaries, are the primary intervention.** Summarising performs the operation
  that produces the learning. Follows D'Mello et al. (2016), d = 0.47.
- **State names describe observations:** `on_pace`, `skimming`, `struggling`, `drifting`, `absent`,
  `unknown`. Do not reintroduce `confused` or `overloaded`. (`classifier.js` still emits them
  because it is generated; `content.js` translates in one place.)
- **Statefulness is level B.** Access is via issued install token. Fingerprinting refused.
- **Aggregate class analytics are anonymous and cohort-level only.** Not per student. Individual
  verification is the weaker and riskier pitch, and universities are moving away from
  proctoring-adjacent tooling. The individual receipt stays student-facing and formative.
- **File import is a Reader-plan entitlement.** Gated server-side. The extension asks; it does not
  decide.
- **AGPL-3.0 for the client.** The server is a separate program over a network API. Do not cite
  AGPL's network clause as a deterrent — it does almost nothing for an extension.
- **No XP, levels, streaks, leaderboards, reader types, shareable cards, or confetti.** Three
  reasons: (a) the stated diagnosis is that attention is damaged by dopamine loops, so answering
  with a dopamine loop is incoherent; (b) tangible rewards reliably undermine intrinsic motivation
  for tasks people already find interesting; (c) **data corruption** — if reward attaches to
  answering questions, and questions fire on detected struggle, a reader who wants reward reads
  slowly and scrolls back to *appear* to struggle. That is genuine human scrolling, merely
  performed, and it poisons the corpus the detector depends on. **Confetti on reaching the end of a
  page is specifically refused**: it rewards scroll depth, which is the completion metric this
  product exists to replace.
- **No personality instruction to the LLM.** Tone is fixed copy templates. A personality instruction
  hands an education product used by minors, sold to institutions, the ability to improvise. **This
  is a safety boundary, not a style preference.** Consequently *epistemic adversary mode* — "I'll
  take the opposing position, defend yours" — is **not approved** in free-form. Any future version
  must draw challenges from the passage's own stated counterarguments, template-shaped, never
  generative on contested topics.

### Type and colour

- **Fonts: Literata** (reading) + **Plus Jakarta Sans** (UI), bundled under SIL OFL 1.1. Nothing is
  fetched from Google. Reached **only** through `--alc-serif` / `--alc-ui` in
  `src/styles/fonts.css`. **Any future font binary must arrive with its licence in the same
  directory.**
- **Palette: warm paper, matte.** `--alc-paper` `#F9F7F2`, `--alc-text` `#333333`. Never pure white,
  never pure black.
- **`--alc-accent` `#7E60AE` is the brand colour**; `--alc-accent-2` `#5F4589` for type.
- **`--alc-sage` `#A3B1A3` is reserved for moments the system itself produced**: the evidence badge,
  the quoted span, the paragraph highlight, a correct answer. **Not a second accent.** Its meaning
  is a function of how rarely it appears.
- **No gradients, glows, or shadows on controls.** Elevation is a hairline. `--shadow-sm` is `none`
  on purpose. The floating card is the one exception.
- **State hues avoid the brand colour by design** — on pace → sage, skimming → `#5B7A99`,
  struggling → `#9A6B2F`, drifting → `#7E6E5A`.

---

## Requires human approval — scaffold only, do not author

- Privacy policy, terms, DPA, refund policy. Structure and TODO markers only. Generated legal text
  will be plausible and wrong about actual data flows.
- Any pricing figure or quota number.
- Any change to the invariants above.
- Any move from level B toward C.
- Deleting or replacing the trained classifier.
- Anything touching payments or user PII.

Stop and ask rather than proceeding.

---

## Open questions — do not resolve implicitly

1. **The difficulty ladder vs. the span requirement.** The intended progression is
   Recall → Explain → Connect → Apply → Challenge. The server rejects any question whose `span` does
   not appear verbatim in the passage — that rule is what stops the model inventing a question and
   asking it of a struggling reader. But **Connect, Apply and Challenge have answers not in the
   passage by construction.** Either the span rule becomes level-dependent with a *different*
   validator for ungrounded levels, or the ladder stops at Explain. **Do not weaken the span check
   to accommodate the ladder.**
2. **Free-text answers, model-scored.** Would make tier-1 ground truth a model output, granting an
   LLM higher authority than telemetry, and sends reader-authored prose to a third party. If built:
   passage and reader answer must occupy **separate delimited data fields**, never concatenated into
   instruction position; the grader must return a **constrained shape** (verdict enum + span) and
   anything else resolves to `unknown`; hard length cap; own rate-limit bucket; never render model
   output as HTML. The disclosure question is separate from the code question.
3. **Aggregate analytics architecture** — client-side unlinkable vs. server-side identified.
4. **Images in intervention cards.** Fetching third-party images breaks the zero-third-party-request
   assertion and is a rights problem. Referencing a diagram **already on the page** is cheap and is
   retrieval rather than illustration. Only the latter is under consideration.
5. **File handles for re-import.** Paths cannot be stored; `FileSystemFileHandle` objects can be
   persisted to IndexedDB and re-permissioned on a user gesture. Requires an extension page
   (`showOpenFilePicker` is unavailable in content scripts), a `NotFoundError` state for moved files,
   and a re-upload fallback for Firefox, which has no File System Access API.

---

## Conventions

- ES modules; no bundler-specific syntax in content scripts. Modules load dynamically via
  `loadModule()` in `content.js`, which is why they must appear in `web_accessible_resources`.
- **New telemetry detectors** go in `src/content/telemetry/`, each exporting `{ update(), signal() }`.
  Register the type in `state-engine.js`: assertable types get a branch in `fromTelemetry()` with a
  confidence and an evidence sentence; corroboration-only types go in `CORROBORATING_TYPES` **and**
  `CORROBORATION`, plus a `CORROBORATION_GUARD` if the signal is only meaningful for one subtype.
  **An unregistered type is silently ignored** — `fromTelemetry()` returns null. That is how
  `cursor_reading` died.
- `orchestrator.js` decides; `ui-controller.js` renders. `ui-controller.js` owns `openPopups` and
  nothing else may mutate it — dedup and `MAX_POPUPS` eviction both depend on it being the only
  record of what is on screen. Create cards through `reservePopup()` / `showPopup()`.
- `handleComprehensionSignal()` is a **renderer only.** Never call it from a detector. Detectors
  call `stateEngine.update({ telemetry: signal })` and nothing else.
- Keep modules under ~300 lines. Six currently are not; do not add a seventh.
- Tests in Vitest. Priority: extractors against fixtures, state engine against synthetic sequences,
  interruption budget, failure-to-`unknown` paths, and the missing-key guard.
- No new runtime dependencies without asking.
- Two stylesheets with the same basename in different directories is how defect (2) hid. Do not
  recreate that arrangement.

---

## Verification

```bash
npm run lint && npm test
npm run test:browser            # English article
PAGE=zh npm run test:browser    # Chinese article — guards the segmentation path
node build.mjs                  # both targets
```

Then load unpacked and confirm manually:

- Loads with no console errors on a plain article page
- Reading detection runs with the camera **off**
- No `getUserMedia` call unless the reader explicitly enabled the camera
- No network request contains image or video data
- Third-party requests remain at zero

**Report what you verified, not what you believe should work.** Where you assumed rather than
checked, say so. Where this file conflicts with the code, the code wins — and update this file in
the same PR.

---

## Working agreement

- **One item per session.** If a diff cannot be read in one sitting, the item was scoped too large.
  Split it and say so.
- **Open a PR. Never push to main.**
- **Flag conflicts between the brief and the code, every time.** That behaviour is correct and
  should not be moderated.
- **Standing objections to raise unprompted:** any accuracy figure in any framing; any drift toward
  covert observation, fingerprinting, or per-student institutional reporting; any addition that
  transmits reading content; any "small change" that turns out to introduce a storage surface.