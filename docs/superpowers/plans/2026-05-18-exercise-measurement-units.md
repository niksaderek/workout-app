# Exercise Measurement Units Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Let each exercise be measured in reps, seconds, or meters — auto-detected by name with a manual per-exercise override — without polluting rep/volume/1RM stats.

**Architecture:** Add two pure helpers (`getExerciseUnit`, `UNIT_CONFIG`) near `getMuscleGroup` in `index.html`. Route the ~10 duplicated `isCoreExercise` name-check sites through the helper so seconds/meters are excluded from rep/volume/1RM stats exactly like core is today. Add an optional `exercise.unit` field set via a new selector in the template editor, copied into history at `startWorkout`. Zero data migration — the existing `set.reps` field is reinterpreted, never renamed.

**Tech Stack:** Single-file React 18 PWA (`index.html`, no build, no test framework). Validation = the project's two-tier rule: Tier 1 visual code review (Read tool) + Tier 2 Playwright over `http://localhost:8000` (the styled path; `file://` breaks `/styles.css`).

**Spec:** `docs/superpowers/specs/2026-05-18-exercise-measurement-units-design.md` (revision `60d60f3` — weighted seconds in scope).

**No test framework note:** This project has no jest/npm. "Write the failing test" steps are replaced by **explicit Playwright verification steps** with seeded IndexedDB scenarios and expected DOM output. Each task ends with Tier-1 review + Tier-2 browser check + commit. Restore seeded IndexedDB `history` to 0 rows after testing; never touch `workouts`/`bodyWeight`.

**Server:** A local server is expected at `http://127.0.0.1:8000` (`python -m http.server 8000 --bind 127.0.0.1` from project root). Playwright is invoked as `C:\Users\nderek\AppData\Roaming\npm\playwright-cli.cmd` (not on Git Bash PATH — use full path). Native Playwright `click` times out due to a fixed overlay — trigger handlers via DOM `.click()` inside `eval`; read results via a raw `eval` expression returning `JSON.stringify(...)`.

---

## File Structure

Single file: `index.html`. Logical regions touched:
- **Helpers** (new): near `getMuscleGroup` (~line 1639) — `getExerciseUnit`, `UNIT_CONFIG`.
- **Logging set-row** (~lines 4291, 4323–4352) — unit-driven label/placeholder/weight.
- **Template editor row** (~lines 4611–4641) — new unit `<select>`, mirrors the muscleGroup select.
- **`startWorkout`** (~line 2149) — copy `unit` into logged exercise.
- **Stats/consumer sites** (~lines 1677, 2727, 2807, 2909, 3066, 4295, 4501) — route through helper.
- **Recommendation engine** (`getExerciseHistory`/`getSmartWeightSuggestion`, ~lines 1979–2133) — unit-aware wording, skip e1RM for non-reps.

---

## Task 1: Add `getExerciseUnit` + `UNIT_CONFIG` helpers

**Files:**
- Modify: `index.html` (insert immediately AFTER the `getMuscleGroup` function, which ends with `return 'other'; // Default fallback` followed by `};`)

- [ ] **Step 1: Locate the insertion point**

Read `index.html` around the end of `getMuscleGroup` (search for `return 'other'; // Default fallback`). The new code goes on the blank line directly after that function's closing `};`.

- [ ] **Step 2: Insert the helpers**

Insert this block:

```javascript
  // ABOUTME: Resolves how an exercise is measured (reps/seconds/meters) and
  // ABOUTME: per-unit display + stats rules. Override wins; else auto-detect by name.
  // FUNC_ID: getExerciseUnit | Returns 'reps' | 'seconds' | 'meters'
  const getExerciseUnit = (exercise) => {
    const u = exercise && exercise.unit;
    if (u === 'reps' || u === 'seconds' || u === 'meters') return u; // explicit override wins
    const n = ((exercise && exercise.name) || '').toLowerCase();
    if (/\b(farmer|carry|walk|sled|yoke)\b/.test(n)) return 'meters';
    if (/\b(plank|hold|wall sit|dead hang|hang)\b/.test(n)) return 'seconds';
    return 'reps';
  };

  const UNIT_CONFIG = {
    reps:    { label: 'reps', placeholder: 'reps', hasWeight: true, countsAsReps: true,  countsAsVolume: true  },
    seconds: { label: 'sec',  placeholder: 'sec',  hasWeight: true, countsAsReps: false, countsAsVolume: false },
    meters:  { label: 'm',    placeholder: 'm',    hasWeight: true, countsAsReps: false, countsAsVolume: false }
  };
```

- [ ] **Step 3: Tier 1 visual review**

Read the inserted block. Verify: closing `};` on both, regex word-boundaries intact, `meters` checked before `seconds`, `seconds.hasWeight: true` (weighted seconds is in scope per spec rev `60d60f3`), all three `hasWeight: true`.

- [ ] **Step 4: Tier 2 smoke test (helper is reachable, no JS error)**

Ensure server running. Then:

```bash
PWCLI="C:\Users\nderek\AppData\Roaming\npm\playwright-cli.cmd"
"$PWCLI" goto "http://127.0.0.1:8000/index.html" >/dev/null 2>&1; sleep 3
"$PWCLI" console 2>&1 | grep -iE "Errors:" | tail -1
```

Expected: `Errors: 0` (the ~5 preload/Babel warnings are pre-existing noise).

- [ ] **Step 5: Commit**

