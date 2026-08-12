# Partner answers — `BACKEND-INTEGRATION-PLAN.md` § 5(c)

> Filed from inbox `partner-answers.md`. The single running file for every answer to the partner
> studio's C-list and the B-items we own. Sections are appended in delivery order, not ID order — use
> the index to find one. Cited by `unity-backend-integration-decisions.md`'s verdict table.
>
> **B1 (orientation) is filed separately** in
> `departments/game-design/docs/unity-integration-owner-decisions.md` — it's a product/UX decision,
> not a backend/API one. The index row below is kept for completeness.

**Every answer here is read off the source, with the file and line cited, never off our contract
docs.** That is not a style choice: so far *every* question the partner has raised has turned out to
be a place where our docs were wrong rather than merely silent, so the code is the authority and the
doc is the thing that gets corrected.

## Index

| # | Question | Verdict | Landed in |
|---|---|---|---|
| C1 | `POST /devices` undocumented | Endpoint exists; guessed field name was wrong (`token`, not `fcmToken`) | unity#18 |
| C2 | No way to clear auth tokens | Agreed; `clearAuthTokens()` added to the interface | unity#16 (`.aar`), unity#18 (contract) |
| C3 | Telling a 401 survived silent refresh | Refresh is lazy inside `readAuthToken()`; compare-and-retry rule is correct | unity#18 |
| C4 | Does the host park an OAuth `code` on cold start? | **It did not — a real bug.** Now always parked; `OnAuthCodeReceived` is **removed** | batch 4 below |
| C5 | Which `source` does a gym session upload as? | **Neither — the client uploads nothing for a gym session.** All four contradicting statements are wrong | batch 3 below |
| C6 | Terminator for `requestHealthRead` | Built: `OnHealthReadComplete` + count, exactly once per call on every path | batch 4 below |
| C7 | Machine-readable `OnHealthReadError` code | Built, prefixes verbatim. `no_provider:` exposed a launch crash on provider-less devices | batch 4 below |
| C8 | `manualEntryFlag` when HC supplies none | Client never authors it — the host does. Unsupplied resolves `false`; a `true` hard-rejects | batch 3 below |
| C9 | Optionality of every `X \| null` field | Zod pasted. **`.optional()` ≠ `.nullable()`** — a default Json.NET null 400s the whole bundle | batch 3 below |
| C10 | `GET /state` cannot feed Home | Right, ship it narrower. But `weeklyReward` already exists, and streak/penalty/boss are **not coming** | batch 3 below |
| C11 | Config keys + live values | Default was right: one `appconfig.json`, five keys. Values sent and verified | unity#18, unity#20 |
| C12 | `UnityBridge.oauthRedirect()` getter | Built as proposed | batch 4 below |
| C13 | What is `requiresManualFlag` for? | Nothing actionable — the server pre-filters the catalogue | batch 3 below |
| C14 | `stopRunSession` can deliver zero replies | Built: always exactly one reply. New `OnRunSessionError`, **not** an empty `OnRunSessionEnded` | batch 4 below |
| C15 | `permissionStatus() == DENIED` has no remedy | Built: `UnityBridge.openHealthConnectSettings()`; gate the affordance on its return value | batch 4 below |
| C16 | Rejected/ineligible have no user-facing meaning | Checker→copy mapping given. `ineligible_workout` **cannot fire today** — the rule catalogue is empty | batch 3 below |
| C17 | No rate or refresh policy | Real numbers given. `/state` has no per-user limit — event-driven only, never poll | batch 3 below |
| C18 | Manifest fragment missing the HC permission entries | Confirmed, and it was **three** defects, not one | unity#19, ours#7 |
| B1 | Orientation | **Deferred to a joint call — deliberately.** Host imposes nothing, so it costs zero either way | filed in `departments/game-design/docs/unity-integration-owner-decisions.md` |

---

# Batch 1 — C1, C2, C3, C11

