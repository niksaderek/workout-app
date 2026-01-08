# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**Workout Pro** is a single-file React PWA (Progressive Web App) for tracking workouts with offline support. The entire application is contained in `workout-tracker.tsx` - a self-contained React component with embedded JSX and Tailwind CSS classes.

**Tech Stack:**
- React 18 (via CDN)
- IndexedDB for local storage
- Inline SVG icons (Dumbbell, Activity, Zap, Target, Flame, etc.)
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

### Single-File Structure (index.html)

The application uses a monolithic component architecture with all code embedded in `index.html`:

**Icon Components** (lines 34-106):
- Inline SVG icon components (Dumbbell, Activity, Zap, Target, Flame, etc.)
- `WorkoutIcon` - Mapper component that converts emoji identifiers to SVG icons
- `IconPicker` - Visual icon selector component for workout day creation/editing

**Data Layer** (lines 108-166):
- `initDB()` - IndexedDB initialization with schema versioning
- `dbOperations` - CRUD operations for workouts and history
- Two object stores: `workouts` (workout templates) and `history` (completed workouts)

**Markdown Parser** (lines 190-300):
- `parseMarkdownWorkout()` - Intelligent parser supporting multiple markdown formats
- Supports: headers with inline data, bulleted lists, table format, compact notation (3x8)
- Auto-assigns icons cyclically from default icon set
- Handles missing fields with sensible defaults (sets: 3, reps: 10)

**State Management**:
- `workouts` - Workout day templates (4 default days)
- `completedWorkouts` - Historical workout logs
- `activeView` - Navigation state (home/stats/history/logging/edit/edit-history)
- `currentLog` - Active workout session tracking
- `editingWorkout` - Template being edited
- `editingHistory` - Past workout being edited
- `darkMode` - Theme preference (persisted to localStorage)
- `showUploadModal` - Controls markdown upload confirmation modal
- `parsedWorkouts` - Temporarily stores parsed markdown workouts
- `draggedExerciseIndex` - Current exercise being dragged (for reordering)
- `timelineRange` - Selected time range for progress timeline
- `fileInputRef` - React ref for hidden file input element
- `substituteDropdown` - Controls exercise substitution dropdown visibility
- `exerciseProgressModal` - Controls exercise progress chart modal

**Core Functions:**

1. **Workout Logging**:
   - `startWorkout()` - Initializes workout session with smart weight suggestions
   - `getLastWeight()` - Retrieves previous weight for progressive overload
   - `updateSet()` - Update weight/reps for individual sets
   - `finishWorkout()` - Saves completed workout to IndexedDB
   - `getMuscleGroupSuggestions()` - Returns 5-7 exercise alternatives from same muscle group
   - `substituteExercise()` - Swaps exercise name while preserving set data

2. **Statistics Calculation**:
   - `getWeekStats()` - Weekly metrics (count, volume, reps, core sets)
   - `getDetailedStats()` - Comprehensive analytics (weekly/monthly comparisons, PRs)
   - Volume calculation: `weight × reps` (excludes core exercises)
   - Personal Records tracking for main lifts (deadlift, squat, bench, OHP, lat pulldown)
   - `getExerciseProgress()` - Returns weight/volume progression data for any exercise
   - `getMostTrainedExercises()` - Returns top 10 exercises by frequency

3. **Template Management**:
   - `startEditWorkout()` / `saveEditedWorkout()` - Modify workout day templates
   - `addExercise()` / `deleteExercise()` - Exercise CRUD
   - `addNewWorkoutDay()` / `deleteWorkoutDay()` - Workout day CRUD
   - `moveExerciseUp()` / `moveExerciseDown()` - Reorder exercises via up/down buttons (touch-friendly)

4. **History Editing**:
   - `saveEditedHistory()` - Update completed workout in history
   - `updateHistorySet()` - Modify weight/reps for individual sets in past workouts
   - Allows fixing mistakes in logged data without affecting templates

5. **Progress Timeline**:
   - `getTimelineData()` - Aggregates workout history by week/month
   - Tracks volume progression, workout frequency, and main lift PRs
   - Interactive charts with clickable data points

