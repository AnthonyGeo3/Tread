# CLAUDE.md — Tread (trainer tracker)

Single-user PWA tracking miles on Anthony's Nike Pegasus 40. The shoe stands toe-down
and fills with liquid as miles accumulate toward a replacement target; the liquid
stays world-level and sloshes with device tilt, the shoe self-rights like a gimballed
glass. The real product goal is **habit retention**: Anthony is mid-Couch-to-5K and
this app exists to make running feel satisfying so he doesn't drop the hobby.
Judge every feature against that, not against "fitness app" convention.

**Current build: B15 · sw.js CACHE `tread-v15` · deployed on GitHub Pages (user AnthonyGeo3)**

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
Plus (B11–B14): `ejKey/ejService/ejTemplate` (EmailJS recap creds), `recapLast` (ms of last
recap send), `c25k` (`{done:0–27}` or `null` = programme off), `events` (`[{id, name, d /*epoch ms*/}]`),
`next` (epoch ms midnight of the single planned next run, or 0). Events also carry optional
`m` (target distance in miles, B15). All have defensive defaults at the top of the script;
`c25k`, `events` (incl. `m`), `next` are carried in the backup payload (v15).
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
auto-send, never on routine auto-send failure. **(Superseded by B12 — the email is now also
the backup; see below.)**

B12: the recap email doubles as the **backup** (folds backlog item #1 into the recap
instead of separate clipboard buttons). `recapBody` appends a machine-readable block to the
motivational copy — a fenced `backupPayload()` = `JSON.stringify({v, target, metric, entries})`
(email credentials are deliberately **excluded** — least data, and so restore can't clobber
the target device's own EmailJS config). Because it's baked into `{{message}}`, every recap
carries a full backup regardless of the user's template, and there's no separate `{{backup}}`
variable to forget. Freshness caveat: a backup is only as current as the last weekly send —
hitting **Save & test** in the recap dialog forces a fresh backup email on demand.
New `Restore` footer link → `ask()`-dialog with a `<textarea>` (`#dlgArea`, added to the
shared dialog alongside a 4th `onOk` arg). `parseBackup` is forgiving: it slices from the
first `{` to the last `}` so the user can paste the **whole email** (preamble, emoji and all),
`JSON.parse`s that, validates `entries` is an array, and sanitises each row (numeric `m>0`,
`d>0`, optional `t>0`, regenerates missing ids). Restore **replaces** target/metric/entries
(danger-styled, confirmed) but leaves `ejKey/ejService/ejTemplate/recapLast` untouched, so
recovering onto a configured device doesn't wipe its email setup.

B13: weekly rhythm strip (`#rhythmWrap`, inline SVG `#rhythm`, no libs) between the trend
chart and the run log. Shows the **last 8 Monday-weeks** as bars = runs per week
(`rhythmWeeks` reuses the same `(getDay()+6)%7` Monday boundary as the "this week" stat).
Each week draws a faint full-height ghost **track**; a volt `rBar` fills to
`count/maxCount` of the height (min 5px so a 1-run week is visible). Rest weeks show just
the empty track — **no red, no shame** (anti-goals); their hover `<title>` reads "rest
week", not "0 runs". The current week's track gets a muted outline (`.rNow`) to mark
in-progress. Header note is factual: "N of 8 weeks". Hidden until 2+ runs **and** at least
one run in the last 8 weeks — if the user has lapsed (nothing in 8 weeks) the strip stays
hidden rather than showing a wall of empty slots (the welcome-back toast covers that case).

B14: a `.plan` section between the readout and the add row holding three cards, each shown
only when relevant (a global `[hidden]{display:none!important}` reset was added so id-level
`display` rules can't defeat the `hidden` attribute — `#nextCard` is `display:flex`):
- **C25K programme card** (`#c25kCard`, opt-in via footer `C25K` link or its `Adjust` button
  → `openC25K`). Stores `data.c25k = {done:0–27}` (27 sessions = 9 weeks × 3). Displays the
  **next** run to do — `c25kPos(done)` → "Week X · run Y of 3" — a volt progress bar, and
  "N of 27 runs". **Logging a run auto-advances `done`** (`addMiles`); reaching 27 fires a
  graduation toast ("…you just graduated Couch to 5K! 🎓", overrides milestone/longest copy)
  and the card switches to a "Graduated 🎓" state. Position is adjustable (the dialog sets
  the next week/run; blank week turns the card off) in case auto-advance drifts from a
  non-programme run. Advance is capped at 27.
- **Next-run cue** (`#nextCard`): a **single** planned day, `data.next` (midnight ms) or a
  `#nextPlan` ghost prompt when unset (shown once ≥1 run). Deliberately one-at-a-time so
  missed runs never pile into a guilt mountain. `dayLabel` shows Today/Tomorrow/weekday/
  "12 Sep"; overdue is warm not red (`.past`, "no rush — nudge it along whenever"), today is
  `.due` (volt border). `+1 day` bump = `max(next+DAY, todayMid)` (never lands in the past,
  so an overdue cue bumps to today, then tomorrow). Tapping the day opens a date picker
  (blank clears). Logging a run **clears the cue only if it was due** (`midnight(next) <=
  todayMid`) — an early/bonus run leaves a future plan standing.
- **Events / races** (`#eventsCard` + `#eventAdd2` ghost): user-entered `data.events`
  `[{id,name,d,m?}]`, shown as upcoming-only (past auto-filtered) countdowns via `countdown()`.
  Add/edit through `openEvent` (name + date picker + optional distance; blank name deletes when
  editing). Names are `esc()`-escaped — the only free-text field rendered via innerHTML, so
  **keep it escaped**. B15: events take an optional distance `m`; when set, the row shows a
  volt readiness bar of **longest run / event distance** with a factual, non-shaming note
  ("Longest run so far: 2.5 of 3.1 mi", or "You can already run this distance 👊" once
  reached) — the "am I on track" signal. Distance-less events just show name + date + countdown.
The shared `ask()` dialog gained per-field `type2/inputmode2/min2/max2` (and `*3`) so the
secondary inputs can be date/number (events date, C25K week+run) — additive, existing callers
unaffected.

## Backlog — prioritised for the real goal (keep running through and past C25K)

1. **Export / import backup — shipped in B11–B12.** Delivered as the weekly recap email
   (every send embeds a full JSON backup) plus a `Restore` footer link that reads a backup
   pasted from any recap email. This replaced the originally-planned clipboard copy/paste
   buttons. Note the freshness caveat: the newest backup is only as recent as the last
   weekly send (or a manual Save & test).
2. **C25K programme card + events — shipped in B14.** Programme card ticks runs off toward
   W9R3 graduation; a single bump-able next-run cue; a user-managed races/events list with
   countdowns. See the B14 note above. **Post-graduation follow-through is still open**: the
   card currently stops at "Graduated 🎓" — the planned next step is to roll graduation into
   a rhythm-keeping goal (the weekly rhythm strip is the natural anchor) so the structure
   doesn't just vanish at the exact known drop-off point.
3. **Weekly rhythm strip — shipped in B13.** Last 8 Monday-weeks as bars (runs per week)
   between the trend chart and the run log; rest weeks are quiet empty slots, never red.
   Hidden when lapsed (nothing in 8 weeks). See the B13 note above.
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
