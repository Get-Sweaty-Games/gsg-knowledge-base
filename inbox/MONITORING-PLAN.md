# Monitoring — Reign & Gain

Cross-cutting: touches `reign-and-gain-unity`, `reign-and-gain-backend`, `gsg-landingpage`, Play
Console.

**Status: Phases 1–3 done. Phase 4 blocked.** Crash reporting (Sentry) was adopted separately and
supersedes this plan's earlier "not needed before there are players" call — see
`SENTRY-ALPHA-PLAN.md` in this folder for that work. Session replay (PostHog) was also adopted
separately, on 2026-08-18, narrower than the blanket "not built" claim this plan makes below — see the
correction directly under the "Deliberately not built" table, and the update to the vendor-onboarding
checklist that follows it. Neither correction is a deletion of the original claim: both are still
substantially true and are marked where they are not.

## Done

**Phase 1 — backend logs the error code of every failed request.** Merged, deployed, verified against
production.
- A Fastify `onSend` hook logs the route **pattern** (never the raw URL — `/gyms/nearby` carries
  lat/lng in its query string), status, error code, and user id (JWT sub only; `anon` for
  unauthenticated routes).
- Routed 404s are logged (real product signal); 401/429 are not (framework noise + flood protection
  for unauthenticated callers).
- Backend PRs: #48 (the hook), #56 (fixed pino's *own* default logging of the raw resolved URL,
  including query string — a leak the hook itself never had), #58 (a 5xx no longer echoes the raw
  driver error message to the client; it now logs it server-side instead).
- Cron (`runTick.ts`/`dailyTick.ts`) now exits 1 on a failed sweep instead of always reporting
  success.
- `process.on('unhandledRejection'|'uncaughtException')` added — nothing caught these before.

