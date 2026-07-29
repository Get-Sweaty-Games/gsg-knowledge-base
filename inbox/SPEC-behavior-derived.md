# Behavior Spec — derived (DRAFT, needs reconciliation)

> **Provenance / health warning.** This spec was reverse-derived from the **`main`**
> branch's docstrings, code comments, `supabase/migrations/*.sql`, and `CLAUDE.md`
> during a test-independence audit (2026-07-22). It is a *positive statement of
> intended behavior per domain* — the raw material for a real spec, not the spec
> itself.
>
> **Reconciled against the `godot-pivot` branch (2026-07-23).** The client-side
> spec (`CLAUDE.md`, `docs/PLAN-godot.md` on that branch) has been folded in — see
> the new **Client** section below and the penalty-freeze note under *Consequence
> tick*. **Where the two disagreed, the branch's client spec wins.**
>
> **Client engine DECIDED 2026-07-28: Unity.** The Godot evaluation is
> archived at git tag `archive/godot-pivot-phase1` (branch `godot-pivot` retained).
> **Topology inverted the same day: Unity-as-base, not UaaL** — a partner studio
> (`Get-Sweaty-Games/reign-and-gain`) owns and builds the entire Unity client, and
> our Kotlin host ships to them as `hostbridge.aar`. Our own thin client shell was
> deleted. See `docs/unity-plugin-topology.md` § Reversal.
> Two things this does *not* mean, both load-bearing:
>
> 1. **Nothing about the client is proven live on device.** The C# that existed in
>    this repo compiled clean on 2026-07-28, but that code is now deleted and the
>    client is being rebuilt by the partner against `docs/unity-client-brief.md`.
>    What is unproven is everything past the contracts: `hostbridge.aar` has not
>    been built, delivered, or loaded; the manifest-fragment launcher override has
>    not been exercised; no bridge round trip has run on real hardware. Deciding the
>    engine closed exactly one of the three conditions gating the release cluster;
>    see `docs/PLAN-closeout.md` Phase 5.
>
>    Recorded so it isn't "fixed" into a bug: that compile emits 6 `UAC1001`
>    serialization warnings on the nullable fields of `Bridge/RawHealthSignals.cs`.
>    They are **false alarms** — that DTO is deserialized only via Newtonsoft
>    (`GameBootstrap.cs:417,459`), never `JsonUtility`, and the nullability is
>    deliberate and load-bearing (`null` = signal absent vs `0.0` = a measured zero;
>    collapsing it would bias the evidence bundle). Do not "resolve" those warnings
>    by removing the `?`.
> 2. **The § Client section below stays engine-neutral, deliberately.** It describes
>    *behavior*, and behavior did not change when the engine got picked. The
>    `godot-pivot`-derived content was folded in precisely *because* it was
>    engine-neutral, and it is still correct. Engine-specific plumbing (autoload/nav
>    design, the HTTP-layer sketch, the GDScript port gotchas) remains out of scope
>    here — that is implementation, not spec.
>
> The Moat / Game / Edge sections remain **backend behavior derived from `main`**;
> the branch's `CLAUDE.md` locks the same backend decisions, so nothing there
> contradicted them. Sections were filled in one domain at a time by the agents that
> audited `backend/src`; the Client section was added during the 2026-07-23
> reconciliation.

---

## Moat

### Verification — trust scoring

- A submitted `EvidenceBundle` carries `source` (health_connect / strava / manual / tracked_gym_session / tracked_run) and `activityType`. `VerificationEngine.score(bundle, ctx)` resolves a `SourceProfile` for `source`, then runs every registered `SignalChecker` named in that profile's `weights` map (a checker not named in the map does not run at all).
- Profile lookup order: gameconfig row `verification.sourceProfiles` (Zod-validated) → built-in `DEFAULT_SOURCE_PROFILES` → **fail-closed default** `{ acceptThreshold: Infinity, weights: {} }` for any source with no profile anywhere. Fail-closed means every bundle for an unregistered source is rejected regardless of content — this is the moat's backstop against a half-added source (enum/catalog updated, profile forgotten).
- Weighted trust score = `Σ(weight[checker] × score[checker]) / Σ(weight[checker])` over applicable checkers; `1` when total weight is `0` (no weighted checkers → pass on absence of gates).
- `rejected = hardReject(any checker's reject:true) OR total < acceptThreshold`. A weight-0 checker still runs and can hard-reject (e.g. `metric-rate`) — weight 0 means "gate, not vote."
- **Coverage-cap multiplier** (`profile.coverageCap === true` only — client-attested sources opt in; pull sources like Strava opt out because their trust is provenance, not metric coverage):
  - `expectedCount` = number of applicable magnitude-band fields for `bundle.activityType` (distance expected if `band.distanceMin != null`, HR expected if `band.hrMin/hrMax != null`). `expectedCount === 0` (unbanded type, e.g. `steps`) → factor `1`, no cap.
  - `presentCount` = how many of those are actually present in `bundle.metrics`.
  - `coverage = presentCount / expectedCount`.
  - `floor = 0.85` if `bundle.originPackage` resolves to a known hardware integrator (Garmin etc., via `domain/sources.ts` registry), else `0.7`.
  - `coverageFactor = floor + (1 − floor) × coverage`.
  - `total = weighted × coverageFactor` — a multiplier applied AFTER the weighted mean, never as a peer checker in the average (a peer term would dilute anchor-gated profiles).
- Default source profiles (`sourceProfiles.ts`):
  - `health_connect`: threshold `0.6`, weights `{manual-entry:0, distance-length:1, heart-rate:1, metric-rate:0}`, `coverageCap:true`.
  - `strava`: threshold `0`, weights `{manual-entry:0, provider-flagged:0}`, no coverage cap (magnitude bands dropped — provenance is the proof).
  - `manual` (free-form typed entry): threshold `0`, weights `{metric-rate:0}` — trust rides on the `manual_logging_enabled` whitelist enforced at the route, not scored here.
  - `tracked_gym_session`: threshold `0.6`, weights `{gym-presence:1, metric-rate:0}`, `coverageCap:true` — gym-presence is the SOLE anchor (no accel collected in the passive ping model).
  - `tracked_run`: threshold `0.6`, weights `{track-consistency:1, accel-presence:1, metric-rate:0}`, `coverageCap:true`.

### Verification — checker contracts

