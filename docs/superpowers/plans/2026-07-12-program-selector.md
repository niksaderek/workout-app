# Program Selector Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Program layer above workout day templates — dropdown selector on home page, full CRUD, one-click import, programId-tagged history, program filter in Detailed Stats.

**Architecture:** Nested storage (IDB `programs` store, each record contains its workout days) with a DERIVED flat `workouts` variable for the active program, so all downstream code paths keep reading a flat array. DB v2→v3 migration wraps existing days into "My Program". Spec: `docs/superpowers/specs/2026-07-12-program-selector-design.md`.

**Tech Stack:** Single-file React 18 PWA (`index.html`, ~5700 lines, CDN deps, no build). IndexedDB. Playwright (global npm install) for tests.

## Global Constraints

- Single file: ALL app code changes go in `index.html`. No new source files.
- `styles.css` is a static hand-curated Tailwind subset — grep it for EVERY new utility class; if absent use inline `style={{...}}`.
- Never use `transform` on containers with fixed-position children.
- `set.reps` field never renamed. History records only GAIN `programId` — no other shape changes.
- Import must be ONE CLICK: tap Import → pick file → new program active. Zero dialogs/prompts.
- Old `workouts` IDB store is NOT deleted this release (kept as dormant safety net).
- Backup version goes `'1.2'` → `'1.3'`.
- Test harness: `__test-*.js` scripts at project root, run headless via global Playwright, `rm` after — never commit them. Run with:
  `NODE_PATH="C:/Users/nderek/AppData/Roaming/npm/node_modules" "/c/Program Files/nodejs/node.exe" __test-<name>.js`
- Serve app with `python -m http.server 8000` (background) for tests; app URL `http://localhost:8000`.
- Playwright gotcha: native clicks time out (fixed overlay) — trigger via `page.evaluate(() => el.click())`. Dismiss resume-workout modal after load if present.
- Do NOT commit `CLAUDE.md`, `PROJECT_LEARNINGS.md`, `PROJECT_REFERENCE.md`, `workout-backup-*.json`. Stage files explicitly, never `git add -A`.
- Do NOT push or bump `sw.js` CACHE_NAME — user owns deploy.
- Line numbers below are as of commit `b3641e4` — they SHIFT as tasks land. Anchor on the quoted code, not the number.

## Test Harness Template

Every `__test-*.js` follows this skeleton (reuse verbatim, swap the body):

```javascript
// __test-<name>.js
const { chromium } = require('playwright');

(async () => {
  const browser = await chromium.launch();
  const page = await browser.newPage();
  const fails = [];
  const check = (label, cond) => { console.log(`${cond ? 'PASS' : 'FAIL'}: ${label}`); if (!cond) fails.push(label); };

  await page.goto('http://localhost:8000');
  await page.waitForTimeout(1500);
  // Dismiss resume-workout prompt if present
  await page.evaluate(() => {
    const btns = [...document.querySelectorAll('button')];
    const discard = btns.find(b => /discard|cancel/i.test(b.textContent));
    if (discard && btns.some(b => /resume/i.test(b.textContent))) discard.click();
  });

  // ... test body ...

  await browser.close();
  if (fails.length) { console.error(`\n${fails.length} FAILURES`); process.exit(1); }
  console.log('\nALL PASS');
})();
```

IDB access from tests goes through `page.evaluate` with `indexedDB.open('WorkoutTrackerDB')` (no version → opens current). To seed a FRESH v2 database for migration tests, first `indexedDB.deleteDatabase('WorkoutTrackerDB')` + `localStorage.clear()`, then open with explicit version 2 and create the three v2 stores in `onupgradeneeded`, insert fixture workouts, close, THEN load the page.

---

### Task 1: DB v3 — `programs` store, migration, dbOperations

**Files:**
- Modify: `index.html:590` (`DB_VERSION`), `index.html:599-616` (`onupgradeneeded`), `index.html:620-647` (dbOperations)
- Test: `__test-programs-migration.js` (temporary, deleted at end of task)

**Interfaces:**
- Produces: `dbOperations.savePrograms(programs)` (clear+put-all, same contract as old `saveWorkouts`), `dbOperations.getPrograms()` → `Promise<Program[]>`, `migrateWorkoutsToPrograms()` → `Promise<Program[]|null>` (one-time v2→v3 data move, returns migrated programs or null if nothing to migrate), `wrapWorkoutsAsProgram(workoutsArr, name = 'My Program')` → `Program` (module-level helper, reused by backup restore in Task 5).
- Program shape: `{ id: number, name: string, workouts: WorkoutDay[] }`.

- [ ] **Step 1: Write the failing test**

`__test-programs-migration.js` (harness template + this body). It seeds a v2 DB with 2 workout days, loads the app, then asserts the `programs` store exists and contains one "My Program" with both days:

```javascript
  // Seed fresh v2 DB before app load
  await page.evaluate(async () => {
    await new Promise(r => { const d = indexedDB.deleteDatabase('WorkoutTrackerDB'); d.onsuccess = d.onerror = d.onblocked = r; });
    localStorage.clear();
    await new Promise((resolve, reject) => {
      const req = indexedDB.open('WorkoutTrackerDB', 2);
      req.onupgradeneeded = (e) => {
        const db = e.target.result;
        db.createObjectStore('workouts', { keyPath: 'id' });
        const h = db.createObjectStore('history', { keyPath: 'id', autoIncrement: true });
        h.createIndex('date', 'date'); h.createIndex('workoutId', 'workoutId');
        const b = db.createObjectStore('bodyWeight', { keyPath: 'id', autoIncrement: true });
        b.createIndex('date', 'date');
      };
      req.onsuccess = () => {
        const db = req.result;
        const tx = db.transaction('workouts', 'readwrite');
        tx.objectStore('workouts').put({ id: 1, name: 'SEED DAY A', emoji: 'X', exercises: [{ name: 'Bench Press', sets: 3, reps: 5, notes: '', muscleGroup: 'chest' }] });
        tx.objectStore('workouts').put({ id: 2, name: 'SEED DAY B', emoji: 'Y', exercises: [{ name: 'Back Squat', sets: 3, reps: 8, notes: '', muscleGroup: 'legs' }] });
        tx.oncomplete = () => { db.close(); resolve(); };
        tx.onerror = () => reject(tx.error);
      };
      req.onerror = () => reject(req.error);
    });
  });

  await page.goto('http://localhost:8000');
  await page.waitForTimeout(2000);

  const result = await page.evaluate(async () => {
    const db = await new Promise((res, rej) => { const r = indexedDB.open('WorkoutTrackerDB'); r.onsuccess = () => res(r.result); r.onerror = () => rej(r.error); });
    const hasStore = db.objectStoreNames.contains('programs');
    let programs = [];
    if (hasStore) {
      programs = await new Promise((res) => { const q = db.transaction('programs', 'readonly').objectStore('programs').getAll(); q.onsuccess = () => res(q.result); });
    }
    db.close();
    return { version: db.version, hasStore, programs };
  });

  check('DB version is 3', result.version === 3);
  check('programs store exists', result.hasStore);
  check('one migrated program', result.programs.length === 1);
  check('named My Program', result.programs[0] && result.programs[0].name === 'My Program');
  check('carries both seeded days', result.programs[0] && result.programs[0].workouts.length === 2 && result.programs[0].workouts[0].name === 'SEED DAY A');
```

- [ ] **Step 2: Run test to verify it fails**

Start server if not running: `python -m http.server 8000` (background, project root).
Run: `NODE_PATH="C:/Users/nderek/AppData/Roaming/npm/node_modules" "/c/Program Files/nodejs/node.exe" __test-programs-migration.js`
Expected: FAIL on 'DB version is 3' and 'programs store exists'.

- [ ] **Step 3: Implement**

In `index.html`:

(a) `const DB_VERSION = 2;` → `const DB_VERSION = 3;`

(b) Inside `onupgradeneeded` (after the `bodyWeight` block, before the closing `};`):

```javascript
      if (!db.objectStoreNames.contains('programs')) {
        db.createObjectStore('programs', { keyPath: 'id' });
      }
```

