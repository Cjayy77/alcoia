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
   regressions, selection, copy, blur. Precise, always available, no permission needed. **The only
   detection path.**

✅ **Webcam gaze has been removed**, not merely demoted. It used to sit as tier 3 here, corroborating
telemetry and — in one narrow case — disambiguating an idle-with-focus reader. Both roles are gone:
there is no sensor, no classifier, no calibration flow that trains one, and no code path that reads
a gaze coordinate. See "Actual repository state" below for what was deleted and why, including two
modules (`idle-overlay.js`, `lang-detect.js`) that were not on the removal item's original file list
but turned out to exist solely to serve the gaze pipeline.

If a task appears to reintroduce gaze, a camera permission, or a webcam-derived signal, stop and ask.

---

## Scope

**In this repository:** content scripts, telemetry detectors, state engine, interruption policy,
question card, quiz, receipt, popup and extension pages, styles, build.

**Not in this repository:**

| Thing | Where | Note |
|---|---|---|
| API server | Separate private repo | Fully moved as of this migration item — see status below |
| Accounts, entitlements, install tokens, assist counter | Server | Client displays; server decides |
| Question generation and validation | Server | The client consumes questions; it never authors them |
| Aggregate class analytics | Server | Cohorts cannot be aggregated on one client |
| Educator / team portals | Separate web app | The extension never sees the admin console |
| Pricing, tiers, legal text | Human decision | Scaffold only, escalate |

✅ **Migration complete.** `alcoia/server/` has been deleted from this repository (owner confirmed
it is already preserved in the separate private server repo before deletion). The API server is no
longer read-only reference here — it is simply not here. `tests/questions.test.js` (31 tests) and
`tests/receipt.test.js` (24 tests) now import from `tests/contract/questions.js` and
`tests/contract/receipt-signing.js` — vendored, dependency-free snapshots of the server's pure
question-validation and receipt-signing logic, kept only so the client's assumptions about the
server's contract (the verbatim-span requirement; the receipt canonicalisation format) stay under
test in this repo. They are not shipped, are not covered by this repo's AGPL grant, and must never
be imported from shipped code — the client still never authors questions. `build.mjs`'s `EXCLUDE`
set no longer has a `server` entry, since there is nothing under `alcoia/` left to match. Any task
that requires a change to server logic is out of scope here; say so and stop.

Deleting the directory from the working tree does not remove it from git history. A history
rewrite or a fresh repository would be required if that ever matters — not attempted here.

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

✅ **Client side implemented.** `alcoia/src/shared/install-token.js` — `sra_install_token` in
`chrome.storage.local`, checked before every fetch and shared across overlapping callers so several
tabs waking at once still cost one request, not one each. **Loaded from content.js, not
background.js**, despite the actual AI-call fetch happening in the background worker: this file is
a genuine ES module, loaded the same way every other content module is (`loadModule()`, a dynamic
`import()`), and Chrome disallows dynamic `import()` from inside a service worker outright —
confirmed directly against real Chromium via `tests/browser/smoke.mjs`, not assumed from the spec.
content.js resolves the token and passes it to `background.js` as a plain string in the message
body (`callBackend()` in content.js); `background.js`'s `'summarize'`/`'apiPost'` handler is a dumb
relay that requires one to already be attached (`X-Alcoia-Install-Token` header) and refuses to
fetch without it. A 401/403 from the real endpoint is reported back as `tokenRejected`, which
content.js turns into `installToken.invalidate()` — the automatic version of "the reader deletes it
and gets a fresh one." Unit tests in `tests/install-token.test.js` cover issuance, the shared
in-flight dedup, and all three failure modes resolving to `null` rather than throwing.
`tests/browser/smoke.mjs` gained `FAIL=token` (the token endpoint itself returns 503) alongside the
existing `FAIL=questions`, plus an always-on check that every AI request the mock server received
carried the token header.

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

✅ **Diagnostics page implemented** — `alcoia/src/popup/diagnostics.html`/`.js`, opened from
popup.html's Developer section. Shows extension version, install-token status (masked to its last
4 characters — this page is required to be safe to screenshot, and the full token is a bearer
credential even though it identifies an install rather than a person) with a "Delete token" control,
every local `sra_*` setting, and a capped local log of AI calls that failed silently (invariant 9
means the reader never sees an error otherwise) via the new `diag-log.js`. **Plan and assist count
are not shown** — there is no account/entitlements endpoint anywhere in this repository or the
information to invent its shape, and building one is server work, out of scope here (see Scope).
The page states this plainly rather than fabricating a number; wiring it up is a follow-on item once
that endpoint exists. Nothing on the page reads the current tab, so it cannot leak a page's URL,
title or passage text by construction.

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