6. **Markdown Upload**:
   - `triggerFileUpload()` - Opens file picker for .md files
   - `handleFileChange()` - Validates and parses uploaded markdown file
   - `handleImportChoice()` - Handles user choice (replace all vs add to existing)
   - Supports multiple markdown formats automatically

7. **Data Export**:
   - `exportToCSV()` - Export history to CSV with Croatian date formatting

8. **Exercise Autocomplete**:
   - `ExerciseInput` component - Dropdown with 60+ popular exercises
   - Organized by muscle group (chest, back, shoulders, legs, arms, core, other)
   - Filters as you type, shows up to 8 suggestions
   - Still allows custom exercise names

9. **Smart Auto-Regulation**:
   - `calculateEstimated1RM()` - Epley formula: weight × (1 + reps / 30)
   - `getSmartWeightSuggestion()` - Returns weight suggestion based on difficulty rating
   - Difficulty-based progression:
     - Easy → +2.5 kg ("felt easy last time")
     - Medium → same weight ("good weight")
     - Hard → same weight ("felt hard last time, repeat")
     - No rating → last weight ("based on last workout")
   - Shows suggestion with Zap icon during logging
   - Optional difficulty rating per exercise (3 buttons: Easy/Medium/Hard)
   - No completion checkboxes needed (removed for simplicity)

10. **Exercise-Specific Progress Charts**:
   - Tap any exercise in Stats → Exercise Progress section
   - Modal shows last 10 workouts with 2 charts:
     - Max Weight Progress (line chart)
     - Total Volume Progress (line chart)
   - Core exercises show Total Reps instead of weight
   - Displays peak weight and total workout count
   - Fuzzy exercise name matching

11. **Mid-Workout Exercise Substitution**:
   - Swap button (↻) next to exercise name in logging view
   - Dropdown shows 5-7 muscle group alternatives
   - Preserves all set data (weight/reps) when substituting
   - Handles custom exercises gracefully (no suggestions)

### UI Views

- **Home** - Dashboard with stats cards, workout day cards, "Upload MD" button
- **Stats** - Detailed analytics, PRs, Exercise Progress section, weekly/monthly trends, progress timeline with interactive charts
- **Logging** - Active workout session with smart weight suggestions, difficulty rating, exercise substitution, and "Add Set" button
- **History** - Past workout logs with edit, delete, and CSV export functionality
- **Edit** - Workout template editor with up/down button exercise reordering (mobile-friendly) and ExerciseInput autocomplete
- **Edit-History** - Edit past workout data (weight/reps) without affecting templates

### Modals

- **Add Day Modal** - Create new workout day with IconPicker
- **Upload Modal** - Markdown upload confirmation (replace vs add to existing)
- **Timeline Detail Modal** - Shows workout details for selected week/month on timeline
- **Exercise Progress Modal** - Shows weight/volume charts for any exercise (last 10 workouts)
- **Exercise Substitution Dropdown** - Shows muscle group alternatives when swapping exercises
- **Delete Confirmation Modals** - Confirm before deleting workout days or history entries

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
    difficulty: 'easy' | 'medium' | 'hard' | null, // Optional difficulty rating
    sets: [{ weight: string, reps: string|number }]
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
Core exercises (plank, hanging leg raises, ab rollout) are treated differently:
- Weight input disabled in UI
- Excluded from volume calculations
- Counted separately in statistics
- Detection: `name.toLowerCase().includes('core'|'plank'|'hanging'|'rollout')`

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

### Inline SVG Icons
The app uses inline SVG components instead of external icon libraries for reliability:
- No CDN dependencies for icons (eliminates loading failures)
- Icons: Dumbbell (🦵), Activity (💪), Zap (⚡), Target (🎯), Flame (🔥), Plus, Edit, Trash2, BarChart3, Download, Clock, Calendar, TrendingUp, Award, Home
- `WorkoutIcon` component maps emoji identifiers to SVG icons for consistent display
- All icons render reliably across devices and network conditions

### Markdown Upload Feature
Users can import workout routines from markdown files:

