# PLAN — Garmin sync & the motivation centre (B20–B22, optional B23)

**Author: Fable 5 (design). Implementer: Opus (build).** Anthony bought a Garmin Forerunner
970. Garmin Connect now tracks his runs and shoe mileage automatically — which is Tread's
mechanical job. The decision (agreed with Anthony) is to **combine, not compete**: Garmin
becomes the data source, Tread becomes the *motivation centre* — the place that makes the
data feel like something, so he keeps running when he doesn't feel like it.

This document is the full spec. Opus: read the whole thing before writing code, then build
in the phase order given. Each phase is an independent release with the normal ritual
(footer `TREAD B<n>` + `sw.js` cache bump + CLAUDE.md note + commit/push). Verify each
phase in the browser before moving on, per house practice.

---

## 0. Product position (read this before any code)

Tread's moat is **emotional**, not analytical. Garmin Connect already renders every metric
a watch can produce; if Tread tries to be a dashboard it becomes a worse Garmin Connect and
loses the one thing it has: the shoe, the liquid, the warmth, the anti-shame stance.

**Governing rule for every decision below: import little, feel much.** A new number only
earns pixels if it answers *"will this make Anthony run on a day he doesn't feel like
it?"* Everything else stays in Garmin Connect where it belongs.

Two existing CLAUDE.md positions are consciously amended by this plan:

1. **"No backend" softens to "no backend *for the client's core loop*".** A tiny personal
   read-only bridge Worker is now allowed. The PWA stays a single offline-first
   `index.html`; if the bridge is down or unconfigured, everything still works manually.
