# PLAN — Scope Closeout

Phased plan to close every open item in `docs/TODO.md` + every "NOT YET IMPLEMENTED" in
`Interpretable-Context-Methodology/departments/engineering/docs/backend-behavior-spec.md`
(sibling repo), ordered so unblocked, high-leverage work ships first and blocked/release-gated work
is quarantined at the end.

**Verified against code 2026-07-23** (three read-only audit passes: backend/SQL, Android host, Unity
client). Every "today" statement below is confirmed at a real `file:line`; the docs did **not** drift
from the code. Highest migration = `0043`, so new migrations start at `0044`.

---

## Decisions needed before / during (these gate specific phases)

1. ~~Notification trigger~~ **RESOLVED (2026-07-24): weekly-target nudge, no mechanic unfrozen.** The
   two existing stubs (`sendLossAversionReminder`, `sendGuildShieldAlert`) both presuppose **FROZEN**
   mechanics (dormant penalty loop; frozen guild boss) — neither can ship as-is, and unfreezing either
   would contradict a locked CLAUDE.md decision (2026-07-21 penalty freeze; 2026-07-23 guild-boss
   freeze), reopening a product call that was explicitly closed. Build the delivery pipe now and wire
   ONE new non-frozen trigger — a nudge tied to Home's visibly-resetting weekly target (the surviving
   tension signal). Both frozen stubs stay dormant/undeleted.
2. ~~Periodic gym-ping home~~ **RESOLVED (2026-07-24): host-side `WorkManager`.** Kotlin,
   **engine-independent**, survives backgrounding, buildable now — decouples Phase 4 from the
   engine pivot entirely (which is also why this survived the pivot closing on Unity, 2026-07-28).
   Rejected: an in-app timer in the engine — foreground-only, which defeats the purpose of passive
   gym sessions.
3. ~~`track-smoothness` vs existing `TrackConsistency`~~ **RESOLVED (2026-07-24): not redundant, keep
   both — but `track-smoothness` is NOT Phase-1-shaped.** `TrackConsistencyChecker` only corroborates
   aggregate track-vs-claim numbers (both client-attested); `track-smoothness` checks point-to-point
   physical plausibility within the track itself — an orthogonal, harder-to-fake signal. However the raw
   per-sample points it needs don't exist in `EvidenceBundle` today (only aggregates) — it needs host-side
   capture work first, so it's moved out of Phase 1. See 1d.
