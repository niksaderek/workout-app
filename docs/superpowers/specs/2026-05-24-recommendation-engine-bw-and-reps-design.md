# Recommendation Engine: Bodyweight-Aware Progress, Rep Progression, and Strength-Standard Mapping Fix

**Date:** 2026-05-24
**Status:** Approved (awaiting user spec review)
**Touches:** `index.html` — `getSmartWeightSuggestion`, `getExerciseHistory`, `getMainLifts1RMs`, `dbOperations`

## Context

The current proactive recommendation engine (`getSmartWeightSuggestion`) detects plateau and progression using only the raw e1RM (Epley) of the lift itself. Two real-world progress signals are missed:

1. **Rep progression at same weight, below planned target.** A user going 100 kg × 6 → 100 kg × 7 (target 8) bumps e1RM, so the plateau check correctly does NOT fire — but the engine has no message for "you added a rep, keep climbing"; it silently falls through to the difficulty fallback.
2. **Bodyweight-relative progress.** A user who holds the same lift weight/reps while losing 1–2 kg of bodyweight is getting stronger. The engine never reads from the `bodyWeight` IndexedDB store, so this gain is invisible. Same for bodyweight exercises (pull-ups, push-ups, dips, chin-ups): same reps at a lower bodyweight = a strength gain.

Separately, the Strength Standards / 1RM panel uses fuzzy substring matching to assign exercises to main-lift categories. "Squat" catches Leg Press, Hack Squat, Bulgarian Split Squat, Goblet Squat — inflating/distorting the squat 1RM and the strength-level progress bar.

## Goals

- Add a **rep-up** branch that fires when reps increased at the same-or-higher weight but the planned-rep target was not yet hit.
- Add a **bodyweight-relative override** that reclassifies a raw-metric plateau as progress when the lifter's strength-to-bodyweight ratio improved.
- Extend the override to bodyweight exercises by using an `effectiveE1RM = bodyWeight × (1 + reps/30)` proxy.
- Replace the fuzzy main-lift match with an explicit whitelist per category so non-competition variants do not pollute the strength standards.

## Non-Goals

- Schema changes (no new IndexedDB fields, no migration).
- UI changes to the logging view (the new branches reuse existing `type` values: `progress`, `rep-up`, `hold`).
- Body-weight trend analysis beyond comparing two snapshot points (current vs the workout that set the earlier best).
- Changing the existing plateau definition, deload trigger, or swap logic.
- Reworking `getAll1RMs` (it returns every exercise; the whitelist applies only at the main-lift consumer).

## Design

### 1. Bodyweight history lookup

**New `dbOperations.getAllBodyWeights()`** — cursor over the `bodyWeight` store, returns an array sorted ascending by `date`.

**App-load wiring** — mirror the existing `completedWorkouts` load pattern: on mount, read all body weights into a new state array `bodyWeightHistory` (cached for the session). Refresh after `handleSaveBodyWeight` so a new BW entry is immediately visible to the engine.

**New helper `getBodyWeightAt(dateISO)`** — pure function over `bodyWeightHistory`:
- Returns the entry whose `date` is closest to `dateISO` (any direction).
- No max window — users log BW infrequently; the nearest entry is the best available estimate.
- Returns `null` when `bodyWeightHistory` is empty.

### 2. Augment `getExerciseHistory`

Add two fields to each returned entry (no behavior change for existing consumers):
- `bodyWeight` — number or null, via `getBodyWeightAt(workout.date)`.
- `isBodyweight` — boolean, via `isBodyweightExercise(exerciseName)`.

### 3. Relative metric

Per history entry, define a single `relMetric`:

- **Weighted lift** (`!isBodyweight`, has `bestE1RM > 0`, has `bodyWeight`): `relMetric = bestE1RM / bodyWeight`.
- **Bodyweight exercise** (`isBodyweight`, has `bodyWeight`, has `topSetRepsHit > 0`): `relMetric = bodyWeight × (1 + topSetRepsHit / 30)` (effective e1RM; absolute load proxy that increases when BW grows OR reps grow).

  Note: for bodyweight exercises the "relative" metric is intentionally absolute. Dividing by BW would cancel BW out and reduce to a pure rep count. We want the override to fire when reps held but BW dropped — i.e., the user is performing the same work at a lower load — which actually corresponds to **decreased** effective e1RM, not increased. Therefore the bodyweight branch flips the comparison: see step 5.

- **Otherwise:** `relMetric = null` (override cannot fire).

