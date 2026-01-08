# Workout Pro - Personal Workout Tracker

A modern, feature-rich workout tracking PWA with statistics, progress monitoring, and offline support.

## Features

- 📊 **Detailed Statistics** - Track volume, reps, PRs, and progress over time
- 💪 **Personal Records** - Monitor your strength gains on main compound lifts
- 🌓 **Dark/Light Mode** - Comfortable viewing in any lighting
- 📱 **PWA Support** - Install on Android/iOS as a native app
- 💾 **IndexedDB Storage** - Your data stays on your device, works offline
- 📈 **Progress Tracking** - Weekly and monthly analytics
- 📥 **CSV Export** - Export your data for external analysis

## Tech Stack

- React 18
- IndexedDB for local storage
- Lucide React icons
- Responsive design for mobile-first experience

## Deployment

### Deploy to Netlify

1. Fork or clone this repository
2. Connect your GitHub repo to Netlify
3. Netlify will auto-detect the settings from `netlify.toml`
4. Deploy!

### Install as PWA on Android

1. Open the deployed URL in Chrome/Edge
2. Tap the menu (⋮) in the top right
3. Select "Add to Home screen" or "Install app"
4. The app will appear on your home screen like a native app

## Usage

- **Home**: View your workout program and quick stats
- **Stats**: Detailed analytics and personal records
- **History**: Review past workouts
- **Logging**: Track sets, reps, and weights in real-time

## Data Storage

All data is stored locally in IndexedDB:
- Works offline
- No internet required after initial load
- Your data never leaves your device
- Export to CSV anytime for backup

## Development

This is a single-file HTML application with embedded React for easy deployment.

## License

Personal use only.

---

Built with ❤️ for tracking gains 💪