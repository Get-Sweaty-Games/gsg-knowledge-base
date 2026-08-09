# Integration Decisions — Unity ↔ Backend (backend/API side)

> Filed from inbox `integration-decisions-unity-backend.md` (frozen 2026-08-08 decision record).
> This doc carries the backend/API/wire-protocol portions only. Owner-level product decisions
> B1–B6 (orientation, gold naming, run loop, guild scope, workout hook) are filed separately in
> `departments/game-design/docs/unity-integration-owner-decisions.md`.
>
> **This is a decision record, not a status document.** It records why choices were made during
> the Unity↔backend integration, and what the partner studio answered. It says nothing about what
> is built, done, or outstanding — for that, read the Unity repo's issue #247 or the code itself.
> **Do not add status to this file.**

## Provenance, and why this file was frozen

This was `docs/INTEGRATION-ROADMAP.md` in `Get-Sweaty-Games/reign-and-gain-unity`, dated 2026-07-30.
It reconciled three documents that were drifting apart, and it succeeded at that. It then failed in a
new way.

The predecessor wrote per-phase status in **five** separate sections of itself. Two more copies lived
outside it: `docs/BACKEND-INTEGRATION-PLAN.md` carried its own `BUILT`/`MISSING` tables, and the
tracking issues restated phase status in their bodies. Seven sources for one fact.

They disagreed. A cleanup pass on 2026-08-07 corrected six contradictions and introduced or missed
three more within a day: two issues it listed as open had closed, a third had closed six days earlier,
and its own count of inbound links was wrong. Line-number citations from the sibling plan into it had
rotted by twenty-four lines.

The lesson is the same one its own postmortem drew (§5 below), one level up: **a document that
restates a fact owned elsewhere will drift, and adding a section to keep it current adds a source
rather than removing one.** So the status is deleted rather than corrected, and what remains is the
part that cannot go stale — decisions already made, with their reasons.

Sibling material, single copies, do not duplicate:

- `departments/engineering/docs/unity-partner-qa-record.md` (this repo) — the partner's four answer
  batches covering C1–C18 plus B1. The **only** copy. §1's verdicts cite into it.
- `docs/BACKEND-INTEGRATION-PLAN.md` (Unity repo) — the file-by-file plan. Its §5 carries the original
  question list and its §6 the effort estimates.

---

## 1. What the vertical slice cut, and what became of each cut

The approved vertical slice was Phases 0–4 of the full plan with named pieces cut. The full plan's
Phases 5–8 were never replaced; they picked up afterwards, unmodified in scope.

Every cut below was later reversed or reduced. The table is kept because **each reversal has a reason
worth not re-deriving**, not to state what exists.

| Cut from the slice | What the cut became, and why |
|---|---|
| `com.unity.nuget.newtonsoft-json` — use `JsonUtility` instead | **Reversed.** The DTO layer needs field-omission semantics (`NullValueHandling.Ignore`) that `JsonUtility` cannot express, and `POST /runs/settle` is `additionalProperties:false`, so an emitted null is a 400 |
| The evidence bundle / workout→claim loop | **Reversed**, minus `ActivityUploadQueue`, which stays deferred on purpose — see §4 |
| Gyms, guild create/join/invite/members, push, Strava | **Reversed at materially reduced scope**, per C5 and C1 — see §2 |
| The run-settlement seam (`IRunSettlement`) | **Reversed early**, per the full plan's §6 recommendation, while `RunController`'s surface area was still small |
| The browser-PKCE fallback | **Reversed** once native sign-in shipped and the host's `consumePendingAuthCode` turned out to already exist |

What the slice **did** keep from later material: read-only `RosterView`/`GuildView` rendering straight
off `GET /state`, with no guild actions — cheap, and it makes all four shell tabs honest rather than
three tabs and a stub.

---

## 2. Partner answers — C1 to C18

One row per item. Full reasoning sits at the cited location in
`departments/engineering/docs/unity-partner-qa-record.md`.

