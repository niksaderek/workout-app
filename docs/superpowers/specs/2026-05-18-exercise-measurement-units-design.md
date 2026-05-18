# Design: Per-Exercise Measurement Units (reps / seconds / meters)

**Date:** 2026-05-18
**Status:** Approved (design), pending implementation plan
**File affected:** `index.html` (single-file React PWA)

## Context

Some exercises are not measured in reps. Side Plank is held for **seconds**;
Farmer's Walk / sled work covers a **distance in meters** (often loaded).
Today the app has only a `reps` field, and exercise-type behavior (core /
bodyweight) is decided by `name.toLowerCase().includes(...)` checks
duplicated in ~8 places. This feature lets the second logged value mean
reps, seconds, or meters per exercise, while centralizing the scattered
type-detection logic.

User decisions captured during brainstorming:
1. **Unit assignment:** auto-detect by name **+ manual override**.
2. **Weight relevance:** depends on exercise — planks/holds = seconds, no
   weight; Farmer's Walk/sled = meters **with** weight.
3. **Stats impact:** track separately — seconds/meters excluded from
   reps/volume/1RM/strength stats (same treatment core gets today).
4. **Existing data:** auto-detect, **no migration** — saved records are
   never rewritten.
5. **Override scope:** stored on the **template exercise** (`exercise.unit`),
   copied into history at `startWorkout`.

## Goals

- A logged set's second value can represent reps, seconds, or meters.
- Auto-detect the unit from the exercise name; allow per-exercise manual
  override in the template editor.
- Seconds/meters exercises do not pollute rep/volume/1RM/strength stats.
- Zero migration; full backward compatibility with existing saved
  templates, history, and v1.1 backups.
- Reduce the existing duplicated `isCoreExercise` detection by routing it
  through one helper.

## Non-Goals (YAGNI)

- No "seconds + weight" combination (weighted plank is an edge case; use an
  override and accept no weight field — explicitly out of scope).
- No one-time data migration.
- No new dedicated stat views (plank-time chart, total-distance card).
- No CSV schema change (the `Reps` column keeps its name; value is still
  correct, e.g. `60` for a 60-second plank). Deliberate non-change to avoid
  breaking the existing spreadsheet / Croatian-locale workflow.
- No new plateau/1RM math for time/distance in the recommendation engine.

## Architecture

### Core abstraction (new, near `getMuscleGroup`)

```js
// Returns 'reps' | 'seconds' | 'meters'. Takes the exercise OBJECT so it
// can read an explicit override; name-only callers pass { name }.
const getExerciseUnit = (exercise) => {
  if (exercise.unit === 'reps' || exercise.unit === 'seconds' || exercise.unit === 'meters')
    return exercise.unit;                       // explicit manual override wins
  const n = (exercise.name || '').toLowerCase();
  if (/\b(farmer|carry|walk|sled|yoke|loaded carry)\b/.test(n)) return 'meters';
  if (/\b(plank|hold|wall sit|dead hang|hang)\b/.test(n))       return 'seconds';
  return 'reps';
};

const UNIT_CONFIG = {
  reps:    { label: 'reps', placeholder: 'reps', hasWeight: true,  countsAsReps: true,  countsAsVolume: true  },
  seconds: { label: 'sec',  placeholder: 'sec',  hasWeight: false, countsAsReps: false, countsAsVolume: false },
  meters:  { label: 'm',    placeholder: 'm',    hasWeight: true,  countsAsReps: false, countsAsVolume: false },
};
```

- `meters` is matched **before** `seconds` so "Sled Walk" → meters.
- The data value continues to live in the existing `set.reps` field. It is
  **reinterpreted**, never renamed — this is what makes it zero-migration.
- `countsAsReps/Volume: false` for seconds/meters mirrors today's core
  exclusion.

### Data model

- New optional field `exercise.unit` (`'reps'|'seconds'|'meters'`) on a
  template exercise. Absent ⇒ auto-detect. Setting "Auto" in the editor
  deletes the field.
- `startWorkout()` copies `unit: ex.unit` into the logged exercise object
  alongside the existing `plannedReps` / `muscleGroup` copy, so history
  records carry whatever was in effect.
- IndexedDB schema unchanged (still v2). Backup/restore v1.1 unaffected
  (`unit` rides along if present, ignored if absent) — no version bump.

