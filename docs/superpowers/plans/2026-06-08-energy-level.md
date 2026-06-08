# Energy Level Feature Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a per-workout energy level (Low/Medium/High) recorded at the top of the logging screen, displayed as a badge in History, and correlated with volume in Stats.

**Architecture:** Single file (`index.html`) — add `energyLevel` to `currentLog` state, wire a new `updateEnergyLevel` handler, insert the picker UI block between the workout header and exercise list, add the history badge, extend `getTimelineData` to carry `energyLevel`, add the emoji overlay to `LineChart`, and add a new `getEnergyVolumeStats` helper + Stats section.

**Tech Stack:** React 18 (CDN), IndexedDB (no schema change needed), Tailwind CSS, inline SVG.

---

### Task 1: Add `energyLevel` to `currentLog` initial state

**Files:**
- Modify: `index.html:2283` (`startWorkout` function)

- [ ] **Step 1: Find the currentLog initial state in startWorkout**

In `startWorkout` (line 2283), find the object passed to `setCurrentLog`. It looks like:
```javascript
setCurrentLog({
  workoutId: workout.id,
  workoutName: workout.name,
  workoutEmoji: workout.emoji,
  date: new Date().toISOString(),
  exercises: workout.exercises.map(item => ({
    ...
    difficulty: null,
    ...
  }))
});
```

- [ ] **Step 2: Add `energyLevel: null` to the currentLog object**

Add the field at the top level of the object (not inside exercises):
```javascript
setCurrentLog({
  workoutId: workout.id,
  workoutName: workout.name,
  workoutEmoji: workout.emoji,
  date: new Date().toISOString(),
  energyLevel: null,
  exercises: workout.exercises.map(item => ({
    ...
  }))
});
```

- [ ] **Step 3: Verify visually**

Open `index.html` in browser, start a workout, open DevTools → Console, run:
```javascript
// Paste into console after starting a workout:
// Check localStorage for currentWorkout
JSON.parse(localStorage.getItem('currentWorkout')).energyLevel
```
Expected output: `null`

- [ ] **Step 4: Commit**
```bash
git add index.html
git commit -m "feat: add energyLevel field to currentLog initial state"
```

---

### Task 2: Add `updateEnergyLevel` handler

**Files:**
- Modify: `index.html` — add handler near other `update*` handlers (around line 2313)

- [ ] **Step 1: Find the updateSet handler** (line ~2313) as a placement reference:
```javascript
const updateSet = (exIdx, setIdx, field, value) => {
  const updated = { ...currentLog };
  ...
  setCurrentLog(updated);
};
```

- [ ] **Step 2: Add `updateEnergyLevel` directly after `updateSet`**
```javascript
const updateEnergyLevel = (level) => {
  setCurrentLog(prev => ({
    ...prev,
    energyLevel: prev.energyLevel === level ? null : level
  }));
};
```
Toggle behaviour: tapping the already-selected level deselects it (sets to null).

- [ ] **Step 3: Commit**
```bash
git add index.html
git commit -m "feat: add updateEnergyLevel handler with toggle behaviour"
```

---

### Task 3: Add energy picker UI to logging screen

**Files:**
- Modify: `index.html:4322` — after the closing `</div>` of the Workout Header block, before the `{/* Exercises */}` map

- [ ] **Step 1: Locate the insertion point**

Find this comment at line ~4324:
```jsx
            {/* Exercises */}
            {currentLog.exercises.map((exercise, exIdx) => (
```

- [ ] **Step 2: Insert the energy picker block between the header div and the exercises map**