### 4. Engine decision flow (revised order)

1. No history → `null` *(unchanged)*
2. Non-reps unit branch *(unchanged)*
3. Single history → `difficultyFallback` *(unchanged)*
4. Compute window / `isPlateau` / `allHard` on raw metric *(unchanged)*
5. **NEW: Bodyweight-relative override** (see step 5 below).
6. `isPlateau && allHard` → deload *(unchanged)*
7. `isPlateau` → swap or hold *(unchanged)*
8. **NEW: Rep-up at same-or-higher weight, below planned target** (see step 6 below).
9. `topSetRepsHit >= plannedRepTarget` → existing rep-range progression branch *(unchanged)*
10. Default `difficultyFallback` *(unchanged)*

### 5. Bodyweight-relative override

Runs only when `isPlateau` is true on the raw metric.

Compute `relMetric` for `latest` and for the `earlierBestEntry` (the history entry whose raw metric equals `earlierBest`). Both must be non-null.

**Weighted lift:**
- If `latestRel > earlierRel × 1.01`:
  - `type = 'progress'`
  - `weight = lastWeight.toFixed(1)`
  - `reason = "Same lift, you're {Δkg} kg lighter — relative strength up, keep going"` where `Δkg = (earlierBW - latestBW).toFixed(1)`.

**Bodyweight exercise:**
- If reps held or grew (`latest.topSetRepsHit >= earlierBestEntry.topSetRepsHit`) **and** BW dropped (`latestBW < earlierBW × 0.99`):
  - `type = 'progress'`
  - `weight = ''` (bodyweight exercise — no load suggestion)
  - `reps = String(latest.topSetRepsHit)`
  - `reason = "Same reps at {Δkg} kg lighter bodyweight — that's a strength gain"`.

If neither condition matches, fall through to existing plateau handling.

### 6. Rep-up branch

Runs after plateau handling, before the existing rep-range progression branch.

Guards:
- `unit === 'reps'`
- `history.length >= 2`
- `plannedRepTarget !== null`
- `prev = history[history.length - 2]` exists with `bestWeight > 0` and `topSetRepsHit > 0`

Rule:
- If `latest.bestWeight >= prev.bestWeight` (same or heavier load) **and** `latest.topSetRepsHit > prev.topSetRepsHit` (more reps) **and** `latest.topSetRepsHit < plannedRepTarget`:
  - `type = 'rep-up'`
  - `weight = lastWeight.toFixed(1)`
  - `reason = "+{Δreps} rep last time — push for one more to hit {target}"`.

Hitting or exceeding `plannedRepTarget` is already handled by the existing branch at step 9 (which advises +2.5 kg or +1 rep depending on difficulty); leave it alone.

### 7. Strength-standard mapping fix

Replace fuzzy substring matching in `getMainLifts1RMs` with an explicit whitelist:

```js
const MAIN_LIFT_WHITELIST = {
  Squat:            ['Back Squat', 'Front Squat', 'Squat', 'Box Squat', 'Pause Squat', 'High Bar Squat', 'Low Bar Squat'],
  'Bench Press':    ['Bench Press', 'Barbell Bench Press', 'Flat Bench Press', 'Competition Bench Press', 'Pause Bench Press'],
  Deadlift:         ['Deadlift', 'Conventional Deadlift', 'Sumo Deadlift', 'Trap Bar Deadlift', 'Hex Bar Deadlift'],
  'Overhead Press': ['Overhead Press', 'OHP', 'Standing Overhead Press', 'Strict Press', 'Military Press', 'Barbell Shoulder Press'],
};
```

Match rule: **case-insensitive exact match** of `exercise.name.trim()` against the category's whitelist. No substring; no contains. Exercises not in any whitelist contribute to `getAll1RMs` (Exercise Progress section) but not to `getMainLifts1RMs` / Strength Standards.

**Explicitly excluded (regression intent):**
- Squat: Leg Press, Hack Squat, Bulgarian Split Squat, Goblet Squat, Pistol Squat, Split Squat, Smith Squat, Belt Squat
- Bench Press: Dumbbell Bench, Incline Bench, Decline Bench, Close-Grip Bench, Floor Press, Machine Chest Press
- Deadlift: Romanian Deadlift, Stiff-Leg Deadlift, Single-Leg Deadlift, Snatch-Grip Deadlift, Deficit Deadlift, Block Pull
- Overhead Press: Dumbbell Press, Push Press, Push Jerk, Arnold Press, Landmine Press, Machine Shoulder Press

