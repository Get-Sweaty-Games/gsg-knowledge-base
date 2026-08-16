# Reign & Gain — internal to closed testing: one dated critical path

**Written 2026-08-11. Every date below is calendar 2026.**

Third of three planning docs. `MONITORING-PLAN.md` covers backend observability.
`SENTRY-ALPHA-PLAN.md` covers client crash reporting. This one covers the Play path.

Produced by a 10-agent workflow: three gate-research agents verified against live Play documentation,
two competing sequences, four adversarial critiques, one synthesis. An earlier run lost the critiques
and synthesis to a usage limit; the research was recovered from the journal and replayed from cache.

**Verified directly by the orchestrator** (these outrank agent claims):

| Claim | Evidence | Status |
|---|---|---|
| No `ACCESS_BACKGROUND_LOCATION` in the shipped AAR | unzipped `hostbridge.aar`, read `AndroidManifest.xml` | Confirmed — only `ACCESS_FINE`/`COARSE_LOCATION` |
| Seven Health Connect read permissions, incl. `READ_HEALTH_DATA_IN_BACKGROUND` | same manifest | Confirmed |
| Two services with `foregroundServiceType="location"` | same manifest: `RunSessionService`, `GymSessionService` | Confirmed |
| `versionCode` is 211 today | `git rev-list --count HEAD` | Confirmed |
| The web deletion page exists | `gsg-landingpage/app/src/App.tsx:58` → `DeleteAccountPage`, linked from `PrivacyPage.tsx:239` | Confirmed |
| **No in-app deletion path exists** | `SettingsPage.cs` has no delete row; the only `Application.OpenURL` in AppUI is `AppRoot.Activity.cs:363` | Confirmed — this is a real compliance gap |

---

## The answer, in four sentences

Closed testing can start around 18 August, but only if the Foreground Service permissions declaration is
answered today or tomorrow. That declaration is the one gate that decides the date: it fails at
`edits.commit` on **every** track including internal, so nothing publishes and no store-listing edit
commits until it is saved. Everything else on the path is owner work that fits into four days, plus two
Google queues that run in parallel underneath it. Earliest credible production is **~2026-09-18, and
realistically mid-October**, because the account check below came back as the slower of the two cases.

---

## Settled 2026-08-11: the 12-testers/14-days rule DOES apply

Google's rule is scoped in one sentence, verbatim: *"Developers with personal accounts created after
November 13, 2023, will need to test their apps before those apps are eligible to be published for
distribution on Google Play."*

The two exemptions are Organization accounts and personal accounts created on or before 2023-11-13.
**This account is neither.** The owner confirmed on 2026-08-11 that it is a **personal** account created
**about one month earlier, in July 2026**. The exact creation date was not needed: the rule turns on a
single boundary, and July 2026 clears 2023-11-13 by two and a half years.

So production access requires **12 testers opted in continuously for 14 days** of closed testing, and
Google then reviews the production application on top of that. Plan against the personal-account
scenario at the bottom of this document, not the Organization one.

The rule also gates **open testing and pre-registration**, not only production. It does not gate the
closed test itself.

---

## Dated critical path

### Owner actions — the only compressible part