```jsx
            {/* Energy Level Picker */}
            <div className={`${darkMode ? 'bg-gray-800 border-gray-700' : 'bg-white border-gray-200'} border rounded-2xl p-4 shadow-sm`}>
              <div className={`text-xs font-semibold uppercase tracking-wide ${darkMode ? 'text-gray-400' : 'text-gray-500'} mb-3`}>
                ⚡ Energy level
              </div>
              <div className="flex gap-2">
                {[
                  { value: 'low', emoji: '🔴', label: 'Low' },
                  { value: 'medium', emoji: '🟡', label: 'Medium' },
                  { value: 'high', emoji: '🟢', label: 'High' }
                ].map(option => (
                  <button
                    key={option.value}
                    onClick={() => updateEnergyLevel(option.value)}
                    className={`flex-1 flex flex-col items-center py-2 px-1 rounded-lg text-sm font-medium transition-all ${
                      currentLog.energyLevel === option.value
                        ? darkMode
                          ? 'bg-blue-600 text-white'
                          : 'bg-blue-500 text-white'
                        : darkMode
                          ? 'bg-gray-700 hover:bg-gray-600 text-gray-300'
                          : 'bg-gray-100 hover:bg-gray-200 text-gray-700'
                    }`}
                  >
                    <span className="text-base leading-tight">{option.emoji}</span>
                    <span className="text-xs leading-tight mt-0.5">{option.label}</span>
                  </button>
                ))}
              </div>
            </div>
```

- [ ] **Step 3: Verify in browser**

Start a workout. The energy picker should appear below the workout header and above the first exercise card. Tap each level — selected one gets blue highlight. Tap again to deselect.

- [ ] **Step 4: Commit**
```bash
git add index.html
git commit -m "feat: add energy level picker to logging screen"
```

---

### Task 4: Add energy badge to History view

**Files:**
- Modify: `index.html:4601` — history workout card header row

- [ ] **Step 1: Locate the history card header row**

Find this block at line ~4601:
```jsx
                  <div className="flex items-start gap-3 mb-4">
                    <WorkoutIcon emoji={workout.workoutEmoji} size={32} ... />
                    <div className="flex-1">
                      <h3 ...>{workout.workoutName}</h3>
                      <p ...>{date string}</p>
                    </div>
                    <div className="flex gap-2">
                      {/* edit/delete buttons */}
                    </div>
                  </div>
```

- [ ] **Step 2: Add energy badge inside the `flex-1` div, below the date `<p>`**

```jsx
                    <div className="flex-1">
                      <h3 className={`text-lg font-bold ${darkMode ? 'text-white' : 'text-gray-900'}`}>{workout.workoutName}</h3>
                      <p className={`text-sm ${darkMode ? 'text-gray-400' : 'text-gray-600'}`}>
                        {new Date(workout.date).toLocaleString('hr-HR', {
                          weekday: 'short',
                          day: 'numeric',
                          month: 'short',
                          hour: '2-digit',
                          minute: '2-digit'
                        })}
                      </p>
                      {workout.energyLevel && (() => {
                        const cfg = {
                          low:    { emoji: '🔴', label: 'Low energy',    bg: darkMode ? 'bg-red-950'   : 'bg-red-100',   text: darkMode ? 'text-red-400'   : 'text-red-700'   },
                          medium: { emoji: '🟡', label: 'Medium energy', bg: darkMode ? 'bg-amber-950' : 'bg-amber-100', text: darkMode ? 'text-amber-400' : 'text-amber-700' },
                          high:   { emoji: '🟢', label: 'High energy',   bg: darkMode ? 'bg-green-950' : 'bg-green-100', text: darkMode ? 'text-green-400' : 'text-green-700' }
                        }[workout.energyLevel];
                        return (
                          <span className={`inline-flex items-center gap-1 mt-1 px-2 py-0.5 rounded-full text-xs font-semibold ${cfg.bg} ${cfg.text}`}>
                            {cfg.emoji} {cfg.label}
                          </span>
                        );
                      })()}
                    </div>
```

- [ ] **Step 3: Verify in browser**

Finish a workout with High energy selected. In History, the card should show a green "🟢 High energy" badge below the date. Workouts with no energy rating show no badge.

- [ ] **Step 4: Commit**
```bash
git add index.html
git commit -m "feat: add energy level badge to history workout cards"
```

---

### Task 5: Pass `energyLevel` through `getTimelineData`

**Files:**
- Modify: `index.html:3195` — `getTimelineData` grouped object initialisation and workout push

- [ ] **Step 1: Locate the grouped object initialisation** (line ~3195):
```javascript
      if (!grouped[key]) {
        grouped[key] = {
          date: key,
          workouts: [],
          volume: 0,
          reps: 0,
          mainLifts: { ... }
        };
      }
      grouped[key].workouts.push(workout);
```

- [ ] **Step 2: Add `energyLevel` field to the grouped initialisation and update it when pushing**