**Supported Formats:**
1. **Simple Headers**: `# Workout Day`, `## Exercise`, `Sets: 3`, `Reps: 8`
2. **Bulleted Lists**: `## Exercise`, `- Sets: 3`, `- Reps: 8-10`
3. **Table Format**: `| Exercise | Sets | Reps |` with data rows
4. **Compact Notation**: `## Exercise`, `3x8`

**Features:**
- Automatic format detection (no user configuration needed)
- Emoji detection in headers (e.g., `# Leg Day 🦵`)
- Auto-assigns icons cyclically if not specified in markdown
- User choice on import: "Replace All" or "Add to Existing"
- Missing fields default to sensible values (sets: 3, reps: 10)

**Example Markdown:**
```markdown
# Leg Day 🦵
## Squat
Sets: 4
Reps: 8

## Leg Press
3x10

# Upper Body 💪
| Exercise | Sets | Reps |
|----------|------|------|
| Bench Press | 3 | 8 |
| Rows | 3 | 10 |
```

### Icon Picker UI
Visual icon selection component used in:
- Add Day modal (when creating new workout days)
- Edit modal (when modifying existing workout days)
- 5 icons displayed in grid: Dumbbell, Activity, Zap, Target, Flame
- Selected icon highlighted with blue ring
- Replaces previous text-based emoji input for better UX

### PWA Updates & Service Worker
The app uses an intelligent service worker update strategy:
- **Network-first for HTML**: Always fetches latest HTML from network, falls back to cache if offline
- **Cache-first for CDN resources**: React, Tailwind, and other CDN dependencies served from cache for speed
- **Automatic update detection**: Service worker checks for updates on every app launch
- **User-friendly update UI**: Blue banner notification prompts user to update when new version is available
- **Cache versioning**: Service worker version in `sw.js` (currently `workout-pro-v4`)

**How updates work:**
1. When you deploy a new version, the service worker detects the change
2. New service worker installs in background and notifies the app
3. User sees a blue "New version available!" banner at the top
4. Clicking "Update Now" reloads the app with the latest version
5. Old cache is automatically cleared, new version is cached

**To force an update after code changes:**
- Bump `CACHE_NAME` version in `sw.js` (e.g., `workout-pro-v5`)
- Deploy to Netlify
- Users will see the update banner on next app launch or refresh

## Common Modifications

**Adding new statistics:**
- Modify `getDetailedStats()` function to calculate new metrics
- Update stats UI in the Stats view section

**Changing workout day defaults:**
- Edit initial `workouts` state array
- Or delete IndexedDB in browser DevTools → Application → Storage → IndexedDB

**Adding new exercise fields:**
- Update workout template structure in initial state
- Modify `addExercise()` default values
- Update edit modal UI
- Update logging view UI

**Changing PR tracking:**
- Modify `mainLiftsForPRs` object to add/remove exercises
- Update detection logic in statistics calculation

**Adding new icons to IconPicker:**
1. Create new inline SVG component following existing pattern
2. Add to `availableIcons` array in IconPicker component
3. Add emoji-to-icon mapping in WorkoutIcon component
4. Icon will automatically appear in Add Day and Edit modals

**Modifying markdown parser:**
- Edit `parseMarkdownWorkout()` function to support new formats
- Add new regex patterns for exercise/sets/reps detection
- Update default values for missing fields

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

## Data Import/Export

### CSV Export
Users can export workout history to CSV via Download button in History view. CSV format:
```
Date,Workout,Exercise,Set,Weight,Reps
```
Dates use Croatian locale (`hr-HR`).

### Markdown Import
Users can import workout routines via "Upload MD" button on Home screen:
1. Click "Upload MD" button next to "Add Day"
2. Select a .md file with workout routines
3. Choose "Replace All" (clear existing) or "Add to Existing"
4. Workouts are parsed and saved to IndexedDB

See **Markdown Upload Feature** section above for supported formats and examples.

## Styling Approach

Uses Tailwind utility classes directly in JSX:
- Dark mode: Conditional class names based on `darkMode` state
- Theme colors: Gray-scale with green accents for success states
- Mobile-first: Responsive grid layouts, fixed bottom navigation
- Animations: Gradient backgrounds, hover effects, shadow transitions
