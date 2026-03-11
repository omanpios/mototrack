# MotoTrack — Project Context
**Handoff document for Claude CLI · Generated March 11 2026**

---

## What this project is

A **mobile-first motorcycle maintenance tracker** built as a self-contained PWA. Single `index.html` file — no server, no build pipeline, no backend. Designed for riders: garages, outdoors, trails, sunlight, low-light.

**Live features:**
- Track engine hours OR odometer (km/mi) per bike
- Multiple bikes, switchable from header
- Service schedule with overdue / due-soon / ok states
- Session logging (start reading → end reading → notes → cost)
- Pre-ride checklist (per-bike, customisable categories + items)
- History tab with session details
- Stats tab (totals, averages, cost breakdown)
- Riding mode toggle: Normal / Aggressive (tightens service intervals)
- Dark theme, Lucide icons (inline SVG, no CDN), Rajdhani + Share Tech Mono fonts
- localStorage persistence (data survives restarts)
- PWA: installable, fully offline

**Not yet built:**
- Google Drive sync (was in DEPLOY.md but never implemented — docs have been corrected)

---

## File inventory

| File | Purpose |
|---|---|
| `index.html` | Complete app — React + all code bundled inline (312KB) |
| `moto-tracker-pwa.jsx` | Source JSX (edit this, then re-bundle) |
| `sw.js` | Service worker — cache-first, offline-first |
| `manifest.json` | PWA installability metadata |
| `README.md` | Project overview |
| `DEPLOY.md` | Netlify / GitHub Pages deploy instructions |

---

## Tech stack

- **React 19** (bundled inline via esbuild — zero CDN dependencies)
- **CSS custom properties** — full design token system
- **localStorage** — all persistence, key `mototrack_v3`
- **No build step for editing** — edit `moto-tracker-pwa.jsx`, re-bundle with esbuild
- **esbuild** used for bundling (available at `/home/claude/.npm-global/lib/node_modules/tsx/node_modules/esbuild/bin/esbuild`)

### Re-bundling after edits
```bash
ESBUILD=/home/claude/.npm-global/lib/node_modules/tsx/node_modules/esbuild/bin/esbuild
NODE_PATH=/home/claude/.npm-global/lib/node_modules

NODE_PATH=$NODE_PATH $ESBUILD moto-tracker-pwa.jsx \
  --bundle --minify --jsx=automatic \
  --platform=browser --target=es2017 \
  --outfile=bundle.js

# Then inline bundle.js into index.html replacing the <script> block
```

---

## Design system

### Color tokens (`--variable`)
```css
/* Surfaces */
--bg:  #080809   /* base background — near-black with blue-purple tint */
--s1:  #111116   /* header, nav, cards */
--s2:  #191920   /* inputs, expanded card rows */
--s3:  #222229   /* notes bg, hover states */
--s4:  #2a2a32   /* deepest surface */

/* Borders */
--bd:  #2c2c3a   /* decorative card borders */
--bd2: #383849   /* stronger borders */

/* Text */
--txt:  #eeeef5  /* primary — 17:1 contrast, AAA */
--txt2: #8888a0  /* secondary — 5.5:1, AA */
--txt3: #7b7b96  /* labels/hints/icons — 4.88:1 on bg, AA */
                 /* RAISED from #4a4a62 (was 2.3:1, FAIL) in accessibility audit */

/* Accent */
--red:   #c82000  /* primary action + overdue state (dual role — known issue) */
--red2:  #ff3300  /* active nav, alert text */
--red3:  #7a1200  /* dark red for borders */
--amb:   #d97706  /* due-soon / warning */
--grn:   #16a34a  /* ok / success */
--grn-d: #052e16  /* ok badge background */
--blu:   #1d4ed8  /* info / blue actions */
--blu-d: #1e3a5f  /* blue badge background */

/* Fonts */
--fd: 'Rajdhani', system-ui, sans-serif       /* display / headings */
--fm: 'Share Tech Mono', 'Courier New', mono  /* data / labels / mono */
--fb: 'Barlow', system-ui, sans-serif         /* body */
```