- **GymPresenceChecker** (`gym-presence`, weight varies by profile) — input: `bundle.gpsContext` + the user's registered gyms (`GymService.listGyms`). No `gpsContext` → score `0` (soft, `reject:false`). No registered gyms → score `0`. Otherwise delegates to `findMatchingGym` (shared with the reward path — see matchGym below) and returns its score (`0`, `0.5`, or `1`). Never hard-rejects; the profile threshold is the decision-maker.
- **MetricRateChecker** (`metric-rate`, weight `0` in every client-attested profile — hard gate only) — input: `bundle.metrics` + claimed `[startedAt,endedAt)` window. Computes a window-scaled ceiling per metric: `25 kcal/min`, `1000 m/min` (60km/h), `300 steps/min`, all scaled by `min(windowSeconds, MAX_SESSION_SECONDS=6h)/60` minutes; plus `durationSeconds ≤ trueWindowSeconds + 60s` slack (uses the TRUE, unclamped window — active time exceeding elapsed time is impossible at any session length). Window is floored at `MIN_WINDOW_SECONDS=60s` (inverted/zero window can't zero the ceilings). Any metric over its ceiling → `score:0, reject:true` with a reason naming the metric; otherwise `score:1, reject:false`. This is the anti-mint hard gate closing the "huge metric over a short/inflated window" reward-mint vector.
- **TrackConsistencyChecker** (`track-consistency`, weight `1` for `tracked_run`) — input: `bundle.runTrack {sampleCount, trackedDistanceMeters}` vs `bundle.metrics.distanceMeters`. No `runTrack` → score `0` (no evidence). `sampleCount < 10` → score `0.3` (too sparse to mean anything). No claimed distance to compare → score `0.7` (track-only, better than nothing). Otherwise route-agreement score `= clamp01(1 − |tracked − claimed| / claimed)`. Then an energy-commensurability bound is applied AFTER route agreement and can only LOWER it (never rescue): `plausibleKcal = 150 + 300 × trackedKm`; if `activeEnergyKcal` is claimed and `plausibleKcal / activeEnergyKcal < routeAgreementScore`, the final score drops to that ratio (perfect distance agreement must not rescue an energy claim the route can't support). Never hard-rejects.
- **Motion-corroboration — SHIPPED (Phase 1 task 1d; agreed 2026-07-23, built 2026-07-27).** Ships
  as an **edit to the existing `AccelPresenceChecker`** (`accel-presence`), not a new registered
  `SignalChecker` — `docs/PLAN-closeout.md` revised the original "new checker class" framing below
  once it checked the actual `EvidenceBundle` shape: editing a checker's own `evaluate()` body is
  normal maintenance, not an Open/Closed violation (that principle guards the
  `VerificationEngine`'s scoring loop, not a checker's internals). Same `name`/weight (`1`,
  unchanged) as before, so **no `SourceProfile` or gameconfig change was needed** — and because
  it's not a new peer term in the weighted mean but a change to what the existing term returns,
  the weight-0-registration guidance below does not apply to it. Logic: guard on
  `accelPresence === false` (not falsy — an auto-synced source that merely *omits* the field is
  never penalized) combined with `runTrack.trackedDistanceMeters > 50` (`MIN_MOVEMENT_METERS`,
  above GPS-jitter noise for a genuinely stationary phone) → `score: 0, reject: false`. GPS moving
  while the accelerometer explicitly reports stationary is a classic location-spoof tell (sit
  still, feed a fake GPS track); resolves the "GPS moving but phone not moving" scenario.
- **`track-smoothness` — still AGREED, NOT YET IMPLEMENTED.** A real route wobbles; a suspiciously
  smooth line suggests a synthetic/replayed track — a heuristic on `runTrack` sample noise. Not
  buildable as scoped: `EvidenceBundleSchema.runTrack` only carries **aggregates**
  (`sampleCount`, `trackedDistanceMeters`) today, not the raw per-sample points the noise heuristic
  needs, so it needs host-side capture work first (see `docs/PLAN-closeout.md`, moved out of
  Phase 1 for this reason). When built, it IS a genuinely new checker, so the registration form
  below still applies to it.
  - **Registration form (for a new checker like the still-pending `track-smoothness`) — weight-0
    conservative hard gates, not weighted peers.** Per the anchor-dilution rule, adding a
    neutral-passing peer to the `track-consistency`-anchored tracked profile would dilute the
    anchor; so it should register at **weight 0** ("gate, not vote") and **reject only an
    egregious, unambiguous spoof**, passing the ambiguous middle — which is what keeps an honest
    player on bad GPS (dense city / tunnel / forest) from being flagged. *Open detail:* if binary
    proves too blunt, the alternative is a graded **post-weighted-mean multiplier** (the
    coverage-cap pattern), which lowers trust without diluting or rescuing. **Deferred to Tier-2:**
    cross-day **replay detection** (identical track repeated daily) — needs stored per-run track
    fingerprints; documented, not built now.

### Kinetic translation — evidence → rewards

- `KineticTranslationEngine.translate(bundle, activityTypeOverride?)`: `activityType = override ?? bundle.activityType` (override is used for `tracked_gym_session`, where reward follows the VENUE's discipline, not the client's per-session claim).
- Loads `KineticFormula` from gameconfig key `kinetic.formula` (Zod-validated, falls back to `DEFAULT_KINETIC_FORMULA` on missing/legacy/malformed row).
- `coeffs = formula.byType[activityType] ?? formula.default`; `rawXp = floor(Σ metric_value × coefficient)` summed once across all configured metrics, then floored once (not per-metric).
- `xp = min(rawXp, formula.maxXpPerActivity ?? 1000)` — the absolute anti-mint backstop (bounds the reward regardless of how extreme/coherent a fabricated claim is; no rate checker can bound this because kcal/duration both scale with a client-controlled window).
- `gold = floor(xp × formula.goldPerXp)` — derived AFTER the XP cap, so gold is capped too.
- `stats`: zero-filled `{str,dex,con,wis}`, with `stats[axis] = formula.statPointsPerActivity` where `axis = ACTIVITY_STAT_AXIS[activityType]` (`null` for `workout` → no stat delta).
- Default coefficients (0017/0034 seed mirror): `running/cycling → 0.1×kcal`, `walking → 0.08×kcal`, `strength → 0.15×kcal + 0.05×durationSeconds`, `hiit → 0.15×kcal`, `yoga/pilates/dance/mobility → 0.12×kcal`, `steps → 0.001×stepCount`, `workout → 0.1×kcal`. `goldPerXp:1`, `statPointsPerActivity:1`, `maxXpPerActivity:1000`, `maxXpPerDay:2000` (this last one is enforced in Postgres via the 0028 trigger under an advisory lock, NOT in this engine — carried in the type only so config/interface/default stay one mirrored document).
- Stat axis (structural, code not config — `activityTypes.ts`): `running/walking/cycling/steps → CON`, `strength → STR`, `hiit/dance → DEX`, `yoga/pilates/mobility → WIS`, `workout → null`.

### Gyms — session model, discipline, venue validation

- **Passive ping model** (Phase B, no accelerometer collected): app-open location pings are matched against the user's registered gyms via `findMatchingGym` (shared haversine matcher in `matchGym.ts`, also used by GymPresenceChecker so both callers agree on which venue a point sits at). A miss stores nothing but is counted; a match is stored to `gym_pings`.
- `findMatchingGym`: slack `= min(claimedAccuracyMeters, 100)` (accuracy is client-reported/untrusted, credited at most 100m). Inside `radius + slack` → score `1` (first inside-fence hit wins, whole list scanned before falling back to borderline). Inside `2×radius + slack` but outside the fence → score `0.5` (GPS drift / parking lot). Otherwise no match (`null`). A venue with `validationStatus === "rejected"` is skipped entirely (reject-only / fail-open — every other status, including `pending`/`unverified`, still matches).
- **Departure detection**: `AWAY_PINGS_TO_CLOSE = 2` — a single away ping is tolerated as jitter (in-memory per-user counter, resets on any at-gym ping); the second consecutive away ping triggers `finalizeNow` (closes immediately, bypassing staleness). In-memory counter is single-instance-only — a server restart loses it, but the staleness/daily-tick fallback still catches the session. **Confirmed not a landmine (scenario review 2026-07-23):** this is a deliberate simplification (`ponytail:` comment at the field), not an overlooked assumption — the failure mode (lost count on restart, or a second backend instance never seeing the same counter) degrades to "closes late via the 90-min staleness sweep or the daily tick" rather than losing the session, and the upgrade path (move to a service-role column) is already named and only triggers if the backend goes multi-instance, which it isn't today.
- **Session grouping** (`GymSessionFinalizer.groupPingsIntoSessions`): pings for one user+gym sorted ascending, split into a new session wherever the gap to the previous ping exceeds `SESSION_GAP_MS = 90min`. `SESSION_GAP_MS` doubles as the staleness cutoff for the sweep path (`finalizeStaleSessions`, called opportunistically after every matched ping and by the daily tick with no `userId` — sweeps every user with a stale unfinalized ping).
- **Credited duration** = `last.pingedAt − first.pingedAt` for the session, floored at `MIN_SESSION_CREDIT_SECONDS = 60` (a single stale ping still earns something). `finalizeNow`'s duration is exactly the last-at-gym-ping minus first (the walk home after the 2nd away ping is never counted, since away pings are never stored).
- Each session becomes a synthetic `tracked_gym_session` `EvidenceBundle`: `activityType = DISCIPLINE_ACTIVITY_TYPE[gym.discipline]`, `manualEntryFlag:true`, `gpsContext` = the gym's own coords with `accuracyMeters:0`, `metrics.durationSeconds` = the credited duration. `activityId = gymSessionId(userId, gymId, firstPingIso)` — a deterministic (non-cryptographic, format-only) UUID so re-finalizing the same session (concurrent tick + ping, crash-and-retry) upserts onto the same `activities` row instead of duplicating. Pings are claimed via `finalized_at IS NULL` update — awarded THEN marked, so a crash before the mark re-processes the same pings next run (idempotent). A deleted gym's pings still get marked finalized, just with no award (`sessionsFinalized` doesn't count them).
- **Real periodic ping trigger — NOT YET IMPLEMENTED (blocker on this whole model reflecting a true session, tracked in `docs/TODO.md`):** the entire passive-session design above assumes pings arrive throughout a workout, but today only `GameBootstrap.cs`'s `BeginAppOpenPing` fires — one best-effort ping per app open, not a recurring one. **No periodic scheduling exists anywhere in `android-host`** (no `WorkManager`/`AlarmManager`/foreground service); the dev driver's `startGymTracking` mocks the intended 5-min cadence for dogfooding only. Until a real trigger is built, credited duration for a live session under-reports true time at the gym (it only spans whatever pings an app-open cadence happens to produce). The architecture choice is **no longer open** — resolved 2026-07-24 to a **host-side `WorkManager`** (Kotlin, engine-independent, survives backgrounding); an in-app engine-side timer was rejected as foreground-only, which defeats the purpose of passive sessions. See `docs/PLAN-closeout.md` Phase 4.
- **Discipline variants** (`GYM_DISCIPLINES = strength|yoga|pilates|dance`, `user_gyms.discipline`, 0033/0034): `DISCIPLINE_ACTIVITY_TYPE` maps discipline → activity type 1:1 by name (identity mapping today, kept as a named seam). Reward then flows through the normal `ACTIVITY_STAT_AXIS`: `strength→STR`, `yoga→WIS`, `pilates→WIS`, `dance→DEX`. Discipline resolution precedence on venue creation: auto-detected (yoga/dance only, via Google Places `primaryType`) **wins over** the user's declared discipline, which wins over the `strength` default. Pilates is undetectable via Places (labeled generic `gym`) — it only ever lands via user declaration.
- **Venue validation** (`validateVenue.ts`, hybrid, best-effort, NEVER throws — registration is best-effort so a broken provider degrades the result, never fails the caller): Tier 1 Overpass (keyless/free, OSM tags `leisure=fitness_centre|amenity=gym|sport=fitness` within 150m) → Tier 2 Google Places `searchNearby` (env-gated on `PLACES_API_KEY`, self-skips if unset) → escalates only if Tier 1 didn't validate. Each tier returns `true`(found)/`false`(clean negative)/`null`(couldn't answer — never counted as a negative). Result: any `true` → `"validated"`; no `true` but at least one clean `false` → `"unverified"`; every tier errored/skipped → `"pending"` (we know nothing). 3s per-request timeout via `AbortController`. A `"rejected"` (admin-only verdict) venue is intentionally never reused for a new registration at the same coords — a new registration there gets a fresh venue that re-runs auto-validation, since a legitimately different gym could have opened where a bogus one was rejected.
- **registerUserGym** (`register_user_gym` RPC, migration 0039, venue/link split): `user_gyms` is a thin per-user link (id, user_id, venue_id) to the shared `gym_venues` table (coords, name, validation, discipline) — one venue row reused across every user who registers there, so validating one user's gym validates it for everyone linked. `GymService.registerGym` resolves the venue (reuse-nearby-inside-fence via `findNearbyVenue`, else create), dedups an already-linked venue before calling the RPC (repeat register of an owned gym never 409s even at the cap), and the RPC itself does the per-user cap check (`MAX_GYMS_PER_USER = 5`) + link insert atomically under a per-user advisory lock, returning an empty set when the cap is reached (mapped to `null` → route 409). A unique-constraint race (23505, concurrent register of the same venue) is caught and re-read as a normal dedup rather than propagated as an error. Deleting a link never deletes the venue (survives for every other user linked to it).
- **Admin resolution route exists and works today (verified 2026-07-23):** `POST /admin/gyms/:id/resolve` (`routes/gyms.ts`) is guarded by `isAdminRequest` (shared admin key, not `requireUser` — there's no user identity on this call) and calls `gymService.resolveValidation(id, status)` to set a venue to `validated`/`rejected`. **The queue has no listing consumer yet** — no route enumerates venues sitting in `pending`/`unverified`/`review_requested`; an operator needs the venue id from elsewhere (direct DB query) until a dashboard exists. Intended consumer: a future admin dashboard will list the queue and drive this same route — the resolve endpoint itself is already correct and does not need to change when that ships.

### Location capture — foreground-service model (AGREED — **NOT YET IMPLEMENTED**)

> **Agreed direction, no code yet (decided 2026-07-23).** Host-only capability, engine-neutral
> (the host owns location regardless of the client engine). Today both live-recorded
> sources capture location **foreground-only** via `LocationManager` with no background service
> (`RunSessionTracker.kt` flags this as a known ceiling), so Android 10+ stops delivering fixes
> within seconds of the app being backgrounded/killed — a `tracked_run` only samples while the app
> is on screen, and gym pings only fire on app-open.

- **Fix = one shared location-typed foreground service**, started while the app is in the
  foreground when the user begins a run **or** a gym session. It keeps location flowing while the
  app is backgrounded / screen-off for that session's duration, then stops.
- **Permission posture (the hard constraint): no new runtime permission the user must approve.**
  Reuses the already-granted `ACCESS_FINE_LOCATION`; adds only the **install-time / auto-granted**
  `FOREGROUND_SERVICE` + `FOREGROUND_SERVICE_LOCATION` manifest permissions (no dialog).
  `ACCESS_BACKGROUND_LOCATION` (the "Allow all the time" prompt) is deliberately **not** added —
  an FGS started while the app is visible does not need it. Cost accepted: a **persistent
  notification while a session is active** (mandatory for any FGS; standard for run/gym trackers,
  and it clears when the session ends).
**`tracked_run` lifecycle** — the FGS spans one run. Only *when* fixes are collected changes;
the evidence-bundle shape and every server stage are untouched.

1. **Start** (foreground) — user taps start → `RunSessionTracker.start()` starts the FGS.
2. **Capture** (foreground *or* backgrounded) — fixes accumulate for the session's duration;
   screen-off / app-backgrounded no longer stops sampling. This is the only stage this rule adds.
3. **Stop** → `RunSessionTracker.stop()` stops the FGS and returns the raw window + samples, which
   assemble into a `tracked_run` bundle.
4. **Ingest** — the bundle enters the standard pipeline **unchanged**. → *see the already-expanded
   stages:* adjacent-fragment merge (*Ingest pipeline* step 4) and route scoring
   (*Verification — checker contracts*, `TrackConsistencyChecker`).

**`tracked_gym_session` lifecycle** — a shift from fully-passive to a **user-started** session.
The FGS session is *additive* client-side capture; the existing app-open passive ping and the
server finalizer are unchanged and remain the fallback.

1. **Start** (foreground) — user taps "I'm at the gym" → the **same** FGS starts. (This is the
   one behavior change vs. today's fully-passive model — it needs the tap, the price of the
   no-new-permission constraint. It is **not** app-closed / passive capture.)
2. **Ping** (foreground *or* backgrounded) — a single-shot fix every `GYM_PING_INTERVAL_MS` →
   `POST /gyms/ping`, now continuing while backgrounded *during the session*. The legacy
   **app-open passive ping** stays as additive/fallback coverage, feeding the same `gym_pings`.
3. **Group + finalize** — server-side, **unchanged**. → *see the already-expanded stage:*
   *Gyms — session model* (passive ping model, `findMatchingGym`, departure detection /
   `AWAY_PINGS_TO_CLOSE`, session grouping, credited duration). This rule changes only the
   client-side ping cadence feeding that stage, never its logic.
4. **Ingest** — each finalized session becomes a synthetic `tracked_gym_session` bundle → standard
   pipeline. → *see:* *Gyms — session model* (synthetic bundle assembly) and *Ingest pipeline*
   step 5 (gym-discipline override).

- **Rejected alternatives:** geofencing or periodic background location (the "right" tool for
  truly app-*closed* gym detection) both require `ACCESS_BACKGROUND_LOCATION` — a new runtime
  prompt, which violates the no-new-permission rule; and a permanent 24/7 FGS (always-on
  notification, battery) is bad UX for occasional pings. True app-closed pinging is therefore
  **out of scope** under the no-new-permission constraint.
- **Scope of the build (host-only, not started):** a new foreground `Service`, the two manifest
  permission lines, and wiring `RunSessionTracker` + the gym-ping starter to start/stop it. iOS
  port: `CLLocationManager` background location updates are the equivalent surface behind
  `HostBridge`.

**Crash / battery durability — persist-to-disk + recover on relaunch (AGREED — NOT YET
IMPLEMENTED, scenario review 2026-07-23).** The FGS above keeps a *backgrounded* run alive, but a
*killed* process (battery death, OS reclaim) still loses everything, because samples live only in
`RunSessionTracker`'s in-memory list until `stop()` — unlike gym pings, which are already durable
server-side (each is `POST /gyms/ping`'d as it happens, then finalized by the staleness sweep).

- **Fix (host-only, no server change):** write run/cycle samples **to disk as they're collected**;
  on next app launch, detect an unfinished session, assemble it into a bundle, and upload it into
  the normal ingest pipeline (late, but not lost).
- **Accepted tradeoff:** nothing is credited until the app reopens — chosen over mid-run streaming
  (a new endpoint + a run finalizer) because it's the lighter fix and the FGS already covers the
  common backgrounded case; only an outright process kill needs recovery.
- Resolves the "battery dies mid-workout → still get credit for what was recorded" scenario for
  runs (gym sessions already survive it).

**`tracked_cycle` live mode (AGREED — added to scope, scenario review 2026-07-23).** An on-device
tracked cycling mode alongside `tracked_run`, sharing the **same** FGS + disk-persistence capture.

- Activity type is **declared** (the mode the user starts), **never server-inferred** — there is
  no speed/cadence classifier (rejected as weak-trust + a new abuse surface). Cycling otherwise
  arrives pre-labelled from Health Connect / Strava; the scenario's "the app tells cycling apart
  from running" is the *source* labelling it, not us.
- Needs its own `SourceProfile` (mirror `tracked_run`: `{track-consistency:1, accel-presence:1,
  metric-rate:0}`, `coverageCap:true`) plus the two new anti-spoof checkers, and a **cycling-tuned
  energy-commensurability bound** in `TrackConsistencyChecker` — the running `plausibleKcal = 150 +
  300 × km` under-counts a ride (cycling covers far more distance per kcal), so it must not
  false-penalize honest cyclists. `cycling → CON` is already in `ACTIVITY_STAT_AXIS`.

### Health Connect background sync — permission-staged onboarding (AGREED — **NOT YET IMPLEMENTED**, task `4e`)

> **Agreed direction, no code yet (folded into Phase 4 as task 4e, 2026-07-27).** Host-only,
> engine-neutral — closes the fairness gap the *Daily XP cap* section flags: `[HealthConnectReader]`
> only reads on app launch today, so `activities.created_at` (the receipt-day XP-cap anchor) can
> trail `started_at` by days for a user who doesn't reopen the app. A background sync keeps receipt
> day close to performed day for the common case, without weakening the receipt-day anchor itself
> (still `created_at`, still forgery-resistant).

- **Permission sequencing is staged across three moments, not requested all at once:**
  1. **App launch** — request the normal Health Connect read set (`requiredPermissions` today:
     exercise sessions, active calories, heart rate, distance, steps). This is the existing
     foreground grant; nothing changes here.
  2. **After onboarding** — once the user has seen the app work (not at first launch, alongside a
     wall of other prompts), show a short explanation of the benefit: *"We'll automatically detect
     your activities even when the app isn't open."* This is a UI moment, not a permission dialog —
     it exists to raise the odds of the *next* step's system prompt being accepted, not to request
     anything itself.
  3. **Only then** — request Health Connect's background-read permission
     (`HealthPermission.PERMISSION_READ_HEALTH_DATA_IN_BACKGROUND`), a separate grant from the
     foreground read set above. **Denial degrades, it doesn't block:** a user who declines keeps
     the existing foreground-only behavior (app-launch re-sync) exactly as it works today — the
     background job below simply never has data to read and self-skips, never crash-loops or nags.
  - Rationale for staging over asking everything at first launch: bundling a background-access
    prompt into initial onboarding (before the user has any reason to trust the app) measurably
    depresses grant rates for the whole permission, and Health Connect denials cannot be re-prompted
    without the user leaving the app to change it in Settings — so the *first* ask is the one that
    matters most.
- **Scheduling — reuses 4a's periodic `WorkManager`, not a second scheduler.** A **30–60 minute**
  cadence (looser than 4a's 5-minute gym-ping worker; background health sync has no live-session
  responsiveness requirement, just "don't let receipt day drift from performed day by more than
  about an hour"). Exact interval is a host-side tuning constant, not a gameconfig value — it has no
  server-side effect to tune remotely.
- **Each run reads only NEW data, not the existing foreground path's full re-scan.** Today's
  `readWorkoutsInWindow()` re-reads the whole 7-day `LOOKBACK` window on every call and relies on
  server-side overlap dedup to make that safe — deliberate for the foreground app-launch path (see
  its `ponytail:` note), but wrong for a job firing every 30–60 minutes: re-uploading the same
  multi-day window dozens of times a day is pure waste even though dedup keeps it *correct*. The
  background worker instead reads **only what changed since its last successful run**, via Health
  Connect's Changes API (`client.getChangesToken` / `client.getChanges`) token, or (if the changes
  token proves unreliable across process death / HC data-restore events) a plain stored
  last-sync-timestamp watermark — either is a NEW, separate read path from `readWorkoutsInWindow()`,
  not a modification to it, since the two callers have genuinely different correctness needs (a
  human reopening the app after days away wants the full window; a background tick firing hourly
  wants only the delta).
- **Failure handling inherits 4b/4c** (permission revocation without crash-looping the scheduler;
  doze/battery-respecting primitive) rather than defining its own — see task 4e in
  `docs/PLAN-closeout.md`.
- Per the host testing convention, the changes-token/watermark bookkeeping (whatever it turns out
  to be) is a pure function candidate for `android-host/hostlogic/` with a JUnit4 test; the actual
  Health Connect Changes API call and worker lifecycle are dogfooded via the dev driver, not
  unit-tested (same split as the rest of `HealthConnectReader`).

### Strava

- `StravaSyncService.syncUser(userId)`: requires an existing `stravaConnectionStore` connection — throws `Error("Strava not connected")` with `statusCode:400` if absent.
- `afterEpoch` = `connection.syncedAfter` converted to epoch seconds (Strava's `after` filter), or `undefined` for a first-ever sync (pulls the recent window with no lower bound).
- Pages through `stravaClient.listActivities` at `PAGE_SIZE=30`, up to `MAX_PAGES=20` hard stop (a misbehaving cursor can't loop forever). Strava returns newest-first, so **every page of a backlog must be walked before advancing the cursor** — advancing off page 1 alone would silently and permanently skip everything below page 1's oldest activity on the next sync (the bug this pagination fix closes). Loop stops early once a page comes back shorter than `PAGE_SIZE` (no wasted probe page).
- Each fetched activity → `mapStravaActivity` (pure map, no I/O/scoring) → `ingestActivity` (the SAME pipeline `POST /activities` uses). `mapStravaActivity`: `sport_type` (falling back to legacy `type`) mapped via a fixed table into the shared `ActivityType` vocabulary, unmapped sports bucket into generic `"workout"` (never an off-list string); Strava's own `manual` flag surfaces verbatim as `manualEntryFlag` (untrusted signal, not laundered away — hard-rejected by `manual-entry` in the strava profile); Strava's own `flagged` verdict surfaces as `providerFlagged` (hard gate — the ONE plausibility signal trusted for a pull source, since Strava computed it, not the client); `activityId` is a deterministic UUIDv5-shaped hash of `strava:{id}` so re-syncing the same activity upserts, not duplicates; `gpsContext` from `start_latlng` (`accuracyMeters:0` placeholder — no accuracy given by the summary endpoint) or `null` if absent/empty.
- Result buckets summed into `StravaSyncSummary {fetched, accepted, duplicates, ineligible, rejected}` from each `ingestActivity` call's outcome (`result.reason === "duplicate_workout" | "ineligible_workout"` bucket accordingly; anything else not accepted buckets as `rejected`).
- Cursor (`syncedAfter`) only ever advances FORWARD: tracks the newest `startedAt` seen across ALL pages, and only persists via `setCursor` if it actually changed from the stored value.

### Eligibility (NOT YET IMPLEMENTED)

- `EligibilityFilter.check(bundle)`: Stage 4 of ingest, runs BEFORE `VerificationEngine` scores trust — decides whether a workout is countable AT ALL (vs. how trustworthy it is). By design a **pass-through stub** today: the rule catalog (`WorkoutEligibilityRule[]`) is empty, so every bundle is eligible. Contract for when rules land: each rule declares `appliesTo(bundle)` (scoped to a source/activityType) + `evaluate(bundle)`; the filter runs only applicable rules and returns the FIRST failure (`{eligible:false, reason}`); `{eligible:true}` only when no applicable rule fails. Open/Closed — a new rule is a new class registered in the constructor array, never an edit to the filter's loop.

---

## Game

### Two-phase digestion (activities lifecycle)

- **Ingest** (`ingestActivity`, `POST /activities`) never credits a character. A verified bundle is scored, and on accept the precomputed award is written to `activities.awarded` with `registered=false` — a *pending* row.
- **Register** (`POST /activities/register` → `register_pending_activities(p_user_id)` RPC) is the only path that credits a character. It locks the user's `accepted=true, registered=false` rows `FOR UPDATE`, sums `awarded.xp` / `awarded.gold` / `awarded.stats.{str,dex,con,wis}` across them, applies the sum to `characters` in one `UPDATE`, then flips those rows `registered=true`.
- Exactly-once by construction: a re-run (repeat button press, retried request) finds zero rows still `registered=false` under its own lock and returns `{count: 0, applied: {xp:0, gold:0, stats: all-zero}}` — a true no-op, not a partial re-apply.
- No pending rows at all → same zero-result shape, not an error.
- Split of responsibility: the **client only triggers** register (no payload, no computed values); the **server computes** every number in `applied` and `character`. `register_pending_activities` is revoked from `public/anon/authenticated` — service-role only.
- `activities.awarded` shape (canonical since migration `0012_unify_award_stats.sql`): `{ "xp": int, "gold": int, "stats": { "str": int, "dex": int, "con": int, "wis": int } }`. `register`'s `applied` field mirrors this exact shape.

### Ingest pipeline (`ingestActivity`)

Order of operations, each a hard gate before the next runs:

1. **Dedup** (`findDuplicateActivity`) — two independent checks, either blocks ingest with `{accepted:false, reason:"duplicate_workout"}`:
   - *Native-id*: same `source_workout_id`, `accepted=true`, different `activity_id` → same source record re-read in a window.
   - *Time-overlap*: another `accepted=true` row for the user whose `[started_at, ended_at)` overlaps this bundle's window. Cross-type by default (a watch's `hiit` and Health Connect's generic `workout` for one session are one workout); the synthetic 24h `steps` row is partitioned out and only dedupes against other `steps` rows, since it legitimately spans every real workout in its day.
2. **Eligibility** (`EligibilityFilter`) — pass-through today; a future per-(source,type) countability gate. Ineligible → `{accepted:false, reason:"ineligible_workout"}`.
3. **Trust scoring** (`VerificationEngine`) — independent server re-score of the evidence. `trust.rejected=true` → the row is still recorded (`recordActivity`, `awarded=null`) for audit, but `accepted=false` and no award exists.
4. **`tracked_run` fragment merge** (only for `bundle.source === "tracked_run"`, only on accept) — before recording a new row:
   - Looks for a prior `tracked_run` row for this user that is `accepted=true, registered=false`, whose `ended_at` falls in `[incoming.startedAt − 5min, incoming.startedAt]` (`RUN_MERGE_GAP_MS = 5 * 60 * 1000`). This window catches a fumbled Stop→Start on one physical run; a true overlap (run still in progress) was already caught by step 1.
   - `mergeRunBundles(prior, incoming)`: keeps `prior`'s `activityId`/`sourceWorkoutId`/`startedAt` (identity), takes `incoming.endedAt`, sums additive metrics (`activeEnergyKcal`, `durationSeconds`, `distanceMeters`, `runTrack.sampleCount/trackedDistanceMeters/movingSeconds`), and duration-weight-averages `avgHeartRate` (never sums it — summing would push HR outside its plausibility band).
   - The merged bundle is re-scored and re-translated from scratch (not a delta) so it earns exactly one per-activity stat point and one correctly-summed XP award for what is really one run.
   - The DB write is guarded: `UPDATE activities SET ended_at, evidence_bundle, trust_score, awarded WHERE activity_id = prior AND registered = false`. Zero rows updated (prior got registered between read and write) or an exclusion-constraint violation (`23P01`, extending the window collides with another accepted row) both mean "can't merge" — falls through to recording the fragment as its own new row rather than losing data.
   - Success emits an `run_fragment_merged` analytics event (`intoActivityId`, `fragmentActivityId`).
   - A merge-time reject (re-scored evidence somehow fails trust) is treated as unexpected and also falls through, rather than destroying the prior valid accept.
5. **Gym-discipline override** (`tracked_gym_session` with `gpsContext` only) — if the session's GPS matches a gym whose `validationStatus === "validated"`, the *venue's* discipline (not the client's claimed `activityType`) drives the XP/stat variant via `DISCIPLINE_ACTIVITY_TYPE`. An unvalidated match keeps the client's claim (trust-equivalent to no match at all) — this is a trust *upgrade*, so it only engages once the venue is backend-confirmed.
6. **Record + award** — `kinetic.translate(bundle, effectiveActivityType)` precomputes the pending award; `recordActivity` **upserts** on `activity_id` (`onConflict: "activity_id"`) with `registered:false`. `activityId` (client-generated UUID) is the idempotency key: a retried request with the same id lands as the same row, not a duplicate.
   - A race on this insert (two concurrent uploads of the same native id, or overlapping windows from two sources both passing the pre-check before either commits) surfaces as `23505` (unique) or `23P01` (exclusion) — both are caught and mapped to the same `duplicate_workout` outcome as the soft dedup check, not an error.
7. **No referral payout at ingest.** As of migration `0043` the referral fires at *register* time, not here — the gate requires a `registered=true` activity, which an accepted-but-pending ingest can never satisfy. The `creditReferralIfPending` call therefore lives in `POST /activities/register`, not `ingestActivity` (see Guild + referral section, `credit_referral`).

### Daily XP cap (migration `0028_daily_xp_cap.sql`; window re-keyed by `0046` — **SHIPPED**)

- Enforced by a Postgres `BEFORE INSERT OR UPDATE OF awarded` trigger (`clamp_daily_awarded_xp`) on `activities` — not application code — because ingest requests are independent transactions (PostgREST) and a TS-level sum-then-clamp would race under concurrent uploads.
- No-ops (`return new` unchanged) when `awarded IS NULL` or `accepted IS NOT true`.
- Serializes per user via `pg_advisory_xact_lock(hashtext('daily_xp:' || user_id))` so two concurrent ingests for the same user can't both read the same "spent so far" and both mint (cross-user hash collisions are harmless — just spurious serialization).
- Cap comes from `gameconfig.kinetic.formula.maxXpPerDay`, default fallback **2000** if the row/key is missing or non-numeric (gameconfig is an untrusted, live-tunable surface). This is a *second* ceiling layered on top of `0027`'s **1000 XP** per-activity cap — the per-activity cap alone doesn't stop many non-overlapping windows each claiming 1000.
- "Spent" = `SUM(awarded->>'xp')` over this user's `accepted=true, awarded IS NOT NULL` rows **within the user's local RECEIPT day**, **excluding the row being written** (`activity_id <> new.activity_id`) — the self-exclusion exists so an idempotent upsert-retry (which fires this trigger via the `UPDATE OF awarded` path, e.g. the run-fragment merge's UPDATE) doesn't count its own prior value against itself.
- **The window is a local calendar day (midnight→midnight in `profiles.timezone`) keyed on server receipt time (`created_at`) — changed from a rolling 24h window by `0046`.** The rolling window was wrong on its own terms: a workout Mon 19:00 and another Tue 08:00 are 13h apart, so the Tuesday session was clamped against Monday's spend despite being a different day. A cap called "per day" runs midnight to midnight.
- **It stays keyed on `created_at`, never the client-controlled `started_at`** — this is the anti-backdate anchor. The server clock is the only bound a forged bundle cannot move: a client fabricating 20 backdated days and uploading them at once puts them all in ONE receipt day, clamped to a single day's budget. Note this is deliberately the **opposite** column from the collapse/axis-cap window in the same trigger (see next section) — fairness vs. forgery-resistance, two different jobs. The asymmetry is intentional and must not be "fixed."
- **Residual (accepted, `0046`):** midnight-to-midnight is strictly weaker than rolling-24h at the boundary — 2000 XP at 23:50 plus 2000 at 00:10 is now legal. That is what "per day" means once you commit to calendar days, and the receipt-day anchor still caps a forged multi-day upload at one day's budget, which is the property that matters against forgery.
- **Dependency:** background Health Connect sync (host-side, **NOT YET IMPLEMENTED** — see § *Health Connect background sync — permission-staged onboarding*, task `4e`) is what keeps `created_at` near `started_at` for legitimate users. Until it ships, a user who batch-syncs two training days on one app-open still collides against a single receipt day's budget — a known, documented interim window, not a design intent.
- Clamp behavior when `xp > remaining budget` (a clamp, not a reject — the workout still counts toward the weekly target):
  - `xp` → `floor(remaining)`.
  - `gold` → `floor(gold * remaining / xp)` (proportional scale-down).
  - if `remaining == 0`: all four stats zeroed too, so a capped-out farmer can't keep minting stat points at 0 XP.
  - `awarded.dailyCapApplied = true` added for observability; `register_pending_activities` reads only `xp/gold/stats` and ignores this flag.

### Per-type reward collapse + per-axis stat cap (migrations `0045` + `0046` — **SHIPPED**, all three Mechanisms)

> **Agreed 2026-07-23, built 2026-07-27 (Phase 1 tasks 1b/1c).** Supersedes an earlier "one
> rewarded activity per `activity_type` per day, 2nd rejected as duplicate" framing — that was
> wrong on two counts: the real key for stat points is the **stat axis** (coarser than type), and
> nothing here is a reject. This is **two clamps with different keys**, siblings of the
> `maxXpPerDay` cap (0028), **not** a dedup gate — implemented by extending the same
> `clamp_daily_awarded_xp` trigger (0028) with the new logic, one `create or replace`, not a
> second trigger (firing-order/ordering reasons; see `docs/PLAN-closeout.md`).

Two independent mechanisms, both scoped to the user's **local day** (see *Day scope* below). No
workout is ever rejected by this rule — every one stays `accepted=true` and recorded; only the
**awarded XP/gold/stats** are reduced (to 0 for the surplus).

**Mechanism 1 — per-`activity_type` reward collapse (XP/gold).** Within one `activity_type` on one
local day, only the **single best** workout rewards: the day's XP/gold for that type is the
`max` over that day's same-type workouts, **not** the sum. Because `gold`/stat scale with XP, "the
max" is the whole award of the highest-XP workout of that type that day. A later, lower same-type
workout adds **0** XP/gold; a later, higher one tops the day up to its new max (a high-water mark).
*Example:* two strength workouts collapse to one — `max(reward₁, reward₂)`, one STR, one weekly
credit (see Mechanism 3).

**Mechanism 2 — per-axis stat cap.** At most **N** stat points per stat axis (STR/DEX/CON/WIS) per
local day; **N comes from gameconfig** (default **1**, Zod-validated with an in-code fallback — a
tunable like `maxXpPerDay`, never hardcoded). The axis map is `ACTIVITY_STAT_AXIS`
(`strength→STR`, `hiit/dance→DEX`, `running/walking/cycling/steps→CON`, `yoga/pilates/mobility→WIS`,
`workout→null`). Two **different** types on the **same** axis (run + cycling → CON; hiit + dance →
DEX) each earn their own XP/gold and their own weekly credit, but the shared axis yields only **one**
stat point that day. `workout` (null axis) grants no stat and consumes no axis slot.

**Mechanism 3 — weekly credit is per distinct `(activity_type, local_day)` — SHIPPED.** Same type
twice in a day = **one** weekly credit (they collapsed); different types = separate credits even on
the same axis. `countAcceptedActivities` (`dailyTick.ts`) no longer counts raw `accepted` rows: it
selects `(activity_type, started_at)` over the week window and returns the size of the distinct
`(activity_type, localDateOf(started_at, profiles.timezone))` set. This closes the interim gap
`0045` shipped with, where a same-type second workout that collapsed to 0 XP/gold still bought a
fresh weekly credit.

- Deduped in **TypeScript**, not SQL — no RPC, no migration. Since `0046` re-keyed the collapse to
  `started_at`, the week window and the day bucket are the same column, and `localDateOf`
  (`weekWindow.ts`, already unit-tested) is the exact JS twin of `(ts at time zone tz)::date`.
  Per-user weekly row volume is under ~25.
- **Both** call sites count this way: `evaluateClosedWeek` and `reconcileGraceWindow`. The second
  matters — it re-counts a penalized week to decide a reversal, so a stale count there would
  silently mis-refund.
- **Drift risk, live:** SQL (`0046`) and TS (`localDateOf`) now each own a local-day definition. They
  must agree; a DST-boundary test is the guard.

| Same day | XP / gold | Stat | Weekly credits |
|---|---|---|---|
| strength + strength | `max(s1, s2)` | 1 STR | 1 |
| run + run | `max(r1, r2)` | 1 CON | 1 |
| run + cycling | run + cycling (both) | 1 CON | 2 |
| hiit + dance | both | 1 DEX | 2 |
| strength + run | both | 1 STR + 1 CON | 2 |

*(All three columns are current, shipped behavior. Rows on **different** days never collapse and
always earn separate credits, whenever they sync.)*

- **Day scope — keyed on the PERFORMED day (`started_at`), per user local day
  (`profiles.timezone`). Changed from `created_at` by `0046`.** The original receipt-time keying was
  wrong in a common case: Health Connect batch-syncs up to 7 days on app open, so a user who trained
  Monday and Tuesday but opened the app Wednesday had both workouts land in one receipt-day bucket
  and the second collapsed to 0 XP. Two workouts on two calendar days are two workouts, however late
  they sync.
- **This is deliberately the opposite column from the daily XP cap in the same trigger.** Collapse and
  axis cap key on `started_at` because that is what *fairness* means; the XP cap keys on `created_at`
  because that is the only clock a client cannot forge. Two windows, two jobs. The asymmetry is
  intentional and documented in `0046`'s header — do not "fix" it into consistency.
- **Residual (accepted, `0046`):** a client backdating `started_at` across N fake days can claim N
  axis stat points instead of 1. Bounded by the receipt-day XP budget — once exhausted, the trigger
  zeroes all stats regardless of `started_at`. Accepted: denying a legitimate batch-syncing user
  their real second workout is the worse failure mode.
- **Enforcement (server-side, race-safe like 0028).** Both clamps are day-window high-water-marks
  computed against the user's already-awarded rows for the local day, **excluding the row being
  written** (`activity_id <> new.activity_id`, so an idempotent upsert / run-merge doesn't count
  its own prior value). A Postgres `BEFORE INSERT OR UPDATE OF awarded` trigger under a per-user
  `pg_advisory_xact_lock` — same posture as the daily-XP cap — because concurrent PostgREST
  ingests race a TS-level read-then-write.
  - *XP/gold:* award only the amount by which this workout exceeds the best same-`(user,
    activity_type, local_day)` reward already granted; `≤ 0 → 0`. Net day total for the type = the max.
  - *Stat:* grant the axis point only if the user has `< N` stat points on that axis today; else 0.
  - Uses the **effective** `activity_type`/axis (post gym-discipline override, step 5), never the
    client's raw claim.
- **Schema need — turned out unnecessary; no `stat_axis` column was added.** The original plan
  above assumed the axis map needed surfacing in Postgres. The shipped trigger avoids that: it
  reads the row's already-stored `activity_type` (already the EFFECTIVE, post-gym-discipline-
  override type — `ingestActivity` writes the resolved type) and `awarded->stats` (already
  computed from that effective type by `kinetic.translate`), so per-axis sums fall out of a
  `sum(...) filter (where activity_type <> 'steps')` over the four `awarded->stats` keys directly
  — no separate axis column or mapping table needed. `activity_type` + a derived `local_day` cover
  Mechanisms 1 and 3 with no schema addition.
- **Stacking:** runs **before** `maxXpPerActivity` (0027) and `maxXpPerDay` (0028) — confirmed as
  shipped: the trigger computes the Mechanism 1 collapse first, then applies the 0028 receipt-day
  remaining-budget clamp on top of the collapsed value in the same pass. Two window scans, two
  columns (`started_at` for the collapse, `created_at` for the budget), one advisory lock.
- **`steps` precedence — RESOLVED as recommended.** The synthetic 24h `steps` aggregate maps to
  CON. Shipped behavior: `steps` is exempt from the axis cap in both directions — a `steps` row's
  own stat is never capped, and it never counts toward the axis-cap sum for other rows (`activity_type
  <> 'steps'` filter on the spent-sum), so it can never preempt a real CON workout's axis point.
- **Observability — shipped.** `awarded.typeCollapsed` / `awarded.axisStatCapped` flags (alongside
  the existing `dailyCapApplied`) are stamped whenever each clamp actually changes the stored
  value; `register_pending_activities` reads only `xp/gold/stats` and ignores them, as designed.

### One claim per native workout, across ALL accounts (migration `0044` — **SHIPPED**)

> **Agreed 2026-07-23, built 2026-07-27 (Phase 1 task 1a).** Closes the two-accounts-one-device
> double-claim: the native-id dedup was scoped `per user` (both the `findDuplicateActivity`
> check and the `0014` unique index carried `user_id`), so two logins on one phone reading the
> **same** provider record could each get credit. The fix makes the native-id key **global**.

- **A provider-native workout record is claimable exactly once across every account — first
  sync wins.** Health Connect is an OS-level shared store: two accounts on one device read the
  **same** HC record (same UID); likewise two accounts pointed at one Strava account read the
  same activity id. The account that syncs it **first** gets the pending award; any later account
  claiming the same `source_workout_id` is **rejected as a duplicate** (`{accepted:false,
  reason:"duplicate_workout"}`) and does **not** count toward that account's weekly target.
- **Only the native-id key goes global. Time-overlap dedup stays per-user** (step 1, check 2) —
  two *different* real people legitimately train at the same wall-clock time, so a global
  time-overlap check would block everyone who works out at 6pm. This rule is purely "the same
  provider record can't be spent twice," which native ids already identify uniquely.
- **Scope = sources that carry a provider-native `source_workout_id`** (`health_connect`,
  `strava`, and future Oura). It does **not** reach `manual`, `tracked_run`, or
  `tracked_gym_session` — their ids are client/server-minted per account, so there is no shared
  key to dedup on. Cross-account dedup there would require a *trusted* device identity, which the
  client can't supply (the client is untrusted); out of scope for this rule.
- **No device identifier is needed or used.** The native id is globally unique per provider, so
  keying on `source_workout_id` alone expresses "one real workout, one claim" without trusting any
  client-reported device id (see the shipped key shape note below — no `source` column needed).
- **Enforcement, as shipped — mirrors the existing native-id path, minus the user scope:**
  - **App (`findDuplicateActivity`):** dropped `.eq("user_id", userId)` from check 1 (the
    `source_workout_id` branch only — the time-overlap branch, check 2, stays user-scoped). Also
    added a date bound not in the original plan: `.gt("started_at", recentCutoff)` where
    `recentCutoff = now - 14d` (2× the Health Connect lookback window of 7d — no client can
    re-fetch a workout older than its own lookback, so a stale accepted row can never again be a
    replay target). This keeps the common-case SELECT cheap; it's a graceful early-out, not the
    correctness guarantee — the DB constraint below is that.
  - **DB:** replaces the `0014` index `activities_user_source_workout_id_unique
    (user_id, source_workout_id)` with a global partial unique index on
    **`(source_workout_id) WHERE source_workout_id IS NOT NULL AND accepted = true`** — **one
    deviation from the original plan text above: the shipped key has NO `source` column**, unlike
    the originally-proposed `(source, source_workout_id)`. This is safe because of the *Scope*
    bullet above: only provider-native sources ever populate `source_workout_id` at all
    (`manual`/`tracked_run`/`tracked_gym_session` never set it), and Health Connect's UUID-shaped
    ids and Strava's sequential integers don't collide in practice — but it does mean the
    uniqueness guarantee is technically global-across-all-providers, not per-provider-namespaced.
    Unbounded by date (a partial-index predicate must be static; correctness must hold for all
    time — the app-side 14d bound above is a performance early-out only, not part of the
    guarantee). **Bonus fix beyond the original plan:** the predicate is `accepted = true`, not
    unconditional, which also fixes an existing quirk where a once-rejected sync of a native id
    could never be re-submitted and accepted later (the old `0014` index constrained ALL rows
    including rejected ones) — a rejected row no longer occupies a permanent slot. A concurrent
    race (two accepted inserts of the same id both pass the pre-check) surfaces as `23505` on the
    loser, mapped to the same `duplicate_workout` outcome the pre-check returns — the existing
    `isUniqueViolation` catch in `recordActivity` already handles it.
  - **Migration caveat (unchanged from plan, still applies going forward):** `CREATE UNIQUE INDEX`
    would fail if two accounts had *already* claimed one id in prod before this shipped; verified
    clean pre-launch.
- **Accepted tradeoff — guessable ids (decided 2026-07-23).** Strava activity ids are sequential
  integers, so global first-claim-wins is a grief vector: a forged `source=strava,
  source_workout_id=<victim's id>` accepted first denies the victim credit for that one workout.
  Judged **low severity** (attacker must know the victim's id, pre-claim it, and forge a bundle
  that clears the strava profile; cost to the victim is a single workout the weekly loop
  tolerates) and **accepted** rather than gated behind trusted-sync provenance. Health Connect —
  the actual same-device shared-store case — uses unguessable UUIDs and carries no such risk. If
  abuse ever appears, the upgrade path is to honor a native id as a cross-account key only when it
  arrived via an authenticated server-side sync, not a raw client POST.
- **Open implementation detail — still unresolved as shipped:** the code reuses `duplicate_workout`
  for both cases (confirmed: no distinct `cross_account_duplicate` reason exists in
  `ingestActivity.ts`). Adding one for analytics (recommended — a same-account window re-read and
  a rival account racing for the same record are different signals worth counting apart) remains
  open, not done.

### Scope limits & Tier-2 deferrals (scenario review 2026-07-23)

Resolutions for scenarios that are **not** bugs against the spec — either accepted limits or
deliberate deferrals — so they don't get mistaken for missing behavior later.

- **No account ban exists — by design.** Trust rejects an individual *bundle*, never an account;
  the "elite athlete falsely banned" scenario can't occur. The `metric-rate` ceilings are lenient
  **anti-absurdity** gates (60 km/h, 25 kcal/min), and `maxXpPerActivity` / `maxXpPerDay` bound the
  reward regardless — so a physically-implausible-but-under-ceiling claim (e.g. a 3-min mile ≈
  32 km/h) is *accepted and XP-capped*, not rejected. Tightening to per-`(source,type)` realistic
  ceilings is a **Tier-2** refinement, not MVP.
- **Shared-phone mis-attribution is an accepted limit.** Workouts credit the currently-logged-in
  account. The cross-account native-id rule (above) stops the *double-claim* (same provider record,
  two accounts on one phone), but nothing attributes a workout to the "right" person — there is no
  per-workout biometric identity, and won't be for MVP.
- **iOS / Apple Watch deferred.** Android-first; an Apple Watch does not feed Android Health
  Connect, so "Apple Watch just works" is false for the MVP. Garmin *does* (it's in
  `SUPPORTED_INTEGRATOR_PACKAGES`). iOS/HealthKit is the deferred port behind `HostBridge`.
- **Deferred to Tier-2:** per-activity **dispute / appeal** flow (admin *venue* review exists;
  there is no per-activity or "my gym was wrongly marked unverified" appeal yet); cross-day
  **replay detection**; **sybil / multi-account** farming beyond the referral verification gate +
  cap. Documented, not built.
- **`GeofenceNegativeChecker`'s generic venue cross-reference is OUT OF SCOPE (decided 2026-07-23),
  not merely deferred.** Its original Phase-3 TODO ("cross-reference `gpsContext` against required
  venue geofences") is superseded: real venue-scoped geofencing already exists and is live via the
  `gym_venues` table + `GymPresenceChecker`/`matchGym` (haversine fence + accuracy slack, reject-only
  on `validationStatus='rejected'`). The generic checker stays registered at weight 0, always passing
  (`score:1, reject:false`) — a harmless permanent no-op, not a gap to close.

### Consequence tick (daily sweep, `DailyTick.run`)

> **PENALTY APPLICATION IS FROZEN (product decision, `godot-pivot` source of truth, resolved 2026-07-21).**
> The penalty/consequence mechanic is **extracted and dormant** — kept in the codebase as a
> self-contained unit, **not wired into live progression**, and surfaced in **no client screen** (no
> penalty language anywhere in the UI). The mechanics documented below (`applyPenalty`, gold loss,
> grace-window reversal) are the **preserved-for-future-use** shape, not a live-credited loop. What
> stays live: the **tick still runs and still detects shortfalls** — detection acts on absence, and
> only a scheduled job can see a miss, so it must keep running. *Design consequence:* with penalties
> dormant the loop has no downside, so the client's **Home surfaces the weekly-target visibly
> resetting** (progress → zero, days-remaining) as the one surviving tension signal — see the Client
> section. `weekly_target = 0` remains the intentional rest-week opt-out.

- Detects **absence**, not presence — a missed weekly target is a non-event only a scheduled job can catch, which is why this is the component most likely to break silently.
- For each profile, evaluates the **most recently CLOSED local week** in that user's own timezone (`profiles.timezone`) relative to `weekStartsOn` (gameconfig `consequence.params.weekStartsOn`, default **1 = Monday**). Cadence-robust: however late/infrequently the tick runs, it always evaluates the last full week — a multi-week outage self-heals by evaluating only the latest closed week on recovery, deliberately not stacking stale penalties.
- **Idempotency**: stamped per `(user_id, local_date = week.endDate)` in `tick_stamps`, written *after* evaluation (`upsert ... ignoreDuplicates: true`) — a crash before the stamp write is safe, since the penalty RPC's own unique ledger key is the second, authoritative lock.
- **Shortfall rule**: `countAcceptedActivities(week, timezone) < profile.weekly_target` → `applyPenalty`. `accepted=true` is the bar — a verified-but-still-pending (unregistered) sync counts; the user showed up, that's what matters here. **The count is distinct `(activity_type, local day of started_at)`, not raw rows** (Mechanism 3 — see *Per-type reward collapse*): a same-type second workout on the same day collapsed to 0 XP and does not buy a second weekly credit either.
- `weekly_target` default **3**. `weekly_target = 0` → the shortfall comparison (`count < 0`) is never true → **no penalty is ever applied for that user**. **CONFIRMED-INTENTIONAL**: this is the mechanism for an opt-in rest week, not an incidental side effect.
- `apply_weekly_penalty(user_id, local_date, gold_loss)`:
  - Locks the character row `FOR UPDATE`; no character row → `{applied:false, gold_delta:0}`.
  - `delta = -min(current_gold, max(gold_loss, 0))` — gold is clamped so it never goes negative, and the *actual* clamped delta (not the nominal `gold_loss`) is what gets stored, so a reversal later refunds exactly what was taken.
  - Inserts one `consequence_events(user_id, local_date, type='gold_loss', gold_delta, status='applied')` row with `ON CONFLICT (user_id, local_date, type) DO NOTHING` — this unique key is the exactly-once lock; a second call for the same (user, week-close date) inserts nothing and returns `{applied:false, gold_delta:0}` without touching gold again.
  - Default `goldLoss` = **25** (gameconfig `consequence.params`, tunable).
- **Grace window**: every sweep run (independent of the stamp — the stamp blocks re-*penalizing*, never a reversal), re-checks every `gold_loss` event `status='applied'` with `local_date >= today − graceDays` (default `graceDays` = **2**). Re-counts accepted activities for that penalized week using the same server-verified rows ingest itself reads (so a backfilled fake claim still has to clear full verification to buy a refund); if the recount now meets `weekly_target`, calls `reverse_weekly_penalty`.
- `reverse_weekly_penalty(user_id, local_date)`: flips the matching `status='applied'` ledger row to `'reversed'` (append-only spirit — status change, never delete) and refunds the exact stored `gold_delta` back onto `characters.gold`. The `status='applied'` predicate is the exactly-once guard — reversing an already-reversed or never-applied week is a no-op (`{reversed:false, gold_refunded:0}`).
- **Timezone change mid-week (scenario review 2026-07-23):** the week boundary is recomputed each run from the user's *current* `profiles.timezone`; a mid-week change simply shifts where the boundary falls. The "evaluate only the most-recently-closed week" + grace-window design already absorbs the shift (no double-evaluation, no skipped week) — **no special-casing.** A traveller just gets a slightly shorter/longer transition week; acceptable, and it self-heals the following week.
  - **By what event does `profiles.timezone` actually change?** `PATCH /profile` (`routes/profile.ts`), owner-authed, validated to a real IANA zone (`Intl.supportedValuesOf("timeZone")`) — the only other write path is RLS letting the client PATCH the same column directly via PostgREST (0018), same value, no extra gating there either. **There must be no device-locale auto-detection in the client** — this was verified true of the previous in-repo client (a plain manual text field in Settings), but that client was deleted 2026-07-28 and the Unity client is now the partner studio's. It has therefore changed status from an *observed* property to a **required** one: a client that silently PATCHes `profiles.timezone` from the device locale would reintroduce exactly the mid-week shift this section reasons about. So a traveller only gets a mid-week shift if they manually go into Settings and retype their zone — nothing changes it silently on flight landing. This doesn't change the "acceptable, self-heals" verdict above; it just confirms the trigger is a deliberate user action, not background drift, which makes the transition-week quirk rarer in practice than "every flight" would suggest.
- **Scenario reconciliation — the gold-loss penalty is NOT applied or reversed today, and that stays the target (penalty freeze upheld, 2026-07-23).** The "miss target → lose gold" / "late data → gold refunded" scenarios describe the *preserved-for-future* mechanic, **not** current or intended MVP behavior. The tick keeps *detecting* shortfalls (above); the anti-stacking and grace-reversal logic stay in code but dormant; Home's visibly-resetting weekly target remains the sole surviving tension signal.
- One bad user (corrupt timezone, transient error) is caught, logged, and skipped per-profile — never aborts the whole sweep.
- Gym-session finalization (`gymSessionFinalizer.finalizeStaleSessions`) runs as an **independent sweep with an independent failure mode** at the end of `run()` — a bug there must never abort the consequence sweep, which is the more important half of this job.

### Weekly-target reward (migration `0047` + `WeeklyRewardService` — **SHIPPED**, task 1e)

The positive-side mirror of the (frozen) penalty: clearing the weekly workout bar grants a bonus,
rather than merely avoiding a loss. Decision #5 resolved 2026-07-27; built the same day. With
penalties dormant this is the **only** live consequence of the weekly loop.

- **The bar is server-owned, and is NOT `profiles.weekly_target`.** Eligibility is
  `distinctWeeklyCredits >= gameconfig progression.weeklyReward.workoutsRequired` (default **5**).
  `profiles.weekly_target` (default 3) is **client-writable** (0018 grants `update` on it to
  `authenticated`), so paying out against it would turn a preferences field into a reward dial the
  player sets themselves. `weekly_target` continues to drive only Home's display and the dormant
  penalty comparison. **These are deliberately two different numbers.**
- **The count is the Mechanism 3 distinct-day count**, so same-type same-day duplicates that
  collapsed to 0 XP cannot also inflate reward eligibility.
- **Recorded by `reconcileWeeklyReward` (`dailyTick.ts`), which runs every sweep and is
  deliberately NOT gated by the tick stamp.** `evaluateClosedWeek` early-returns on `hasStamp`;
  gating the reward the same way would permanently forfeit any week whose qualifying workout synced
  *after* the sweep that stamped it — the common case, not an edge one, since Health Connect only
  syncs on app open and a Sunday-evening workout routinely lands after Monday's tick has already
  counted the week. This is the reward-side twin of `reconcileGraceWindow`.
- **Bounded by the claim window, not an unlimited lookback.** `reconcileWeeklyReward` returns early
  (before querying) when `week.endDate < localDate(now) − claimGraceDays`: a week past its claim
  window could only produce a row nobody can redeem. Same bound that stops the penalty side
  re-evaluating arbitrarily old weeks, and it closes the farming vector of an absent user
  backfilling old-but-verified activities to harvest many past weeks' rewards on demand.
- **Ledger:** one `consequence_events` row, `type='weekly_reward'`, `status='pending'`,
  `local_date` = the closed week's end date. The pre-existing `unique (user_id, local_date, type)`
  key is the exactly-once guard for *recording* — re-running the sweep inside the window is a no-op
  via `ignoreDuplicates`. Payload lands in the new `reward_detail jsonb` column, shaped like
  `activities.awarded` (`{ xp, gold, stats }` plus `level`, `itemTemplateKey`, `skillKey`) so a
  future item grant is another key, not another migration.
- **Level scaling is wired but inert.** `multiplier = level ^ levelGrowth` (`levelGrowth` default
  1). `characters.level` is `default 1` and **no production code increments it**, so every payout is
  1× today. The hook is intentional — it activates when level-up ships, with no formula change.
- **`itemTemplateKey` / `skillKey` are recorded intent only — nothing grants them.** There is no
  `skills` table and nothing writes `inventory_items`. Do not read a non-null key as an item having
  been handed out.
- **Claiming is player-triggered, never automatic.** `POST /rewards/claim` → RPC
  `claim_weekly_reward(p_user_id, p_cutoff_date)`: advisory lock, then one statement flips every
  eligible `pending` row to `'claimed'` and sums its `reward_detail`, then credits `characters`.
  **The `status = 'pending'` predicate is the exactly-once guard** — a second call matches zero rows
  and credits nothing. Returns `{ claimed, applied: { xp, gold, stats }, character }`, mirroring
  `register_pending_activities` so the client reuses one DTO.
- **Why it does NOT ride `activities` / two-phase digestion literally.** The 2026-07-24 advice said
  to land it through `awarded → register`; that turned out to be the wrong mechanism, because
  `register_pending_activities` aggregates only over `activities` rows. Synthesizing a reward row
  there would trip the `activities_no_overlap` exclusion constraint (0026), re-fire the
  `0045`/`0046` award-clamping trigger, and inflate the very weekly counter this feature reads.
  `credit_referral` (0043) is the existing precedent for crediting outside the activity path. The
  *spirit* of two-phase digestion is preserved exactly: record pending → player claims → credited
  exactly once.
- **Grace window:** `claimGraceDays`, default **2**, its own key rather than borrowing
  `consequence.params.graceDays` (that one belongs to the dormant penalty). Recording, `/state`
  surfacing, and claiming all compute the **same** cutoff from one shared helper, so `/state` can
  never offer a button the claim endpoint then refuses. *Tuning note:* 2 days is tight for a
  **claim** window — the penalty's 2 days bounded late *data* arriving, not the player opening the
  app. It is one live-tunable gameconfig field, no redeploy.
- **`/state`** carries `weeklyReward: { localDate, xp, gold, stats } | null` — server-computed, the
  client only renders it.
- **Fails loud on a missing character, deliberately.** `claim_weekly_reward` raises (rolling the
  whole claim back, so the reward survives for a retry) if the user has no `characters` row. Without
  that guard the CTE flipped the ledger to `'claimed'` while the character `UPDATE` matched zero rows
  — plpgsql does not raise on a zero-row UPDATE — **burning an earned reward and still returning
  `claimed: 1`**. It also returned a *truthy* character object with every field null, because
  `UPDATE ... RETURNING * INTO` on zero rows leaves the composite's fields null rather than the
  record null, so `to_jsonb()` renders `{...: null}`; a caller checking `if (result.character)` would
  read that as success. A profile holding a pending reward with no character is a broken invariant
  (signup creates it), not a state worth a graceful path. **`register_pending_activities` (0012) had
  the same zero-row-UPDATE shape — closed in migration `0048`** with the identical guard. There it was
  worse: without it, the pending `activities` rows still flipped to `registered = true` after the
  no-op credit, permanently burning the award rather than just the ledger entry.

### Guild + referral

- **World boss is FROZEN — may not ship (decided 2026-07-23).** The shared weekly guild boss is
  **on hold and may be cut entirely**; do not build against it, and treat the shape below as
  preserved-for-possible-future-use, not a committed feature. It is **not** part of the near-term
  guild scope (create/join/invite/members + referral, which stay live). If the weekly-loop
  decisions land on a different guild payoff (see *Client → weekly loop*), the boss is dropped.
  - *Preserved shape (if it ever ships):* **boss HP is derived, never stored** —
    `guild_boss_contributions` is an append-only ledger; `bossHpRemaining = startHp −
    SUM(contributions)`, no mutable boss row (no write contention, full history for accountability).
    Currently `GuildService.addContribution` / `getBossHpRemaining` are stubs that throw — this is
    an intended shape, not current behavior, and now not a scheduled one either.
- All guild/referral RPCs are plain (not `SECURITY DEFINER`), run as the caller (the backend's service-role key), and are revoked from `public/anon/authenticated` — same trust posture as `register_pending_activities`. The client only ever reads its own rows via RLS.
- **`create_guild(user_id, name)`**: advisory-locks on `guild_membership:<user_id>` (serializes concurrent creates), then checks membership existence — already in a guild → empty result set (service maps this to `null` → HTTP 409); else inserts the guild and seats the creator as `role='owner'`.
- **`ensure_invite_code(user_id, code)`**: advisory-locks per user, sets `profiles.invite_code` only if currently null, else returns the existing code — idempotent, so concurrent first-time calls can't both "win" and diverge. On the (lottery-odds) unique collision with another user's code, the caller (`GuildService.getOrCreateInviteCode`) retries exactly once with a fresh random `randomBytes(8)` base64url candidate.
- **`join_guild_via_invite(user_id, code)`** — discriminated result by `status`:
  - `invalid_invite` — code matches no profile.
  - `self_invite` — code belongs to the caller.
  - `already_in_guild` — caller already has a membership (checked under the same `guild_membership:<user_id>` advisory lock as `create_guild`).
  - `joined` — inserts membership as `role='member'` into the inviter's current guild (`guildId` is `null` if the inviter hasn't founded one yet — the referral still records; no membership row is inserted in that case). `referralRecorded` is `true` only when the invitee is genuinely new (`NOT EXISTS` an `accepted=true` activity for them — an existing user redeeming a code is not a referral conversion) **and** their `referrals` row is the first ever inserted for that `invitee_id` (PK, `ON CONFLICT DO NOTHING`).
- **`credit_referral(invitee_id, inviterXp, inviterGold, inviteeXp, inviteeGold, maxReferrals)`** — called after the invitee's first workout is *registered* (from `POST /activities/register`); safe to call unconditionally on every register since it's fully idempotent:
  - `no_pending` — no uncredited `referrals` row for this invitee.
  - `not_verified_yet` — the gate (migration `0043`): invitee has no `accepted = true AND registered = true` activity yet — a verified workout actually applied to their character, not merely ingested-and-pending. Stops the reward being farmed by fake signups that never pass verification, or paid before the workout is really claimed.
  - `cap_reached` — inviter already has `count(credited=true) >= maxReferrals` (default **50**, gameconfig `guild.referral.maxRewardedReferralsPerInviter`). The referral still flips to `credited=true` (terminal state) so it stops being retried on every subsequent register, but nothing is paid.
  - success — both `characters` rows credited (`inviterXp`/`inviterGold` to inviter, `inviteeXp`/`inviteeGold` to invitee), referral flipped `credited=true`. Advisory-locked on `referral_credit:<invitee_id>` so concurrent calls can't double-pay.
  - Default reward magnitudes (gameconfig `guild.referral`, Zod-validated with in-code `DEFAULT_GUILD_REFERRAL` fallback): `inviterXp=100, inviterGold=50, inviteeXp=50, inviteeGold=25`.
- **CORRECTION (2026-07-23, IMPLEMENTED — migration `0043`): the referral reward is a *game invite* reward, gated on a REGISTERED activity, not merely an accepted one.**
  - **Framing — game invite, not guild invite.** The payout exists to reward bringing a new player into the app, full stop — it is not, and must not become, a guild-recruitment mechanic. `credit_referral` never checks guild membership and stays that way. Redeeming a personal invite code still separately joins the inviter's guild as a side effect when they have one founded (confirmed **kept**, 2026-07-23) — that side effect is orthogonal to the reward, not what earns it.
  - **Gate:** the `not_verified_yet` check is `accepted = true AND registered = true` for the invitee's activity — a character-applied award — not `accepted = true` alone. The old accepted-only bar was satisfied at ingest time, before the two-phase digestion's register step ran, so it could pay out for a workout still pending and never actually applied to the invitee's character.
  - **Call site:** `creditReferralIfPending` is called ONLY from `POST /activities/register` (`activityRoutes`), immediately after `characterService.registerPendingActivities` credits — the moment `registered` flips true. The old ingest-time call in `ingestActivity` was removed (under the `registered=true` gate it could never satisfy the condition, so it was dead). Register is retried idempotently, so it self-backstops a swallowed best-effort failure.
- **Roster read** (`listMembers`, `GET`-only) — solo caller (no guild) returns `[]`, not an error. Deliberately excludes gold from the summary (gold is the miss-penalty target; exposing peers' balances would leak the punishment economy) while surfacing level/XP/stats and `lastActivityAt` (most recent `accepted=true` activity) as the social-proof / accountability signal. A member with no `characters` row (signup-trigger race) is silently skipped rather than rendered as a zeroed ghost.

### Guild leadership (succession, kick, transfer — 2026-07-23)

- **Activity signal = `profiles.last_active_at`** (migration 0042). Bumped server-side (best-effort) on every `GET /state` (app open). This is the deliberate choice OVER `auth.users.last_sign_in_at`: host-owned token refresh (a LOCKED architecture decision) silently refreshes the access token without ever updating `last_sign_in_at`, so it goes stale for the most active users — it would misfire succession on exactly the wrong people. `last_active_at` is a true app-open marker. NOT client-writable: 0018's allowlist re-grant on `profiles` (display_name, timezone, weekly_target) excludes it by default, so a client cannot PATCH fake activity to keep leadership — the server writes it with the service-role key. Same posture as `invite_code`.
- **Hard vacancy → BEFORE DELETE trigger `reassign_guild_owner_on_departure`.** Deliberate pattern break from the advisory-locked RPCs: account deletion cascades through FKs (`auth.users → profiles → guild_members`) and never calls an RPC, so a trigger is the ONLY thing that can react to a cascade. One trigger covers BOTH `leave_guild` and the account-deletion cascade (one reassignment implementation). When the departing row's `role='owner'`, it promotes the highest-level remaining member — tie-break `xp` desc, then earliest `joined_at`. NO activity filter on this path: a hard vacancy must leave SOME owner if any member remains, else the guild is leaderless; the dormancy sweep re-homes leadership later if that successor is itself dormant. A member with no `characters` row (signup race) is dropped by the inner join (same "skip the ghost" posture as `listMembers`); if the only remaining member is such a ghost, the guild is left ownerless (accepted edge).
- **`leave_guild(user_id)`** — advisory-locked on the same `guild_membership:<user_id>` key as create/join; deletes the caller's membership (trigger handles successor if they were owner). Returns `{left:true,guildId}` or `{left:false,reason:"not_in_guild"}`. Route `POST /guilds/leave` → 200 / 409.
- **`kick_guild_member(owner_id, target_id)`** — guild-scoped advisory lock `guild_leadership:<guild_id>` with a post-lock re-verify of ownership (a concurrent transfer could have demoted the caller). Reasons: `cannot_kick_self` (use leave instead), `not_owner`, `target_not_in_guild`; success `{kicked:true,guildId}`. Route `POST /guilds/kick {targetUserId}` → 200/400/403/404.
- **`transfer_guild_leadership(owner_id, target_id)`** — same validation + guild lock; swaps roles (old owner → member, target → owner; target stays in guild). Reasons mirror kick (`cannot_transfer_self`, `not_owner`, `target_not_in_guild`); success `{transferred:true,guildId}`. Route `POST /guilds/transfer {targetUserId}` → 200/400/403/404.
- **`succeed_inactive_guild_owners(inactivity_days, active_window_days)`** — the SCHEDULED dormancy sweep, called as a THIRD independent sweep by the daily tick (own try/catch, like the gym-session finalizer — a failure never aborts the consequence sweep; adds `leadershipSuccessions` to `TickResult`). For each guild whose owner's `last_active_at` is older than `inactivity_days` (or null), promote the highest-level member who is THEMSELVES active (`last_active_at` within `active_window_days`); tie-break `xp` desc then earliest `joined_at`. The dormant owner STAYS in the guild as a plain member. A guild with no currently-active member is SKIPPED — better a dormant owner than an equally dormant successor. Per-guild advisory lock + post-lock re-verify. Returns the count of guilds whose leadership moved.
- **Config** (gameconfig `guild.leadership`, Zod-validated with in-code `DEFAULT_GUILD_LEADERSHIP` fallback, mirroring `guild.referral`): `inactivityDays=30`, `activeWindowDays=7`.
- **Accepted limit:** the BEFORE DELETE trigger does not share the `guild_leadership` advisory lock, so a simultaneous "owner deletes account" + "owner transfers leadership" could race — an accepted low-probability MVP limit, not worth distributed locking (same spirit as the other accepted limits in the Scope-limits section).
- **Trust posture:** all new functions are plain (NOT security definer), run as the caller (service-role), and are revoked from public/anon/authenticated — identical to the 0030 guild/referral RPCs.

---

## Edge / Platform

> _This section was reconstructed by the orchestrator from the edge auditor's
> findings report (its clean-spec pass was cut short when agents were released).
> Slightly less polished than Moat/Game; verify against source during reconciliation._

### Account deletion (`DeletionRequestService`, landing routes)

- **Request** (`POST /account-deletion`) — **always returns HTTP 200 regardless of whether the email matches an account.** This is the entire anti-email-enumeration property: a differential response (200 vs 404) would let an attacker probe which emails are registered. A raw verification token is generated (`randomBytes`), its `sha256` hash stored, the plaintext emailed; send failures are swallowed so the response shape can't leak match/no-match either.
- **Verify + delete** (`verifyAndDelete(rawToken)`, hit from `GET /account-deletion/verify`):
  - Look up the request row by `token_hash = sha256(rawToken)` **scoped to `status = 'pending'`**. Missing / expired / already-consumed / wrong-status → returns `invalid_or_expired`, and neither the lookup RPC nor `deleteUser` is called.
  - On a valid token, the request is marked consumed **before** the irreversible delete: `token_hash → null`, `status → 'verified'` fires ahead of the `find_user_id_by_email` RPC (ordering asserted via execution-order log, not just "both called") — so a replayed link can't re-trigger a delete.
  - Resolve the account: `find_user_id_by_email(email)`. `null` → request ends `status='rejected'` with the "no matching account" note, `auth.admin.deleteUser` is **never** called. A returned id → `auth.admin.deleteUser(id)` with exactly that id, request ends `status='completed'`, notes null.
  - One `deleteUser` call is sufficient to remove all downstream rows because `profiles.id` cascades from `auth.users.id` and `characters.user_id` cascades from `profiles.id` (see `handle_new_user` cascade below).

### Auth helpers

- **`hashToken` / `generateVerificationToken`** — pure: token is `randomBytes`-derived; `hashToken` is `sha256` → 64-char lowercase hex. Storing only the hash means a DB leak doesn't expose live tokens.
- **`adminKey`** — gate for admin-only endpoints (e.g. venue validation verdicts); constant-time-ish compare against the configured admin key `(inferred: exact compare semantics — verify)`.
- **`rateLimitKey`** — derives the per-caller bucket key for `@fastify/rate-limit`.

### Domain types — the trust boundary (`domain/types.ts`)

- **`EvidenceBundleSchema` (Zod)** validates every client upload at the boundary; fail-fast on malformed input (a rejected bundle is analytics-tracked, not silently dropped).
- **`.max()` anti-abuse ceilings** on client-supplied numeric metrics that feed a reward formula — a named security control (not just `.nonnegative()`), because an unbounded metric is a direct XP/gold-farming vector while checkers are dormant. Confirmed ceilings: `activeEnergyKcal ≤ 20000`, `avgHeartRate ≤ 250`, `durationSeconds ≤ 100000`, `stepCount ≤ 300000`, plus a `distanceMeters` / `runTrack.trackedDistanceMeters` ceiling. These are anti-absurdity caps; the real trust defense is still server-side verification.
- **Source registry (`domain/sources.ts`)** — backend-owned. `SourceMode ∈ {auto, manual, tracked}`: `auto` = provider sync (Health Connect, Strava), `manual` = user-typed entry, `tracked` = live-recorded on-device (`tracked_run`, `tracked_gym_session`). `modeOf(source)` and `SOURCE_CATALOG[source].mode` both carry this mapping (dual source of truth, guarded by a drift test — collapsing them is a deferred refactor).
- **`secondary_source`** — for Health Connect, the originating app package (the app that actually wrote the data into HC); `null` for direct sources. `SUPPORTED_INTEGRATOR_PACKAGES` is the trusted-hardware-integrator allowlist (e.g. Garmin's `com.garmin.android.apps.connectmobile`) that raises the coverage-cap floor 0.7 → 0.85 in verification.

### Analytics (`platform/analytics`)

- **Server-side only** — the client is untrusted, so there is no client telemetry channel. `analytics.track(event, {userId?, payload?})` inserts into `analytics_events` (`userId` omitted → `user_id: null`; `payload` omitted → `{}`).
- Most learning is a **SQL view over an existing append-only domain table** (`activities`, `consequence_events`, `guild_boss_contributions`); `analytics_events` is only for events with no home in a domain table (e.g. a malformed upload, a `run_fragment_merged`). Analysis is Postgres-native — views, no third-party tool.

### Notifications (`NotificationService`) — FCM push, built (commit `e54ee90`, round-trip proven 2026-07-28)

- **Push delivery via FCM (Android); APNs (iOS) deferred with iOS itself.** `backend/src/services/account/notification/NotificationService.ts` fans out server-side; registering/refreshing the platform push token is **host-owned** (§ Client → the host does "push, lifecycle").
- **Token storage**: `device_tokens (user_id, platform ∈ {ios, android}, token unique)` (0001) — an **owner-writable exception** to the read-only RLS posture (client manages its own token directly via `_own` insert/select/delete policies; cascade-deletes with the profile).
- **`sendLossAversionReminder(userId)` and `sendGuildShieldAlert(guildId)` remain stubs that `throw "Not implemented"` — still correctly blocked.** Both presuppose FROZEN mechanics: the loss-aversion penalty loop (§ Consequence tick) and the guild world boss (§ Guild, frozen and may not ship). Left unbuilt on purpose; building either would contradict the freeze.
- **Two non-frozen nudges ship instead, both tied to `weekly_target`** (the one surviving tension signal that isn't behind a freeze) and hooked into the daily sweep (`dailyTick.ts`):
  - **`sendMidweekBehindTargetNudge(userId, completed, target)`** — user is behind their own `weekly_target` with days left in the week to close the gap. Gated on `weekly_target > 0` (the rest-week opt-out is never nudged) and only fires while `completed < target`.
  - **`sendUnclaimedRewardNudge(userId)`** — sent once a closed week's reward has been recorded and is sitting unclaimed.
- **`NotificationService.enqueue(userId, title, body)`** is the shared fan-out: reads all tokens for the user via `deviceTokenService.tokensFor`, calls `send()` (the FCM HTTP v1 call) per token, deletes the token on an `"unregistered"` outcome (FCM's signal that the token is dead — handles rotation/invalidation), and tracks `push_send_failed` on `"error"`. **Never throws** — called from the daily tick's per-user sweep, so a producer that aborts on failure would inherit the tick's silent-failure problem by *also* stopping the sweep partway through; all outcomes are logged and swallowed here instead.
- **Host side is wired**: `AndroidManifest.xml` declares `POST_NOTIFICATIONS`; `AndroidHostBridge.registerPush()` fetches the FCM token and hands it to the listener (never POSTs directly — swallows the "no `google-services.json`" exception so the pipe goes dark instead of crashing); `onPushTokenReceived` (`DevDriverActivity.kt`) POSTs it via `BackendClient.postDeviceToken`, writing the `device_tokens` row.
- **A real end-to-end round-trip was proven 2026-07-28** using `backend/src/services/account/notification/runTestPush.ts` (`npm run test:push:prod -- <userId>`, run from Render Shell for the real `FCM_*` secrets) — `{ outcome: 'sent' }` and the notification arrived on-device.
- **Known gap: `registerPush()` only fires from a manual "Register push" button in `DevDriverActivity` (~line 207), not automatically on sign-in.** `DevDriverActivity` is the current LAUNCHER (Unity dormant), so the button is reachable, but nothing prompts a real user to tap it — almost no real user has a `device_tokens` row today. Auto-firing `registerPush()` after sign-in succeeds is the fix; not yet started (`docs/TODO.md`).

### Local notifications (`PushNotifications.kt`) — LOCAL, not FCM (built 2026-07-27)

Distinct from the `NotificationService` FCM pipe above — no server round-trip, no `device_tokens`, fires
directly off client-local state or a ping response. Two types, one channel each (`run_status` LOW,
`alerts` DEFAULT — importance is fixed at channel-creation time, so they can't share a channel):

- **Run notification** (`showRunOngoing`/`cancelRun`, `RUN_NOTIFICATION_ID`) — ongoing, non-swipe-dismissable
  (`setOngoing(true)`, `setAutoCancel(false)`), tied 1:1 to `RunSessionTracker.start()`/`stop()`.
- **Gym arrival notification** (`showGymArrival`/`cancelGymArrival`, `GYM_ARRIVAL_NOTIFICATION_ID`) —
  one-shot, `setAutoCancel(true)`, fires on the away→at-gym ping transition
  (`GymArrivalEffect.NotifyArrival`, `DevDriverActivity.kt:922-932`).

**Fixed and device-verified 2026-07-28.** Root cause confirmed: `onStopGymTrackingClicked()`
(`DevDriverActivity.kt:864-872`) only cancelled the polling coroutine (`gymTrackingJob?.cancel()`) — it
never touched `PushNotifications`, and `showGymArrival` had no matching cancel call anywhere, unlike the
run notification's `cancelRun`. Added `PushNotifications.cancelGymArrival(context)` and call it from
`onStopGymTrackingClicked()`, alongside resetting `wasAtGym = false` so a fresh tracking session correctly
re-detects the away→at-gym edge instead of treating the gym as already-arrived from the prior session.
Confirmed on-device: notification now clears on "Stop gym tracking."

### Routes + RLS posture

- **`GET /state`** — the single snapshot that feeds the render layer (character, inventory, guild, etc.). The client displays server-returned values only; no progression formula is echoed client-side (DRY + trust).
- **`PATCH /profile`** — the owner-writable exception. Only genuine prefs are writable: `display_name`, `timezone`, `weekly_target`. Any flag the server reads for a trust/eligibility decision (`manual_logging_enabled`, `onboarding_complete`) is **column-revoked** from `authenticated` (migration 0018) so a client can't PATCH it via PostgREST — RLS restricts *which row*, not *which column*, so the column grant is the real guard.
- **`GET /guilds/members`** — roster read; solo caller returns `[]` (200, not 404). Excludes gold (see Guild section).
- **RLS invariant:** clients are **read-only** on all server-authoritative state (characters, activities, inventory, ledgers) — only the backend's service-role key writes them. `profiles` (prefs) and `device_tokens` are the owner-writable exceptions. Guild-scoped reads route through the `SECURITY DEFINER` helper `current_user_guild_id()` to avoid RLS policy self-recursion.

### Backing SQL functions (untested-before-this-session → now covered)

- **`find_user_id_by_email(p_email text) → uuid`** (0037, `language sql`, `security definer`) — bridges the PostgREST-inaccessible `auth` schema. Case-insensitive on both sides (`lower(email) = lower(p_email)`). No match → `null` (not an error). `limit 1` with no `order by`, so two lowercase-colliding emails would return a **non-deterministic** row — relies on Supabase enforcing email uniqueness upstream. Feeds the irreversible account-delete path: a wrong return here deletes the wrong account.
- **`current_user_guild_id() → uuid`** (0001, `stable`, `security definer`, no args) — `select guild_id from guild_members where user_id = auth.uid()`. Keys solely off `auth.uid()` (can't be manipulated by passed input); `guild_members.unique(user_id)` guarantees at most one row → well-defined scalar. Feeds three RLS policies. Too-permissive = cross-guild **data leak**; too-restrictive = member locked out of their own guild (functional bug, not a leak). Unauthenticated (`auth.uid()` null) → `null` → guild-scoped selects return empty, not an error.
- **`handle_new_user()` + trigger `on_auth_user_created`** (0001, `after insert on auth.users`, `security definer`) — bootstraps a new user in the signup transaction (all-or-nothing with the `auth.users` insert): one `profiles` row (`display_name` null, `timezone='UTC'`, `weekly_target=3`) and one `characters` row (`level=1, xp=0, gold=0, str=dex=con=wis=0`). Cascade on delete: `profiles.id → auth.users.id ON DELETE CASCADE`, `characters.user_id → profiles.id ON DELETE CASCADE` — the mechanism that makes one `admin.deleteUser` cleanly remove every downstream row.

---

## Client

> Folded in from the `godot-pivot` branch's client spec (2026-07-23), stated **engine-neutrally**.
> The engine is now decided — **Unity, 2026-07-28**, built by a partner studio under the
> Unity-as-base topology — but this section stays engine-neutral on purpose: it describes behavior,
> and behavior changed neither when the engine got picked nor when the topology inverted. Keeping it
> that way is also what keeps it honest about the trust boundary, which is a property of the
> architecture rather than of the engine — a Kotlin plugin `.aar` is still native code outside the
> IL2CPP runtime, so the host-owned refresh-token guarantee is untouched by the flip. Engine-specific plumbing (autoloads, nav/back-stack, the
> per-request HTTP layer, the GDScript/C# port gotchas) is implementation and deliberately **not**
> specified here.

### Trust posture (client is untrusted — applies to every screen)

- The client **assembles an evidence bundle of raw signals and displays server-returned numbers**. It never computes an award, a trust score, or a reward. This holds identically for the meta systems (Roster/skill-tree, Game campaign) as for the Home workout loop — a mock returning a number is a **placeholder for a server response, never a client-side formula that later gets "promoted."**
- The one client-side numeric operation that touches rewards is presentational: the **claim count-up** animates from the pre-claim snapshot to the post-claim snapshot. Both numbers come from the server (`GET /state`), so it is a diff of **two server snapshots** — the trust boundary is untouched.
- `GET /state` is the single snapshot feeding the render layer; the client re-derives no progression formula.

### Auth / token split (host-owned refresh)

- The native **host** (Kotlin) does platform-native work only: health read → raw signals, OS permissions, hardware-backed token storage, **token refresh**, push, lifecycle. **App networking (`GET /state`, evidence upload) and the interactive OAuth login live in the client layer**, not the host.
- The one auth exception: **the host owns the token-refresh call.** The client performs interactive login and hands the initial access+refresh pair to the host in a single expression; the host stores both hardware-backed, silently refreshes against Supabase's token endpoint on access-token expiry, and exposes only the *current access token* to the client. **The refresh token is never retained, persisted, logged, or read back in client managed memory** — it transits the login response once and goes straight into host storage. This is the whole point of host-owned refresh: the long-lived crown jewel stays out of the most decompilable layer.
- **401 policy: never auto-relogin.** A 401 latches once (concurrent 401s from a fan-out act once), drops the user to the login/splash gate, and waits for a user gesture — an app-focus re-sync fires with no gesture, and the OS credential sheet throttles after repeated dismissals.

### Screen surface + scope

- **18 screens, tab bar Home / Roster / Game / Guild.** **Portrait-locked, phone-only** — a stated constraint (no tablet, no landscape, no light/dark theming, English strings inline).
- **Backend-backed today:** Home (weekly-target hub + claim), sources, profile/name edit, settings, gyms, and the real half of Guild (create/join/invite/members).
- **Zero backend — mocked, quarantined:** **Roster/skill-tree**, **Guild weekly-credits**, and the **Game chapter campaign**. These have no tables, no routes, no `GameConfig` entries. They render against quarantined mock data so UI shape can be validated first; going real is a per-function swap to an API call with the screens unchanged. **The mock shape is not the API contract** — a separate backend design pass derives the schema and the mock adapts to it, never the reverse.
- **No offline queue / cache and no generic HTTP retry.** Every screen is server-backed (no signal ⇒ dead screen, accepted for MVP). Retry is user-visible only, because `POST /activities` is idempotent and register is exactly-once but `POST /guilds` is neither — a blanket retry would create duplicate guilds.

### The weekly loop — four product decisions (OPEN — **NEEDS USER DECISION**)

The three mocked meta systems all hang off one question — *"what is the weekly loop?"* — which `profiles.weekly_target` already anchors server-side. **These must be answered before the mocks are written**, since they determine the eventual schema, and each must be answered as *one* coherent loop (everything meta resolves at the weekly boundary), not four unrelated fictions. Recommended shapes below; the **numbers/cadence are the founders' call**. Whichever way they land, the trust boundary is unchanged — the server computes the value, the client displays it.

- **Guild bonus → flat, per-member-who-hit-target, capped** (recommended). Scaled/contribution-weighted bonuses punish being in a guild with a slow member and create a "carry" dynamic — backwards for a product selling accountability. *UI:* Guild is N lit/dim slots + one number. **The number is open.**
- **Builds → earned by completing a week, not bought with gold** (recommended). Routing acquisition through the verified workout (the moat), not a second currency, avoids a farmable economy loop while checkers are dormant. *UI:* a pick-one-of-three choice on week completion. **Cadence (every week vs milestone weeks) is open.**
- **Respec → free, at week boundaries only** (recommended). Irreversible trees make players avoid the tree; a paid respec is monetization this MVP lacks. *Schema note:* respec-able builds need an **owned-instance table, not a template column** — the template-vs-instance split (see CLAUDE.md / Moat) governs.
- **Battle → deterministic stat check, server-resolved, no combat resolver** (recommended). Answers "did my workouts make me stronger?" with a stat comparison in one screen; collapses the preview→fight split into a two-column comparison (effective stats vs the level's requirement) + one outcome.

### Backend design pass (not yet scoped)

Roster/skill-tree, guild weekly credits, and the chapter campaign require a **data-model + API design pass** (tables, routes, `GameConfig` entries) that does not exist yet. It is backend work sized separately and **blocked on the four decisions above**, since they determine the schema.

---