| # | Question (short) | Verdict | Where |
|---|---|---|---|
| C1 | `POST /devices` body undocumented | Field is `token`, **not** `fcmToken` as guessed. An `fcmToken` body 400s | batch 1, `## C1 — POST /devices` |
| C2 | No way to clear stored auth tokens | `clearAuthTokens()` added to the host bridge | batch 1, `## C2` |
| C3 | Telling a 401 survived silent refresh | Refresh is lazy inside `readAuthToken()`; compare-and-retry-once is the correct rule | batch 1, `## C3` |
| C4 | Does the host park an OAuth `code` on cold start? | **Real bug** — see callouts | batch 4, `## C4` |
| C5 | Which `source` does a gym-session upload use? | **None. The client uploads nothing for a gym session, ever** — see callouts | batch 3, `## C5` |
| C6 | Terminator for `requestHealthRead` | `OnHealthReadComplete(count)` — fires exactly once, always last, on every path | batch 4, `## C6` |
| C7 | Machine-readable `OnHealthReadError` code | Prefixes as proposed. Exposed a second, separate launch-crash bug — see callouts | batch 4, `## C7` |
| C8 | `manualEntryFlag` when Health Connect supplies none | **Never author it for `health_connect`.** The host computes and hands it over; unsupplied resolves `false`. A wrong default silently destroys every upload, or defeats the best cheat filter available | batch 3, `## C8` |
| C9 | Optionality of every `X \| null` field | `.optional()` ≠ `.nullable()`; a default Json.NET null 400s the whole bundle. The server now strips nulls before validating | batch 3, `## C9` |
| C10 | `GET /state` cannot feed Home as briefed | Ship narrower — see callouts | batch 3, `## C10` |
| C11 | Which config keys are authoritative | One `appconfig.json`, five keys including `googleWebClientId` | batch 1, `## C11` |
| C12 | `oauthRedirect` cannot be asserted at runtime | `UnityBridge.oauthRedirect()` static, as proposed. Adopted by the boot asserts — §3 | batch 4, `## C12` |
| C13 | What is `requiresManualFlag` for? | Nothing actionable. The server pre-filters the catalogue; **never map it to a bundle field** | batch 3, `## C13` |
| C14 | `stopRunSession` with no session delivers nothing | Always exactly one reply — `OnRunSessionEnded` or `OnRunSessionError`, never an empty `OnRunSessionEnded`, deliberately | batch 4, `## C14` |
| C15 | `permissionStatus() == DENIED` has no remedy | `UnityBridge.openHealthConnectSettings()`; gate the affordance on its `bool` return | batch 4, `## C15` |
| C16 | Rejected/ineligible have no user-facing meaning | Checker→copy mapping given, four hard-reject checkers named. `ineligible_workout` **cannot fire today** — the rule catalogue is empty | batch 3, `## C16` |
| C17 | No rate or refresh policy | 120/min global, per-route per-user limits listed. `GET /state` has no per-user cap but **must never be polled** — event-driven only | batch 3, `## C17` |
| C18 | Manifest fragment missing HC permission entries | **Three defects, not one** — see callouts | batch 2, `## C18` |

### Callouts — real bugs, not doc gaps

- **C4 — a cold-start OAuth code was silently dropped.** `UnityHostActivity.handleDeepLink` parked
  invite codes for a late-listening Unity but pushed the auth code immediately, so a cold-start auth
  deep link vanished with nothing in the log. The code is now always parked and drained via
  `consumePendingAuthCode()`. The push-only `OnAuthCodeReceived` was **removed** from the host, not
  deprecated — which is what makes the race in §4's "pulled, never pushed" decision unrepeatable.
- **C7 — `no_provider:` could never have fired.** `HealthConnectReader`'s constructor called
  `HealthConnectClient.getOrCreate(context)` as a default argument, which throws when no Health
  Connect provider is installed. The app crashed at launch on a provider-less device, before any read
  could classify anything as `no_provider:`. Fixed with `by lazy`.
- **C5 — the route accepted client-uploaded `tracked_gym_session`.** The gym flow was redesigned
  server-side to passive location pings with server-side synthesis, but `POST /activities` never
  enforced it: only `strava` sat on the pull-only rejection list, so a client-built bundle would have
  been silently accepted, bypassing ping-grouping entirely. Both server-ingest-only sources now `403`
  with `server_ingest_only_source` — this also **renamed** the error `strava` used to return, from
  `pull_only_source`.
- **C18 — three manifest defects.** A missing `activity-alias` for the Android 14+
  `VIEW_PERMISSION_USAGE` route, unconditional since targetSdk is pinned to 36; a missing
  `ACTION_SHOW_PERMISSIONS_RATIONALE` filter, the Android-13-and-below half of the same mechanism —
  together meaning **the fragment worked on no Android version at all**; and a `pathPrefix` where the
  real manifest uses an exact `path` on the invite App Link, which would have matched web-only sibling
  paths.