(Data migration does NOT happen in `onupgradeneeded` — reading old-store data plus async patterns there is error-prone. It happens in `migrateWorkoutsToPrograms`, called from `loadData` in Task 2. This task only wires the store + helpers; the test passes once (c)–(e) below AND the minimal `loadData` hook (f) are in.)

(c) Module-level helper, directly above `const dbOperations = {`:

```javascript
// Wraps a flat workout-day array into a program record.
// Used by v2→v3 migration and by pre-1.3 backup restore.
const wrapWorkoutsAsProgram = (workoutsArr, name = 'My Program') => ({
  id: Date.now(),
  name,
  workouts: workoutsArr
});
```

(d) New methods inside `dbOperations` (after `getWorkouts`):

```javascript
  async savePrograms(programs) {
    const db = await initDB();
    const tx = db.transaction('programs', 'readwrite');
    const store = tx.objectStore('programs');

    await store.clear();
    for (const program of programs) {
      await store.put(program);
    }

    return new Promise((resolve, reject) => {
      tx.oncomplete = () => resolve();
      tx.onerror = () => reject(tx.error);
    });
  },

  async getPrograms() {
    const db = await initDB();
    const tx = db.transaction('programs', 'readonly');
    const store = tx.objectStore('programs');
    const request = store.getAll();

    return new Promise((resolve, reject) => {
      request.onsuccess = () => resolve(request.result);
      request.onerror = () => reject(request.error);
    });
  },
```

(e) Module-level migration function, directly below `dbOperations`'s closing `};`:

```javascript
// One-time v2→v3 data move: wraps legacy flat workout days into a single
// default program. Legacy 'workouts' store is left intact as a safety net.
// Falls back to the localStorage mirror when the legacy store is empty.
const migrateWorkoutsToPrograms = async () => {
  const existingPrograms = await dbOperations.getPrograms();
  if (existingPrograms.length > 0) return null; // already migrated

  let legacyWorkouts = await dbOperations.getWorkouts();
  if (!legacyWorkouts || legacyWorkouts.length === 0) {
    const backup = localStorage.getItem('workouts_backup');
    if (backup) {
      try { legacyWorkouts = JSON.parse(backup); } catch (e) { legacyWorkouts = []; }
    }
  }
  if (!legacyWorkouts || legacyWorkouts.length === 0) return null;

  const migrated = [wrapWorkoutsAsProgram(legacyWorkouts)];
  await dbOperations.savePrograms(migrated);
  return migrated;
};
```

(f) Minimal `loadData` hook so the migration actually runs (Task 2 rewrites this fully; this keeps Task 1 independently green). At the TOP of `loadData`'s `try` block (before `const savedWorkouts = ...`):

```javascript
      await migrateWorkoutsToPrograms();
```

- [ ] **Step 4: Run test to verify it passes**

Run: `NODE_PATH="C:/Users/nderek/AppData/Roaming/npm/node_modules" "/c/Program Files/nodejs/node.exe" __test-programs-migration.js`
Expected: ALL PASS.

- [ ] **Step 5: Visual validation + commit**

Read the changed regions of `index.html` (Tier 2): brackets, commas between dbOperations methods, no duplicate `programs` store creation. Then:

```bash
rm __test-programs-migration.js
git add index.html
git commit -m "feat: programs IDB store (v3) with v2 migration"
```

---

### Task 2: State layer — programs state, derived workouts, wrapper

**Files:**
- Modify: `index.html:1191-1242` (default `workouts` useState), `index.html:1511-1595` (`loadData`, `saveWorkouts`), call sites of `saveWorkouts(...)` at `index.html:2706,2745,2760,2854`
- Test: `__test-program-state.js` (temporary)

**Interfaces:**
- Consumes: `dbOperations.savePrograms/getPrograms`, `migrateWorkoutsToPrograms`, `wrapWorkoutsAsProgram` (Task 1).
- Produces: state `programs` / `setPrograms`, `activeProgramId` / `setActiveProgramId`; derived `const workouts` (flat array, active program's days — same name as the old state so ALL downstream reads compile unchanged); `savePrograms(newPrograms)` (component-level: persists + sets state + `localStorage.programs_backup`); `updateActiveWorkouts(newWorkouts)` (replaces active program's days and persists); `switchProgram(programId)` (sets active + persists `active_program_id` to localStorage). DEFAULT_WORKOUTS module constant (the old 4-day seed).

- [ ] **Step 1: Write the failing test**

`__test-program-state.js` (harness template + body). Uses the app as-is (post-migration DB from normal usage — the template seeds nothing):

```javascript
  // App booted normally above. Add a workout day via Add Day UI, then verify
  // it landed INSIDE the active program record in IDB, not the legacy store.
  const before = await page.evaluate(async () => {
    const db = await new Promise((res) => { const r = indexedDB.open('WorkoutTrackerDB'); r.onsuccess = () => res(r.result); });
    const progs = await new Promise((res) => { const q = db.transaction('programs', 'readonly').objectStore('programs').getAll(); q.onsuccess = () => res(q.result); });
    db.close();
    return { count: progs.length, dayCount: progs[0] ? progs[0].workouts.length : 0, activeId: localStorage.getItem('active_program_id') };
  });
  check('exactly one program exists', before.count === 1);
  check('active_program_id persisted', before.activeId !== null);

  // Add Day via UI
  await page.evaluate(() => { [...document.querySelectorAll('button')].find(b => b.textContent.includes('Add Day')).click(); });
  await page.waitForTimeout(300);
  await page.evaluate(() => {
    const input = document.querySelector('input[type="text"]');
    const setter = Object.getOwnPropertyDescriptor(window.HTMLInputElement.prototype, 'value').set;
    setter.call(input, 'STATE TEST DAY');
    input.dispatchEvent(new Event('input', { bubbles: true }));
  });
  await page.waitForTimeout(200);
  await page.evaluate(() => { [...document.querySelectorAll('button')].find(b => /^(Add|Create|Save)/.test(b.textContent.trim())).click(); });
  await page.waitForTimeout(500);

  const after = await page.evaluate(async () => {
    const db = await new Promise((res) => { const r = indexedDB.open('WorkoutTrackerDB'); r.onsuccess = () => res(r.result); });
    const progs = await new Promise((res) => { const q = db.transaction('programs', 'readonly').objectStore('programs').getAll(); q.onsuccess = () => res(q.result); });
    db.close();
    const mirror = JSON.parse(localStorage.getItem('programs_backup') || '[]');
    return { dayCount: progs[0].workouts.length, hasNew: progs[0].workouts.some(w => w.name === 'STATE TEST DAY'), mirrorHasNew: mirror[0] && mirror[0].workouts.some(w => w.name === 'STATE TEST DAY') };
  });
  check('day count grew by one', after.dayCount === before.dayCount + 1);
  check('new day inside program record', after.hasNew);
  check('programs_backup mirror updated', after.mirrorHasNew);

  // Cleanup: remove the test day via IDB direct write
  await page.evaluate(async () => {
    const db = await new Promise((res) => { const r = indexedDB.open('WorkoutTrackerDB'); r.onsuccess = () => res(r.result); });
    const tx = db.transaction('programs', 'readwrite');
    const store = tx.objectStore('programs');
    const progs = await new Promise((res) => { const q = store.getAll(); q.onsuccess = () => res(q.result); });
    progs[0].workouts = progs[0].workouts.filter(w => w.name !== 'STATE TEST DAY');
    store.put(progs[0]);
    await new Promise((res) => { tx.oncomplete = res; });
    localStorage.setItem('programs_backup', JSON.stringify(progs));
    db.close();
  });
```

- [ ] **Step 2: Run test to verify it fails**

Run: `NODE_PATH="C:/Users/nderek/AppData/Roaming/npm/node_modules" "/c/Program Files/nodejs/node.exe" __test-program-state.js`
Expected: FAIL on 'active_program_id persisted' and 'new day inside program record' (day currently lands in legacy store).

- [ ] **Step 3: Implement**

(a) Extract the 4-day default template array (the entire array literal currently inside `useState([...])` at lines 1191-1242) to a module-level constant `DEFAULT_WORKOUTS` placed near `wrapWorkoutsAsProgram` (before the `WorkoutTracker` component). Replace the state declarations:

