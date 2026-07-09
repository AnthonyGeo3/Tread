# CLAUDE.md — Tread (trainer tracker)

Single-user PWA tracking miles on Anthony's Nike Pegasus 40. The shoe stands toe-down
and fills with liquid as miles accumulate toward a replacement target; the liquid
stays world-level and sloshes with device tilt, the shoe self-rights like a gimballed
glass. The real product goal is **habit retention**: Anthony is mid-Couch-to-5K and
this app exists to make running feel satisfying so he doesn't drop the hobby.
Judge every feature against that, not against "fitness app" convention.

**Current build: B11 · sw.js CACHE `tread-v11` · deployed on GitHub Pages (user AnthonyGeo3)**

## Hard rules (house style — do not violate)

- One `index.html`: all CSS and JS inline, vanilla, no frameworks, no build tools, no deps, no CDNs (offline PWA).
- Deployable files are exactly: `index.html`, `sw.js`, `manifest.webmanifest`, `icon-192.png`, `icon-512.png`. Nothing else ships.
- Every release bumps **both** the footer label `TREAD B<n>` in index.html **and** `CACHE = 'tread-v<n>'` in sw.js. They move together, always.
- Single profile, `localStorage` only. No Firebase, no accounts, no sync. (Amy doesn't use this one.)
- All storage access stays wrapped in try/catch — the app must still run session-only if storage is unavailable.
- Preserve `prefers-reduced-motion` handling and `:focus-visible` styles.
- Relative paths everywhere (`./sw.js` etc.) — it's served from a repo subpath.

## Architecture map (index.html)

- `:root` CSS tokens: concrete-studio palette (`--bg1/--bg2`), ink, volt accent. Light background is deliberate — it mirrors his product photo; don't dark-mode it.
- Inline SVG `#shoeSvg`, `viewBox="0 0 380 620"`, portrait, toe down, sole facing left.
  JS depends on these ids — **never rename**: `shoeClip`, `shadow`, `shoeG`, `liquidG`,
  `liquidPath`, `meniscus`, `bubbles`, `sTop`, `sBot`.
- `<script>` is one IIFE, sectioned by comments: storage → wear colour → moments →
  actions → dialog → physics → geometry constants → frame loop → PWA registration.

## Data schema

`localStorage['tread']` = `{ target: number, metric?: 'dist'|'pace'|'time', entries: [{ id: string, m: number /*miles*/, d: number /*epoch ms*/, t?: number /*duration in seconds, optional*/ }] }`
The run log renders sorted by `d` descending — dates are user-editable, so never rely on
array insertion order for display.
Everything else (totals, %, weekly count, longest, milestones) is derived — keep it
that way. Any schema change needs defensive defaults on load (see top of script).

## Physics constants (frame loop)

- Liquid level is **volume-conserving**, not height-based: `VCOLS` (baked array of
  `[x, yTop, yBottom]` strips of the shoe interior, 8px apart) gives the vessel's shape;
  `solveCut(frac, tan)` bisects for the surface line whose wet area = fill fraction at
  the current angle. This is why the puddle can't vanish or flood when tilted. `TOPY`/`BOTY`
  remain only for bubble spawn positions. Wave amplitude is damped near empty/full
  (`effAmp`) so slosh never pops above the toe or the collar.
- `CX=190` liquid pivot x; `PIVY=250` shoe gimbal pivot; surface sampled `L=20`→`R=360`, floor `640`.
- Springs (semi-implicit Euler, `spring(state,target,w,zeta,dt)`):
  shoe `w=9 ζ=0.45` (speedy self-right, slight overshoot) · liquid `w=6.5 ζ=0.30` (sloshy)
  · fill `w=3.6 ζ=0.95` (smooth rise on add). Wave amplitude capped at 15.
- Liquid target angle counters device tilt **and** current shoe angle so it reads world-level.

## Artwork

Portrait paths were baked from a landscape master via the mapping `(x,y) → (391−y, x)`
(90° clockwise). If editing the drawing, edit the baked coordinates in place — small
tweaks are fine by hand. Icons are renders of the same paths at 55% fill; if the shoe
changes materially, regenerate both PNGs to match. **If the silhouette changes at all,
`VCOLS` in index.html must be re-derived** (rasterize the silhouette at 380×620, take
vertical strips every 8px, record `[x, firstY+1, lastY-1]`) or liquid volume will be wrong.

## Testing gotchas

- Claude.ai / artifact preview runs in a sandboxed iframe: **motion sensors are blocked
  and the service worker won't register there.** Drag-to-slosh still works. Real tilt
  must be tested from the GitHub Pages URL over HTTPS.
- Android Chrome: tilt works with no prompt. iOS 13+: the "Enable tilt" button appears
  and must be tapped each session (`DeviceOrientationEvent.requestPermission`).
- After deploying, the old SW may serve one stale load; the CACHE bump + skipWaiting
  means the next refresh gets the new build.

## Release ritual

Edit → bump `TREAD B<n>` + `tread-v<n>` → commit → push via GitHub Desktop → open the
Pages URL, refresh twice, confirm footer shows the new build number.

## Shipped in B3–B8 (so you don't rebuild them)

Milestone toasts on crossing 5/10/25/35/50/75/100/150/200/250/300/350/400 total miles;
first-run and first single-run ≥3.1 mi ("full 5K") moments; new-longest-run toast;
"this week: N runs · longest X mi" line; gentle welcome-back toast when opening after
a ≥5-day gap. One toast per event, max splash on celebration.

B4: optional time-to-complete on each run (add-row field between miles and the button;
`parseTime` treats `:`, `.` and space as interchangeable separators — `25:58`, `25.58`,
`25 58` and microwave-style `2558` all mean 25:58 — plus plain minutes and `h:mm:ss`.
There is deliberately **no** decimal-minutes reading: on Android keypads the dot is the
only separator available, so `25.58` must mean 25:58, never 25.58 minutes). Tap a log row's **date** to edit it via a native date picker (capped at
today, keeps the original time-of-day); tap its **time** (or the `–:–` placeholder) to
add, change, or clear a duration through the shared dialog.