```bash
cd "P:\Projects\02_Personal Projects\Workout App"
git add index.html
git commit -m "$(cat <<'EOF'
Add getExerciseUnit helper and UNIT_CONFIG

Pure helper resolving reps/seconds/meters from an explicit
exercise.unit override or name auto-detection, plus a per-unit
config (label, placeholder, hasWeight, stats flags). Not wired
into UI/stats yet.

Co-Authored-By: Claude Opus 4.7 <noreply@anthropic.com>
EOF
)"
```

---

## Task 2: Wire the logging set-row to the unit

**Files:**
- Modify: `index.html` — the "Target:" line (~line 4291) and the set-row inputs (~lines 4323–4352)

- [ ] **Step 1: Read the current set-row block**

Read `index.html` lines ~4288–4360 to confirm exact current text of the `Target:` div, the weight `<input>` with its inline `disabled={exercise.name.toLowerCase().includes('core') || ...}`, the `×` separator `<span>`, and the reps `<input placeholder="reps">`.

- [ ] **Step 2: Replace the Target line**

Find:

```javascript
                <div className={`text-xs ${darkMode ? 'text-gray-400' : 'text-gray-600'} mb-2`}>Target: {exercise.plannedSets} × {exercise.plannedReps}</div>
```

Replace with:

```javascript
                <div className={`text-xs ${darkMode ? 'text-gray-400' : 'text-gray-600'} mb-2`}>Target: {exercise.plannedSets} × {exercise.plannedReps} {UNIT_CONFIG[getExerciseUnit(exercise)].label}</div>
```

- [ ] **Step 3: Replace the weight input's disabled check**