**Phase 2 — client failure messages are self-describing.** Merged (`da55c49`, PR #362).
- `ApiFailureCopy.TryCopyFor` covers the `malformed_body` and `"unknown"` codes, and an optional
  `httpStatus` parameter adds a status-keyed 5xx message read before the per-code switch.
- `GymPage`, `GuildPage`, `ManualEntryPanel`, `NameEntryPanel` still show a generic 5xx message —
  filed as unity #369 if per-page 5xx copy is ever wanted.

**Phase 3 — native symbol level pinned for Android.** Merged (`95c8142`, PR #374).
- `DemoBuild.PinNativeDebugSymbols`: `DebugSymbols.level = Full` on release / `None` on sideload,
  `.format = Zip | LegacyExtensions` (the correct bitmask — a bare `Zip` silently renames every symbol
  file inside the archive).
- Prerequisite for **Google Play** Vitals symbolication only — Sentry's own symbol upload doesn't read
  this archive, it uploads from the IL2CPP staging directory directly.
- The archive-exists check in `build-android.sh` reports rather than hard-fails (filename was
  unconfirmed at the time) — tracked as unity #373 to tighten once a real release build confirms the
  name.

## Not done

**Phase 4 — upload symbols to Google Play.** Blocked on the Foreground Service declaration
(issue #322 / #363), which 403s `edits:commit` on every track including internal. No engineering
workaround — the permissions come from `hostbridge.aar`, built in the backend repo.

**Open questions, still unanswered:**
- Which Render plan (Hobby vs paid)? Log retention (7 days assumed) depends on this — check the
  dashboard.
- Who actually reads these logs, and how often? If the answer is "nobody," Better Stack heartbeats fit
  better than a log-only design.

## Deliberately not built

| Not built | Why | Revisit when |
|---|---|---|
| PostHog (client or server) | Play forbids sending fitness properties to third-party analytics; product events already live in Supabase `analytics_events`. | Real traffic exists and a behaviour question a SQL view can't answer comes up. |
| Firebase Crashlytics | Not wired — `google-services.json` is gitignored, and `launcherTemplate.gradle` only applies the FCM plugin, not Crashlytics. | Never — Sentry is adopted instead. |
| A dedicated `analytics_events` row per API failure | 7-day Render log retention is enough at current scale. | An incident older than 7 days needs reconstructing, or the same aggregate query gets typed twice. |
| A log drain (Better Stack, Axiom, Grafana) | Second dashboard for a solo owner; Render already retains 7 days. | The 7-day window loses a real incident. |
| An uptime monitor | Render already scrapes `/health` and restarts on failure. | Launch day — UptimeRobot free tier, five minutes of setup. |
| Alerts of any kind | Nobody is on call; an unwatched alert is worse than none. | There's a player whose bad hour would otherwise be missed. |
| Client-minted `x-request-id` | Nobody reads a guid out of a support message; the visible status code does the same job. | A support channel or "copy diagnostics" button exists. |
| Render Pro ($25/mo) | Buys latency percentiles nobody is asking about. | Never, on current evidence. |

**Correction, 2026-08-18: the PostHog row above is now partly out of date.** PostHog was added to the
Unity client (`reign-and-gain-unity/Assets/Scripts/Bootstrap/PostHogBootstrap.cs`,
`SessionReplayConsent.cs`), but narrowly — **session replay only, watching how new alpha testers play
their first sessions, never product analytics.** The row's underlying claim is unaffected for the half
that matters: product analytics is still Postgres-native, unchanged, and PostHog carries no analytics
events, no feature flags and no application-lifecycle events (`CaptureApplicationLifecycleEvents` and
the other product-analytics-shaped flags are pinned off in code, on purpose, so PostHog cannot quietly
become the analytics vendor this plan rejected). Recording is off on the two screens that render
fitness and location data — Workouts and Gym — because PostHog's Unity SDK has no masking API, so not
filming those screens is the only available control; `AppRoot.ShowPage` enforces it on every page
change. The distinction this plan drew (fitness data to a third party vs. product analytics in
Supabase) still holds; what changed is that a video of on-screen play is now a third category the plan
did not previously name.

## If a client-side vendor is ever added (PostHog, etc.)

**This happened on 2026-08-18** — see the correction above. The checklist was written before that
landing and has now been walked item by item against it. Status is recorded per item rather than in a
summary line, because the two that remain open are the two that matter.

Not optional paperwork — all of this before the first event leaves the device:
- **DONE.** Signed DPA naming the vendor as processor. PostHog's is self-serve: generate, e-sign and
  download a countersigned copy at `app.posthog.com/legal`. The text published at `posthog.com/dpa` is
  a non-binding preview and is NOT the agreement — only the in-app generated copy is.
- **DONE.** Sub-processor list update in `PrivacyPage.tsx` — a different repo with its own deploy, can't
  land atomically with the code change. PostHog was added to "Who we share it with", and the count in
  that paragraph went from two providers to three; the count is the part a reader notices is wrong.
- **DONE.** The `PrivacyPage.tsx` analytics-SDK sentence, which used to forbid exactly this.
- **NOT BUILT, deliberately, and the reasoning is time-boxed.** An account-deletion call to the vendor —
  Supabase's delete cascade doesn't reach a third party. PostHog CAN delete a person with
  `delete_recordings=true`, or one recording by id, so the pipeline is buildable: the client can read
  `PostHog.DistinctId`, the backend could store it and call the API inside `verifyAndDelete`. It was not
  built because **the free plan caps recording retention at 30 days** and the cohort is 12-50 people, so
  a manual erasure route (email in, find by nickname and date, delete) covers the case for less work
  than the pipeline. **Both halves of that argument are conditional.** A larger cohort makes "find it by
  hand" false; leaving the free plan raises retention to 90 days or a year and makes the published
  policy text wrong. Retention changes are NOT retroactive either, so recordings keep the window they
  were captured under. Revisit on either trigger.
- **DONE.** A Play Data Safety amendment — App activity → App interactions, declared as both
  collected AND shared, since Google's own definition of that category names screenshots. It was
  deliberately deferred while the closed-testing release sat in Google's review queue, because filing a
  change during a pending review can push the app to the back of it. Filed on 2026-08-18, after that
  release published and the queue was empty.
- **DONE.** Disable log-based event capture explicitly, or the free tier dies in week one (53
  `Debug.LogError` sites in the client). `CaptureLogs` is pinned off in `PostHogBootstrap`, alongside
  three flags that default ON and would each have undone a decision already made — `CaptureExceptions`
  most of all, which is not gated by `CrashReportingConsent` and would have kept reporting a player who
  had switched crash reporting off.

**One item nobody put on this list, and it applies to every future vendor:** set the retention period
explicitly rather than accepting the project default. The privacy policy now states a 30-day figure, so
that setting is load-bearing text rather than a preference.
