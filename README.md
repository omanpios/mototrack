# MotoTrack

A mobile-first motorcycle maintenance tracker. Built as a PWA — installable on Android/iOS, works offline.

## Features
- Track engine hours or odometer across multiple bikes
- Service schedule with overdue/due-soon alerts
- Session logging with notes
- Pre-ride checklist (per bike)
- Cost tracking
- Dark theme optimised for garage and outdoor use

## Running locally

```bash
npx serve . --listen tcp://0.0.0.0:8080
```

Then open `http://localhost:8080` in Chrome and tap **Add to Home Screen**.

## Deploy

See [DEPLOY.md](DEPLOY.md) for Netlify and GitHub Pages instructions.

## Stack
- React 18 (via CDN, no build step)
- Vanilla CSS with CSS custom properties
- localStorage persistence
- PWA: manifest + service worker