The four flagged as critical-path for Phases 1–3. All four are now closed on our side — C2
needed a host change and it has already shipped into the `.aar`; the other three were documentation
defects and the contract docs have been corrected.

## C1 — `POST /devices`

**The endpoint already exists and is live.** It was never unspecified, only undocumented. Guessed
field name was wrong: the field is `token`, not `fcmToken` — a `fcmToken` body 400s.

```
POST /devices
Authorization: Bearer <access token>     (required)
Content-Type: application/json

{ "platform": "android",   // enum, exactly "android" | "ios"
  "token": "<FCM registration token>" }  // 1..4096 chars
```

| Response | Body | Meaning |
|---|---|---|
| `200` | `{ "ok": true }` | Registered or re-registered. |
| `400` | `{ "error": "invalid_device" }` | Body failed validation — wrong platform value, empty or >4096-char token. |
| `401` | — | No/invalid bearer. |
| `429` | — | Rate limit: 10 registrations per user per minute. |

Notes: re-registration is idempotent (call every launch after login); the route works with no FCM
config on our side (a `200` does not prove push works end to end — do not treat as a health check);
a `429` is worth a retry on next launch rather than a permanent give-up.

Source: `backend/src/routes/devices.ts`.

## C2 — clearing stored auth tokens

Do not ship `storeAuthTokens("", "")` — `read()` is `prefs.getString(KEY_AUTH_TOKEN, null)`, which
returns the stored empty string once one has been written, so `readAuthToken()` returns `""`,
every request sends a bare `Bearer `, and the server 401s forever with no way to tell "logged out"
from "token rejected".

`SecureTokenStore.clear()` already existed (`SecureTokenStore.kt:90`) but was never exposed on
`HostBridge`. **Done — `clearAuthTokens()` is on the interface**, ships in the `.aar`:

```
fun clearAuthTokens()
```
> Erases both the access and refresh token from secure storage. After this call `readAuthToken()`
> returns `null` (never `""`). Safe to call when already signed out — a no-op, never an error.

Source: `android-host/hostbridge/src/main/kotlin/com/getsweatygames/reignandgain/HostBridge.kt`,
`SecureTokenStore.kt:76-91`.

## C3 — how to tell a 401 survived the host's silent refresh

The host refreshes lazily, inside `readAuthToken()`. No background timer. Actual implementation
(`AndroidHostBridge.kt:248-256`):

1. Read the stored access token. `null` → return `null` (not signed in).
2. Not expired → return unchanged.
3. Expired → exchange refresh token against Supabase **synchronously**, persist the rotated pair,
   return the new access token.
4. Refresh fails (offline, revoked, network error) → return the **stale** token unchanged; the
   server's 401 drives the client back to login.