Change to:
```javascript
      if (!grouped[key]) {
        grouped[key] = {
          date: key,
          workouts: [],
          volume: 0,
          reps: 0,
          energyLevel: null,
          mainLifts: { deadlift: [], squat: [], bench: [], ohp: [] }
        };
      }
      grouped[key].workouts.push(workout);
      // Use the energy level of the first rated workout that day
      if (!grouped[key].energyLevel && workout.energyLevel) {
        grouped[key].energyLevel = workout.energyLevel;
      }
```

- [ ] **Step 3: Commit**
```bash
git add index.html
git commit -m "feat: pass energyLevel through getTimelineData"
```

---

### Task 6: Add energy emoji overlay to LineChart (volume chart only)

**Files:**
- Modify: `index.html:283` — `LineChart` component

- [ ] **Step 1: Add energy emoji rendering inside the `{data.map((point, i) => (` block** (line ~371), after the `<circle>` element and before the tooltip `{isHovered && ...}`:

```jsx
              {/* Energy emoji above point (volume chart only) */}
              {metric === 'volume' && point.energyLevel && (
                <text
                  x={x}
                  y={y - 12}
                  textAnchor="middle"
                  fontSize="10"
                >
                  {point.energyLevel === 'low' ? '🔴' : point.energyLevel === 'medium' ? '🟡' : '🟢'}
                </text>
              )}
```

- [ ] **Step 2: Extend the tooltip to include energy level** — inside the `{isHovered && (...)}` block, after the date `<text>` element (line ~422), add:

```jsx
                    {metric === 'volume' && point.energyLevel && (
                      <text
                        x={x}
                        y={y - 4}
                        textAnchor="middle"
                        fontSize="9"
                        fill={darkMode ? '#9CA3AF' : '#6B7280'}
                      >
                        {`Energy: ${point.energyLevel === 'low' ? '🔴 Low' : point.energyLevel === 'medium' ? '🟡 Med' : '🟢 High'}`}
                      </text>
                    )}
```

Also expand the tooltip `<rect>` height from `30` to `42` when `metric === 'volume' && point.energyLevel` — change the rect:
```jsx
                    <rect
                      x={x - 35}
                      y={y - 40}
                      width="70"
                      height={metric === 'volume' && point.energyLevel ? 42 : 30}
                      ...
                    />
```

- [ ] **Step 3: Verify in browser**

Go to Stats → Progress Timeline. Rated workout days should show a small emoji above the dot on the volume chart. Hovering shows "Energy: 🟢 High" in the tooltip.

- [ ] **Step 4: Commit**
```bash
git add index.html
git commit -m "feat: add energy emoji overlay and tooltip to volume timeline chart"
```

---

### Task 7: Add `getEnergyVolumeStats` helper and Stats section

**Files:**
- Modify: `index.html` — add helper near other stat helpers (~line 3150), add Stats JSX section

- [ ] **Step 1: Add `getEnergyVolumeStats` helper** before `getTimelineData` (line ~3153):

```javascript
  const getEnergyVolumeStats = () => {
    const rated = completedWorkouts.filter(w => w.energyLevel);
    if (rated.length < 3) return null;

    const getWorkoutVolume = (workout) =>
      workout.exercises.reduce((total, exercise) => {
        if (!UNIT_CONFIG[getExerciseUnit(exercise)].countsAsVolume) return total;
        return total + exercise.sets.reduce((s, set) => {
          const w = parseFloat(set.weight) || 0;
          const r = parseFloat(set.reps) || 0;
          return s + (w > 0 ? w * r : 0);
        }, 0);
      }, 0);

    const groups = { low: [], medium: [], high: [] };
    rated.forEach(w => groups[w.energyLevel].push(getWorkoutVolume(w)));

    const mean = arr => arr.length === 0 ? null : arr.reduce((a, b) => a + b, 0) / arr.length;
    const meanLow    = mean(groups.low);
    const meanMedium = mean(groups.medium);
    const meanHigh   = mean(groups.high);

    const baseline = meanMedium ?? meanLow ?? meanHigh ?? 1;
    const pct = (val) => val === null ? null : Math.round(((val - baseline) / baseline) * 100);

    return {
      total: rated.length,
      rows: [
        { level: 'high',   emoji: '🟢', label: 'High',   mean: meanHigh,   delta: pct(meanHigh)   },
        { level: 'medium', emoji: '🟡', label: 'Medium', mean: meanMedium, delta: pct(meanMedium) },
        { level: 'low',    emoji: '🔴', label: 'Low',    mean: meanLow,    delta: pct(meanLow)    },
      ].filter(r => r.mean !== null),
      maxMean: Math.max(meanLow ?? 0, meanMedium ?? 0, meanHigh ?? 0)
    };
  };
```