| Date | Action | Time | Why it is here |
|---|---|---|---|
| ~~Tue 08-11~~ | ~~Read the Play account type and creation date.~~ **Done 08-11: personal, created July 2026. The 12/14 rule applies.** | — | Decided the production half. |
| Tue 08-11 | Open the closed-track creation flow far enough to list which **App setup** tasks Play shows as incomplete. Write the list down. Do not fix them yet. | 20 min | "App setup complete" is a prerequisite for the closed track and surfaces only here. |
| Tue 08-11 | On App content, open the **Health apps** form and read whether it submits on its own or states it applies to your next publishing request. | 15 min | Decides parallel vs serial. See "Deadlock check". |
| Tue 08-11 | Record two unlisted demo videos: `RunSessionService` and `GymSessionService`. Show a clearly user-initiated start. Show the persistent notification throughout. Record from the current internal build. | 4 h | The FGS form cannot be saved without a video link per service. |
| Tue 08-11 | Answer the **Foreground service permissions declaration**. Two entries, one per service. | 1 h | **This unblocks everything.** |
| Tue 08-11 | Send the tester invitation mail. Ask for 16-20 confirmed yeses and the **exact Google address** each will use. | 30 min | Longest-lead human item. |
| Wed 08-12 | Verify the 403 is gone: make a trivial store-listing edit and commit it. | 15 min | Proves the declaration cleared `edits.commit`, two days before you bet the release on it. |
| Wed 08-12 | Deploy the rewritten privacy policy. Load the deployed page in a real browser and confirm it renders. | 2 h | The site is a client-rendered SPA. A source route is not proof of a live page. |
| Wed 08-12 | Write the seven Health Connect justifications. Use the text below. | 1 h | |
| Wed 08-12 | Submit the **Health apps declaration** — **after** the policy is live. | 30 min | Reviewers compare justifications against the published policy. |
| Wed 08-12 | Start the first `release.yml` run in `mode: build-only`. Budget the day for debugging. | 4 h background | Never run. `build-only` touches no Play state. |
| Thu 08-13 | Land Sentry. In the same commit, add the in-app **Delete account** row in `SettingsPage`, using `Application.OpenURL` to `https://getsweatygames.com/reignandgain/delete-user-data`. Pattern exists at `AppRoot.Activity.cs:363`. | — | Play has required both an in-app path and a web link since 2024-04-15. The client has only the web link. |
| Thu 08-13 | Sentry project: EU region, **Prevent Storing of IP Addresses** on, short retention. | 30 min | `SendDefaultPii = false` does not stop the ingest server storing the IP. |
| Thu 08-13 | Seed a **permanent** Supabase test account. Warm the Render backend. Run the whole flow from a clean device. Write the reviewer instructions. | 1 h | A reviewer who cannot sign in rejects the release. A cold backend reads as broken. |
| Thu 08-13 | On that device confirm: (a) `getGrantedPermissions()` is non-empty after the Health Connect flow; (b) the app still records and settles a session with Health Connect denied; (c) the pre-prompt disclosure screens appear before the OS dialogs. | 1 h | (a) guards a silent no-op. (b) decides whether the closed test can start before the grant. (c) is prominent disclosure. |
| Thu 08-13 | Answer three form inputs from the code: does `authBootstrap` persist or display the Google profile name; is there a guest mode; does any guild endpoint show one player's name or activity to another. | 45 min | These drive Data safety `Name`, `Required vs Optional`, and `Shared`. |
| Thu 08-13 PM | **One Console session.** File in order: Data safety, App access, content rating (IARC), target audience, ads = No, advertising ID = No, and the four no-op declarations. | 4 h | Do all Console work before any upload. |
| Fri 08-14 AM | Cut a release branch at the Sentry commit. Build the AAB from it. Extract its **merged** `AndroidManifest.xml`. Confirm `com.google.android.gms.permission.AD_ID` absent and `cleartextTrafficPermitted` absent. | 45 min | The ad-ID answer is validated against the shipped artifact, not source manifests. |
| Fri 08-14 AM | Run `release.yml` in `mode: build-and-upload`, track `alpha`, from that same branch unchanged. Create and roll out the closed release. | 1 h | |
| Fri 08-14 – Sat 08-16 | Finish recruitment. Confirm 16-20 names with exact Google addresses. Do **not** mail the opt-in link yet. | — | |
| On "Published" (08-15 to 08-21) | Mail the opt-in link to every tester in the same hour. Start a dated per-tester record of who opted in when. | 1 h | Google publishes no progress counter. Your record is the only view of the 14-day continuity. |

### Waiting on Google — not compressible

| Starts | Queue | Realistic duration | Published SLA |
|---|---|---|---|
| 08-12 | Health apps + Health Connect data-type access | 3-14 days clean; 10-25 with one rejection cycle | **None** |
| 08-14 | First closed-testing release review | ~24 h typical, up to 7 days for a first closed release | No |
| 08-13 | Data safety propagation | Up to 7 days, longer in expanded review | No |
| Not applicable | Permissions Declaration Form (high-risk) | Several weeks | "Up to several weeks" |

