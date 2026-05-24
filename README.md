# 💪 Workout Pro

A modern, feature-rich Progressive Web App (PWA) for tracking workouts with offline support, statistics, and progress monitoring.

![License](https://img.shields.io/badge/license-MIT-blue.svg)
![PWA](https://img.shields.io/badge/PWA-enabled-brightgreen.svg)

## ✨ Features

- 📊 **Detailed Statistics** - Track volume, reps, personal records, and progress over time
- 💪 **Personal Records** - Monitor strength gains on main compound lifts (competition-style variants only — Leg Press, RDL, Dumbbell variants are excluded from main-lift totals to keep Strength Standards accurate)
- 🧠 **Proactive Recommendations** - Multi-workout coaching: plateau detection, deload cues, exercise-swap suggestions, "+1 rep last time, push for one more" prompts, and bodyweight-aware progress (same lift at lower bodyweight is flagged as a strength gain)
- 📏 **Per-Exercise Units** - Reps, seconds, or meters per exercise (auto-detected by name; override per template). Time/distance work is counted separately from weighted volume
- ⚖️ **Body Weight & Strength Standards** - Gender-specific Symmetric Strength bands (Untrained → Elite) computed from estimated 1RM relative to current bodyweight
- 📱 **Progressive Web App** - Install on Android/iOS as a native app
- 💾 **Offline Support** - Works completely offline with IndexedDB storage
- 🌓 **Dark/Light Mode** - Comfortable viewing in any lighting condition
- 📈 **Progress Tracking** - Weekly and monthly analytics with trend comparisons
- 📥 **CSV Export & JSON Backup** - Export history to CSV; full backup/restore via JSON (workouts, history, bodyweight, gender)
- ⚡ **Progressive Overload** - Pre-fills weights from previous workouts and adjusts suggestions by your difficulty rating
- 🔒 **Privacy First** - All data stays on your device, no backend required

## 🚀 Quick Start

### Option 1: Use the Live App (Recommended)

1. Visit the deployed app: [Your Netlify URL]
2. On Android: Tap menu (⋮) → "Install app" or "Add to Home screen"
3. On iOS: Tap share icon → "Add to Home Screen"

### Option 2: Run Locally

```bash
# Clone the repository
git clone https://github.com/niksaderek/workout-app.git
cd workout-app

# Open index.html in your browser
# Or use a local server:
python -m http.server 8000
# Then visit http://localhost:8000
```

## 📱 Installation as PWA

### Android (Chrome/Edge)
1. Open the app URL in Chrome
2. Tap the menu (⋮) in the top right
3. Select "Install app" or "Add to Home screen"
4. Tap "Install"
5. The app will appear on your home screen

### iOS (Safari)
1. Open the app URL in Safari
2. Tap the share icon (box with arrow)
3. Scroll down and tap "Add to Home Screen"
4. Tap "Add"

## 🏋️ Usage

### Home Screen
- View your weekly stats (workouts, volume, reps, core sets)
- Access your workout program
- Start a workout with one tap
- Edit or delete workout days
- Add custom workout days

### During Workout
- Track sets, reps, and weights in real-time
- Weights automatically pre-filled from last session
- Mark sets as completed
- Core exercises (plank, hanging leg raises) have weight input disabled
- Decimal weight support (use comma or dot: 22,5kg or 22.5kg)

### Statistics
- **Weekly Progress**: Volume, reps, sets with week-over-week comparison
- **Monthly Overview**: Track monthly trends and improvements
- **Personal Records**: Max weights for main lifts (deadlift, squat, bench, overhead press, lat pulldown)
- **All-Time Stats**: Total workouts, volume, average workouts per week

### History
- View all completed workouts
- Delete individual workout entries
- Export all data to CSV

## 🛠️ Tech Stack

- **React 18** - UI framework (loaded via CDN)
- **IndexedDB** - Local database for workout storage
- **Tailwind CSS** - Utility-first styling (via CDN)
- **Lucide React** - Icon library
- **Service Worker** - Offline functionality
- **Web App Manifest** - PWA capabilities

## 📂 Project Structure

```
workout-app/
├── index.html              # Main HTML file with embedded React app
├── manifest.json           # PWA manifest
├── sw.js                   # Service worker for offline support
├── netlify.toml           # Netlify deployment configuration
├── workout-tracker.tsx     # Original React component (reference)
├── CLAUDE.md              # Development documentation
└── README.md              # This file
```

## 🎨 Customization

### Adding New Workout Days

1. Tap "Add Day" on the home screen
2. Edit the workout name and emoji
3. Add/remove exercises
4. Set custom sets and reps

### Modifying Default Workouts

Edit the `workouts` state in `index.html` (lines 114-172) to change the default workout program.

### Changing Theme Colors

Modify the `theme_color` and `background_color` in `manifest.json` and update the CSS gradient classes in the component.

## 📊 Data Storage

All data is stored locally using IndexedDB:
- **Database Name**: `WorkoutTrackerDB`
- **Stores**:
  - `workouts` - Workout day templates
  - `history` - Completed workout logs

**Privacy**: Your data never leaves your device. No backend, no tracking, no analytics.

## 🔄 Deployment

### Netlify (Automatic)

1. Fork this repository
2. Connect your GitHub repo to Netlify
3. Netlify auto-detects settings from `netlify.toml`
4. Deploy!

Every push to `master` automatically deploys.

### Other Platforms

This is a static site - works on any static hosting:
- Vercel
- GitHub Pages
- Cloudflare Pages
- Firebase Hosting

Just deploy the root directory.

## 📥 Data Export

Export your workout history to CSV:
1. Go to History tab
2. Tap the download icon
3. CSV file downloads with format: `Date,Workout,Exercise,Set,Weight,Reps`

Use exported data in Excel, Google Sheets, or data analysis tools.

## 🤝 Contributing

Contributions are welcome! Please feel free to submit a Pull Request.

1. Fork the repository
2. Create your feature branch (`git checkout -b feature/AmazingFeature`)
3. Commit your changes (`git commit -m 'Add some AmazingFeature'`)
4. Push to the branch (`git push origin feature/AmazingFeature`)
5. Open a Pull Request

## 📝 License

This project is licensed under the MIT License - see below for details:

```
MIT License

Copyright (c) 2025 Nikša Derek

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```

## 🐛 Known Issues

- Service worker may need manual refresh on first install
- Icons (192x192 and 512x512) not included - add your own to `/icon-192.png` and `/icon-512.png`

## 🚧 Roadmap

- [ ] Add workout templates library
- [ ] Exercise instruction videos/GIFs
- [ ] Rest timer between sets
- [ ] Workout calendar view
- [ ] Social sharing of PRs
- [ ] Import CSV data
- [ ] Workout streaks and achievements

## 💬 Support

If you encounter any issues or have questions:
- Open an [Issue](https://github.com/niksaderek/workout-app/issues)
- Check the [CLAUDE.md](CLAUDE.md) file for technical documentation

## 🙏 Acknowledgments

- Built with React and modern web technologies
- Icons by [Lucide](https://lucide.dev/)
- Styling with [Tailwind CSS](https://tailwindcss.com/)

---

**Made with 💪 for tracking gains**

[⬆ back to top](#-workout-pro)