4. ~~Does sleep feed rewards?~~ **RESOLVED (2026-07-24): yes, eventually — but not in Phase 3.**
   Reverses the 2026-07-19 "display only, no analysis" decision in `docs/TODO.md`. Phase 3 ships
   read + display only, as already scoped (no `SignalChecker`, no reward path) — a sleep-reward path
   is real future work but is its own follow-up task, sequenced after Phase 3, not a Phase 3 expansion.
   Written up as **Phase 3.5** (below), gated on Phase 3's 3a/3b landing first.
   When built: needs its own `SignalChecker` registered in `VerificationEngine`'s constructor list
   (Open/Closed — never edit the scoring loop), and per the weighted-mean dilution rule must land as a
   **weight-0 gate or multiplier, not a weighted peer**, on whichever profile it joins. **Confirmed scope
   (2026-07-24): sleep rewards but does not count toward `weekly_target`** — it grants its own XP/gold/
   stats (or item/skill, per decision #5's note) independent of the weekly-activity mechanic; whatever
   counts activities toward `weekly_target` (`dailyTick.ts` / `evaluateClosedWeek`) must not be touched to
   include sleep.
5. ~~Weekly-target reward magnitude~~ **RESOLVED (2026-07-27).** Players hitting `weekly_target` (N
   workouts, N = configurable; default 5) during their local-week window can claim a global reward (same for
   all players): **level-scaled XP/gold/stats + level-scaled global item/skill**. Player must actively claim
   it; grace period applies (reuses `graceDays`, same as penalty reversal); rewards persist unclaimed through
   grace window. Credits via two-phase digestion (`awarded` → register) for exactly-once semantics. See task
   1e for full implementation scope.

6. **Day-window keying — RESOLVED (2026-07-27), shipped as `0046`.** The `clamp_daily_awarded_xp`
   trigger now uses **two** day windows on **two different columns, deliberately**: per-type collapse
   and per-axis stat cap key on the **performed** day (`started_at`); the daily XP cap keys on the
   **receipt** day (`created_at`) and became a local calendar day instead of a rolling 24h window.
   Fairness vs. forgery-resistance — the asymmetry is intentional and must not be "fixed" into
   consistency. Rejected: keying everything on `started_at` (removes the only mint bound a forged
   bundle cannot move) and keying everything on `created_at` (punishes batch-syncing users, the
   original bug).

   **AMENDED (shipped as `0054`).** The receipt-day cap is no longer a flat `maxXpPerDay`: it is
   `maxXpPerDay × maxCatchUpDays` (default 7, matching `HealthConnectReader.LOOKBACK`), and the
   per-axis stat cap gained the same receipt-day ceiling. A client-supplied value can therefore
   now widen the budget — by a **bounded** factor, which is what makes it acceptable. Two things
   hold the amendment up, and neither is optional:
   - A new **performed-day XP budget** (`maxXpPerDay` per `started_at` day, cumulative across all
     receipt days) pins the *sustained* mint rate at today's level; the multiplier only prices a
     one-time burst. It also closes a hole `0046` shipped with — nothing previously bounded one
     performed day's XP across receipt days at all.
   - A **forward bound on `startedAt`**. Its refine was one-sided, so future days were unbounded;
     with a per-performed-day budget that would have been an unbounded run of fresh budgets.

   Fairness of the receipt-day cap **no longer depends on 4e** — that is the point of the
   amendment. `4e` is now a latency optimisation, not a correctness dependency.
7. **Weekly-reward gate — RESOLVED (2026-07-27).** The reward fires at a **server-owned** gameconfig
   `workoutsRequired` (default 5), **not** `profiles.weekly_target` (default 3, client-writable via
   0018). Decision #5's "N workouts, N = gameconfig var, default 5" and the column's default of 3 were
   two different numbers; this resolves them apart rather than together. `weekly_target` stays a pure
   display/personal-goal preference. Rejected: gating on `weekly_target` (a client could PATCH it to 1,
   claim, and PATCH back — a prefs column becomes a reward dial, violating the trust-boundary rule).

Out of scope for this plan (separate product calls): the 4 OPEN weekly-loop decisions (guild bonus,
build cadence, respec, battle resolution) — those gate Phase-4 *client* mocks, not the backend below.

---

## Phase 1 — Backend anti-cheat hardening (UNBLOCKED · highest moat value)

Pure backend + SQL. No device, no engine decision, no frozen mechanic. This is the moat — server-only,
fully integration-testable. **Do first.**

- [x] **1a. Cross-account native-id dedup** (new migration `0044`). Today `findDuplicateActivity`
  (`ingestActivity.ts:113`) is scoped `.eq("user_id", userId)` — per-user only, so the same health-platform
  workout replayed under two accounts credits twice. Add a cross-account guard on the native/source workout
  id. **Scoped threat model (revisited 2026-07-24):** `sourceWorkoutId` is a real Health Connect record id
  (`session.metadata.id`, `HealthConnectReader.kt:146`) — there is no app UI to type or edit it, so this
  guard stops the *casual* zero-effort replay (log out, log into a second account on the same device,
  resync — Health Connect is a shared per-device datastore, so both uploads carry the identical native id).
  It does **not** stop a determined attacker running a MITM proxy or a hand-rolled API client to fabricate a
  distinct id per account — nothing server-side can, since the whole bundle is client-attested. Ship it as
  "raises the bar against casual multi-account/referral farming," not "closes cross-account replay."
  **Needs its own index** — both existing `source_workout_id` indexes (`0013`, `0014`) are composite
  `(user_id, source_workout_id)`, user-scoped leading column; a cross-account check
  (`WHERE source_workout_id = $1 AND accepted = true`, no `user_id` filter) can't use either efficiently
  and would be a full table scan as the table grows.
  **Bound the query by `started_at`, not just `accepted` (2026-07-24 refinement) — this matches the
  reachable threat, not just a performance trim.** `HealthConnectReader.kt:369` sets
  `LOOKBACK = Duration.ofDays(7)`; every HC read is windowed `now.minus(LOOKBACK)..now`
  (`HealthConnectReader.kt:91,125`). No client ever re-uploads a workout more than ~7 days after it
  happened — HC simply stops returning it. So any existing accepted row that could ever collide with a
  *new* incoming bundle must itself have `started_at` within that same recent window; a row older than
  that can never again be the target of a cross-account replay, because no client can still fetch and
  resubmit it. Add `create index activities_source_workout_id_recent_idx on activities
  (source_workout_id, started_at) where source_workout_id is not null and accepted = true` (composite,
  not a `now()`-bound partial predicate — partial index predicates must be static) and filter the query
  with `AND started_at > now() - interval '14 days'` (double the lookback for clock-skew/DST slack, not
  a security-relevant number — just a safety margin). This date-bounded SELECT stays as the early,
  graceful rejection path (a clean `duplicate_workout` response without hitting a constraint violation).
  **A pre-check SELECT alone is not race-safe — needs a real unique constraint too.** Two concurrent
  uploads of the same native id from two different accounts can both pass the SELECT before either
  commits, then both INSERT — the exact race `0014_source_workout_id_unique.sql` already solved for the
  *per-user* case via a unique index (its own comment: "the unique index makes the second INSERT fail
  with 23505 ... the route catches and returns as a graceful duplicate response"). Mirror that here:
  `create unique index activities_source_workout_id_accepted_unique on activities (source_workout_id)
  where source_workout_id is not null and accepted = true` (no `user_id` in the key — that's the whole
  point of "cross-account"). First accepted insert for a given native id wins; the second's insert throws
  23505, caught the same way `isUniqueViolation` already catches it, mapped to `duplicate_workout`. Note
  the unique constraint's predicate can't carry the same date bound (partial-index predicates must be
  static, and it must hold for all time to guarantee correctness) — the two mechanisms solve different
  problems: the constraint is what actually prevents the double-credit under concurrency; the date-bounded
  SELECT is what keeps the common-case, no-conflict path a cheap, targeted lookup and gives a graceful
  error before ever reaching the constraint.
  **Do NOT filter the dedup check by `registered`** — a verified-but-still-pending
  (unregistered) row is still a real duplicate per the existing per-user semantics
  (`findDuplicateActivity`'s own doc comment); filtering `registered = true` would reopen exactly this
  ingest→register window for a second account to double-credit through, and buys no query-speed benefit
  either (`source_workout_id` equality is already maximally selective). Ships with a
  `*.integration.test.ts` (real Postgres) proving a second account can't re-credit the same native id,
  that a stale (>14-day) matching row is correctly ignored, and that two concurrent same-native-id
  inserts under the constraint resolve to exactly one accepted row.
- [x] **1b/1c. Per-axis stat cap + per-type reward collapse — EXTEND `0028`, do not add a parallel trigger**
  (revised 2026-07-24). Real farming hole confirmed: `statPointsPerActivity: 1` is flat, so many tiny
  sub-cap activities each mint a full stat point on one axis while staying under the *total* 2000 XP/day
  cap — stats only zero out when the cap is fully exhausted (`0028_daily_xp_cap.sql:36-104`). But a
  separate `0045` trigger is the wrong shape: two `BEFORE INSERT/UPDATE OF awarded` triggers on the same
  column makes Postgres firing order matter (undocumented, alphabetical-by-name), takes a second redundant
  `pg_advisory_xact_lock` on the same user inside the same transaction, and re-runs the exact same rolling-
  window scan a second time. **Fold 1b/1c directly into `clamp_daily_awarded_xp()`** — one lock, one window
  scan extended to sum per-axis stats and per-`activityType` totals alongside the existing XP sum, one
  function to reason about. **Also add the index this scan has always been missing:**
  `create index ... on activities (user_id, created_at) where accepted = true and awarded is not null` —
  today's only composite index is `activities_user_started_idx (user_id, started_at)`
  (`0001_init_schema.sql:85`), which does not cover the trigger's `created_at` filter; a partial index
  scoped to exactly the scanned rows keeps it small. Cap/curve values live in `gameconfig`, read via
  `GameConfigService.getValidated`. Ships with an integration test proving the extended trigger composes
  correctly (per-axis cap, reward-collapse curve, and the original XP cap all correct in one insert).
  **Migration number: `0045`** (a `create or replace` of `clamp_daily_awarded_xp` + the new index, one
  file — not two).
- [x] **1d. Anti-spoof signals — split into two, only one is Phase-1-shaped** (revised 2026-07-24, checked
  against the actual `EvidenceBundle` shape, not just the SPEC prose).
  - **`motion-corroboration` is an EDIT to the existing `AccelPresenceChecker.ts`, not a new file.** The
    SPEC says it plainly: "Promotes the existing `accel-presence` signal from mere presence to GPS↔accel
    corroboration." Today's checker only reads `bundle.accelPresence` in isolation
    (`AccelPresenceChecker.ts:14`); the upgrade adds the joint condition — GPS shows movement (`runTrack`)
    while the accelerometer reads stationary. Same signal, smarter logic, same `name`
    (`"accel-presence"`), same weight — editing a checker's own `evaluate()` body is normal maintenance,
    not an Open/Closed violation (that principle guards `VerificationEngine`'s scoring loop, not a
    checker's internals). **No new gameconfig weight-seed needed** — the name is already seeded. Unit-test
    the new joint-condition branch in the existing `AccelPresenceChecker` test file.
  - **`track-smoothness` is genuinely new logic, but is NOT buildable as scoped — it needs host-side work
    first, so it does NOT belong in a "no device" Phase 1.** Checked `EvidenceBundleSchema.runTrack`
    (`domain/types.ts:78-84`): it only carries **aggregates** — `sampleCount`, `trackedDistanceMeters`,
    `movingSeconds`. There is no raw per-sample point array (lat/lng/timestamp per GPS fix) anywhere in the
    bundle. A "route wobbles, a too-smooth line is a tell" heuristic needs the actual point sequence to
    measure inter-sample jitter/speed jumps — that data does not exist server-side today. Building this
    requires: (a) a host-side (Kotlin) change to capture and transmit raw track points, (b) an
    `EvidenceBundleSchema` extension carrying them (with the usual `.max()` ceiling — a point array is
    exactly the kind of client-supplied numeric-adjacent input that convention exists for), and only then
    (c) the checker itself. **Move `track-smoothness` out of Phase 1 into a new host+backend task** (natural
    fit alongside Phase 2's host work, or its own small phase) — it is not a same-shape sibling to
    `motion-corroboration` the way the original plan assumed. Register it as a new `SignalChecker` impl in
    `VerificationEngine`'s constructor list when it's built — never edit the scoring loop (Open/Closed) —
    and per the weighted-mean dilution rule, add it as a **weight-0 gate or multiplier, NOT a weighted
    peer** to the already-anchor-gated `tracked_run` profile (`track-consistency:1, accel-presence:1,
    metric-rate:0`, threshold 0.6). Unit-test the checker; seed its weight via a gameconfig migration once
    it has a home.

  Exit for 1d as it now stands in Phase 1: just the `AccelPresenceChecker` edit — no migration, no schema
  change, unit-test only.

- [x] **1e. Weekly-target reward** — **SHIPPED 2026-07-27** (migration `0047`, `WeeklyRewardService`,
  `POST /rewards/claim`, `reconcileWeeklyReward` in the tick, `weeklyReward` on `/state`). Full shipped
  behavior: spec § *Weekly-target reward*. Four things landed differently from the scoping text below,
  each for a reason worth carrying forward:
  1. **The reward bar is a server-owned gameconfig `workoutsRequired` (default 5), NOT
     `profiles.weekly_target`.** That column is client-writable (0018), so gating a payout on it would
     turn a preferences field into a reward dial. The two numbers are deliberately different.
  2. **It does NOT ride two-phase digestion literally.** `register_pending_activities` aggregates only
     `activities` rows; a synthetic reward row there would trip `activities_no_overlap` (0026), re-fire
     the award-clamping trigger, and inflate the very weekly counter the feature reads. Modeled on
     `credit_referral` (0043) instead. Record-pending → player-claims → credited-exactly-once preserves
     the *spirit* intact.
  3. **Recording lives in a new `reconcileWeeklyReward`, not in `evaluateClosedWeek`.** The first
     implementation put it there and was wrong: `evaluateClosedWeek` early-returns on the tick stamp, so
     any week whose qualifying workout synced after that week's sweep would have been forfeited forever
     — the common case, since Health Connect syncs only on app open. The new method runs every sweep,
     ungated by the stamp, bounded by the claim window.
  4. **Level scaling is wired but inert** (`level ^ levelGrowth`, 1× today — `characters.level` is never
     incremented by any code); **item/skill are recorded intent only** — no `skills` table exists and
     nothing writes `inventory_items`.
- [ ] ~~1e original scoping~~ (superseded by the above; kept for the reasoning trail)
  (SPEC RESOLVED — sequence AFTER 1b/1c's trigger design is settled;
  shares the same reward-crediting path). Positive mirror of the (frozen) penalty: players who hit
  `weekly_target` (N workouts, N = gameconfig var, default 5) within their local-week window can claim a
  reward. **Attribution already correct for free** — `evaluateClosedWeek` counts by `started_at`
  (`dailyTick.ts:216`), so a late-synced backfilled workout lands in the week it was *performed*, never
  double-counting into the current week. **Reward spec (resolved, not pending):** all players get the same
  **global** reward — XP/gold/stats + one global item/skill, **all scaled by the player's current level**
  (not flat). Player must **actively claim** the reward (not auto-credited); a **grace period** applies
  (reuse existing `graceDays`, same pattern as penalty reversal) and **rewards persist unclaimed through
  the grace window** — they don't auto-delete. The claiming path must reuse two-phase digestion (`awarded` →
  register) so the reward credit is exactly-once; the ledger side needs a `weekly_reward` sibling to
  `reconcileGraceWindow`, with its own exactly-once event type on `consequence_events` (same
  `(user_id, local_date, type)` unique-key pattern as `gold_loss`). **Schema work:** `consequence_events.type`
  has a hard `check (type in ('gold_loss', 'gear_degrade'))` (`0001_init_schema.sql:221`) — `weekly_reward`
  isn't in the allowed set today, and the table only has a `gold_delta integer` column with no shape for
  XP/stats/item. This migration must widen the check constraint and add a `reward_detail jsonb` column
  (matching `activities.awarded`'s shape so it absorbs future item/skill grants without a second migration).
  Integration-tested: reward is claimable once per week, never for the in-progress week, never past the grace
  window, and persists unclaimed through grace period.

Exit: all five land behind integration tests; `npm test` + `npm run test:integration` green.

---

## Phase 1.5 — Weekly-credit distinct-day counting (Mechanism 3) (follow-up to Phase 1, gated on 1b/1c shipping first)

Closes the interim gap 1b/1c's migration (`0045`) intentionally left open, documented in its own
comment and in the spec (§ *Per-type reward collapse + per-axis stat cap*): rewards now collapse per
type/axis per local day, but the weekly-target counter (`countAcceptedActivities`,
`dailyTick.ts:216`) still counts every raw `accepted=true` row in the week window — so a same-type
second workout that just collapsed to 0 XP/gold still counts as a fresh credit toward
`weekly_target`. This was deliberately NOT folded into `0045` — it changes the miss-rule /
consequence-tick path, a different and more sensitive blast radius than the reward-crediting
trigger (`CLAUDE.md` flags the daily tick as the most likely first failure point, since it acts on
absence and bugs there are silent) — but it was never actually written down as a task until now; it
existed only as a migration comment with no owner.

- [x] **1.5-pre. Two day windows in one trigger** (migration `0046`) — **SHIPPED 2026-07-27, added
  mid-flight.** Not in the original plan; surfaced while scoping 1.5a and fixed two bugs in the shipped
  `clamp_daily_awarded_xp()`:
  - **Mechanisms 1 & 2 now key on `started_at` (performed day), not `created_at`.** Health Connect
    batch-syncs up to 7 days on app open, so a user who trained Monday and Tuesday but opened the app
    Wednesday had both land in one receipt-day bucket and the second collapse to 0 XP. Two workouts on
    two calendar days are two workouts.
  - **The `0028` daily XP cap became a local calendar day, was a rolling 24h window.** Mon 19:00 and
    Tue 08:00 are 13h apart, so the Tuesday session was clamped against Monday's spend despite being a
    different day. It stays keyed on `created_at` — that is the anti-backdate anchor, and its fairness
    now depends on Phase 4's **4e** (background HC sync) landing.
  - The two scans keying on **different columns is deliberate** — fairness vs. forgery-resistance —
    and is documented in `0046`'s header so nobody "fixes" it into consistency.
  - Residuals accepted and recorded: N backdated fake days can claim N axis stat points (bounded by the
    receipt-day budget zeroing stats once exhausted); 2000 XP at 23:50 + 2000 at 00:10 is now legal.
- [x] **1.5a. `countAcceptedActivities` → distinct `(activity_type, local_day)`** — **SHIPPED
  2026-07-27.** Took route (b) (dedupe in TS via the existing `localDateOf`), not route (a): once
  `0046` re-keyed the collapse to `started_at`, the week window and the day bucket became the same
  column, so a plpgsql RPC bought nothing. Per-user weekly row volume is under ~25. New live drift risk
  recorded in the spec: SQL and TS now each own a local-day definition and must agree.
- [x] **1.5b. `reconcileGraceWindow` recount** — **SHIPPED 2026-07-27**, no separate code change (same
  function), covered by its own test case as required.
- [x] **1.5c. Schema:** none needed, as predicted.
- [ ] ~~1.5a original scoping~~ (superseded; kept for the reasoning trail) Today's query
  (`dailyTick.ts:216-225`) is `count("id") .eq("accepted", true) .gte/.lt("started_at", week
  bounds)` — a raw row count. Must become `COUNT(DISTINCT (activity_type, local_day))` over the
  same window, where `local_day` reuses the exact server-receipt (`created_at`) /
  `profiles.timezone` scoping Mechanisms 1/2 already established in `0045` — not a second,
  independently-drifting definition of "day." **Implementation call for whoever builds this:**
  either (a) a small `plpgsql` RPC (mirrors `register_pending_activities`'s pattern of pushing
  timezone-sensitive counting into Postgres, callable via `supabase.rpc`), or (b) fetch
  `(activity_type, created_at)` rows for the window and dedupe in TS against `profile.timezone` —
  cheaper to ship but duplicates the local-day math `0045` already has in SQL, a drift risk if the
  two definitions ever diverge. Prefer (a) unless per-user row volume makes (b) clearly simpler.
- [ ] ~~1.5b original scoping~~ Same fix applies to `reconcileGraceWindow`'s recount
  (`dailyTick.ts:175`), which calls the same `countAcceptedActivities` — no separate code change once
  1.5a lands, but the test must cover the grace-window recount path too, not just the primary
  `evaluateClosedWeek` path: a stale distinct-count bug there would silently mis-refund a penalty
  reversal.
- [ ] ~~1.5c original scoping~~ Schema: none needed either way — this reads existing columns
  (`activity_type`, `created_at`, `started_at`); route (a) adds one small function, route (b) adds none.

**Exit — MET 2026-07-27.** `evaluateClosedWeek` and `reconcileGraceWindow` both count distinct
`(activity_type, local day of started_at)`, not raw rows. Covered by unit tests on both call sites
(same-type-same-day → 1 credit, different-day → 2, different-type-same-day → 2, plus a Tokyo case
proving local-day rather than UTC-date bucketing, plus a grace-window case proving duplicates cannot
buy a false penalty reversal) and by integration tests on the `0046` trigger itself.

---

## Phase 2 — Notification delivery pipe + non-frozen trigger (UNBLOCKED — decision #1 resolved)

The whole FCM path is scaffolding today; **none of the pipe depends on frozen mechanics** — only the
*trigger* does. Build the pipe, wire one new non-frozen trigger. Unblocks the Data Safety "Device or other
IDs" declaration (which must ship coupled to real collection).

- [x] **2a. Host — permission + token.** Add `POST_NOTIFICATIONS` to `AndroidManifest.xml` (absent today;
  Android 13+ shows nothing without it; runtime-requested → handle denial). Implement FCM token registration
  behind the existing `registerPush()` seam (`AndroidHostBridge.kt:238` is a no-op stub; `HostBridge.kt:131`
  declares it) so the iOS/APNs port reuses one seam.
- [x] **2b. Server — token write + rotation.** Client writes its own `device_tokens` row (0001, owner-writable
  RLS — re-read the policy before relying on it, per the trust rule). Handle FCM token rotation: delete the
  row on the "unregistered" response (a stale row is a permanent silent no-send).
- [x] **2c. `NotificationService.enqueue`.** Implement the actual FCM/APNs fan-out (`NotificationService.ts` is
  two throwing stubs today, no `enqueue`). Log every send — a push producer inherits the tick's silent-failure
  problem.
- [x] **2d. Non-frozen trigger.** Add a new method (e.g. `sendWeeklyTargetNudge(userId)`) wired to the daily
  sweep, tied to the weekly-target reset — **not** to the dormant penalty or the frozen boss. Leave the two
  frozen stubs dormant/undeleted. Built as two nudges (`sendMidweekNudge`, `sendUnclaimedRewardNudge`),
  tested (`dailyTick.test.ts`, `nudges.integration.test.ts`).
- [x] **2e. Policy (rides with the code, same release).** Tick **Device or other IDs** in Data Safety —
  done 2026-07-28. Real device token registered, real FCM send confirmed (`outcome: 'sent'`), notification
  received on-device, declaration ticked (Collected, not ephemeral, users can choose, App functionality +
  Developer communications — see `docs/TODO.md`). App access instructions text was already pasted in an
  earlier session. **CLOSED 2026-08-17.** `registerPush()` now fires automatically from
  `AndroidHostBridge.storeAuthTokens` (fresh login) and from `UnityHostActivity.onCreate` (every start, which
  is what covers an already-signed-in user), and the host POSTs `/devices` itself rather than relying on a
  Unity forward that `HostBridgeReceiver.cs` never implemented. Before this, `device_tokens` was EMPTY in
  production: 22 `nudge_midweek` and 5 `nudge_reward_pending` rows had been written between 2026-07-26 and
  2026-08-16, every one of them fanning out to zero devices. Those 27 slots are burned permanently — the
  ledger claim is exactly-once and is deliberately written BEFORE the send.
- [x] **2f. Active-tracked_run notification (NEW, 2026-07-24) — LOCAL, not FCM, independent of 2a–2c.**
  `RunSessionTracker.start()`/`isActive()` (`RunSessionTracker.kt:41-88`) already backs a UI toggle and
  already knows locally when a run session is active — no new live-session concept needed on the server.
  On a successful `start()` (returns `true`), the host calls a new **dedicated** endpoint (e.g.
  `POST /activities/run-started`, ack-only, per-user rate-limited like its siblings) and, on 200, posts a
  local ongoing Android notification; `stop()` cancels it. Chosen over reusing an existing endpoint so the
  call has a semantic home if it's ever worth persisting (e.g. correlating against the eventual
  `EvidenceBundle.startedAt`) — the endpoint does nothing but ack today. Needs `POST_NOTIFICATIONS` (shared
  with 2a) but no FCM token, no `device_tokens` row, no fan-out — can ship independent of the FCM pipe.
  **Device-tested 2026-07-28: start/finish notifications confirmed working** (`showRunOngoing`/`cancelRun`,
  `PushNotifications.kt`).
  **Call wired 2026-08-06 (issue #13).** Until then nothing in the shipped `.aar` called the route at all —
  only the dev driver did — so it was live, documented and dead. `AndroidHostBridge.startRunSession` now
  fires it **fire-and-forget** off `scope` on a successful `start()`. The "and, on 200, posts a local
  notification" half of the text above stays **withdrawn** per 4d: an FGS must post within 5s of starting,
  which a round trip cannot promise. Sited on the bridge, not on `RunSessionService`, because a run can
  start while the anchor service does not (`RunSessionTracker.start`'s `SecurityException` path) — a call
  living in the service would miss exactly those runs.
- [x] **2g. Gym-presence notification (NEW, 2026-07-24) — LOCAL, reuses an existing response.** `POST
  /gyms/ping` already returns `{ atGym: boolean, gymId? }` synchronously (`GymPingService.recordPing`,
  `GymPingService.ts:31-54`) — the same request that fires the ping already tells the caller whether this
  ping landed at a registered gym. No new endpoint, no push: when the response has `atGym: true`, the host
  posts a local notification directly off that response, guarded to the away→at-gym transition
  (`GymArrivalEffect`) so re-foregrounding while still at the gym doesn't re-fire it. **Fixed and
  device-verified 2026-07-28:** device-tested with an open bug (notification didn't clear after "Stop gym
  tracking" — see `Interpretable-Context-Methodology/departments/engineering/docs/backend-behavior-spec.md`
  § Local notifications, sibling repo), root cause was a missing
  `cancelGymArrival()` call in `onStopGymTrackingClicked()`. Added, compiles clean, retested on-device —
  notification now clears correctly.

Exit: a token round-trips to `device_tokens`; a test send logs; declaration flipped in the same release;
2f/2g fire their local notifications off the existing toggle/ping paths with no FCM dependency.

---

## Phase 3 — Health Connect sleep (read + display) (UNBLOCKED)

Locked decision: sleep is **displayed as-is, no analysis, no recommendations**. So this is read → bundle →
display, with **no reward path** — a reward path is coming later (decision #4, resolved 2026-07-24) but is
its own follow-up task, not part of this phase. Code + coupled policy.

- [x] **3a. Host — SHIPPED 2026-07-28.** `READ_SLEEP` declared in `AndroidManifest.xml`;
  `HealthConnectReader.readLastNightSleep()` reads the most recent `SleepSessionRecord` in the existing
  7-day `LOOKBACK` window. **One thing landed differently and matters:** the sleep permission is NOT in
  `requiredPermissions` — it went into a new `optionalPermissions` set, because `DevDriverActivity` and
  `UnityHostActivity` both gate on an all-or-nothing `granted.containsAll(requiredPermissions)`. Adding
  sleep there would have made a user who grants everything-except-sleep read as fully denied, silently
  breaking the workout and step reads that already work. A declined sleep grant returns null, never an
  error, and never routes to `onHealthReadError`.
- [x] **3b. Server — SHIPPED 2026-07-28.** Optional `sleep` sub-object on `EvidenceBundleSchema`
  (`startedAt`/`endedAt`/`totalMinutes ≤ 1440`), mirroring `runTrack`. **No `SignalChecker`.** Chosen over
  a new `ACTIVITY_TYPES` member deliberately: an activity type creates `accepted` rows that
  `countAcceptedActivities` (`dailyTick.ts:216`) would sweep into `weekly_target` — exactly what decision
  #4 forbids. As a sub-object that is structurally impossible rather than a rule someone must remember.
  No migration needed: `ingestActivity.ts:87` stores the whole bundle in the `evidence_bundle` jsonb column.
- [x] **3c. Display — SHIPPED 2026-07-28.** DevDriver button + `onSleepRead` ("Last night: 7h 20m" /
  "No sleep data recorded."), and the full Unity bridge round trip: `requestSleepRead` on the `HostBridge`
  interface, `AndroidHostBridge` impl, `UnityHostActivity` → `UnitySendMessage("OnSleepRead", …)`,
  `HostBridgeReceiver.OnSleepRead` + `HostBridge.RequestSleepRead` C#-side, documented in
  `unity-bridge-contract.md` (partner's `reign-and-gain-unity` repo, `docs/`). Copy is descriptive only. **The C# half is unverified by construction**
  — no Unity CI, and the Unity path is not live on device (Phase 5 gate 2).
  **Device-verified 2026-07-28 (Kotlin path):** permission grant → read → "Last night: 7h 12m" rendered
  correctly, and the partial-grant regression check passed — workout sync still succeeded while sleep was
  ungranted, confirming the `optionalPermissions` split does what it was built for. One transient gotcha
  worth knowing: an initial read returned "No sleep data recorded." simply because Health Connect did not
  yet hold the session — an empty read and a failed read are currently indistinguishable in the UI, since
  `AndroidHostBridge.requestSleepRead` catches every exception and reports it as absence (the `Log.w` is
  the only signal). Acceptable for a dev harness; worth splitting if sleep ever gets a real user-facing
  screen.
- [x] **3d. Policy — PARTIALLY DONE.** "sleep management" in the Health Apps declaration was already
  ticked earlier. **Privacy policy done:** `PrivacyPage.tsx` now names sleep as its own item under
  "What we collect" — read from Health Connect, displayed in-app, explicitly **not sent to or stored on
  our servers**, per the 2e lesson (declare what is actually exercised, not the transmitted-data language
  used for the workout bullet above it). **Still open:** the Data Safety Health-info entry in Play Console
  needs sleep named — same read/displayed-not-collected framing, until Phase 3.5 wires an upload.

**Exit — PARTIALLY MET.** Sleep reads, displays, and has its ceiling-guarded schema field. It does **not**
"round-trip into the bundle": nothing populates `EvidenceBundle.sleep`, deferred deliberately because there
is no sane producer yet — the bundle is per-workout and `readWorkoutsInWindow()` returns up to 7 days, so
attaching last night's sleep to a five-day-old workout is meaningless, and no sleep-specific upload endpoint
exists. Phase 3 ships display-only, following the `OnTodayStepsRead` precedent ("display only — never
uploaded or awarded"). Defining the producer is Phase 3.5's job, which needs it anyway to feed rewards.
Privacy policy text is done; the Play Console Data Safety Health-info entry remains open and external.

---

## Phase 3.5 — Sleep reward (follow-up to Phase 3, gated on Phase 3 shipping first)

Decision #4's resolution: sleep **will** feed rewards, but not as part of Phase 3 (display-only, no
`SignalChecker`, no reward path — see Phase 3's exit). This is that follow-up, sequenced strictly after
3a/3b land (needs the sleep field already in `EvidenceBundleSchema` to exist as a target).

- [x] **3.5a. SHIPPED.** Resolved differently from the two options sketched below, and the difference
  matters: sleep did **not** get attached to a same-day workout, and did **not** get a dedicated
  sleep-only ingest path. It became a real `ACTIVITY_TYPES` member instead, mirroring the synthetic
  `steps` row, so it rides the existing ingest → verify → award → register path with no new endpoint.
  `HealthConnectReader.readSleepInWindow()` reads sessions off the shared watermark (like workouts —
  a sleep session is a discrete record with a stable HC id, so it needs none of the step day-window
  math) and `AndroidHostBridge.requestHealthRead()` fans them out alongside workouts and steps, so
  **one sync delivers all three, including when sleep is the only thing in the window.** Also fixed
  the display read: `readLastNightSleep()` was scanning a flat 7 days and labelling a five-day-old
  session "last night"; it is now scoped and filtered on `isLastNight`. The duration rides as
  `metrics.sleepMinutes` (the reward-feeding claim — the kinetic engine reads only `metrics`) plus
  the pre-existing `sleep` sub-object as the independent corroboration window.
  <details><summary>Original plan text (superseded)</summary>

  3a/3b shipped
  the schema field (`EvidenceBundleSchema.sleep`) and the on-device read, but nothing populates it and no
  sleep-specific upload endpoint exists (Phase 3 exit note). `3.5b` below has nothing to check without
  this. Decide and build the actual trigger: e.g. attach last night's sleep only to an upload alongside
  a **same-day** workout (never a stale one from `readWorkoutsInWindow()`'s 7-day lookback, which is the
  reason Phase 3 punted this), or add a dedicated sleep-only ingest path if no same-day workout exists.
  Whichever is chosen, wire it host/client-side (Unity + `HostBridge`) so a real upload actually carries a
  populated `sleep` sub-object before `SleepChecker` is written.
  </details>
- [x] **3.5b. SHIPPED as `SleepConsistencyChecker`** (`sleep-consistency`), registered in
  `VerificationEngine`'s constructor list and added to the `health_connect` source profile at
  **weight 0**, per the dilution rule. It enforces a physical invariant rather than a physiological
  one: you cannot sleep more minutes than elapsed, bounded by the *tighter* of the sleep sub-object's
  window and the activity's own window. Under-claiming passes (time in bed exceeds time asleep); only
  the mint direction rejects. It neutral-passes every non-sleep bundle, which it must — the profile it
  lives on also carries ordinary workouts and step-days. Without it a sleep bundle would clear the 0.6
  threshold on no evidence at all, because `distance-length` and `heart-rate` both neutral-pass a
  bundle carrying neither metric.
  <details><summary>Original plan text (superseded)</summary>

  Register it in `VerificationEngine`'s constructor
  list (Open/Closed — never edit the scoring loop for a new source). Per the weighted-mean dilution rule,
  it must join its profile as a **weight-0 gate or multiplier, not a weighted peer** — same treatment as
  `track-smoothness` (1d) and the same reasoning: a checker that always passes on presence alone dilutes
  an anchor-gated profile if given equal weight.
  </details>
- [x] **3.5c. SHIPPED.** Sleep grants XP/gold/stats through the normal two-phase digestion path and is
  excluded from weekly progress by `WEEKLY_CREDIT_EXCLUDED_TYPES` (`domain/activityTypes.ts`), filtered
  inside `countWeeklyCredits` rather than at its call sites — that function is shared by the tick's
  payout and `/state`'s progress bar, and a per-caller filter would eventually be applied to one and
  forgotten on the other. `dailyTick.ts` / `evaluateClosedWeek` were **not** touched.
  **Discovered while doing this:** `countWeeklyCredits` had no type exclusion at all, so `steps` earns
  a weekly credit *every day* — 7 free credits against a default `weekly_target` of 3. Left as-is
  deliberately (fixing it changes live behaviour for existing users) and tracked in `docs/TODO.md`.
  **Also accepted:** sleep's WIS axis collides with yoga/pilates/mobility under the per-axis daily stat
  cap, which does not exempt sleep the way it exempts steps — a day with both a WIS workout and a synced
  sleep yields one wis point, not two. Recorded in `activityTypes.ts` and `TODO.md`.
  <details><summary>Original plan text (superseded)</summary>

  Confirmed scope (decision #4): sleep grants
  its own XP/gold/stats (or item/skill, per decision #5's jsonb-payload note) through the normal two-phase
  digestion path (`awarded` → register), but must **never** be counted by whatever sums activities toward
  `weekly_target` (`dailyTick.ts` / `evaluateClosedWeek`) — that logic must not be touched to include sleep.
  </details>
- [x] **3.5d. SHIPPED — migration `0051_sleep_activity_type.sql`.** Seeds both the `sleep-consistency`
  weight (0) on the `health_connect` profile and the `kinetic.formula` coefficient
  (`byType.sleep = { sleepMinutes: 0.05 }` → ~24 XP for a full 8h night, deliberately below a real
  session's worth since sleep is passively recorded). Uses `jsonb_set` rather than the full-document
  restatement 0034 used: for a purely additive key, restating requires reproducing every key added since
  plus anything live-tuned right now, and any omission silently reverts a tuned value.
  The same migration re-partitions the overlap exclusion constraints — a sleep session spanning midnight
  legitimately overlaps an early-morning workout, so under the old 0026 predicate it would have
  **rejected that workout**. Sleep now gets the treatment steps already had (its own
  `activities_no_overlap_sleep`), mirrored application-side by `SELF_PARTITIONED_ACTIVITY_TYPES`.
- [~] **3.5e. Privacy policy DONE, Play Console still open.** `PrivacyPage.tsx`'s "Sleep data" bullet now
  says sleep is sent to and stored on our servers and used to award rewards, replacing the
  display-only-on-device wording. **Still open and external:** the Play Console Data Safety Health-info
  entry, which must be flipped to match **before this reaches real users** — `ingestActivity.ts` stores
  the whole `evidence_bundle` wholesale, sleep included, with no carve-out.

Exit — **STILL NOT MET, but gate (a) closed 2026-08-03.**

- **(a) Integration test — DONE.** `backend/src/services/game/activity/test/sleepActivityType.integration.test.ts`
  proves all three: a sleep session (22:00→06:00) and an *overlapping* early-morning run both insert as
  `accepted`; a second overlapping sleep row fails with `23P01`; and `countWeeklyCredits` returns 1, not 2
  — asserting 1 rather than 0 deliberately, since 0 would be indistinguishable from the query finding
  nothing. Green in the full suite (16 files / 136 tests), not just in isolation.
- **(b) Play Console Health-info disclosure — still open, still external.**
- **On-device dogfooding — still not done, and it would have FAILED if it had been.** Found 2026-08-03
  while building 4e: `devdriver/EvidenceBundleAssembler.buildJson` emitted only `activeEnergyKcal`,
  `durationSeconds`, `avgHeartRate` and `distanceMeters` — it never emitted `metrics.sleepMinutes`, never
  emitted `metrics.stepCount`, and never emitted the `sleep` sub-object at all. So every sleep bundle the
  dev driver has ever uploaded arrived with nothing for `byType.sleep = { sleepMinutes: 0.05 }` (0051) to
  multiply and nothing for `SleepConsistencyChecker` to corroborate: **sleep earned 0 XP, silently.** The
  failure mode was a zero, not an error, which is why nobody caught it. Fixed by replacing that function
  with the pure `hostlogic/EvidenceBundleJson.kt` (`toEvidenceBundleJson`), which emits all six metrics
  plus both sub-objects and is regression-tested on exactly the step-day and sleep cases that were broken.
  Phase 3.5's reward path has therefore **never actually been exercised end to end** — treat the first
  successful on-device sleep award as new, unproven work, not a re-verification.

---

## Phase 4 — Real periodic gym-ping + location foreground-service (UNBLOCKED — decision #2 resolved: host-side WorkManager)

Today only one app-open ping fires (`GameBootstrap.cs:618`; no timer) and there is **no** WorkManager /
AlarmManager / foreground Service anywhere in the host — only a lifecycle-bound `delay` loop in the dev
driver. The passive-session backend (`GymPingService`, `GymSessionFinalizer`, `gym_pings` 0035/0036) is
already built and waiting for a real trigger.

**Re-scoped 2026-08-04.** The text below was written 2026-07-24, four days before the Unity-as-base flip,
and assumed the host could ping and upload on its own. It cannot: `HostBridge.kt` forbids host app
networking, and every result leaves via `UnitySendMessage`, which needs a live scene and a
`HostBridgeReceiver` GameObject — neither exists when the app is closed. Two decisions followed:

1. **`ACCESS_BACKGROUND_LOCATION` is REJECTED, permanently.** It is not a rate limit but binary access
   control: since Android 10, a background `getCurrentLocation()` with only fine/coarse granted returns
   **null**, silently (`HealthConnectReader.currentGps()` already swallows that into null). One ping a day
   needs the same grant as a thousand. The cost is a Play Console declaration, prominent disclosure and a
   video review — and under Unity-as-base that declaration lands on the **partner's** listing, while 5h
   (signing/submission ownership) is still open. Nothing about the topology caused this; the same gate
   applied under UaaL and under Flutter. The flip only made the paperwork someone else's.
2. **The host got a NARROW networking exception** — background worker only, documented as exception 3 in
   `HostBridge.kt`. The guardrail's target is the award/verification *computation*, not byte transport;
   the worker carries raw signals the server independently re-scores.

- [x] **4a/4b/4c — SHIPPED 2026-08-04, on-device gate CLOSED 2026-08-07. Shape revised
  2026-08-04 (second pass): gated auto-start, not a manual tap.** Gym tracking is a
  **location-typed foreground service, auto-started off the
  existing app-open ping's own response** — never a passive background-location scheduler, and no
  "start gym session" button. `GameBootstrap.cs:618`'s one-shot ping already returns
  `{ atGym: boolean, gymId? }` (`GymPingService.recordPing`) with no FGS involved: a single fix under
  while-in-use `ACCESS_FINE_LOCATION` while the app is foregrounded needs no service at all — the FGS
  is only for keeping *updates* flowing once the app leaves foreground. So: (1) the app-open ping
  fires exactly as today, no notification; (2) only when that ping answers `atGym: true`, the host
  starts the FGS, which holds a 5-minute-cadence ping loop even with the app fully closed, on the
  plain `ACCESS_FINE_LOCATION` grant (`PeriodicWorkRequest`'s 15-minute floor doesn't apply to a
  service); (3) the FGS self-stops (`stopSelf()`) the moment it sees **2 consecutive `atGym: false`
  responses**, reusing `GymPingService`'s own close signal (`AWAY_PINGS_TO_CLOSE = 2`,
  `GymPingService.ts:10`) so host and server agree on session-over without a manual stop, plus a
  hard duration ceiling (TBD, ~3-4h) as a dead-man's switch. `RunSessionService` (4d) is very likely
  the same service, generalized — swap `PushNotifications.runOngoingNotification` for a gym-specific
  string, swap the start/stop caller from `RunSessionTracker.start/stop` to the ping-gated logic
  above. **Recovered vs. the first-pass (manual-tap) text:** no button — the notification only ever
  appears once the app-open ping has already confirmed the user is at a gym, which is also exactly
  what keeps it from firing on an ordinary app open. **Still traded away, unchanged from
  2026-07-24:** a gym visit on a day the user never opens the app is still not credited — true
  passive detection was never on the table, only the *manual-tap* requirement was removed. **Do not
  reintroduce the WorkManager gym-ping design** — at the 15-minute floor, `AWAY_PINGS_TO_CLOSE = 2`
  means ~30 min to detect departure, and credited duration (`last ping − first ping`) systematically
  under-counts.

  **What actually shipped, and TWO corrections to the text above.**

  1. **`RunSessionService` is NOT "the same service, generalized" — that was wrong.** Its own
     docstring is explicit that it is a *lifetime anchor holding no logic*: it works only because
     `RunSessionTracker` lives in the app process and does everything. The gym service has nothing
     to anchor — with the app closed there is no tracker, no Unity player and no listener — so
     `GymSessionService` **owns its work loop**: the ping, the fix, the POST, the counters, the
     ceiling, the `stopSelf`. The two genuinely share ~15 lines, now extracted as
     `startLocationForeground(...)` (the `startForeground` API gate plus the degrade-don't-crash
     `try/catch`). `RunSessionService`'s behavior is unchanged.
  2. **The HOST owns the gating ping, and NEVER requests the location permission.** The host does
     its own one-shot ping on launch (`UnityHostActivity.startGymSessionIfAtGym`) under networking
     exception 3, so **nothing was added to `HostBridge`** — the iOS port surface is untouched, and
     no partner change was needed to ship or dogfood. It only ever *reads* an existing
     `ACCESS_FINE_LOCATION` grant and silently no-ops without one;
     `UnityBridge.requestLocationPermission()` stays the only path that can raise a location dialog,
     called by Unity when it sees fit. A prompt on first launch is not acceptable — product decision
     2026-08-04. Accepted cost: two ping rows per app open while the partner's `GameBootstrap.cs:618`
     ping also fires, which costs nothing because credited duration is `last − first at-gym ping`.

  New files: `hostlogic/GymSessionEffect.kt` (the pure keep-going/stop state machine, 13 tests),
  `hostbridge/GymPinger.kt` (one ping, never throws), `hostbridge/GymSessionService.kt` (the FGS).
  `HOST_VERSION` → `04/08/2026-b`. **No new permission and no new Gradle dependency** — everything
  runs on platform `LocationManager` + `HttpURLConnection`, so the partner's `mainTemplate.gradle`
  is untouched. **Backend unchanged**, as designed; the only backend edit was fixing
  `openapi.ts`'s `POST /gyms/ping` 200 example, which documented `{}` and showed neither `atGym`
  nor `gymId`.

  Three decisions the design text left open, resolved in code: a **`Failed` ping (no fix, no
  network, 5xx) is neither an away ping nor a reset** — it carries the away count through
  untouched, because reading a blip as a departure truncates a real session and letting a blip
  reset the count delays departure detection to the ceiling; a **consecutive-failure cap of 5**
  (~25 min) ends a broken session sooner than the ceiling would; the **duration ceiling is 3h**.

  Known ceiling, left as a calibration knob: no wake lock and no `AlarmManager`, so Doze can defer
  a tick. Credit is unaffected (a deferred tick still lands with a real timestamp); only departure
  detection lags. Upgrade path if on-device drift proves real: `setExactAndAllowWhileIdle`.

  **PROVEN ON DEVICE — the gate is CLOSED (2026-08-07).** The gate was: at a registered gym, run
  the harness's "Run gym gate now" button, background the app fully, wait, then walk out and wait
  two ping cycles. Pass = the server ingests one `tracked_gym_session` whose credited duration
  matches real dwell time; a 60-second credit would have meant the background loop never ran (that
  is `MIN_SESSION_CREDIT_SECONDS`, the one-ping floor this whole phase exists to beat). The run
  passed, confirmed on a real phone. 4a/4b/4c are now verified, not merely code complete — the
  background loop survives a fully backgrounded app and credits real dwell time.

  Both contract follow-ups are now CLOSED (2026-08-04): `unity-bridge-contract.md`'s
  "foreground-only — no background service yet" claim was corrected when the doc was bumped to
  `04/08/2026-b`, and `unity-rest-contract.md` — which never documented `POST /gyms/ping` at all —
  was deleted outright, with Swagger UI (`/docs`) becoming the sole HTTP contract.
- [x] **4d. SHIPPED 2026-08-04 — `RunSessionService`.** A location-typed foreground service that is a
  **lifetime anchor, not a location owner**: the while-in-use check is per-process, so raising the process
  is enough and `RunSessionTracker`'s sampling code is untouched. `START_NOT_STICKY` (samples are in memory;
  a resurrected service would post "Run in progress" over a run whose samples are gone) and a `try/catch`
  degrading to `stopSelf()` — API 34+ throws `SecurityException` without a location grant, Android 12+ throws
  `ForegroundServiceStartNotAllowedException` on a background start, and neither may crash the partner's app.
  **Also fixed the bug 4d exposed:** `ACCESS_FINE_LOCATION` was never requested on the Unity path at all
  (`UnityHostActivity.onCreate` asked only for `POST_NOTIFICATIONS` and Health Connect), so run tracking was
  already collecting **zero samples** on the partner's build. Added `UnityBridge.requestLocationPermission()`.
  2f's "notification only after the server acks" gate was removed — an FGS notification is mandatory within
  5s of start, which a network round trip cannot promise. The `POST /activities/run-started` call stays.
- [x] **4e. SHIPPED 2026-08-04 — `HealthSyncWorker`.** `CoroutineWorker`, 30-minute periodic,
  `NetworkType.CONNECTED` as the ONLY constraint (deliberately not `requiresBatteryNotLow` — a phone at 14%
  silently not syncing is the exact drift this closes), `ExistingPeriodicWorkPolicy.UPDATE` so a future
  cadence change reaches existing installs. Enqueued from `UnityHostActivity.onCreate`, wrapped in
  `catch (Throwable)`: if the partner omits the `androidx.work` line from `mainTemplate.gradle`,
  `WorkManager.getInstance()` throws `NoClassDefFoundError` **at onCreate** and their app will not launch.
  Four things landed differently from the scoping text below, each for a reason worth carrying forward:
  1. **The Changes API was dropped entirely.** Its premise — "a full 7-day re-scan is wasteful hourly" —
     stopped being true at commit `7bcb2b8`: `syncWindowStart` already narrows every read to the watermark,
     so at a 30-minute cadence each read covers ~30 minutes. The token would have added an expiring value
     to persist, a data-restore invalidation path, and a mandatory watermark fallback anyway — three new
     failure modes to save nothing.
  2. **The watermark rule now DIFFERS between the two paths, deliberately.** `AndroidHostBridge` still
     advances on read success because it cannot see whether Unity's later upload worked. The worker performed
     its own uploads, so it advances only when every outcome is terminal. `HealthSyncState.markSynced` became
     monotonic (`maxOf`) because two writers now exist with no lock. Do not "fix" these into agreement.
  3. **`activityId` is deterministic** (`UUID.nameUUIDFromBytes(sourceWorkoutId)`), so a retry reuses the
     same id and the server's upsert on `activity_id` is a true no-op, rather than a second insert caught
     only by the `source_workout_id` unique index.
  4. **The background HC permission is a THIRD permission set**, checked independently. It could go in
     neither existing one: `requiredPermissions` is `containsAll`-gated by both Activities (adding it
     recreates the Phase 3a bug where granting everything-but-one reads as fully denied), and
     `optionalPermissions` is `containsAll`-checked by the sleep path (adding it would break sleep for
     anyone who declines background).

  <details><summary>Original scoping text (superseded)</summary>

  Folded in 2026-07-27 — same `WorkManager`, not a second scheduler; steps locked 2026-07-27, full
  rationale in spec § *Health Connect background sync — permission-staged onboarding*. Runs a background read + upload path so `activities.created_at`
  lands near `started_at` instead of at next app-open. **This is a correctness dependency for
  migration `0046`, not a nice-to-have:** `0046` re-keyed the daily XP cap to the local **receipt**
  day (`created_at`) — deliberately, since `started_at` is client-controlled and the server clock is
  the only mint bound a forged bundle cannot move. That choice is only *fair* if `created_at` tracks
  reality. Until this ships, a user who trains Monday and Tuesday but opens the app once on Wednesday
  has both workouts share one receipt day and collide against a single day's XP budget — a known,
  documented interim window (see spec § *Daily XP cap*). Reward collapse and weekly credit are already
  immune (they key on `started_at`); only the XP cap is exposed.
  - **Permission sequencing (staged, not requested all at once):** (1) app launch requests the
    existing foreground HC read set, unchanged; (2) after onboarding, explain the benefit
    ("We'll automatically detect your activities even when the app isn't open.") as a UI moment, not
    a system prompt; (3) only then request Health Connect's background-read permission
    (`PERMISSION_READ_HEALTH_DATA_IN_BACKGROUND`). A decline degrades to today's foreground-only
    behavior — never blocks the app or re-nags aggressively.
  - **Cadence:** reuses 4a's periodic `WorkManager`, scheduled at a **30–60 minute** interval (looser
    than 4a's 5-minute gym-ping cadence — no live-session responsiveness need here).
  - **Incremental read, not a full re-scan:** each run reads only data new since its last successful
    run — Health Connect's Changes API token, or a stored last-sync timestamp if the token proves
    unreliable across process death — rather than reusing `readWorkoutsInWindow()`'s full 7-day
    re-scan-and-dedup approach, which is correct but wasteful at an hourly cadence.
  - Inherits 4b/4c's failure handling (permission revocation without crash-looping, doze/battery).
    Reliability is still best-effort — OEM battery killers mean some users still batch-sync, which is
    why the interim window stays documented rather than assumed away.
  - Per the host testing convention, the changes-token/watermark bookkeeping is a pure-function
    candidate for `android-host/hostlogic/` with a plain JUnit4 test; the Changes API call and worker
    lifecycle are dogfooded via the dev driver, not unit-tested.
  </details>

**Exit — ALL GATES CLOSED.** 4d and 4e are written, compile into the shipped `.aar`, and are
covered where the convention allows: 13 new pure-function tests in `:hostlogic` (99 total green) for bundle
assembly and the sync decision. Gates 1 and 3's device tests have happened and passed (below); gate 2 was
closed as obsolete by `0054`, which fixed in the server the fairness property that gate was waiting on a
host behaviour to deliver.

All three gates closed (2026-08-04, 2026-08-06, 2026-08-05):

1. **CLOSED 2026-08-04 — the lifetime-anchor assumption is confirmed, not just plausible.** 4d rested on
   the claim that a location-typed foreground service keeps `RunSessionTracker`'s existing `LocationManager`
   updates flowing, because the while-in-use check is per-process. Device test: backgrounded the app, screen
   off, walked, stopped. Result: `sampleCount: 176` over `durationSeconds: 361` — expected `≈ 361 / 2 = 180.5`
   at `RunSessionTracker.SAMPLE_INTERVAL_MILLIS = 2_000L`, actual 176 (97%), consistent with continuous
   background sampling rather than the handful a foreground-only run yields. The service-owns-the-listener
   fallback is not needed.
2. **CLOSED — OBSOLETE, superseded by `0054`.** This gate asked for a device experiment: record real
   workouts on two consecutive local days, do not open the app, then assert both rows carry non-zero
   XP. Its stated control was *"without the worker both rows share one `created_at` day and the second
   clamps against the first's spend"* — and after `0054` that control no longer exists. The receipt-day
   budget is now `maxXpPerDay × maxCatchUpDays`, so both rows reward in full **with or without** the
   background-sync worker. Run as written, the experiment would pass trivially and prove nothing.

   The property it was guarding is now an executable server test — *"rewards every real performed day
   in a batch sync"* in `dailyXpCap.integration.test.ts` — which is a better gate than a device
   experiment anyway: it runs in CI and cannot silently regress. Closing this on the migration rather
   than on the device, deliberately, because the alternative is someone closing it on a null result.

   `4e` remains worth shipping for latency (fresher data, better nudge timing) — it is simply no
   longer load-bearing for reward fairness.
3. **CLOSED 2026-08-05 — sleep syncs end to end with non-zero XP.** Device test (manual sync via the dev
   driver's "Run health sync now"): a real Health Connect sleep session (`sleepMinutes: 473`, ~7.9h)
   synced and was accepted — `activities` row `71950775-f6e0-49f5-9c60-cac8b3bf1d90`,
   `awarded: {xp: 23, gold: 23, stats: {wis: 1}}`. The sleep→XP path works end to end; the broken
   dev-driver assembler this was blocked on (Phase 3.5's exit) is confirmed gone.

Also still open: the partner's `mainTemplate.gradle` line (documented in the bridge contract, not shippable
by `deliver-hostbridge.yml`, which carries only the `.aar` and the contract doc), and Play's API 34+
Foreground Service permissions declaration for `FOREGROUND_SERVICE_LOCATION` — far lighter than the
background-location review this phase rejected, but it lands on the partner's listing, same 5h question.

Residual accepted, not fixed: a workout ending 23:30 and uploaded at 00:05 still lands on the next local
day. The window narrows from *days* to *one midnight crossing* — same class as `1.5-pre`'s accepted
residual — and Doze plus OEM battery killers make 30 minutes a floor, not a guarantee.

---

## Phase 5 — Release-gated cluster (3 gates: engine decided [CLOSED], client live on device [CLOSED], Play install [CLOSED])

**Reshaped 2026-07-28 by the Unity-as-base flip** (`Interpretable-Context-Methodology/departments/engineering/docs/unity-plugin-topology-decision.md`, sibling repo). The partner
studio now builds the APK from their Unity project; our host ships as `hostbridge.aar`. Several items
below changed owner, and one dissolved outright — read the reasons, they are not cosmetic.

Three conditions gate this phase: (1) engine decided — **CLOSED 2026-07-28** (Unity); (2) client live
on device — **CLOSED**; (3) Play / internal-testing install exists — **CLOSED**.

**All three gates and every item below (5b–5h) confirmed CLOSED 2026-08-25**, verified against the
partner repo rather than this file's own checkboxes, which had gone stale — none of them had been
ticked as the work actually landed. Evidence: `reign-and-gain-unity` is in active daily production
iteration (issue-tracker daily digests, ongoing tutorial/combat/UI PRs against a live app); their
issue #441 ("Ship an updated build to Play") is closed and states signing credentials were "already
configured" — closing 5d/5h; deep-link and sign-in flows are exercised by real production bugs (their
#623, #758), not bring-up tests — closing 5b/5c; three more `hostbridge.aar` delivery tags shipped
after this doc was last touched (`24/08/2026`, `-b`, `-c`), each opening and merging a partner PR —
closing 5f/5g; and the push-token round trip (5e) closed differently than scoped below — the host now
calls `registerPush()` itself from `UnityHostActivity.onCreate`/`storeAuthTokens` and POSTs `/devices`
directly, rather than relying on a Unity-side receiver (see `HANDOFF.md` PR #143 history) — no partner
C# change was needed after all.

A friends-and-family launch is live on this build, per `CLAUDE.md`. This phase and this file are now
pure history. Full task-by-task text preserved below for the reasoning trail; content has been moved to
`Interpretable-Context-Methodology/inbox/` and this file is being deleted from this repo — see the
pointer left in `CLAUDE.md`.

- [x] **5a. LAUNCHER swap-back — DISSOLVED 2026-07-28, not completed.** Under Unity-as-base the
  partner's build is the launcher by construction: their manifest fragment names `UnityHostActivity`
  directly (`unity-client-brief.md` § 7, partner's `reign-and-gain-unity` repo, `docs/`). The problem this ticket existed to solve —
  `DevDriverActivity` holding LAUNCHER in the shipped app — cannot occur, because `:app` is a dev
  harness that is never shipped. Recorded rather than deleted so the next reader doesn't hunt for it.
- [x] **5b. CLOSED — confirmed live in production, not just re-run once.** The invite deep-link and
  the manifest-merge claim it rests on are exercised continuously by real users; the partner's own
  bug tracker has production issues against the flow (e.g. their #623, a guild invite tapped
  mid-session), which only happens once the flow already works end to end on their build.
- [x] **5c. CLOSED.** Native Google Sign-In runs on the partner's build and from Play installs — same
  evidence as 5b, a live app with real sign-ins daily. The browser fallback's removal was not
  separately confirmed and is not urgent now the primary path is proven in production.
- [x] **5d. CLOSED.** Resolved as a decision, not left open: the partner's issue #441 ("Ship an
  updated build to Play") states the signing credentials were "already configured" when written, and
  that issue is closed. Whichever answer landed (partner signs / we sign their AAB), it shipped.
- [x] **5e. CLOSED — differently than scoped.** The Unity-side `OnPushTokenReceived`/`OnPushTokenError`
  receiver was never needed: the host now calls `registerPush()` itself, from
  `UnityHostActivity.onCreate` and `AndroidHostBridge.storeAuthTokens`, and POSTs `/devices` directly
  — no partner C# change required. Shipped in PR #143's history (see `HANDOFF.md`).
- [x] **5f. CLOSED.** Three more `hostbridge.aar` delivery tags shipped after this file was last
  updated (`24/08/2026`, `-b`, `-c`), each triggering `deliver-hostbridge.yml` and opening a partner PR
  that merged.
- [x] **5g. CLOSED.** Partner integration is done — a live, feature-complete app ships from their
  build with `UnityHostActivity` as launcher; targetSdk/manifest details are theirs to hold now.
- [x] **5h. CLOSED.** Same evidence as 5d — issue #441 confirms the signing-key question and Play
  Console ownership were both settled before that issue closed.

Exit — **MET.** All three install paths verified in production by a live friends-and-family launch,
not just a pre-release checklist.

---

## Sequencing summary

| Phase | Blocked? | Surface | Ships value |
|---|---|---|---|
| 1 Anti-cheat hardening | No | backend + SQL | The moat — cross-account dedup, stat caps, anti-spoof |
| 1.5 Weekly-credit distinct-day counting | No, but sequenced after 1b/1c ship | backend (tick) | Closes `0045`'s documented interim gap — collapsed duplicates stop inflating weekly credit |
| 2 Notification pipe | No (decision #1 resolved) | host + backend + policy | Weekly nudge + truthful Data Safety |
| 3 HC sleep | No | host + backend + policy | Sleep display + HC declaration |
| 3.5 Sleep reward | No, but sequenced after 3a/3b ship | backend | Sleep starts feeding rewards, never `weekly_target` |
| 4 Periodic ping | No (decision #2 resolved) | host (WorkManager) | Real gym sessions, not one-shot pings |
| 5 Release cluster | Engine CLOSED; client-live-on-device + Play install still OPEN | `.aar` + partner integration + assetlinks | Pre-release correctness |

Phases 1–4 are all fully unblocked and can run in parallel (disjoint files) now that decisions #1 and #2
are resolved. Phase 1.5 and Phase 3.5 are each unblocked in scope but sequenced after their own
prerequisite lands (1b/1c for 1.5; Phase 3's sleep field for 3.5). Phase 5 is the only remaining gated
phase, but the gate is no longer purely external: engine choice (the product decision) is closed,
leaving Unity-live-on-device (internal engineering, ours to unblock) and Play install (external,
Google-side) both open.