The app does **not** request `ACCESS_BACKGROUND_LOCATION` — verified from the AAR. That is why the
several-week form is off this path. **Do not add background location.**

### Deadlock check on 08-11

The plan assumes the Health Connect review runs against a submitted form, in parallel, without a release.
Google's wording — *"This process must be completed for all publishing requests"* — can also be read as
tying the review to a publishing request. Read the form's state in the Console on 08-11.

**If the health review needs a submitted release**, invert: create the closed release on 08-14 as
planned, but seed the tester list with two or three internal people only. Mail the real 16-20 testers
only after the grant lands. No tester then spends continuity days on a build that cannot read Health
Connect.

**The closed test does not wait for the health grant either way.** `manualLoggingEnabled` arrives from
the backend `/sources` response (`Assets/Scripts/Net/Dto/SourcesDtos.cs:37`, consumed at
`AppRoot.Activity.cs:277`) and gates the `LOG WORKOUT` button. Turn it on server-side for tester
accounts. Testers exercise the full loop by manual entry while the health review runs.

---

## Data safety — complete answer set

`Shared = No` everywhere: the backend is first-party and Sentry is a contracted processor.
`Ephemeral = No` everywhere: the backend persists.

| Data type | Collected | Required / Optional | Purposes | Judgement call |
|---|---|---|---|---|
| Precise location | **Yes** | Optional | App functionality | No. `ACCESS_FINE_LOCATION` in the manifest; `GetNearbyGyms` sends `?lat=&lng=`. |
| Approximate location | **Yes** | Optional | App functionality | No. Declaring one without the other is its own mismatch. |
| Fitness info | **Yes** | Optional | App functionality | No. |
| Health info | **Yes** | Optional | App functionality | **Answer Yes.** Heart rate is a vital sign and sleep is not exercise. Neither fits "Fitness info". |
| Email address | **Yes** | **Required** | App functionality, Account management | Required unless a true guest mode exists. Check 08-13. |
| User IDs | **Yes** | Required | App functionality, Account management | Add "Fraud prevention" only if IDs drive guild ban enforcement. |
| Name | **Yes** | Optional | App functionality, Account management | Yes if `authBootstrap` stores or shows the Google display name. If not, answer No and declare the hero name under "Other user-generated content". |
| App interactions | **Yes** | Required | App functionality, Analytics | No. |
| Other user-generated content | **Yes** | Optional | App functionality | Guild names are free text, so this almost certainly applies. |
| Crash logs | **Yes** | **Required** | App functionality, Analytics | `CaptureLogErrorEvents = false` does **not** let you answer "not collected". |
| Diagnostics | **Yes** | Required | App functionality, Analytics | Session tracking is on. That is performance telemetry. |
| Device or other IDs | **Yes** | Required | App functionality, Analytics | **The most important answer.** Google's definition names "Firebase installation ID". You ship FCM today. If the live form says "not collected", it is false right now. Correct it in this submission. |
| Financial info, Messages, Photos/videos, Audio, Files, Calendar, Contacts, Browsing history, Search history, Installed apps | No | — | — | The `<queries>` block resolves Health Connect labels on device only. |

**Advertising ID:** not collected. Answer the separate **Ads** declaration as "no ads". Keep both
consistent.

**Encrypted in transit: Yes.** Confirm no `cleartextTrafficPermitted` in the merged manifest first.

**Data deletion: Yes.** Enter `https://getsweatygames.com/reignandgain/delete-user-data`. Do **not**
enter `POST /account-deletion` — a reviewer loads the URL in a browser and a JSON endpoint errors.

Before answering `Shared = No` on User IDs, Name and App activity, confirm what the guild endpoints
return to other members. Player-to-player visibility is a different question from third-party sharing.

---

## Health apps declaration — justification text

**Category:** Health and Fitness. **Use case:** Health-integrated games.
**Feature checklist:** Activity and fitness, plus Sleep management — sleep is surfaced as "LAST NIGHT'S
SLEEP" (`WorkoutsPage.cs:78-83`, `RenderSleep` at :383), so the checkbox is honest.

