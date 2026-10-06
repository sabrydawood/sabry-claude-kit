---
name: launch-video-babel-engine
description: Est8Core marketing launch video is vanilla React 18 UMD + Babel-standalone-in-browser (no build step) — verify by transpiling with the exact CDN Babel version, not by running a bundler.
metadata:
  type: project
---

`apps/marketing/public/launch-video/` (Est8Core CRM repo) is a self-contained animated
launch video: React 18 UMD + `@babel/standalone` loaded via `<script type="text/babel" src="...">`
tags directly in `index.html` — no Next.js, no imports/exports/TS, no bundler. Every file
(`brand.jsx`, `screens.jsx`, `scenes-*.jsx`, `animations.jsx`) is plain global functions wired
together with `Object.assign(window, {...})` at the end of each file.

**Why:** there is no build step to catch syntax errors before the client sees them — a broken
file just fails silently in the browser console at runtime.

**How to apply:** when editing one of these files, don't just eyeball JSX validity or count
braces. Fetch the *exact* Babel version pinned in `index.html` (e.g.
`https://unpkg.com/@babel/standalone@7.29.0/babel.min.js`) into the scratchpad, `require()` it
in Node (it's UMD, attaches to `module.exports`), and call `Babel.transform(src, { presets:
['react'] })` against the edited file. Go further: stub the globals the file depends on
(`E8`, `Icon`, `Avatar`, `rise`, `sceneFade`, `countUp`, `toAr`, `useSprite`, `Easing`, `clamp`,
`React.createElement`) and actually invoke the exported scene functions — this catches
`ReferenceError`s from typos in global names that a pure parse check would miss. Clean up the
downloaded babel.min.js from scratchpad afterward.

Design language reference: `E8` tokens (navy/gold/neutrals) in `brand.jsx`, reusable product-UI
in `screens.jsx` (`AppFrame`, `LeadCard`, `Toast`, `KpiCard`), animation primitives in
`animations.jsx` (`rise`, `sceneFade`, `Easing`, `useSprite`). Existing scenes in `scenes-a.jsx`/
`scenes-b.jsx` (`SceneKanban`, `SceneDeal`, `SceneDashboard`) are the fidelity bar for any new
"hero mockup" work — big (1920×1080 canvas), detailed, RTL, staged `rise()` entrances + perpetual
`Math.sin(localTime * k)` float/tilt loops.
