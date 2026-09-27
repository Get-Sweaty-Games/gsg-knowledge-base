# Backend Integration Plan — Reign & Gain as the Game tab

> **This document is a PLAN. It records intent and reasoning, not what exists.**
>
> **It carries no status, deliberately.** It used to: every phase table had a `BUILT` / `MISSING`
> column and the header carried five dated closure notes. Those were removed on 2026-08-08, because
> a status column reads as current however loudly a header disclaims it — the last audit's one
> remaining "NOT DONE" row had itself been done for days without anyone noticing.
>
> **For what is built and what is left, read
> [issue #247](https://github.com/Get-Sweaty-Games/reign-and-gain-unity/issues/247).** For what the
> tree does today, read the tree. **When a phase completes, update the issue — never this file.**
>
> **For the decisions taken while building it** — the partner's C1–C18 answers, the owner's B1–B6
> calls, and the reasoning behind the wire — read the frozen decision record in the sibling
> knowledge-base repo, at the plain-text path
> `Interpretable-Context-Methodology/inbox/integration-decisions-unity-backend.md`.

**Where a row disagrees with the tree, the tree wins.** That is the standing rule for every plan in
`docs/`, and it is why the rows below are left as written rather than retro-edited: a plan that is
quietly rewritten to match what shipped stops being a record of what was intended, and the gap between
the two is often the interesting part.

Two conventions the rows rely on:

- **§1.9 and §4 cite by symbol, not by line.** `RunController.cs`, `GameRoot.cs`, `RunResultView.cs`,
  `StubCombatController.cs`, `CombatController.cs` and `RunState.cs` were each observed to move
  mid-session, more than once. A symbol survives a refactor; a line number is a snapshot of the moment
  it was checked. Re-locate by symbol wherever a cited number stops matching.
- **Two owner decisions that the rows below assume are settled:** §3.3's JSON question resolved to
  Newtonsoft, as §3 recommends; and the appconfig split (§A14) resolved to keeping the file committed
  as-is, with no sample and no gitignore entry — the reasoning is under §1.2.

Written 2026-07-28 against `unity-client-brief.md`, `unity-bridge-contract.md` and a third document,
`unity-rest-contract.md`, which was deleted on 2026-08-04 after drifting from the code — its HTTP
detail now lives in the backend's Swagger UI at <https://api.reignandgain.getsweatygames.com/docs>,
except for interactive login, which moved into `unity-bridge-contract.md`. Written against the code
as it stood on branch `chore/atlas-compression-and-file-splits`.

Throughout, **"the contract says X"** means it is written in one of Yarin's three documents.
**"I recommend X"** means it is a judgement call this plan is making. Every claim about existing code
carries a `file:line`.

Settled by the owner and not re-argued here: rebuild from prose (no reference implementation
requested); `hostbridge.aar` gets a placeholder and development must proceed without it; the signing
keystore is a placeholder; Play Console submission and the Data Safety declaration are Yarin's.

---

## 0. The one-paragraph summary

The integration itself is ordinary REST work. Three things are not:

1. **The bridge is unverifiable until the `.aar` lands.** Yarin's §10 says items 1–2 are the
   integration risk; they are also the two we cannot prove. The answer is a seeded fake that
   reproduces the *asynchronous shape* of the wire, not just its return values.
2. **The "server owns all maths" rule collides with this project's determinism invariant far less
   than it looks.** The client computes exactly one currency, it is run-scoped, and nothing about it
   is ever credited to anything. §4 works this out precisely and proposes the boundary.
3. **The brief describes screens the contracts cannot feed and a wire method the REST contract never
   documents.** §5 lists 39 of these. Four are on the critical path of Phases 1–3 and should go to
   Yarin before any code is written.

---

## 1. File-by-file build plan

### 1.1 Folder and assembly placement

The existing layout is `Assets/Scripts/<Area>/`, one area per concern, everything in one runtime
assembly. Four new areas, following that pattern:

| Path | Namespace | What |
|---|---|---|
| `Assets/Scripts/Host/` | `ReignAndGain.Host` | The Unity↔Kotlin bridge and its fake. |
| `Assets/Scripts/Net/` | `ReignAndGain.Net` | Auth, REST transport, DTOs. Contains no gameplay type. |
| `Assets/Scripts/Evidence/` | `ReignAndGain.Evidence` | Evidence-bundle assembly. Pure, no Unity types. |
| `Assets/Scripts/AppUI/`, `Assets/Scripts/AppFlow/` | `ReignAndGain.App` | The app shell. |
| `Assets/Plugins/Android/` | — | Manifest, gradle templates, the `.aar` drop point. |

All runtime code lands in the **existing** `ReignAndGain` assembly
(`Assets/Scripts/ReignAndGain.asmdef`). No new asmdef: a second runtime assembly would need explicit
references in both directions and buys nothing, and `rootNamespace: ReignAndGain` already tolerates
sub-namespaces. Two asmdef facts that matter later:

- `ReignAndGain.asmdef` has `"overrideReferences": false`, so a precompiled DLL delivered by a UPM
  package auto-references with no asmdef edit.
- `Assets/Tests/EditMode/ReignAndGain.EditModeTests.asmdef` has `"overrideReferences": true` with
  `"precompiledReferences": ["nunit.framework.dll"]`. **Any new precompiled dependency must be added
  there by hand** or the tests will not see it. This is the single easiest step to miss.

`AndroidJavaClass` / `AndroidJavaObject` come from `com.unity.modules.androidjni`, already in
`Packages/manifest.json`. `UnityWebRequest` comes from `com.unity.modules.unitywebrequest`, also
already present. No package additions are needed for the wire itself — only for JSON (§3).

### 1.2 Phase 0 — Prerequisites

*Depends on nothing. Must land first.* Not all files; several are settings edits.

| Item | Change | Why / contract |
|---|---|---|
| `Packages/manifest.json` | add `"com.unity.nuget.newtonsoft-json": "3.2.1"` | §3 of this plan; the JSON question is decided in §3.3 |
| `Assets/Tests/EditMode/ReignAndGain.EditModeTests.asmdef` | add `"Newtonsoft.Json.dll"` to `precompiledReferences` | see §1.1 |
| `Assets/link.xml` | preserve `ReignAndGain` DTO types against IL2CPP managed stripping | `ProjectSettings.asset` sets `stripEngineCode: 1` with a platform-default `managedStrippingLevel`. It preserves the whole assembly rather than a type list; the reasoning is in the file's own header |
| `ProjectSettings/ProjectSettings.asset` → `AndroidTargetSdkVersion` | `0` → `36` | brief §8, explicitly. Pinned to 36 by agreement across both repos, so treat it as shared with `@Get-Sweaty-Games/unity-devs` |
| `ProjectSettings/ProjectSettings.asset` → `ForceInternetPermission` | `0` → `1` (Require) | *recommendation.* "Auto" infers `android.permission.INTERNET` from a code scan; with stripping on, inferring is a coin-flip not worth taking |
| ~~`Assets/Resources/appconfig.sample.json`~~ | ~~committed template~~ | **Decided against.** No sample file exists — see the note below |
| `Assets/Resources/appconfig.json` + `.meta` | committed with live values, reviewed via CODEOWNERS | **The brief's split is deliberately reversed.** Brief §2 said keep it gitignored; the reasoning for overriding that is below |

**The `appconfig` split was reversed from the brief, and the owner has now confirmed it stays reversed
(2026-08-05).** `Assets/Resources/appconfig.json` is tracked with live values, `.gitignore` names it
nowhere, and no sample exists. This was already defensible on the merits before the owner's call: the
key is `sb_publishable_*`, which is client-safe by design and ships inside the APK either way, and
`AppConfig.IsComplete` actively rejects an `sb_secret_` key (`AppConfig.cs:101`), so the leak the
gitignore split would have prevented was never reachable. A sample full of blanks would fail
`IsComplete` at boot regardless, so the "fresh clone boots clean" benefit the brief wanted was mostly
notional. CODEOWNERS already treats the path as owned and reviewed, which only works for a committed
file — a gitignored `appconfig.json` would drop out of every diff a reviewer sees. No further action:
this row exists to record that the divergence from the brief was audited and kept, not to flag
outstanding work.

| File | Namespace | Responsibility | Mandated by |
|---|---|---|---|
| `Assets/Scripts/Net/AppConfig.cs` | `ReignAndGain.Net` | POCO + `Load()` from `Resources.Load<TextAsset>("appconfig")`. Exposes `BackendBaseUrl`, `SupabaseUrl`, `SupabasePublishableKey`, `GoogleWebClientId`, `OauthRedirect`. **The `appconfig.sample` fallback this row originally specified was decided against** — see the note above. **Its fields are lower-camel to match the wire**, not the PascalCase properties named here: `JsonUtility` matches field names character for character. | brief §2 |
| `Assets/Scripts/Host/HostWireNames.cs` | `ReignAndGain.Host` | Every frozen wire string as a `const`: the JNI class `com.getsweatygames.reignandgain.UnityBridge`, the GameObject name `HostBridgeReceiver`, all 13 receiver method names, the OAuth redirect, the invite scheme and host. One file to diff against the contract's "Naming summary". | bridge contract §"Naming summary" |

### 1.3 Phase 1 — The wire (Yarin §10.1)

*Depends on Phase 0.* This is integration risk #1.

| File | Namespace | Responsibility | Mandated by |
|---|---|---|---|
| `IHostBridge.cs` | `ReignAndGain.Host` | The `HostBridge` methods plus the `UnityBridge` statics — both frozen lists live in `IHostBridge.cs` / `AndroidHostBridge.cs`'s `UnityBridge` class and are not re-enumerated here, since a second count is exactly what goes stale. Mirrors the Kotlin interface exactly, so it is also the iOS port surface. | bridge §"Methods C# may call". Two later host deliveries widened this: `Configure` took a 4th argument, `backendBaseUrl` (PR #146), and `RequestLocationPermission` / `RequestBackgroundHealthRead` arrived as `UnityBridge` statics |
| `HostPermissionStatus.cs` | `ReignAndGain.Host` | `enum { NotRequested, Denied, Granted }` + tolerant parse of the Kotlin `name()` string (an unrecognised value must not throw). | bridge, `permissionStatus` |
| `AndroidHostBridge.cs` | `ReignAndGain.Host` | The real JNI path. No platform `#if` — `AndroidJavaClass` compiles on every platform and only fails at construction, so the single guard in `HostBridge.Resolve` covers both "not Android" and "plugin missing" with one code path. Caches the `AndroidJavaClass` and the `AndroidJavaObject` from `get()` rather than re-fetching per call. **Null-guards `get()`** — a mis-merged manifest means the host never called `register(...)`, `get()` returns null, and every call would NRE; that must be one logged error, not a crash per frame. | brief §4, bridge §"Unity → host" |
| `FakeHostBridge.cs` | `ReignAndGain.Host` | Editor / no-plugin implementation. Returns canned data and **schedules the receiver callbacks over several frames**, so the async shape is identical to the device. See §2.1. | owner decision 2 |
| `HostBridge.cs` | `ReignAndGain.Host` | Static locator: `HostBridge.Current` → `AndroidHostBridge` when `Application.platform == RuntimePlatform.Android` *and* the JNI class resolves, else `FakeHostBridge`. **The only platform check in the codebase.** | owner decision 2 |
| `HostBridgeReceiver.cs` | `ReignAndGain.Host` | The MonoBehaviour with **exactly** the 13 public `void M(string)` methods. Zero logic — each parses (or doesn't) and raises a C# event. `DontDestroyOnLoad`. Never deactivated. | bridge §"host → Unity", brief §3 |
| `HostSignals.cs` | `ReignAndGain.Host` | The typed event surface the receiver raises, so a screen subscribes without hunting for the GameObject. Keeps the receiver a dumb 13-method shim whose names never need to move. | — (structural) |
| `HostPayloads.cs` | `ReignAndGain.Host` | `RawHealthSignals`, `GpsContext`, `HealthMetrics`, `RunTrack`, `ConnectedSourcesPayload`, `ConnectedSource`, `SleepSummary`, `PresenceSnapshot`, `GoogleIdTokenPayload`. Every optional numeric a `double?`; every optional object a nullable reference. | bridge §JSON shapes. Deserialized with Newtonsoft per §3.3, which meets the `double?` requirement `JsonUtility` could not |
| `HostBootstrap.cs` | `ReignAndGain.Host` | `[RuntimeInitializeOnLoadMethod(BeforeSceneLoad)]`: create the receiver GameObject, `DontDestroyOnLoad` it, then call `Configure(supabaseUrl, publishableKey, googleWebClientId)`. §2 says config first, before anything that needs it. It passes `backendBaseUrl` too, per the widened `Configure` above. | brief §2, §3 |
| `HealthReadSession.cs` | `ReignAndGain.Host` | Owns the 0..N fan-out: starts on `requestHealthRead`, collects each `OnHealthSignalsRead`, and **always completes**. The CLAUDE.md "anything holding a lock must time out" invariant applied to a wire instead of a QTE. | bridge §"One read fans out to many signals"; ambiguity A1, **superseded**. Build it against the current wire contract, not A1's original quiescence-timer default: `OnHealthReadComplete` is the sole normal terminator, and a single 30s hard cap is the only defensive timer, firing only when that terminator never arrives at all. The "Superseded" note under A1 says why the quiescence timer was dropped |

**Phase 1 verification — and it should be wider than this plan specified.**
`Assets/Tests/EditMode/HostWireTests.cs` is the guard: `ReceiverExposesExactlyTheThirteenWireMethods`
plus one test each for void return, instance (non-static), exactly one `string` parameter, and public
visibility — and an `OnAuthCodeReceived_IsAbsent` guard this plan never asked for. A rename here fails
*silently on device* ("could not find method" in logcat and nothing else), which is the same failure
mode that left the push path dead.

**A reflection guard cannot catch a stale list.** This suite diffs the receiver against
`HostWireNames.ReceiverMethods`, and both sides were once generated from the same out-of-date source —
so it passed cleanly while being wrong. The postmortem is §6 of the decision record named at the top.

**The count is 13, not the 12 this plan said.** Corrected throughout on 2026-08-04. One known copy is
still stale: `Assets/link.xml`'s header comment says "twelve methods". It is left for the next PR that
already touches `Assets/`, because that path is outside `tests.yml`'s `paths-ignore` and a one-word
comment fix there would cost a full ~15-minute Unity run on its own.

### 1.4 Phase 2 — Auth (Yarin §10.2)

*Depends on Phase 1.* Integration risk #2. Not end-to-end verifiable without the `.aar` **and** live
Supabase values.

| File | Responsibility | Mandated by |
|---|---|---|
| `Net/Pkce.cs` | `code_verifier` (43–128 chars, unreserved set) and `code_challenge = base64url(SHA-256(verifier))`. Uses `System.Security.Cryptography.RandomNumberGenerator` — **the one deliberate exception to the `SeededRng` invariant**, because a reproducible PKCE verifier is a security hole. Must be called out in `docs/ARCHITECTURE.md`'s Determinism section or the next reader will "fix" it. | REST §Auth Path 2 |
| `Net/SupabaseAuthClient.cs` | The two token exchanges only: `?grant_type=id_token` and `?grant_type=pkce`. Sends `apikey` and `Content-Type`. Returns a struct that goes straight to `storeAuthTokens` and is then zeroed. Never persists, never refreshes. | REST §Auth steps 3 / 4 |
| `Net/AuthSession.cs` | The login state machine: native first; `"cancelled"` **aborts**; any other `OnGoogleSignInError` falls back to browser PKCE. Holds the verifier for the browser flow only. Calls `storeAuthTokens` exactly once, then drops both strings. | brief §6 |
| `Net/AuthTokenProvider.cs` | The **only** caller of `readAuthToken()`. Supplies `Authorization: Bearer`. Owns the 401 policy (ambiguity C3). | REST §Conventions |

### 1.5 Phase 3 — REST + Home (Yarin §10.3)

*Depends on Phase 2.*

| File | Responsibility | Mandated by |
|---|---|---|
| `Net/RestClient.cs` | One coroutine `UnityWebRequest` wrapper. GET/POST JSON, base URL from `AppConfig`, bearer from `AuthTokenProvider`, the single 401 retry, typed result. **No endpoint knowledge.** | REST §Conventions |
| `Net/ApiResult.cs` | `ApiResult<T>` — `Ok`, `HttpStatus`, `Value`, `ErrorCode`, `RawBody`. Makes "the server said no" a value the caller must handle, not an exception someone forgets. | REST error shapes |
| `Net/Dto/StateDtos.cs` | `GameStateSnapshot`, `CharacterDto`, `StatBlockDto`, `PendingDto`, `GuildStatusDto` (nullable), `onboardingComplete`, `generatedAt`. | `GET /state` |
| `Net/Dto/SourcesDtos.cs` | `SourceSelectionState`, `SourceCatalogEntry`, `SourceMode`, `SetSourcesRequest`. | `GET`/`POST /sources` |
| `Net/Dto/ActivityDtos.cs` | `EvidenceBundle`, `MetricsDto`, `RunTrackDto`, `GpsContextDto`; `ActivityUploadResponse` with **nullable** `pending`, `trust`, `reason`, `detail` — the three response variants are discriminated by *field presence*; `TrustDto`, `TrustSignalDto`; `RegisterResult`. | `POST /activities`, `/register` |
| `Net/Dto/GuildDtos.cs`, `GymDtos.cs`, `StravaDtos.cs` | The remaining documented shapes. | REST §Guilds, §Gyms, §Strava |
| `Net/Enums/WorkoutSource.cs` | The five values, wire strings, `IsPullOnly`. `EvidenceBundle`'s constructor **rejects** `strava`, so a `403 pull_only_source` is unreachable rather than merely discouraged. | REST §Enums; brief §5 |
| `Net/Enums/ActivityType.cs` | The eleven values and wire strings. Adding a value is a backend change. | REST §Enums |
| `Net/BackendApi.cs` | One method per **documented** endpoint and nothing more: `GetState`, `GetSources`, `PostSources`, `PostActivity`, `PostRegister`, `GetGyms`, `PostGym`, `GetStravaConnect`, `PostStravaSync`, `PostGuild`, `PostGuildJoin`, `GetGuildInvite`, `GetGuildMembers`. **`/devices` is deliberately absent** — see C1. | REST, whole document |
| `AppFlow/AppFlow.cs` | Shell state machine, same allowed-transition-table shape as `Assets/Scripts/Runtime/GameFlow.cs:33-55`, so it is unit-testable with no scene. `Splash → SignIn → SourcePicker → Shell{Home,Roster,Game,Guild}`, plus action modes `{ManualEntry, TrackedRun, Settings}`. Routes on `onboardingComplete`. | brief §9 (summary only — the authoritative flow doc is missing, C15) |
| `AppFlow/AppRoot.cs` | The shell's `GameRoot` twin: builds the shell canvas, owns the tab bar, hosts the existing `GameRoot` behind the Game tab. | brief §9 |
| `AppUI/HomeView.cs` | Renders a `GameStateSnapshot` and nothing else. Claim bar shown when `pending.count > 0`. | REST `GET /state`; brief §9 |
| `AppUI/ClaimBar.cs` | Claim tap → `PostRegister` → render `applied` and `character` from the response. Never sums anything. | brief §6 two-phase |
| `AppUI/AppIcons.cs` | New procedural glyphs for the shell: four tab icons, a glyph per stat (`str`/`dex`/`con`/`wis`), a pending badge, per-source marks. Built the way `Assets/Scripts/UI/IconFactory.cs:19-265` builds its sprites. **Not cosmetic** — the emoji invariant forbids the obvious shortcut and `RgSkin`/`ProcTex` forbid flat panels. | CLAUDE.md invariants |

### 1.6 Phase 4 — Source picker (Yarin §10.4)

*Depends on Phase 3.*

| File | Responsibility |
|---|---|
| `AppUI/SourcePickerView.cs` | One screen, two entries: onboarding (gated on `onboardingComplete: false`) and Settings. `catalog` drives what renders — render whatever it contains; `selected` pre-checks; submit replaces the whole selection. |
| `AppUI/ConnectedSourcesStrip.cs` | The device-local `OnConnectedSourcesRead` subtitle, cross-referenced against `supportedIntegratorPackages`: recognised apps lead ("Syncing: …"), unrecognised-but-syncing are still shown ("Also detected: …"). Display only; never uploaded. |

### 1.7 Phase 5 — The core loop (Yarin §10.5) — *"get it green before anything cosmetic"*

*Depends on Phase 4.* This is the phase that earns the integration.

| File | Responsibility | Mandated by |
|---|---|---|
| `Evidence/EvidenceBundleAssembler.cs` | `RawHealthSignals` + `WorkoutSource` → `EvidenceBundle`. Mints `activityId`; maps `startedAtIso`→`startedAt`; passes `sourceWorkoutId` and `originPackage` through untouched; **omits** absent metrics rather than zeroing them; **forwards `sleep` unchanged** (see below); **never** sets `providerFlagged`; forces `metrics.distanceMeters` from `runTrack.trackedDistanceMeters` for `tracked_run`. Pure static, no Unity types — unit-testable exactly like `CombatMath`. | brief §5 |

> **Sleep — the "do not populate" hold is LIFTED** as of host `03/08/2026`, and this plan said the
> opposite until 04/08. Sleep is uploaded, verified and rewarded. It arrives as an ordinary signal in
> `requestHealthRead`'s fan-out with `activityType: "sleep"`, and the assembler must forward **both**
> `metrics.sleepMinutes` **and** the `sleep` sub-object exactly as the host hands them over.
>
> They are not redundant, which is why "tidying away" the apparent duplicate breaks uploads: the
> first is the reward-feeding claim, the second is the independent window the server corroborates it
> against, and a bundle claiming more minutes than its own window spans is **hard-rejected**.
>
> `OnSleepRead` remains a **display** read only. Uploading from it as well double-counts the night.
> A declined sleep permission is not an error — sleep signals simply do not arrive.
>
> ⚠️ Gated on us, not on the client work: the **Play Data Safety Health-info declaration** is still
> outstanding and must land before a sleep-bearing build reaches real users. The privacy-policy flip
> it was waiting on has shipped. Build and test freely; do not ship one to production until it is
> confirmed — realistically at the next Play Console upload. Source: brief §5.
| `Evidence/ActivityUploadQueue.cs` | Per-signal upload, serialised. Persists the `sourceWorkoutId → activityId` mapping so a retry genuinely reuses the idempotency key (A11). Aggregates accepted/rejected/duplicate/ineligible into one narrative summary. | brief §5, REST §Conventions |
| `Evidence/IsoTime.cs` | ISO-8601-with-`Z` format and parse, in one place, because a missing `Z` 400s the whole bundle and `DateTime.ToString("o")` on a `DateTimeKind.Unspecified` value silently omits it. | brief §5 ("Zod `.datetime()` rejects a missing `Z`") |
| `AppUI/SyncFlowView.cs` | The read→upload→claim narrative: trigger `requestHealthRead`, show progress across the 0..N fan-out, re-fetch `/state` on completion, hand off to the claim bar. **Must render `OnHealthReadError` code `empty_window:` as success** — see below. | brief §10.5, bridge contract `04/08/2026` |

> **`empty_window:` is a success state, not a failure.** Since the `04/08/2026` host delivery the
> background `HealthSyncWorker` usually drains the window before the user ever taps sync, so
> `empty_window:` is now the **common** result of a manual sync rather than an edge case. It must read
> as "You're up to date". Rendering it on the error path makes a perfectly working build look broken,
> and the contract calls this out as the single most likely way for that to happen.
>
> `HostWireNames.ErrorEmptyWindow` already exists for it. As of 04/08 **nothing consumes it**, because
> no sync UI exists yet — this is the first screen that will, so the requirement is recorded here
> rather than left to be rediscovered.

### 1.8 Phase 6 — Everything else (Yarin §10.6)

*Depends on Phase 5.*

The partner answers resolved every block this table originally carried. What survives are constraints,
not blocks, and they are folded into the rows.

| File | Responsibility |
|---|---|
| `AppUI/ManualEntryView.cs` | Manual workout entry → `source: manual` with a live `requestPresenceSnapshot`. **Never author `manualEntryFlag` for `health_connect`** (C8) — the host computes it, and a wrong default silently destroys every upload or defeats the best cheat filter available. |
| ~~`AppUI/GymSessionView.cs`~~ | ~~The gym-session flow.~~ **Do not build this. C5 removed it entirely** — there is no client-side gym upload path to build against. The client's whole gym surface is `POST /gyms` once and `POST /gyms/ping` on app-open; session synthesis is server-side. |
| `AppUI/TrackedRunView.cs` | `startRunSession` / `stopRunSession`, with a hard timeout on `OnRunSessionEnded`. C14 guarantees exactly one reply — `OnRunSessionEnded` or `OnRunSessionError`, never an empty `OnRunSessionEnded`. |
| `AppUI/GymsSettingsView.cs` | `GET`/`POST /gyms`, the five-gym cap, `409 gym_limit_reached`. |
| `AppUI/GuildView.cs` | create / join / invite / members. **World-boss surface deliberately not built** (B5) — C10 independently confirms those fields are hardcoded `null` at the route. |
| `AppUI/RosterView.cs` | Character and stats from `/state`. |
| `App/InviteCodeDrain.cs` | `consumePendingInviteCode()` once live and logged in, and again on `OnApplicationFocus(true)`. |
| `App/PushRegistration.cs` | `registerPush()` plus **both** token receiver methods — omitting either rebuilds the exact bug that left the original push path dead. **The field is `token`, not `fcmToken`** (C1); an `fcmToken` body 400s. |
| `AppUI/StravaSettings.cs` | `GET /strava/connect` → `Application.OpenURL`; `POST /strava/sync`; hide the button on `503 strava_not_configured`. |

### 1.9 Phase 7 — The run-settlement seam

*Depends only on Phase 0. **Recommended second, before any new screen.*** Rationale and detail in §4.

**Citations below are symbol-first, not line-first.** This table's original line numbers were flagged
as pre-refactor before this phase even started; by the time this
correction was written, `RunController.cs`, `GameRoot.cs`, `RunResultView.cs`, `StubCombatController.cs`
and `CombatController.cs` had each been observed to move mid-session, more than once. A symbol survives
a refactor; a line number is a snapshot of the moment it was checked. Re-locate by symbol if a cited
number stops matching.

| File / edit | Responsibility |
|---|---|
| `Runtime/RunOutcome.cs` | Plain-C# description of a finished run: `Seed, Cleared, EndRow, BossRow, RowCount, Gold, PartyHp, Fights, Rounds, Nodes, Pickups`. Promoted almost verbatim from `Assets/Tests/EditMode/RunBehaviourHarness.cs`'s `HeadlessRunResult`; see §4.3 for why the field list is not identical. |
| `Runtime/RunSettlementResult.cs` | R2 account state handed back: `IsSettled` plus **nullable** `Xp`/`Gold` (null means the server said nothing, 0 means it said zero). Static `Unsettled`. |
| `Runtime/IRunSettlement.cs` | `void Submit(RunOutcome, Action<RunSettlementResult>)`. The single seam — the only type permitted to reference both a run type and an account type. |
| `Runtime/LocalRunSettlement.cs` | Calls back immediately with `RunSettlementResult.Unsettled`. Exactly today's behaviour: nothing is credited anywhere. Writes nothing to disk. |
| edit `RunController`'s `OnRunEnded` event | `event Action<bool>` → `event Action<RunOutcome>`. Every raise site (`EndRunAsDefeat` and the two `EnterEncounter` callbacks) now builds its outcome through one `BuildOutcome(bool cleared)` helper, so the raise sites cannot drift apart on which fields get read. |
| edit `GameRoot` | Owns an `IRunSettlement` (`LocalRunSettlement` today) alongside `ICombat` and `EncounterComposer`; on `OnRunEnded`, calls `Submit` and routes the settled result into `RunResultView.ShowSettlement`. |
| edit `RunResultView` | Gains `ShowSettlement(RunSettlementResult)` **alongside**, not instead of, its unchanged `Initialise(RunState, bool)`. Reward figures render **only** from `ShowSettlement`, and only when `IsSettled` is true. |
| edit `StubCombatController.EnterEncounter` | Deleted the duplicated gold formula; now calls `CombatMath.GoldReward`. |
| `Tests/EditMode/NetBoundaryTests.cs` | The two reflection guards described in §2.3. |

### 1.10 Phase 8 — Android build configuration

*Cannot start until `hostbridge.aar` exists (except the manifest and templates, which can be
authored blind). **Do not create anything under `Assets/` for this until Unity is free.***

| Item | Notes |
|---|---|
| `Assets/Plugins/Android/AndroidManifest.xml` | brief §7.2 verbatim **except** the theme — see A4. |
| `Assets/Plugins/Android/hostbridge.aar` | Absent. Add a `README.md` in that folder naming the exact expected filename and its origin, so a missing file is discoverable rather than mysterious. |
| `Assets/Plugins/Android/mainTemplate.gradle` | Every `implementation` line from §7.3 — do not pin the count here, the `04/08/2026` delivery already added one. `kotlin-stdlib` first and commented as the one people forget; `androidx.work:work-runtime-ktx` is the one that fails hardest, since `WorkManager.getInstance()` runs in `UnityHostActivity.onCreate` and the host survives its absence by logging an ERROR, leaving background sync silently dead. |
| `Assets/Plugins/Android/launcherTemplate.gradle` | `apply plugin: 'com.google.gms.google-services'`. |
| `Assets/Plugins/Android/google-services.json` | Gitignored; ask Yarin. Without it `registerPush` degrades silently. |
| `Assets/Scripts/Editor/` addition | Extend `SetupValidator.cs` with a `Reign & Gain / Validate Integration Config` check: appconfig present, redirect literal matches `HostWireNames`, `AndroidTargetSdkVersion == 36`, `hostbridge.aar` present. Cheap, and it catches precisely the silent failures §7–§8 warn about. |

---

## 2. The seams that protect us

### 2.1 A missing `.aar` must not block anything

Three pieces: `IHostBridge` (the interface), `HostBridge.Current` (one locator, the codebase's only
platform check), `FakeHostBridge` (the stand-in). Three properties the fake must have, each learned
from a real hazard in this repo:

1. **It must be seeded, not random.** CLAUDE.md forbids `UnityEngine.Random`, and a fake that returns
   a random step count makes an AutoPilot run non-reproducible — the exact failure the invariant
   exists to prevent. Use `SeededRng` with a fixed seed so "the fake device" is one fixed device.
2. **It must reproduce the async shape, not just the values.** `requestHealthRead` must deliver
   0..N callbacks spread over several frames, with no terminator. `startGoogleSignIn` must be able to
   return `false`, and to deliver `"cancelled"`, and to deliver some other reason. If Phase 5 is
   written against a synchronous fiction it will break on device, and it will break in the timing
   layer, which is the hardest place to debug.
3. **It must be able to fail on demand.** An editor menu (or inspector toggles on the fake) to force:
   permission denied, no Health Connect, empty window, sign-in cancelled, sign-in unavailable,
   `stopRunSession` with no session, push token error. Every one of these has a distinct UI branch and
   none of them is otherwise reachable before the `.aar` arrives.

And on the real side: `AndroidHostBridge` must treat `UnityBridge.get() == null` as a degraded state
that logs once, not as an exception per call. That state is what a mis-merged manifest produces, and
it is the most likely first-device-session failure.

### 2.2 REST DTO shape

Three rules, all enforceable by review in seconds:

- **Every optional wire field is a C# nullable** — `int?`, `double?`, `bool?`, or a nullable
  reference. Never a defaulted value type. On this contract, *presence is information*: `pending`
  absent means "rejected", `trust` absent means "duplicate", and `metrics.distanceMeters` absent
  means "Health Connect had no distance" — which is a completely different claim from `0`.
- **Every reward-bearing property is get-only.** `xp`, `gold`, `stats.*`, `pending.*`, `applied.*`,
  `trust.total`. `+=` then does not compile, so the "never recompute" rule is a build error rather
  than a review comment.
- **`ReignAndGain.Net` references no gameplay type.** No `RunState`, no `RewardEffects`, no
  `RewardDefSO`. The `Net` namespace is a transport layer that happens to carry numbers.

### 2.3 "Server owns all maths", enforced structurally

Discipline will not survive twelve screens. Four mechanisms:

1. **Two reflection tests** in `Assets/Tests/EditMode/NetBoundaryTests.cs`:
   (a) every numeric property on every type in `ReignAndGain.Net.Dto` has no public setter;
   (b) no type in `ReignAndGain.Net` references `ReignAndGain.RunState` or `ReignAndGain.RewardEffects`,
   and no type outside `ReignAndGain.Net` plus the settlement seam references a DTO type.
   About 40 lines. Fails the day someone crosses the line.
2. **Views take DTOs, not numbers.** `HomeView.Initialise(GameStateSnapshot)`, not
   `Initialise(int xp, int gold)`. A screen that cannot be handed a loose `int` cannot display a
   computed one. This is the same shape as the existing rule in `docs/ARCHITECTURE.md:310` — "a view
   never computes a rule" — extended to "a view never computes an award".
3. **No local patching of the snapshot.** After `POST /activities/register`, `POST /sources` or
   `POST /strava/sync`, re-fetch `GET /state`. Patching the local snapshot with the `applied` block
   is a client-side computation wearing a disguise, and it will drift the first time the server adds a
   level-up side effect.
4. **A grep that a reviewer can run:** `rg "(Xp|Gold|xp|gold)\s*[+-]=" Assets/Scripts/Net` must
   return nothing, ever.

---

## 3. The JSON library decision

### 3.1 What the contract actually demands of a serializer

- **Omission on write.** `metrics` is "object, all optional". Sending `avgHeartRate: 0` or
  `distanceMeters: 0` is not "no data", it is *a claim of zero* — and a `running` bundle claiming
  0 m of distance is either scored as a bad workout or rejected. Same for
  `gpsContext: {lat:0, lon:0, accuracyMeters:0}`, which places the user in the Gulf of Guinea and
  will fail the geofence check against a registered gym.
- **Absence detection on read.** `POST /activities` returns three different 200 bodies discriminated
  purely by which key is missing: `pending` absent → rejected; `trust` absent → duplicate. A
  serializer that reports absent-as-default renders a rejection as "+0 XP, +0 gold, accepted".
  That is the *worst possible* bug in this integration, because it looks like a success.
- **Nullable value types and nullable objects** — `guild: null`, `displayName: null`,
  `lastActivityAt: null`, `sourceWorkoutId: string|null`, `inviteUrl?`.
- **Pass-through of an unknown blob** — `details: { /* zod flatten */ }` has no fixed shape.

### 3.2 The options

**`JsonUtility` (built in).** Cannot omit a field on serialize — it writes every field of the type,
always. That single limitation disqualifies it for `POST /activities` before any of its other problems
(no `Dictionary`, no nullable value types, no distinction between absent and default, and no top-level
array — the bridge contract already worked around that last one by wrapping `ConnectedSources` in a
`sources` key). It could parse the *bridge* payloads. It cannot write the bundle.

**Hand-written parser and writer.** Realistically 600–900 lines including a tokenizer, plus its own
test suite. Zero dependency and total control over presence semantics, which is genuinely attractive.
Against it: it puts our own bugs in the one code path where a mistake is a 400 with a Zod flatten we
cannot easily read, and the failure mode of a subtly wrong writer is "the server rejects everything
and we cannot tell why". Not worth it for a fixed, documented, modestly sized contract.

**`System.Text.Json.dll`, already sitting in `Assets/Plugins/NuGet/`.** **Reject.** `.gitignore:13`
excludes `Assets/Plugins/NuGet/` wholesale (only `.nuget-installed.json` is un-ignored at `:14`), so
that DLL is a machine-local MCP artifact and is not in the repository — a colleague's clone would fail
to compile. Worse, its `.meta` is configured `Any: enabled: 1` with `Exclude Editor: 1` and
`Editor: enabled: 0`, i.e. it would be **shipped in the Android player and absent from the Editor** —
a configuration nobody chose. Flag it in `VERIFY-WITH-OWNER.md` as a build-size and stripping hazard
to check on the first real Android build, but never reference it from game code.

**`com.unity.nuget.newtonsoft-json` (UPM, 3.2.1).** Unity-maintained repackaging of Newtonsoft.
Gives `NullValueHandling.Ignore` (omission on write, per-property), `T?` (absence on read),
`Dictionary<,>`, and `JObject`/`JToken` for the `details` blob. Costs: a managed DLL in the APK
(~1.6 MB before stripping), and reflection-based (de)serialization under IL2CPP means DTO members can
be stripped — fixed with a `link.xml` or `[Preserve]`, a well-documented two-line problem.

### 3.3 Recommendation — ✅ DECIDED 2026-08-05: Newtonsoft (audited 2026-08-04, decided next session)

**Owner decision, 2026-08-05: Newtonsoft, as originally recommended below.** `com.unity.nuget.newtonsoft-json`
`3.2.1` is added to `Packages/manifest.json` and `Newtonsoft.Json.dll` is wired into
`ReignAndGain.EditModeTests.asmdef`'s `precompiledReferences`. This does not yet mean any payload type
is written — `RawHealthSignals`, `HealthMetrics`, `SleepSummary` and the rest of `HostPayloads` are
still the next real Phase 1 task, now unblocked. The audit trail below is kept for the reasoning.

**As audited 2026-08-04, the tree used `JsonUtility` everywhere and Newtonsoft was not installed.** Seven usages across
`AppConfig.cs`, `HostPayloads.cs`, `HostBridgeReceiver.cs` and two Editor files; zero references to
`Newtonsoft` or `JsonConvert` anywhere in `Assets/`. `com.unity.nuget.newtonsoft-json` was never added
to `Packages/manifest.json`. Whoever built Phase 0/1 chose the other way and did not record why.

**This is not a cosmetic divergence, and it is not yet resolved.** Justification 1 below — the reason
this section recommends Newtonsoft at all — is that the most dangerous defect in this integration is a
rejected or duplicate activity rendering as a zero award, and that is the *default behaviour* of any
serializer conflating absent with default. `JsonUtility` is exactly such a serializer: it **cannot
serialize nullable value types at all**, so the `HostPayloads` row's "every optional numeric a
`double?`" is not merely unimplemented, it is unimplementable as written while `JsonUtility` stands.

It has not bitten yet only because the eight payload types that carry optional numerics do not exist —
`GoogleIdTokenPayload` is two `string` fields, where absent and empty are equally harmless. The
collision lands the moment `RawHealthSignals`, `HealthMetrics` or `SleepSummary` is typed, which is the
next real Phase 1 task, and it lands again in Phase 5's `EvidenceBundleAssembler`.

**Decide before writing those types, not after.** Three viable routes, in rough order of cost:
add Newtonsoft as this section originally argued; keep `JsonUtility` and carry an explicit
`hasX`/sentinel presence flag per optional field, which the assembler must then honour; or keep
`JsonUtility` for the bridge and confine Newtonsoft to the REST layer. The one option that is not
available is leaving it undecided and typing the payloads anyway — that silently picks "absent means
zero", which is the §0 violation this plan exists to prevent.

The original recommendation, unchanged, follows.

**Add `com.unity.nuget.newtonsoft-json` and use it for both the bridge payloads and the REST layer.**

Justification, in order of weight:

1. The most dangerous defect available in this integration is a rejected or duplicate activity
   rendering as a zero award, and that defect is the *default behaviour* of any serializer that
   conflates absent with default. Newtonsoft makes presence a type-level concept, so the §0 invariant
   is enforced by the compiler rather than by whoever reviews the pull request.
2. It is the only option that is both in the repository and in the Editor. It goes into
   `Packages/manifest.json`, which **is** source-controlled — unlike `Assets/Plugins/NuGet/` — so
   every teammate gets it on `git pull` with no extra step. That matters more than usual here: most of
   this team are not build engineers.
3. The bridge contract's own aside — `ConnectedSources` is wrapped in a `sources` key so it "also
   parses under Unity's `JsonUtility`, **not only Newtonsoft**" — indicates Yarin's previous client used
   Newtonsoft. Matching it removes a whole class of "works on his side" divergence.
4. Its IL2CPP behaviour is the best-understood of any option, and the fix is known in advance.

Two follow-through steps, easy to forget:

1. **`ReignAndGain.EditModeTests.asmdef` needs `"Newtonsoft.Json.dll"` in `precompiledReferences`.** It
   sets `overrideReferences: true`, so the reference is not inherited.
2. **`Assets/link.xml` needs no edit for this.** Its
   `<assembly fullname="ReignAndGain" preserve="all"/>` is a whole-assembly preserve that already
   covers any type added to `ReignAndGain.Host`, DTOs included. Worth knowing before someone adds a
   per-type entry that does nothing.

---

## 4. The architectural tension

CLAUDE.md: *"the engine owns rules; views own timing"*, `CombatEngine` pure and synchronous, a run
fully reproducible from its seed, test-pinned. `docs/ARCHITECTURE.md:263`: *"A run must reproduce
exactly from its seed. That is the reason bug reports are actionable at all."*
Yarin §0: *"The client **never computes** XP, gold, stats, loot, levels, trust scores, or any award."*

These appear to be in direct conflict. They are not, and the reason is worth stating precisely,
because getting it wrong in either direction is expensive.

### 4.1 Where the client computes rewards today — exact sites

**Combat gold — the formula and its three application sites:**

- `Assets/Scripts/Combat/CombatMath.cs:228-233` — the formula, rebalanced since this section was first
  written: `GoldReward(floor, rng)` = `Math.Max(1, floor / 2) + rng.Range(3) / 2`, then `+1` at 15%.
  (This section used to quote the pre-rebalance formula, `(2 + floor) + rng.Range(3)` then `+2` at
  15% — that was not just a stale quote, it was the exact formula `StubCombatController` was still
  running, unaware of the rebalance, until Phase 7 deleted its private copy in favour of calling this
  one. See below.)
- `Assets/Scripts/Combat/CombatEngine.cs:774` — `RollGold()`, drawing from the engine's `_rng`.
- `Assets/Scripts/CombatUI/CombatController.cs:1375-1378` — **application**, inside its post-fight
  `Finish` coroutine: `int gold = _engine.RollGold(); _run.Gold += gold; _fx.Banner($"VICTORY! +{gold}
  GOLD", …)`.
- `Assets/Scripts/Combat/StubCombatController.cs` — used to be a **second, duplicated copy** of the
  formula, inlined, used by `GameRoot`'s headless mode and the PlayMode boot test. As of Phase 7 it
  calls `CombatMath.GoldReward` directly instead (§1.9).
- `Assets/Tests/EditMode/RunBehaviourHarness.cs:186-187` — a **third** application site.

**Interlude gold:**

- `Assets/Scripts/Interlude/TreasureNode.cs:27-29` — `run.Rng.Range(5, 10)` then `run.Gold += …`.
- `Assets/Scripts/Interlude/EventOutcomes.cs:52-53` — `run.Gold += choice.Magnitude`.
- `Assets/Scripts/Interlude/ShopStock.cs:76,83` — gold **spent**: `run.Gold -= reward.Cost`.

**Non-gold gains applied straight to `RunState`:**
`Assets/Scripts/Interlude/RewardEffects.cs:34-58` (MaxHp, heals, `RerollMax`, `FocusMax`, Bulwark,
Regen, QteWide, cures) and `Assets/Scripts/Interlude/RestNode.cs:44`.

**Display sites:** `Assets/Scripts/UI/MapView.Chrome.cs:207`, `RunResultView.BuildPursePlaque`,
`Assets/Scripts/InterludeUI/ShopView.cs:349`, and the `[RUN]`/`[WALK]` log lines in `GameRoot` (line
numbers omitted for `RunResultView` and `GameRoot` — both are under concurrent edit; see the note at
the top of §1.9).

**And the finding that resolves the tension: there is no XP and no character level in the client at
all.** Grepping all 86 scripts for `xp` / `experience` returns nothing gameplay-related; the only
`Level` is `Assets/Scripts/Definitions/SkillDefSO.cs:30`, a WIP stub whose own comment says every
skill stays at level 1 and nothing reads it. So the client computes exactly **one** number that looks
like a reward, and `RunController.StartRun`'s `RunState` initializer sets `Gold = 0` at the start of
every run. Nothing carries it between runs; `RunState.Gold` is discarded when the run ends.

### 4.2 Why that is not a §0 violation

`RunState.Gold` and `GET /state.character.gold` are **two different quantities that happen to share a
name.** Run gold is a within-simulation resource, like ammunition: created inside one dungeon run,
spent inside the same run at `ShopStock.Purchase` (`ShopStock.cs:83`), and destroyed with the run.
It is never credited to an account, never persisted, never displayed outside the run, and never
crosses a wire.

Yarin's §0 forbids the client deciding **what a workout was worth** — what gets credited to the
persistent character. It cannot coherently forbid a deterministic simulation from having internal
state, or it would also forbid the client from computing enemy HP. The moat is about *crediting*, and
nothing about the run is credited anywhere today.

So the conflict is entirely about **naming and about the future**: the moment a run's outcome is meant
to be worth something, the client must not be the thing that decides how much.

### 4.3 The minimal refactor

Three changes. None touches the maths, and none moves an RNG draw.

**(1) Make the run's outcome a value instead of a side effect.**
`RunController.OnRunEnded` becomes `event Action<RunOutcome>` — it was `event Action<bool>`,
raised from `EndRunAsDefeat` and from the two `EnterEncounter` callbacks, and consumed by `GameRoot`'s
own `OnRunEnded` handler. `RunOutcome` is plain C#.

**The decided shape is `Seed, Cleared, EndRow, BossRow, RowCount, Gold, PartyHp, Fights, Rounds,
Nodes, Pickups`** — not the `CoinEarned, CoinSpent, PartyHpRemaining, … Trace` shape this paragraph
originally proposed. Three departures from that original shape, each deliberate:

- **No `CoinEarned`/`CoinSpent` split.** Nothing in the codebase tracks earned and spent gold
  separately — `RunState.Gold` is a single running balance that combat and treasure add to and the
  shop subtracts from (§4.1). Splitting it on the way out would mean new bookkeeping across four files
  for two fields nothing downstream reads.
- **`Gold` and `PartyHp`, not `CoinEarned...`/`PartyHpRemaining`.** These keep the test harness's
  existing names on purpose: `Assets/Tests/EditMode/SeedBaselineTests.cs:31` states its `Baseline`
  struct mirrors `HeadlessRunResult.Baseline()`'s output exactly, and
  the owner's B2 decision says outright not to rename `Gold` — twelve baseline rows and roughly twenty
  call sites reference it (§4.5's B2, decided in
  `Interpretable-Context-Methodology/inbox/integration-decisions-unity-backend.md` §3).
- **No `Trace`.** A production replay trace means instrumenting the real fight path — a feature, not a
  refactor — and §7 says not to invent the endpoint it would be designed against. `RunOutcome.cs`
  carries a `TODO(contract)` marker where it would go, instead of a guess at its shape.

Every field it *does* carry **already existed**, in the test harness:
`Assets/Tests/EditMode/RunBehaviourHarness.cs`'s `HeadlessRunResult` (`:38-73`) defines almost exactly
this shape, and `HeadlessRun.Play` builds one per seed. The refactor is largely "promote the harness's
result type into production code". That the codebase independently grew this type for its own testing
is the strongest available evidence that the seam is natural rather than imposed.

**(2) Add one interface, and implement it locally as a no-op.**
`IRunSettlement { void Submit(RunOutcome, Action<RunSettlementResult>); }`, with `LocalRunSettlement`
calling back immediately with `RunSettlementResult.Unsettled` — bit-identical to today's behaviour,
since today nothing is credited. `GameRoot` injects it exactly as it already injects `ICombat` and
`EncounterComposer`. `RunResultView` gained a new `ShowSettlement(RunSettlementResult)` method —
**alongside**, not folded into, its existing `Initialise(RunState, bool)` — and renders **reward**
figures only from `ShowSettlement`'s argument, so a reward line on the result screen *physically
cannot* be computed locally: there is no local number available to draw it from. When the server
endpoint exists, one new class implements the interface and nothing else changes.

**(3) Delete the duplicated formula.** `StubCombatController.cs` used to re-implement
`GoldReward` inline; it now calls `CombatMath.GoldReward` instead. You cannot relocate a formula that
exists in two places, and you certainly cannot prove you relocated it.

### 4.4 What must NOT change, and why

`CombatController.cs:306` and `RunBehaviourHarness.cs:140` both construct the engine with
**`run.Rng`** — the single per-run stream, not a per-fight child.
`docs/ARCHITECTURE.md:265-269` states this explicitly and lists `gold` among the things drawn from it.
So `RollGold()` consumes **1 or 2 draws** from the shared run stream on every win (`rng.Range(3)`,
then `rng.Chance(0.15)`).

Therefore: **removing, skipping or relocating the gold roll shifts every subsequent draw in the run.**
The map is already generated by then, inside `RunController.StartRun`'s own `RunState` initializer,
but every later fight's intents and card draws, every affliction roll, `ShopStock.Roll`,
`TreasureNode.Grant` and the event-definition draw in `GameRoot` all move. All **12** pinned rows in
`Assets/Tests/EditMode/SeedBaselineTests.cs:180-202` would change; `RunInvariantTests.cs:206` (same
seed → same final gold) would still pass, so the *only* signal would be the baseline table — which is
precisely what that file's header says it exists for, but it is churn with no behavioural gain.

**So: keep the roll where it is, keep it in the stream, keep applying it to `RunState`. Change only
who is allowed to read it as a reward.** Phase 7 did exactly this — see §4.3 — and left the roll's
position in the stream untouched.

Corollary for the eventual server call: design the submission around **"here is the seed and the
inputs"**, not **"trust these totals"**. A server that re-scores needs to be able to re-derive the
run, which means `RunOutcome` must eventually carry the seed and the player's decision sequence.
`HeadlessRunResult.Trace` (`RunBehaviourHarness.cs:67`) is already the right shape for that field — it
exists for the determinism test. **`RunOutcome` deliberately does not carry it** — see §4.3's note on
`Trace` — and carries a `TODO(contract)` marker instead until an endpoint exists to design it against.
Designing around totals would produce a seam that cannot compose with a re-scoring server, which is
the whole point of §0.

### 4.5 The boundary, stated explicitly

- **R1 — simulation state.** Anything inside `RunState` is computed locally, seeded, test-pinned, and
  **never displayed with reward framing** (no "+80 XP", no "you earned"). `RunState.Gold` is in-dungeon
  coin: earned in the dungeon, spent in the dungeon, discarded at the end.
- **R2 — account state.** Anything inside `GameStateSnapshot` / `RegisterResult` / `ActivityUploadResponse`
  is only ever assigned from a parsed response field. Never computed, never incremented client-side.
  Get-only properties make that a compile error (§2.2).
- **R3 — one bridge.** The *only* thing that may touch both is `IRunSettlement`: one `RunOutcome` out,
  one `RunSettlementResult` in. Nothing else in the codebase may reference both a `RunState` and a DTO,
  and that is checkable with a grep — which is what makes it structural rather than disciplinary.

The server-resolved run endpoint **does not exist**, so Phase 7 ships `LocalRunSettlement` only, with
`RunSettlementResult`'s shape left minimal and a `// TODO(contract): endpoint TBD` marker. Do not
invent the endpoint. Do not invent the workout→game hook either — Yarin §9 and
`VERIFY-WITH-OWNER.md:85` both say it is undecided, and the seam is exactly what keeps it cheap.

**One naming recommendation, for the owner (B2):** do **not** rename `RunState.Gold`. The field name
appears in 12 baseline rows' printed format (`RunBehaviourHarness.cs:72`) and across ~20 call sites,
and a rename buys no behaviour. Instead document it (an XML-doc on `RunState.Gold` and a line in
`docs/ARCHITECTURE.md`) and change only the **player-facing** word if the owner wants the two purses
distinguished on screen. Note that the in-dungeon purse display (`MapView.Chrome.cs:207`) is already a
bare number beside a coin glyph, with no literal word "Gold" — the same rule `RunResultView`'s own
purse-plaque doc comment states explicitly — so this may be free.

---

## 5. Ambiguities and conflicts

Blunt, and in three buckets. Each entry: what is unclear, why it blocks or risks work, and a default
to proceed under.

> **These are the questions as originally asked. Every one of them has since been answered.**
>
> The answers are **not** restated here — a partial local copy of partner material beside a complete
> one is the exact failure documented in §6 of the decision record. Read the verdicts there:
> `Interpretable-Context-Methodology/inbox/integration-decisions-unity-backend.md`, §2 for C1–C18 and
> §3 for B1–B6.
>
> Several defaults below were **overruled** by the answer that followed. Where an entry carries its own
> "Superseded" note, that note wins over the default above it. Where it does not, check the decision
> record before implementing a default from this section.

### (a) We can decide ourselves

**A1 — `requestHealthRead` has no batch-complete signal.** 0..N `OnHealthSignalsRead` "followed by
nothing"; no count, no terminator, no correlation id. *Risk:* we cannot know when to re-fetch `/state`
and reveal the claim bar; and any UI that waits for completion waits forever — the exact soft-lock
shape CLAUDE.md's timeout invariant was written for. *Default, as originally written:* `HealthReadSession`
with a quiescence timer (3 s after the last message, 15 s hard cap from the call) that **always**
completes; upload each signal as it arrives so a late arrival is harmless; re-fetch `/state` once on
completion. Treat a mid-batch `OnHealthReadError` as end-of-batch, not as invalidating signals already
received. Also ask Yarin (C6).

**Superseded — do not implement the quiescence-timer default above.** `OnHealthReadComplete` landed in
host `29/07/2026-b`; this section predates it. The wire now has a real terminator, guaranteed to fire
exactly once, always last, on every path (see `unity-bridge-contract.md` § "One read fans out to many
signals"). Build against *that*:

- **The terminator is the sole normal end-of-batch signal.**
- **A single 30 s hard cap is the only defensive timer, and there is no quiescence clock.** It exists
  solely to catch the terminator never arriving at all — host crash, stale `.aar`, dropped JNI
  callback — never to guess "done" from a gap between messages.
- **`OnHealthReadError` never ends the session by itself**, matching the original default.

A quiescence timer is rejected specifically because `unity-bridge-contract.md` says the terminator
"replaces inferring 'done' from a timer, which is either too short (drops workouts) or too long".
Keeping one reintroduces the too-short failure on a legitimately slow read — a cold Health Connect
cache fanning out workouts, step-days and sleep in one call.

**Four things stay outside this class on purpose:** `/state` re-fetch, upload-per-signal, retry on a
forced (hard-cap) completion, and re-delivery of a signal arriving after the session closed. Each is
the calling layer's problem to define, and wiring them here would guess at an unwritten caller's needs.

**A2 — `OnHealthReadError` is unstructured plain text conflating three different situations.**
Brief §3 lists permission denied (actionable: show a grant affordance), no Health Connect app
(actionable: link to Play), and no workout found (**not an error at all**) under one plain-text
method. *Risk:* the UI cannot branch without string-matching prose that Yarin is free to reword.
*Default:* treat every `OnHealthReadError` as non-fatal and render "no workouts found"; drive the
permission affordance off `permissionStatus()`, which **is** structured. Never string-match the
reason. Also ask Yarin (C7).

**A3 — `SleepSummary` is the one payload with no example JSON and no field types.** Given only as
`{ startedAt, endedAt, totalMinutes }`. *Risk:* low. *Default:* parse `totalMinutes` as `double?`,
render rounded. And never upload it — brief §5 is emphatic.

**A4 — brief §7's `android:theme="@style/UnityThemeSelector"` is wrong for this project.**
`ProjectSettings/ProjectSettings.asset:87` has `androidApplicationEntry: 2` = **GameActivity**, and
the bridge contract itself says `UnityHostActivity` subclasses `UnityPlayerGameActivity`.
`UnityThemeSelector` is the classic-`Activity` theme; GameActivity uses
`@style/BaseUnityGameActivityTheme`. *Risk:* a mis-themed or black launch, or a manifest-merge
conflict, on the first device build. The brief already grants us this call ("take Unity's values if
they differ"). *Default:* run one Android build with the plugin absent, read Unity's own generated
`AndroidManifest.xml` under `Temp/gradleOut/`, and copy its `android:theme` and `configChanges`
verbatim. Record in `VERIFY-WITH-OWNER.md`.

**A5 — no contract states a JSON library.** *Default:* add `com.unity.nuget.newtonsoft-json`; see §3.

**A6 — `Assets/Plugins/NuGet/System.Text.Json.dll` is a trap, not an opportunity.** Gitignored
(`.gitignore:13`) so it is not in the repo, and its `.meta` ships it in the player while excluding it
from the Editor. *Default:* never reference it; flag the shipping-in-player configuration in
`VERIFY-WITH-OWNER.md` as something to check on the first real Android build (APK size and stripping).

**A7 — `code_challenge_method=s256` is lowercase in the contract**; RFC 7636 says `S256`. GoTrue
accepts lowercase. *Default:* send the contract's literal exactly, with a comment saying it is
deliberate, so nobody "corrects" it.

**A8 — the PKCE `code_verifier` may not survive the browser round-trip.** The contract says keep it
in memory. On Android, launching the system browser can get Unity's process killed; a `code` then
arrives (or doesn't — see C4) with no verifier to exchange it against, and login fails with no useful
error. *Risk:* intermittent login failure on low-memory devices, which is the worst kind of bug to
diagnose remotely. *Default:* persist the verifier to `PlayerPrefs` for the duration of the flow and
delete it on exchange or after 10 minutes. Defensible because the contract itself says the code "is
single-use and worthless without the `code_verifier`" — and the verifier alone is worthless without
the code. Note in `VERIFY-WITH-OWNER.md`; this is the only place we deliberately persist an auth
artifact.

**A9 — `manualEntryFlag` is optional on the wire and required in the bundle.** Brief §3: optional
fields are "omitted when absent". Brief §5: `manualEntryFlag` is **bool (required)**. If Health
Connect gives no value, the client must invent one — and inventing `false` is asserting "the OS did
not flag this as hand-entered", which is a **trust signal**, i.e. exactly the category §0 says the
client must not decide. *Default (interim):* `false` for `health_connect`, `true` for `manual`, and log
it. Escalated as C8, because the correct answer is Yarin's.

**A10 — `sourceWorkoutId`: "`string | null`" versus "omit for sources without one".** Brief §5's
table says nullable; the REST contract says omit. Whether the Zod field is `.optional()`,
`.nullable()` or `.nullish()` decides whether sending `null` 400s the whole bundle. *Default:* **omit
the key entirely** whenever a value is absent, for every optional bundle field. Omission is accepted
under all three Zod variants; `null` is not. Also raised as C9 for the response direction.

**A11 — `activityId` reuse across a relaunch is unspecified.** §5 says "a retry must reuse it" but not
for how long. *Risk:* low — the contract says the stable `sourceWorkoutId` catches a re-read, so
nothing double-credits. But a fresh `activityId` for the same workout turns a silent 200 upsert into a
user-visible `duplicate_workout`, i.e. a different narrative for the same real event. *Default:*
persist the `sourceWorkoutId → activityId` mapping so a retry is a true retry.

**A12 — `GET /sources.catalog[].requiresManualFlag` appears in the example and in no prose.** Its
meaning is unstated: "bundles from this source must set `manualEntryFlag: true`"? "this row only
appears when `manualLoggingEnabled`"? *Default:* ignore it for rendering — the bullets already say the
server filters `catalog` — and derive **no** bundle field from it. Also ask Yarin (C13).

**A13 — new shell UI will silently hang `AutoPilot` if we are not careful.** `AutoPilot.cs:224-245`
(`CombatStep`) iterates **every** `Button` in the scene, excluding only children of `MapView`,
`CharacterSelectView` and `DeckDraftView`, and its final fallback is
`else if (card == null) card = b;`. A persistent shell tab bar visible during combat would therefore
be treated as a playable card and the smoke test would start pressing "Guild" mid-fight.
`ClickFirstReachableNode` (`:202-211`) clicks the first interactable Button under the active
`MapView`, so anything reparented under `MapView` is also hazardous. *Default:* the Game tab hides all
shell chrome while `GameRoot` owns the screen, **and** add the shell root type to `CombatStep`'s
exclusion list in the same commit. Also add `HostBridgeReceiver` to CLAUDE.md's frozen-name list — the
bridge makes it a name that breaks the wire silently, which is strictly worse than breaking a smoke
test.

**A14 — `appconfig.json`'s gitignore split — DECIDED 2026-08-05, owner confirmed: no split.** The brief's
original default was to gitignore the file (below, unchanged for the record) and commit a sample with a
load-time fallback. The tree instead committed the real file with live values, and the owner confirmed
that stays: `sb_publishable_*` is client-safe by design and ships in the APK regardless, `IsComplete()`
already rejects a `sb_secret_` key, and CODEOWNERS review on the path only works while it's a real
committed file. No `appconfig.sample.json`, no `AppConfig.Load()` fallback, no `SetupValidator` check
for it — the original default below is superseded.

*Original brief default, for the record:* commit `appconfig.sample.json`; ignore both `appconfig.json`
and `appconfig.json.meta`; load the sample as fallback with one warning; add the check to
`SetupValidator`.

**A15 — the three contract docs live at the repo root, but every cross-reference calls them
`docs/unity-*.md`.** Trivial, except that `Assets/Scripts/Editor/ContributorMenu.cs:26` opens docs by
repo-relative path, so any menu entry we add must match reality. *Default:* leave them where they are
(another agent may be touching them) and use root-relative paths. Note it.

### (b) Needs the owner

**B1 — orientation, and it is the biggest unflagged item in the brief.**
This project is landscape-locked: `ProjectSettings.asset:11 defaultScreenOrientation: 4` with both
portrait rotations disabled (`:61-62`), `GameRoot.cs:52` sets a `1920x1080` reference and `:141-145`
sets `matchWidthOrHeight: 0` (match **width**, chosen so a taller device letterboxes rather than
cropping the board), and every screen in `Assets/Scripts/UI/` is composed for that frame.
Yarin's shell — splash → sign-in → source picker, then a **Home / Roster / Game / Guild tab bar** with
a claim bar and a Settings sub-screen — is the shape of a portrait phone app. **Neither contract states
an orientation anywhere**, and the document that would (`docs/screen-flow.mermaid`) was never uploaded.
Three options, all with real cost:

| Option | Cost | Risk |
|---|---|---|
| Landscape everywhere | Cheapest. One canvas config, all existing art and `RgSkin` composition reused. | A fitness dashboard that only works sideways reads as broken; likely contradicts designs we have not seen. |
| Portrait shell, landscape game, `Screen.orientation` switched on entering/leaving the Game tab | Two canvas configurations and two layout idioms; every shell screen authored portrait-first. §7 describes a `configChanges` list, but neither `AndroidManifest.xml` nor `hostbridge.aar` actually declares one — see the correction below — so whether the switch is cheap or triggers an Activity restart is unverified. | Moderate. Needs care at the transition and on the splash. |
| Portrait everywhere | Rebuilds the entire existing game UI. | Unacceptable. |

*My recommendation to put to the owner:* **option 2**, shell authored portrait-first. But this is a
product and art decision with the largest downstream cost in this plan and it must not be made
implicitly by whoever writes the first shell screen. **Decide before Phase 3.**

**Decided, and this table is now historical.** Issue #21 (closed 2026-08-01) picked **landscape
everywhere**, not option 2 — the reasoning is recorded in
`Interpretable-Context-Methodology/inbox/integration-decisions-unity-backend.md` §3. Separately,
the `configChanges` claim above was never true: no manifest in this repo declares one. It didn't
change the decision, but don't cite this table's cost/risk column as verified — see
[#44](https://github.com/Get-Sweaty-Games/reign-and-gain-unity/issues/44).

**B2 — two different quantities will both be called "gold" on screen** (`RunState.Gold` versus
`character.gold`). Recommendation and rationale in §4.5: document, don't rename; change player-facing
copy only if the owner wants them distinguished.

**B3 — must the existing run loop still work with no account and no network?** The game today boots
from `Assets/Scenes/Bootstrap.unity` with a single `GameRoot` GameObject (scene line 133), and both
`AutoPilot` and the PlayMode boot test start the game directly. Under the shell, Yarin gates "the
game" on `onboardingComplete`. *Recommendation:* keep a `GameRoot`-only entry path (a scripting define
or a `SceneWiring` toggle) that bypasses the shell entirely, so the 333-test + AutoPilot verification
ladder in CLAUDE.md survives and does not depend on a backend being reachable. The owner should
confirm that is a sanctioned development path rather than a hidden bypass.

**B4 — logout cannot be implemented today.** Brief §9 lists it in Settings; there is no bridge method
to clear stored tokens (C2). The owner needs to know Settings ships without logout, or that the
feature waits on Yarin.

**B5 — scope of the frozen guild surface.** Brief §9 says `Roster`, `Guild` weekly-credits and `Game`
campaign screens were "designed as UI shape only" and the guild world-boss backend "is **frozen and
may not ship**". *Recommendation:* build none of the frozen surface; only the live routes
(create / join / invite / members) plus a roster fed by `/state`. Owner to confirm we are not on the
hook for world-boss UI.

**B6 — the workout→game hook stays undecided.** Brief §9 and `VERIFY-WITH-OWNER.md:85` both say so.
*Recommendation:* the owner should **not** decide it now. Phase 7's seam is what makes deferring it
cheap, which is exactly what Yarin asks for. Listed here only so the deferral is visible rather than
forgotten.

### (c) Needs Yarin

**C1 — `POST /devices` is completely undocumented.** Required by brief §3's "known dead path, and your
job to close" and by the bridge contract's `OnPushTokenReceived` row. The REST contract claims to be
"every HTTP call the Unity client makes" and has no `/devices` section: no body, no field names, no
response, no errors. *Blocks:* the one item he explicitly flagged as the thing not to skip.
*Proposed default to run past him:* `POST /devices` `{ "fcmToken": "…", "platform": "android" }`,
authed, any 2xx = success, any error = silent degradation. **Do not ship a guess** — a wrong field
name leaves push broken while looking implemented, which is strictly worse than the current dead path.
Implement both receiver methods regardless; omitting them is what caused the original bug.

**C2 — there is no way to clear stored auth tokens.** `HostBridge` exposes `storeAuthTokens` and
`readAuthToken` and nothing else. Logout, account switching, and "the refresh token is permanently
invalid" all need a clear. *Proposed default:* `storeAuthTokens("", "")` — but that is a guess about
host behaviour with empty strings, and if `readAuthToken()` then returns `""` rather than `null` every
request sends `Bearer ` and 401s forever, with the app unable to tell it is logged out. **Needs a
`clearAuthTokens()` on the frozen interface, or an explicit statement of the sentinel.**

**C3 — how does the client know a 401 "survived the host's silent refresh"?** Brief §6 says re-login
only on such a 401. But `readAuthToken()` is the only observable: no force-refresh, no expiry, no way
to see whether a refresh was attempted. *Proposed default:* on a 401, call `readAuthToken()` again; if
the string **differs** from the one we sent, retry once; if identical, prompt re-login. **This only
works if the host refreshes lazily inside `readAuthToken()`** rather than on a background timer.
Confirm which. Load-bearing in both directions: too eager and we hammer the backend; too keen on
re-login and we throw a Credential Manager sheet at every transient 401 — which the brief itself warns
gets throttled after repeated dismissals and then "presents as sign-in silently failing with no error
anywhere".

**C4 — does the host park an OAuth `code` on a cold start?** Invite codes get an explicit
park-and-pull design (`consumePendingInviteCode`) precisely because a cold start has no Unity to
receive a `UnitySendMessage`. The browser-PKCE redirect has the **same** cold-start shape — Android can
kill Unity while the system browser is foreground — but `OnAuthCodeReceived` is push-only, with no
`consumePendingAuthCode`. If it is not parked, the browser fallback is unreliable by construction on
exactly the low-end devices that lack native sign-in. *Proposed default:* assume push-only, persist
our verifier (A8), and accept that a process death during login means tapping sign-in again.

**C5 — which `source` does a gym session upload as, and is `tracked_gym_session` client-uploadable at
all?** A direct self-contradiction across the two documents:
- brief §5's `source` enum includes `tracked_gym_session` and `tracked_run`, and the REST contract has
  a whole "Tracked runs" section describing `tracked_run` uploads;
- yet REST `POST /activities` says "**Client-attested sources only** (`health_connect`, and `manual`
  when enabled)";
- REST §Gyms says a "**manual** workout claim" is what gets corroborated against registered gyms,
  implying the gym flow uploads `source: "manual"`;
- REST §Enums says "for a `tracked_gym_session` the claimed type is a hint only", implying it does not.

*Blocks:* the entire gym-session flow. The wrong `source` is either a `403 manual_logging_not_enabled`
or a silent trust penalty, and we cannot tell which by testing. *Proposed default:* **do not build the
gym flow until answered.** It is Phase 6, so this costs nothing today.

**C6 — can `requestHealthRead` gain a terminator?** (pairs with A1) `OnHealthReadComplete` with a
count, or a count on the first message. Cheap on his side; removes a timing heuristic from ours.

**C7 — can `OnHealthReadError` carry a machine-readable code?** (pairs with A2) A prefix would do:
`permission_denied:`, `no_provider:`, `empty_window:`. Without it we cannot show a permission
affordance without string-matching prose.

**C8 — what is `manualEntryFlag` when Health Connect does not supply one?** (pairs with A9) It is a
trust signal, so §0 says the client must not choose. Needs an explicit rule.

**C9 — the optionality of every field written `X | null`.** Zod's `.optional()`, `.nullable()` and
`.nullish()` are three different contracts and the docs use one notation for all three. Specifically
on write: `sourceWorkoutId`, `originPackage`, `gpsContext`, every `metrics.*`. And on read: `guild`
is documented as nullable, yet a *present* `guild` object has `"guildId": string|null` — when would a
present guild have a null id? The contract says these shapes are generated from code, so pasting the
Zod would settle all of it in one message.

**C10 — `GET /state` cannot feed the Home screen the brief describes.** §9: "Home carries the
**reward-framed weekly target** and the claim bar"; §9 also mentions "Guild **weekly-credits**"; and
the REST guild-members note references a "**miss-penalty economy**". No documented endpoint or field
exposes a weekly target, credit, streak, or penalty. Since §0 forbids computing a displayed number,
**Home as described cannot be built.** *Blocks:* Phase 3's headline screen. *Proposed default:* ship
Home with character + pending + claim only, and request the field.

**C11 — brief §2 contradicts itself about config, in consecutive code blocks.** `configure()` requires
`googleWebClientId`; the `appconfig.json` shape immediately below lists `backendBaseUrl`,
`supabaseUrl`, `supabasePublishableKey` and `oauthRedirect` — **no `googleWebClientId`**. Separately,
that JSON carries `backendBaseUrl` while the REST contract says the base URL is "injected at build
time (`BACKEND_BASE_URL`)" — two mechanisms for one value. *Blocks:* we cannot write the config loader
without knowing which keys are authoritative. *Proposed default:* one `appconfig.json` with all five
keys, and treat "injected at build time" as satisfied by that file. Needs confirmation **plus the live
values** — nothing in Phase 2 can be device-tested without them.

**C12 — `oauthRedirect` cannot actually be asserted.** §2 says it is listed "so you can assert they
match" the plugin's compiled-in `OAUTH_REDIRECT`, but nothing on the bridge exposes the plugin's
value, and §7 notes Unity plugin manifests have no `manifestPlaceholders`, so the manifest literal is
also unreadable at runtime. As written the assertion is a comment, not a check. *Proposed:* add a
`UnityBridge.oauthRedirect()` static so the mismatch is detectable at boot instead of at the one
moment a user tries the browser fallback.

**C13 — what is `requiresManualFlag` for?** (pairs with A12)

**C14 — `stopRunSession` with no active session "delivers nothing".** Any UI awaiting
`OnRunSessionEnded` therefore hangs — precisely the soft-lock CLAUDE.md's timeout invariant was
written for (`QteView` froze a fight this way, with no exception and nothing in the log). We will time
out regardless. *Proposed:* the host delivers an empty or error result instead, so both sides agree on
"always exactly one reply per request" — which is a much easier contract to reason about than
"sometimes zero".

**C15 — `permissionStatus() == DENIED` has no remedy path.** There is deliberately no
`requestPermissions`, and the host shows the dialog on `requestHealthRead` — but Android caps
re-prompts, so on a hard denial that dialog never appears again and `requestHealthRead` becomes a
no-op that reports an unstructured error forever. Does the host fall back to opening Health Connect's
own settings screen? If not, we must deep-link it, which is a host concern (it needs an Activity).

**C16 — rejected and ineligible have no specified user-facing meaning.** The contract says "display
the rejection in-narrative" and nothing more. We cannot compute a reason, so all we can echo is
`trust.signals[].checker` / `reason`, which are backend identifiers (`manual-entry`,
`geofence-negative`). *Proposed default:* one generic narrative line, no checker names on screen.
Ask for a user-safe message field or a documented checker→copy mapping.

**C17 — no rate or refresh policy.** "Fetch `/state` on launch and on refresh"; `POST /strava/sync`
"on launch and on manual refresh". With an `OnApplicationFocus` invite drain already prescribed, app
resume is an obvious extra refresh point. Ask for a minimum interval so we do not get throttled.

**C18 — four documents were cited but not delivered.** This was asked when none of them existed here.
Two arrived afterwards (`screen-flow.mermaid`, `unity-plugin-topology.md` — the latter since filed in
the knowledge base as `departments/engineering/docs/unity-plugin-topology-decision.md`), and `PLAN-closeout.md` was
delivered and then deliberately dropped again — so do not go looking for it. The column below is kept
for **why each one mattered**, which is what a future reader needs; it is not a statement about what
`docs/` contains today.

| Document | Cited for | What its absence cost |
|---|---|---|
| `docs/screen-flow.mermaid` | §9 calls it "the app-shell flow"; the brief gives one paragraph | **The most damaging omission.** Screen inventory, back-navigation, where Settings hangs, what the tab bar contains — and, critically, **orientation and layout** (→ B1). Affects all of Phases 3–6. |
| `docs/PLAN-closeout.md` | "item 5e", the push dead path | Whether `/devices` (C1) is already specified there, and what other known-open items exist. |
| `docs/unity-plugin-topology.md` | Twice: why `onCreate` stays host-owned; and the `activity-alias` requirement tied to `targetSdk >= 34` | §8 says "the manifest carries an `activity-alias` requirement tied to `targetSdk >= 34`" — but **the manifest fragment in §7 declares no `activity-alias`**. Without this doc we cannot tell whether our manifest is incomplete, and this is one of the failures §8 says fails *quietly*. |
| `docs/TODO.md` | Why the `id_token` grant cannot serve Sign in with Apple | Informational only, unless iOS enters scope. |

**Send C1, C2, C3 and C11 today.** All four sit on the critical path of Phases 1–3, and each has a
multi-day tail if answered late.

---

## 6. Effort and recommended order

Ideal engineer-days, assuming the fake bridge (so nothing waits on the `.aar` except Phase 8), and
assuming the C-list answers arrive before the phase that needs them.

**These are the original estimates for the whole job, not what is left.** Do not read the Days column
as remaining work — see [#247](https://github.com/Get-Sweaty-Games/reign-and-gain-unity/issues/247)
for that.

| Phase | Work | Days | Blocked on |
|---|---|---|---|
| 0 | Prerequisites, settings, config loader | 0.5 | — |
| 7 | Run-settlement seam + reflection guards | 1.5 | — |
| 1 | Bridge, fake, receiver, name test | 2.5 | — (unverifiable on device) |
| 2 | Auth: PKCE, both exchanges, handoff, 401 policy | 2.5 | C3, C11 for real values |
| 3 | REST client + all DTOs + `AppFlow` + `AppRoot` + Home | 4 | **B1 before any screen**; C10 for full Home |
| 4 | Source picker + connected-sources strip | 1.5 | — |
| 5 | Assembler + upload queue + sync flow + claim | 3 | A9/C8 |
| 6 | Gyms 1 · manual 1 · tracked run 1 · guild 1.5 · roster 0.5 · push 0.5 · Strava 0.5 | 6 | gym session C5; push C1 |
| 8 | Manifest + gradle templates + first device build | 1 to author, **1–3 more** on the first real build (Gradle resolution, manifest merge, IL2CPP stripping) | the `.aar` |
| — | Shell art pass to `RgSkin`/`ProcTex` standard (`AppIcons`, panels, backdrops) | 2 | deliberately last |

**≈ 25 ideal-days** of in-scope unblocked work, plus Phase 8's tail.

### Recommended order — risk first, cosmetics last

1. **Phase 0.**
2. **Phase 7 — the settlement seam, before any new screen.** Out of Yarin's numbering deliberately.
   It is a refactor of code covered by 333 tests and a 12-row pinned baseline; doing it while the
   surface area is small is far cheaper than after Phases 3–6 depend on `RunController`'s signature.
   It also settles the §0 question before anyone writes their first reward line, which is when the
   wrong pattern gets copied.
3. **Phase 1, then Phase 2.** Yarin's §10 calls these the integration risk, and they are also the two
   that are only *fake*-verifiable until the `.aar` lands. Start them early and **log the exact JNI
   call sequence and every receiver payload**, so the first device session is a verification rather
   than an exploration.
4. **Phase 3 (minus the weekly target) → 4 → 5.** Get read → upload → claim green. This is Yarin's
   "get it green before anything cosmetic" and it should be the first thing demonstrated end to end.
5. **Phase 8 the moment the `.aar` arrives** — do not wait for Phase 6. The first device build will
   surface Gradle and manifest problems whose fix time is unbounded, and finding them earlier is worth
   more than finishing the guild screen.
6. **Phase 6**, then the shell art pass.

### Verification, per CLAUDE.md's ladder

- **EditMode tests** for everything pure: host payload parsing (including every "field absent" case),
  `EvidenceBundleAssembler`, `IsoTime`, the `AppFlow` transition table, DTO round-trips, the
  `HostBridgeReceiver` name/signature reflection test, and the two `Net` boundary guards.
- **`FakeHostBridge` + AutoPilot** for the shell, with the fake's failure toggles exercised.
- **AutoPilot must stay green at every step** — which requires B3 (a shell bypass) and A13 (the
  exclusion-list fix) to land with the first shell screen, not after it.
- **A written "first device session" checklist** for Phase 8, in the order §7 warns about:
  `.aar` present → `configure()` succeeds → `get()` non-null → `requestTodaySteps` →
  `OnTodayStepsRead` fires. Yarin's §10.1 is explicit that this is the proof to get first, and it is
  also the cheapest thing to do wrong.

---

## 7. Things this plan deliberately does not do

- It does not invent the server-resolved run endpoint, or the workout→game hook. Both are named as
  undecided by Yarin (§0 corollary, §9) and by `VERIFY-WITH-OWNER.md:85`.
- It does not build the frozen guild world-boss surface, or the `Roster`/`Game` campaign screens that
  §9 describes as "UI shape only".
- It does not populate `sleep` in any bundle. Brief §5 is explicit that uploading one requires the
  privacy policy and Data Safety declaration to flip in the same release, and that is Yarin's.
- It does not send `providerFlagged`, and it makes a `source: "strava"` bundle unconstructible.
- It does not guess `/devices`' body (C1), the gym-session source (C5), or the logout sentinel (C2).