The compare-and-retry rule (on a 401, re-read; if the string differs, retry once; if identical,
prompt re-login) is correct. Three gotchas: `readAuthToken()` blocks on a network call on the expiry
path (must not run on the Android main thread; Unity's scripting thread is safe); the refresh token
rotates, so don't call concurrently from two threads; step 4's stale-token return is deliberate — it
makes the server the single authority on session death. Consequence: because step 4 returns the
identical string when merely offline, gate the re-login prompt on the request having actually
reached the server — treat a transport failure as "retry later," only run compare-and-re-login on a
real HTTP 401.

Source: `AndroidHostBridge.kt:235-256`, `TokenRefresher.kt`.

## C11 — which config keys are authoritative

One `appconfig.json` with all five keys; "injected at build time" is satisfied by that file. The
`googleWebClientId` omission was a straight bug in the brief — `configure()` takes it
(`UnityBridge.kt:93`, three required args) and the JSON block didn't list it, now corrected. The
"injected at build time (`BACKEND_BASE_URL`)" REST-contract line is stale, pre-dating the topology
flip — removed.

```json
{
  "backendBaseUrl":         "https://<our-backend-host>",
  "supabaseUrl":            "https://<project>.supabase.co",
  "supabasePublishableKey": "sb_publishable_...",
  "googleWebClientId":      "<...>.apps.googleusercontent.com",
  "oauthRedirect":          "com.getsweatygames.reignandgain://auth-callback"
}
```

| Key | Consumer |
|---|---|
| `backendBaseUrl` | `RestClient` only. The host never does app networking. |
| `supabaseUrl`, `supabasePublishableKey`, `googleWebClientId` | Passed to `configure()` at boot. |
| `oauthRedirect` | Browser-PKCE flow. **Not** passed to `configure()` — compiled into the plugin, since cold-start deep-link handling runs before any C# is alive. |

Source: `UnityBridge.kt:84-111`, `docs/unity-client-brief.md:58-88`, `docs/unity-rest-contract.md:18`.

## What this changes on our side

| Item | Change | Status |
|---|---|---|
| C1 | `POST /devices` section added to `unity-rest-contract.md` | done |
| C2 | `clearAuthTokens()` added to `HostBridge` + `AndroidHostBridge`, added to brief + bridge contract | done — in the `.aar` |
| C3 | Lazy-refresh sequence, 401 compare-and-retry rule, offline caveat, main-thread prohibition documented | done |
| C11 | `googleWebClientId` added to brief's JSON block with which-value-goes-where table; stale build-time line removed | done |

Gates re-run after the host change: `:hostlogic:test` 63 tests / 0 failures / 0 errors,
`:hostbridge:assembleRelease` clean, `:app:assembleDebug` green.

---

# Batch 2 — C18

## C18 — the manifest fragment's missing Health Connect entries

Checked § 7's fragment against `android-host/app/src/main/AndroidManifest.xml` (the only manifest
that has ever run this host on a device — ground truth over the brief). Three defects, not one:

1. **No `activity-alias`.** On Android 14+ the Health Connect permission flow routes through
   `VIEW_PERMISSION_USAGE`; without the alias the permission screen never engages —
   `getGrantedPermissions()` stays empty and `requestHealthRead` becomes a permanent silent no-op.
   targetSdk is pinned to 36, so this was unconditional.
2. **No `ACTION_SHOW_PERMISSIONS_RATIONALE` intent filter** — the Android 13-and-below half of the
   same mechanism. So the fragment worked on **no** Android version at all.
3. **`pathPrefix` where the real manifest uses an exact `path`** on the invite App Link — would also
   claim web-only sibling paths.

Neither permission entry can ship inside the `.aar` — both attach to the **launcher Activity**,
which the plugin deliberately does not declare (a library declaring `LAUNCHER` injects a second
launcher into the APK). The `.aar`'s `<queries>` block is a different mechanism (lets the host *see*
Health Connect apps, doesn't declare us *to* Health Connect). These entries are permanently the
partner's to carry in their manifest.

Also delivered with this batch: `PLAN-closeout.md` and the topology decision doc (filed separately
as `departments/engineering/docs/unity-plugin-topology-decision.md`). `TODO.md` withheld —
iOS-relevance only.

Source: `android-host/app/src/main/AndroidManifest.xml`, `docs/unity-client-brief.md` §§ 7–8.

---

# Batch 3 — the doc-answer pile

## C5 — which `source` does a gym session upload as?

**Neither. The client uploads no activity bundle for a gym session at all.** All four brief
statements the partner found in conflict are wrong.

| Step | Call | Who |
|---|---|---|
| 1 | `POST /gyms` — register a venue, once, with a declared discipline | client |
| 2 | `POST /gyms/ping` — a location ping on app-open. A ping matching no registered gym stores nothing | client |
| 3 | Group pings into sessions, synthesize the activity, award it | server |

No session start, no session stop, no duration, no upload. Server-side, `GymSessionFinalizer` groups
unfinalized pings per gym — a gap over 90 minutes starts a new session — and synthesizes the
`tracked_gym_session` bundle: `startedAt` = first ping; credited duration = last − first ping,
floored to 60s; `activityType` from the gym's declared discipline; `gpsContext` from the gym's
coords; then the normal `ingestActivity` pipeline. Two triggers, both server-side: active-departure
(two consecutive away-pings) and a staleness sweep. Both idempotent (deterministic `activityId`).