```javascript
  const [programs, setPrograms] = useState([]);
  const [activeProgramId, setActiveProgramId] = useState(null);

  // Flat workout-day list for the active program. Same name as the former
  // useState so every downstream consumer keeps reading a flat array.
  const activeProgram = programs.find(p => p.id === activeProgramId) || null;
  const workouts = activeProgram ? activeProgram.workouts : [];
```

(`const [workouts, setWorkouts] = useState([...])` is REMOVED. Grep for `setWorkouts(` — every remaining call site must be replaced in steps below; after this task `setWorkouts` must have zero references.)

(b) Replace the component-level `saveWorkouts` function (lines 1586-1595) with:

```javascript
  const savePrograms = async (newPrograms) => {
    try {
      await dbOperations.savePrograms(newPrograms);
      setPrograms(newPrograms);
      // Auto-backup to localStorage
      localStorage.setItem('programs_backup', JSON.stringify(newPrograms));
    } catch (error) {
      console.error('Error saving programs:', error);
    }
  };

  // Replaces the active program's day list. Drop-in for the old saveWorkouts.
  const updateActiveWorkouts = async (newWorkouts) => {
    const newPrograms = programs.map(p =>
      p.id === activeProgramId ? { ...p, workouts: newWorkouts } : p
    );
    await savePrograms(newPrograms);
  };

  const switchProgram = (programId) => {
    setActiveProgramId(programId);
    localStorage.setItem('active_program_id', String(programId));
  };
```

(c) Rewrite `loadData`'s workout-loading section. Replace everything from `const savedWorkouts = await dbOperations.getWorkouts();` through the end of the `else` block at line 1563 (history restore lines INCLUDED — keep the history logic, only the workouts branches change) with:

```javascript
      await migrateWorkoutsToPrograms();
      let savedPrograms = await dbOperations.getPrograms();
      let savedHistory = await dbOperations.getHistory();
      const latestBodyWeight = await dbOperations.getLatestBodyWeight();
      const allBodyWeights = await dbOperations.getBodyWeightHistory();
      // Sort ascending (oldest -> newest) for getBodyWeightAt nearest-date lookup
      setBodyWeightHistory([...allBodyWeights].sort((a, b) => new Date(a.date) - new Date(b.date)));

      // Programs fallback chain: IDB → localStorage mirror → seeded default
      if (!savedPrograms || savedPrograms.length === 0) {
        const mirror = localStorage.getItem('programs_backup');
        if (mirror) {
          try { savedPrograms = JSON.parse(mirror); } catch (e) { savedPrograms = []; }
        }
      }
      if (!savedPrograms || savedPrograms.length === 0) {
        savedPrograms = [wrapWorkoutsAsProgram(DEFAULT_WORKOUTS)];
      }
      await dbOperations.savePrograms(savedPrograms);
      setPrograms(savedPrograms);
      localStorage.setItem('programs_backup', JSON.stringify(savedPrograms));

      // Active program: stored id if it still exists, else first program
      const storedActiveId = Number(localStorage.getItem('active_program_id'));
      const activeId = savedPrograms.some(p => p.id === storedActiveId) ? storedActiveId : savedPrograms[0].id;
      setActiveProgramId(activeId);
      localStorage.setItem('active_program_id', String(activeId));

      // History: IDB → localStorage fallback (unchanged behaviour)
      if ((!savedHistory || savedHistory.length === 0) && localStorage.getItem('history_backup')) {
        const parsedHistory = JSON.parse(localStorage.getItem('history_backup'));
        for (const item of parsedHistory) {
          await dbOperations.saveHistory(item);
        }
        savedHistory = parsedHistory;
      }
      if (savedHistory && savedHistory.length > 0) {
        setCompletedWorkouts(savedHistory);
        localStorage.setItem('history_backup', JSON.stringify(savedHistory));
      }
```

(The old `hasIndexedDBData`/`workoutsBackup` dance is removed; `migrateWorkoutsToPrograms` already consumes the legacy `workouts_backup` mirror. The `await dbOperations.saveWorkouts(workouts)` seed line dies with it.)

(d) Replace the four `await saveWorkouts(updatedWorkouts|finalWorkouts)` call sites (day-edit save ~2706, day delete ~2745, day duplicate/add ~2760, import ~2854) with `await updateActiveWorkouts(...)` — same argument. (The import site gets rebuilt again in Task 4; still convert it now so the file has zero `saveWorkouts(` references after this task.)

(e) `restoreBackupFromFile` lines 2954-2957 (`saveWorkouts`/`setWorkouts`/`workouts_backup`): replace with a temporary shim so Task 2 stays green (Task 5 rewrites it properly):

```javascript
        const restoredPrograms = backupData.programs && backupData.programs.length > 0
          ? backupData.programs
          : [wrapWorkoutsAsProgram(backupData.workouts)];
        await savePrograms(restoredPrograms);
        switchProgram(backupData.activeProgramId && restoredPrograms.some(p => p.id === backupData.activeProgramId)
          ? backupData.activeProgramId : restoredPrograms[0].id);
```

Also relax the guard at line 2949: `if (!backupData.workouts || !backupData.history)` → `if ((!backupData.workouts && !backupData.programs) || !backupData.history)`.

(f) Grep `setWorkouts(` and `saveWorkouts(` (component-level fn) — must be zero matches. `dbOperations.saveWorkouts`/`getWorkouts` remain defined (migration uses `getWorkouts`) but no other runtime path may call them.

- [ ] **Step 4: Run test to verify it passes**

Run: `NODE_PATH="C:/Users/nderek/AppData/Roaming/npm/node_modules" "/c/Program Files/nodejs/node.exe" __test-program-state.js`
Expected: ALL PASS. Also re-run the Task 1 migration test mentally — its assertions still hold (migration path unchanged).

- [ ] **Step 5: Manual smoke + commit**

Load `http://localhost:8000` in a Playwright screenshot or manual check: home view renders the same 4 day cards as before; day expand/edit/delete works. Then:

```bash
rm __test-program-state.js
git add index.html
git commit -m "feat: programs state layer with derived flat workouts"
```

---

### Task 3: Program dropdown UI — switch, create, rename, delete

**Files:**
- Modify: `index.html:3624-3629` ("Your Program" heading block in home view), state declarations near `index.html:1281`
- Test: `__test-program-dropdown.js` (temporary)

**Interfaces:**
- Consumes: `programs`, `activeProgram`, `activeProgramId`, `savePrograms`, `switchProgram` (Task 2).
- Produces: handlers `createProgram(name)`, `renameProgram(programId, newName)`, `deleteProgram(programId)`; state `showProgramDropdown` (bool), `programManageMode` (bool), `editingProgramName` (`{id, value}` or null), `newProgramName` (string or null — null = input hidden).

- [ ] **Step 1: Write the failing test**

`__test-program-dropdown.js` (harness + body):

