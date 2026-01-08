# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**Workout Pro** is a single-file React PWA (Progressive Web App) for tracking workouts with offline support. The entire application is contained in `workout-tracker.tsx` - a self-contained React component with embedded JSX and Tailwind CSS classes.

**Tech Stack:**
- React 18 (via CDN)
- IndexedDB for local storage
- Lucide React icons
- Tailwind CSS (via CDN)
- No build process - single HTML file deployment

**Deployment Target:** Netlify (configured via `netlify.toml`)

## Development Commands

Since this is a single-file HTML application with no build process:

- **No npm/yarn commands** - React and dependencies loaded via CDN
- **No compilation needed** - Direct deployment of `workout-tracker.tsx`
- **Local testing:** Open file directly in browser or use any local server (e.g., `python -m http.server`)
- **Deployment:** Push to Netlify (auto-deploys from `netlify.toml`)

## Code Architecture

### Single-File Structure (workout-tracker.tsx:1-1304)

The application uses a monolithic component architecture:

**Data Layer** (lines 4-84):
- `initDB()` - IndexedDB initialization with schema versioning
- `dbOperations` - CRUD operations for workouts and history
- Two object stores: `workouts` (workout templates) and `history` (completed workouts)

**State Management** (lines 147-156):
- `workouts` - Workout day templates (4 default days)
- `completedWorkouts` - Historical workout logs
- `activeView` - Navigation state (home/stats/history/logging/edit)
- `currentLog` - Active workout session tracking
- `darkMode` - Theme preference (persisted to localStorage)

**Core Functions:**

1. **Workout Logging** (lines 220-267):
   - `startWorkout()` - Initializes workout session with last weights pre-filled
   - `getLastWeight()` - Retrieves previous weight for progressive overload
   - `updateSet()` / `toggleSetComplete()` - Track sets during workout
   - `finishWorkout()` - Saves completed workout to IndexedDB

2. **Statistics Calculation** (lines 387-611):
   - `getWeekStats()` - Weekly metrics (count, volume, reps, core sets)
   - `getDetailedStats()` - Comprehensive analytics (weekly/monthly comparisons, PRs)
   - Volume calculation: `weight × reps` (excludes core exercises)
   - Personal Records tracking for main lifts (deadlift, squat, bench, OHP, lat pulldown)

3. **Template Management** (lines 305-363):
   - `startEditWorkout()` / `saveEditedWorkout()` - Modify workout day templates
   - `addExercise()` / `deleteExercise()` - Exercise CRUD
   - `addNewWorkoutDay()` / `deleteWorkoutDay()` - Workout day CRUD

4. **Data Export** (lines 269-303):
   - `exportToCSV()` - Export history to CSV with Croatian date formatting

### UI Views (lines 624-1302)

- **Home** (lines 656-768) - Dashboard with stats cards and workout day cards
- **Stats** (lines 770-912) - Detailed analytics, PRs, weekly/monthly trends
- **Logging** (lines 914-1005) - Active workout session with set-by-set tracking
- **History** (lines 1007-1088) - Past workout logs with delete functionality
- **Edit** (lines 1090-1179) - Workout template editor

### Key Data Structures

**Workout Template:**
```javascript
{
  id: number,
  name: string,
  emoji: string,
  exercises: [{ name: string, sets: number, reps: string|number, notes: string }]
}
```

**Workout Log:**
```javascript
{
  id: number (auto-generated),
  workoutId: number,
  workoutName: string,
  workoutEmoji: string,
  date: ISO string,
  exercises: [{
    name: string,
    plannedSets: number,
    plannedReps: string|number,
    sets: [{ weight: string, reps: string|number, completed: boolean }]
  }]
}
```

## Important Implementation Details

### IndexedDB Schema
- Database: `WorkoutTrackerDB` (version 1)
- Store 1: `workouts` - Workout templates (keyPath: `id`)
- Store 2: `history` - Completed workouts (keyPath: `id` auto-increment)
- Indexes on `history`: `date`, `workoutId`

### Core Exercise Detection
Core exercises (plank, hanging leg raises) are treated differently:
- Weight input disabled in UI
- Excluded from volume calculations
- Counted separately in statistics
- Detection: `name.toLowerCase().includes('core'|'plank'|'hanging')`

### Data Persistence
- Workouts: Saved on every template modification
- History: Saved on workout completion
- Theme: `localStorage.setItem('theme', 'dark'|'light')`
- All data stored client-side only (no backend)

### Progressive Overload
`getLastWeight()` (line 208) pre-fills weights from most recent completed workout for same exercise, making it easy to track progression.

### Decimal Weight Support
Weight inputs accept commas and convert to dots (line 249): `value.replace(',', '.')` for European number format.

### Skipped Exercise Handling
Sets with `weight: 0` and `reps: 0` are filtered out in history view and statistics to handle skipped exercises gracefully.

## Common Modifications

**Adding new statistics:**
- Modify `getDetailedStats()` (line 432) to calculate new metrics
- Update stats UI in activeView === 'stats' (line 770)

**Changing workout day defaults:**
- Edit initial `workouts` state (line 87-145)
- Or delete IndexedDB and refresh to reset

**Adding new exercise fields:**
- Update workout template structure (line 87)
- Modify `addExercise()` (line 316) default values
- Update edit UI (line 1117)
- Update logging UI (line 936)

**Changing PR tracking:**
- Modify `mainLiftsForPRs` object (line 499) to add/remove exercises
- Update detection logic (lines 513-541)

## Netlify Deployment

Configured via `netlify.toml`:
- No build command needed
- Publishes root directory (contains single HTML file)
- SPA redirect: All routes → `/index.html`
- Environment: Node 18 (though not used)

To deploy: Connect GitHub repo to Netlify - auto-deploys on push.

## PWA Installation

The app can be installed on Android/iOS:
1. Open in mobile browser (Chrome/Safari)
2. Browser will prompt to "Add to Home Screen"
3. Runs offline after first load (IndexedDB + cached React CDN)

## Data Export/Backup

Users can export workout history to CSV via Download button in History view. CSV format:
```
Date,Workout,Exercise,Set,Weight,Reps
```
Dates use Croatian locale (`hr-HR`).

## Styling Approach

Uses Tailwind utility classes directly in JSX:
- Dark mode: Conditional class names based on `darkMode` state
- Theme colors: Gray-scale with green accents for success states
- Mobile-first: Responsive grid layouts, fixed bottom navigation
- Animations: Gradient backgrounds, hover effects, shadow transitions