Fonts load from Google Fonts when online, fall back to system fonts offline (non-blocking `<link media="print" onload>`).

### WCAG compliance status (post-audit)
- `--txt` on all surfaces: **AAA** ✅
- `--txt2` on all surfaces: **AA** ✅
- `--txt3` on bg/s1: **AA** ✅ | on s2/s3: **AA-Large** ✅
- Status colors (overdue/soon/ok): **AA–AAA** ✅
- All buttons: **AA** ✅
- Input default borders: best-effort (focus state hits 3:1 ✅)

---

## Data model

All state in `localStorage` key `mototrack_v3`. Shape:

```js
{
  bikes: [
    {
      id: string,
      name: string,
      make: string,
      model: string,
      year: number,
      color: string,           // hex, used for bike dot indicator
      trackingUnit: "hours"|"km"|"mi",
      unitConfirmed: boolean,  // gates session logging until confirmed
      initialReading: number,
      ridingMode: "normal"|"aggressive",
      createdAt: ISO string,
    }
  ],
  sessions: [
    {
      id: string,
      bikeId: string,
      startReading: number,
      durationReading: number, // the added amount
      notes: string,
      cost: number,
      createdAt: ISO string,
    }
  ],
  tasks: [
    {
      id: string,
      bikeId: string,
      name: string,
      category: string,        // "Engine"|"Air"|"Drivetrain"|"Cooling"|"Brakes"|"Electrical"|"Suspension"|"Other"
      icon: string,            // emoji key → maps to Lucide icon
      intervalNormal: number,
      intervalAggressive: number,
      lastAt: number|null,     // reading when last serviced
      completedAt: ISO string|null,
      notes: string,
      cost: number,
    }
  ],
  odometer_snapshots: [        // for distance-unit bikes (km/mi)
    {
      id: string,
      bikeId: string,
      reading: number,
      source: "session"|"manual",
      createdAt: ISO string,
    }
  ],
  checklistCategories: [
    {
      id: string,
      bikeId: string,
      name: string,
      icon: string,            // emoji key → maps to Lucide icon via CL_ICON_MAP
      order: number,
      createdAt: ISO string,
    }
  ],
  checklistItems: [
    {
      id: string,
      bikeId: string,
      categoryId: string,
      name: string,
      order: number,
      enabled: boolean,
      createdAt: ISO string,
    }
  ],
  preRideChecks: {
    [bikeId]: { [itemId]: boolean }  // check state, resets on new session
  }
}
```

### Service interval logic
```
totalReading = initialReading + sum(session.durationReading for bike's sessions)
nextDue = task.lastAt + (ridingMode === "aggressive" ? task.intervalAggressive : task.intervalNormal)
remaining = nextDue - totalReading
status = remaining <= 0 ? "overdue" : remaining <= dueSoonThreshold ? "due-soon" : "ok"
```

`dueSoonThreshold`: 2h (hours) / 200km / 120mi

---

## Icon system

Inline SVG — no Lucide package, no CDN. All paths stored in `ICON_PATHS` object.

```js
// Renderer component
function Ic({ icon, size = 16, color, style, className })
// Splits path string on " M" to support multi-path icons

// Service task icons (emoji → icon name)
const SERVICE_ICON_MAP = { "🛢️": "Droplets", "🔩": "Bolt", ... }

// Checklist category icons
const CL_ICON_MAP = { "🏍": "Bike", "🛵": "Motorbike", "🔧": "Wrench", ... }
```

**Recently added icon:** `Motorbike` (Lucide v0.545.0) — replaces second `Bike` slot for `🛵` emoji in the Edit Category modal. Path data:
```
"M5 19a3 3 0 1 0 0-6 3 3 0 0 0 0 6z M19 19a3 3 0 1 0 0-6 3 3 0 0 0 0 6z M5 16l2-5h6l3 4 M14 11l-3-6H9 M16 8h2a2 2 0 0 1 2 2v3h-5 M14 5h3"
```

