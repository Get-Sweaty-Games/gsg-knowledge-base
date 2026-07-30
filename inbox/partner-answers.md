# Partner answers — `BACKEND-INTEGRATION-PLAN.md` § 5(c)

The single running file for every answer to your C-list and the B-items we own. Sections are
appended in delivery order, not ID order — use the index to find one.

**Every answer here is read off the source, with the file and line cited, never off our contract
docs.** That is not a style choice: so far *every* question you have raised has turned out to be a
place where our docs were wrong rather than merely silent, so the code is the authority and the doc
is the thing that gets corrected.

## Index

| # | Question | Verdict | Landed in |
|---|---|---|---|
| C1 | `POST /devices` undocumented | Endpoint exists; your guessed field name was wrong (`token`, not `fcmToken`) | [unity#18](https://github.com/Get-Sweaty-Games/reign-and-gain-unity/pull/18) |
| C2 | No way to clear auth tokens | Agreed; `clearAuthTokens()` added to the interface | [unity#16](https://github.com/Get-Sweaty-Games/reign-and-gain-unity/pull/16) (`.aar`), [unity#18](https://github.com/Get-Sweaty-Games/reign-and-gain-unity/pull/18) (contract) |
| C3 | Telling a 401 survived silent refresh | Refresh is lazy inside `readAuthToken()`; your compare-and-retry rule is correct | [unity#18](https://github.com/Get-Sweaty-Games/reign-and-gain-unity/pull/18) |
| C4 | Does the host park an OAuth `code` on cold start? | **It did not — a real bug.** Now always parked; `OnAuthCodeReceived` is **removed** | batch 4 below |
| C5 | Which `source` does a gym session upload as? | **Neither — the client uploads nothing for a gym session.** All four contradicting statements are wrong | batch 3 below |
| C6 | Terminator for `requestHealthRead` | Built: `OnHealthReadComplete` + count, exactly once per call on every path | batch 4 below |
| C7 | Machine-readable `OnHealthReadError` code | Built, your prefixes verbatim. `no_provider:` exposed a launch crash on provider-less devices | batch 4 below |
| C8 | `manualEntryFlag` when HC supplies none | You never author it — the host does. Unsupplied resolves `false`; a `true` hard-rejects | batch 3 below |
| C9 | Optionality of every `X \| null` field | Zod pasted. **`.optional()` ≠ `.nullable()`** — a default Json.NET null 400s the whole bundle | batch 3 below |
| C10 | `GET /state` cannot feed Home | Right, ship it narrower. But `weeklyReward` already exists, and streak/penalty/boss are **not coming** | batch 3 below |
| C11 | Config keys + live values | Your default was right: one `appconfig.json`, five keys. Values sent and verified | [unity#18](https://github.com/Get-Sweaty-Games/reign-and-gain-unity/pull/18), [unity#20](https://github.com/Get-Sweaty-Games/reign-and-gain-unity/pull/20) |
| C12 | `UnityBridge.oauthRedirect()` getter | Built as proposed | batch 4 below |
| C13 | What is `requiresManualFlag` for? | Nothing you can act on — the server pre-filters the catalogue. Your default was right | batch 3 below |
| C14 | `stopRunSession` can deliver zero replies | Built: always exactly one reply. New `OnRunSessionError`, **not** an empty `OnRunSessionEnded` | batch 4 below |
| C15 | `permissionStatus() == DENIED` has no remedy | Built: `UnityBridge.openHealthConnectSettings()`; gate the affordance on its return value | batch 4 below |
| C16 | Rejected/ineligible have no user-facing meaning | Checker→copy mapping given. `ineligible_workout` **cannot fire today** — the rule catalogue is empty | batch 3 below |
| C17 | No rate or refresh policy | Real numbers given. `/state` has no per-user limit — event-driven only, never poll | batch 3 below |
| C18 | Manifest fragment missing the HC permission entries | Confirmed, and it was **three** defects, not one | [unity#19](https://github.com/Get-Sweaty-Games/reign-and-gain-unity/pull/19), [ours#7](https://github.com/yarins0/reign-and-gain/pull/7) |
| B1 | Orientation | **Deferred to a joint call — deliberately.** Host imposes nothing, so it costs us zero either way. `screen-flow.mermaid` was already delivered in unity#18 and does *not* answer it | B1 below |

---

# Batch 1 — C1, C2, C3, C11

The four you flagged as critical-path for Phases 1–3. All four are now closed on our side — C2
needed a host change and it has already shipped into the `.aar`; the other three were documentation
defects and the contract docs have been corrected. The corrected `unity-client-brief.md`,
`unity-bridge-contract.md` and `unity-rest-contract.md` come with this reply; take them as
replacements for the copies you have.

---

## C1 — `POST /devices`

**The endpoint already exists and is live.** It was never unspecified, only undocumented — the REST
contract's claim to cover "every HTTP call the Unity client makes" was false. Our fault; the section
is being added.

**Do not use your proposed default.** You guessed `{ "fcmToken": …, "platform": … }`; the field is
`token`, not `fcmToken`. A `fcmToken` body 400s. Your own note — *"a wrong field name leaves push
broken while looking implemented"* — is exactly what would have happened, so the instinct to ask
rather than ship the guess was right.

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
| `429` | — | Rate limit: **10 registrations per user per minute**. |

Notes that matter for your implementation:

- **Re-registration is idempotent** — call it every launch after login if that's simplest; the route
  is a register-or-update, not an insert.
- **It works with no FCM config on our side.** The route is mounted unconditionally; only the later
  *send* goes dark. So a `200` here does **not** prove push works end to end, and you should not
  treat it as a health check for the feature.
- Your "any 2xx = success, any error = silent degradation" policy is right and we'd keep it, with
  one exception: a `429` is worth a retry on next launch rather than a permanent give-up, since it
  means you called it too often, not that the token was bad.

Source: `backend/src/routes/devices.ts`.

---

## C2 — clearing stored auth tokens

**You are right that the interface has no way to do it, and you are right that your proposed
workaround is dangerous. Do not ship `storeAuthTokens("", "")`.**

The failure you predicted is real and we confirmed it in the store: `read()` is
`prefs.getString(KEY_AUTH_TOKEN, null)`, which returns the **stored empty string**, not `null`,
once an empty string has been written. So after that call `readAuthToken()` returns `""`, every
request sends a bare `Bearer `, the server 401s forever, and the client cannot distinguish "logged
out" from "token rejected". That is precisely the trap you described.

**The capability exists in the host but is not exposed.** `SecureTokenStore.clear()` is already
implemented (`SecureTokenStore.kt:90`); it simply was never lifted onto the `HostBridge` interface,
because logout was not in any earlier flow. Adding it is a few lines plus a contract edit, and it is
the right fix rather than a sentinel.

**Done — `clearAuthTokens()` is on the interface as of this reply**, so it will be in the first
`.aar` you receive and you never integrate logout twice. Contract:

```
fun clearAuthTokens()
```
> Erases both the access and refresh token from secure storage. After this call `readAuthToken()`
> returns `null` (never `""`). Safe to call when already signed out — it is a no-op, never an error.

So build `ClearAuthTokens()` into `IHostBridge` and `FakeHostBridge` normally — no stub, no
`TODO(contract)`, no rework. The real method is there.

Source: `android-host/hostbridge/src/main/kotlin/com/getsweatygames/reignandgain/HostBridge.kt`,
`SecureTokenStore.kt:76-91`.

---

## C3 — how to tell a 401 survived the host's silent refresh

**The host refreshes lazily, inside `readAuthToken()`. There is no background timer.** Your proposed
default therefore works exactly as you designed it, and you can build on it.

The actual implementation (`AndroidHostBridge.kt:248-256`):

1. Read the stored access token. `null` → return `null` (not signed in).
2. If it is **not** expired → return it unchanged.
3. If expired → read the refresh token, exchange it against Supabase **synchronously**, persist the
   rotated pair, return the **new** access token.
4. If the refresh fails (offline, revoked, network error) → return the **stale** token unchanged and
   let the server's 401 drive you back to login.

So your rule — *on a 401, call `readAuthToken()` again; if the string differs, retry once; if
identical, prompt re-login* — is correct, and each branch maps onto a real case:

- **String differs** → step 3 fired; the 401 was ordinary expiry that has now been healed. Retry.
- **String identical** → either step 2 (token wasn't expired, so the 401 is a genuine rejection —
  revoked session, deleted user) or step 4 (refresh was attempted and failed). Both mean re-login.
  You cannot distinguish them, and you do not need to: the remedy is the same.

Three things about it that are not in the contract and will bite you:

1. **`readAuthToken()` blocks on a network call** on the expiry path. It is documented as *must not
   be called from the Android main thread*. Unity's scripting thread is safe. Do not marshal it onto
   the main thread, and prefer fetching the token **before** you build the request rather than
   inside a render-critical path. It only actually blocks about once per token lifetime (~hourly).
2. **The refresh token rotates.** Step 3 persists a new refresh token as well as a new access token.
   Nothing on your side needs to care, but it means the host is stateful across the call and you
   should not call `readAuthToken()` concurrently from two threads.
3. **Step 4 returning a stale token is deliberate**, not a bug — it makes the server the single
   authority on whether a session is dead, rather than having the host guess offline.

One consequence for your 401 policy: because step 4 hands back the identical string when the device
is merely **offline**, your rule will prompt re-login on a network blip. Recommend gating the
re-login prompt on the request having actually reached us — i.e. treat a transport-level failure as
"retry later", and only run the compare-and-re-login path on a real HTTP 401 from the server. Your
brief §6 warning about Credential Manager throttling on repeated dismissals is the reason to be
strict about this.

Source: `AndroidHostBridge.kt:235-256`, `TokenRefresher.kt`.

---

## C11 — which config keys are authoritative

**Your proposed default is right: one `appconfig.json` with all five keys, and "injected at build
time" is satisfied by that file.** Both halves of the contradiction you found are our documentation
errors, not two competing mechanisms.

**The `googleWebClientId` omission is a straight bug in the brief.** § 2's `configure()` call takes
it and the JSON block below it does not list it. `configure()` is the authority —
`UnityBridge.configure(supabaseUrl, supabasePublishableKey, googleWebClientId)`, three arguments,
all required (`UnityBridge.kt:93`). The JSON block is being corrected.

**"Injected at build time (`BACKEND_BASE_URL`)"** in the REST contract is stale wording from before
the topology flip, when the base URL came from our Gradle build. Under Unity-as-base we no longer
own your build, so there is no build-time injection available to us — the file is the mechanism.
That line is being removed.

Authoritative shape, five keys:

```json
{
  "backendBaseUrl":         "https://<our-backend-host>",
  "supabaseUrl":            "https://<project>.supabase.co",
  "supabasePublishableKey": "sb_publishable_...",
  "googleWebClientId":      "<...>.apps.googleusercontent.com",
  "oauthRedirect":          "com.getsweatygames.reignandgain://auth-callback"
}
```

Which value goes where:

| Key | Consumer |
|---|---|
| `backendBaseUrl` | Your `RestClient` only. The host never does app networking. |
| `supabaseUrl`, `supabasePublishableKey`, `googleWebClientId` | Passed to `configure()` at boot. The host needs all three: the first two for silent refresh, the third for the native sign-in sheet. |
| `oauthRedirect` | Your browser-PKCE flow. **Not** passed to `configure()` — it is compiled into the plugin, because cold-start deep-link handling has to run before any C# is alive. |

`oauthRedirect` is in the file so you can assert it matches; you correctly note in C12 that nothing
currently exposes the plugin's value to assert *against*. Adding a `UnityBridge.oauthRedirect()`
static is a good suggestion and we are taking it — that answer comes with the C5–C18 batch.

Live values are being sent separately, out of band. They do not go in this file and they do not go
in your repo.

Source: `UnityBridge.kt:84-111`, `docs/unity-client-brief.md:58-88`,
`docs/unity-rest-contract.md:18`.

---

## What this changes on our side

| Item | Change | Status |
|---|---|---|
| C1 | `POST /devices` section added to `unity-rest-contract.md` | ✅ done |
| C2 | `clearAuthTokens()` added to `HostBridge` + `AndroidHostBridge` (delegates to the existing `SecureTokenStore.clear()`, which `remove()`s both keys rather than blanking them); added to the brief and the bridge contract's method surface | ✅ **done — in the `.aar`, verified present on both the interface and the impl** |
| C3 | Lazy-refresh sequence, the 401 compare-and-retry rule, the offline caveat and the main-thread prohibition documented in `unity-bridge-contract.md` + summarised in the brief | ✅ done |
| C11 | `googleWebClientId` added to the brief's JSON block with a which-value-goes-where table; the `BACKEND_BASE_URL` build-time line removed from the REST contract | ✅ done |

Gates re-run after the host change: `:hostlogic:test` 63 tests / 0 failures / 0 errors,
`:hostbridge:assembleRelease` clean, `:app:assembleDebug` green.

C5–C18 follow separately. Of those, **B1 (orientation)** is the one we think you have under-ranked:
it is a product decision with the largest downstream cost in your plan, it gates your Phase 3, and
we should settle it before you write the first shell screen.

---

# Batch 2 — C18

Delivered as [unity#19](https://github.com/Get-Sweaty-Games/reign-and-gain-unity/pull/19); the brief
fix on our side is [ours#7](https://github.com/yarins0/reign-and-gain/pull/7).

## C18 — the manifest fragment's missing Health Connect entries

**You are right, and the gap is worse than you reported: there are three defects, not one.**

We checked § 7's fragment against `android-host/app/src/main/AndroidManifest.xml` — the only manifest
that has ever run this host on a device, so it is the ground truth, not the brief.

1. **No `activity-alias`.** On Android 14+ the Health Connect permission flow is routed through
   `VIEW_PERMISSION_USAGE`, and without the alias the permission screen never engages:
   `getGrantedPermissions()` stays empty and `requestHealthRead` becomes a **permanent silent
   no-op** — the app's entire data path failing with no error, no exception and nothing in logcat.
   Your targetSdk is pinned to 36, so this was unconditional, not a "on newer devices" risk.
2. **No `ACTION_SHOW_PERMISSIONS_RATIONALE` intent filter either** — which is the Android 13-and-below
   half of the same mechanism. So **the fragment worked on no Android version at all.** You had not
   spotted this one; it would have looked identical to defect 1 in testing.
3. **`pathPrefix` where the real manifest uses an exact `path`** on the invite App Link. As written
   it would also claim siblings like `/reignandgain/invite-terms` and open the app on links meant
   for the web.

**Neither permission entry can ship inside the `.aar`, and this is the part worth internalising:**
both attach to the **launcher Activity**, which the plugin deliberately does not declare — a library
that declares `LAUNCHER` injects a second launcher into your APK. The `.aar`'s `<queries>` block is
a *different* mechanism and does not substitute: it lets the host **see** Health Connect apps, it
does not declare us **to** Health Connect. So these entries are permanently yours to carry in your
manifest, and § 8 now points at the fragment as the authority instead of asserting a requirement the
fragment never satisfied.

Also sent with that PR: `PLAN-closeout.md` and `unity-plugin-topology.md`, the two remaining
cited-but-missing docs. `TODO.md` is deliberately withheld — it is iOS-relevance only.

Source: `android-host/app/src/main/AndroidManifest.xml`, `docs/unity-client-brief.md` §§ 7–8.

---

# Batch 3 — the doc-answer pile

## C5 — which `source` does a gym session upload as?

**Neither. The client uploads no activity bundle for a gym session at all** — so all four statements
you found in conflict are wrong, including the two you read as plausible. You were right not to
build the flow, but the reason is bigger than a mis-specified enum value: **the client-side gym
upload you were about to build does not exist on our side.**

### What the gym flow actually is

| Step | Call | Who |
|---|---|---|
| 1 | `POST /gyms` — register a venue, once, with a declared discipline | you |
| 2 | `POST /gyms/ping` — a location ping on app-open. A ping matching no registered gym stores **nothing** | you |
| 3 | Group pings into sessions, synthesize the activity, award it | **us, server-side** |

That is the entire client surface. **There is no session start, no session stop, no duration and no
upload.** The user does not "log" a gym workout; they register a gym once and then simply open the
app while they are there.

Server-side, `GymSessionFinalizer` groups a user's unfinalized pings per gym — a gap over
**90 minutes** starts a new session — and synthesizes the `tracked_gym_session` bundle itself:

- `startedAt` = first ping in the session;
- credited duration = last ping − first ping, floored to **60 s** so a single lone ping is still
  worth something rather than zero;
- `activityType` from the **gym's** declared discipline, not from any claim;
- `gpsContext` from the registered gym's coordinates;
- then straight into the same `ingestActivity` pipeline every other source uses.

Two triggers, both ours: an **active-departure** path (two consecutive away-pings confirm the user
left, closing the session on the spot) and a **staleness sweep** (after any matched ping, and from
the daily tick). Both are idempotent — the session's `activityId` is deterministic, so a double
finalize no-ops on the upsert.

### Your four statements, individually

| Statement | Verdict |
|---|---|
| brief § 5's `source` enum includes `tracked_gym_session` | **Wrong for your purposes.** It is a real member of `WORKOUT_SOURCES` and the picker catalogue — the user *selects* "Gym session" as a logger — but it is never a value you put in a bundle. The enum conflates "sources that exist" with "sources you may upload". |
| REST `POST /activities`: "client-attested sources only (`health_connect`, and `manual` when enabled)" | **Wrong in the other direction** — it omits `tracked_run`, which you genuinely do upload. |
| REST § Gyms: a "manual" claim is corroborated against registered gyms | **Stale.** True before the gym flow was redesigned to passive pings. `manual` is now free-form self-report with *no* presence evidence at all — which is why it is whitelist-gated and `tracked_gym_session` is not. |
| REST § Enums: "for a `tracked_gym_session` the claimed type is a hint only" | **Correct in substance, vacuous in practice** — the type is resolved from the gym's discipline, and you never claim one. |

**What you may actually upload:** `health_connect` and `tracked_run`, plus `manual` if the user
carries the manual-logging flag.

### A defect this exposed on our side — now fixed

The route did not **enforce** the above. `POST /activities` rejected only `PULL_ONLY_SOURCES` (which
contained just `strava`) and gated `manual`; `tracked_gym_session` was neither, so a client-uploaded
one was accepted. That contradicted the safety argument written into its own trust profile —
*"safe because `tracked_gym_session` has no other producer"* — which is what justifies its
presence-only, single-anchor scoring. A direct upload also skips the ping grouping that collapses
repeated app-opens at one gym into a single award.

**Closed.** Both server-ingest-only sources now reject at the route:

```
403  { "error": "server_ingest_only_source", "source": "tracked_gym_session" }
```

**This renames an error code you may already have handled.** `strava` previously returned
`403 pull_only_source`; both now return `server_ingest_only_source`, because the client-facing rule
is identical ("you may not upload this source") even though the reasons differ. Either response is a
**client bug**, not a runtime condition to design for — if you see it, you built an upload path that
should not exist.

Source: `backend/src/routes/gyms.ts:104`, `backend/src/services/moat/gyms/GymSessionFinalizer.ts`,
`backend/src/domain/sources.ts:20-45`,
`backend/src/services/moat/verification/sourceProfiles.ts:84-104`,
`backend/src/routes/activities.ts:50-78`.

---

## C10 — `GET /state` cannot feed the Home screen

**Your conclusion is right, your remedy is right, and you should ship Home narrower and flag it —
exactly as you proposed.** But three of the four things you listed as missing are not missing in the
way you think, and two of them mean *less* work rather than more.

### 1. `/state` already returns a claim field you don't have documented

The snapshot carries `weeklyReward`, which our REST contract does not mention. Actual shape:

```jsonc
"weeklyReward": {           // or null
  "localDate": "2026-07-27",
  "xp": 250,
  "gold": 100,
  "stats": { "str": 1, "dex": 0, "con": 1, "wis": 0 }
}
```

Non-null means there is a claimable weekly-target reward waiting; render the claim affordance and
call `POST /rewards/claim`. Null means nothing to claim — which covers both "target not yet met" and
"already claimed", and you cannot distinguish them from this field alone. **So the claim bar on Home
is buildable today**, and it is server-computed end to end, so it does not violate § 0.

### 2. The weekly target — and the trap we nearly shipped you

**`/state` now carries `weeklyProgress`.** Added while answering this, and it deliberately does *not*
have the shape we first intended:

```jsonc
"weeklyProgress": {         // or null, only if the profile row is missing
  "credits": 2,             // verified credits so far this week
  "personalTarget": 3,      // the player's OWN goal — profiles.weekly_target
  "rewardThreshold": 3      // the SERVER's bar — what the reward actually pays out on
}
```

We were about to send you a single `weeklyTarget` field. That would have been a bug, and it is worth
explaining because the distinction has to survive into your UI:

- **`personalTarget` is `profiles.weekly_target`, and the player can write it themselves** (RLS grants
  authenticated update on exactly `display_name`, `timezone`, `weekly_target`). It is a preference.
- **`rewardThreshold` is the server-owned bar the weekly reward is gated on**, and it is deliberately
  *not* keyed off `weekly_target` — gating a payout on a value the player controls would hand them
  their own reward dial.

They are usually equal and they are not the same number. **Render the claim bar against
`rewardThreshold`.** If you render it against `personalTarget`, a player who sets their target to 1
does one workout, sees a full bar, and gets nothing — the exact silent-mismatch class of bug the
single-snapshot design is supposed to prevent. `personalTarget` is for "your goal: 3/week" copy and
nothing else.

**How `credits` counts, because it is not a workout count:** one credit per distinct
(activity type, local day). Two runs on the same day is **one** credit; a run and a lift on the same
day is two. Days are the *user's* local days, and the week starts on the same configured day the
reward is evaluated against. Do not derive this number yourself from an activity list — it will
disagree with the reward.

Two more things:

- There is **no `GET /profile`** — only `PATCH /profile`, which validates the timezone and caps the
  target at 21. To *change* the target, PATCH it; you do not need to read the profile row directly.
- **`weekly_target = 0` is the intentional rest-week opt-out**, not an unset value. Render it as
  "no target this week", never as "0 / 0" or a nagging empty bar. The mid-week nudge already
  suppresses itself on 0; your UI should match.

### 3. Streak and the miss-penalty do not exist — and are not coming

Both are dead references in the brief, and this is the part that removes work:

- **There is no streak, anywhere in the domain.** Not an undocumented field, not a deferred one — the
  miss rule is a **weekly target, not a daily streak**, by an explicit product decision. Nothing will
  ever expose a streak count. Don't build it, don't leave a placeholder for it.
- **Penalty application is frozen.** The daily tick still runs and still detects weekly shortfalls,
  but the penalty mechanic is dormant on purpose and surfaces in **no** client screen. The brief's
  "miss-penalty economy" phrasing on the guild-members note is describing a frozen mechanic; it
  should not appear in your UI at all.

### 4. Guild weekly-credits: the boss surface is frozen

`guild.bossName`, `guild.bossHpRemaining` and `guild.bossHpMax` are **hardcoded `null` in the route
today** and the world-boss feature is frozen and may not ship. The live guild surface is
create / join / invite / members plus referral, and nothing else. So the "weekly-credits" line in
§ 9 is likewise not buildable and not coming — build the guild tab against the guild endpoints, and
treat those three snapshot fields as permanently null.

This also answers half of C9: **a present `guild` object never has a null `guildId`.** The route
constructs it from a membership row, so `guildId` is a real string whenever the object exists. The
`| null` in the type is an artefact of it being declared alongside the three boss fields that *are*
always null. Read it as: `guild === null` → no guild; `guild !== null` → `guildId` is a string.

Source: `backend/src/routes/state.ts:81-90`, `backend/src/domain/types.ts:153-171`,
`backend/src/services/game/reward/WeeklyRewardService.ts:59-64,169-181`,
`backend/src/routes/profile.ts:22-32`, `supabase/migrations/0001…:31`, `0018…:13-14`.

---

## C9 — the optionality of every field written `X | null`

You were right to insist on this, and it is more urgent than you framed it: **on current behaviour,
a Json.NET serializer left on its defaults will `400` your entire bundle.** Details below the schema.

### The write schema, verbatim

`EvidenceBundleSchema`, copied from `backend/src/domain/types.ts:18-100` with the commentary
stripped:

```ts
z.object({
  activityId:      z.string().uuid(),
  source:          z.enum(WORKOUT_SOURCES),
  sourceWorkoutId: z.string().nullish(),
  originPackage:   z.string().nullish(),
  activityType:    z.enum(ACTIVITY_TYPES),
  startedAt:       z.string().datetime(),
  endedAt:         z.string().datetime(),
  manualEntryFlag: z.boolean(),
  providerFlagged: z.boolean().optional(),
  accelPresence:   z.boolean().optional(),
  gpsContext: z.object({
      lat:            z.number(),
      lon:            z.number(),
      accuracyMeters: z.number().nonnegative(),
    }).nullable().default(null),
  metrics: z.object({
      activeEnergyKcal: z.number().nonnegative().max(20000).optional(),
      durationSeconds:  z.number().nonnegative().max(100000).optional(),
      avgHeartRate:     z.number().nonnegative().max(250).optional(),
      distanceMeters:   z.number().nonnegative().max(1000000).optional(),
      stepCount:        z.number().nonnegative().max(300000).optional(),
    }).default({}),
  runTrack: z.object({
      sampleCount:           z.number().int().nonnegative().max(100000),
      trackedDistanceMeters: z.number().nonnegative().max(1000000),
      movingSeconds:         z.number().nonnegative().max(100000).optional(),
    }).optional(),
  sleep: z.object({
      startedAt:    z.string().datetime(),
      endedAt:      z.string().datetime(),
      totalMinutes: z.number().nonnegative().max(1440),
    }).optional(),
})
```

### What each variant accepts

| Variant | Omit the key? | Send `null`? | Fields |
|---|---|---|---|
| *(bare)* | ❌ 400 | ❌ 400 | `activityId`, `source`, `activityType`, `startedAt`, `endedAt`, `manualEntryFlag` |
| `.nullish()` | ✅ | ✅ | `sourceWorkoutId`, `originPackage` |
| `.optional()` | ✅ | **❌ 400** | `providerFlagged`, `accelPresence`, `runTrack`, `sleep`, **and every `metrics.*` member** |
| `.nullable().default(null)` | ✅ | ✅ | `gpsContext` |
| `.default({})` | ✅ | **❌ 400** | `metrics` (the object itself) |

The one that will actually bite you: **`.optional()` is not `.nullable()`.** `"accelPresence": null`,
`"runTrack": null`, `"sleep": null`, `"metrics": null` and `"metrics": { "distanceMeters": null }`
each fail validation, and the response is a `400 invalid_evidence_bundle` for the **whole bundle** —
not a dropped field. A partially-populated `metrics` (the normal case: Health Connect gives you
duration but no heart rate) must therefore **omit** the absent members, not null them.

### Why this is a live risk rather than a theoretical one

`sourceWorkoutId` and `originPackage` are `.nullish()` *specifically because* your serializer emits
an unset field as `null` — that is written into our own schema comments as the justification. **The
same reasoning was never applied to the other seven optional fields**, so the protection is
half-applied. That is our inconsistency, not your bug.

**Closed on our side — you are no longer exposed to this.** `POST /activities` now strips
null-valued keys recursively before validating, so a null and an absent key are equivalent for every
optional field in the bundle. We did it at the route rather than by widening ten fields to
`.nullish()`: the schema stays honest about which fields are *genuinely* nullable (`gpsContext`)
instead of making everything nullable to appease one serializer, and it is one guard at the trust
boundary rather than ten annotations that the eleventh field will forget.

Two things this deliberately does **not** do:

- **A null inside a required field still fails**, and should:
  `"gpsContext": { "lat": 1, "lon": 2, "accuracyMeters": null }` is a malformed object, not an unset
  one.
- **It does not apply to `POST /sources`, `POST /gyms` or any other route** — only the evidence
  bundle. Those schemas have no optional numerics to trip over today, but do not assume the same
  tolerance everywhere.

**We would still suggest setting `NullValueHandling.Ignore` on your serializer.** It costs one line,
it makes your wire payloads smaller and easier to read in a log, and it stops this class of bug at
the source rather than relying on one server-side guard to keep absorbing it.

Also: **never send `providerFlagged` at all.** It is a provider anti-cheat verdict that only
server-pulled sources set, and `ProviderFlaggedChecker` hard-rejects a bundle carrying `true`. The
schema tolerates it only because pull sources share this type; a client sending it can only hurt
itself.

### On read

Response types are plain TypeScript types with **no runtime validation** — nothing re-validates on
the way out. For `GET /state` specifically the route builds the snapshot as a complete object
literal, so **every documented key is always present**; `| null` there means the *value* may be
null, never that the key is missing. `guild` is covered in C10 above: a present `guild` object
always has a string `guildId`.

Source: `backend/src/domain/types.ts:18-102,153-171`, `backend/src/routes/state.ts:81-90`.

---

## C8 — what is `manualEntryFlag` when Health Connect does not supply one?

**You are right that you must not choose it, and you never have to: for `health_connect` the client
does not author this field at all.** The host computes it and hands it to you already set, in the
`RawHealthSignals` you receive over the bridge. Copy it through verbatim.

The host's rule (`HealthConnectReader.kt:270`):

```kotlin
records.any { it.metadata.recordingMethod == Metadata.RECORDING_METHOD_MANUAL_ENTRY }
```

evaluated over the session record **plus** its calorie, heart-rate and distance records. Two
consequences worth stating explicitly:

- **It is an equality test, not a truthiness test.** When Health Connect supplies no recording
  method the value is `RECORDING_METHOD_UNKNOWN`, which is not `MANUAL_ENTRY`, so the flag resolves
  to **`false`**. "Unknown" is treated as "not hand-typed". That is the answer to your question as
  asked.
- **It is `any`, not `all`.** One hand-entered heart-rate sample attached to an otherwise
  device-recorded session flips the whole workout to `true`.

**And it is the most consequential single field in the bundle.** On a `health_connect` bundle,
`manualEntryFlag: true` causes `ManualEntryChecker` to **hard-reject** the workout outright —
`reject: true`, no weighted score, no appeal from any other signal. So a bug that defaults it to
`true` silently destroys every upload; a bug that defaults it to `false` silently defeats the single
highest-value cheat filter we have. Do not synthesize it, do not default it, do not "fix" it.

For the sources you *do* author yourself, the value is not a judgement call either:

| Source | `manualEntryFlag` | Why |
|---|---|---|
| `health_connect` | **from the host** | Never your choice. |
| `manual` | `true` | Definitionally hand-typed. |
| `tracked_run` | `false` | The app recorded the route live. |
| `tracked_gym_session` | *n/a* | Server-synthesized — see C5. |

Note that neither the `manual` nor the `tracked_run` trust profile includes `manual-entry` in its
checker set at all, so on those two the flag is recorded but never scored. It is a live gate only on
`health_connect` and `strava`. Our own reference assembler sets exactly these values
(`EvidenceBundleAssembler.kt:110,114`).

Source: `android-host/hostbridge/.../HealthConnectReader.kt:155,270`,
`backend/src/services/moat/verification/checkers/ManualEntryChecker.ts`,
`backend/src/services/moat/verification/sourceProfiles.ts:47-115`.

---

## C13 — what is `requiresManualFlag` for?

**Your proposed default is exactly right: derive nothing from it.** It is a server-side filter
predicate that leaks into the response, and it carries no information you can act on.

`GET /sources` filters the catalogue before you see it:

```ts
SOURCE_CATALOG.filter((entry) => !entry.requiresManualFlag || flags.manual_logging_enabled)
```

So **every entry in the catalogue you receive is already selectable by that user.** A non-flagged
user simply does not get a `manual` entry; a flagged user gets one with `requiresManualFlag: true`.
There is no state in which the field means "render this, but disabled" — never grey an entry out on
it, and never map it to a bundle field.

`manual` is the only entry that carries `true`, and the gate behind it is enforced twice more
regardless of what the client does: `POST /sources` returns `403 manual_logging_not_enabled` if you
select it unflagged, and so does `POST /activities` on upload. The client cannot route around it, so
it does not need to police it.

The field you actually want for the picker UI is **`mode`** — `auto` (background sync), `manual`
(the user types the numbers) or `tracked` (the app records live). That is the axis worth grouping
or labelling the picker by.

Source: `backend/src/services/moat/source/SourceService.ts:65-66`,
`backend/src/domain/sources.ts:50-76`, `backend/src/routes/sources.ts:33-40`.

---

## C16 — rejected and ineligible have no user-facing meaning

Taking your second option: **a documented checker → copy mapping**, rather than a message field on
the response. Copy belongs on your side in your voice — we would only be shipping you English
through a JSON field, and badly, since we cannot see the screen it lands on.

Two corrections to the premise first, one of which saves you work:

**`ineligible_workout` cannot currently fire.** The eligibility rule catalogue is deliberately
**empty** — the pipeline stage is wired but every workout passes through it today. Implement the
branch (it is real, and rules land later) but do not invest UI in it, and do not expect to be able
to test it. It is not a state a user can reach right now.

**You already receive a stable machine-readable vocabulary; the prose beside it is the part you
should ignore.** A rejection returns `trust: { total, rejected, signals: [...] }`, and each signal
carries `checker` (a stable id), `score`, `reject` and a `reason`. **The `reason` strings are
internal engineering prose** — "Sample flagged as manually entered by the OS." — and you are right
not to put them on screen. The `checker` ids are the contract.

### How to pick the copy

1. Find the first signal with `reject: true` — that is a hard gate, and it alone decided the outcome.
2. If **no** signal has `reject: true`, the bundle simply scored below its source's threshold. Use
   the generic line; there is no single cause to name.

Only four checkers can hard-reject. The rest contribute scores and can never be the named cause:

| `checker` | What actually happened | Suggested meaning to convey |
|---|---|---|
| `manual-entry` | The OS flagged the sample as hand-typed in the health app | It wasn't recorded by a device, so it can't be counted. **Actionable by the user** — this is the one worth an explanatory line. |
| `metric-rate` | A reward-feeding metric exceeded its window-scaled physical ceiling | The numbers aren't physically possible for that duration. |
| `geofence-negative` | Location contradicts the claim | Where the phone was doesn't match the workout. |
| `provider-flagged` | Strava's own anti-cheat flagged the activity | The source itself marked it suspicious. |
| *(no `reject: true`)* | Weighted score fell below the source's `acceptThreshold` | Generic: not enough evidence to verify this one. |

The soft checkers you may see in `signals` but must never name as the cause: `accel-presence`,
`distance-length`, `heart-rate`, `gym-presence`, `track-consistency`.

**Forward-compatibility rule, and please build this in from the start:** checker ids are stable, but
the set **grows** — adding a verification source adds a checker. Treat an unrecognised `checker` id
as the generic case rather than failing or showing the raw id. That way a new checker on our side
never puts an engineering string in front of a player.

Source: `backend/src/services/moat/eligibility/EligibilityFilter.ts:10-25`,
`backend/src/services/moat/verification/checkers/*.ts`, `backend/src/routes/activities.ts:91-106`.

---

## C17 — rate and refresh policy

Here are the real numbers. There are two layers, and they are enforced independently.

**Layer 1 — global, per IP: 120 requests / minute** across every route.

**Layer 2 — per user** (keyed on the JWT `sub`, so it survives IP rotation). Only the routes below
carry one; anything not listed has *no* per-user limit and is bounded only by the global 120/min:

| Route | Per user / minute |
|---|---|
| `POST /activities` | **60** |
| `POST /activities/register` | 10 |
| `POST /activities/run-started` | 10 |
| `POST /gyms/ping` | **30** |
| `POST /gyms` | 10 |
| `POST /devices` | 10 |
| `POST /rewards/claim` | 10 |
| `POST /strava/sync` | 10 |
| `GET /guilds/members` | 30 |
| guild create / join / leave / kick / transfer | 10 each |
| **`GET /state`, `GET /sources`, `GET /gyms`** | **none** — global limit only |

Ingest is deliberately the loosest: a Health Connect sync legitimately uploads one bundle per workout
in the lookback window, and 60 is sized for that burst.

### The policy we'd ask you to hold to

- **`GET /state`** — on launch, on resume, and after any mutation you make (claim, register, guild
  action). It has no per-user limit, but it is the most expensive call we serve; **do not poll it on
  a timer.** Event-driven only.
- **`POST /strava/sync`** — launch and explicit user pull-to-refresh only. Never on resume: it hits
  a third-party API we don't control, and its own limits are stricter than ours.
- **`POST /gyms/ping`** — once per app-open, as the brief describes. 30/min is headroom for
  foreground churn, not an invitation; pinging more often does not create a better session, because
  duration is last-ping − first-ping regardless of how many pings sit between them (see C5).
- **`POST /activities`** — as bundles become available. Retries reuse the same `activityId`, which is
  the idempotency key, so a retry is always safe.
- **The invite drain on `OnApplicationFocus`** the brief prescribes is fine as specified.

**Handling a `429`:** back off and retry on the next natural trigger — never a tight retry loop.
Nothing above is a hard failure state; every one of these calls is safe to simply do later.

Source: `backend/src/app.ts:43-45`, `backend/src/routes/*.ts` rate-limit constants,
`backend/src/services/account/auth/rateLimitKey.ts`.

---

# Batch 4 — the host-code pile: C4, C6, C7, C12, C14, C15

Six questions that needed the `.aar` changed rather than a document corrected. All six are **built**
and ship together as host version **`29/07/2026-b`** — one delivery, one tag. Assert it at boot with
`UnityBridge.hostVersion()`; if you see `29/07/2026` you are holding the previous build and none of
what follows exists.

**Two of these change the wire, so read C4 and C14 before writing against them.** C4 *removes*
`OnAuthCodeReceived`. The receiver goes from 12 methods to 13: minus `OnAuthCodeReceived`, plus
`OnHealthReadComplete` and `OnRunSessionError`.

---

## C4 — does the host park an OAuth `code` on a cold start?

**It did not, and you found a real bug. Fixed: it is now parked, always.** Your proposed default —
"assume push-only and accept that a process death during login means tapping sign-in again" — would
have been the right call against the old build, but the failure was worse than you assumed, so
don't build for it.

`UnityHostActivity.handleDeepLink` had the two deep links doing opposite things three lines apart:

```kotlin
is DeepLink.Auth   -> link.code?.let { sendToUnity("OnAuthCodeReceived", it) }  // immediate push
is DeepLink.Invite -> UnityBridge.setPendingInviteCode(link.code)               // parked
```

and `handleDeepLink` is called from `onCreate`. The invite branch is parked with a comment saying,
in as many words, that Unity may not be listening yet this early in the lifecycle. The auth branch
had the exact bug the invite branch was already fixed for: on a cold start the code was pushed into
a void and dropped with **nothing in the log**, and login failed with no diagnosable cause — on
precisely the low-end devices that have no native sign-in to fall back to.

**The fix, and the shape to code against:**

```csharp
using (var cls = new AndroidJavaClass("com.getsweatygames.reignandgain.UnityBridge"))
    string authCode = cls.CallStatic<string>("consumePendingAuthCode");  // String? — clears on read
```

Drain it at the same point you already drain invites (`OnApplicationFocus`). One slot, one code,
cleared on read.

**`OnAuthCodeReceived` is gone — delete the handler, don't leave it as a second path.** The host
parks on the *warm* path too, not just cold start, and that is deliberate: the code is single-use,
so two delivery mechanisms race and whichever exchange loses fails the login the winner just
completed. One path, always.

**One thing this moves onto your side:** the `code_verifier` must now survive process death. It
always had to for the cold-start case to work at all, but with the code reliably parked, a
persisted-verifier gap becomes the *only* remaining way for the browser fallback to fail. Persist
it (`PlayerPrefs` is fine — it is worthless without the single-use code) and clear it after the
exchange.

Source: `android-host/hostbridge/.../UnityHostActivity.kt` `handleDeepLink`,
`android-host/hostbridge/.../UnityBridge.kt` `setPendingAuthCode` / `consumePendingAuthCode`.

---

## C6 — can `requestHealthRead` gain a terminator?

**Yes. Built: `OnHealthReadComplete`, payload = the number of signals that read delivered.**

Fires **exactly once per `requestHealthRead`, always last**, on every path — signals delivered,
empty window, permission dialog, hard failure. It is implemented in a `finally` around the whole
read rather than at each exit, specifically so a future branch cannot forget it and quietly hand
back the timer heuristic.

Delete the timer. Act on each `OnHealthSignalsRead` as it arrives, exactly as you do now, and use
the terminator only to stop waiting. The count is a cross-check (did I receive as many as it
claims), not the trigger.

**One sequencing subtlety, because it will otherwise look like a bug.** The terminator ends the
*call*, not the *permission journey*. The permission dialog is asynchronous and the read is not, so
a read that hits the dialog terminates immediately with `count: 0` — before the user has answered.
What happens next arrives unsolicited:

- **granted** → the host re-reads automatically; a fresh fan-out arrives with its own terminator;
- **denied** → a lone `OnHealthReadError` with `permission_denied:` and no terminator after it.

So: treat an `OnHealthReadError` with no outstanding request as a state update, not a reply. The
alternative — holding the terminator open across the dialog — would mean a process death while the
dialog is up leaves you waiting forever, which is the exact failure this question exists to remove.

Source: `android-host/hostbridge/.../AndroidHostBridge.kt` `requestHealthRead`.

---

## C7 — can `OnHealthReadError` carry a machine-readable code?

**Yes, and your suggested prefixes are what shipped.** Every reason on `OnHealthReadError` (and the
new `OnRunSessionError`) is now `code: human text` — stable prefix including the colon, a space,
then diagnostic prose. **Branch on the prefix only**; the tail is not a display string and gets
reworded without notice.

| Code | Fires when | What to do |
|---|---|---|
| `permission_denied:` | reads not granted, or the user refused the dialog | permission affordance + `openHealthConnectSettings()` (see C15) |
| `no_provider:` | no Health Connect app on the device | "Install Health Connect". **Not** a permission affordance — there is nothing to grant |
| `empty_window:` | the 7-day window held no workout | friendly "nothing to sync"; not a failure |
| `read_failed:` | anything else (IPC failure, malformed record) | generic retry; the tail is the underlying exception, for a bug report |
| `no_active_session:` | `stopRunSession` with nothing running (C14) | caller bug — reconcile your toggle against `isRunActive()` |

Answering this turned up a second defect worth telling you about, because it explains a crash you
would otherwise have spent a day on. `no_provider:` **could never have been reported**, because
`HealthConnectReader` built its client in a constructor default argument:

```kotlin
private val client: HealthConnectClient = HealthConnectClient.getOrCreate(context)
```

`getOrCreate` throws when no provider is installed, and the reader is constructed in `onCreate` —
so on a device with no Health Connect the app died at launch, before any read could classify
anything. It is now behind `by lazy`, and the same device reaches a read and gets `no_provider:`
back. Fixed in this build.

Source: `android-host/hostlogic/.../HealthReadDispatch.kt` (the `HEALTH_ERR_*` constants),
`android-host/hostbridge/.../AndroidHostBridge.kt` `permissionDeniedReason` / `failureReason`,
`android-host/hostbridge/.../HealthConnectReader.kt` `isProviderAvailable`.

---

## C12 — `oauthRedirect` cannot actually be asserted

**Correct, and built as you proposed.**

```csharp
using (var cls = new AndroidJavaClass("com.getsweatygames.reignandgain.UnityBridge"))
    string redirect = cls.CallStatic<string>("oauthRedirect");
```

Compare it against `appconfig.oauthRedirect` at boot and fail loudly on a mismatch. Your diagnosis
was exactly right: the value is compiled into the `.aar` (cold-start deep-link handling needs it
before any C# is alive) and plugin manifests have no `manifestPlaceholders`, so "assert they match"
was a comment rather than a check. Without this the mismatch surfaces at the single worst moment —
a real user falling back to the browser login.

This pairs with C4: `oauthRedirect()` proves the *scheme* matches, but the redirect also has to be
in **Supabase's allowed-redirect list**, which neither side can read at runtime. That one is ours
to confirm in the dashboard.

Source: `android-host/hostbridge/.../UnityBridge.kt` `oauthRedirect`.

---

## C14 — `stopRunSession` with no active session delivers nothing

**Agreed, and built: `stopRunSession` now always answers exactly once.** `OnRunSessionEnded` on
success, or the new `OnRunSessionError` with `no_active_session:` when there was nothing running.
Your framing was the right one — "always exactly one reply" is a far easier contract than
"sometimes zero" — and keep your timeout anyway.

**One deliberate deviation from your proposal.** You offered "an empty or error result"; it is
strictly the error, never an empty `OnRunSessionEnded`. A run that genuinely collected zero GPS
samples is *real evidence* the server still has to see and judge (indoor treadmill, denied location
permission — raw absence, not a verdict). If a stop-with-no-session arrived on the same message,
the two would be indistinguishable to you, and you would upload a fabricated `tracked_run` for a
run that never happened. Separate messages keep that impossible.

Source: `android-host/hostbridge/.../AndroidHostBridge.kt` `stopRunSession`,
`android-host/hostbridge/.../HostBridge.kt` `onRunSessionError`.

---

## C15 — `permissionStatus() == DENIED` has no remedy path

**Right, and it was as bad as you describe. Built:**

```csharp
using (var cls = new AndroidJavaClass("com.getsweatygames.reignandgain.UnityBridge"))
    bool opened = cls.CallStatic<bool>("openHealthConnectSettings");
```

Opens Health Connect's own permission screen. Render the affordance on
`permissionStatus() == DENIED`, but **gate it on the return value**: `false` means there is no
remedy to offer (no registered host Activity, or no Health Connect provider on the device at all —
which also shows up as `no_provider:` on any read), and a button that does nothing is worse than no
button.

It is a static on `UnityBridge` rather than a `HostBridge` method for the same reason
`startGoogleSignIn` is, and you predicted it: launching an Activity needs an Activity, which
`HostBridge` implementations deliberately never hold, and `HostBridge` is the frozen iOS port
surface. (iOS equivalent, when we get there: `UIApplication.openSettingsURLString`.)

Source: `android-host/hostbridge/.../UnityBridge.kt` `openHealthConnectSettings`,
`android-host/hostbridge/.../UnityHostActivity.kt` `openHealthConnectSettings`.

---

# B1 — orientation

**Not decided, and we are not going to decide it unilaterally.** You were right to escalate this and
right that it must not be settled implicitly by whoever writes the first shell screen. Three things
below: a correction, what we can rule out, and the call itself coming back to you.

## The documents you thought were missing are already in your repo

**Three of the four have been delivered**, and your "never uploaded" list was written before the
merges that carried them:

| Doc | Where it is |
|---|---|
| `docs/screen-flow.mermaid` | your `4761361` (**#18**) — confirmed byte-identical to ours, modulo line endings |
| `docs/unity-plugin-topology.md` | your `d05d887` (**#19**) — this is the one that settles the `activity-alias` / `targetSdk` question your §8 raised |
| `docs/PLAN-closeout.md` | your `d05d887` (**#19**) |
| `docs/TODO.md` | genuinely not delivered. Informational, and iOS is not in scope — say the word if you want it anyway |

Worth re-checking your working copy before hunting for any of these: ours was three commits behind
yours when we went looking, which is how this reply nearly went out claiming two of them were still
outstanding.

**But `screen-flow.mermaid` does not answer B1, so do not go looking.** It is a screen inventory and
a navigation graph — nothing in it states an orientation, a canvas reference, or a layout. That was
an omission rather than a decision, and we have now said so *in the file itself* rather than leaving
the next reader to discover it the way you did. Two corrections ship with this reply: the header no
longer calls it "Godot client screen flow" (that route is archived; the IA survived both pivots
unchanged), and it now carries an explicit note that orientation is deliberately unstated and points
here — including a warning not to read its vertical drawing direction as portrait intent.

Take it as authoritative for what you asked it for: screen inventory, back-navigation, where
Settings hangs (off Home, not a tab), and what is in the tab bar.

## What we can rule out: the host imposes nothing

Whatever you choose costs **zero** on our side. No `.aar` change, no version bump, no manifest
change from us. Specifically:

- the manifest fragment in brief §7 declares **no `android:screenOrientation`** — the host takes no
  position and never has;
- its `configChanges` already lists `orientation|screenLayout|screenSize|smallestScreenSize|density`,
  so a runtime `Screen.orientation` switch is absorbed **without an Activity restart**. Your own
  reading of this was correct.

So option 2's rotation is free at the Android layer, and none of the three options is cheaper or
more expensive *for us*. This is entirely your C# and our product call — which is exactly why it
comes back to you rather than being answered here.

## The call, back to you

Your table is the right framing and we accept the costs as you scoped them. Two things to weigh in
with, and then we want your studio's view before we pick:

**We think the two halves are genuinely different postures**, and your table does not say this.
Logging a workout and claiming a reward is a one-handed, standing-in-a-gym interaction — portrait.
A dungeon run is lean-back — landscape. On that reasoning **option 2 is where we lean too**, and we
note it is also your recommendation. Option 1's saving is real but it is a saving on the half that
is already built, paid for by the half that users touch every day.

**What would change our mind:** if authoring the shell portrait-first costs materially more than the
"two canvas configurations" line in your table suggests — you have `RgSkin` and the existing
composition, we do not — then say so with a number. Our reasoning is about product posture, and it
is not worth weeks.

**What we need back**, before your Phase 3 as you asked:

1. Your pick of the three, or a fourth if we have both missed one.
2. If option 2: where the switch fires. We would put it at the Game tab boundary, but the splash and
   sign-in are the awkward cases (they precede any tab) and you own that transition.
3. Whether you want the decision written into `screen-flow.mermaid` as an annotation per screen, or
   stated once in the brief. We will write it wherever you will actually read it.

Nothing in the backend or the host blocks on this, so it does not gate anything we owe you — but it
gates your Phase 3, so it is the one open item where the delay is entirely ours to avoid.