### Callouts — answers that remove work

- **C10 — no streak, ever; penalty frozen; guild-boss fields permanently null.** The miss rule is a
  weekly target, not a daily streak, so no streak field will ever exist — **do not build a
  placeholder**. The miss-penalty mechanic runs server-side but is dormant by design and belongs in no
  client screen. `guild.bossName` / `bossHpRemaining` / `bossHpMax` are hardcoded `null` at the route;
  the world-boss feature is frozen and may not ship.
- **C5 — no client-side gym-upload path at all.** The `GymSessionView` upload flow the full plan
  describes has nothing server-side to build against. The client's entire gym surface is registering a
  venue once (`POST /gyms`) and pinging on app-open (`POST /gyms/ping`). Grouping pings into a session
  and synthesizing the activity is entirely server-side (`GymSessionFinalizer`).

---

## 3. The two boot assertions, and why they behave differently

C12's redirect check and the `hostVersion()` check were each documented in three places, implemented in
none, and belonged to no phase in any table. Both decisions are pure statics taking their inputs as
parameters, so the logic is EditMode-testable with no JNI and no Play Mode.

**They behave differently on purpose:**

- **Redirect → hard fail.** A scheme mismatch's only production symptom is a redirect that never comes
  back, discovered by a real user mid-login. But be honest about the mechanism: **a throw from a
  `[RuntimeInitializeOnLoadMethod]` is caught by Unity, logged, and the app carries on loading.** The
  throw alone stops nothing. `HostBootstrap.RedirectMismatch` is what has teeth, because `SignInView`
  reads it and builds no sign-in button at all. **Do not delete either half as redundant.**
- **Host version → loud `Debug.LogError`, never a throw.** A hard throw would make a `const` in the
  Unity repo a gate on the *backend* repo's release cycle: every future `.aar` delivery would be a dead
  app until somebody bumped a string, and whoever receives the `.aar` usually cannot. Nothing in the
  auth flow depends on the version. It is a diagnosis aid.

**Both predicates return `false` for a null or empty plugin value, and that guard is load-bearing.** In
the Editor the JNI class is unbound, so both getters return null on **every Play-Mode entry**, and
Unity's Test Framework fails a test on any unexpected error log. An unguarded assert turns every
PlayMode test red.

**What the redirect check does not prove:** that the redirect is in Supabase's allowed-redirect list.
Neither side can read that at runtime.

---

## 4. Decisions taken during implementation

Salvaged from the predecessor's phase table, which mixed these into a status column. They are the
reasons a future reader would otherwise have to re-derive from a diff.

### Auth

- **`HttpResponse.Status` is `int?`, and that is load-bearing.** Null means no HTTP answer arrived.
  That is what gates C3's compare-and-re-login onto a *real* 401. The host returns the stale token
  unchanged when its refresh fails offline, so the naive rule prompts sign-in on a network blip — and
  Credential Manager throttles repeated dismissals into silent permanent failure.
- **`AuthTokenProvider` was deliberately not built.** It would wrap one existing host call and have
  zero callers until the REST client existed, and the repo already carried six symbols shipped with no
  consumer. The *policy* is the deliverable and ships as a pure function; the provider ships with its
  first caller.
- **The browser fallback escalates on any native failure except `cancelled`.** Credential Manager
  throttles repeated prompting, so a browser thrown at someone who just dismissed the sheet is that
  same re-prompt by another route.
- **The `auth_code` / `code_verifier` grant body is confirmed against live GoTrue.** A bogus code under
  `auth_code` reaches flow-state lookup and answers `404 flow_state_not_found`, while the rival
  candidate `code` answers byte-identically to an empty `{}` body. **The empty-body control is what
  makes it proof rather than an absence of complaint.**
- **The code is pulled on focus, never pushed.** `OnAuthCodeReceived` was deleted from the wire so two
  paths could not race a single-use code. `Application.deepLinkActivated` would resurrect that race
  under a new name.
- **`Pkce` uses `RandomNumberGenerator` — the one deliberate exception to the `SeededRng` invariant.**
  A seed-reproducible verifier is not a secret. Recorded in the Unity repo's `docs/ARCHITECTURE.md`
  Determinism section so it does not get "fixed".
- **Owner decisions, not judgement calls:** the verifier goes to the host keystore rather than
  `PlayerPrefs`, so a process death mid-login is recoverable instead of burning the code; it carries no
  expiry timer and is cleared on use; Google's mark sits on the existing `RgSkin` button rather than
  Google's own button design; and a redirect mismatch disables **only** the fallback, because native
  sign-in never used the redirect, so removing the button disabled the one path a mismatch cannot
  break.
