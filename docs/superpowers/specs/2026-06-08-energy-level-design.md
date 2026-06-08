# Energy Level Feature — Design Spec
_Date: 2026-06-08_

## Summary

Add a per-workout energy level rating (Low / Medium / High) recorded before the workout starts. Displayed in History and correlated with volume performance in Stats.

---

## 1. Data Model

Add one field to the workout log object stored in the `history` IndexedDB store:

```javascript
{
  // ...existing fields...
  energyLevel: 'low' | 'medium' | 'high' | null  // null = not rated
}
```

- Field is optional — user can skip without consequence.
- `finishWorkout()` already spreads `currentLog` into the saved record; `energyLevel` carries through automatically.
- No schema migration needed (IndexedDB reads missing fields as `undefined`; treat as `null`).

---

## 2. Logging Screen — Entry UI

**Placement:** Top of the active logging screen (`activeView === 'logging'`), between the workout title/date header and the first exercise block.

**Component:** Three equal-width buttons in a horizontal row inside a card:

| Button | Emoji | Label |
|--------|-------|-------|
| Low    | 🔴    | Low   |
| Medium | 🟡    | Medium |
| High   | 🟢    | High  |

- Emoji stacked above label text, centered.
- All three buttons same width (`flex: 1`).
- Selected state: blue border + blue-tinted background (matches existing difficulty button style).
- Unselected: default gray card background.
- Skippable — no required selection, no validation on finish.
- Wired to `currentLog.energyLevel` via a new `updateEnergyLevel(level)` handler (toggles off if tapped again).
- Section label: `⚡ Energy level` in small gray caps above the buttons.

---

## 3. History View — Badge

Show a colored badge on each past workout card that has a non-null `energyLevel`.

| Level  | Badge text        | Colors (dark mode)                  |
|--------|-------------------|-------------------------------------|
| high   | 🟢 High energy    | green bg (`#14532d`), green text    |
| medium | 🟡 Medium energy  | amber bg (`#451a03`), amber text    |
| low    | 🔴 Low energy     | red bg (`#450a0a`), red text        |

- Positioned top-right of the workout card header row (same row as workout name).
- Hidden when `energyLevel` is null (no badge, no empty space).

---

## 4. Stats — Energy vs Volume Summary

**Location:** Stats view, new section below the existing weekly/monthly comparison cards.

**Render condition:** Only shown when ≥ 3 workouts have a non-null `energyLevel`.

**Calculation:**
1. Compute per-workout total volume (existing logic).
2. Group workouts by `energyLevel`.
3. Compute mean volume per group.
4. Use `medium` group mean as baseline. Show `high` and `low` as `+X%` / `-X%` delta vs baseline.
5. If a group has 0 workouts, omit that row.

**UI:** Horizontal bar chart, one row per energy level (high → medium → low). Bar width proportional to mean volume relative to max. Delta label right-aligned. Footer: "Based on N rated workouts · vs your average volume".

---

## 5. Stats — Timeline Overlay

**Location:** Existing "Total Volume (kg)" timeline chart.

**Change:** When a workout day has a non-null `energyLevel`, render a small emoji (🔴/🟡/🟢) above the bar/dot for that day.

- No emoji when `energyLevel` is null.
- Existing hover tooltip gains an "Energy: 🟢 High" line when rated.
- No change to chart layout, axes, or existing tooltip behavior.

---

## 6. Out of Scope

- No energy input on the Edit-History screen (historical edits not supported for this field).
- No energy filter in History view.
- No per-exercise-type energy correlation.
- No push notifications or reminders tied to energy.

---

## 7. Affected Code Locations

| Area | Change |
|------|--------|
| `currentLog` initial state in `startWorkout()` | Add `energyLevel: null` |
| Logging view JSX | Add energy picker block below header |
| New handler `updateEnergyLevel(level)` | Toggle `currentLog.energyLevel` |
| History view workout card JSX | Add conditional badge |
| `getTimelineData()` | Pass `energyLevel` through per-day data |
| Timeline chart JSX | Render emoji above bar; add to tooltip |
| Stats view JSX | Add Energy vs Volume summary section |
| New helper `getEnergyVolumeStats()` | Compute grouped volume deltas |