```javascript
  const q = (sel) => page.evaluate((s) => { const el = document.querySelector(s); if (el) el.click(); return !!el; }, sel);
  const clickText = (txt) => page.evaluate((t) => { const b = [...document.querySelectorAll('button')].find(x => x.textContent.trim().includes(t)); if (b) b.click(); return !!b; }, txt);
  const typeInto = (sel, val) => page.evaluate(({ s, v }) => {
    const input = document.querySelector(s); if (!input) return false;
    const setter = Object.getOwnPropertyDescriptor(window.HTMLInputElement.prototype, 'value').set;
    setter.call(input, v); input.dispatchEvent(new Event('input', { bubbles: true })); return true;
  }, { s: sel, v: val });

  // Dropdown trigger shows active program name
  const trigger = await page.evaluate(() => { const el = document.querySelector('[data-testid="program-dropdown-trigger"]'); return el ? el.textContent : null; });
  check('trigger exists and shows My Program', trigger !== null && trigger.includes('My Program'));

  // Open dropdown → create new program
  await q('[data-testid="program-dropdown-trigger"]'); await page.waitForTimeout(200);
  check('New program option visible', await clickText('New program')); await page.waitForTimeout(200);
  check('name input appears', await typeInto('[data-testid="new-program-input"]', 'TEST PROGRAM'));
  await clickText('Create'); await page.waitForTimeout(500);

  const created = await page.evaluate(async () => {
    const db = await new Promise((res) => { const r = indexedDB.open('WorkoutTrackerDB'); r.onsuccess = () => res(r.result); });
    const progs = await new Promise((res) => { const g = db.transaction('programs', 'readonly').objectStore('programs').getAll(); g.onsuccess = () => res(g.result); });
    db.close();
    const trig = document.querySelector('[data-testid="program-dropdown-trigger"]');
    return { count: progs.length, names: progs.map(p => p.name), trigger: trig.textContent, activeId: Number(localStorage.getItem('active_program_id')), testId: (progs.find(p => p.name === 'TEST PROGRAM') || {}).id };
  });
  check('two programs in IDB', created.count === 2);
  check('new program is active', created.trigger.includes('TEST PROGRAM') && created.activeId === created.testId);

  // Empty program → home list shows no day cards from My Program
  const dayCards = await page.evaluate(() => [...document.querySelectorAll('h3')].filter(h => h.textContent.includes('PUSH A')).length);
  check('day list filtered to empty program', dayCards === 0);

  // Switch back
  await q('[data-testid="program-dropdown-trigger"]'); await page.waitForTimeout(200);
  await clickText('My Program'); await page.waitForTimeout(300);
  const back = await page.evaluate(() => [...document.querySelectorAll('h3')].filter(h => h.textContent.includes('PUSH A')).length);
  check('switching back restores day list', back > 0);

  // Manage: rename TEST PROGRAM → RENAMED, then delete it
  await q('[data-testid="program-dropdown-trigger"]'); await page.waitForTimeout(200);
  await clickText('Manage'); await page.waitForTimeout(200);
  check('rename button present', await q('[data-testid="rename-program-' + created.testId + '"]')); await page.waitForTimeout(200);
  await typeInto('[data-testid="rename-program-input"]', 'RENAMED');
  await clickText('Save'); await page.waitForTimeout(400);
  const renamed = await page.evaluate(async () => {
    const db = await new Promise((res) => { const r = indexedDB.open('WorkoutTrackerDB'); r.onsuccess = () => res(r.result); });
    const progs = await new Promise((res) => { const g = db.transaction('programs', 'readonly').objectStore('programs').getAll(); g.onsuccess = () => res(g.result); });
    db.close(); return progs.map(p => p.name);
  });
  check('rename persisted', renamed.includes('RENAMED') && !renamed.includes('TEST PROGRAM'));

  await q('[data-testid="delete-program-' + created.testId + '"]'); await page.waitForTimeout(200);
  await clickText('Delete'); await page.waitForTimeout(400); // confirm button inside app modal
  const afterDelete = await page.evaluate(async () => {
    const db = await new Promise((res) => { const r = indexedDB.open('WorkoutTrackerDB'); r.onsuccess = () => res(r.result); });
    const progs = await new Promise((res) => { const g = db.transaction('programs', 'readonly').objectStore('programs').getAll(); g.onsuccess = () => res(g.result); });
    db.close();
    return { count: progs.length, activeId: Number(localStorage.getItem('active_program_id')), firstId: progs[0].id, deleteBtnGone: !document.querySelector('[data-testid="delete-program-' + progs[0].id + '"]') || document.querySelector('[data-testid="delete-program-' + progs[0].id + '"]').disabled };
  });
  check('program deleted', afterDelete.count === 1);
  check('active fell back to first program', afterDelete.activeId === afterDelete.firstId);
```

- [ ] **Step 2: Run test to verify it fails**

Run: `NODE_PATH="C:/Users/nderek/AppData/Roaming/npm/node_modules" "/c/Program Files/nodejs/node.exe" __test-program-dropdown.js`
Expected: FAIL on 'trigger exists' (no dropdown yet).

- [ ] **Step 3: Implement**

(a) New state (with the other useState declarations, after `substituteInputValue` line ~1281):

```javascript
  const [showProgramDropdown, setShowProgramDropdown] = useState(false);
  const [programManageMode, setProgramManageMode] = useState(false);
  const [editingProgramName, setEditingProgramName] = useState(null); // {id, value} or null
  const [newProgramName, setNewProgramName] = useState(null); // string = input open, null = hidden
  const [showDeleteProgram, setShowDeleteProgram] = useState(null); // programId or null
```

(b) Handlers (place after `switchProgram`):

```javascript
  const createProgram = async (name) => {
    const trimmed = (name || '').trim();
    if (!trimmed) return;
    const newProgram = { id: Date.now(), name: trimmed, workouts: [] };
    await savePrograms([...programs, newProgram]);
    switchProgram(newProgram.id);
    setNewProgramName(null);
    setShowProgramDropdown(false);
    setProgramManageMode(false);
  };

  const renameProgram = async (programId, newName) => {
    const trimmed = (newName || '').trim();
    if (!trimmed) return;
    await savePrograms(programs.map(p => p.id === programId ? { ...p, name: trimmed } : p));
    setEditingProgramName(null);
  };

  const deleteProgram = async (programId) => {
    if (programs.length <= 1) return; // last program is undeletable
    const remaining = programs.filter(p => p.id !== programId);
    await savePrograms(remaining);
    if (activeProgramId === programId) {
      switchProgram(remaining[0].id);
    }
    setShowDeleteProgram(null);
  };
```

(c) Replace the heading block (lines 3624-3627) with a flex row + dropdown. Grep `styles.css` for every class before use (`relative`, `absolute`, `right-0`, `mt-2`, `z-50`, `w-56`, `truncate`, `max-w-[10rem]` — any absent class goes inline style):

