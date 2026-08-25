# PLAN-unity.md — Fitness RPG (Unity Client Route)

> **Status: the chosen client route** (engine decided 2026-07-28 — Unity 6 LTS). The *engine* choice
> below is unchanged; what changed the same day is **who builds the APK**: the topology inverted from
> UaaL to **Unity-as-base**, and the Unity client moved to a partner studio's repo. Our host now ships
> as `hostbridge.aar`. Statements below phrased as "host embeds Unity" are superseded — see
> `Interpretable-Context-Methodology/departments/engineering/docs/unity-plugin-topology-decision.md` § Reversal (sibling repo,
> queued for filing) and `unity-client-brief.md` (now in the partner's `reign-and-gain-unity`
> repo, `docs/`). This
> document is the **architecture rationale + phase history** for that route; the **active execution
> plan is `docs/PLAN-closeout.md`**. Both files contain a "Phase 5" and they are different things —
> this one's is hardening/iOS, PLAN-closeout's is the release-gated cluster. Check which you mean.
>
> It re-architects **only the client**; the backend (Fastify + Supabase, server-authoritative
> verification, append-only ledger, daily tick, data-driven GameConfig) is **unchanged** and is
> referenced, not relitigated, here.
>
> **Note on inline `docs/PLAN.md` references below:** that file (the superseded Flutter+Flame plan)
> was deleted 2026-07-27 and is archived at git tag `archive/flutter-flame-phase1`. The references
> are left in place as historical context for why decisions were made — they are not live links.

## Context
A mobile fitness RPG where real-world **verified** workouts translate into RPG progression (XP, STR/DEX/CON/WIS, gold, loot) and missed workouts inflict roguelike consequences, with opt-in async guilds chipping at a weekly world boss. The defensible moat is **cheat-resistant, server-side verification**. This route swaps the render engine from Flutter+Flame to **Unity (richer 2D)** because the team considers the game the primary component. Built by two junior founders on a ~10-week MVP sprint; low-thousands of users; async (no realtime netcode). **Android-first**, with the iOS build deliberately kept cheap to add later.

## Decisions Made
- **Engine: Unity 6 LTS (2D), replacing Flame.** Driver is richer 2D game feel as the centerpiece; timeline stays ~10 weeks. The Flutter+Flame route in `docs/PLAN.md` is set aside, not deleted.
- **Client topology: Unity-as-base + a Kotlin plugin `.aar`, Android-first** *(revised 2026-07-28; originally "native Android (Kotlin) host + Unity-as-a-Library (UaaL), Option D" — preserved here as superseded)*. Unity's own build generates the APK; our host ships as a plugin the partner drops into `Assets/Plugins/Android/`, and their manifest fragment names `UnityHostActivity` as launcher so the host still owns `onCreate`. **The wrapper rejection is unaffected and still stands**: no `flutter_unity_widget`/`flutter_embed_unity`, no `@azesmway/react-native-unity` — the dead-end risk lives in those community wrappers (the RN wrapper's last release was Nov 2024 with ~100+ open issues; the original Flutter widget only reached Unity 6 via an experimental branch/fork). Both topologies depend only on Unity itself.
- **iOS kept cheap via a thin `HostBridge`.** All platform-specific native code is confined behind one small interface (health read, OS permissions, secure token storage, push, attestation). The iOS port is "re-implement `HostBridge` against HealthKit" — nothing else.
- **Logic boundary (server-authoritative moat intact).** Unity owns *significant presentational and client-plumbing* logic (animation, combat choreography, prediction, networking, evidence-bundle assembly). The **server remains authoritative for every awarded value** — XP, gold, loot, stats, boss HP, and verification scoring. No award/progression logic in the (decompilable) client.
- **Networking & auth live in Unity/C#, not the host.** The shared C# layer owns the REST client, auth, `GET /state` fetch, and evidence upload. The native host only supplies raw health signals + a securely stored token. Rationale: maximizes cross-platform reuse (iOS-cheap), and since the evidence bundle is *claims, never verdicts* that the server re-scores, a decompiled client can only submit forgeries the server already rejects — near-zero added security cost.
- **Gameplay reward model: DEFERRED (explicit open decision).** Whether playing earns rewards via stamina-gating, an independent capped track, or a hybrid is **not decided yet**. Locked invariant regardless of later choice: **gameplay rewards are resolved server-side; the client never computes them.** MVP rewards come from verified workouts plus the already-server-resolved dungeon/boss loop. A server-authoritative run-resolution endpoint (and headroom for a stamina resource) is built so any model slots in later without rework.
- **Inherited unchanged** (from `CLAUDE.md`/`PLAN.md`): Fastify single service; Supabase Postgres/Auth/Storage; composable signal-checkers → weighted trust score; **two-phase digestion** (a verified sync records a *pending* award; a client-triggered `POST /activities/register` applies all pending gains exactly-once — supersedes the earlier instant credit) + daily consequence tick; REST/JSON with a single `GET /state` snapshot; append-only contribution ledger; data-driven GameConfig; Tier-1 verification sources (HealthKit/Health Connect, Oura, accel-presence, geofence-negative, Strava; `manual` has since split into `manual` + `tracked_gym_session`); weekly-target miss rule; gear = durability-scaled flat bonuses; RLS read-only clients.