- **The burned-code branch needs both copies gone** — the in-memory field is still what a surviving
  process spends.
- **iOS is not covered by any of it.** The three parking statics are Android JNI and `HostBridge.Resolve`
  hands iOS a `FakeHostBridge`. **Do not plan that work from a document** —
  `grep -rn "IOS-TODO" Assets/Scripts/` is the live list, recorded at the three lines that actually
  decide it so it cannot drift.

### Shell and REST

- **`TimeZoneInfo.TryConvertWindowsIdToIanaId` does not exist on this compile surface.** It arrived in
  .NET 6; the project is netstandard2.1. `DeviceTimezone` tests the IANA shape instead and sends
  nothing from the Editor.
- **Sign-out needed a whole `AppRoot.SignOut`, not a `ClearAuthTokens` call.** `AppFlow.Current` has no
  reset, and a stale `Hub` made the *next* sign-in refuse its own transition into a blank canvas.
- **`AutoPilot` gained a hub step** because it cannot press a button under an `AppSurface`.
- **`referralRecorded` on the guild join response is the product's entire referral surface.**
- **Nothing navigates between the shell pages, by design.** That is a game-design decision, tracked
  separately — see `departments/game-design/docs/unity-integration-owner-decisions.md`.
- A second, unclaimable ledger exists at `POST /rewards/claim`. It was found while widening the API
  surface; it is not the claim route the loop uses.

### Evidence and the claim loop

- `EvidenceBundleAssembler` is a pure static and **forwards, never improves** — tracked-run distance
  forcing, and the sleep dual-field rule.
- **`empty_window` is a success, not an error.**
- **Claimed gold feeds the current run** — owner-decided.
- **`ActivityUploadQueue` stays deferred.** Dedup is an in-memory `sourceWorkoutId → activityId` map,
  and the server's own `sourceWorkoutId` dedup is the backstop across process death.

### Run settlement

- **The result screen is shown *before* the settlement is submitted.** AutoPilot waits on that screen
  and would misreport a slow callback as a missing one.
- **`RunResultView.ShowSettlement` is the only path a server figure may reach the screen by.**
- **`StubCombatController`'s duplicated gold formula was deleted** — it had been quietly paying the
  *pre-rebalance* rate.
- **The request schema is `additionalProperties:false`.** One extra field 400s every run, permanently.
- **The endpoint credits nothing, deliberately.** The server records the run and answers
  `isSettled:false` with null xp and gold, because what a run is worth is undecided and re-scoring one
  needs a decision trace no client emits. A reward figure can no longer be computed client-side — the
  result screen has no local number to draw one from.

---

## 5. The wire-drift postmortem — the failure this document was created to end

Found and fixed 2026-07-30. Kept because the failure mode is the reason the predecessor existed, and
because the freeze at the top of this file is the same lesson one level up.

The C# wire committed in the Unity repo's PR #29 had been written against the **pre-batch-4** contract:

- It declared `OnAuthCodeReceived`, a method **removed** from the host per C4 — a dead path in C# for
  something that no longer existed on the wire.
- It omitted `OnHealthReadComplete` (C6) and `OnRunSessionError` (C14), methods the host **does** send,
  which the receiver would therefore drop silently: no matching method, no event raised, nothing
  logged.

Verified empirically rather than against the doc, by unpacking `hostbridge.aar` and binary-grepping its
class files.

**The reflection guard could not catch it.** `HostWireTests` diffs the receiver against
`HostWireNames.ReceiverMethods`, and both sides were written from the same stale list. **A test that
grades against its own stale constant passes cleanly while being wrong.**

**Root cause:** the Unity repo carried only batch 1 of the partner's answers, as a local extract.
Batches 2–4 — including C4, C6, C7 and C14, all of which change the wire — landed only in this repo's
`departments/engineering/docs/unity-partner-qa-record.md` and were never read by the session that
shipped the implementation. A later, complete answer sat in a sibling repo while the implementing work
used an earlier, partial copy.

The partial copy was deleted on 2026-07-31 after confirming its content byte-for-byte against batch 1
of the complete file, so the trap is disarmed rather than merely documented. It remains in git history
if the deletion ever needs auditing. **Do not re-add a local copy of partner answers to the Unity
repo** — point at this repo instead.