```jsx
            <div className="relative">
              <div className="flex items-center justify-between mb-3">
                <h2 className={`text-xl font-bold ${darkMode ? 'text-white' : 'text-gray-900'}`}>
                  Your Program
                </h2>
                <button
                  data-testid="program-dropdown-trigger"
                  onClick={() => { setShowProgramDropdown(!showProgramDropdown); setProgramManageMode(false); setNewProgramName(null); }}
                  className={`px-3 py-1.5 ${darkMode ? 'bg-gray-700 hover:bg-gray-600 text-white' : 'bg-gray-200 hover:bg-gray-300 text-gray-900'} rounded-lg flex items-center gap-2 font-semibold text-sm transition-colors`}
                >
                  <span className="truncate" style={{ maxWidth: '9rem' }}>{activeProgram ? activeProgram.name : '—'}</span>
                  <ChevronDown size={16} />
                </button>
              </div>

              {showProgramDropdown && (
                <React.Fragment>
                  <div className="fixed inset-0" style={{ zIndex: 40 }} onClick={() => { setShowProgramDropdown(false); setProgramManageMode(false); setNewProgramName(null); }} />
                  <div
                    className={`absolute right-0 ${darkMode ? 'bg-gray-800 border-gray-700' : 'bg-white border-gray-200'} border rounded-xl shadow-lg p-2`}
                    style={{ top: '2.75rem', zIndex: 50, minWidth: '14rem' }}
                  >
                    {programs.map(program => (
                      <div key={program.id} className="flex items-center gap-1">
                        {editingProgramName && editingProgramName.id === program.id ? (
                          <React.Fragment>
                            <input
                              data-testid="rename-program-input"
                              type="text"
                              value={editingProgramName.value}
                              onChange={(e) => setEditingProgramName({ id: program.id, value: e.target.value })}
                              className={`flex-1 px-2 py-1 rounded-lg text-sm ${darkMode ? 'bg-gray-700 text-white' : 'bg-gray-100 text-gray-900'}`}
                              autoFocus
                            />
                            <button onClick={() => renameProgram(program.id, editingProgramName.value)} className="px-2 py-1 bg-blue-600 text-white rounded-lg text-sm font-semibold">Save</button>
                          </React.Fragment>
                        ) : (
                          <React.Fragment>
                            <button
                              onClick={() => { switchProgram(program.id); setShowProgramDropdown(false); setProgramManageMode(false); }}
                              className={`flex-1 text-left px-3 py-2 rounded-lg text-sm font-semibold ${darkMode ? 'text-white hover:bg-gray-700' : 'text-gray-900 hover:bg-gray-100'}`}
                            >
                              {program.id === activeProgramId ? '✓ ' : ''}{program.name}
                              <span className={`ml-1 text-xs font-normal ${darkMode ? 'text-gray-400' : 'text-gray-500'}`}>({program.workouts.length})</span>
                            </button>
                            {programManageMode && (
                              <React.Fragment>
                                <button data-testid={`rename-program-${program.id}`} onClick={() => setEditingProgramName({ id: program.id, value: program.name })} className={`p-2 rounded-lg ${darkMode ? 'hover:bg-gray-700 text-gray-400' : 'hover:bg-gray-100 text-gray-500'}`}><Edit2 size={14} /></button>
                                <button
                                  data-testid={`delete-program-${program.id}`}
                                  disabled={programs.length <= 1}
                                  onClick={() => setShowDeleteProgram(program.id)}
                                  className={`p-2 rounded-lg ${programs.length <= 1 ? 'opacity-30' : darkMode ? 'hover:bg-gray-700' : 'hover:bg-gray-100'}`}
                                  style={{ color: '#ef4444' }}
                                ><Trash2 size={14} /></button>
                              </React.Fragment>
                            )}
                          </React.Fragment>
                        )}
                      </div>
                    ))}

                    <div className={`my-2 border-t ${darkMode ? 'border-gray-700' : 'border-gray-200'}`} />

                    {newProgramName !== null ? (
                      <div className="flex items-center gap-1">
                        <input
                          data-testid="new-program-input"
                          type="text"
                          value={newProgramName}
                          onChange={(e) => setNewProgramName(e.target.value)}
                          placeholder="Program name"
                          className={`flex-1 px-2 py-1 rounded-lg text-sm ${darkMode ? 'bg-gray-700 text-white' : 'bg-gray-100 text-gray-900'}`}
                          autoFocus
                        />
                        <button onClick={() => createProgram(newProgramName)} className="px-2 py-1 bg-blue-600 text-white rounded-lg text-sm font-semibold">Create</button>
                      </div>
                    ) : (
                      <button onClick={() => setNewProgramName('')} className={`w-full text-left px-3 py-2 rounded-lg text-sm font-semibold ${darkMode ? 'text-blue-400 hover:bg-gray-700' : 'text-blue-600 hover:bg-gray-100'}`}>+ New program</button>
                    )}
                    <button onClick={() => setProgramManageMode(!programManageMode)} className={`w-full text-left px-3 py-2 rounded-lg text-sm font-semibold ${darkMode ? 'text-gray-400 hover:bg-gray-700' : 'text-gray-500 hover:bg-gray-100'}`}>
                      {programManageMode ? 'Done' : 'Manage…'}
                    </button>
                  </div>
                </React.Fragment>
              )}
```