**What may actually be uploaded:** `health_connect` and `tracked_run`, plus `manual` if the
manual-logging flag is set.

**Defect exposed and fixed:** `POST /activities` only rejected `PULL_ONLY_SOURCES` (just `strava`)
and gated `manual`; `tracked_gym_session` was neither, so a client-uploaded one would have been
silently accepted. Closed — both server-ingest-only sources now `403`:

```
403  { "error": "server_ingest_only_source", "source": "tracked_gym_session" }
```

This renames the error `strava` used to return, from `pull_only_source`.

Source: `backend/src/routes/gyms.ts:104`, `backend/src/services/moat/gyms/GymSessionFinalizer.ts`,
`backend/src/domain/sources.ts:20-45`,
`backend/src/services/moat/verification/sourceProfiles.ts:84-104`,
`backend/src/routes/activities.ts:50-78`.

## C10 — `GET /state` cannot feed the Home screen

Ship Home narrower, as proposed. Three corrections:

**1. `/state` already returns a claim field.** `weeklyReward` (non-null = claimable):
```jsonc
"weeklyReward": { "localDate": "2026-07-27", "xp": 250, "gold": 100, "stats": { "str": 1, "dex": 0, "con": 1, "wis": 0 } }
```
Null covers both "not yet met" and "already claimed" — indistinguishable from this field alone.

**2. `weeklyProgress` was added while answering this** — two distinct numbers, do not conflate:
```jsonc
"weeklyProgress": { "credits": 2, "personalTarget": 3, "rewardThreshold": 3 }
```
`personalTarget` = `profiles.weekly_target`, player-writable, a preference. `rewardThreshold` =
server-owned bar the reward actually pays on, deliberately not keyed off `weekly_target` (else the
player sets their own reward dial). **Render the claim bar against `rewardThreshold`.** `credits`
counts one credit per distinct `(activity type, local day)` — do not derive it from an activity
list. No `GET /profile` exists — only `PATCH /profile` (validates timezone, caps target at 21).
`weekly_target = 0` is the intentional rest-week opt-out — render as "no target this week."

**3. No streak, ever; penalty frozen.** The miss rule is a weekly target, not a daily streak — no
streak field will ever exist. Penalty mechanic is dormant, surfaces in no client screen.

**4. Guild boss surface frozen.** `guild.bossName`/`bossHpRemaining`/`bossHpMax` hardcoded `null`;
world-boss feature may not ship. Live guild surface: create/join/invite/members plus referral only.
A present `guild` object never has a null `guildId`.

Source: `backend/src/routes/state.ts:81-90`, `backend/src/domain/types.ts:153-171`,
`backend/src/services/game/reward/WeeklyRewardService.ts:59-64,169-181`,
`backend/src/routes/profile.ts:22-32`, `supabase/migrations/0001…:31`, `0018…:13-14`.

## C9 — the optionality of every field written `X | null`

**On current behaviour, a Json.NET serializer left on its defaults will 400 the entire bundle.**

`EvidenceBundleSchema` variants (`backend/src/domain/types.ts:18-100`):

| Variant | Omit the key? | Send `null`? | Fields |
|---|---|---|---|
| *(bare)* | 400 | 400 | `activityId`, `source`, `activityType`, `startedAt`, `endedAt`, `manualEntryFlag` |
| `.nullish()` | ok | ok | `sourceWorkoutId`, `originPackage` |
| `.optional()` | ok | **400** | `providerFlagged`, `accelPresence`, `runTrack`, `sleep`, every `metrics.*` member |
| `.nullable().default(null)` | ok | ok | `gpsContext` |
| `.default({})` | ok | **400** | `metrics` (the object itself) |