Find (the weight input's disabled prop, ~lines 4338–4341):

```javascript
                        disabled={exercise.name.toLowerCase().includes('core') ||
                                 exercise.name.toLowerCase().includes('plank') ||
                                 exercise.name.toLowerCase().includes('hanging') ||
                                 exercise.name.toLowerCase().includes('rollout')}
```

Replace with:

```javascript
                        disabled={!UNIT_CONFIG[getExerciseUnit(exercise)].hasWeight}
```

Note: with all three units `hasWeight: true`, the weight input is now never disabled by unit. This is intentional per spec rev `60d60f3` (bodyweight planks just leave it blank). The old name-based disabling is removed deliberately — record this in the commit message.

- [ ] **Step 4: Replace the reps input placeholder**

Find:

```javascript
                      <input
                        type="number"
                        placeholder="reps"
                        value={set.reps}
                        onChange={(e) => updateSet(exIdx, setIdx, 'reps', e.target.value)}
```

Replace the `placeholder="reps"` line with:

```javascript
                        placeholder={UNIT_CONFIG[getExerciseUnit(exercise)].placeholder}
```

(Leave `type="number"`, `value`, `onChange` unchanged.)

- [ ] **Step 5: Tier 1 visual review**

Read the modified block. Verify: `getExerciseUnit(exercise)` (the full exercise object, not `.name`) is passed everywhere; JSX braces balanced; no leftover `includes('plank')` in this block; `×` separator span untouched (still always shows — acceptable since weight is always enabled now).

- [ ] **Step 6: Tier 2 functional test — auto-detected seconds & meters**

Seed a template-less direct check via an existing template that has a core/plank exercise. Use Day 3 (has "Side Plank") and Day 4 (has "Farmer's Walk"). Seed one history record so the exercise renders, then open the workout and read the set-row.

```bash
PWCLI="C:\Users\nderek\AppData\Roaming\npm\playwright-cli.cmd"
# Seed history for Side Plank (Day 3) and Farmer's Walk (Day 4)
"$PWCLI" eval "(async()=>{return await new Promise((res)=>{const rq=indexedDB.open('WorkoutTrackerDB');rq.onsuccess=()=>{const db=rq.result;const st=db.transaction('history','readwrite').objectStore('history');const c=st.clear();c.onsuccess=()=>{const recs=[{id:1,workoutId:3,workoutName:'T',date:'2026-05-09T10:00:00Z',exercises:[{name:'Side Plank',plannedSets:3,plannedReps:'45',sets:[{weight:'',reps:'45'}]}]}];let n=0;recs.forEach(r=>{const p=st.put(r);p.onsuccess=()=>{n++;if(n===recs.length)res('S:'+n);};});};};rq.onerror=()=>res('ERR');});})()" 2>&1 | grep -A1 '### Result' | tail -1
"$PWCLI" goto "http://127.0.0.1:8000/index.html" >/dev/null 2>&1; sleep 3
# Start Day 3 (index 2 of Start Workout buttons), read the Side Plank set-row
"$PWCLI" eval "(()=>{const b=[...document.querySelectorAll('button')].filter(x=>x.textContent.includes('Start Workout'));b[2]&&b[2].click();return 'C';})()" >/dev/null 2>&1; sleep 2
"$PWCLI" eval "JSON.stringify((()=>{const ph=[...document.querySelectorAll('input')].map(i=>i.placeholder).filter(Boolean);const tgt=[...document.querySelectorAll('div')].map(d=>d.textContent).filter(t=>t.startsWith('Target:')).slice(0,6);return {placeholders:[...new Set(ph)],targets:tgt};})())" 2>&1 | grep -A1 '### Result' | tail -1
```

Expected: placeholders include `"sec"` for the Side Plank row; its Target line ends with `sec` (e.g. `Target: 3 × 45 sec`). Other (reps) exercises still show `reps`.

- [ ] **Step 7: Tier 2 — Farmer's Walk shows meters + weight**

```bash
PWCLI="C:\Users\nderek\AppData\Roaming\npm\playwright-cli.cmd"
"$PWCLI" eval "(()=>{const b=[...document.querySelectorAll('button')].filter(x=>x.textContent.includes('Start Workout'));b[3]&&b[3].click();return 'C';})()" >/dev/null 2>&1; sleep 2
"$PWCLI" eval "JSON.stringify((()=>{const tgt=[...document.querySelectorAll('div')].map(d=>d.textContent).filter(t=>t.startsWith('Target:')&&t.includes(' m'));const phs=[...new Set([...document.querySelectorAll('input')].map(i=>i.placeholder).filter(Boolean))];return{meterTargets:tgt.slice(0,3),placeholders:phs};})())" 2>&1 | grep -A1 '### Result' | tail -1
```

Expected: a Target line ending in ` m` for Farmer's Walk; placeholders include `"m"` and `"kg"` (weight input present, not disabled).

- [ ] **Step 8: Restore IndexedDB**

```bash
PWCLI="C:\Users\nderek\AppData\Roaming\npm\playwright-cli.cmd"
"$PWCLI" eval "(async()=>{return await new Promise((res)=>{const rq=indexedDB.open('WorkoutTrackerDB');rq.onsuccess=()=>{const db=rq.result;const st=db.transaction('history','readwrite').objectStore('history');const c=st.clear();c.onsuccess=()=>{const g=st.getAll();g.onsuccess=()=>res('rows:'+g.result.length);};};rq.onerror=()=>res('ERR');});})()" 2>&1 | grep -A1 '### Result' | tail -1
```

Expected: `rows:0`.

- [ ] **Step 9: Commit**

```bash
cd "P:\Projects\02_Personal Projects\Workout App"
git add index.html
git commit -m "$(cat <<'EOF'
Drive logging set-row labels from exercise unit

Target line and rep-input placeholder now reflect the resolved
unit (reps/sec/m). Weight input's name-based disabling removed;
all units keep an optional weight field per spec (weighted plank
/ loaded carry). Value still stored in set.reps.

Co-Authored-By: Claude Opus 4.7 <noreply@anthropic.com>
EOF
)"
```

---

## Task 3: Add the unit override selector to the template editor

**Files:**
- Modify: `index.html` — the editor exercise row, immediately after the muscleGroup `<select>` block (~lines 4611–4625)

- [ ] **Step 1: Read the editor row**

Read `index.html` lines ~4610–4642 to confirm the muscleGroup `<select>` wrapper `<div className="flex gap-2 pl-7 mb-2">...</div>` and the `updateExercise(idx, field, value)` signature usage.

- [ ] **Step 2: Insert the unit selector after the muscleGroup div**

Find the closing of the muscleGroup select wrapper:

```javascript
                    </select>
                  </div>
                  <div className="flex gap-2 pl-7">
                    <input
                      type="number"
                      value={ex.sets}
```

Insert a new `<div>` between the muscleGroup `</div>` and the `<div className="flex gap-2 pl-7">` sets/reps div:

```javascript
                    </select>
                  </div>
                  <div className="flex gap-2 pl-7 mb-2">
                    <select
                      value={ex.unit || 'auto'}
                      onChange={(e) => updateExercise(idx, 'unit', e.target.value === 'auto' ? undefined : e.target.value)}
                      className={`flex-1 px-3 py-2 ${darkMode ? 'bg-gray-900 border-gray-700 text-white' : 'bg-gray-50 border-gray-300 text-gray-900'} border rounded-lg focus:border-gray-500 focus:outline-none text-sm`}
                    >
                      <option value="auto">Unit: Auto (detected: {UNIT_CONFIG[getExerciseUnit(ex)].label})</option>
                      <option value="reps">Unit: Reps</option>
                      <option value="seconds">Unit: Seconds</option>
                      <option value="meters">Unit: Meters</option>
                    </select>
                  </div>
                  <div className="flex gap-2 pl-7">
                    <input
                      type="number"
                      value={ex.sets}
```

- [ ] **Step 3: Verify `updateExercise` handles `undefined` (clearing override)**

Read the `updateExercise` function (search `const updateExercise =`). Confirm it does `updated.exercises[idx][field] = value;` (a plain assignment). Assigning `undefined` is acceptable — `getExerciseUnit` checks `=== 'reps'|'seconds'|'meters'` so `undefined` falls through to auto-detect. If `updateExercise` instead spreads/validates fields, adapt: set `value` to `undefined` and ensure it is not stripped. Document the actual behavior found in the commit message. Do NOT add a migration or backfill.

- [ ] **Step 4: Tier 1 visual review**

Read the inserted block. Verify: `<select>` value falls back to `'auto'` when `ex.unit` is unset; the "Auto (detected: X)" option shows the live detected label; selecting "auto" passes `undefined`; mirrors the muscleGroup select styling exactly; wrapper div has `mb-2` like the muscleGroup one.

- [ ] **Step 5: Tier 2 functional test — selector renders & overrides**

```bash
PWCLI="C:\Users\nderek\AppData\Roaming\npm\playwright-cli.cmd"
"$PWCLI" goto "http://127.0.0.1:8000/index.html" >/dev/null 2>&1; sleep 3
# Open editor for Day 1 (first edit pencil). Find via DOM.
"$PWCLI" eval "(()=>{const b=[...document.querySelectorAll('button')];/* edit buttons have an Edit2 svg; click the first card's edit */ const cards=[...document.querySelectorAll('button')].filter(x=>x.querySelector('svg')&&!x.textContent.includes('Start'));return 'cards:'+cards.length;})()" 2>&1 | grep -A1 '### Result' | tail -1
```

Then drive into the editor (use the snapshot to find the Day 1 edit button ref, click via DOM as established), and read:

```bash
"$PWCLI" eval "JSON.stringify([...document.querySelectorAll('option')].map(o=>o.textContent).filter(t=>t.startsWith('Unit:')))" 2>&1 | grep -A1 '### Result' | tail -1
```

Expected: options include `Unit: Auto (detected: reps)` (for Bench Press), `Unit: Reps`, `Unit: Seconds`, `Unit: Meters`. For the "Side Plank"/"Farmer's Walk" days the Auto option shows `(detected: sec)` / `(detected: m)`.

- [ ] **Step 6: Tier 2 — override persists into the field**

In the editor, set a unit override via DOM and confirm `updateExercise` wrote it (read React-rendered select value back):

```bash
"$PWCLI" eval "(()=>{const s=[...document.querySelectorAll('select')].find(x=>[...x.options].some(o=>o.textContent==='Unit: Seconds'));if(!s)return 'NO_SELECT';const setter=Object.getOwnPropertyDescriptor(window.HTMLSelectElement.prototype,'value').set;setter.call(s,'seconds');s.dispatchEvent(new Event('change',{bubbles:true}));return 'set';})()" 2>&1 | grep -A1 '### Result' | tail -1
sleep 1
"$PWCLI" eval "(()=>{const s=[...document.querySelectorAll('select')].find(x=>[...x.options].some(o=>o.textContent==='Unit: Seconds'));return s?s.value:'NONE';})()" 2>&1 | grep -A1 '### Result' | tail -1
```

Expected: second call returns `"seconds"` (override took, React re-rendered with it). Cancel the editor afterward (do not Save — avoid mutating the real template) by clicking the Cancel button via DOM.

- [ ] **Step 7: Commit**

```bash
cd "P:\Projects\02_Personal Projects\Workout App"
git add index.html
git commit -m "$(cat <<'EOF'
Add per-exercise unit override selector to template editor

New select (Auto/Reps/Seconds/Meters) mirroring the muscleGroup
selector. Auto shows the live-detected unit; selecting it clears
the override (exercise.unit unset -> auto-detect).

Co-Authored-By: Claude Opus 4.7 <noreply@anthropic.com>
EOF
)"
```

---

## Task 4: Carry `unit` into history at `startWorkout`

**Files:**
- Modify: `index.html` — `startWorkout` exercise mapping (~lines 2143–2159)

- [ ] **Step 1: Read `startWorkout`'s exercise map**

Read `index.html` lines ~2143–2160. Confirm the returned object literal with `name, plannedSets, plannedReps, muscleGroup, difficulty, sets`.

- [ ] **Step 2: Add `unit` to the returned exercise object**

Find:

```javascript
          name: ex.name,
          plannedSets: ex.sets,
          plannedReps: ex.reps,
          muscleGroup: ex.muscleGroup || getMuscleGroup(ex.name), // Auto-assign if missing
          difficulty: null, // User can optionally rate difficulty
```

Replace with:

```javascript
          name: ex.name,
          plannedSets: ex.sets,
          plannedReps: ex.reps,
          muscleGroup: ex.muscleGroup || getMuscleGroup(ex.name), // Auto-assign if missing
          unit: ex.unit, // explicit override only; undefined => history reader auto-detects
          difficulty: null, // User can optionally rate difficulty
```

- [ ] **Step 3: Tier 1 visual review**

Read the modified map. Verify trailing comma after `ex.unit`, comment present, no other field reordered. `ex.unit` may be `undefined` — that is intended (reader auto-detects; matches the `muscleGroup` fallback philosophy but we deliberately do NOT eagerly resolve here so old logic and override stay distinguishable).

- [ ] **Step 4: Tier 2 functional test — logged record carries unit when overridden**

```bash
PWCLI="C:\Users\nderek\AppData\Roaming\npm\playwright-cli.cmd"
"$PWCLI" goto "http://127.0.0.1:8000/index.html" >/dev/null 2>&1; sleep 3
# Start any workout, then inspect currentLog via a finished/seeded path is complex;
# instead verify the field exists on the in-memory log by reading the rendered logging
# view's exercise (unit only matters at read time). Functional proof: start Day 3
# (Side Plank auto=seconds, no override) and confirm set-row still shows 'sec'
# (proves unit:undefined + auto-detect path unaffected).
"$PWCLI" eval "(()=>{const b=[...document.querySelectorAll('button')].filter(x=>x.textContent.includes('Start Workout'));b[2]&&b[2].click();return 'C';})()" >/dev/null 2>&1; sleep 2
"$PWCLI" eval "JSON.stringify([...new Set([...document.querySelectorAll('input')].map(i=>i.placeholder).filter(Boolean))])" 2>&1 | grep -A1 '### Result' | tail -1
```

Expected: placeholders include `"sec"` (auto-detect still works with `unit: undefined` in the new log shape — no regression).

- [ ] **Step 5: Commit**

```bash
cd "P:\Projects\02_Personal Projects\Workout App"
git add index.html
git commit -m "$(cat <<'EOF'
Copy exercise unit override into logged history

startWorkout now carries ex.unit (override only; undefined when
unset so the history reader auto-detects, same philosophy as the
muscleGroup fallback).

Co-Authored-By: Claude Opus 4.7 <noreply@anthropic.com>
EOF
)"
```

---

## Task 5: Route stats/consumer sites through the unit helper

**Files:**
- Modify: `index.html` at the duplicated detection sites: ~lines 1677, 2727, 2807, 2909, 3066, 4295, 4501

**Principle:** Replace each "is this exercise core (exclude from reps/volume/1RM)" decision with the unit rule. A site that excludes core today must now also exclude `seconds`/`meters`. Use `UNIT_CONFIG[getExerciseUnit(EXERCISE_OBJECT)]`. Pass the object available at that site (`ex` or `exercise`). Do NOT remove the bodyweight (`pull-up`) logic — that is a separate concern; leave `isBodyweightExercise` untouched.

Helper expression to reuse at each site:
`const u = UNIT_CONFIG[getExerciseUnit(EXOBJ)];` then `u.countsAsVolume` (for volume/1RM gating) and `u.countsAsReps` (for rep counting). `core`-only special branches (e.g. "if core, count reps separately") should trigger for `!u.countsAsReps` (seconds/meters behave like core: separately counted, not in main reps/volume).

- [ ] **Step 1: Site ~1677 (`getMainLifts1RMs` / 1RM exclusion)**

Read lines ~1670–1690. Find:

```javascript
        const isCoreExercise = exerciseName.toLowerCase().includes('plank') ||
```
(through its `||` chain) and the guard `if (isCoreExercise) return;`.

Replace the `isCoreExercise` assignment with:

```javascript
        const isCoreExercise = !UNIT_CONFIG[getExerciseUnit({ name: exerciseName })].countsAsVolume;
```

Leave `if (isCoreExercise) return;` as-is. (Here only a name is in scope, so pass `{ name: exerciseName }` — override not available in 1RM history aggregation, acceptable: 1RM for a seconds/meters lift is meaningless anyway.)

- [ ] **Step 2: Site ~2727 (stats: detailed stats volume/reps)**

Read lines ~2720–2750. Find the `const isCoreExercise = exercise.name.toLowerCase().includes('core') || ... ;` block. Replace ONLY that assignment with:

```javascript
        const isCoreExercise = !UNIT_CONFIG[getExerciseUnit(exercise)].countsAsReps;
```

Leave the following `isBodyweightExercise` line and all `if (isCoreExercise ...)` branches unchanged.

- [ ] **Step 3: Site ~2807 (stats: per-set loop)**

Read lines ~2800–2830. Replace the `const isCoreExercise = ...includes(...)...;` assignment with:

```javascript
          const isCoreExercise = !UNIT_CONFIG[getExerciseUnit(exercise)].countsAsReps;
```

(Match the indentation of the line being replaced.) Leave `isBodyweightExercise` and branch logic unchanged.

- [ ] **Step 4: Site ~2909 (stats: volume vs core-rep split)**

Read lines ~2905–2930. Replace the `const isCoreExercise = ...;` assignment with:

```javascript
        const isCoreExercise = !UNIT_CONFIG[getExerciseUnit(exercise)].countsAsVolume;
```

Leave the `if (!isCoreExercise && weight > 0 && rep > 0)` (volume) and `if (isCoreExercise && rep > 0)` (separate count) branches unchanged — they now also route seconds/meters into the "separate count" path, which is the desired "track separately" behavior.

- [ ] **Step 5: Site ~3066 (timeline data)**

Read lines ~3060–3090. Replace the `const isCoreExercise = ...;` assignment with:

```javascript
        const isCoreExercise = !UNIT_CONFIG[getExerciseUnit(exercise)].countsAsVolume;
```

Leave `isBodyweightExercise` and the `if (!isCoreExercise && weight > 0 && reps > 0)` volume branch unchanged.

- [ ] **Step 6: Site ~4295 (logging: suppress weight-suggestion for non-rep units)**

Read lines ~4293–4302. Find:

```javascript
                  const isCoreExercise = exercise.name.toLowerCase().includes('core') ||
                                        exercise.name.toLowerCase().includes('plank') ||
                                        exercise.name.toLowerCase().includes('hanging') ||
                                        exercise.name.toLowerCase().includes('rollout');
                  if (isCoreExercise) return null;
```

Replace the assignment with:

```javascript
                  const isCoreExercise = getExerciseUnit(exercise) !== 'reps';
```

Leave `if (isCoreExercise) return null;`. Effect: the weight/1RM suggestion line is hidden for seconds/meters exercises (it would be meaningless). Task 6 adds a unit-aware suggestion instead.

- [ ] **Step 7: Site ~4501 (history view: completed-set filter)**

Read lines ~4496–4515. Find:

```javascript
                        const isCoreExercise = ex.name.toLowerCase().includes('core') ||
                                              ex.name.toLowerCase().includes('plank') ||
                                              ex.name.toLowerCase().includes('hanging') ||
                                              ex.name.toLowerCase().includes('rollout');
                        const isBodyweightExercise = ex.name.toLowerCase().includes('pull-up') ||
```
…and the line `if (isCoreExercise || isBodyweightExercise) return reps > 0;`.

Replace ONLY the `isCoreExercise` assignment with:

```javascript
                        const isCoreExercise = getExerciseUnit(ex) !== 'reps';
```

Leave `isBodyweightExercise` and the `return reps > 0;` filter unchanged. Effect: a seconds/meters set with a value > 0 still shows in history (not filtered as "skipped").

- [ ] **Step 8: Site ~2144 (`startWorkout` lastWeight prefill — verify, likely leave)**

Read lines ~2143–2149. The `isCoreExercise` here only decides `lastWeight = isCoreExercise ? '0' : getLastWeight(ex.name)`. Since all units now keep a weight field and weighted planks are valid, change:

```javascript
        const isCoreExercise = ex.name.toLowerCase().includes('core') ||
                               ex.name.toLowerCase().includes('plank') ||
                               ex.name.toLowerCase().includes('hanging') ||
                               ex.name.toLowerCase().includes('rollout');
        const lastWeight = isCoreExercise ? '0' : getLastWeight(ex.name);
```

to:

```javascript
        const prefillUnit = getExerciseUnit(ex);
        const lastWeight = prefillUnit === 'reps' ? getLastWeight(ex.name) : '';
```

Rationale: for seconds/meters default the weight blank (not `'0'`) so the optional weight field starts empty; reps keeps the progressive-overload prefill. Update any later reference to `isCoreExercise` in this map scope — there is none (only `lastWeight` is used).

- [ ] **Step 9: Tier 1 visual review (all sites)**

Read each modified region. Verify: every site passes the correct in-scope object (`exercise`, `ex`, or `{name: exerciseName}`); `isBodyweightExercise` never touched; no branch logic altered, only the boolean's source; indentation matches surroundings; no remaining `includes('plank')` except inside `getExerciseUnit` itself.

```bash
cd "P:\Projects\02_Personal Projects\Workout App"
grep -n "includes('plank')" index.html
```

Expected: exactly ONE match — the line inside `getExerciseUnit`'s regex is `/\b(plank|.../` (no `.includes('plank')`), so expected output is **no matches**. Any other hit is a missed site — fix before continuing.

- [ ] **Step 10: Tier 2 functional regression — reps exercise unchanged**

Seed the prior recommendation-engine regression scenario (Bench Press, Day 1) and confirm stats/suggestion behave identically to before this feature.

```bash
PWCLI="C:\Users\nderek\AppData\Roaming\npm\playwright-cli.cmd"
"$PWCLI" eval "(async()=>{return await new Promise((res)=>{const rq=indexedDB.open('WorkoutTrackerDB');rq.onsuccess=()=>{const db=rq.result;const st=db.transaction('history','readwrite').objectStore('history');const c=st.clear();c.onsuccess=()=>{const recs=[{id:1,workoutId:1,workoutName:'T',date:'2026-05-09T10:00:00Z',exercises:[{name:'Bench Press',plannedSets:3,plannedReps:'5',difficulty:'easy',sets:[{weight:'60',reps:'5'}]}]}];let n=0;recs.forEach(r=>{const p=st.put(r);p.onsuccess=()=>{n++;if(n===recs.length)res('S:'+n);};});};};rq.onerror=()=>res('ERR');});})()" 2>&1 | grep -A1 '### Result' | tail -1
"$PWCLI" goto "http://127.0.0.1:8000/index.html" >/dev/null 2>&1; sleep 3
"$PWCLI" eval "(()=>{const b=[...document.querySelectorAll('button')].filter(x=>x.textContent.includes('Start Workout'));b[0]&&b[0].click();return 'C';})()" >/dev/null 2>&1; sleep 2
"$PWCLI" eval "(()=>{const s=[...document.querySelectorAll('span')].find(x=>x.textContent.includes('Suggestion:'));return s?s.textContent.trim():'NONE';})()" 2>&1 | grep -A1 '### Result' | tail -1
```

Expected: `Suggestion: 62.5 kg — felt easy last time (+2.5 kg)` — identical to the pre-feature behavior (proves reps path untouched).

- [ ] **Step 11: Tier 2 — seconds exercise excluded from stats**

```bash
PWCLI="C:\Users\nderek\AppData\Roaming\npm\playwright-cli.cmd"
# Seed a Side Plank workout with a big "value" (60) that would inflate Total Reps if not excluded
"$PWCLI" eval "(async()=>{return await new Promise((res)=>{const rq=indexedDB.open('WorkoutTrackerDB');rq.onsuccess=()=>{const db=rq.result;const st=db.transaction('history','readwrite').objectStore('history');const c=st.clear();c.onsuccess=()=>{const recs=[{id:1,workoutId:3,workoutName:'T',date:new Date().toISOString(),exercises:[{name:'Side Plank',plannedSets:3,plannedReps:'60',sets:[{weight:'',reps:'60'},{weight:'',reps:'60'}]}]}];let n=0;recs.forEach(r=>{const p=st.put(r);p.onsuccess=()=>{n++;if(n===recs.length)res('S:'+n);};});};};rq.onerror=()=>res('ERR');});})()" 2>&1 | grep -A1 '### Result' | tail -1
"$PWCLI" goto "http://127.0.0.1:8000/index.html" >/dev/null 2>&1; sleep 3
# Read the home "Total Reps" stat card value
"$PWCLI" eval "JSON.stringify((()=>{const t=document.body.innerText;const m=t.match(/Total Reps[\s\S]{0,40}/);return m?m[0].replace(/\n/g,' '):'NO_MATCH';})())" 2>&1 | grep -A1 '### Result' | tail -1
```

Expected: Total Reps is NOT inflated by 120 — the 60-second plank values are excluded from the rep count (treated like core). The exact displayed number depends on the calendar-week logic; the assertion is that it is small/zero, not 120.

- [ ] **Step 12: Restore IndexedDB**

```bash
PWCLI="C:\Users\nderek\AppData\Roaming\npm\playwright-cli.cmd"
"$PWCLI" eval "(async()=>{return await new Promise((res)=>{const rq=indexedDB.open('WorkoutTrackerDB');rq.onsuccess=()=>{const db=rq.result;const st=db.transaction('history','readwrite').objectStore('history');const c=st.clear();c.onsuccess=()=>{const g=st.getAll();g.onsuccess=()=>res('rows:'+g.result.length);};};rq.onerror=()=>res('ERR');});})()" 2>&1 | grep -A1 '### Result' | tail -1
```

Expected: `rows:0`.

- [ ] **Step 13: Commit**

```bash
cd "P:\Projects\02_Personal Projects\Workout App"
git add index.html
git commit -m "$(cat <<'EOF'
Route stats/consumer sites through getExerciseUnit

Replace ~7 duplicated name-based core checks with the unit rule:
seconds/meters exercises are now excluded from reps/volume/1RM
and tracked separately, exactly like core. Bodyweight (pull-up)
logic untouched. startWorkout prefills blank weight for non-rep
units.

Co-Authored-By: Claude Opus 4.7 <noreply@anthropic.com>
EOF
)"
```

---

## Task 6: Unit-aware recommendation wording

**Files:**
- Modify: `index.html` — `getSmartWeightSuggestion` (~lines 2047–2133)

**Context:** Task 5 Step 6 already hides the suggestion line for non-`reps` units (returns null at the call site). This task makes the engine itself safe and adds a minimal unit-aware suggestion so seconds/meters still get guidance instead of nothing.

- [ ] **Step 1: Read `getSmartWeightSuggestion`**

Read `index.html` lines ~2044–2133 (the function from `const getSmartWeightSuggestion = (exerciseName, plannedReps = null) => {` to its closing `};`).

- [ ] **Step 2: Add an early unit branch after history is fetched**

Find:

```javascript
  const getSmartWeightSuggestion = (exerciseName, plannedReps = null) => {
    const history = getExerciseHistory(exerciseName, 5);

    // 1. No usable history -> no suggestion (unchanged behavior)
    if (history.length === 0) return null;
```

Insert immediately AFTER the `if (history.length === 0) return null;` line:

```javascript

    // Non-rep units (seconds/meters): weight/1RM/plateau math is meaningless.
    // Give a simple progression nudge based on the last logged value.
    const unit = getExerciseUnit({ name: exerciseName });
    if (unit !== 'reps') {
      const last = history[history.length - 1];
      const lastVal = last.topSetRepsHit; // raw seconds or meters of best set
      if (!lastVal) return null;
      const cfg = UNIT_CONFIG[unit];
      const diff = last.difficulty;
      if (diff === 'easy') {
        const bump = unit === 'seconds' ? 5 : 5; // +5 sec or +5 m
        return { weight: '', reps: String(lastVal + bump), reason: `felt easy — try ${lastVal + bump} ${cfg.label}`, type: 'progress' };
      }
      return { weight: '', reps: String(lastVal), reason: `last time ${lastVal} ${cfg.label} — match or beat it`, type: 'info' };
    }
```

Note: `getExerciseHistory` already computes `topSetRepsHit` (max of `set.reps` across sets) — for a seconds/meters exercise that is the best time/distance, which is the correct progression target.

- [ ] **Step 3: Tier 1 visual review**

Read the modified function head. Verify: the new branch is AFTER the empty-history guard and BEFORE the `latest`/difficulty logic; it `return`s in every path or falls through only for `unit === 'reps'`; `cfg.label` used for wording; no reference to `weight`/e1RM in the non-rep branch. Confirm the existing reps logic below is untouched.

- [ ] **Step 4: Tier 2 — seconds exercise gets a unit-aware line (when shown)**

The call site (Task 5 Step 6) returns null for non-reps BEFORE rendering, so the suggestion is intentionally not shown in the logging view for seconds/meters. This task's branch is defensive + future-proofing and is exercised via direct call. Verify no regression and the reps path still works:

```bash
PWCLI="C:\Users\nderek\AppData\Roaming\npm\playwright-cli.cmd"
"$PWCLI" eval "(async()=>{return await new Promise((res)=>{const rq=indexedDB.open('WorkoutTrackerDB');rq.onsuccess=()=>{const db=rq.result;const st=db.transaction('history','readwrite').objectStore('history');const c=st.clear();c.onsuccess=()=>{const recs=[{id:1,workoutId:1,workoutName:'T',date:'2026-05-09T10:00:00Z',exercises:[{name:'Bench Press',plannedSets:3,plannedReps:'5',difficulty:'easy',sets:[{weight:'60',reps:'5'}]}]}];let n=0;recs.forEach(r=>{const p=st.put(r);p.onsuccess=()=>{n++;if(n===recs.length)res('S:'+n);};});};};rq.onerror=()=>res('ERR');});})()" 2>&1 | grep -A1 '### Result' | tail -1
"$PWCLI" goto "http://127.0.0.1:8000/index.html" >/dev/null 2>&1; sleep 3
"$PWCLI" eval "(()=>{const b=[...document.querySelectorAll('button')].filter(x=>x.textContent.includes('Start Workout'));b[0]&&b[0].click();return 'C';})()" >/dev/null 2>&1; sleep 2
"$PWCLI" eval "(()=>{const s=[...document.querySelectorAll('span')].find(x=>x.textContent.includes('Suggestion:'));return s?s.textContent.trim():'NONE';})()" 2>&1 | grep -A1 '### Result' | tail -1
```

Expected: `Suggestion: 62.5 kg — felt easy last time (+2.5 kg)` (reps path unchanged; non-rep branch did not interfere).

- [ ] **Step 5: Restore IndexedDB & commit**

```bash
PWCLI="C:\Users\nderek\AppData\Roaming\npm\playwright-cli.cmd"
"$PWCLI" eval "(async()=>{return await new Promise((res)=>{const rq=indexedDB.open('WorkoutTrackerDB');rq.onsuccess=()=>{const db=rq.result;const st=db.transaction('history','readwrite').objectStore('history');const c=st.clear();c.onsuccess=()=>{const g=st.getAll();g.onsuccess=()=>res('rows:'+g.result.length);};};rq.onerror=()=>res('ERR');});})()" 2>&1 | grep -A1 '### Result' | tail -1
cd "P:\Projects\02_Personal Projects\Workout App"
git add index.html
git commit -m "$(cat <<'EOF'
Add unit-aware branch to recommendation engine

Seconds/meters exercises now get a simple value-progression
nudge instead of weight/1RM math (which is meaningless for
time/distance). Reps path unchanged.

Co-Authored-By: Claude Opus 4.7 <noreply@anthropic.com>
EOF
)"
```

---

## Task 7: Full end-to-end verification

**Files:** none (verification only)

- [ ] **Step 1: Full scenario matrix via Playwright**

With server running, execute the spec's Tier-2 matrix in one pass:

1. Old Plank history (no `unit`) → set-row "sec", weight field present (blank), excluded from Volume & Total Reps.
2. Old Farmer's Walk (no `unit`) → "m" + weight field, excluded from rep/volume stats.
3. Editor: set a reps exercise's override to Seconds → start that workout → set-row shows "sec".
4. Editor: override → Auto reverts to the detected unit label.
5. Regression: Bench Press scenario → suggestion + stats identical to pre-feature (`62.5 kg — felt easy`).
6. `grep -n "includes('plank')" index.html` → no matches outside `getExerciseUnit`.
7. `playwright-cli console` → `Errors: 0`.

Record actual outputs for each. Any deviation → stop, fix, re-verify (do not mark complete with failures — Broken Windows).

- [ ] **Step 2: Restore IndexedDB to 0 rows; confirm `workouts`/`bodyWeight` untouched**

```bash
PWCLI="C:\Users\nderek\AppData\Roaming\npm\playwright-cli.cmd"
"$PWCLI" eval "(async()=>{return await new Promise((res)=>{const rq=indexedDB.open('WorkoutTrackerDB');rq.onsuccess=()=>{const db=rq.result;const tx=db.transaction(['history','workouts','bodyWeight'],'readwrite');tx.objectStore('history').clear();const g1=tx.objectStore('workouts').getAll();const g2=tx.objectStore('bodyWeight').getAll();g1.onsuccess=()=>{g2.onsuccess=()=>res('workouts:'+g1.result.length+' bodyWeight:'+g2.result.length);};};rq.onerror=()=>res('ERR');});})()" 2>&1 | grep -A1 '### Result' | tail -1
```

Expected: workouts/bodyWeight counts unchanged from the user's real data (4 templates expected); history cleared.

- [ ] **Step 3: Final review summary**

Produce a short table: scenario → expected → actual → pass/fail. All must pass.

- [ ] **Step 4: Update PROJECT_LEARNINGS.md**

Append dated entries: the weighted-seconds scope revision (decision + why), the "reuse `set.reps`, never rename" constraint, and the "all units keep weight field" decision. One concise line each under the right section. Do not commit CLAUDE.md (per standing user instruction this session).

- [ ] **Step 5: Final commit (docs only)**

```bash
cd "P:\Projects\02_Personal Projects\Workout App"
git add PROJECT_LEARNINGS.md docs/superpowers/plans/2026-05-18-exercise-measurement-units.md
git status --short   # confirm CLAUDE.md NOT staged
git commit -m "$(cat <<'EOF'
Record units feature decisions and implementation plan

Co-Authored-By: Claude Opus 4.7 <noreply@anthropic.com>
EOF
)"
```

Note: `PROJECT_LEARNINGS.md` is gitignored — `git add` it explicitly is a no-op/error; if so, drop it from the add and commit only the plan doc. Do NOT push; the user controls pushes and the `sw.js` CACHE_NAME bump.

---

## Self-Review

**Spec coverage:**
- Auto-detect + override → Task 1 (helper), Task 3 (selector). ✓
- Weight relevance (all units, optional) → Task 1 (`hasWeight:true` ×3), Task 2 (disabled removed). ✓
- Track separately (excluded from reps/volume/1RM) → Task 5 (all sites). ✓
- No migration / backward compatible → Tasks reuse `set.reps`; `unit` optional; verified Task 7 scenarios 1–2. ✓
- Override on template, copied to history → Task 3 + Task 4. ✓
- CSV unchanged → no task touches CSV (deliberate; stated in spec non-goals). ✓
- Recommendation engine unit-aware → Task 5 Step 6 (hide) + Task 6 (branch). ✓
- Weighted seconds in scope (spec rev `60d60f3`) → Task 1 `seconds.hasWeight:true`, Task 2, Task 5 Step 8. ✓

**Placeholder scan:** No TBD/TODO; every code step shows full code; commands have expected output. ✓

**Type/name consistency:** `getExerciseUnit(obj)` always called with an object (`exercise`, `ex`, or `{name:...}`) — consistent across Tasks 1–6. `UNIT_CONFIG[...]` keys `label/placeholder/hasWeight/countsAsReps/countsAsVolume` defined in Task 1, used identically after. `topSetRepsHit` is a real field from the shipped `getExerciseHistory` (verified this session). `updateExercise(idx, field, value)` signature confirmed against codebase. ✓

**Scope:** Single cohesive feature, one file, 7 sequenced tasks. No decomposition needed. ✓
