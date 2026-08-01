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
| B | **Strava as a pipe.** Garmin→Strava auto-sync plus Strava's self-service API. Fully-supported APIs on both hops — the most institutionally robust route. But it adds a third-party account Anthony doesn't want and can never carry wellness data. | **Break-glass fallback only.** The account isn't even created unless route E ever dies for good (§8). |
| C | Manual `.fit` share into the PWA (Web Share Target + inline FIT parser). | Rejected — no backend, but *manual*, which defeats the entire point. |
| D | Unofficial Garmin login from the Worker itself. | Rejected — credential-holding in the cloud, and Garmin's bot protection is hostile to datacenter IPs like a Worker's. |
| E | **`garth` relay on Anthony's always-on PC** (the approach GarminDB is built on). A small scheduled Python script performs Garmin's own app-style login via `garth` (unofficial, actively maintained), polls Garmin Connect for recent activities every ~10 min, normalizes, and POSTs them to the bridge. Credentials and tokens never leave Anthony's machine. | **✅ Chosen.** Anthony's PC doubles as his Plex server — always on — so relay latency matches a cloud poll. Native end-to-end, no third-party account, and the future wellness side-channel (§5) becomes *the same script* fetching one more thing daily. |

*How the verdict moved (twice):* the official API fell to the business gate; the PC
relay was initially rejected on latency ("PC must be awake → runs arrive hours late →
the arrival ritual dies") — then Anthony pointed out the PC is an always-on Plex server,
which dissolves that objection entirely. The honest residual cost of E is the unofficial
auth: Garmin can change login internals and break `garth` for a few days until the
library updates. The failure mode is graceful (arrivals pause, Tread still works, manual
add exists, catch-up on recovery), and route B remains documented as break-glass.

**On the FIT-file assumption:** we still do *not* need FIT parsing or per-second
streams. Garmin Connect's activity **summaries** (as `garth` returns them) carry every
insight this plan shows. GarminDB's full FIT archive is a nice *personal backup* habit,
but it's not Tread's pipeline.

### Chosen shape

```
Forerunner 970 ──auto──▶ Garmin Connect (cloud)
                              ▲ poll every ~10 min (garth, unofficial login)
                        PC relay — bridge/relay.py on the always-on Plex box
                              │ POST /ingest  (Bearer BRIDGE_TOKEN)
                        tread-bridge (CF Worker + KV)
                              │ GET /runs?since=…  (Bearer)
                        Tread PWA (fetch on open; offline-first)

        [parked, §5]  the same relay adds a daily wellness snapshot → POST /wellness
```

- **The relay pushes; the Worker never holds Garmin credentials.** `relay.py` runs on
  Windows Task Scheduler every ~10 min: `garth` login (token cached locally, ~year-long;
  occasional interactive re-auth), fetch recent activities, filter to runs, normalize to
  the contract below, POST the batch to the bridge. The Worker just authenticates,
  upserts into KV, serves `GET /runs`. Missed windows self-heal because the relay always
  asks for "recent N", not "since last success" alone.
- **The ingest side is an adapter.** The client's only contract is `GET /runs` plus the
  normalized shape below. If `garth` ever breaks for good, the break-glass Strava
  adapter (§1.B) replaces the relay — the Worker gains a poller, the client never knows.
- **House style extends to the bridge**: two small files, `bridge/worker.js` (plain JS,
  no npm, paste-deployable in the Cloudflare dashboard) and `bridge/relay.py` (Python,
  stdlib + `garth` only). Neither is Pages-deployable — Opus: amend the CLAUDE.md hard
  rule to say the *Pages* deploy set is exactly the five files, and that `bridge/`
  deploys separately (Worker → Cloudflare, relay → Anthony's PC).

### Build order (fixtures first)

Everything the client needs — B20 merge logic and settings, the B21 arrival ritual, the
B22 insight — is built and verified against the bridge's `?demo=1` fixtures, which need
no credentials at all. The live relay is wired last, once the client work is proven.
Manual logging works throughout, so nothing is ever broken mid-build.

### Data contract (the only coupling between Worker and client)

Normalized run, as stored in KV and returned by the bridge:

```json
{ "gid": "garmin-14963210055", "src": "garmin",
  "d": 1756725600000,        // epoch ms, activity start
  "m": 2.3,                  // miles, 1dp
  "t": 1462,                 // moving duration, seconds
  "hr": 156, "hrx": 172,     // avg / max heart rate, bpm (optional)
  "cad": 164,                // avg cadence, steps/min (optional — see units note below)
  "el": 118,                 // elevation gain, feet (optional)
  "name": "Morning Run",     // activity title (optional)
  "gear": "pegasus-40-uuid"  // Garmin gear id (optional — stored now, used later, #7)
}
```

(`src` stays in the shape so the break-glass Strava adapter — or the wellness
side-channel — can coexist; the client treats all sources identically. Garmin tracks
gear natively, which Anthony already uses — the relay includes the gear id if cheaply
fetchable, else drop and revisit with backlog #7.)

Bridge endpoints (all JSON):

- `GET /runs?since=<epoch_ms>` — runs with `d > since`, ascending. Auth: `Authorization:
  Bearer <BRIDGE_TOKEN>`. This is the only endpoint the PWA calls routinely.
- `GET /runs?demo=1` — no auth, returns a small hardcoded fixture set. Exists so the
  client can be built and verified in preview without live credentials — it is the
  backbone of the build order above. Keep it.
- `POST /ingest` (Bearer) — the relay's write path: accepts a JSON array of normalized
  runs, upserts by `gid`, returns `{stored, skipped}`.
- `GET /health` — unauthenticated `{ok, lastIngest, runCount}` for debugging. "Relay
  gone quiet" shows up here as a stale `lastIngest`.
- `POST /wellness` (Bearer) — reserved for the parked §5 side-channel; may be stubbed.

Relay internals (`bridge/relay.py` — **verify `garth` usage and Garmin Connect field
names/units at build time**; this doc is design-authoritative, not API-authoritative):

- Python 3, dependencies: `garth` only. Config (env or a small `.ini` beside it):
  Garmin email, bridge URL, `BRIDGE_TOKEN`. First run performs the interactive `garth`
  login (handles MFA) and caches tokens locally (~year-long); after that it's headless.
- Every ~10 min via Windows Task Scheduler: fetch the latest ~10 activities, filter to
  running types, normalize (metres→miles 1dp; **check whether Connect reports run
  cadence single-leg or total steps/min**), POST the batch to `/ingest`. Always sends
  "recent N" so missed windows self-heal; the Worker's upsert makes re-sends harmless.
- A `--backfill N` flag for first-run seeding of history (replaces the old admin
  endpoint idea — backfill is just a bigger poll from the same script).
- Log to a local file, quiet on success; on repeated auth failure print the re-login
  instruction. Never crash the scheduled task loop.

Worker internals (`bridge/worker.js`):

- One secret: `BRIDGE_TOKEN`. KV namespace bound as `KV`. No Garmin credentials, no
  OAuth, no cron — the Worker is a dumb authenticated shelf: `POST /ingest` upserts
  `run:<gid>`, `GET /runs` reads, `lastIngest` timestamp maintained.
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

Also parked: gear filtering per shoe (entries store Garmin `gear`; revisit with backlog
#7 shoe shelf), and the **wellness side-channel** — now simply the existing relay
fetching a daily body-battery / training-readiness snapshot and POSTing it to
`/wellness`, feeding *kind* readiness-aware nudges. No new infrastructure at all: same
script, same token, one more fetch. This is how the watch's deep data eventually pays
off natively. It still waits until the core loop has proven itself.

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

1. Cloudflare dashboard: create a Worker (`tread-bridge`) and a KV namespace bound as
   `KV`; set the single secret `BRIDGE_TOKEN` (generate a long random string); paste
   `bridge/worker.js` when Opus produces it. No cron trigger needed — the relay pushes.
2. On the Plex box: install Python 3 if absent, `pip install garth`, drop `relay.py` and
   its config (Garmin email, bridge URL, token) where Opus specifies, run it once
   interactively to complete the Garmin login (MFA handled here; token then cached
   ~a year), run `--backfill` once to seed history, then add the Task Scheduler job
   (every 10 min, on boot, run whether logged on or not).
3. Paste the bridge URL + token into Tread's new Garmin settings dialog.

(Optional, unrelated to the pipeline: run GarminDB locally now and then as a personal
archive of your full FIT history — a good data-ownership habit that rides the same
`garth` login.)

**Break-glass path only if the `garth` route ever dies for good (§8):** create a
default-private Strava account, connect it in Garmin Connect, create a Strava API app,
and Opus swaps the ingest adapter into the Worker — no client changes.

## 8. Risks

| Risk | Mitigation |
|---|---|
| **Garmin changes login internals and `garth` breaks** (the inherent cost of the unofficial route) | Historically the library recovers in days; the failure mode is graceful — arrivals pause, Tread keeps working, manual add exists, and the relay's "recent N" poll catches everything up on recovery. `pip install -U garth` is usually the whole fix; worst case re-run the interactive login. If it ever breaks *permanently*, the break-glass Strava adapter (§1.B/§7) replaces the relay with zero client changes. |
| Garmin objects to unofficial access | Single user reading his own account's data at ~6 polls/hour — the same pattern GarminDB and its ecosystem have run publicly for years. Low risk, honestly held; the break-glass path exists if policy hardens. |
| PC/network outage (Plex box down) | Runs queue in Garmin Connect; the next successful poll ingests them. The ritual degrades to "arrives later", never "lost". `/health.lastIngest` makes silence visible. |
| `garth`/Connect field details in this doc are stale | Opus verifies field names/units (cadence single-leg vs total, distance units) against the live library at build time; this doc is design-authoritative, not API-authoritative. |
| Double counting during transition | Merge rules in §2.3; verified with fixtures. |
| Scope creep into dashboard-land | §0 and §6. The reviewer question for every addition: *does this get him out the door on a bad day?* |