`.optional()` is not `.nullable()` — sending `null` on any of those fields 400s the *whole* bundle,
not just drops the field. A partially-populated `metrics` must **omit** absent members, never null
them. **Closed on our side:** `POST /activities` now strips null-valued keys recursively before
validating, so null and absent are equivalent for every optional field. Still recommend
`NullValueHandling.Ignore` client-side. A null inside a **required** field still fails (that's
correct). Never send `providerFlagged` at all — `ProviderFlaggedChecker` hard-rejects `true`, and it
only exists on the schema because pull sources share the type.

On read: response types have no runtime validation; `GET /state` always includes every documented
key, so `| null` means the value may be null, never that the key is missing.

Source: `backend/src/domain/types.ts:18-102,153-171`, `backend/src/routes/state.ts:81-90`.

## C8 — what is `manualEntryFlag` when Health Connect does not supply one?

For `health_connect` the client never authors this field — the host computes it and hands it over
verbatim via `RawHealthSignals`. Host rule (`HealthConnectReader.kt:270`):
```kotlin
records.any { it.metadata.recordingMethod == Metadata.RECORDING_METHOD_MANUAL_ENTRY }
```
Equality test, not truthiness: `RECORDING_METHOD_UNKNOWN` is not `MANUAL_ENTRY`, so unsupplied
resolves `false`. It's `any`, not `all` — one hand-entered sample flips the whole workout to `true`.

**Most consequential field in the bundle:** `manualEntryFlag: true` on `health_connect` causes
`ManualEntryChecker` to hard-reject outright, no appeal. A bug defaulting it `true` silently destroys
every upload; defaulting `false` silently defeats the best cheat filter. Never synthesize or default
it.

| Source | `manualEntryFlag` | Why |
|---|---|---|
| `health_connect` | from the host | Never the client's choice |
| `manual` | `true` | Definitionally hand-typed |
| `tracked_run` | `false` | App recorded the route live |
| `tracked_gym_session` | n/a | Server-synthesized (see C5) |

Source: `android-host/hostbridge/.../HealthConnectReader.kt:155,270`,
`backend/src/services/moat/verification/checkers/ManualEntryChecker.ts`,
`backend/src/services/moat/verification/sourceProfiles.ts:47-115`.

## C13 — what is `requiresManualFlag` for?

Derive nothing from it. `GET /sources` already filters the catalogue
(`SOURCE_CATALOG.filter((entry) => !entry.requiresManualFlag || flags.manual_logging_enabled)`), so
every entry received is already selectable — never grey an entry out on this field, never map it to
a bundle field. `manual` is the only entry that carries `true`; the gate is enforced twice more
server-side regardless of client behavior. The field worth building the picker UI around is `mode`
(`auto` / `manual` / `tracked`).

Source: `backend/src/services/moat/source/SourceService.ts:65-66`,
`backend/src/domain/sources.ts:50-76`, `backend/src/routes/sources.ts:33-40`.

## C16 — rejected and ineligible have no user-facing meaning

Documented checker → copy mapping, not a message field (copy belongs client-side, in their voice).
`ineligible_workout` cannot currently fire — the eligibility rule catalogue is deliberately empty.
A rejection returns `trust: { total, rejected, signals: [...] }`; each signal carries `checker` (a
stable id), `score`, `reject`, `reason` — the `reason` strings are internal prose, ignore them; the
`checker` ids are the contract. Pick copy by finding the first `reject: true` signal; if none, the
bundle scored below threshold (generic line).

Only four checkers can hard-reject:

| `checker` | What happened | Suggested copy |
|---|---|---|
| `manual-entry` | OS flagged the sample as hand-typed | Wasn't recorded by a device — actionable |
| `metric-rate` | A metric exceeded its window-scaled physical ceiling | Numbers aren't physically possible for that duration |
| `geofence-negative` | Location contradicts the claim | Where the phone was doesn't match the workout |
| `provider-flagged` | Strava's own anti-cheat flagged it | The source marked it suspicious |
| *(no reject)* | Weighted score below threshold | Generic: not enough evidence |