Rationale: Symmetric Strength standards are calibrated on competition barbell lifts. Variants differ in leverage, range of motion, or assistance — counting them inflates the level and undermines the gauge.

## Data Flow

- App mount: `dbOperations.getAllBodyWeights()` → `setBodyWeightHistory(array)`.
- After `handleSaveBodyWeight` succeeds: re-run `getAllBodyWeights()` and update state.
- `getExerciseHistory` reads `bodyWeightHistory` via `getBodyWeightAt` to attach a `bodyWeight` snapshot per workout.
- `getSmartWeightSuggestion` consumes the augmented history; no other consumer of `getExerciseHistory` reads the new fields.
- `getMainLifts1RMs` consumes `getAll1RMs` and filters via `MAIN_LIFT_WHITELIST`.

## Edge Cases

- **BW store empty** → all entries get `bodyWeight: null`; override never fires; existing behavior preserved.
- **Single BW entry in store** → every workout snapshot is identical; ratios equal; override never fires (correct — no trend exists).
- **BW logged for current but not for older workouts** → `latestBW` exists but `earlierBW` is null → override skipped.
- **BW logged for older but not current** → same skip.
- **BW increased (bulk phase) with same lift numbers** → `latestRel < earlierRel` → override doesn't fire; raw plateau path runs. This is accurate: at higher BW, the same lift is relatively weaker.
- **Custom-named bodyweight exercise** where `isBodyweightExercise` returns false (e.g., "Australian Rows") → falls into weighted path with `weight = 0`, so `bestE1RM = 0` and `relMetric = null` → override skipped. Acceptable; same limitation as today.
- **Weight stayed same, reps decreased, BW dropped** → relative metric may still drop overall → override does not fire (correct).
- **Reps held, BW dropped <1%** → below noise floor; override does not fire (1% threshold matches plateau threshold).
- **Two workouts with identical raw metric, BW changed** → `earlierBestEntry` is the *first* matching entry; consistent and deterministic.

## Testing Plan

Test environment: seed the IndexedDB `history` (and `bodyWeight`) stores via the existing Playwright DOM-click pattern. Reload at `http://localhost:8000`, dismiss the resume-workout modal, then drive UI. Clear back to zero rows after each scenario.

1. **Rep-up below target.** Bench Press, plannedReps = 8. History: 100×6, 100×6, 100×7, all `medium`, BW constant.
   Expect: `type: 'rep-up'`, reason includes "+1 rep last time".
2. **BW-relative override on weighted lift.** Bench Press: 100×6, 100×6, 100×6 (raw plateau). BW history: 82, 81, 80 kg.
   Expect: `type: 'progress'`, reason includes "you're 2.0 kg lighter".
3. **BW-relative override on bodyweight exercise.** Pull-ups: 8, 8, 8 reps. BW: 82, 81, 80 kg.
   Expect: `type: 'progress'`, reason includes "Same reps at 2.0 kg lighter bodyweight".
4. **Deload regression.** Bench Press 100×6×3 flat, all `hard`, BW flat.
   Expect: existing deload (drop ~10%, type `deload`).
5. **Empty BW store regression.** Bench Press 100×6×3 flat, no BW entries.
   Expect: existing plateau → swap (Leg Press or other muscle alt) or hold.
6. **Strength-standard whitelist.** Seed Back Squat 100×5 and Leg Press 200×8. Open Stats.
   Expect: Squat 1RM ≈ 117 kg (from Back Squat only). Leg Press absent from main-lifts.
7. **Whitelist regression — Trap Bar Deadlift counts.** Seed Trap Bar Deadlift 140×3.
   Expect: appears under Deadlift category.
8. **Whitelist regression — RDL excluded.** Seed Romanian Deadlift 80×8.
   Expect: NOT under Deadlift. Still visible in Exercise Progress section.
9. **Whitelist regression — Dumbbell Bench excluded.** Seed Dumbbell Bench Press 30×8.
   Expect: NOT under Bench Press.

## Open Questions

None. All decisions resolved during brainstorm:
- Rep-up scope: both below-target (new branch) and at/above-target (existing branch) covered.
- BW lookup: nearest-date, no max window.
- Bodyweight-exercise metric: `effectiveE1RM` proxy with flipped comparison (reps held + BW dropped).
- Mapping fix: whitelist (not blacklist, not strict-prefix).
- Bug fix and engine work bundled into one spec at user request.