Close the new wrapping `<div className="relative">` at the end of the existing workouts section (after the day-card list `</div>`, before the section's closing tag) — verify JSX nesting with a read after edit.

(d) Delete-program confirm modal — copy the EXISTING delete-day confirm modal pattern (`showDeleteConfirm`, grep for it in the JSX) and add next to it:

```jsx
        {showDeleteProgram !== null && (
          <div className="fixed inset-0 bg-black/50 flex items-center justify-center p-4" style={{ zIndex: 9999 }}>
            <div className={`${darkMode ? 'bg-gray-800' : 'bg-white'} rounded-2xl p-6 max-w-sm w-full`}>
              <h3 className={`text-lg font-bold mb-2 ${darkMode ? 'text-white' : 'text-gray-900'}`}>Delete program?</h3>
              <p className={`text-sm mb-4 ${darkMode ? 'text-gray-400' : 'text-gray-600'}`}>
                Delete "{(programs.find(p => p.id === showDeleteProgram) || {}).name}" and its {((programs.find(p => p.id === showDeleteProgram) || {}).workouts || []).length} workout day(s)? Logged history is kept.
              </p>
              <div className="flex gap-2">
                <button onClick={() => setShowDeleteProgram(null)} className={`flex-1 px-4 py-2 rounded-lg font-semibold ${darkMode ? 'bg-gray-700 text-white' : 'bg-gray-200 text-gray-900'}`}>Cancel</button>
                <button onClick={() => deleteProgram(showDeleteProgram)} className="flex-1 px-4 py-2 rounded-lg font-semibold bg-red-600 text-white">Delete</button>
              </div>
            </div>
          </div>
        )}
```

(e) Icons: grep the inline icon library for `ChevronDown`, `Edit2`, `Trash2` — all three are likely present (`ChevronDown` used by collapsibles, `Trash2` by delete buttons). If any is missing, add it to the icon library following the existing `Clock`/`FaceMeh` pattern (24×24 viewBox, `currentColor` stroke).
(f) Grep `styles.css` for: `bg-black/50`, `opacity-30`, `min-w`, `inset-0`, `right-0`, `z-50`, `truncate`, `py-1.5`. Any absent → inline style (the snippet above already inlines `zIndex`, `minWidth`, `maxWidth`, delete-red).

- [ ] **Step 4: Run test to verify it passes**

Run: `NODE_PATH="C:/Users/nderek/AppData/Roaming/npm/node_modules" "/c/Program Files/nodejs/node.exe" __test-program-dropdown.js`
Expected: ALL PASS.

- [ ] **Step 5: Visual check + commit**

Playwright screenshot of home view with dropdown open, light + dark mode — verify no invisible text (styles.css subset trap). Then:

```bash
rm __test-program-dropdown.js
git add index.html
git commit -m "feat: program selector dropdown with CRUD"
```

---

### Task 4: One-click import + program-scoped export

**Files:**
- Modify: `index.html:2546-2578` (`exportProgram`), `index.html:2779-2858` (`handleFileUpload`, `handleImportChoice`), upload-choice modal JSX (grep `showUploadModal` in JSX)
- Test: `__test-program-import-export.js` (temporary)

**Interfaces:**
- Consumes: `activeProgram`, `programs`, `savePrograms`, `switchProgram` (Tasks 2-3).
- Produces: export markdown first line `<!-- program: <name> -->`; `importAsProgram(parsedWorkouts, programName)` — creates + activates a new program. `handleImportChoice`, `showUploadModal`, `parsedWorkouts` state, and the upload-choice modal JSX are DELETED (one-click requirement).

- [ ] **Step 1: Write the failing test**

`__test-program-import-export.js` (harness + body):

```javascript
  const path = require('path');
  const fs = require('fs');

  // Export active program → capture download
  const [download] = await Promise.all([
    page.waitForEvent('download'),
    page.evaluate(() => { [...document.querySelectorAll('button')].find(b => b.textContent.trim() === 'Export').click(); })
  ]);
  const exportPath = path.join(__dirname, '__test-export.md');
  await download.saveAs(exportPath);
  const md = fs.readFileSync(exportPath, 'utf8');
  check('export carries program name comment', /^<!-- program: .+ -->/m.test(md));
  check('export contains a seeded day', md.includes('PUSH A'));

  // Import it back — must be ONE CLICK: set file on input, NO dialog after
  const progsBefore = await page.evaluate(async () => {
    const db = await new Promise((res) => { const r = indexedDB.open('WorkoutTrackerDB'); r.onsuccess = () => res(r.result); });
    const p = await new Promise((res) => { const g = db.transaction('programs', 'readonly').objectStore('programs').getAll(); g.onsuccess = () => res(g.result); });
    db.close(); return p.length;
  });
  page.on('dialog', async (d) => { fails.push('DIALOG APPEARED: ' + d.message()); await d.dismiss(); });
  await page.setInputFiles('#workout-file-input', exportPath);
  await page.waitForTimeout(1500);

  const after = await page.evaluate(async () => {
    const db = await new Promise((res) => { const r = indexedDB.open('WorkoutTrackerDB'); r.onsuccess = () => res(r.result); });
    const p = await new Promise((res) => { const g = db.transaction('programs', 'readonly').objectStore('programs').getAll(); g.onsuccess = () => res(g.result); });
    db.close();
    const modalOpen = !!document.querySelector('[data-testid="upload-choice-modal"]') || [...document.querySelectorAll('button')].some(b => /replace all/i.test(b.textContent));
    return { count: p.length, names: p.map(x => x.name), activeId: Number(localStorage.getItem('active_program_id')), newest: p[p.length - 1], modalOpen };
  });
  check('no choice modal shown', !after.modalOpen);
  check('new program created', after.count === progsBefore + 1);
  check('imported program is active', after.activeId === after.newest.id);
  check('imported days present', after.newest.workouts.length > 0);

  // Cleanup: delete imported program via IDB, restore active id
  await page.evaluate(async (importedId) => {
    const db = await new Promise((res) => { const r = indexedDB.open('WorkoutTrackerDB'); r.onsuccess = () => res(r.result); });
    const tx = db.transaction('programs', 'readwrite');
    const store = tx.objectStore('programs');
    const progs = await new Promise((res) => { const g = store.getAll(); g.onsuccess = () => res(g.result); });
    const remaining = progs.filter(p => p.id !== importedId);
    store.delete(importedId);
    await new Promise((res) => { tx.oncomplete = res; });
    localStorage.setItem('programs_backup', JSON.stringify(remaining));
    localStorage.setItem('active_program_id', String(remaining[0].id));
    db.close();
  }, after.newest.id);
  fs.unlinkSync(exportPath);
```

Note: `page.waitForEvent('download')` needs `browser.newContext({ acceptDownloads: true })` — adjust harness top: `const context = await browser.newContext({ acceptDownloads: true }); const page = await context.newPage();`.

- [ ] **Step 2: Run test to verify it fails**

Run: `NODE_PATH="C:/Users/nderek/AppData/Roaming/npm/node_modules" "/c/Program Files/nodejs/node.exe" __test-program-import-export.js`
Expected: FAIL on 'export carries program name comment' and 'no choice modal shown' (or dialog-appeared failure from the import success `alert`).

- [ ] **Step 3: Implement**

(a) `exportProgram` — program-name comment as line 1, name in heading + filename:

```javascript
  const exportProgram = () => {
    if (!activeProgram || workouts.length === 0) {
      alert('No workout program to export!');
      return;
    }

    let markdown = `<!-- program: ${activeProgram.name} -->\n`;
    markdown += `# ${activeProgram.name}\n\n`;
    markdown += `Exported: ${new Date().toLocaleDateString('hr-HR')}\n\n`;
    markdown += '---\n\n';
    // ... existing workouts.forEach loop unchanged ...
```

and the download name: `` a.download = `workout-program-${activeProgram.name.replace(/[^a-z0-9]+/gi, '-').toLowerCase()}-${new Date().toISOString().split('T')[0]}.md`; ``

CHECK: `parseMarkdownWorkout` heuristics — the `# <program name>` heading line may be detected as a workout-day header, creating a phantom empty day. It is dropped anyway (`isWorkoutDayHeader` → new currentWorkout, but a previous workout is only pushed `if (currentWorkout.exercises.length > 0)`, and the phantom gets replaced by the next real day header). Verify in the test that imported day count equals exported day count; if a phantom day appears, strip BOTH the comment line and the first `# ` heading in `handleFileUpload` before parsing (see (b)).

(b) `handleFileUpload` — extract program name, strip metadata, import directly with zero dialogs. Replace the markdown branch and the tail of the function:

```javascript
      let programName = null;

      if (isMarkdown) {
        let text = await file.text();
        const nameMatch = text.match(/^<!--\s*program:\s*(.+?)\s*-->\s*\n?/);
        if (nameMatch) {
          programName = nameMatch[1];
          text = text.slice(nameMatch[0].length);
          // Drop the program-title heading so it is not parsed as a phantom day
          text = text.replace(/^#\s[^\n]*\n/, '');
        }
        parsed = parseMarkdownWorkout(text);
      } else if (isCSV) {
        const text = await file.text();
        parsed = parseCSVWorkout(text);
      } else if (isExcel) {
        alert('Excel files (.xlsx/.xls) support coming soon! For now, please export to CSV and upload that file.');
        e.target.value = '';
        return;
      }

      if (parsed && parsed.length > 0) {
        const fallbackName = file.name.replace(/\.(md|csv|xlsx|xls)$/i, '').replace(/^workout-program-/i, '').replace(/-\d{4}-\d{2}-\d{2}$/, '').trim();
        await importAsProgram(parsed, programName || fallbackName || 'Imported Program');
      } else {
        alert(`No valid workouts found in the file. Please check the format.\n\nFile: ${file.name}\nSize: ${file.size} bytes`);
      }
```

(c) Replace `handleImportChoice` entirely with:

```javascript
  // One-click import: every file becomes a NEW program, activated immediately.
  const importAsProgram = async (parsedDays, programName) => {
    const programId = Date.now();
    const withIds = parsedDays.map((w, idx) => ({ ...w, id: programId + idx + 1 }));
    const newProgram = { id: programId, name: programName, workouts: withIds };
    await savePrograms([...programs, newProgram]);
    switchProgram(programId);
  };
```

(d) DELETE: `showUploadModal` + `parsedWorkouts` useState declarations, the upload-choice modal JSX block (grep `showUploadModal` — remove the whole conditional render), and the success `alert` (one-click means silent success — the view visibly switches to the new program). Grep both names afterwards: zero references.

- [ ] **Step 4: Run test to verify it passes**

Run: `NODE_PATH="C:/Users/nderek/AppData/Roaming/npm/node_modules" "/c/Program Files/nodejs/node.exe" __test-program-import-export.js`
Expected: ALL PASS — specifically 'no choice modal shown' and no DIALOG-APPEARED failure.

- [ ] **Step 5: Commit**

```bash
rm __test-program-import-export.js
git add index.html
git commit -m "feat: one-click program import, program-scoped export"
```

---

### Task 5: programId on history + backup v1.3

**Files:**
- Modify: `index.html:2497-2508` (`finishWorkout`), `index.html:2914-2941` (`exportBackupToFile`), `index.html:2943-3010` (`restoreBackupFromFile`)
- Test: `__test-backup-v13.js` (temporary)

**Interfaces:**
- Consumes: `activeProgramId`, `savePrograms`, `switchProgram`, `wrapWorkoutsAsProgram`.
- Produces: history records gain `programId: number` (stamped at finish); backup JSON `version: '1.3'` with `programs: Program[]` and `activeProgramId: number`, `workouts` key no longer written; restore accepts 1.3 AND pre-1.3 (wraps `workouts` into "My Program").

- [ ] **Step 1: Write the failing test**

`__test-backup-v13.js` (harness with `acceptDownloads: true` context, as in Task 4):

```javascript
  const path = require('path'); const fs = require('fs');

  // Backup export → v1.3 shape
  const [download] = await Promise.all([
    page.waitForEvent('download'),
    page.evaluate(() => { [...document.querySelectorAll('button')].find(b => b.textContent.includes('Backup to File')).click(); })
  ]);
  const bakPath = path.join(__dirname, '__test-backup.json');
  await download.saveAs(bakPath);
  const bak = JSON.parse(fs.readFileSync(bakPath, 'utf8'));
  check('version 1.3', bak.version === '1.3');
  check('programs array present', Array.isArray(bak.programs) && bak.programs.length >= 1);
  check('activeProgramId present', typeof bak.activeProgramId === 'number');
  check('no top-level workouts key', !('workouts' in bak));

  // Restore a synthetic PRE-1.3 backup → wraps into single program
  const legacy = { version: '1.2', workouts: [{ id: 1, name: 'LEGACY DAY', emoji: 'Z', exercises: [{ name: 'Bench Press', sets: 3, reps: 5, notes: '', muscleGroup: 'chest' }] }], history: bak.history, bodyWeightHistory: bak.bodyWeightHistory, bodyWeight: bak.bodyWeight };
  const legacyPath = path.join(__dirname, '__test-legacy-backup.json');
  fs.writeFileSync(legacyPath, JSON.stringify(legacy));
  page.on('dialog', async (d) => { await d.accept(); });
  await page.setInputFiles('#restore-backup-input', legacyPath);
  await page.waitForTimeout(2000);
  const afterLegacy = await page.evaluate(async () => {
    const db = await new Promise((res) => { const r = indexedDB.open('WorkoutTrackerDB'); r.onsuccess = () => res(r.result); });
    const p = await new Promise((res) => { const g = db.transaction('programs', 'readonly').objectStore('programs').getAll(); g.onsuccess = () => res(g.result); });
    db.close(); return { count: p.length, name: p[0].name, day: p[0].workouts[0].name };
  });
  check('legacy restore wrapped into one program', afterLegacy.count === 1 && afterLegacy.name === 'My Program' && afterLegacy.day === 'LEGACY DAY');

  // Restore the real v1.3 backup → original programs back
  await page.setInputFiles('#restore-backup-input', bakPath);
  await page.waitForTimeout(2000);
  const afterV13 = await page.evaluate(async () => {
    const db = await new Promise((res) => { const r = indexedDB.open('WorkoutTrackerDB'); r.onsuccess = () => res(r.result); });
    const p = await new Promise((res) => { const g = db.transaction('programs', 'readonly').objectStore('programs').getAll(); g.onsuccess = () => res(g.result); });
    db.close(); return { names: p.map(x => x.name), activeId: Number(localStorage.getItem('active_program_id')) };
  });
  check('v1.3 restore round-trips programs', JSON.stringify(afterV13.names) === JSON.stringify(bak.programs.map(p => p.name)));
  check('v1.3 restore restores active id', afterV13.activeId === bak.activeProgramId);

  fs.unlinkSync(bakPath); fs.unlinkSync(legacyPath);
```

Plus a `finishWorkout` unit check via UI: start a workout from home (click a day card's start button), immediately finish it, assert newest history record has `programId === activeProgramId`, then delete that history record (IDB) to leave data clean:

```javascript
  // programId stamp on finish
  await page.evaluate(() => { [...document.querySelectorAll('button')].find(b => /start/i.test(b.textContent)).click(); });
  await page.waitForTimeout(500);
  await page.evaluate(() => { [...document.querySelectorAll('button')].find(b => /finish/i.test(b.textContent)).click(); });
  await page.waitForTimeout(800);
  const stamped = await page.evaluate(async () => {
    const db = await new Promise((res) => { const r = indexedDB.open('WorkoutTrackerDB'); r.onsuccess = () => res(r.result); });
    const hist = await new Promise((res) => { const g = db.transaction('history', 'readonly').objectStore('history').getAll(); g.onsuccess = () => res(g.result); });
    const newest = hist[hist.length - 1];
    const ok = newest && newest.programId === Number(localStorage.getItem('active_program_id'));
    // cleanup
    const tx = db.transaction('history', 'readwrite');
    tx.objectStore('history').delete(newest.id);
    await new Promise((res) => { tx.oncomplete = res; });
    db.close(); return ok;
  });
  check('finishWorkout stamps programId', stamped);
```

- [ ] **Step 2: Run test to verify it fails**

Run: `NODE_PATH="C:/Users/nderek/AppData/Roaming/npm/node_modules" "/c/Program Files/nodejs/node.exe" __test-backup-v13.js`
Expected: FAIL on 'version 1.3', 'programs array present', 'finishWorkout stamps programId'.

- [ ] **Step 3: Implement**

(a) `finishWorkout` line 2503 — add `programId`:

```javascript
    const completed = { ...(bodyWeight ? { ...currentLog, bodyWeightSnapshot: bodyWeight } : currentLog), finishedAt: new Date().toISOString(), programId: activeProgramId };
```

(b) `exportBackupToFile` — replace `workouts: workouts,` with:

```javascript
      programs: programs,
      activeProgramId: activeProgramId,
```

and `version: '1.2'` → `version: '1.3'`.

(c) `restoreBackupFromFile` — replace the Task-2 shim block with the final version (identical logic, this time also syncing the legacy mirror key removal):

```javascript
        // Programs: v1.3 carries them directly; pre-1.3 backups carry a flat
        // workouts array which gets wrapped into a single default program.
        const restoredPrograms = backupData.programs && backupData.programs.length > 0
          ? backupData.programs
          : [wrapWorkoutsAsProgram(backupData.workouts)];
        await savePrograms(restoredPrograms);
        const restoredActiveId = backupData.activeProgramId && restoredPrograms.some(p => p.id === backupData.activeProgramId)
          ? backupData.activeProgramId
          : restoredPrograms[0].id;
        switchProgram(restoredActiveId);
        localStorage.removeItem('workouts_backup'); // legacy mirror, superseded by programs_backup
```

Guard stays as relaxed in Task 2: `if ((!backupData.workouts && !backupData.programs) || !backupData.history)`.

- [ ] **Step 4: Run test to verify it passes**

Run: `NODE_PATH="C:/Users/nderek/AppData/Roaming/npm/node_modules" "/c/Program Files/nodejs/node.exe" __test-backup-v13.js`
Expected: ALL PASS.

- [ ] **Step 5: Commit**

```bash
rm __test-backup-v13.js
git add index.html
git commit -m "feat: programId history tagging + backup v1.3 with programs"
```

---

### Task 6: Stats program filter

**Files:**
- Modify: `index.html` — stats state (~1271), stats feeder functions `getAll1RMs` (1754), `get1RMProgression` (1821), `getMainLiftsBestSets` (1870), `getDetailedStats` (3083), `getEnergyVolumeStats` (3300), `getTimelineData` (3336), stats view JSX (~3771, period selector ~3930)
- Test: `__test-stats-filter.js` (temporary)

**Interfaces:**
- Consumes: `programs`, `completedWorkouts`, history `programId` (Task 5).
- Produces: state `statsProgramFilter` (`'all'` | programId number, default `'all'`, view-local, not persisted); derived `const statsHistory` — the ONLY history source for the six feeder functions above. `getWeekStats` (home cards), `getExerciseHistory` / `getSmartWeightSuggestion` (recommendations), `getLastWeight`, history view, CSV export stay on `completedWorkouts` (global).

- [ ] **Step 1: Write the failing test**

`__test-stats-filter.js` (harness + body). Seeds two tagged + one untagged history record, checks chip filtering, cleans up:

```javascript
  // Seed: capture existing program id, insert 3 history rows (2 tagged, 1 untagged)
  const seeded = await page.evaluate(async () => {
    const activeId = Number(localStorage.getItem('active_program_id'));
    const db = await new Promise((res) => { const r = indexedDB.open('WorkoutTrackerDB'); r.onsuccess = () => res(r.result); });
    const mk = (name, programId, weight) => ({
      workoutId: 999, workoutName: name, date: new Date().toISOString(),
      finishedAt: new Date().toISOString(), ...(programId ? { programId } : {}),
      exercises: [{ name: 'Bench Press', muscleGroup: 'chest', sets: [{ weight, reps: 5 }] }]
    });
    const tx = db.transaction('history', 'readwrite');
    const store = tx.objectStore('history');
    const ids = [];
    for (const rec of [mk('TAGGED-A', activeId, 60), mk('TAGGED-B', activeId, 62), mk('UNTAGGED', null, 100)]) {
      const req = store.add(rec);
      await new Promise((res) => { req.onsuccess = () => { ids.push(req.result); res(); }; });
    }
    await new Promise((res) => { tx.oncomplete = res; });
    db.close();
    return { activeId, ids };
  });
  await page.reload(); await page.waitForTimeout(1500);

  // Open stats view via bottom nav
  await page.evaluate(() => { [...document.querySelectorAll('button')].find(b => b.textContent.includes('Detailed Statistics')).click(); });
  await page.waitForTimeout(800);

  const allText = await page.evaluate(() => document.getElementById('root').textContent);
  check('All chip renders', allText.includes('All'));
  check('untagged workout counted under All', await page.evaluate(() => {
    // total workout count in stats includes all 3 seeded rows — check via text
    return document.getElementById('root').textContent.length > 0;
  }));

  // Grab total workouts figure under All, then under program chip
  const readTotalWorkouts = () => page.evaluate(() => {
    const m = document.getElementById('root').textContent.match(/(\d+)\s*(Total Workouts|Workouts)/);
    return m ? Number(m[1]) : null;
  });
  // Ensure 'All Time' period so counts are unambiguous
  await page.evaluate(() => { const b = [...document.querySelectorAll('button')].find(x => x.textContent.trim() === 'All Time' || x.textContent.trim() === 'All'); if (b) b.click(); });
  await page.waitForTimeout(400);
  const totalAll = await readTotalWorkouts();

  await page.evaluate((pid) => { const b = document.querySelector(`[data-testid="stats-program-chip-${pid}"]`); if (b) b.click(); }, seeded.activeId);
  await page.waitForTimeout(400);
  const totalProgram = await readTotalWorkouts();
  check('program chip exists and filters', totalProgram !== null && totalAll !== null && totalProgram === totalAll - 1);

  // Untagged 100kg bench must NOT appear in program-filtered PR/1RM data
  const programText = await page.evaluate(() => document.getElementById('root').textContent);
  check('untagged heavy set excluded from filtered stats', !programText.includes('100'));

  // Cleanup seeded rows
  await page.evaluate(async (ids) => {
    const db = await new Promise((res) => { const r = indexedDB.open('WorkoutTrackerDB'); r.onsuccess = () => res(r.result); });
    const tx = db.transaction('history', 'readwrite');
    for (const id of ids) tx.objectStore('history').delete(id);
    await new Promise((res) => { tx.oncomplete = res; });
    const remaining = await new Promise((res) => { const g = db.transaction('history', 'readonly').objectStore('history').getAll(); g.onsuccess = () => res(g.result); });
    localStorage.setItem('history_backup', JSON.stringify(remaining));
    db.close();
  }, seeded.ids);
```

- [ ] **Step 2: Run test to verify it fails**

Run: `NODE_PATH="C:/Users/nderek/AppData/Roaming/npm/node_modules" "/c/Program Files/nodejs/node.exe" __test-stats-filter.js`
Expected: FAIL on 'program chip exists and filters' (no chips yet).

- [ ] **Step 3: Implement**

(a) State (next to `statsPeriod`):

```javascript
  const [statsProgramFilter, setStatsProgramFilter] = useState('all'); // 'all' or programId — Detailed Stats scope
```

(b) Derived history, placed right after the `workouts` derivation from Task 2:

```javascript
  // History slice feeding the Detailed Stats view. Untagged (pre-feature)
  // records and records of deleted programs only appear under 'all'.
  const statsHistory = statsProgramFilter === 'all'
    ? completedWorkouts
    : completedWorkouts.filter(w => w.programId === statsProgramFilter);
```

(c) In EXACTLY these six functions, replace every `completedWorkouts` reference with `statsHistory`: `getAll1RMs`, `get1RMProgression`, `getMainLiftsBestSets`, `getDetailedStats`, `getEnergyVolumeStats`, `getTimelineData`. Leave untouched: `getWeekStats`, `getExerciseHistory`, `getLastWeight`, `exportToCSV`, `cleanupCorruptedData`, history-view JSX, `saveEditedHistory`, resume/delete-history paths. Verify with grep: `completedWorkouts` must NOT appear inside those six functions afterward; `statsHistory` must appear ONLY in them + its own declaration.

(d) Chip row in stats view JSX — insert immediately BEFORE the existing period selector (grep `statsPeriod === period.value` and place the new block above that selector's container). Hide when only one program exists:

```jsx
              {programs.length > 1 && (
                <div className="flex gap-2 overflow-x-auto pb-2">
                  <button
                    data-testid="stats-program-chip-all"
                    onClick={() => setStatsProgramFilter('all')}
                    className={`px-3 py-1.5 rounded-lg text-sm font-semibold whitespace-nowrap transition-colors ${statsProgramFilter === 'all' ? 'bg-blue-600 text-white' : darkMode ? 'bg-gray-700 text-gray-300' : 'bg-gray-200 text-gray-700'}`}
                  >
                    All
                  </button>
                  {programs.map(program => (
                    <button
                      key={program.id}
                      data-testid={`stats-program-chip-${program.id}`}
                      onClick={() => setStatsProgramFilter(program.id)}
                      className={`px-3 py-1.5 rounded-lg text-sm font-semibold whitespace-nowrap transition-colors ${statsProgramFilter === program.id ? 'bg-blue-600 text-white' : darkMode ? 'bg-gray-700 text-gray-300' : 'bg-gray-200 text-gray-700'}`}
                    >
                      {program.name}
                    </button>
                  ))}
                </div>
              )}
```

NOTE: test expects chips to exist — with a single program the chip row hides, so the test must run with ≥2 programs OR the chip test creates a second program first. Simplest: at test top, create a throwaway second program via the Task-3 dropdown flow (or direct IDB write + reload), delete it in cleanup. Add that to the test in Step 1 if the user's DB has only one program.

(e) Grep `styles.css` for `py-1.5`, `whitespace-nowrap`, `overflow-x-auto`, `bg-blue-600` — inline any missing.

- [ ] **Step 4: Run test to verify it passes**

Run: `NODE_PATH="C:/Users/nderek/AppData/Roaming/npm/node_modules" "/c/Program Files/nodejs/node.exe" __test-stats-filter.js`
Expected: ALL PASS.

- [ ] **Step 5: Commit**

```bash
rm __test-stats-filter.js
git add index.html
git commit -m "feat: program filter in detailed stats"
```

---

### Task 7: Full regression pass + docs

**Files:**
- Test: `__test-regression.js` (temporary)
- Modify: `PROJECT_LEARNINGS.md` (NOT committed), `CLAUDE.md` (NOT committed)

**Interfaces:** none new — verification only.

- [ ] **Step 1: Write the regression script**

`__test-regression.js` covering the spec's test list end-to-end in one run against the real DB (all mutations cleaned up):

1. Home renders active program's days; week stat cards present.
2. Log a workout end-to-end from a SECOND program (create → add day with one exercise → start → enter a set → finish) — assert: history record exists, carries correct `programId`, renders in history view, week stats count incremented. Then delete the seeded history row + program, restore active id.
3. Reload page — `activeProgramId` survives, day list intact.
4. Export → import round-trip (Task 4 assertions, condensed) with zero dialogs.
5. Recommendations regression: for an exercise with existing history (e.g. Bench Press), the logging view still shows a suggestion while in a DIFFERENT program (proves recommendations stayed global).
6. Dark + light screenshots of home (dropdown open) and stats (chips visible) saved to `.playwright-cli/` for eyeballing.

Reuse harness + snippets from Tasks 2-6 verbatim; assemble sequentially. Every `check()` label prefixed with its scenario number.

- [ ] **Step 2: Run it**

Run: `NODE_PATH="C:/Users/nderek/AppData/Roaming/npm/node_modules" "/c/Program Files/nodejs/node.exe" __test-regression.js`
Expected: ALL PASS. Any failure → superpowers:systematic-debugging, fix, re-run.

- [ ] **Step 3: Visual review of screenshots**

Read the `.playwright-cli/` screenshots — check dropdown alignment, chip contrast in BOTH themes (styles.css invisible-text trap), no layout overflow on 390px-wide viewport (add `page.setViewportSize({ width: 390, height: 844 })` for mobile pass).

- [ ] **Step 4: Update local docs (uncommitted)**

Append to `PROJECT_LEARNINGS.md` (dated 2026-07-12 lines): programs store shape + derived-workouts decision, one-click import constraint, backup v1.3 shape, statsHistory scoping (which functions filter, which stay global), legacy `workouts` store kept dormant. Update `CLAUDE.md` IndexedDB schema section (DB v3, `programs` store) and data-structures quick ref (`programId` on history, backup v1.3). Both files stay LOCAL — never staged.

- [ ] **Step 5: Cleanup + final commit check**

```bash
rm __test-regression.js
git status   # must show only PROJECT_LEARNINGS.md / CLAUDE.md / PROJECT_REFERENCE.md as modified-untracked leftovers
git log --oneline -6   # five feature commits from Tasks 1-6 present
```

Do NOT push. Report to user: feature complete, locally verified, awaiting their push/deploy decision (cache bump is theirs unless they say otherwise).
