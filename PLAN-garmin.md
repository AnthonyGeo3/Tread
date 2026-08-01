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

(On Connect+: Anthony isn't subscribing, but the free Garmin Connect tier still includes
the core analysis — VO2max trend, training status, body battery and the like. Connect+
paywalls the premium "AI insights" layer on top, not the dashboards. So "Garmin owns
analysis" holds on the free tier, and Tread's position is unchanged.)

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
| A | **Official Garmin Connect Developer API.** Native, richest, wellness-capable. | **Rejected — business-gated.** The developer-program application is explicitly for companies; Anthony checked the form and confirmed. Not worth misrepresenting a hobby app to sneak through. Revisit only if Garmin ever opens individual access. |
| B | **Strava as a pipe.** Garmin→Strava auto-sync is built-in, phone-side and near-instant; Strava's API is self-service (an app in minutes). Run summaries carry distance, moving time, avg/max HR, cadence, elevation — everything B20–B22 shows. | **✅ Chosen for run ingest.** Framed honestly: Strava here is *transport plumbing*, not a platform — free account, every activity default-private, zero social use, set up once and never opened again. |
| C | Manual `.fit` share into the PWA (Web Share Target + inline FIT parser). | Rejected — no backend, but *manual*, which defeats the entire point. |
| D | Unofficial Garmin login from the Worker itself. | Rejected — credential-holding in the cloud, and Garmin's bot protection is hostile to datacenter IPs like a Worker's. |
| E | **GarminDB / `garth` PC relay** (the Reddit recommendation). `garth` performs Garmin's own app-style login (unofficial); GarminDB batch-downloads FIT + wellness into local SQLite. A scheduled PC script could push runs to the bridge. | **Rejected for run ingest** — it only runs while the PC is awake, so runs land hours-to-a-day late, which kills the arrival ritual (§3); unofficial auth adds fragility. **Earmarked for the wellness side-channel** (§5), where daily latency is exactly fine. GarminDB is also worth running occasionally as a *personal archive* of FIT history — good instinct, just not Tread's pipeline. |

*The principle that settles it — match each source to the latency its feature needs.*
The arrival ritual needs **minutes**, and with the official API closed to individuals,
only the phone-side Garmin→Strava hop delivers minutes. Wellness nudges (parked, §5)
need only **"this morning"**, which a small daily `garth` job on Anthony's PC can serve
natively — so the watch's deep data still pays off later without Strava ever carrying it.

**On the FIT-file assumption:** we still do *not* need FIT parsing or per-second streams.
Strava's run **summaries** carry every insight this plan shows. Nothing in B20–B22 needs
more.

### Chosen shape

```
Forerunner 970 ──auto──▶ Garmin Connect ──auto (phone)──▶ Strava (private, unused)
                                                            │ (poll, cron ~15 min)
                                                     tread-bridge (CF Worker + KV)
                                                            │ GET /runs?since=… (Bearer)
                                                      Tread PWA (fetch on open;
                                                             offline-first)
        [parked, §5]  PC (daily garth job) ──wellness snapshot──▶ tread-bridge
```

- **Polling, not webhooks, in v1.** A ~15-minute cron poll of "latest N activities" is
  ~96 requests/day (far inside Strava's limits for one user), has no subscription
  choreography, and self-heals missed events. Latency is minutes — all the arrival
  ritual needs, since the payoff moment is *the next time he opens Tread*. Webhooks are
  a v1.1 upgrade only if latency ever annoys.
- **The ingest side is an adapter.** The client's only contract is `GET /runs` plus the
  normalized shape below. The parked wellness side-channel (§5) would be a second,
  independent ingest path into the same KV — its failure or absence never touches runs.
- **House style extends to the Worker**: one plain-JS file, `bridge/worker.js`, no npm, no
  build step, deployable by pasting into the Cloudflare dashboard. `bridge/` is **not** a
  Pages-deployable file — Opus: amend the CLAUDE.md hard rule to say the *Pages* deploy
  set is exactly the five files, and that `bridge/worker.js` deploys separately to
  Cloudflare.

### Build order (fixtures first)

Everything the client needs — B20 merge logic and settings, the B21 arrival ritual, the
B22 insight — is built and verified against the bridge's `?demo=1` fixtures, which need
no credentials at all. The live Strava adapter is wired last, once the client work is
proven. Manual logging works throughout, so nothing is ever broken mid-build.

### Data contract (the only coupling between Worker and client)

Normalized run, as stored in KV and returned by the bridge:

```json
{ "gid": "strava-14963210055", "src": "strava",
  "d": 1756725600000,        // epoch ms, activity start
  "m": 2.3,                  // miles, 1dp
  "t": 1462,                 // moving time, seconds
  "hr": 156, "hrx": 172,     // avg / max heart rate, bpm (optional)
  "cad": 164,                // avg cadence, steps/min (optional — see units note below)
  "el": 118,                 // elevation gain, feet (optional)
  "name": "Morning Run",     // activity title (optional)
  "gear": "g12345678"        // Strava gear id (optional — stored now, used later, #7)
}
```

(`src` stays in the shape so a future native source — or the wellness side-channel —
can coexist; the client treats all sources identically.)

Bridge endpoints (all JSON):

- `GET /runs?since=<epoch_ms>` — runs with `d > since`, ascending. Auth: `Authorization:
  Bearer <BRIDGE_TOKEN>`. This is the only endpoint the PWA calls routinely.
- `GET /runs?demo=1` — no auth, returns a small hardcoded fixture set. Exists so the
  client can be built and verified in preview without live credentials — it is the
  backbone of the build order above. Keep it.
- `GET /health` — unauthenticated `{ok, lastPoll, tokenOk, runCount}` for debugging.
- `GET /auth/start` → Strava OAuth consent → `GET /auth/callback` stores the refresh
  token in KV. One-time, done by Anthony in a browser.
- `POST /admin/backfill?n=60` (Bearer) — pulls the last N activities for first-run
  seeding.

Worker internals (implementation notes — **verify current Strava specifics against
their live API docs at build time**: endpoint paths, scope names, token lifetimes,
field names/units. This doc is design-authoritative, not API-authoritative):

- Secrets: `STRAVA_CLIENT_ID`, `STRAVA_CLIENT_SECRET`, `BRIDGE_TOKEN` (long random
  string Anthony generates). KV namespace bound as `KV`.
- Token dance: access tokens are short-lived; refresh on expiry/401, persist the rolling
  refresh token in KV. Scope must cover reading all own activities (historically
  `activity:read_all` — verify).
- Cron (~*/15): fetch latest ~10 activities, filter to run types (`Run`, `TrailRun`,
  `VirtualRun`), normalize (metres→miles 1dp; **Strava's run cadence is per-leg —
  double it for steps/min, verify**), upsert into KV as `run:<gid>`, update `lastPoll`.
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

Also parked: gear filtering per shoe (entries store Strava `gear`; revisit with backlog
#7 shoe shelf), and the **wellness side-channel** — a small daily `garth` job on
Anthony's PC (route E, §1) POSTing a body-battery / training-readiness snapshot to the
bridge, feeding *kind* readiness-aware nudges. Daily latency is exactly right for a
morning nudge, the unofficial auth risk is isolated (if the job breaks, only wellness
pauses — runs are untouched), and credentials never leave Anthony's own machine. This is
how the watch's deep data eventually pays off natively. It still waits until the core
loop has proven itself.

## 6. What NOT to build — guardrails for the implementer

- No training-load / VO2max / recovery / zones UI. Garmin Connect owns analysis.
- No negative framing anywhere, ever. The EF card's only failure mode is absence.
- No accounts, no multi-user; the bridge token is a personal shared secret.
- No social surface from any source (kudos, segments, other people's anything) — the
  anti-goals forbid comparison.
- Don't touch the physics, the shoe, or the palette. The arrival pour *uses* the existing
  spring system; it does not replace it.
- The client must remain fully functional with the bridge unconfigured, misconfigured, or
  dead — every bridge call is progressive enhancement wrapped in the same try/catch
  humility as storage.

## 7. Anthony's manual homework (before/while Opus builds)

Nobody else can do these:

1. **Create the Strava pipe account**: sign up free, then in privacy settings default
   every activity to private ("Only You") — it's transport, not social. In Garmin
   Connect: Settings → Connected Apps → connect Strava so activities auto-sync.
2. Create a Strava API application (instant, free, from your account's API settings) →
   note client ID + secret.
3. Cloudflare dashboard: create a Worker (`tread-bridge`) and a KV namespace bound as
   `KV`; set the three secrets (`STRAVA_CLIENT_ID`, `STRAVA_CLIENT_SECRET`,
   `BRIDGE_TOKEN` — generate a long random string for the last); add the ~15-min cron
   trigger; paste `bridge/worker.js` when Opus produces it.
4. Visit `/auth/start` once in a browser and approve; then run the backfill command Opus
   gives you to seed history.
5. Paste the bridge URL + token into Tread's new Garmin settings dialog.

(Optional, unrelated to the pipeline: run GarminDB locally now and then as a personal
archive of your full FIT history — a good data-ownership habit, and the same `garth`
login it uses is what the parked wellness side-channel (§5) would build on.)

## 8. Risks

| Risk | Mitigation |
|---|---|
| Strava API details in this doc are stale | Opus verifies endpoints/scopes/fields against the live docs at build time; this doc is design-authoritative, not API-authoritative. |
| Garmin→Strava sync detaches or hiccups | The hop is phone-side and historically very reliable; polling reconciles missed windows; `/health` shows `lastPoll`; arrivals visibly stopping is itself the alarm. Manual add always works. |
| Strava tightens API terms | Recent tightenings target AI-training and Strava-replica apps; a single-user personal transport pipe is squarely inside normal use. Poll conservatively regardless. |
| Token refresh bugs strand the bridge | `/health` exposes token state; cron retries; worst case re-run `/auth/start`. |
| Double counting during transition | Merge rules in §2.3; verified with fixtures. |
| Future wellness side-channel breaks (unofficial `garth` auth) | Parked feature, isolated by design (§5): only wellness pauses; runs and the core loop are untouched. |
| Scope creep into dashboard-land | §0 and §6. The reviewer question for every addition: *does this get him out the door on a bad day?* |