B5: per-run pace (min/mi) shown in small text under the time in each log row —
always computed as `t / m`, never entered, no averages or cross-run comparisons.

B6: time inputs use `inputmode="decimal"` so the keypad reliably offers a full stop.

B7: volume-conserving liquid. The old model held a fixed-height surface pivoting on
x=190, so at low fill a tilt could lift the plane off the toe puddle entirely (liquid
vanished) or plunge it across the toe (flooded). Now fill % maps to **volume**, so the
level follows the vessel shape — the narrow toe fills fast, the wide midfoot slower.

B8: trend line chart between the add row and the run log (inline SVG, no libs). Plots
runs only — never zero-filled rest days — connected in date order, last 30 runs, real
time-scaled x-axis. Tapping anywhere on the chart cycles distance → pace → time
(choice persisted as `metric`). **Pace axis is inverted (faster at the top)** so an
upward line means progress in all three modes — keep it that way. Hidden until 2 runs
exist; pace/time modes show a gentle empty note if no durations are logged.

B9: "nice numbers" y-axis on the trend chart (`niceAxis`). Replaced the old
min/midpoint/max ticks (which read raw run values like 1.6/1.9/2.2) with rounded bounds
and a rounded step. Metric-aware: distance snaps to decimal-mile steps (0.1/0.2/0.5/1…),
pace and time snap to time-friendly steps (15s/30s/1m/2m/5m…) so ticks land on whole
minutes. Aims for ~3 intervals; dist labels use a decimal count derived from the step.

B10: run log is now a scroll container (`#log` max-height 250px, ~5 rows, momentum +
contained overscroll) so it stops growing the page. A sort pill (`#sortBtn`, styled like
the chart's `metricBtn`) in a new `.histHead` cycles the order: Newest (date desc,
default) → Longest (distance) → Longest time → Fastest (pace). Untimed runs sink to the
bottom for the time/pace orders. Sort is **session-only** — not persisted, resets to date
order on load — so opening the app is always predictable. Button hidden until 2+ runs.