- [ ] **Step 2: Find the Stats section insertion point**

Search for the weekly/monthly comparison cards block — find the closing of those cards (grep for `getDetailedStats` usage in JSX, around line 3576–3800). Insert the energy section after the existing comparison cards and before the PRs/1RM section.

Specifically, find a comment or block boundary like:
```jsx
                {/* Personal Records */}
```
or
```jsx
                {/* Estimated 1RM */}
```
and insert the energy section before it.

- [ ] **Step 3: Add the Energy vs Volume Stats JSX**

```jsx
                {/* Energy vs Volume */}
                {(() => {
                  const stats = getEnergyVolumeStats();
                  if (!stats) return null;
                  return (
                    <div className={`${darkMode ? 'bg-gray-800 border-gray-700' : 'bg-white border-gray-200'} border rounded-2xl p-5 shadow-sm`}>
                      <h3 className={`text-lg font-bold ${darkMode ? 'text-white' : 'text-gray-900'} mb-4`}>
                        ⚡ Energy vs Volume
                      </h3>
                      <div className="space-y-3">
                        {stats.rows.map(row => (
                          <div key={row.level} className="flex items-center gap-3">
                            <div className="w-20 text-sm flex items-center gap-1">
                              <span>{row.emoji}</span>
                              <span className={darkMode ? 'text-gray-300' : 'text-gray-700'}>{row.label}</span>
                            </div>
                            <div className={`flex-1 ${darkMode ? 'bg-gray-700' : 'bg-gray-200'} rounded-full h-4 overflow-hidden`}>
                              <div
                                className={`h-full rounded-full transition-all ${
                                  row.level === 'high' ? 'bg-green-500' :
                                  row.level === 'medium' ? 'bg-yellow-500' : 'bg-red-500'
                                }`}
                                style={{ width: `${Math.round((row.mean / stats.maxMean) * 100)}%` }}
                              />
                            </div>
                            <div className={`w-12 text-right text-sm font-medium ${
                              row.delta === null || row.delta === 0 ? (darkMode ? 'text-gray-400' : 'text-gray-500') :
                              row.delta > 0 ? 'text-green-500' : 'text-red-500'
                            }`}>
                              {row.delta === null || row.delta === 0 ? 'avg' : `${row.delta > 0 ? '+' : ''}${row.delta}%`}
                            </div>
                          </div>
                        ))}
                      </div>
                      <div className={`text-xs ${darkMode ? 'text-gray-500' : 'text-gray-500'} mt-3`}>
                        Based on {stats.total} rated workouts · vs your average volume
                      </div>
                    </div>
                  );
                })()}
```

- [ ] **Step 4: Verify in browser**

Log 3+ workouts with different energy levels. Go to Stats — "⚡ Energy vs Volume" section should appear with bars and delta percentages. Before 3 rated workouts exist the section is hidden.

- [ ] **Step 5: Commit**
```bash
git add index.html
git commit -m "feat: add energy vs volume stats section with correlation bars"
```

---

### Task 8: Bump service worker cache version and deploy

**Files:**
- Modify: `sw.js`

- [ ] **Step 1: Find current CACHE_NAME in sw.js**
```bash
grep "CACHE_NAME" sw.js
```

- [ ] **Step 2: Increment the version number** (e.g. `workout-pro-v9` → `workout-pro-v10`)

- [ ] **Step 3: Commit and push**
```bash
git add sw.js
git commit -m "chore: bump service worker cache to v10 for energy level release"
git push
```

- [ ] **Step 4: Verify on Netlify**

After deploy, open the app. You should see the "New version available!" banner. Click "Update Now" — energy picker should appear at top of logging screen.