---

## Key components / tabs

| Tab | Component | Key functionality |
|---|---|---|
| DASH | `DashTab` | Hourmeter/odometer card, overdue alert, overview grid, priority task list, recent sessions |
| SERVICE | `TrackTab` | Full service schedule, expand/collapse per task, complete task flow |
| MANAGE | `ManageTab` | Bike CRUD, task CRUD, service templates |
| HISTORY | `HistoryTab` | Session list, session detail modal, stats aggregation |
| CHECK | `CheckTab` | Pre-ride checklist, per-bike categories and items, manage mode |

**Modals:** All bottom-sheet style (`position:fixed`, `align-items:flex-end`), max 92vh, blur overlay.

---

## Responsiveness fixes applied (this session)

1. `html`, `body`, `#root`, `.app` all set to `width:100%` + `overflow-x:hidden` — fixes the gap on right side of screen in Chrome mobile
2. `.app` uses `max-width:480px` for desktop centering, `width:100%` fills phone
3. `.hcard-actions .btn { flex:1 }` — LOG SESSION + SERVICE buttons fill card width equally
4. `.tname` — `white-space:nowrap; overflow:hidden; text-overflow:ellipsis` — long task names clip
5. `.tstatus { min-width:82px; flex-shrink:0 }` — status badge column never gets crushed
6. `.sbox { min-height:64px }` — overview grid cells uniform height
7. `.nav { padding-bottom:env(safe-area-inset-bottom,0px) }` — notch/gesture bar support
8. `.nitem { min-height:48px }` — minimum touch target
9. `.scroll { padding-bottom:calc(72px + env(safe-area-inset-bottom,0px)) }` — content not hidden behind nav
10. `.nicon { color:var(--txt3) }` + `.nitem.active .nicon { color:var(--red2) }` — **nav icons were invisible** (SVG stroke inheriting from nothing), fixed by explicit color

---

## Offline / PWA

The app was claiming to be offline-first but wasn't, because:
- React, ReactDOM, Babel were loaded from `unpkg.com` on every boot
- The service worker explicitly skipped `unpkg.com` requests
- Without those files the app was a blank page offline

**Fix applied:** Used esbuild to pre-bundle React + all app code into a single `index.html` (312KB). Zero external runtime dependencies. Service worker now caches only `index.html` + `manifest.json` on install → full offline from first visit.

Service worker version: `mototrack-v2`

---

## GitHub repo

Two commits:
1. `84866d8` — Initial commit — MotoTrack PWA
2. `04c42e2` — Bundle React inline — full offline support, zero CDN dependencies

To push: download `mototrack-repo.zip`, unzip, then:
```bash
git remote add origin https://github.com/YOUR_USERNAME/mototrack.git
git branch -M main
git push -u origin main
```

For GitHub Pages: Settings → Pages → Deploy from branch → main → / (root)

---

## Known issues / future work

- **Red dual-role conflict:** `--red` is used for both primary action buttons AND the overdue/danger semantic state. Cognitively ambiguous, especially for safety-critical outdoor use. Consider shifting overdue to a distinct red-orange.
- **Google Drive sync:** Fully designed, never implemented. Would need OAuth2 client setup at console.cloud.google.com, `appDataFolder` scope, and a `DriveSettingsModal` component.
- **Progress bar accessibility:** `.pbar` uses only color to show service interval progress. `title` attribute added but no visible numeric label for color-blind users.
- **Input default borders:** `#4a4a60` on `--s2` is ~2.35:1 — below WCAG 1.4.11 3:1 for interactive component boundaries. Focus state (red border) hits 3:1. Acceptable compromise for the dark aesthetic.
- **Unit lock:** `trackingUnit` is immutable after bike creation by design. No migration path if user picks wrong unit.
- **Checklist reset:** `preRideChecks` should reset on new session log — verify this is implemented.
