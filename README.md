# Warsash Sailing Club — Pursuit Race Countdown

A progressive web app (PWA) that runs the start sequence for pursuit races: a live clock, per-minute sound signals, countdowns to the next signal and next class start, and a lookup tool so any helm can find their start time.

Built as a single static page with no backend — once loaded it works entirely offline, which is exactly what you want on the water.

## Features

- **Live start sequence** — shows the last class away, the next class starting, a live clock with milliseconds, and a countdown to the next start
- **Sound signals** — for each class start: five short beeps (5-4-3-2-1) then a long beep on the start itself; minutes with no start are silent
- **Visual flash backup** — the countdown box flashes on every signal, so a muted or silent phone still shows the cue
- **Find Start tab** — pick a class from the dropdown to see its start time plus the starts either side of it
- **Screen wake lock** — stops the phone auto-locking mid-sequence (iOS suspends JS and audio when the screen locks)
- **Offline / installable** — service worker caches the whole app; add it to the home screen for a fullscreen, app-like launch
- **Editable schedule** — drop a `schedule.json` next to the app to override the built-in start list without touching the code

## Repo structure

```
pursuit-race-countdown/
├── index.html            The entire app (markup, styles, and logic)
├── manifest.json         PWA manifest (name, icons, standalone display)
├── sw.js                 Service worker (offline caching)
├── icon-192.png          App icon
├── icon-512.png          App icon
├── apple-touch-icon.png  iOS home screen icon
├── schedule.json         Race day start lists, keyed by date
└── README.md
```

## The schedule

`index.html` ships with a fallback start list baked in (`FALLBACK_STARTS`). To set race-day schedules, host a `schedule.json` alongside `index.html` — the app polls it every 30 seconds and on returning to the tab, so edits show up on devices mid-sequence without a reload.

### Multi-date format (recommended)

An object keyed by ISO date, so a whole series can live in one file:

```json
{
  "2026-07-08": [
    { "time": "18:30", "class": "RS FEVA XL", "py": 1248 },
    { "time": "18:32", "class": "ILCA 4",     "py": 1218 }
  ],
  "2026-07-15": [
    { "time": "18:30", "class": "RS FEVA XL", "py": 1248 }
  ]
}
```

The app picks which day to show automatically:

1. **Today's date**, if present — full countdowns and sound signals run
2. Otherwise the **next upcoming race day** — schedule and Find Start work, header shows "Next race day: …", and no signals fire
3. Otherwise the **most recent past race day** — shown as "Last race day: …"

Signals only ever sound on the race day itself, so a schedule uploaded early can't trigger beeps on the wrong evening.

### Legacy format

A flat array of entries is still accepted and is treated as applying to **today**:

```json
[
  { "time": "18:30", "class": "RS FEVA XL", "py": 1248 }
]
```

### Entry fields

- `time` — 24-hour `HH:MM` on the device's clock
- `class` — display name; classes sharing a time are grouped into one start
- `py` — Portsmouth Yardstick number (informational; not used in calculations)

If `schedule.json` is missing, unreachable, or invalid, the built-in list is used (dated today), so the app never shows an empty screen.

## Deployment

The app is plain static files — any web host works, with one hard requirement: **HTTPS**. Service workers (offline support) and the install prompt are disabled on plain HTTP.

Typical S3 + CloudFront deploy:

1. Upload the contents of `pursuit-race-countdown/` to a bucket (files at the root of the site, or all under the same prefix — the manifest, service worker, and icons must sit alongside `index.html`)
2. Front it with CloudFront for HTTPS
3. Upload `schedule.json` to the same location on race day

**When you change `index.html`, bump the `CACHE` version string in `sw.js`** (e.g. `pursuit-v1` → `pursuit-v2`). The service worker serves the cached copy first, so without a version bump, installed devices can keep running the old version for a while.

## Installing on a phone (iOS)

1. Open the site in **Safari** (not Chrome — iOS only allows PWA install from Safari)
2. Share → **Add to Home Screen**
3. Launch from the new icon — fullscreen, no browser chrome, loads from cache

### Getting sound to work on iPhone

- Turn **OFF the mute switch** and turn the **ringer volume up** with the side buttons — Safari plays web audio at ringer volume, not media volume
- Tap anywhere on the page once after opening (browser autoplay policy requires one interaction before audio is allowed — the app arms itself silently on the first tap)
- Use the **Test Sound** button to confirm before the sequence starts
- Keep the app in the foreground during the sequence — iOS suspends audio in backgrounded tabs. The wake lock keeps the screen on, but don't switch apps

## Local development

Service workers don't run from `file://`, so serve the folder:

```bash
python3 -m http.server 8000
```

Then open `http://localhost:8000` — `localhost` counts as a secure context, so the service worker registers and offline behaviour can be tested (DevTools → Application → Service Workers, then tick "Offline" and reload).

Note that because the service worker caches aggressively, during development it's easiest to use DevTools → Application → "Update on reload", or bump the cache version between changes.

## How the timing works

- The logic ticks every **250ms**; per-second signal moments are deduplicated by key so nothing double-fires, and worst-case signal lateness is one tick. The clock display runs on its own `requestAnimationFrame` loop so the milliseconds read smoothly
- Signals fire **only at class starts** — 5-4-3-2-1 shorts then a long beep on the start. Between starts (and outside the sequence) the app is silent
- The "Next Sound Signal" and "Starts in" countdowns are derived from the same target time with the same `Math.ceil`, so they always change at exactly the same instant
- Beeps are generated with the Web Audio API (square wave, 1200 Hz short / 900 Hz long) with a gain envelope to avoid clicks and to cut through outdoor noise
- All times use the **device's clock** — make sure the phone running the sequence agrees with official race time

## Limitations

- Start times all belong to their schedule date; a single sequence crossing midnight is not handled
- There's no clock synchronisation between devices — each phone trusts its own time
- iOS will not play the signals if the app is backgrounded or the screen is manually locked
