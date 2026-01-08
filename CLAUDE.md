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

**Core Functions:**

1. **Workout Logging** (lines 420-497):
   - `startWorkout()` - Initializes workout session with last weights pre-filled
   - `getLastWeight()` - Retrieves previous weight for progressive overload
   - `updateSet()` / `toggleSetComplete()` - Track sets during workout
   - `finishWorkout()` - Saves completed workout to IndexedDB

2. **Statistics Calculation** (lines 640-870):
   - `getWeekStats()` - Weekly metrics (count, volume, reps, core sets)
   - `getDetailedStats()` - Comprehensive analytics (weekly/monthly comparisons, PRs)
   - Volume calculation: `weight × reps` (excludes core exercises)
   - Personal Records tracking for main lifts (deadlift, squat, bench, OHP, lat pulldown)

3. **Template Management**:
   - `startEditWorkout()` / `saveEditedWorkout()` - Modify workout day templates
   - `addExercise()` / `deleteExercise()` - Exercise CRUD
   - `addNewWorkoutDay()` / `deleteWorkoutDay()` - Workout day CRUD
   - `reorderExercise()` - Reorder exercises via drag-and-drop

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

### UI Views

- **Home** - Dashboard with stats cards, workout day cards, "Upload MD" button
- **Stats** - Detailed analytics, PRs, weekly/monthly trends, progress timeline with interactive charts
- **Logging** - Active workout session with set-by-set tracking and "Add Set" button
- **History** - Past workout logs with edit, delete, and CSV export functionality
- **Edit** - Workout template editor with drag-and-drop exercise reordering and ExerciseInput autocomplete
- **Edit-History** - Edit past workout data (weight/reps) without affecting templates

### Modals

- **Add Day Modal** - Create new workout day with IconPicker
- **Upload Modal** - Markdown upload confirmation (replace vs add to existing)
- **Timeline Detail Modal** - Shows workout details for selected week/month on timeline
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
