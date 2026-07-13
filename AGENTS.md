# AGENTS.md — BoostDoc

Guidance for AI coding agents (OpenAI Codex, etc.) working in this repository.
This is the Codex counterpart to a CLAUDE.md and describes how the project is
built, tested, and structured, plus the domain gotchas that are easy to get wrong.

## What BoostDoc is

An **offline, zero-dependency Progressive Web App** that analyses BMW tuning logs
(MHD and Datazap CSV exports) for boost, timing, fueling, WGDC and IAT problems.
Everything runs client-side in the browser — open `index.html` and load a CSV. No
server, no build step, no npm packages at runtime.

- Primary engine focus: N54, with presets for N55, B58 Gen1/Gen2, S55, S58, S63.
- UI language is German; the app is localised into 7 languages via `i18n.js`.

## Repository layout

| Path | Role |
|------|------|
| `index.html` | App shell + static markup (upload zone, panels, settings, bottom nav) |
| `analyzer.js` | **Pure analysis logic.** CSV parsing, column mapping, unit conversion, row-state classification, boost/timing/fueling/WGDC/IAT evaluation. UMD dual export (Node + browser global `window.LogAnalyzer`). No DOM. |
| `app.js` | **UI layer.** Reads `window.LogAnalyzer` + `window.BoostDocI18n`, renders results, charts, comparison, trend, vehicle profiles, presets. Owns all DOM. |
| `i18n.js` | Translation tables + `t()` helper. Browser global `window.BoostDocI18n`. |
| `styles.css` | All styling. Light/dark themes via `body[data-theme]`. Responsive down to phones. |
| `manifest.json` | PWA manifest. |
| `tests/` | Node-based `.cjs` regression tests (see Testing). |
| `tests/expected-results/` | 25 JSON snapshots of expected analysis output for anonymised fixtures. |
| `test-data/` | Anonymised sample CSVs + `anonymized/` fixtures used by the fixture test. |
| `docs/BoostDoc_Codebasis_Anfaenger.md` | Beginner-oriented codebase walkthrough (German). |
| `CHANGELOG.md` / `PATCHNOTES.md` | Release notes (newest first). |

## Setup, build, test

Requires **Node ≥ 18** (developed on Node 22). No `npm install` needed — there are
zero dependencies.

```bash
npm run lint    # node --check on analyzer.js, app.js, i18n.js (syntax only)
npm test        # runs all 5 regression suites
npm run build   # lint + test (use this as the pre-push / CI gate)
```

There is **no bundler and no transpile step** — "build" just means lint + tests pass.
To run the app, open `index.html` directly in a browser (or serve the folder
statically); there is nothing to compile.

## Testing

Five suites, all plain Node with `node:assert` (no framework):

- `analyzer-state-aware.test.cjs` — row classification (WOT / burble / shift / idle / data-error).
- `analyzer-csv-robustness.test.cjs` — CSV format edge cases, unit conversion, column mapping, diagnosis routing.
- `i18n.test.cjs` — translation table integrity.
- `analyzer-i18n.test.cjs` — analyzer issue `i18nKey`s resolve to real translations.
- `analyzer-fixtures.test.cjs` — runs all 25 anonymised fixtures against `tests/expected-results/*.json` and asserts no private identifiers leak into fixtures.

Rules for changing tests:
- **Keep all 5 green.** Run `npm test` after every change to analysis logic.
- The fixture test asserts a small, stable set of fields (rows, platform, status, pullCount, fuel). Don't casually edit expected-results snapshots — a snapshot change means you changed analysis behaviour, and that must be intentional and documented in `CHANGELOG.md`.
- **Never commit real logs or private data.** Fixtures must be anonymised: no original filenames, VIN fragments, private IDs, or local paths.

## Domain gotchas (read before touching `analyzer.js`)

These are subtle and have caused real bugs before:

- **Pressure units.** `pressureToPsi(value, column)` converts by unit tag in the column name:
  - `bar` ×14.5038, `MPa` ×145.038.
  - **`hPa` is always treated as gauge** (×0.0145038). This is proven by WG-position data (WGpos=0% at spool-up ⇒ actual<target ⇒ both gauge). The old `value>1200` absolute-detection heuristic was **removed in v1.6.3** — do not reintroduce it.
  - **`kPa` keeps absolute-pressure detection** (subtract 100 kPa ambient for boost/intake/manifold columns >120 kPa). This intentionally differs from hPa because no Datazap kPa fixture exists to confirm gauge-only. Do not "unify" the two branches.
- **`normalizeTime()`** divides ms-based time columns (`Time [ms]`) by 1000 at every read site so pull durations are in seconds. If you add a new time read, normalise it too.
- **`boostMani` column mapping is legacy/unused** — mapped but not read by analysis, charts, or the channel checklist. Do not remove without a dedicated column-map review (its regex overlaps the `boost` column).
- **`boostDeviation`** is mapped only so the `CHANNEL_CHECKLIST` (in `app.js`) can report column presence. Not used for metrics.
- **S63 preset is a placeholder** copied from `b58_gen2`. Thresholds (`wgdcWarnAvg`, `boostEvaluationMinRpm`, IAT, `timingWarn`) are NOT calibrated — see the TODO block above the `s63` entry in `RULE_PRESETS`. Needs real S63 V8 bi-turbo logs before trusting those numbers.
- **State-aware evaluation:** boost/timing/fueling are only hard-scored on rows classified as clean WOT. Burble/shift/idle/data-error rows are shown as *context*, never as hard red/yellow findings. Keep new checks state-aware.
- **`analyzer.js` must stay DOM-free** so the Node tests can `require()` it. Anything touching the DOM belongs in `app.js`.
- **Preset ↔ dropdown parity:** every key in `RULE_PRESETS` that a vehicle profile can route to must exist as an `<option>` in the `#rulePreset` select in `index.html`, and `presetKeyForEngine` in `app.js` must mirror `presetKeyForVehicle` in `analyzer.js`.

## Conventions

- Vanilla ES (no TypeScript, no framework, no build tooling). Match the surrounding style.
- UI strings go through `i18n.js` — don't hard-code user-facing text in `app.js`.
- Commit messages and CHANGELOG entries in this repo are typically written in German; keep the `type: subject` prefix (`fix:`, `docs:`, `style:`, `chore:`).
- CHANGELOG.md is newest-first; add a new `# BoostDoc vX.Y.Z` section at the top for any behaviour change.

## Branch / repo status (handoff note)

- The GitHub repo was renamed **K7Unit/K7Gang → K7Unit/BoostDoc**.
- `main` currently contains the latest polish (mobile CSS centering + 44px tap
  targets, a dead-CSS/breakpoint cleanup, and these `build`/`lint` scripts) that
  landed via squash-merge.
- The development branch **`claude/codex-app-dev-RJDfl`** carries the full history
  and the v1.6.4 documentation work + the s63-dropdown fix, and now this
  `AGENTS.md`. If it is behind `main` on the CSS cleanup, open a PR to reconcile
  before continuing feature work.

## Where we left off (v1.6.4)

Done: comment-only documentation cleanup of `analyzer.js` (kPa/hPa rationale,
`boostMani`/`boostDeviation` notes, actionable S63 TODO), `s63` dropdown fix,
mobile layout fixes, CSS consolidation, `build`/`lint` scripts.

Open follow-ups:
- Calibrate the S63 preset with real logs (see the TODO in `RULE_PRESETS`).
- Optional: a `serve`/dev-server convenience script (currently you just open `index.html`).