B11: weekly recap email (`Weekly recap` footer link → settings dialog). No backend exists
for Tread, so this is deliberately **client-only**: on app open, if 7+ days have passed
since `data.recapLast`, the app composes a motivational summary of the trailing week
(`weekStats`/`recapCopy` — run count, total miles, longest run, non-comparative, warm even
at zero runs per the anti-goals) and POSTs it straight from the browser to EmailJS's REST
endpoint (`api.emailjs.com`, CORS-enabled, public-key auth — no server secret, so it fits
the static-PWA constraint). This fires on next app open after the week elapses, **not** on
a fixed calendar day — a true cron-scheduled send would require a server holding a copy of
the run log, which was ruled out as a bigger philosophy change than this feature warranted.
Requires the user to create their own free EmailJS service + a template containing a single
`{{message}}` variable, and to set that template's "To Email" field to their own address
(the recipient is configured there, not stored in Tread) — the three resulting IDs
(public key, service ID, template ID) are pasted into the settings dialog and live in
`data.ejKey/ejService/ejTemplate`, save-triggers an immediate test send. Silent no-op if
unconfigured or offline; only surfaces a toast on the deliberate test send or a successful
auto-send, never on routine auto-send failure.

## Backlog — prioritised for the real goal (keep running through and past C25K)

1. **Export / import backup (S)** — buttons in the footer: copy `localStorage['tread']`
   JSON to clipboard, paste to restore. Losing months of logged runs is the fastest way
   to kill both the habit and trust in the app. Do this before anything shinier.
2. **C25K programme card (S–M)** — setting for current week/run (he's mid-programme, so
   an offset, not a start date). Show "Week 6 · run 2 of 3", tick runs off as they're
   logged, big celebration at W9R3 graduation. Completion structure is the strongest
   novice motivator there is.
3. **Weekly rhythm strip (S)** — last 8 weeks as tiny bars (runs per week) above the run
   log, plain inline SVG. At C25K stage consistency matters more than distance; this
   makes "don't break the rhythm" visible without shaming a single missed week.
4. **Manifest shortcuts (S)** — `shortcuts` in the manifest, e.g. "Log a 5K" →
   `./?add=3.1`; handle the query param once on load then strip it. One-tap logging
   from a long-press on the home-screen icon removes the last bit of friction.
5. **Feel tag per run (S)** — optional 😄/🙂/😮‍💨 tap after adding; store as `f` on the
   entry. Later surface "8 of your last 10 runs felt good" — noticing that runs feel
   better than expected is a documented driver of intrinsic motivation.
6. **Weekly recap toast (S–M)** — first open each Monday: "Last week: 3 runs, 7.4 mi —
   your biggest week yet" (or a kind version when it wasn't).
7. **Shoe shelf / multiple pairs (M)** — on "New pair", archive `{name, total, from, to}`
   and show retired pairs as small greyed shoes. "Third pair" is identity: he's not
   someone doing C25K anymore, he's a runner who wears out shoes.
8. **More journey milestones (S)** — extend `MARKS` with Wrexham-anchored distances
   (Chester ~12, Liverpool/Anfield ~35 already in, length of Wales ~160, London ~200).
9. **Pace — shipped in B5.** Per-run only, computed, displayed under the time. Keep it
   that way: no averages, trends, or comparisons unless explicitly requested — a slow
   run must never look like a failed run.
10. **Push reminders via the existing Cloudflare Worker (L)** — real Web Push on his
    Pixel for planned run days. Note: installed PWAs cannot reliably schedule *local*
    notifications; push is the only honest route. Only do this if the on-open nudges
    prove insufficient — reminders he asked for feel supportive, ones he didn't feel like nagging.
11. **Share card (M)** — canvas-render the shoe at current fill + stats to a PNG via
    the Web Share API, so finishing a milestone has something to show Amy.
12. **Strava / Health Connect import (L)** — only if manual logging ever becomes the
    reason runs go unlogged. For now the 5-second manual log *is* part of the reward
    loop; don't automate the ritual away.

### Anti-goals

No streak-shaming or guilt copy, ever — a missed week gets warmth, not red. No pace
leaderboards or comparisons. No ads, gamification currencies, or badge spam. No
Firebase/accounts. Nothing that makes opening the app feel like admin.