## Stack
**Unity 6 LTS (2D, C#)** generating the Android app, with a native **Kotlin** plugin `.aar` supplying the platform layer, over the **unchanged Fastify (Node.js/TypeScript) + Supabase** backend; health read natively and re-validated server-side; the iOS port is deferred but kept cheap behind a thin `HostBridge`.

## System Components

> Backend services are unchanged from `docs/PLAN.md`; only client-side rows are new or reshaped.

| Component | Responsibility | Key Dependencies |
|---|---|---|
| `NativeHost` (Kotlin, **iOS later: Swift**) — ships as `hostbridge.aar` | The only platform-specific layer: read Health Connect/HealthKit, OS permission prompts, secure token storage (Keystore/Keychain), push registration. Still owns `onCreate` (its Activity is named launcher by the consuming project's manifest fragment), so player lifecycle/memory management is retained | Health Connect SDK, OS keystore |
| `HostBridge` (interface) | Narrow contract between host and Unity — host→Unity: raw health signals, auth token, permission status; Unity→host: "request health read", "store token", "register push". **The sole iOS port surface.** | Defined once, implemented per platform |
| `UnityGameModule` (C#) — **built and owned by the partner studio** since 2026-07-28 | Consume `GET /state` snapshots and render dashboard/character/dungeon/boss; run presentational + prediction logic; own the REST client, auth flow, and evidence-bundle assembly/upload. **Zero authoritative game logic.** Built against `unity-client-brief.md` (partner's `reign-and-gain-unity` repo, `docs/`). | HostBridge, GameStateAPI |
| `GameStateAPI` (server, unchanged) | Serve REST `GET /state` snapshots to the Unity client | Fastify, all services |
| `VerificationEngine` (server, unchanged) | Compose signal-checkers → weighted trust score; re-validate evidence bundles | Postgres, GameConfig |
| `KineticTranslationEngine` (server) | Map verified activity → XP + STR/DEX/CON/WIS; for `tracked_gym_session` the effective activity type is resolved server-side from the matched venue's discipline, overriding the client's claim | VerificationEngine, GameConfig |
| `GymService` + venue validation (server, **new**) | User-gym registration (atomic RPC), presence scoring for gym sessions, hybrid venue validation (Overpass→Google→admin review; reject-only fail-open), per-venue discipline (strength/yoga/pilates/dance) | Postgres, Google Places / Overpass |
| `CharacterService` / `EconomyService` (server, unchanged) | Progression, inventory, gold, loot, gear degradation | Postgres, GameConfig |
| `GuildService` / `CombatResolver` (server, unchanged) | Append contribution rows; boss HP = start − Σ contributions; **server-resolve dungeon runs** (the seam for future gameplay rewards); miss penalties | Postgres (ledger), GameConfig |
| `DailyTick` (server, unchanged) | Scheduled sweep: detect misses, apply penalties, accrue boss damage, prompt | CombatResolver, NotificationService |
| `NotificationService` (server, unchanged) | Push: loss-aversion + guild-shield alerts | FCM / APNs |
| `GameConfig` (server, unchanged) | Formulas, drop tables, boss HP as runtime data | Postgres |

## Implementation Phases

> Status legend: `[x]` done · `[~]` partial (see note) · `[ ]` not started.

> **Analytics is a guiding line across all phases** (this is an MVP to *learn from*).
> Each phase ships its events with its features — server-side only, Postgres-native.
> Most learning is a SQL view over an existing append-only table; only genuinely
> homeless events go through `analytics_events`. Definitions, KPIs, and the
> per-phase instrumentation checklist live in `docs/ANALYTICS.md`. (This is product
> analytics; *operational* observability — request logs, the tick alarm — is Phase 5.)

### Phase 1 — Foundation + Native Host Shell (Weeks 1–3)
**Goal:** Auth, schema, health data flowing from the Kotlin host to the server.
**Tasks:**
- [x] Scaffold Fastify + Supabase (CI, env, migrations) — as `docs/PLAN.md` Phase 1.
- [x] Core schema: users, characters, activities, contributions (ledger), GameConfig tables.
- [x] Kotlin host: Health Connect read + permission flow + OS manual-entry metadata flags.
- [x] **Define the `HostBridge` interface now** (before Unity exists) so the contract is frozen early and the iOS surface is explicit.
- [x] Auth (Supabase Auth) + secure token storage in Keystore. *(SecureTokenStore now holds BOTH tokens; host-owned silent refresh is implemented host-side — `storeAuthTokens` + `SupabaseTokenRefresher` (refresh_token grant) + JWT-exp check, frozen on the bridge. Unity's login (native Google Sign-In, browser PKCE fallback) now obtains the pair; on-device proof of a live refresh remains. Contract: `docs/unity-bridge-contract.md` "Login: the two paths, end to end".)*
- [x] Evidence-bundle contract (manual-entry flag, accel-presence boolean, GPS-context) + upload. *(Now carries `source`; activityType is enum-validated.)*
- [x] Analytics foundation: `analytics_events` sink + analysis views + `docs/ANALYTICS.md` (the learning baseline); track malformed uploads (`evidence_invalid`).
**Exit criteria:** ✅ **Met** — a real HyperOS device authenticates, reads Health Connect, and uploads an evidence bundle the server persists end-to-end (via the throwaway dev driver; no Unity yet).

### Phase 2 — Verification & Translation (Weeks 4–5)
**Goal:** Server independently turns verified workouts into RPG progression. (Backend-only; mirrors `docs/PLAN.md` Phase 2.)
**Tasks:**
- [x] `VerificationEngine`: composable signal-checkers → weighted trust score; server-side re-validation. *(Source-aware: per-source profiles from gameconfig; unconfigured sources fail closed. The roster has grown well past the MVP three: manual-entry, provider-flagged, accel-presence, geofence-negative, gym-presence, distance/length, heart-rate, track-consistency, metric-rate — plus coverage-factor and magnitude-band helpers.)*
- [x] `KineticTranslationEngine`: config-driven activity → XP + STR/DEX/CON/WIS. *(Activity→stat axis is a single-source map in `domain/activityTypes.ts`. Reward **shape** is now data-driven too: the `kinetic.formula` gameconfig row holds a per-activity-type metric→coefficient map, XP = floor(Σ metric×coeff); `translate()` is async and self-loads config, with a shape guard that degrades a missing/legacy row to the in-code `DEFAULT_KINETIC_FORMULA`. Migration `0017_seed_kinetic_formula.sql` seeds all 9 types and is applied to the live DB (migrations pushed through `0026`); `DEFAULT_KINETIC_FORMULA` remains the in-code safety net.)*
- [~] `CharacterService` + `EconomyService`: progression, gold, intermittent loot. *(CharacterService.registerPendingActivities applies pending XP/stats/gold at register time via the `register_pending_activities` RPC; loot — `economy.rollLoot` — still a TODO.)*
- [x] Two-phase digestion: a verified sync records a **pending** award; a client-triggered `POST /activities/register` applies all pending gains exactly-once. *(Replaces instant credit; cross-source dedup added; proven on-device via the dev driver's Sync→Claim loop.)*
- [x] **Gym-session vertical** *(added post-plan)*: `manual` split into `manual` + `tracked_gym_session` (`0031`); user-registered gyms with an atomic registration RPC (`0029`) + `GymPresenceChecker`; venue-validation seam (`0032`: `validation_status`, hybrid Overpass→Google→admin review, reject-only fail-open); per-venue **discipline** variants (`0033`/`0034`: strength/yoga/pilates/dance, auto-detected from Google `primaryType` with user-declared fallback) — a gym session's reward stat/XP is resolved server-side from the venue discipline, **ignoring the client's activityType claim** (pilates→WIS, dance→DEX).
- [x] **Anti-farming hardening** *(added post-plan)*: per-user activity-overlap exclusion constraint (`0019`), kinetic reward ceiling (`0027`), per-user rolling-24h XP cap (`0028`), per-user rate limits on the award-path writes.
- [~] Oura + Strava ingestion. *(Strava is DONE: OAuth connect + `POST /strava/sync` pulls through the SAME ingest pipeline as client uploads — dedup → eligibility → trust → pending — with token refresh and a provenance-aware trust profile; migrations `0015`/`0021`. Oura remains deferred.)*
**Exit criteria:** ⏳ **Partial** — a verified Health Connect workout records a **pending** award and a client-triggered register applies it to XP/stats/gold exactly-once (Sync→Claim proven on-device); a manual/couch-logged entry is rejected server-side; cross-source duplicates are dropped; accept-rate/per-checker views exist. Remaining: Oura ingestion and loot (`EconomyService.rollLoot` is still a stub).

### Phase 3 — Embed Unity + State Rendering (Weeks 6–7)
**Goal:** Unity embedded via UaaL, owning networking, rendering live server state.
**Tasks:**
- [x] Integrate Unity 6 LTS into the Kotlin host via Unity-as-a-Library; wire the `HostBridge` implementation. *(`UnityHostActivity` subclasses `UnityPlayerGameActivity`, registers the bridge for JNI, drives HC permissions, and serializes raw signals → Unity; wire convention frozen in `unity-bridge-contract.md` (partner's `reign-and-gain-unity` repo, `docs/`). The `unityLibrary` export is a machine-local artifact — never committed; `:app`/`:unityLibrary` are conditionally excluded from the Gradle tree when absent, which is also why CI tests only the pure-JVM `:hostlogic` module. `DevDriverActivity` remains the LAUNCHER; swapping `UnityHostActivity` back is pending an on-device dogfood pass.)*
- [x] Unity/C# REST client + auth + `GET /state` fetch; host hands raw health signals + token across the bridge; Unity assembles and uploads the evidence bundle. *(Built: `RestClient`/`StateFetcher`, `AuthService` with native Google Sign-In as the primary login and browser PKCE retained as fallback — Unity obtains the token pair interactively and hands it to the host, which owns silent refresh — `EvidenceBundleAssembler`, and `GameBootstrap` auto-sync + on-focus re-sync. Exercised through the code-only `DevDriver` OnGUI harness.)*
- [~] Unity renders dashboard + character from snapshots (presentational only). *(View LOGIC shipped — `DashboardView`/`CharacterView` event wiring off snapshot models. Scene layout/styling is explicitly out of our scope — owned elsewhere; see the CLAUDE.md scope split — so handlers are exercised via `DevDriver`, not a real scene.)*
- [~] **Onboarding source-selection screen.** On first launch `GET /state` returns `onboardingComplete=false` → render the picker from `GET /sources` (`catalog` lists each logger with an auto/manual badge; `manual` only appears when `manualLoggingEnabled`) → user selects one or more → `POST /sources` → continue into the game. Backend is built (routes `sources.ts`, `SourceService`, `user_sources` table, evidence-bundle `source` field, source-aware `VerificationEngine`) AND the C# logic is built (`SourcePickerView`/`SourceRow` wiring + `SourcesService` REST calls); only the scene layout — out of our scope — remains.
- [ ] **Reachable "Sync / Sources" settings screen** from an in-game menu — re-uses the same `GET/POST /sources` to show connected loggers and edit the selection; auto sources show `status`, manual sources expose a "log a workout" entry. (Strava OAuth connect is now DONE backend-side; this screen just renders the seam.)
- [x] *(added post-plan)* **Guild + gym client plumbing** — `GuildService.cs` + `GuildView` wiring (create/join/invite/referral flows) and `GymsService.cs` (gym registration) against the Phase-1 guild backend and gym routes, driven via the `DevDriver` screen-state readout.
- [ ] Lifecycle/memory management: single-instance handling, pause-on-background, memory budget against the UaaL resident-memory floor.
**Exit criteria:** App launches into Unity; the embedded engine fetches and renders live character state over REST; an evidence bundle round-trips through the Unity layer and credits correctly.

### Phase 4 — Roguelike Loop, Guilds & Combat Render (Weeks 8–9)
**Goal:** Consequences, async social play, and server-resolved combat animated in Unity.
**Tasks:**
- [~] `GuildService` (opt-in), append contribution rows, world-boss seeding from config. *(Phase 1 shipped ahead of schedule: guilds, membership, unified personal invite code, two-sided referral bonus gated on the invitee's first verified workout — `0030`, `routes/guilds.ts`, RPC integration tests. Contribution-ledger writes + world-boss seeding not started.)*
- [ ] `CombatResolver`: boss HP from ledger sum; **server-resolved dungeon-run endpoint** (client submits the run, server resolves with server stats/RNG, returns bounded loot — this is the future-gameplay-reward seam); miss penalties. *(Still a throwing stub.)*
- [~] `DailyTick`: scheduled miss sweep, penalties, boss-damage accrual; `NotificationService` reminders/alerts. *(Miss sweep is BUILT ahead of phase: idempotent per (user, week-close localDate), timezone-anchored weekly-target evaluation, grace-window reversal when late data lands — `0023` + integration test. Boss-damage accrual + notifications remain.)*
- [ ] Unity renders dungeon/boss/combat choreography and prediction from server-resolved outcomes.
**Exit criteria:** Missing a workout visibly damages character and guild within a day; a dungeon run resolves server-side and Unity animates the returned result; a boss loses HP from members' real verified workouts; penalty pressure and boss progress are queryable via `v_roguelike_loop` / `v_boss_progress`.

### Phase 5 — Hardening, Memory/Perf & iOS-Readiness (Weeks 10+, buffer)
**Goal:** Production readiness on Android and a proven-cheap path to iOS.

**Status, checked 2026-08-25 before retiring this file:** most of this phase closed without ever
being ticked here. Structured request logging (`pino`) is in `backend/src/logger.ts` and wired
through `app.ts`; `backend/src/routes/health.ts`'s own doc comment confirms a tick health alarm
already polls `/health`. Background sync, Play Store health-data declaration, and the anti-cheat
security items all shipped — see `docs/PLAN-closeout.md` Phases 1–4 (also retired to
`Interpretable-Context-Methodology/inbox/` alongside this file) for the shipped detail. Android passed
Play review: a friends-and-family launch is live, per `CLAUDE.md`. **Genuinely still open:** the
iOS-readiness audit — untouched, and iOS stays deferred per CLAUDE.md's locked decision, so nothing
here is time-pressured. Unity memory/LMK tuning is no longer ours to do — the client is the partner's
since the 2026-07-28 topology flip.

**Tasks (original text, kept for the reasoning trail):**
- [x] Operational observability: structured request logging, error tracking, the daily-tick health alarm. (Product analytics — events + views — is built incrementally from Phase 1 per `docs/ANALYTICS.md`.)
- [x] Failure handling: revoked permissions, data gaps, partial syncs, tick retries/idempotency.
- [x] **Background sync trigger (host WorkManager).** Hourly worker reads Health Connect while the app is closed, plus an "uncollected workout" notification. *(Deferred here from the sync-trigger work. The foreground half already ships: app-focus re-sync wraps `AutoSync` in Unity's `OnApplicationFocus`, gated on `gameLive`, in `GameBootstrap.cs`. New lifecycle/battery surface — hence Phase-5 hardening.)*
- [x] Security: rate limiting, RLS/row-ownership checks, evidence-bundle audit retention, secrets; Tier-2 attestation (Play Integrity) scoped behind `HostBridge`. *(Shipped early: per-user rate limits on award-path writes, rolling-24h XP cap, column-revoked trust flags (`0018`), service-role-only `user_gyms` writes (`0032`), admin venue-review routes behind an admin key. Play Integrity plan T14 exists on an unpushed branch — see `CLAUDE.md`/`HANDOFF.md`.)*
- [ ] **Unity memory/LMK tuning** on cheap Android — now the partner's concern, not ours; not tracked further here.
- [x] Play Store health-data declaration + permission copy.
- [ ] **iOS-readiness audit:** confirm `HostBridge` is the *only* iOS-specific work — verify no networking, game, or progression logic leaked into Kotlin; document the Swift/HealthKit port checklist. Genuinely still open — see `docs/TODO.md`.
**Exit criteria:** Android passes Play review; the daily tick is monitored and idempotent; a single failed external API degrades but doesn't break the system; `HostBridge` is documented as the sole iOS port surface.

## Build-Time Unknowns
_Measurements to take during development — not design decisions._
- **Unity resident-memory footprint on cheap Android** (the ~80–180 MB UaaL floor): does the app survive backgrounding without the low-memory killer; how much can be trimmed.
- **Unity cold-start and resume latency** on target low-end devices.
- **UaaL build-pipeline stability across a Unity LTS patch bump** — how much breakage a version bump costs, to budget upgrade windows.
- **Bridge serialization overhead** for `GET /state` snapshot size at realistic character/inventory complexity.
- _(Carried over)_ false-accept/reject rates of verification filters; daily-tick runtime as guild/boss counts grow; whether the contribution ledger needs added indexes/partitioning.

## Out of Scope (for now)
- **Gameplay reward model** (stamina-gated vs independent-capped vs hybrid) — explicitly deferred; only the server-resolved invariant + run-resolution seam are locked now.
- **iOS host** (Swift + HealthKit behind `HostBridge`) — deferred, kept cheap by design.
- **Flutter / React Native host + community Unity wrapper** — rejected for maintenance-cliff risk.
- **3D, twitch-density real-time combat, multiplayer netcode.**
- _(Carried over)_ Tier-2 verification (WHOOP, contact-camera PPG, gym APIs); AI mercenary/companion; Garmin; B2B; IAP/subscriptions/cosmetics; realtime/websocket guild updates.