✅ **Implemented in `question-card.js`.** Clicking an option only selects it (`.sra-q-selected`);
grading and reveal happen once, from a confidence step that appears below the options with two
rating buttons ('low'/'high') and a "Rather not say" skip, all committing via the same `commit()`
path so no branch can grade without going through it. `response-signals.js`'s `answer()` takes
`confidence` as a third argument and normalizes anything other than the literal strings `'low'` /
`'high'` to `null` — never a guess. The four-outcome copy table below lives in `question-card.js`
as `CALIBRATION_COPY`; the skipped-rating case falls back to the bare `"That's right."` /
`"Not quite."` copy from item 12. Pinned by `tests/question-card.test.js` and the commit-time-
confidence describe block in `tests/response-signals.test.js`.

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

## THE TRAP — historical, kept for context on why the removal was done wholesale

✅ **Resolved: the gaze path — classifier.js, gaze-features.js, gaze-utils.js, webgazer-bootstrap.js,
webgazer.min.js, sra-page-bridge.js — has been deleted, not patched.** This section used to warn
against a specific way of getting that wrong; it is kept so the reasoning survives the deletion.

The trap was in `classifier.js`, which branched on `f.saccade_length`, `f.saccade_std`,
`f.velocity_mean`. Removing those keys from the extractor without retraining made every comparison
become `undefined <= X`, which evaluates to `false` **without throwing.** The classifier did not
crash. It routed down one branch forever and kept emitting confident labels:

```
classifyGazeState({})                   → { label: 'skimming', confidence: 0.722 }
focused sample, 3 saccade keys removed  → focused (0.993) becomes skimming (1.000)
```

That was pinned by `tests/classifier-missing-keys.test.js` and
`tests/classifier-feature-contract.test.js`, which guarded a file that no longer exists — both are
now deleted too, explicitly, rather than left passing vacuously against nothing.

The reasoning that made wholesale removal safe, for the next time a generated-artifact classifier
needs retiring: `orchestrator.js` gated the classify loop on
`!cfg.eyeTrackingEnabled || !host.getLastGazePoint()`. With no sensor there was no gaze point, the
loop returned early, and `classifyGazeState` was never reached even before deletion — so removing
the whole pipeline in one PR, rather than trimming feature keys while a classifier kept running
against them, could not trigger the undefined-comparison failure mode above. The trap applies to
deleting *feature keys while the classifier still runs* — a different and more dangerous operation
than the one actually performed here.

---

## Actual repository state

Verified by reading the tree. Line counts current as of this writing.

