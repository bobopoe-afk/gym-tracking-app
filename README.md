# Training Log

A single-file training log app: exercise tracking, progression suggestions,
session planning, PRs, and a running-schedule overlay. No build step, no
dependencies — everything (HTML, CSS, JS) lives in `index.html`.

## Running it

Just open `index.html` in a browser. That's it.

## ⚠️ Storage — the one thing to know before you touch anything

This app was built and run inside a Claude.ai artifact, which offers an
optional `window.claude.use('db')` bridge for account-level, cross-device
storage. The code:

1. Waits briefly for `window.claude` to appear (`waitForClaudeBridge`).
2. If found, tries to use it for storage (`STORAGE_MODE = 'db'`).
3. If not found, or anything about it fails, it **falls back to plain
   `localStorage`** — silently, with no error shown to the user.

**Outside of Claude.ai (i.e. anywhere you run this from now on), step 1 will
always fail** — there's no `window.claude` on GitHub Pages, Vercel, a local
file, etc. That's not a bug; the fallback was built for exactly this. But it
means:

- Data is saved per-browser (`localStorage`), same as before you had the
  Claude db integration working.
- **No cross-device sync** — this was the entire point of the db capability,
  and you'll lose it the moment this runs outside Claude.
- The "Storage debug" link in the footer will report the bridge as never
  found. That's expected here, not an error.

**Accounts and cross-device sync** are built in (Supabase): fill in `CLOUD`
in `index.html` and follow `SUPABASE_SETUP.md`. Each person signs in with
email + password and gets their own log on every device; with `CLOUD` left
empty the app stays browser-only as described above.

Data export/import (footer: Export data / Import file / Copy data / Paste
data) works regardless of storage backend, since it just serializes `DATA`
directly — handy for migrating data once you've picked a new backend.

## Orientation — it's one big file, here's the map

Search for these to jump around:

- `const DATA = ` / `seedData()` — the whole app's state shape and the
  sample data it starts with on a fresh install
- `function loadData()` / `function persist()` — storage (see above)
- `const CATS = ` — exercise movement groups (push/pull/leg/bilat/aux) plus
  combined session types (legs/upper/lower/full) via `groups`;
  `SPLIT_STYLES` / `sessionTypes()` — the user's chosen split (PPL,
  upper/lower, full body, or the original five)
- `function buildSessionPool` / `function buildPoolForSession` — decides
  which exercises get suggested for a new session, in what order
  (compound-first, then accessory; aux always last)
- `function decideProgression` / `function deterministicSuggestionFor` —
  the progression engine: reads your actual logged history for an exercise
  and works out a next-step suggestion
- `function repRangeFor` — per-exercise rep range (inferred by name, or an
  explicit override you've set)
- `function openWorkoutMode` — the main session editor: logging sets, the
  exercise search/swap picker, reordering
- `function openDayModal` — the Calendar tab's per-day view: logged +
  still-planned exercises, Move/Delete/Copy/Open/Plan-like-this
- `function planSpecificSession` — "Plan like this": copy a past session's
  exercises into a new planned session on a chosen date, with progression
  applied to the targets (nothing gets marked as done)
- `function repeatSpecificSession` — the Today card's "repeat last session"
  quick action: same idea, but logs it immediately as done (used with a
  confirm step, since it writes real data)
- `/* ---------- gyms` — per-gym history for machine/cable exercises. The
  workout screen's gym chips set `session.gym`; logged sets carry that id
  as `machine` (from `DATA.machines`), and suggestions, PRs
  (`prKey(exId, gymId)`), charts and 1RM estimates only compare
  machine/cable entries from the same gym. Free weights are never tagged.
- `function sessionStatus` — planned / in-progress / completed / missed,
  computed per-exercise rather than as a single flag (this matters: a
  session with 1 of 5 exercises logged is "in-progress," not "completed")

## What's already been fixed / built out

Worth knowing so you don't rediscover these the hard way:

- Warm-up ramps for barbell compounds are generated fresh each session
  rather than copied forward from your last logged ramp (which was
  fatiguing people before their work sets).
- The exercise picker (add/swap) is a live search, not a long alphabetical
  `<select>`.
- A session's exercise order defaults to compound-first, accessory-second,
  with aux/core always last — unless you've manually reordered it, which
  always wins.
- Logging one exercise in a session no longer hides the rest of what was
  planned for it (a real bug — display-only, never touched the underlying
  data, but fixed).
- Set roles (`classifySets`): the log doesn't store them, so they're read
  from the session's shape — heaviest load = top set, lighter sets after it
  = back-offs, lighter sets before it = warm-ups unless they're 85%+ of the
  top weight with as many reps (ascending working sets). Progression keeps
  last session's structure and adjusts it (`decideProgression`); muscle
  set counts and target verdicts ignore warm-ups.
- Repeat / Plan like this / templates only ever create targets; sets become
  history only by logging them. A workout opened on a blank session pins
  its exercises into `session.targets` so the list is explicit; Finish sets
  `session.finished`.
- Saving: `persist()` reports failures ("Not saved!" + export prompt);
  outside Claude there's no 12s wait for the bridge and AI-only controls
  are hidden. Unfinished workout sets are kept in localStorage
  (`workoutDrafts:<sessionId>`).
- Logging is tap-based: each exercise in the workout screen is pre-filled
  with today's sets (plan targets, else the progression suggestion); ✓ logs
  them, +/− adjusts weight or reps (a weight change carries to the later
  sets at the same weight). Typing is still available behind "Type".
- Machine and cable weights are tracked per gym: pick the gym once at the
  top of a workout (remembered). History from before gyms existed stays
  untagged and is used as a labelled fallback until a gym has its own.
- Pop-ups don't auto-focus fields on touch screens and the page behind
  them is scroll-locked — both caused the page to jump on phones.
- PR records are a derived cache: rebuilt from `DATA.cells` on load and on
  every render (`buildPRRecords`), so a corrected or deleted set can't
  leave a phantom record behind. Sets with fractional reps (a typo like
  "21.6 reps") are left out of PRs and flagged with a one-tap Fix.
- One volume definition (`setVolumeKg`): weight × reps, "each side" sets
  counted twice. Weights are taken as logged — the app can't tell whether
  "14kg" dumbbells meant one or a pair, so tonnage for paired dumbbell work
  reads low.
- Estimated 1RM (stat, calculator and PR) only for compound lifts
  (`usesE1RM`).
- Warm-ups: empty bar ×10, then fewer reps as weight climbs, ending with a
  ~92% single on heavier days; deadlifts skip the bar, machines get a
  short ramp, isolation work none.
- No rest timer. It was removed on request — it fired unpredictably from
  three different logging paths with no way to see it coming or configure
  it.
