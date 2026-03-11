# MotoTrack — Deploy Guide

## Files in this package
- index.html     — The full app (self-contained)
- manifest.json  — Makes it installable as a PWA
- sw.js          — Service worker (offline support)
- DEPLOY.md      — This guide

---

## Option A: Netlify (easiest — 2 minutes)

1. Go to https://netlify.com → sign up free
2. Click "Add new site" → "Deploy manually"
3. Drag the entire mototrack-pwa folder onto the drop zone
4. Done! You get a URL like: https://mototrack-abc123.netlify.app

---

## Option B: GitHub Pages (free, permanent URL)

1. Go to https://github.com → create account (free)
2. Click "New repository" → name it "mototrack" → Public → Create
3. Upload all 4 files (index.html, manifest.json, sw.js, DEPLOY.md)
4. Go to Settings → Pages → Source: "Deploy from branch" → main → / (root) → Save
5. Your URL: https://YOUR-USERNAME.github.io/mototrack

---

## Install on your phone (after deploying)

1. Open Chrome on your Android phone
2. Go to your app URL
3. Chrome shows "Add to Home screen" banner — tap it
   OR tap the 3-dot menu → "Add to Home screen"
4. Tap "Add" — done! It's on your home screen like a real app

The app opens fullscreen with no browser bar.

---