Soft checkers never to name as cause: `accel-presence`, `distance-length`, `heart-rate`,
`gym-presence`, `track-consistency`. **Forward-compat:** treat any unrecognised `checker` id as the
generic case — the set grows as verification sources are added.

Source: `backend/src/services/moat/eligibility/EligibilityFilter.ts:10-25`,
`backend/src/services/moat/verification/checkers/*.ts`, `backend/src/routes/activities.ts:91-106`.

## C17 — rate and refresh policy

**Layer 1 — global, per IP: 120 req/min** across every route. **Layer 2 — per user** (JWT `sub`):

| Route | Per user / minute |
|---|---|
| `POST /activities` | 60 |
| `POST /activities/register` | 10 |
| `POST /activities/run-started` | 10 |
| `POST /gyms/ping` | 30 |
| `POST /gyms` | 10 |
| `POST /devices` | 10 |
| `POST /rewards/claim` | 10 |
| `POST /strava/sync` | 10 |
| `GET /guilds/members` | 30 |
| guild create/join/leave/kick/transfer | 10 each |
| `GET /state`, `GET /sources`, `GET /gyms` | none — global limit only |

Policy to hold to: `GET /state` on launch/resume/after-mutation, never polled on a timer
(event-driven only, no per-user cap but the most expensive call served); `POST /strava/sync` launch
+ explicit pull-to-refresh only, never on resume; `POST /gyms/ping` once per app-open; `POST
/activities` retries reuse the same idempotent `activityId`. Handle `429` with backoff on next
natural trigger, never a tight retry loop.

Source: `backend/src/app.ts:43-45`, `backend/src/routes/*.ts` rate-limit constants,
`backend/src/services/account/auth/rateLimitKey.ts`.

---

# Batch 4 — the host-code pile: C4, C6, C7, C12, C14, C15

All six **built**, ship together as host version `29/07/2026-b`. Assert with
`UnityBridge.hostVersion()`. C4 removes `OnAuthCodeReceived`; C6 adds `OnHealthReadComplete`; C14
adds `OnRunSessionError`. Receiver goes from 12 methods to 13.

## C4 — does the host park an OAuth `code` on a cold start?

**It did not — a real bug. Fixed: now always parked.** `UnityHostActivity.handleDeepLink` had the
auth deep link pushing immediately (`sendToUnity("OnAuthCodeReceived", it)`) while the invite deep
link parked (`setPendingInviteCode`), called from `onCreate`. A cold-start auth code was dropped with
nothing in the log.

```csharp
using (var cls = new AndroidJavaClass("com.getsweatygames.reignandgain.UnityBridge"))
    string authCode = cls.CallStatic<string>("consumePendingAuthCode");  // clears on read
```

Drain at the same point invites are drained (`OnApplicationFocus`). `OnAuthCodeReceived` is
**removed**, not deprecated — the host parks on the warm path too, deliberately, so two delivery
mechanisms can't race a single-use code. `code_verifier` must now survive process death (persist via
`PlayerPrefs`, clear after exchange).

Source: `android-host/hostbridge/.../UnityHostActivity.kt` `handleDeepLink`,
`android-host/hostbridge/.../UnityBridge.kt` `setPendingAuthCode` / `consumePendingAuthCode`.

## C6 — can `requestHealthRead` gain a terminator?

Built: `OnHealthReadComplete`, payload = count of signals delivered. Fires exactly once per call,
always last, on every path (implemented in a `finally`). Delete any timer heuristic; use the
terminator only to stop waiting. Sequencing subtlety: the terminator ends the *call*, not the
*permission journey* — a read hitting the permission dialog terminates immediately with `count: 0`
before the user answers; what follows arrives unsolicited (granted → host re-reads automatically
with its own terminator; denied → a lone `OnHealthReadError` with `permission_denied:` and no
terminator after it). Treat an `OnHealthReadError` with no outstanding request as a state update, not
a reply.