```
.
├── LICENSE                    AGPL-3.0 — repo root only, see DEFECT
├── build.mjs                  per-target package build
├── eslint.config.js           flat config, defect linter
├── manifests/
│   ├── base.json              shared manifest — THE SOURCE OF TRUTH
│   ├── chrome.json            service_worker
│   └── firefox.json           background.scripts + gecko.id
├── tests/                     Vitest suites (see Verification for current count) + browser smoke
│   └── contract/              vendored, dependency-free snapshots of the server's pure
│                               question/receipt logic — not shipped, tested here only
├── tools/question-quality.mjs question review harness
├── tldr classifier/           notebooks + synthetic CSVs — historical
└── alcoia/
    ├── manifest.json          GENERATED from manifests/. Do not hand-edit
    ├── background.js          service worker
    ├── src/content/
    │   ├── content.js           1471 — host: modules, settings, fetch, highlight, word lookup,
    │   │                               selection, keyboard, SPA nav, render callbacks
    │   ├── ui-controller.js      424 — popups, highlight, toasts, dark mode. Owns openPopups
    │   ├── state-engine.js       271 — signal fusion, one state estimate — telemetry only, no
    │   │                               gaze branch (see the migration note above)
    │   ├── orchestrator.js       266 — detectors, engine, budget, the one subscriber
    │   ├── comprehension-monitor.js 300 — pace vs. difficulty vs. personal baseline
    │   ├── receipt.js            282 — reader-owned session record + preview
    │   ├── reading-map.js        255 — sidebar minimap
    │   ├── focus-ruler.js        239 — reading band; already cursor/reading-line-first, gaze
    │   │                               was only ever the highest-priority of three sources and
    │   │                               is now simply never fed one
    │   ├── reading-calibration.js 188 — WPM calibration, self-paced only (the gaze-training
    │   │                               mode that used to pace the reader is gone)
    │   ├── question-card.js      234 — the retrieval question card; commit-time confidence step
    │   ├── dyslexia-utils.js     136
    │   ├── overlay-utils.js      132
    │   ├── pdf-handler.js        130 — partially wired, see defects
    │   ├── tts-handler.js        126
    │   ├── intervention-policy.js 267 — the interruption budget, one place: session cap scales
    │   │                               with tracked paragraphs / measured reading time
    │   │                               (baseAllowance + earned units, capped at absoluteCeiling
    │   │                               ~25), dismissal-aware backoff on consecutive question-card
    │   │                               dismissals, also exploration sampling (EXPLORATION_SAMPLE_RATE)
    │   ├── session-tracker.js     91
    │   ├── pptx-handler.js        78 — partially wired, see defects
    │   └── telemetry/
    │       ├── segmentation.js       225 — NOT a detector. Word/sentence counting, Intl.Segmenter
    │       ├── text-difficulty.js    185 — NOT a detector. FK + syntactic load
    │       ├── response-signals.js   124 — TIER 1, not telemetry. Placement is historical
    │       ├── session-recall.js     117 — recall pool (in-memory)
    │       ├── paragraph-tracker.js  158 — which paragraph is being read; figures/tables/pre are
    │       │                             tracked landmarks with media: true, not measured as prose
    │       ├── interaction-signals.js 100 — selection, copy, blur/return
    │       ├── cursor-tracking.js    101 — pointer-Y for paragraph-tracker's override; no
    │       │                              signal()/corroborating-type surface (the dead one was
    │       │                              deleted, not left unreachable — see Known defects)
    │       ├── scroll-regression.js   76 — paragraph-index returns
    │       ├── progression-entropy.js 76 — session shape
    │       ├── scroll-dynamics.js     59 — scroll jerk
    │       └── residual-distribution.js 53 — NOT a detector. Per-reader pace thresholds
    ├── src/popup/             popup.js, notes.js, 7 HTML pages
    │   ├── diagnostics.html/.js  version, masked token + delete, local settings, recent
    │   │                          AI-call failures — safe to screenshot, no URLs/titles/
    │   │                          passage text; opened from popup.html's Developer section
    │   └── diagnostics-format.js pure formatting helpers (maskToken, relativeTime,
    │                              escapeHtml) split out so they are unit-testable alone
    ├── src/shared/
    │   ├── config.js          the one place the backend origin is defined; classic script,
    │   │                       loaded before content.js, background.js and popup.js
    │   ├── install-token.js   the opaque per-install token; a real ES module, loaded only
    │   │                       from content.js (loadModule) — see Access control above
    │   └── diag-log.js        capped local log of silently-failed AI calls, feeding the
    │                           diagnostics page; sanitises out any http(s) URL before
    │                           storing, written to from content.js's callBackend()
    ├── src/styles/            fonts.css, overlay.css, panel.css, popup.css (legacy)
    └── src/libs/              pdfjs, jszip, fonts/ (OFL 1.1) — webgazer.min.js (GPLv3, 1.7 MB)
                                is deleted; see the migration note above
```

