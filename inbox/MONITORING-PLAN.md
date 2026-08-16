# Monitoring — Reign & Gain

Cross-cutting: touches `reign-and-gain-unity`, `reign-and-gain-backend`, `gsg-landingpage`, Play
Console.

**Status: Phases 1–3 done. Phase 4 blocked.** Crash reporting (Sentry) was adopted separately and
supersedes this plan's earlier "not needed before there are players" call — see
`SENTRY-ALPHA-PLAN.md` in this folder for that work.

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

## If a client-side vendor is ever added (PostHog, etc.)

Not optional paperwork — all of this before the first event leaves the device:
- Signed DPA naming the vendor as processor.
- Sub-processor list update in `PrivacyPage.tsx` — a different repo with its own deploy, can't land
  atomically with the code change.
- The `PrivacyPage.tsx` analytics-SDK sentence, which currently forbids exactly this.
- An account-deletion call to the vendor — Supabase's delete cascade doesn't reach a third party.
- A Play Data Safety amendment (crash logs / diagnostics / device IDs) — verify the actual field names
  on a real device event before filing, don't assume.
- Disable log-based event capture explicitly, or the free tier dies in week one (53
  `Debug.LogError` sites in the client).