Source: `android-host/hostbridge/.../AndroidHostBridge.kt` `requestHealthRead`.

## C7 — can `OnHealthReadError` carry a machine-readable code?

Built, suggested prefixes shipped verbatim (`code: human text`, branch on the prefix only):

| Code | Fires when | What to do |
|---|---|---|
| `permission_denied:` | reads not granted / user refused dialog | permission affordance + `openHealthConnectSettings()` |
| `no_provider:` | no Health Connect app installed | "Install Health Connect" — not a permission affordance |
| `empty_window:` | 7-day window held no workout | friendly "nothing to sync," not a failure |
| `read_failed:` | IPC failure, malformed record | generic retry |
| `no_active_session:` | `stopRunSession` with nothing running | caller bug — reconcile toggle against `isRunActive()` |

Second defect surfaced: `no_provider:` could never have fired — `HealthConnectReader` built its
client in a constructor default argument (`HealthConnectClient.getOrCreate(context)`), which throws
when no provider is installed, so the app crashed at launch before any read could classify anything.
Fixed with `by lazy`.

Source: `android-host/hostlogic/.../HealthReadDispatch.kt`,
`android-host/hostbridge/.../AndroidHostBridge.kt` `permissionDeniedReason` / `failureReason`,
`android-host/hostbridge/.../HealthConnectReader.kt` `isProviderAvailable`.

## C12 — `oauthRedirect` cannot actually be asserted

Built as proposed:
```csharp
using (var cls = new AndroidJavaClass("com.getsweatygames.reignandgain.UnityBridge"))
    string redirect = cls.CallStatic<string>("oauthRedirect");
```
Compare against `appconfig.oauthRedirect` at boot, fail loudly on mismatch. The value is compiled
into the `.aar` (cold-start deep-link handling needs it before any C# is alive); plugin manifests
have no `manifestPlaceholders`. Does not prove the redirect is in Supabase's allowed-redirect list —
neither side can read that at runtime; confirmed manually in the dashboard.

Source: `android-host/hostbridge/.../UnityBridge.kt` `oauthRedirect`.

## C14 — `stopRunSession` with no active session delivers nothing

Built: `stopRunSession` now always answers exactly once — `OnRunSessionEnded` on success, or the new
`OnRunSessionError` with `no_active_session:` when nothing was running. Deliberate deviation from the
"empty or error" proposal: it is strictly the error, never an empty `OnRunSessionEnded` — a run that
genuinely collected zero GPS samples (indoor treadmill, denied permission) is real evidence the
server must still see and judge, and the two cases must stay distinguishable so a stop-with-no-
session never uploads a fabricated `tracked_run`.

Source: `android-host/hostbridge/.../AndroidHostBridge.kt` `stopRunSession`,
`android-host/hostbridge/.../HostBridge.kt` `onRunSessionError`.

## C15 — `permissionStatus() == DENIED` has no remedy path

Built:
```csharp
using (var cls = new AndroidJavaClass("com.getsweatygames.reignandgain.UnityBridge"))
    bool opened = cls.CallStatic<bool>("openHealthConnectSettings");
```
Opens Health Connect's own permission screen. Render the affordance on
`permissionStatus() == DENIED`, gated on the return value: `false` means no remedy to offer (no
registered host Activity, or no Health Connect provider at all — also surfaces as `no_provider:` on
any read). A static on `UnityBridge` rather than `HostBridge`, same reason as `startGoogleSignIn` —
launching an Activity needs an Activity, which `HostBridge` implementations never hold (it's the
frozen iOS port surface). iOS equivalent: `UIApplication.openSettingsURLString`.

Source: `android-host/hostbridge/.../UnityBridge.kt` `openHealthConnectSettings`,
`android-host/hostbridge/.../UnityHostActivity.kt` `openHealthConnectSettings`.