✅ **The gaze path is deleted, not demoted.** Removed entirely, not just from the tree above:
`src/libs/webgazer.min.js`, `src/content/classifier.js`, `src/content/gaze-features.js`,
`src/content/gaze-utils.js`, `src/content/webgazer-bootstrap.js`, `src/content/sra-page-bridge.js`,
`tests/classifier-missing-keys.test.js`, `tests/classifier-feature-contract.test.js`. Two modules
were deleted too even though they were not on the removal item's original file list, because reading
the actual code showed they existed solely to serve the gaze pipeline and had no other caller once
it was gone: `src/content/idle-overlay.js` (its only trigger was raw gaze features from the classify
loop) and `src/content/lang-detect.js` (its `detectScript()`/`watchScriptChanges()` output fed
nothing except the gaze classifier's script-aware feature patch). `alcoia/demo.html`, a manual QA
page whose every section existed to demonstrate a specific gaze-classifier trigger, is deleted for
the same reason — keeping it would describe a feature that no longer exists, which CLAUDE.md's own
opening line calls worse than no documentation at all. `ui-controller.js`'s
`updateGazeOverPopups`/`isGazeOverAnyPopup` (a second, gaze-point-driven autohide-pause mechanism
layered on top of the mouse-hover one) are deleted rather than left permanently unreachable; the
mouse-hover pause (`_mouseOver`) still works exactly as before. The `scripting` permission is
dropped from `manifests/base.json` — it existed only for the WebGazer main-world injection and one
popup-side content-script reinjection button, both gone. Package size: **~4.7 MB → ~3.0 MB** per
target, almost entirely the deleted 1.7 MB WebGazer bundle.

**Detector count, precisely.** Eleven files in `telemetry/`, but only **eight** export a `create*`
factory; of those, `response-signals.js` is tier 1 rather than telemetry, and `cursor-tracking.js`
has no `signal()`/corroborating-type surface at all — it exposes `getPointerY()`/`isTracking()`,
consumed directly by `paragraph-tracker.js` as a reading-position override, never by the state
engine. So **six live telemetry detectors** feed the engine, plus `comprehension-monitor.js`, the
primary sensor, which lives outside `telemetry/`. Do not describe this as "eleven detectors."

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

- ✅ **Resolved: `cursor_reading`.** Used to be dead twice over — `telemetry/cursor-tracking.js`
  emitted a `type: 'cursor_reading'` object via a `signal()` method; `state-engine.js` listed the
  type in `CORROBORATING_TYPES` (excluding it from asserting) but had no matching `CORROBORATION`
  entry, so the loop skipped it unconditionally; and `orchestrator.js` never called
  `cursorTracker.signal()` in the first place, using only `getPointerY()`. Nothing had ever decided
  what cursor tracking should corroborate in the engine, so inventing that policy retroactively
  would have been guessing. The dead emission path — `signal()` and the `cursor_reading` object —
  is deleted outright (see `cursor-tracking.js`'s header comment); the module's only surface now is
  `update()` / `getPointerY()` / `isTracking()` / `reset()`, consumed by `paragraph-tracker.js` as a
  reading-position override. `state-engine.js` carries no `cursor_reading` reference at all — grep
  confirms it. `tests/cursor-and-progression.test.js` pins the current surface directly (no
  `signal()` export to test against). If cursor evidence is wanted in the engine again later, that
  is a `state-engine.js` decision — a bonus, an evidence sentence, the states it applies to — made
  on purpose, the same way `selection`/`copy`/`scroll_jerk`/`progression` were, not a shape left
  lying around from before.

### Configuration

- ✅ **Fixed: `localhost:3000` no longer hardcoded anywhere.** All four sites — `content.js`
  (`BACKEND_DEFAULT`), `background.js` (fallback URL), `popup.js` (`sra_backend_url` default —
  the one that reaches storage), `popup.html` (placeholder, now set from JS instead of static
  markup) — read `self.ALCOIA_CONFIG.SUMMARIZE_URL` from the one new file,
  `alcoia/src/shared/config.js`, a plain classic script (not a module — none of the four contexts
  that load it are modules) loaded before each site that reads it: in `manifests/base.json`'s
  `content_scripts.js` before `content.js`; via `importScripts()` in `background.js` on Chrome and
  via `manifests/firefox.json`'s `background.scripts` array on Firefox; and via a `<script>` tag in
  `popup.html` before `popup.js`. `http://localhost/*` and `http://127.0.0.1/*` are gone from
  `host_permissions` in `manifests/base.json` — `<all_urls>` already covered them, so nothing was
  lost functionally, only the explicit dev-only declaration in the shipped manifest.
  `config.js`'s `BACKEND_ORIGIN` is currently a placeholder on the reserved `.invalid` TLD — no
  production origin has been assigned yet. **A developer pointing at a local backend still does
  not edit source or the manifest**: the popup's Settings → Backend URL field
  (`sra_backend_url` in storage) overrides the shipped default at runtime, exactly as before.
- ⚠️ **Partially fixed: icon sizes are wired to three distinct files, but the pixel content is
  still a placeholder.** `manifests/base.json`'s `action.default_icon` and `icons` used to point
  16/32/48/128 all at `assets/alcoia-mark-lilac.png` — one image, rescaled by the browser at every
  size, which turns to mush in the toolbar. They now point at three separate files —
  `assets/icon-16.png`, `assets/icon-48.png` (also used for 32, the closer neighbour), and
  `assets/icon-128.png` — so a real asset drop-in later needs no manifest change. **But the three
  files are currently identical copies of the same source PNG**, not separately drawn art, because
  no such art exists yet. This satisfies the letter of "wire the manifest for three distinct
  files" while explicitly not claiming the underlying defect (one image asked to look right at
  every size) is resolved. Ask the owner for real 16px/48px/128px art and replace the three files
  directly — no code or manifest change needed when that happens.
- ✅ **Fixed: the shipped package now contains a `LICENSE` file.** `build.mjs` copies the repo
  root `LICENSE` into `dist/<target>/LICENSE` during `build()`, rather than into `alcoia/` itself
  — one file to keep current instead of two that can drift. `tests/manifest.test.js` asserts it
  exists and is non-empty for every target.

### `gaze-features.js` — deleted, kept here for the record

This section used to document four measurement problems in the gaze feature extractor —
`dist > 30` as a fixation boundary sitting inside the webcam tracker's own ~180px error, a
scroll-blind line-band calculation, focused-looking fallback constants that violated invariant 5,
and a DBSCAN epsilon below the noise floor. The file, and the classifier that consumed its output,
are deleted rather than fixed — see the migration note under "Signal hierarchy" above. Kept as a
record of why "tune the thresholds" was never going to be the right fix for that module, in case a
future gaze-adjacent feature is proposed and the same reasoning applies.

### `pdf-handler.js` / `pptx-handler.js` — partially wired

Both carry `no-unused-vars` warnings for `backendUrl`, `fetchSummary` and `renderPopup`, suggesting
they accept dependencies they never use. Extraction, paragraph matching and the question path for
PDF and PPTX have not been verified end to end. **Verify before building file import on top.**

### Convention drift

`content.js` is **1471 lines** — down from 1754 after the gaze-path removal deleted roughly 280
lines of camera/calibration/tracking wiring, but still the file most in need of splitting further.
Two files exceed the ~300-line convention now (`content.js`, `ui-controller.js` at 424); it was six
before that removal. Settings live in `content.js` as loose `let`s read through accessors
(`settings()` / `getSettings()`) because the storage listener reassigns them and a captured copy
goes stale silently. **Add new logic to the new modules, never to `content.js`.**

---

## Known gaps in test coverage — read before trusting a green run

343 tests pass, in 20 files (two classifier-guard files were deleted alongside the classifier they
guarded — see the gaze-path migration note — and comprehension-monitor.test.js,
failure-paths.test.js and install-token.test.js are new). `npm run lint` exits 0 with 5 warnings
(all `no-unused-vars` in untouched files). These numbers drift with every PR; re-check with
`npm test` and `npm run lint` rather than trusting this line.

**The suite's failure mode is absence, not error.** Four instances so far:

1. A refactor silently deleted the entire keyboard-shortcut handler. **133 tests passed.** ESLint
   caught it, because no test pressed a key.
2. The question card and receipt shipped CSS in a file no code loaded. 216 tests plus a browser
   smoke check passed while the primary intervention rendered as unstyled HTML — the check clicked
   `.sra-q-option` and read `.sra-q-text`, both of which work perfectly on an unstyled button.
3. Every word count was `text.split(/\s+/)`, so a 600-character Chinese paragraph counted as **one
   word** and fell under every threshold. The extension loaded, ran and did nothing on a large
   fraction of the web. Nothing threw. Every test passed.
4. ✅ **Resolved:** `cursor_reading` — the dead emission path is now deleted rather than live in
   the tree; see "Dead code that reads as wired" above.

**Still open:**

- ✅ **Fixed: `comprehension-monitor.js` now has a unit test file**, `tests/comprehension-monitor.test.js`
  (16 tests, `@vitest-environment jsdom`, `vi`'s fake timers throughout since the module reads
  `Date.now()` directly). Covers the running-median WPM baseline (seeding, updating, persistence
  round-trip through the legacy key and the per-language map, and resistance to a single outlier
  sample), expected reading time scaling with word count and with difficulty grade, speed-mismatch
  in both directions, `MIN_WORD_COUNT` gating, and **the reader-to-self fairness safeguard directly**
  — the test that establishes a personal slow-but-steady pace and then confirms the safeguard stops
  flagging it while still catching a genuine outlier is the single most important one in the file.
  While adding this coverage, found and documented (not fixed, per this item's own instructions) a
  real invariant-5 concern: when `text-difficulty.js` cannot measure sentence structure at all (a
  script with no terminal punctuation, `structureIsUnreadable()`), it returns `score: 60, grade:
  'standard'` — a plausible default standing in for missing data — rather than a signal the caller
  can recognise as "no measurement available." `comprehension-monitor.js` never checks the `basis`
  field that would let it tell the difference, so it treats that default exactly like a real
  'standard' paragraph for both baseline calibration and speed-mismatch comparisons. Pinned by the
  "difficulty basis: structure unavailable" test. **Not fixed here — worth its own item**, and
  whichever of `text-difficulty.js` or `comprehension-monitor.js` ends up owning the abstention is
  an open design question, not obviously the former.
- **No end-to-end reading session** with clock advancement across minutes.
- **Question quality is untested.** `npm run questions:review` mechanises what can be mechanised —
  span really in the passage, question/span lexical overlap, giveaway distractors, conspicuous
  option lengths — and **flags rather than scores.** Judging whether a question tests understanding
  is a human job. Do twenty varied pages before shipping.
- ✅ **Fixed: invariant 9 is now guarded**, and one real violation of it was found and fixed while
  adding coverage. `tests/failure-paths.test.js` covers what is unit-testable directly
  (state-engine.js resolving unrecognised/absent/null telemetry to `unknown`; a denied or
  never-recorded decision never spending the interruption budget; question-card.js rejecting
  malformed question shapes). `tests/browser/smoke.mjs` gained a `FAIL=questions` mode that makes
  the mock server return a 422 for every question request, then asserts the endpoint was actually
  called, no card ever reached the screen, and the drop is visible in the debug log as
  `Interruption dropped before render (...) — budget not spent` — the only way to exercise
  content.js's own failure paths, since it is a single non-modular IIFE with nothing exported for
  a unit test to import. **The violation:** `handleAsk()` in content.js used to fall back to a
  full comprehension-offer popup (or, via a since-removed dead branch, a summary) whenever question
  generation failed for any reason — network error, 422, malformed response, or a legitimately
  empty result — which meant a failed *question* call quietly became a shown *explanation*, on a
  product whose stated design is that asking beats summarising. Fixed: `handleAsk()` now returns
  `false` (nothing shown, no budget spent) on any empty result, and the now-fully-dead
  `handleComprehensionSignal()` / `buildComprehensionOfferHtml()` — reachable only from that
  removed fallback and from an `onIntervention` branch that `STATE_ACTIONS` can never actually
  produce — are deleted rather than left as a second `cursor_reading`. Separately,
  `question-card.js`'s `show()` used to crash on a question missing both `span` and `q` (`.slice()`
  on `undefined`) and would otherwise render literally the text "undefined" for other missing
  fields; it now validates the full shape (`q`, all four `options`, `answerIndex`) and degrades to
  `false` instead. `triggerAIForParagraph()`'s PDF/PPTX text extraction is now wrapped in
  try/catch, falling through to the existing empty-text return, rather than an uncaught rejection.
  **Not addressed here:** install-token failure (no token mechanism exists yet — item 9) and
  storage read failure (already handled by `chrome.storage.local.get`'s own default-merging
  contract, not by code in this repo). See `tests/failure-paths.test.js`'s header for the full
  breakdown of what is covered where and why.

---

## Decisions already made — do not re-litigate

- ✅ **Camera removed, not just off by default.** There was no camera path as of the gaze-removal
  item — no `sra_eye` key, no permission, no sensor, nothing for a reading mode to accidentally
  turn on. If a future task proposes adding one back, that is a new invariant-1 decision, not a
  restoration, and needs the same scrutiny a first-time camera feature would get.
- **Telemetry is the only detection path.**
- **Questions, not summaries, are the primary intervention.** Summarising performs the operation
  that produces the learning. Follows D'Mello et al. (2016), d = 0.47.
- **State names describe observations:** `on_pace`, `skimming`, `struggling`, `drifting`, `absent`,
  `unknown`. Do not reintroduce `confused` or `overloaded` — those were the deleted gaze
  classifier's vocabulary, translated at the boundary while it still ran. There is no longer a
  boundary to translate at.
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
- Reading detection runs on telemetry alone — there is no camera path to be off
- No `getUserMedia` call, ever — there is no code path left that would make one
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