2. **"The 5-second manual log is part of the reward loop; don't automate the ritual away"
   (backlog #12).** With a watch, manual logging stops being a ritual and becomes
   double-entry admin — the exact thing Tread hates. So we don't delete the ritual, we
   **relocate it**: the reward moves from *entering* the run to *witnessing it arrive*
   (see B21, the most important section of this plan). One human tap is deliberately kept.

---

## 1. Architecture decision

### Options considered

| | Route | Verdict |
|---|---|---|
| A | **Official Garmin Connect Developer API** (Activity/Health API). Push-webhook model, needs an approval application, OAuth, server endpoint. Richest data incl. wellness (body battery, readiness). | **Later, maybe.** Approval friction and heavyweight for v1. The adapter design below lets it slot in without client changes. Anthony can apply in parallel (free) if he wants wellness data someday. |
| B | **Strava bridge.** Garmin Connect → Strava auto-sync is built-in and near-instant. Strava's API is self-service (create an app in minutes), OAuth2, and its run summaries carry distance, moving time, avg/max HR, cadence, elevation — everything v1 needs. | **✅ Chosen for v1.** |
| C | Manual `.fit` share into the PWA (Web Share Target + inline FIT parser). | Rejected — no backend, but *manual*, which defeats the entire point. |
| D | Unofficial Garmin login scraping from a Worker. | Rejected — fragile, credential-holding, breaks without warning. |

**On the FIT-file assumption:** we do *not* need FIT parsing or per-second streams for
anything in this plan. Run-level summaries carry every insight v1 shows. Full FIT is the
heavyweight path (route A) and only pays off for data Tread shouldn't display anyway.

### Chosen shape

```
Forerunner 970 ──auto──▶ Garmin Connect ──auto──▶ Strava
                                                    │ (poll, cron ~15 min)
                                             tread-bridge (Cloudflare Worker + KV)
                                                    │ GET /runs?since=…  (Bearer token)
                                              Tread PWA (fetch on open; offline-first)
```

- **Polling, not webhooks, in v1.** A 15-minute cron poll of "latest N activities" is ~96
  requests/day (Strava's limits are far above this for one user), has no subscription
  choreography, and self-heals missed events. Latency is minutes — fine, since the payoff
  moment is *the next time he opens Tread*, not real-time. Webhooks are a v1.1 upgrade
  only if latency ever annoys.
- **The Worker is an adapter.** Its only contract with the client is the normalized run
  shape below. Swapping Strava for the official Garmin API later touches only the Worker.
- **House style extends to the Worker**: one plain-JS file, `bridge/worker.js`, no npm, no
  build step, deployable by pasting into the Cloudflare dashboard. `bridge/` is **not** a
  Pages-deployable file — Opus: amend the CLAUDE.md hard rule to say the *Pages* deploy
  set is exactly the five files, and that `bridge/worker.js` deploys separately to
  Cloudflare.

### Data contract (the only coupling between Worker and client)

Normalized run, as stored in KV and returned by the bridge:

```json
{ "gid": "strava-14963210055", "src": "strava",
  "d": 1756725600000,        // epoch ms, activity start
  "m": 2.3,                  // miles, 1dp
  "t": 1462,                 // moving time, seconds
  "hr": 156, "hrx": 172,     // avg / max heart rate, bpm (optional)
  "cad": 164,                // avg cadence, steps/min (optional — see note)
  "el": 118,                 // elevation gain, feet (optional)
  "name": "Morning Run",     // Strava title (optional)
  "gear": "g12345678"        // Strava gear id (optional — stored now, used later)
}
```

Bridge endpoints (all JSON):

- `GET /runs?since=<epoch_ms>` — runs with `d > since`, ascending. Auth: `Authorization:
  Bearer <BRIDGE_TOKEN>`. This is the only endpoint the PWA calls routinely.
- `GET /runs?demo=1` — no auth, returns a small hardcoded fixture set. Exists so the
  client can be verified in preview without live credentials. Keep it.
- `GET /health` — unauthenticated `{ok, lastPoll, tokenOk, runCount}` for debugging.
- `GET /auth/start` → Strava OAuth consent → `GET /auth/callback` stores the refresh
  token in KV. One-time, done by Anthony in a browser.
- `POST /admin/backfill?n=60` (Bearer) — pulls the last N activities for first-run
  seeding.

Worker internals (implementation notes, verify current Strava specifics against their
docs at build time — do not trust this doc for exact field names/scopes):

- Secrets: `STRAVA_CLIENT_ID`, `STRAVA_CLIENT_SECRET`, `BRIDGE_TOKEN` (long random
  string Anthony generates). KV namespace bound as `KV`.
- Token dance: access tokens are short-lived; refresh on 401/expiry, persist the rolling
  refresh token in KV. Scope needs to include reading all activities (verify name —
  historically `activity:read_all`).
- Cron (Cloudflare trigger, ~*/15): fetch latest ~10 activities, filter to runs
  (`Run`, `TrailRun`, `VirtualRun` types), normalize (metres→miles 1dp; **Strava's run
  cadence is per-leg — double it for steps/min, verify**), upsert into KV as
  `run:<gid>`, update `lastPoll`.
- Nothing in the Worker is ever deleted by the client; the bridge is read-only from the
  client's perspective. A nice side effect: **KV is now a second backup** of the run
  history, independent of the email backups.

---

## 2. B20 — Silent sync (plumbing, no ceremony)

Goal: synced runs appear in Tread exactly like manually logged ones. No new UI beyond a
settings dialog. Ship this, watch it work for a few days.

**Client changes (`index.html`):**

1. **Settings**: new footer link `Garmin` → `ask()` dialog with two fields: bridge URL and
   token. Store as `data.bridgeUrl` / `data.bridgeToken` (defensive defaults; **excluded
   from the backup payload**, same reasoning as the EmailJS creds — least data, and a
   restore must not clobber a device's own config). "Save & sync" triggers an immediate
   sync with a toast, mirroring the recap dialog's save-and-test pattern.
2. **Sync on open** (and on `visibilitychange` → visible, throttled to ≥5 min apart):
   `GET {bridgeUrl}/runs?since={data.lastSync||0}`. Timeout ~8s. **Silent on failure** —
   offline-first is sacred; no error toast for routine failures (surface errors only on
   the deliberate save-and-test).
3. **Merge rules** (order matters):
   - Incoming `gid` already on an entry → update that entry in place (fields may improve).
   - Else, if a **manual** entry (no `gid`) exists on the same local calendar day with
     distance within 10% → *upgrade it in place*: attach `gid`, `hr`, `cad`, `el`, `t`
     (watch time wins), keep the manual entry's id. This absorbs the transition period
     where he logs manually *and* the watch syncs.
   - Else append a new entry `{id, gid, src:'g', m, d, t?, hr?, cad?, el?, name?}`.
   - After merge: `data.lastSync = max(d)` of received runs; save; `render(false)` in B20
     (B21 changes this to the arrival moment).
4. **Schema**: entries gain optional `gid, src, hr, cad, el, name`. Backup payload → v20,
   `parseBackup` preserves the new fields (follow the existing optional-field pattern —
   absent keys leave device values untouched). `lastSync` is device-local: **not** in the
   backup.
5. **Run detail popup** (B17) gains lines when data exists: `♥ 156 avg · 172 max`,
   `164 spm · 118 ft climb`. Nothing shown for absent fields.
6. Milestones/moments (`addMiles`'s toast logic) must fire for synced arrivals too —
   extract the moment-detection from `addMiles` into a shared function both paths call.
   In B20 a plain toast is fine; B21 upgrades the ceremony.
7. The existing manual add flow is untouched — it's the fallback for treadmill/no-watch
   runs forever.

**Verification (B20):** point the client at `?demo=1` fixtures (settings accepts the demo
URL, or a temporary override — Opus's choice, keep it clean), confirm: entries appear,
de-dupe upgrade works (seed a same-day manual entry first), totals/fill/chart/rhythm all
update, popup shows HR lines, backup round-trips the new fields, and the app still works
with no bridge configured and with the network dead.

---

## 3. B21 — The arrival ritual ⭐

This is the heart of the plan. The manual-log reward loop is being automated away; this
is its replacement. Get the *feel* right.

**The pour.** When sync lands new run(s) while the app is open at the front (the common
case: he finishes a run, watch syncs, he opens Tread a few minutes later):

1. The liquid does **not** silently jump. Hold the pre-run fill level, then play an
   arrival: fill spring target moves to the new level with a deliberately springy rise
   (temporarily boost wave amplitude the way `render(true)` splashes today, but let it
   read as a *pour*, ~1.5–2s), while a toast lands: **"2.3 mi just arrived from your
   watch"**. Milestone/longest/first-5K moments fire as today, after the pour.
   `prefers-reduced-motion`: skip the theatrics, keep the toast.
2. **The arrival card** — a one-time card at the top of the `.plan` section for the most
   recent new run: date/name, distance, time, pace, HR if present. It carries the *one
   human tap* that keeps this a ritual and not a feed:
   - **Feel tag**: 😄 🙂 😮‍💨 — one tap stores `f` (1/2/3) on the entry. This ships
     backlog #5 as part of the ritual, not a separate feature.
   - **C25K attribution** (only when the programme is active and not graduated): a green
     tick "This was W6R2 ✓" — same semantics as `#c25kDone` today (advance, cap,
     graduation toast, confetti, clear `c25kNext`). The general next-run cue keeps its
     existing due-day auto-clear behaviour.
   - Dismiss (×) just hides the card. It never re-appears for that run, never nags.
   - Card state: `data.pendingArrival = gid` (or null), cleared on tag/tick/dismiss.
     Only ever **one** card — if several runs arrive at once, the card shows the latest
     and the toast aggregates ("2 runs · 5.4 mi arrived"); older ones import silently.
     No queues, no backlog of cards — same philosophy as the single next-run cue.
3. **Feel aggregate** (tiny, ships here since `f` now exists): when ≥10 tagged runs and
   ≥70% are 😄/🙂, a quiet line appears near the rhythm strip: *"8 of your last 10 runs
   felt good."* Below the threshold it simply doesn't render. Never the inverse.

**Verification (B21):** demo fixtures with a staged `lastSync` so arrivals replay;
confirm pour + toast + card, each tap path (feel/C25K/dismiss), aggregate line at the
threshold, reduced-motion path, and that a second open shows no card re-run.

---

## 4. B22 — Kind insights + recap upgrade

Exactly **one** new analytical card, plus enrichment of what exists. Resist adding more.

1. **"Easier" card** — the single most motivating thing HR data can say to a beginner:
   *your easy is getting easier*, even when pace looks flat. Compute per-run efficiency
   (pace per heartbeat: `(t/m)/hr`, runs with both `t` and `hr` only). Compare the median
   of the trailing 21 days vs the previous 21 (require ≥3 qualifying runs in each
   window). If improved ≥ ~2%: show a card — *"Same effort, ~12 s/mi faster than a month
   ago. Your heart is doing less work per mile."* If flat or worse, or data is thin:
   **the card does not render.** Hard rule inherited from the anti-goals: this card has
   no negative state — absence *is* the negative state, and absence looks identical to
   "not enough data". Recompute on render; no stored state.
2. **Recap email**: `recapCopy` gains the week's HR-flavoured warmth when data exists
   ("longest 2.5 mi, avg ♥ 151") and the backup payload inside it now naturally carries
   the enriched entries. No template changes for Anthony.
3. Event readiness bars and the rhythm strip need nothing — they derive from entries and
   now simply update themselves. Confirm, don't rebuild.

**Verification (B22):** fixture sets that put EF improvement above/below threshold and
with thin data — card renders only in the first case; recap compose check; nothing
negative appears anywhere.

---

## 5. B23 (optional, parked until the on-open nudges prove insufficient)

The Worker's existence makes backlog #10 (real Web Push on planned days) honest to build:
client POSTs its planned days (`next`/`c25kNext`) to the bridge; a daily cron sends one
warm push on a planned morning ("Trainers by the door — today's the day you picked").
Opt-in, one per planned day, never on unplanned days, never guilt. **Do not build this in
the same pass as B20–B22.** Ship the sync, live with it, then decide.

Also parked: gear filtering (entries already store `gear`, so when pair #2 arrives the
shoe-shelf work of backlog #7 can filter per-shoe), official Garmin API swap (wellness /
readiness-aware nudges), webhook latency upgrade.

## 6. What NOT to build — guardrails for the implementer

- No training-load / VO2max / recovery / zones UI. Garmin Connect owns analysis.
- No negative framing anywhere, ever. The EF card's only failure mode is absence.
- No accounts, no multi-user; the bridge token is a personal shared secret.
- No Strava social surface (kudos, segments, other people) — anti-goals forbid comparison.
- Don't touch the physics, the shoe, or the palette. The arrival pour *uses* the existing
  spring system; it does not replace it.
- The client must remain fully functional with the bridge unconfigured, misconfigured, or
  dead — every bridge call is progressive enhancement wrapped in the same try/catch
  humility as storage.

## 7. Anthony's manual homework (before/while Opus builds)

Nobody else can do these:

1. In Garmin Connect: connect Strava (Settings → Connected Apps) so activities auto-sync.
2. Create a Strava API application (instant, free) → note client ID + secret.
3. Cloudflare dashboard: create a Worker (`tread-bridge`), a KV namespace bound as `KV`,
   set the three secrets, add the ~15-min cron trigger, paste `bridge/worker.js` when
   Opus produces it.
4. Visit `/auth/start` once and approve; then `POST /admin/backfill` (Opus will give the
   exact command) to seed history.
5. Paste the bridge URL + token into Tread's new Garmin settings dialog.

## 8. Risks

| Risk | Mitigation |
|---|---|
| Strava API details in this doc are stale | Opus verifies endpoints/scopes/fields against live docs at build time; this doc is design-authoritative, not API-authoritative. |
| Missed syncs / Strava hiccup | Polling reconciles automatically; manual add always works. |
| Token refresh bugs strand the bridge | `/health` exposes token state; cron retries; worst case re-run `/auth/start`. |
| Double counting during transition | Merge rules in §2.3; verified with fixtures. |
| Scope creep into dashboard-land | §0 and §6. The reviewer question for every addition: *does this get him out the door on a bad day?* |
