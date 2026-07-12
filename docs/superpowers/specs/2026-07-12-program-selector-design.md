# Program Selector — Design Spec
_Date: 2026-07-12_

## Summary

Add a program layer above workout day templates. A dropdown selector on the main (home) page switches the active program; only the active program's workout days are shown and used by "Add Day". Programs support full CRUD. History, stats, and recommendations stay global and untouched.

Storage is nested (each program record contains its workout days) but the React state layer derives a flat `workouts` array for the active program, so all existing downstream code paths (logging, editing, day CRUD, stats) keep working unchanged.

---

## 1. Data Model

### IndexedDB (v2 → v3)

New object store `programs`, keyPath `id`:

```javascript
{
  id: number,          // Date.now()-based, same convention as workout ids
  name: string,        // e.g. "My Program", "PPL"
  workouts: [ ... ]    // array of existing workout day objects, unchanged shape
}
```

**Migration in `onupgradeneeded` (v3):**
1. Create `programs` store.
2. Read all records from the existing `workouts` store.
3. If any exist, write one program `{ id: Date.now(), name: 'My Program', workouts: [...existing] }` to `programs`.
4. Old `workouts` store is left in place for one version (safety net, no code reads it after migration). Do not delete it in this release.

Migration must also handle the localStorage fallback path (`workouts_backup`): on first load after upgrade, if `programs` is empty but `workouts_backup` exists, wrap it the same way.

### Active program selection

`activeProgramId` persisted in `localStorage` (key `active_program_id`). It is a UI preference, not data. On load: if stored id doesn't match any program, fall back to first program. If no programs exist at all, create empty default `"My Program"`.

### localStorage mirror

Existing `workouts_backup` localStorage mirror is replaced by `programs_backup` (full programs array), written on every program save — same auto-backup role as today.

---

## 2. State Layer

```javascript
const [programs, setPrograms] = useState([]);
const [activeProgramId, setActiveProgramId] = useState(null);

const activeProgram = programs.find(p => p.id === activeProgramId) || null;
const workouts = activeProgram ? activeProgram.workouts : [];
```

- `workouts` is DERIVED — the `useState` for `workouts` is removed.
- A wrapper `updateActiveWorkouts(newWorkouts)` replaces the current `saveWorkouts(updatedWorkouts)` call sites: it maps `programs`, replacing the active program's `workouts` array, then persists via `dbOperations.savePrograms(...)` + updates `programs_backup`.
- All downstream consumers (`startWorkout`, day edit/delete/duplicate, `showAddDay`, home list render, `getWeekStats`, logging flow) continue to read the flat `workouts` variable — no changes.
- History records keep `workoutId` only (no `programId`). Recommendations key on exercise name — unaffected.

### dbOperations changes

- `savePrograms(programs)` — clear + put all (mirrors current `saveWorkouts` pattern).
- `getPrograms()` — getAll from `programs`.
- `saveWorkouts`/`getWorkouts` retired from the runtime path (kept only if migration helper needs a one-time read).

---

## 3. UI — Program Dropdown

**Placement:** next to the "Your Program" heading on the home view (heading row becomes flex with the dropdown at right).

**Closed state:** button showing active program name + chevron-down, styled like existing secondary buttons (gray-700 dark / gray-200 light).

**Open state (dropdown panel):**
- One row per program; active row gets a checkmark. Tap → set active, close.
- Divider.
- `+ New program` → inline prompt for name (reuse existing add-day naming modal pattern), creates empty program, makes it active.
- `Manage…` → management mode listing programs with rename (pencil → inline text input) and delete (trash) actions.

**Delete rules:**
- Deleting the last remaining program is blocked (button disabled with hint).
- Delete confirms via the app's existing confirm pattern; warns day count ("Delete 'PPL' and its 3 workout days?"). Days are deleted with the program (history is untouched — logged sessions remain).
- Deleting the active program activates the first remaining program.

**Styling:** grep `styles.css` before using any new utility class; missing classes go inline `style={{...}}` (known constraint). Dropdown needs `position: relative/absolute` anchoring and a click-outside close (same approach as any existing popover, else simple backdrop div).

---

## 4. Import / Export / Backup

### Export (program-scoped)
Exports the ACTIVE program only. Format: existing export shape plus `programName`. Filename includes program name.

### Import (creates program)
Import file → new program (name from file's `programName`, else filename, else "Imported Program"), made active. Never merges into an existing program. Old-format files (bare workouts array / current format) import the same way with fallback name.

### Backup file (v1.2 → v1.3)
- v1.3 adds `programs: [...]` and `activeProgramId`; drops top-level `workouts` from what's written.
- Restore v1.3 → restore programs + active id directly.
- Restore v1.2/older (has `workouts`, no `programs`) → wrap workouts into single "My Program" (same migration logic, extracted into one shared helper `wrapWorkoutsAsProgram`).
- All other backup fields (history, bodyWeightHistory, etc.) unchanged.

---

## 5. Error Handling

- Missing/corrupt `activeProgramId` → first program.
- Zero programs (fresh install) → auto-create empty "My Program" so home view and Add Day always have a target.
- Migration failure falls back the same way current IDB failure does (localStorage backup path).

---

## 6. Testing (Playwright, local, before any push)

Seed/verify via existing harness conventions (`__test-*.js` at root, global npm Playwright, DOM `.click()` via eval, dismiss resume modal):

1. **Migration:** pre-seed v2 `workouts` store → reload → single "My Program" contains all days; home view unchanged.
2. **Switch:** create second program, add day to it, switch back and forth — lists filter correctly; `activeProgramId` survives reload.
3. **CRUD:** create, rename, delete (with days); last-program delete blocked.
4. **Add Day** goes to active program only.
5. **Export/Import round-trip:** export program A, import → new program appears, active.
6. **Backup:** v1.3 backup/restore round-trip; restore of a real v1.2 backup file wraps into default program.
7. **Regression:** log a workout end-to-end (start → sets → finish) from a non-default program; history entry renders; week stats count it.

---

## Out of Scope

- Program-scoped history/stats/recommendations (deliberately global).
- `programId` tagging on history entries.
- Deleting the legacy `workouts` store (next release).
- Program scheduling/rotation logic.
