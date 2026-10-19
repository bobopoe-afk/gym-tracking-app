# Handover notes

Technical state of the app for whoever picks it up next. The README has the code
map; this covers how it's run, what's been built recently, the conventions the
owner cares about, and what's still open. Nothing personal lives in this repo
(it's public): training data stays in each account's Supabase row.

## Running and shipping

- **Live:** https://bobopoe-afk.github.io/gym-tracking-app/ (GitHub Pages, served
  from `main`). Pushing to `main` is a deploy.
- Work has been done on `claude/fervent-clarke-0b58rr` and pushed to both that
  branch and `main` after each change, so the two are identical at handover
  (`0b7f358`).
- No build step: everything is `index.html`. Accounts and sync are Supabase
  (`SUPABASE_SETUP.md`); the project URL and publishable key are in `CLOUD` in
  `index.html`, which is by design. Never put the service_role key in the app.

## How changes were checked

There's no committed test suite. Each change was checked with throwaway
Playwright scripts (Node 22, `NODE_PATH=/opt/node22/lib/node_modules`), roughly:

- launch Chromium with iPhone 13 emulation, abort requests to fonts and jsdelivr,
  open `file://…/index.html`, wait until `DATA` exists and `#app .loading` is gone;
- replace `DATA` with a backup export (`DATA = JSON.parse(raw); migrateData();
  rebuildPRRecordsFromHistory(); render();`) and call the app's own functions
  (`openWorkoutMode`, `deterministicSuggestionFor`, `parseSessionPasteText`,
  `openDetailModal`, …), tapping real buttons where the flow matters;
- for sync, a stub of the Supabase client in place of the CDN script.

Ask the owner for a fresh backup (footer → Export data) if you need real data;
don't commit it. A small committed harness would be a good addition.

## Conventions the owner cares about

- As few questions and as little typing as possible; mobile first; UK English.
- Coaching preferences (weekly lifts, a big lift each session, variety slot,
  alternating free-weight/machine push days) are per account and **off by
  default**: friends use the app too, and nothing should force one person's
  preferences on everyone.
- Runs are only moved with the user's consent (clash fixes offer it; they never
  do it silently).
- Undo instead of "are you sure" wherever possible.
- Built-in name clean-ups (`NAME_FIXUPS`, `DUPLICATE_MERGES`) run on every load:
  they must never undo something the user chose. Exercises renamed by the user
  carry `renamedByUser`; old names go in `aliases` so pasted logs still match.

## Recently built (newest first)

- Rename any exercise from its page; history is keyed by id so it all follows
  (`renameExercise`, `aliases`). "Booty Builder / hip thrust" is now
  "Hip thrust machine" (it had been wrongly tidied to "Barbell hip thrust").
- Reopening a finished or past workout no longer adds "to do" sets for planned
  sets that weren't logged (`isRecord` in `openWorkoutMode`).
- Gyms list (footer → Gyms): put past sessions at a gym after the fact, one by
  one or all unset at once (`setPastSessionGym`, `sessionsWithMachineWork`).
- "Assisted pull-up (machine)" folded into Pull-up, which takes assistance
  directly (`DUPLICATE_MERGES`); weights typed on exercises named "assisted…"
  are read as assistance (`assistify`).
- Delete any set mid-workout (tap the set number or swipe left).
- Assisted sets (`{type:'assist'}`, shown as −9kg) in typing, pasting and the
  steppers; superset headings in pasted logs are treated as labels.
- Per-set ticks, cheers and PR toasts; soonest-first Coming up; copyable week.
- Run/leg-day clash detection after planning, with consented fixes.
- Coaching preferences, weekly lifts, alternating push-day equipment.

## Open ideas and known issues

- Races only get a 2-day buffer from leg days (`runBuffer`); a half or full
  marathon deserves more before (about 4–5 days) and after. The weekly-heavy-lift
  preference also doesn't know about race week.
- Progression: a load jump resets reps to the bottom of the range, which is harsh
  on small jumps (12.5kg×12 → 13.5kg×6); and generated warm-ups can stack on top
  of the user's own ramp sets (lat pulldown suggestion with six sets).
- Reps in reserve is almost never logged, so effort can't be judged; a one-tap
  way to mark the last set as "hard / had more" would help.
- "Hip Adductor" and "Hip adduction machine" look like one exercise logged
  twice; offer a merge (now easy with renames + `DUPLICATE_MERGES` style folding,
  or a user-facing Merge).
- The `alt.js`-style test for alternating push days predates the move of that
  toggle into Coaching preferences.