**READ_EXERCISE**
> Reign & Gain is a turn-based RPG in which the player's real-world workouts power their in-game
> character. We read completed exercise sessions to convert each workout into in-game progression: the
> session type and duration determine which rewards and character upgrades the player earns. Without
> exercise sessions the core gameplay loop has no input.

**READ_ACTIVE_CALORIES_BURNED**
> Active calories burned scales the size of the in-game reward for a workout, so that a harder session
> yields more progression than an easy one. The value is shown to the player in the post-workout summary
> alongside the rewards granted.

**READ_DISTANCE**
> Distance scores running and walking sessions, which map to a distinct in-game activity type. It is
> displayed in the player's workout summary and determines distance-based challenges.

**READ_HEART_RATE**
> Average heart rate during a session is submitted with the workout as an intensity signal. Our server
> uses it to set the reward tier for that session and to reject sessions where a stationary device would
> otherwise be credited as a workout. It is used for scoring and anti-cheat validation only.

> **Do not claim heart rate is displayed to the player.** It is not. It appears only in
> `ActivityDtos.cs:76`, `EvidenceBundleAssembler.cs` and `HostPayloads.cs`. No UI file renders it. A
> reviewer with the APK can check this in minutes, and a false claim restarts the declaration. If you
> would rather claim display, surface `avgHeartRate` on the workout summary before you file.

**READ_STEPS**
> Daily step totals drive the 24-hour activity feature, which grants small in-game rewards for everyday
> movement outside of dedicated workouts. The step total is displayed to the player on the Workouts
> screen.

**READ_SLEEP**
> Sleep duration is displayed to the player on the Workouts screen as recovery context alongside their
> workouts, under the heading "LAST NIGHT'S SLEEP". It is requested as an optional permission and the app
> functions fully without it.

**READ_HEALTH_DATA_IN_BACKGROUND** — the permission most likely to be denied. Write it as a necessity
argument, not a convenience argument.
> Players record workouts in other apps and on wearables while Reign & Gain is closed. A single scheduled
> periodic sync (HealthSyncWorker) reads only the data types listed above, and only records created since
> the last successful sync. The sync grants the in-game rewards for that workout and delivers a
> notification telling the player what they earned, so the reward arrives at the time of the workout
> rather than at the player's next app launch. Daily progression and streak continuity are scored against
> the day the activity happened; without background read a workout completed after the player's last
> session of the day is credited to the wrong day. We read no data type in the background that we do not
> also read in the foreground. Health data is never used for advertising, is never sold, and is never
> shared with third parties.

> **Only claim the notification if the background sync actually posts one.** The app already requests
> `POST_NOTIFICATIONS` and has copy for it, so wiring one is small. If you do not wire it before 08-12,
> delete the notification sentence and keep the rest.

---

## What Google controls and we do not

| Queue | Realistic duration | Published SLA | Shortenable |
|---|---|---|---|
| Health apps + data-type access review | 3-14 days clean, 10-25 with one rejection | **None** | Only by filing early and not getting rejected. |
| First closed-testing release review | 1-7 days; budget 7 | No | No. |
| Data safety propagation | Up to 7 days | No | Off the critical path by filing 08-13. |
| Foreground service declaration | Clears on save; no pre-approval | No | A rejection costs a full cycle with the 403 in force throughout. |
| 12 testers × 14 continuous days | 14 days, hard floor. **Confirmed to apply — personal account, July 2026.** | Yes | No. More testers does not shorten it. |
| Production access review after applying | "Usually 7 days or less" | Yes | No. |
| Production release review after access | 1-7 days | No | No. This is a second wait. |

---

## Restart risks

**Ordering**

1. **Running `release.yml` in `build-and-upload` while any Console form is open.** The Console edit
   discards the Play API edit and the Console wins. The upload appears to succeed and vanishes. Do all
   Console work first. `build-only` is safe at any time.
2. **Submitting Health apps before the privacy policy is live.** Reviewers compare against the published
   policy. Deploy the morning of 08-12; submit the afternoon.
3. **Filing Data safety before Sentry ships.** The form would describe a build that no longer exists.
4. **Mailing the opt-in link before the release reaches "Published".** The link fails in Draft and in
   Pending publication. Testers who see an error rarely retry.

**Content**