## Components / Changes

### 1. Logging set-row (≈ lines 4291, 4323–4352)

- Compute `const unit = getExerciseUnit(exercise); const cfg = UNIT_CONFIG[unit];`
- Weight input `disabled={!cfg.hasWeight}` — replaces the inline
  `includes('plank')...` string check; keeps the existing dimmed style.
- The `×` separator renders only when `cfg.hasWeight` (no "× 60" for a
  bare plank).
- Second input: `placeholder={cfg.placeholder}`, still bound to
  `set.reps`, `inputMode="numeric"`.
- "Target:" line → `{plannedSets} × {plannedReps} {cfg.label}`
  (e.g. "3 × 60 sec", "3 × 40 m").

### 2. Exercise editor (template Edit view)

- Add a small per-exercise unit selector: **Reps / Seconds / Meters**,
  defaulting to a visible "Auto (detected: X)" state when `exercise.unit`
  is unset. Selecting a unit writes `exercise.unit`; selecting "Auto"
  deletes it. Styling follows the existing sets/reps input row; no broader
  layout redesign.

### 3. Stats / consumers (the "track separately" rule)

Route the ~8 duplicated `isCoreExercise` / `isBodyweightExercise`
decisions (lines ≈ 1677, 2727, 2807, 2909, 3066, plus logging/suggestion
sites) through the unit helper:

- `unit === 'reps'` → behaves exactly as today (no number changes).
- `unit === 'seconds' | 'meters'` → excluded from Total Reps, Volume
  (`weight×reps`), Estimated 1RM, Strength Standards — same as core today.
- Generalize the *condition* (`!UNIT_CONFIG[getExerciseUnit(ex)].countsAsVolume`),
  do **not** remove core logic. Plank auto-detects to `seconds`, so
  existing core behavior does not regress.
- **Recommendation engine** (`getExerciseHistory` / `getSmartWeightSuggestion`):
  for seconds/meters, skip the e1RM/plateau math (meaningless for
  time/distance); reuse the difficulty fallback with unit-aware wording
  (e.g. "last time 50 sec"). No new plateau logic — modest, unit-aware.

### 4. CSV export

No change. Header stays `Date,Workout,Exercise,Set,Weight,Reps`; the value
is already correct for seconds/meters.

## Edge Cases

- **Weighted plank:** auto = seconds, no weight. No seconds+weight combo by
  design (override available but still no weight field). Explicit non-goal.
- **Ambiguous custom names** (e.g. "Walking Lunges" contains "walk"):
  word-boundaried, tight regex minimizes misfires; manual override is the
  escape hatch.
- **Blank seconds/meters value:** same as a blank rep today — filtered out
  by existing `reps > 0` guards.
- **Old data:** "Side Plank" `reps:45` → "45 sec"; "Farmer's Walk"
  `reps:30` → "30 m" + weight. Nothing rewritten.

## Testing (local, before any push — standing user rule)

**Tier 1:** visual code review of the helper + every changed call site.

**Tier 2:** Playwright over `http://localhost` (the styled path; opening
`file://` breaks `/styles.css`). Seed IndexedDB scenarios, restore after,
never touch `workouts`/`bodyWeight`:

1. Old Plank history (no `unit`) → "sec", no weight field, excluded from
   Volume & Total Reps.
2. Old Farmer's Walk → "m" + weight field, excluded from rep/volume stats.
3. Manual override: set a custom exercise to Seconds in editor → start
   workout → input labeled "sec".
4. Override → Auto reverts to detection.
5. Regression: a normal reps exercise (Bench Press) — stats, volume, 1RM,
   and recommendation engine unchanged vs. current behavior.
6. Console clean (0 errors; the ~5 preload/Babel warnings are pre-existing
   noise).

## Backward Compatibility

- No saved record rewritten; `exercise.unit` absent on all existing data ⇒
  name auto-detection.
- IndexedDB schema unchanged (v2). Backup/restore v1.1 unaffected; no
  version bump.
- Existing core handling intact (generalize condition, not remove logic).

## Deploy Note

User controls deploy and the `sw.js` `CACHE_NAME` bump — not part of this
spec. Implementation will not push or bump the service worker unless asked.