5. **Answering "Device or other IDs: not collected".** Already false today because of the Firebase
   installation ID. Inaccurate declarations, *including accidental ones*, carry removal and account
   strikes.
6. **Answering the advertising ID question from source manifests.** It is validated against the merged
   manifest of the shipped AAB.
7. **Claiming heart rate is displayed.** It is not.
8. **A demo video without a clearly user-initiated start and a visible, accurate notification.** The most
   common rejection for location-typed foreground services. `GymSessionService` is the harder one — its
   own source comment says it owns its work loop and runs with the app closed.
9. **A vague background-read justification.** The single most likely denial, on the only queue with no
   SLA.

**Artifact and process**

10. **Expecting to choose a versionCode.** `release.yml:140` and `scripts/build-android.sh:273` both
    derive it from `git rev-list --count HEAD`. It is 211 today. **If an upload half-lands and you retry
    on the same HEAD, the same versionCode is produced and the retry is rejected as a duplicate. The fix
    is to add a commit — an empty one is fine — and re-run.** `build-android.sh:234-242` refuses a dirty
    tree, so the recovery must be a real commit.
11. **Test credentials that expire or get rate-limited.** Review can repeat days later. Seed a permanent
    account.
12. **A cold Render backend at review time.** The reviewer sees a hang or a 500 and rejects the app.
13. **Recruiting exactly 12 testers.** One drop-out on day 10 does not pause a clock — it voids that
    tester's continuity permanently, and the replacement starts from zero. Recruit 16-20.
14. **Confusing "added to the tester list" with "opted in".** A tester who clicks the link signed in as a
    different Google account silently does not count.
15. **Testers who install and never open the app.** Google rejects production applications for
    disengaged testers. Collect per-tester activity counts from your own backend — the app already posts
    run settlements, so the evidence is free.

**Silent, no error message**

16. **Raising Sentry's `MaxBreadcrumbs` above 0.** Default HTTP breadcrumbs capture request URLs and
    query strings, and your URLs carry `?lat=&lng=`. That would put precise location into crash reports
    and invalidate a filed Data safety form, with no build failure and no visible symptom.
17. **Leaving "Prevent Storing of IP Addresses" off while the policy claims IP is not stored.** A policy
    discrepancy, not a form error.
18. **The Health Connect permission flow silently returning nothing.** Android 14+ routes it through the
    `ViewPermissionUsageActivity` alias. Without it, `getGrantedPermissions()` stays empty forever and
    `requestHealthRead` is a permanent no-op with no error. Assert non-empty on a clean Android 14+
    device on 08-13.
19. **`POST /account-deletion` returning 200 for unregistered addresses.** Correct anti-enumeration
    design, but a reviewer testing an arbitrary address sees success and receives no email. Make the
    delete-user-data page state that confirmation is sent only to registered addresses, and tell the
    reviewer in App access to use the supplied test account.

---

## Dates

**Closed testing.** Release submitted Fri 2026-08-14. Published between 08-15 and 08-21; budget 7 days
for a first closed review. **The 18 August target is met on the median case, not guaranteed.** Submission
is at its floor; publication is not yours to compress.

**Production, Organization account — NOT this account.** Kept only to show what was given up. No tester
requirement; earliest credible 2026-09-05, dominated by the health queue.

**Production — THIS ACCOUNT (personal, created July 2026).** Floor: published 08-15, all 12 opt in the same
day, apply 08-29, access ~09-05, live ~09-08 — none of which happens in practice. Realistic: published
~08-18, twelfth tester opted in ~08-21, apply ~2026-09-04, access ~09-11, **live 2026-09-18 to
2026-09-25**. Plan against late September. If the production-access application is rejected once, plan
against mid-October.

Internal testing contributes zero days to that clock, however long the app has been on the track.

---

## Accepted risks

- **The FGS use case may not fit a fitness tracker.** Choose "Background Location Updates: User-initiated
  location sharing" and describe the continuous GPS trace honestly. The remedy for a rejection is a
  `foregroundServiceType` change in `hostbridge.aar`, which is built in the backend repo. If the
  declaration is rejected, open a backend-repo bridge-contract issue the same day.
- **The production dates assume a clean production-access application.** They are no-rejection floors.
