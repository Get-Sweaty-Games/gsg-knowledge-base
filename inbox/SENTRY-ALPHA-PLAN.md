# Crash reporting in the alpha: one week, three repos

Written 2026-08-11. Supersedes the "Sentry, either half — not needed before there are players" row in
`MONITORING-PLAN.md`, which rested on a false premise. The app is on team devices now.

**Produced by a 12-agent workflow**: three probes (Sentry integration against this project, Play
compliance, privacy policy), two competing day-by-day plans, six adversarial critiques (ship-risk,
compliance, scope-cutting), one synthesis. Scores 7.0 and 6.0. The appendix lists every critic flaw and
its resolution.

## TRACK STATUS — resolved 2026-08-11

The open question in the first draft of this plan is answered. **The app is already live on the internal
track.** Closed testing is the next step, targeted around 2026-08-18, and the owner accepts the
"12 testers for 14 continuous days" requirement that comes with it.

So the internal-track recommendation below is not a substitution — it describes where the app already is.
Three consequences change the compliance half of this plan:

1. **The Data safety exemption ends at the closed-testing move.** It applies only while the app is
   *exclusively* on internal. The Data safety form, the Health apps declaration and a first-closed-release
   review are therefore **not deferrable to week two** — they land in the same week as the move. The
   "What does not fit in a week" section below is wrong on this point and is superseded by
   `CLOSED-TESTING-PATH.md` in this folder.
2. **Adding Sentry now is cheaper than adding it later.** The form must be filed for the move regardless.
   With Sentry already in the build, Crash logs / Diagnostics / Device IDs are declared once rather than
   amended after submission.
3. **The Foreground Service declaration gates the closed-testing date, not Sentry.** Internal releases
   skip the policy checks that promotion hits, which is why the app runs fine on internal today.

Everything below about the Sentry integration itself — the day plan, the configuration, the go/no-go, the
rollback — is unaffected.

**Verified directly by the orchestrator** (these outrank agent claims):

| Claim | Evidence | Status |
|---|---|---|
| `versionCode` is 211 today | `git rev-list --count HEAD` → 211 | Confirmed; no collision with 208 |
| The build script already hides the NuGet folder | `scripts/build-android.sh:347` → `hide "Assets/Plugins/NuGet"` | Confirmed |
| `release.yml` defaults to the internal track | `.github/workflows/release.yml:53` → `default: internal`, options `[internal, alpha, beta, production]` | Confirmed |
| `launcherTemplate.gradle:10-12` applies `com.google.gms.google-services`, **not** Firebase Crashlytics | probe read the file | Corrects an error in the earlier `MONITORING-PLAN.md` |

---

## Answer

**Yes. Crash reporting ships, and it reaches the team's devices on Thursday 13 August — five days before
the alpha cut.** The single thing most likely to stop the *alpha on Play* is the unanswered Foreground
Service declaration, which 403s `edits:commit` on every track including internal, is caused by
`hostbridge.aar` rather than by Sentry, and sits in a Google review queue nobody here controls. The
single thing most likely to stop *crash reporting itself* is a duplicate precompiled assembly between
Sentry's bundled BCL DLLs and `Assets/Plugins/NuGet/`, so that gets probed tonight with a real player
build, not on Friday.

Crash data is deliberately decoupled from Play. Play distribution is a separate, parallel ticket.

---

## Decisions taken

| Decision | Reason |
|---|---|
| Ship on Play's **internal** track, never `alpha` | See the owner decision above. `release.yml` already defaults to `internal`. |
| Build **locally** with `scripts/build-android.sh --release` | `release.yml` has never run. Its own header says to budget the first run as a debugging run. It stays off the critical path. |
| Sentry organisation in the **EU region** (`de.sentry.io`) | Set at organisation creation only. Not reversible. Removes the international-transfer question, which the privacy policy has no clause for. |
| **Never set a Sentry user** | Account deletion then has nothing at Sentry to cascade to. Owner decision, 2026-08-11. |
| `CaptureLogErrorEvents = false` and `MaxBreadcrumbs = 0` | Two flags close the quota risk and the health-leak channel. |
| Crash data reaches the **team's devices by sideload**, not by Play | The app is already installed there. Google's review queue must not gate crash reporting. |

---

## Day by day

Repo keys: **U** = `reign-and-gain-unity`, **L** = `gsg-landingpage`, **P** = Play Console / Sentry web.

| Date | Repo | Work | Output you can check |
|---|---|---|---|
| **Tue 11 Aug (today)** | P, L, U | Create the Sentry org in the EU region. Accept the DPA. Turn on IP suppression and health scrubbing. Merge the privacy PR. Start the FGS ticket. Add the package on a branch and start a player build. | A `de.sentry.io` DSN in hand. Privacy PR merged. |
| **Wed 12 Aug** | U, L | Verify the policy is live. Read the probe build result. Configure `SentryOptions.asset`. Fix five log sites and two parse sites. Write one test file. | Served bundle contains "Sentry". Player build compiles. |
| **Thu 13 Aug** | U | Run the EditMode suite with the Editor closed. Build the release AAB. Install on the owner's device. Throw a test exception. **GO/NO-GO at 17:00.** Merge to `main` on GO. | Test event in the EU project. `Logs/test-results.xml` green. |
| **Thu 13 Aug (evening)** | U | Sideload the release build to the team's devices. | Crash reporting live on team hardware. |
| **Fri 14 Aug** | U, P | Watch the first real events for 24 hours. Spot-check three payloads for health values and user ids. | Real crashes arriving, clean. |
| **Sat 15 / Sun 16 Aug** | — | **Slack. No scheduled work.** | — |
| **Mon 17 Aug** | U, P | Upload the AAB to Play as a **draft** on the internal track. This rehearses `edits:commit`. Re-verify the live policy bundle. | `edits:commit` returns 200, or it names the next missing declaration. |
| **Tue 18 Aug** | P | Promote the draft to internal. If Play still 403s, distribute the same signed build by sideload. | App installs from Play, or from the sideload link. |

---

### Tue 11 Aug — today, before anything else

Do these in order. None needs a build.

1. ~~**Check Play track exclusivity.**~~ Done (owner-reported, 2026-08-12) — **the closed-testing track
   is paused**, so no track carries an active release and the app is exclusively on internal. The Data
   safety form exemption this whole plan rests on holds.
2. ~~**Create the Sentry organisation. Select the EU region at creation.**~~ Done (owner, 2026-08-12).
   `de.sentry.io` confirmed, DSN live and verified end to end (probe event `11eef1320eb842ba814e1d3ac9ac5ca1`
   accepted). See `sentry-onboarding-state` memory.
3. ~~**Accept the DPA.**~~ Done (owner, 2026-08-12).
4. ~~**Enable "Prevent Storing of IP Addresses"**~~ Done (owner, 2026-08-12) as org-level scrubbing rules
   on `$user.ip_address` and `$user.geo` (dataset: Errors, Transactions, Attachments) rather than the
   DPA-era toggle by that exact name — same outcome, current Console shape.
5. ~~**Add Advanced Data Scrubbing rules** for the Health Connect field names~~ Done (owner, 2026-08-12) —
   added directly in the Additional Sensitive Fields list. This session never saw the exact field names
   that landed; check them against `Assets/Scripts/Host/HostPayloads.cs` if a health field ever turns up
   unscrubbed.
6. ~~**Add an inbound filter on the `editor` environment.**~~ Done, in two layers, neither of them the
   literal Sentry Console feature this line describes — **Sentry's Inbound Filters do not support
   filtering by an arbitrary tag like `environment`** (checked against the current docs; the supported
   list is browser/crawler/legacy-browser/IP/release/message/URL filters only). What actually ships:
   - `CaptureInEditor = 0` on `SentryOptions.asset`, already pinned and enforced by
     `SentryConfigTests.SentryOptionsAsset_KeepsItsPinnedFlags`. This is the load-bearing control — the
     SDK does not initialise at all while `Application.isEditor` is true, so the PlayMode suite and
     AutoPilot (both driven from inside the Editor) already send nothing.
   - `Assets/Scripts/Bootstrap/SentryOptionsConfig.cs` (added 2026-08-12), a `SentryOptionsConfiguration`
     wired onto `SentryOptions.asset`'s `OptionsConfiguration` field. Its `SetBeforeSend` drops any event
     whose `Environment` tag is `"editor"` — a second guard for the case `CaptureInEditor` gets flipped on
     to verify one event (a documented workflow in `sentry-onboarding-state` memory) and not flipped back.
     **Not** the `BeforeSend` hook the Wed-12-Aug section rules out below — that ban is scoped to
     health-data scrubbing (culprit/payload-keyed, risks suppressing a real crash); this keys on nothing
     but the SDK's own environment tag and carries none of that risk.
7. ~~**Open, merge and deploy the privacy PR.**~~ **Done and independently verified, 2026-08-12.**
   `curl https://getsweatygames.com | grep -o '/assets/index-[^"]*\.js'` →
   `/assets/index-BKRFV7a5.js`; fetched and read directly (not just grepped for "sentry"). All six
   named edits are live, verbatim or near-verbatim: the short version, the SDK clause, the
   sub-processor list (Sentry / Functional Software, Inc. + Google FCM), the retention paragraph (the
   correct "not linked to your account... deletes them on its own retention schedule" wording, not the
   false "nothing there to find" claim it was replacing), the background-sync sentence, and the new
   legal-basis/right-to-object section. `LAST_UPDATED` is bumped to **August 11, 2026**. The HTTPS
   sentence was correctly left untouched, as specified.
8. **Open the Foreground Service declaration as its own ticket.** It needs a description and a demo video
   per service type. Record two `adb screenrecord` captures of `RunSessionService` and
   `GymSessionService` from a device that already has the app. This work exists with or without Sentry.

   **Ticket half done (2026-08-12): [#363](https://github.com/Get-Sweaty-Games/reign-and-gain-unity/issues/363)
   opened**, `team:engineering`, describing both services (`docs/unity-bridge-contract.md:350-364`) and the
   two `adb screenrecord` capture commands needed. **The captures themselves are not done** — `adb devices`
   returned no attached device this session, so the video half needs a device with the app installed
   before it can move.
9. ~~**Start the assembly probe.**~~ **Confirmed (2026-08-12).** `Packages/manifest.json` and
   `Packages/packages-lock.json` both carry `"io.sentry.unity": "https://github.com/getsentry/unity.git#4.8.0"`
   — pinned exactly as specified (not the OpenUPM `scopedRegistries` route, not a bare/unpinned git URL),
   landed in commit `a525158` ("Install the Sentry SDK, pinned, with no DSN and nothing sending", #339).
   `scripts/build-android.sh` has run and produced `Builds/Android/ReignAndGain.aab` without an
   assembly-collision failure, so the probe's actual question — does the NuGet `System.Text.Json` /
   `System.Collections.Immutable` / `System.Reflection.Metadata` trio collide with Sentry's bundled
   copies in a player build — is answered **no**. Separately and NOT part of this probe: that same AAB
   has no `libsentry.so` at all (see GO/NO-GO condition 3) — a missing native lib on a stale build, not
   an assembly collision, and the two should not be conflated.

   On branch `feat/obs-sentry`, add to `Packages/manifest.json`:
   ```json
   "io.sentry.unity": "https://github.com/getsentry/unity.git#4.8.0"
   ```
   Commit `Packages/manifest.json` and `Packages/packages-lock.json` together. Do **not** add `io.sentry`
   to the existing `scopedRegistries` block — OpenUPM tops out at 1.8.0. Do **not** import the
   `.unitypackage` — it drags ~65 MB of `sentry-cli` binaries into `Assets/`. Then run
   `scripts/build-android.sh` and let it finish overnight.

**What the probe tests.** `Assets/Plugins/NuGet/` holds `System.Text.Json.dll`,
`System.Collections.Immutable.dll` and `System.Reflection.Metadata.dll`. Sentry 4.8.0 ships all three.
The NuGet copies carry `Editor: enabled: 0` and `Exclude Editor: 1`, so an Editor-side collision is
unlikely. The collision surface is the **player build**, and `scripts/build-android.sh:347` already hides
the folder for the duration of a build. The probe is expected to pass. Run it anyway — a pass costs
twenty minutes, a fail on Thursday costs a day. If the Editor errors on a duplicate assembly, move the
folder aside: `mv Assets/Plugins/NuGet /tmp/`. Removing the MCP package from `manifest.json` does **not**
help; the DLLs are files under `Assets/`, not package content.

Keep the branch. Do not delete and re-install on Wednesday.

### Wed 12 Aug — configure, and fix the leak at source

1. ~~**Verify the privacy policy is live**~~ (see cross-repo section). Do this first.
2. ~~**Read the probe build result.** Fix a collision now if there is one.~~ **Done (2026-08-12). No
   collision, so nothing to fix.** The answer was already recorded under Tue 11 Aug item 9:
   `scripts/build-android.sh` produced `Builds/Android/ReignAndGain.aab` without an assembly-collision
   failure, which answers the probe's actual question — whether the NuGet `System.Text.Json` /
   `System.Collections.Immutable` / `System.Reflection.Metadata` trio collides with Sentry's bundled
   copies in a **player** build.

   Re-confirmed on current code the same day: a release build from `2e0f98d` ran past
   `UnityILPP.PostProcessing/PostProcessAssembly` without a duplicate-assembly error. That stage is
   where the collision would surface, so passing it is the evidence, not the AAB appearing.

   **Do not read this as covering `libsentry.so`.** That is GO/NO-GO condition 3 and a separate
   question — a missing native library is not an assembly collision, and Tue item 9 warns against
   conflating the two.
3. ~~**Run `Tools > Sentry`.** Paste the EU DSN into `Assets/Resources/Sentry/SentryOptions.asset`.~~ Done —
   the DSN is live and verified end to end (see item 2 above). The Editor-only `OptionsConfiguration`
   guard added 2026-08-12 is wired onto the same asset; see item 6 of Tue 11 Aug.
4. ~~**Apply the configuration table below.**~~ Done — `SentryOptions.asset`'s pinned flags match the
   table and are enforced by `SentryConfigTests.SentryOptionsAsset_KeepsItsPinnedFlags`.
5. **Fix the five log sites that print a parse exception message.** A `JsonException` message embeds the
   JSON path and the offending token — the health field name and a value fragment. Replace `{e.Message}`
   with `{e.GetType().Name}` in each:
   - ~~`Assets/Scripts/Host/HealthReadSession.cs:106`~~ Done (2026-08-12), `HandleSignal`
   - ~~`Assets/Scripts/Host/PresenceSnapshotSession.cs:148`~~ Done (2026-08-12), the `OnPresenceSnapshot` catch
   - ~~`Assets/Scripts/Host/RunSession.cs:182`~~ Done (2026-08-12), `HandleEnded`
   - ~~`Assets/Scripts/AppUI/AppRoot.Activity.cs:87`~~ Done (2026-08-12), `HandleSleepSummary`
   - ~~`Assets/Scripts/AppUI/AppRoot.Activity.cs:103`~~ Done (2026-08-12), `HandleConnectedSources`

   **5 of 5 done.** `Assets/Scripts/Net/SupabaseAuthClient.cs:225` is a separate call site (not one of
   these five) and is still open — see item 6 below.
6. **Wrap two unguarded parses in try/catch.** Both deserialise credentials and both throw unhandled
   today, so the exception message reaches Sentry as an event:
   - ~~`Assets/Scripts/Host/HostBridgeReceiver.cs:48`~~ Done — wrapped, and correctly logs
     `e.GetType().Name` only (line 79-80).
   - ~~`Assets/Scripts/Net/SupabaseAuthClient.cs:217`~~ Done (2026-08-12) — wrapped, and the catch at
     line 225 now logs `e.GetType().Name` too.

   Catch, log the exception type, never the payload. Correct with or without Sentry. **Both done.**
7. ~~**Write `Assets/Tests/EditMode/SentryConfigTests.cs`**~~ Done — file exists with the two tests below
   plus three more (the pinned-flags table, the must-not-be-zero tuning values, and the pattern
   self-check), all four checked and, as of the last recorded run (2026-08-12), part of a 1452/1452
   EditMode pass:
   - Load the options asset. Assert `SendDefaultPii == false`, `CaptureLogErrorEvents == false`,
     `MaxBreadcrumbs == 0`.
   - Read every `.cs` under `Assets/Scripts/`. Fail on any match for `ConfigureScope`,
     `SentrySdk.SetUser` or `.User =`. This is the guard that actually holds — there is no `User` field
     on the options asset, so an asset-level assertion would pass forever and prove nothing.

**No `BeforeSend` hook ships.** `MaxBreadcrumbs = 0` closes the channel that actually carried health
data, and the seven source edits close the message channel where the data exists. A `BeforeSend` hook
does not run on native crashes at all, and a culprit-based hook would suppress crashes in the health-sync
path — the newest code in the repo, and the code you most want reports from.

### Thu 13 Aug — verify, gate, merge, sideload

> **Status as of 2026-08-12 (Wednesday).** Items 1 and 9 are already done, one of them ahead of the
> gate that was supposed to authorise it — read item 9 before trusting the rollback plan. Item 3 is in
> flight. Items 2 and 4 through 8 have not started, and 5 through 7 all need a device that was not
> attached on 12 Aug.

1. ~~**Close the Unity Editor.** Run `scripts/test-editmode.sh`. Read `Logs/test-results.xml`. Do not
   inherit a remembered test count.~~ **Done 2026-08-12, EditMode 1472/1472 green**, run headless with
   the Editor closed, count read from `Logs/test-results.xml` rather than inherited. That run already
   includes the `SentryConfigTests` file this plan asked for.

   **It will need re-running if any code changes before the gate.** The number above is evidence about
   commit `2e0f98d` and nothing later.
2. **Run the PlayMode suite and one AutoPilot end-to-end run.** Sentry installs a
   `[RuntimeInitializeOnLoadMethod(SubsystemRegistration)]` hook that runs before every PlayMode test, so
   an init-order surprise surfaces here or nowhere. Confirm AutoPilot logs `[RUN] CLEARED` or
   `[RUN] DEFEATED` and terminates. Then call `SceneWiring.SetAutoPilot(false)` or
   `Assets/Scenes/Bootstrap.unity` stays dirty.
3. **Build the release AAB once**:
   `scripts/build-android.sh --release ../../reign-and-gain-backend/android-host/keystore.properties <versionName>`.
   Only one build happens this week, and it is the artifact that ships. A debug-signed build proves
   nothing about the release build, so it is not made.

   **Started 2026-08-12 as `0.1.0` from `2e0f98d` (versionCode 252). Not yet finished.** Three earlier
   attempts the same day did not produce an artifact, and none of them was a project fault:

   - **Attempt 1** refused before launching Unity — the working tree was dirty, and the script will not
     build an artifact that matches no commit. The two `bicep*.png.meta` files that had churned
     uncommitted for three sessions were committed to clear it.
   - **Attempt 2** was cancelled by the owner mid-build. The script's restore ran correctly.
   - **Attempt 3** died ~1 minute in: `IPCStream (Upm-32740): IPC stream failed to read (Not connected)`
     then `Failed to resolve packages: operation cancelled`. The Package Manager child process dropped.
     Licensing was **not** the cause despite two alarming lines in the log — the same log records
     `Successfully resolved entitlement details`, Unity Personal, unlimited.
   - **Attempt 4** refused on the stale `Temp/UnityLockfile` that attempt 3's death left behind, with no
     `Unity.exe` running. Removed and re-run.

   **"Once" means one SHIPPING artifact, not one invocation.** Repeated attempts at the same release
   build do not violate this step; making a debug-signed build and reasoning from it would. What the
   step actually requires is that the AAB which ships is built from the commit that ships.
4. **Read `git status --short`.** `ProjectSettings/ProjectSettings.asset` will be dirty because
   `Il2CppLineNumberSupportEnabled` appends `--emit-source-mapping` via
   `PlayerSettings.SetAdditionalIl2CppArgs`. Commit **only** that line. `git checkout --` anything else
   in that diff. This file is shared with `@Get-Sweaty-Games/unity-devs` and carries the pinned
   targetSdk. Commit it **on the branch** so the whole change reverts with one SHA.
5. **Install the release build on the owner's device** with `bundletool build-apks --mode=universal`
   using the same keystore, then `adb install -r`.
6. **Throw a deliberate exception from a shipping code path.** Read the raw event JSON in the EU project.
7. **Verify the FAST rollback while the build is fresh.** Set `Enabled = false` on the options asset,
   rebuild, then `aapt2 dump xmltree` over the APK manifest. Confirm no `io.sentry.dsn` meta-data
   remains. If the DSN survives, the fast rollback is `Enabled = false` **plus**
   `AndroidNativeInitializationType = Runtime`, because `sentry-android-core` merges a
   `SentryInitProvider` ContentProvider that self-initialises before `Application.onCreate`. Record which
   it is.
8. **GO/NO-GO at 17:00.** See below.
9. ~~**On GO: squash-merge `feat/obs-sentry` into `main`.** One commit — the rollback handle for the rest
   of the week.~~ **OVERTAKEN BY EVENTS, AND THE ROLLBACK HANDLE THIS STEP PROMISED DOES NOT EXIST.**
   Read this before relying on the Rollback section below.

   The Sentry work reached `main` on 2026-08-11/12 through the ordinary PR flow, **before the GO/NO-GO
   gate that was meant to authorise it**, and across more than one commit:

   - `a525158` — "Install the Sentry SDK, pinned, with no DSN and nothing sending" (#339)
   - `da55c49` — "Wire SENTRY_AUTH_TOKEN into CI, close the log leaks, and make a 5xx self-describing"
     (#362, branch `chore/sentry-privacy-flags`)

   `origin/feat/obs-sentry` still exists but is stale: its only commit, `0d8a0d8`, is the unsquashed
   twin of `a525158`, so there is nothing left on it to merge.

   **Two consequences, neither cosmetic.** First, "revert one SHA" is no longer the rollback — reverting
   `a525158` alone leaves #362's changes, and reverting both touches the 5xx copy work that has nothing
   to do with Sentry. Second, the gate lost its teeth: NO-GO can no longer be executed by simply not
   merging, because the merge already happened. If the gate fails, the rollback is now the
   `Enabled = false` route in item 7, which is exactly the route item 7 says has never been verified.

   Do not "fix" this by force-pushing history. Decide at the gate whether the flag route is sufficient,
   and if it is not, write the revert as its own PR.
10. **Sideload to the team's devices the same evening.**

### Fri 14 Aug — observe

Watch event volume for 24 hours against the 5,000 errors/month free tier. Open the first three real
events and check three things: no `user.email`, no health field name or value anywhere in the payload,
and an empty `breadcrumbs` array. The Thursday test exception proves the path it was thrown from. Real
crashes come from paths nobody chose. Set an inbound rate limit on the project client key if volume looks
wrong.

### Mon 17 Aug — rehearse the Play commit

Upload the Thursday AAB with
`node scripts/upload-play.js <sa.json> Builds/Android/ReignAndGain.aab --track internal`. The default
status is `draft`, so nothing reaches a tester. This is the first and only chance to learn whether
`edits:commit` still 403s while there is still a day to react. Play names exactly one missing declaration
per 403, so a failure here enumerates the next gate. A failed commit discards the whole edit, so read the
error body before retrying.

Re-verify the live privacy bundle. The fork sync has regressed a live bundle before.

### Tue 18 Aug — cut

Promote the draft to internal. Install from the Play internal link on one physical device. If Play still
403s, distribute the same signed universal APK to the cohort by direct link. Crash reporting is
unaffected either way.

---

## Cross-repo dependency: the privacy policy

The policy lives in a **different repo with a different deploy**:
`gsg-landingpage/app/src/sites/reign-and-gain/pages/PrivacyPage.tsx`.

It must be **live** before the first Sentry event is sent. It merges **today**, two days before any
event.

**Merging is not deploying.** Vercel builds from the personal fork `yarins0/get-sweaty-games-landing-pg`,
force-pushed by `.github/workflows/sync-fork-on-merge.yml`. That chain has silently failed twice — 31
July (one day stale) and 2 August (three days stale) — while the Vercel dashboard read "Ready".

**Do not use `curl https://getsweatygames.com | grep -i sentry`.** It returns nothing whether the deploy
worked or not. `app/index.html` is a bare Vite shell and `app/vercel.json` rewrites every path to it. The
policy text lives in the hashed JS bundle. Use this instead:

```bash
curl -s https://getsweatygames.com | grep -o '/assets/index-[^"]*\.js'
```

Record that hash today. On Wednesday and again on Monday, the hash must have **changed**, and the new
bundle must contain the word Sentry:

```bash
HASH=$(curl -s https://getsweatygames.com | grep -o '/assets/index-[^"]*\.js'); curl -s "https://getsweatygames.com$HASH" | grep -ci sentry
```

The bundle name is content-derived. A stale bundle keeps its old name — exactly the failure that went
undetected for three days. Also check the Actions tab on **both** the org repo and the fork. Do not wait
for `live-deploy-check.yml`; it runs once daily at 09:00 UTC and can be 24 hours late.

### The six edits

Bump `LAST_UPDATED` at `:42` in the **same commit**. The file's own comment at `:37-42` demands it and it
has been forgotten before.

1. **`:220-223`, the SDK clause.** Delete "never a separate tracking channel". That clause is false on
   either reading of whether crash reporting counts as analytics, so do not spend a day on the taxonomy
   argument. Replace with:

   > The game app sends crash reports to Sentry, a third-party service. When the game hits an error or
   > closes unexpectedly, Sentry receives what went wrong: the error, the code path it came from, and the
   > device it happened on. We don't send your name, your email, your account, or any health or location
   > data. Sentry does see the internet address your device connects from, and we have turned off storing
   > it. The game also tells Sentry when it starts and stops, so we can tell how often it crashes; that
   > carries no information about you. Crash reports are stored on your device until the network is
   > available.
   >
   > Those reports are not linked to your account. Sentry gives each installation a random identifier so
   > it can tell fifty crashes on one phone from one crash on fifty phones. We never tell Sentry which
   > account an installation belongs to.
   >
   > Apart from that, the game has no third-party analytics SDKs or ad trackers.

   Note the wording: "we don't send", not "Sentry does not receive". The IP arrives at ingest on every
   event and no SDK flag stops that.

2. **`:152-163`, the sub-processor list.** The current framing is "the infrastructure providers that run
   the app". Sentry does not run the app, so restructure rather than insert. Add Sentry (Functional
   Software, Inc.) and Google Firebase Cloud Messaging — Firebase already ships in the app today and
   already belonged on that list.

3. **`:261-268`, retention.** Do **not** write "there is nothing there to find and delete". That is a
   fresh false claim in the document you are fixing. A persistent installation identifier is still
   personal data. Write instead:

   > Crash reports at Sentry are not linked to your account, so deleting your account does not delete
   > them, and there is nothing in them that identifies you. Sentry deletes them on its own retention
   > schedule.

   Do not publish a specific retention figure until you have read it in the Sentry billing settings.

4. **`:70-77`, the short version.** Add: "The game also sends crash reports to Sentry when something
   breaks, with no account information attached." This is the paragraph most people read.

5. **`:141`, the background sentence.** "That is the only thing it does in the background" becomes "That
   is the only thing it does in the background, apart from reporting a crash if the sync itself fails."

6. **New: legal basis and the right to object.** The policy has no lawful-basis section and no rights
   section at all. Article 13(1)(d) requires the legitimate interests to be stated and Article 21(4)
   requires the right to object to be presented explicitly. Add two sentences naming crash diagnostics as
   legitimate-interest processing and giving an email address to object.

**Not edited:** `:230-231`, the HTTPS sentence. It says "between the app and our servers", which stays
true. Adding Sentry to it is over-disclosure.

---

## GO / NO-GO — Thursday 13 August, 14:00 to 17:00

Five hard conditions. All five must pass. Check each; do not assert any.

| # | Condition | How to check |
|---|---|---|
| 1 | ~~**Privacy policy live**~~ **Done, independently verified, 2026-08-12** | Fetched and read the live bundle (`/assets/index-BKRFV7a5.js`) directly — all six required edits present, `LAST_UPDATED` bumped to August 11, 2026. See item 7 of Tue 11 Aug for the full check. |
| 2 | ~~**EditMode suite green**~~ **Done** — `Logs/test-results.xml`, run ending 2026-08-12 07:10:23Z: `result="Passed"`, 1392/1392, 0 failed. Count is 1392, not the 1452 an earlier session recorded — expected drift, CLAUDE.md says never to inherit a remembered figure. **Not independently confirmed that the Editor was closed for this run** — the file itself doesn't record that, only `scripts/test-editmode.sh` refusing to run with the Editor open does. | `Logs/test-results.xml` written by `scripts/test-editmode.sh` with the Editor **closed**, zero failures. |
| 3 | **NOT done.** `Builds/Android/ReignAndGain.aab` exists (130 MB, inside the 150 MB ceiling) but is dated **2026-08-11 11:09, before** `93cb643` (the `SENTRY_AUTH_TOKEN` wiring) and before every log-leak fix this week. `unzip -l` on it shows `lib/arm64-v8a/`: `lib_burst_generated.so`, `libc++_shared.so`, `libgame.so`, `libil2cpp.so`, `libmain.so`, `libunity.so` — **no `libsentry.so`, no `libsentry-android.so` at all.** This build predates Sentry entirely and cannot be the one condition 4 tests against. A fresh release build is needed. | Under 150 MB (expect 122-124 MB, up from 118.6). `unzip -l` shows `lib/arm64-v8a/libsentry.so` and `libsentry-android.so`, and no other ABI. |
| 4 | **NOT done.** Depends on condition 3, which isn't met — no Sentry-carrying release build exists to test on a device yet. No raw-JSON event record or device-test evidence found in the repo. The DSN probe event recorded in `sentry-onboarding-state` memory (`11eef1320eb842ba814e1d3ac9ac5ca1`) verified ingest end-to-end but was **not** a release-build-on-device test against this exact checklist (managed stack trace, absent `user.email`/`user.username`, `user.id` present, empty `breadcrumbs`, no health field) — that specific check is still open. | Release build on a physical device. Throw a deliberate exception. Read the **raw JSON**, not the UI summary. Require: managed stack trace present; `user.email` absent; `user.username` absent; `user.id` present and equal to the SDK installation id and nothing else; `breadcrumbs` empty; no health field name and no health value anywhere. |
| 5 | ~~**Compliance controls active**~~ **Done** — DPA accepted (owner, 2026-08-12). IP suppression on (org-level `$user.ip_address`/`$user.geo` scrubbing, owner, 2026-08-12). Editor inbound filter on, two layers: `CaptureInEditor=0` (pinned, tested) plus the `SetBeforeSend` guard in `SentryOptionsConfig.cs` (added 2026-08-12). | DPA accepted. "Prevent Storing of IP Addresses" on. Editor inbound filter on. |

**Condition 4 expects a `user.id`.** The Unity SDK mints a random per-installation id when no user is
set. A build with no `user.id` at all would be a surprise, not a pass. Do not write a gate that a correct
build cannot satisfy.

**Soft conditions — record, warn, do not stop:** the event resolves to a file and line;
`Logs/sentry-symbols-upload.log` exists and reports success. `SentryCliOptions.IsValid()` fails soft — a
missing or wrong auth token removes the upload task from the generated Gradle file and the build still
exits 0. An unsymbolicated managed event still carries the exception type, message, method names and
device context. That is most of the value. Symbols can be uploaded against the same build later with
`sentry-cli debug-files upload`. Do not throw away all crash capture over line numbers.

**Cost of a NO-GO: one day.** The alpha still cuts on Tuesday without Sentry, using `adb logcat` off team
devices. The privacy policy stays live and over-discloses, which is harmless.

---

## Exact Sentry configuration

### `Assets/Resources/Sentry/SentryOptions.asset` — committed

| Setting | Value | Why |
|---|---|---|
| `Dsn` | the EU (`de.sentry.io`) DSN | Set at org creation only. Not migratable. |
| **`CaptureLogErrorEvents`** | **`false`** | **The load-bearing flag.** It gates only `LogType.Error` and `LogType.Assert`. Unhandled exceptions and `Debug.LogException` route through a separate integration and are **unaffected**. 25 `Debug.LogError` sites ship (28 more are Editor-only) and `ProceduralSfx.cs:341` fires per unmapped SFX request. Left on, the free tier can be exhausted in hours, and the real crashes are then the ones dropped. |
| **`MaxBreadcrumbs`** | **`0`** | Breadcrumbs are captured from `Debug.Log` regardless of `CaptureLogErrorEvents`, they ride on **every** event including an unrelated combat crash, and they are synced to the native layer where no managed filter reaches them. This is the real health-leak channel. |
| **`SendDefaultPii`** | **`false`** (default) | No email, username, device unique id, or machine name. |
| `AutoSessionTracking` | `true` (default) | Crash-free session rate is the whole signal in a week-long alpha. Sessions are not billed against the error quota. Disclosed in the policy edit rather than turned off. |
| `StructuredLogOnDebugLogWarning` / `Assertion` / `Error` / `Exception` | `false` | Structured Logs is a **separately billed** channel that `CaptureLogErrorEvents` does not gate. |
| `TracesSampleRate` | `0` (default) | No performance quota. |
| `AttachScreenshot` / `AttachViewHierarchy` | `false` (default) | A screenshot of a fitness app can contain health values. |
| `AnrDetectionEnabled` / `EnableAppHangTracking` | `false` (default) | Noise. |
| `AndroidNativeSupportEnabled` / `NdkIntegrationEnabled` | `true` (default) | Native crash capture. |
| `Il2CppLineNumberSupportEnabled` | `true` (default) | This is what dirties `ProjectSettings.asset` with `--emit-source-mapping`. |

**No user id is ever set.** Nothing calls `SentrySdk.ConfigureScope` to set `s.User`. Pinned by the
source-grep test, not by an asset field.

### The auth token — where it lives so it is never committed

The `sentry-cli` auth token is a plain `[SerializeField] string` on
`Assets/Plugins/Sentry/SentryCliOptions.asset`, which the wizard writes by default, under `Assets/`, and
which is copied verbatim into a `sentry.properties` file at build time.

1. Add to `.gitignore`:
   ```
   Assets/Plugins/Sentry/SentryCliOptions.asset
   Assets/Plugins/Sentry/SentryCliOptions.asset.meta
   ```
2. Put the real token, organisation and project into that asset locally.
3. Before staging, run `git diff --cached Assets/Plugins/Sentry/` and confirm no `auth` value appears.

No code, no `SentryCliOptionsConfiguration` subclass. Consequence, stated plainly: **CI has no token, so
a CI build produces no symbols.** That is fine this week, because every build this week is local. Write
the environment-variable subclass in week two, in the same PR that wires `SENTRY_AUTH_TOKEN` into
`release.yml`, where it can be tested.

---

## The team's devices — crashing unobserved today

A different and faster problem than the alpha cohort. Nothing here needs Play, and nothing here waits for
the Foreground Service declaration.

**Today, at zero cost:** run `adb logcat -d > crash.txt` on one team device and grep for
`AndroidRuntime`, `Unity` and `Exception`. Managed C# exceptions are in the log buffer right now. This is
the answer available before any build exists.

**Thursday evening, the real fix:**

1. **Learn which key the existing installs carry**, before anything is built:
   ```bash
   adb shell pm path <package>
   ```
   Then `adb pull` the path and run `apksigner verify --print-certs installed.apk`.
2. **Extract installable APKs from the same release-signed AAB the Play route uses.** One artifact, one
   signature:
   ```bash
   bundletool build-apks --bundle=Builds/Android/ReignAndGain.aab --output=team.apks --mode=universal --ks=<keystore> --ks-key-alias=<alias>
   ```
3. **Install over the existing app:** `adb install -r <universal.apk>`.
4. **If the signature differs**, `adb install -r` fails with `INSTALL_FAILED_UPDATE_INCOMPATIBLE`. Every
   tester must then uninstall first, which loses local run state. Say this in the tester instructions on
   Thursday, not on Tuesday.

**Do not distribute a debug-signed APK.** `scripts/build-android.sh` with no arguments produces one, and
it cannot upgrade a release-signed install.

**Play Android Vitals is not a substitute.** It sees native crashes and ANRs only, it will never see a
managed C# exception, and it needs a Play-installed build. On a ten-device cohort it may report nothing.

---

## Rollback

Sentry writes **only** to generated files under `Library/Bee/Android/Prj/IL2CPP/Gradle/`, which is
gitignored and regenerated per build. It applies **no** Gradle plugin. It never writes to
`Assets/Plugins/Android/`, so `launcherTemplate.gradle`, `mainTemplate.gradle`, `AndroidManifest.xml` and
`hostbridge.aar` are untouched by both the install and the revert. There is no gradle or manifest residue
in the repo to unpick.

**Level 0 — before Thursday 17:00.** Sentry lives only on `feat/obs-sentry`. Rollback is
`git branch -D feat/obs-sentry`.

**Level 1 — after the Thursday merge.** `git revert <squash-sha>` on a branch, PR, squash-merge. Ten
minutes plus a verifying build. The entire tracked footprint is six things:

- the `io.sentry.unity` entry in `Packages/manifest.json`
- its resolved pin in `Packages/packages-lock.json`
- `Assets/Resources/Sentry/` (plus `.meta`)
- `Assets/Plugins/Sentry/` (plus `.meta`; the CLI asset is gitignored)
- `Assets/Tests/EditMode/SentryConfigTests.cs`
- one `--emit-source-mapping` line in `ProjectSettings/ProjectSettings.asset`

All six land in **one** squash-merge, so one SHA reverts all of it. The seven source edits to
`Assets/Scripts/Host/`, `Net/` and `AppUI/` are correct with or without Sentry — keep them.

**Level 2 — the 60-second escape hatch**, if it breaks the day before the cut:

```bash
jq 'del(.dependencies["io.sentry.unity"])' Packages/manifest.json > tmp && mv tmp Packages/manifest.json
```

Then remove the asset folders and the test file:

```bash
rm -rf Assets/Plugins/Sentry Assets/Plugins/Sentry.meta Assets/Resources/Sentry Assets/Resources/Sentry.meta Assets/Tests/EditMode/SentryConfigTests.cs Assets/Tests/EditMode/SentryConfigTests.cs.meta
```

The `rm` is not optional. `SentryConfigTests.cs` references Sentry types, and deleting the package alone
leaves a hard `CS0246`. The precedent for the `jq` line is `.github/workflows/release.yml:113`, which
does the same surgery on `com.ivanmurzak.unity.mcp`. Run this exact command once on Friday and confirm
the EditMode suite is still green afterwards. An untested rollback is a claim.

**Fast rollback, no revert:** set `Enabled = false` on the options asset and rebuild. **Verify on
Thursday whether that also stops the native layer.** If the DSN survives in the merged manifest, the fast
rollback is two settings: `Enabled = false` **and** `AndroidNativeInitializationType = Runtime`.

**Manifest residue check after any rollback build:** `aapt2 dump xmltree` the APK's `AndroidManifest.xml`
and confirm no `io.sentry.dsn` meta-data and no `io.sentry.android.core.SentryInitProvider` provider.

**Nothing rolls back in `gsg-landingpage`.** A policy that discloses a crash reporter you removed is
over-disclosure, which is harmless.

---

## What does not fit in a week

Stated plainly, not hedged.

- **Lawyer review.** One to two weeks including finding someone. Two questions genuinely need one:
  whether Article 9 special-category health data can reach a crash report at all, and whether the
  legitimate-interest basis holds. Legitimate interest is **not** an available basis for Article 9 data,
  so this is a real question, not generic caution. An alpha on team devices and a small invited cohort —
  with a truthful live policy, no account identifier at Sentry, EU hosting, a signed DPA,
  `MaxBreadcrumbs = 0`, and the seven source edits — is a proportionate position to hold meanwhile. That
  is a **deliberate deferral**, and it must close before public launch.
- **The written legitimate-interest assessment.** Two hours, and the owner's own job rather than a
  lawyer's. Schedule it for week two. Until it exists, the Article 6(1)(f) basis is an assertion.
- **Play's closed-testing (`alpha`) track.** Three serialised reviewed gates queue behind the unanswered
  FGS declaration. No sequencing makes that fit. See the owner decision at the top.
- **The Play Data safety form and the Health apps declaration.** Both are exempt on internal testing.
  Write the answers now while they are text, because one is currently wrong in the dossier: **Health info
  must be declared `Collected: YES`, `Shared: NO`.** The app reads seven Health Connect permissions and
  posts workout bundles to the Render backend. Declaring `Health info: No` for an app whose listing
  requests Health Connect permissions is a mismatch Google detects automatically and treats as a false
  declaration. Full set: Crash logs YES/not shared, Diagnostics YES/not shared, Device or other IDs
  YES/not shared (covering **both** Sentry's installation id and Firebase's FID, which is already owed
  today), Health info YES/not shared, Advertising ID NO.
- **The FGS declaration's review outcome.** Submitting it is controllable. Google's answer is not. A
  location-typed service that runs with the app closed is the most likely thing to draw pushback, and
  `hostbridge.aar`'s own manifest comments show a permission was already trimmed for this reason. There
  is no engineering workaround: the permissions come from an AAR built in another repo which must never
  be hand-edited here.
- **A verified `release.yml`.** Never run. Its one self-flagged unverified assumption — whether
  `unity-builder` forwards the `RG_*` env vars — is the exact mechanism a `SENTRY_AUTH_TOKEN` would ride.
  With a 43-minute build and a 120-minute CI timeout there is room for two or three debugging cycles, and
  each competes with the SDK work for the same person. Deferred deliberately.
- **A real CI gate on the Unity half.** `tests.yml` is `disabled_manually`. For this week, "verified"
  means four manual checks by one person: EditMode with the Editor closed, PlayMode the same way, an
  AutoPilot run that terminates, and a physical device install. That is thinner than it should be and it
  is the honest state.
- **Native / NDK symbolication.** `debugSymbolLevel` is a Unity placeholder in `launcherTemplate.gradle`
  fed from the gitignored `Library/EditorUserBuildSettings.asset`, so CI inherits a container default.
  Managed C# traces come from the IL2CPP source mapping and are unaffected. Native crashes in the alpha
  will likely come back unsymbolicated. The `DemoBuild.cs` pin needs an asmdef change
  (`ReignAndGain.Editor.asmdef` has `overrideReferences: false` and does not reference
  `UnityEditor.Android.Extensions`), so it is not the one-line change it looks like. Week two.
- **A persisted consent surface.** `Assets/Scripts/AppUI/PermissionGate.cs` is a 177-line explainer modal
  with a per-show callback and no persistence. It is not a consent store. Building one means a persisted
  choice, a Settings toggle, and gating SDK init on it — a day or two. It **cannot** be retrofitted onto
  crashes already collected. This is an accepted ceiling of the legitimate-interest approach.
- **The 12-testers-for-14-continuous-days production clock.** Internal testing does not satisfy it. The
  internal-track decision defers that requirement rather than removing it. If production matters this
  quarter, start a closed track in parallel — after the alpha, not before it.
- **A guarantee that Play clears by 18 August.** This plan guarantees crash reporting on the team's
  devices by 13 August and on the alpha cohort by 18 August. It cannot guarantee the Play internal track
  by 18 August, because it does not control a review queue.

---

## Appendix: every critic flaw, resolved

| Flaw raised | Resolution |
|---|---|
| `curl \| grep sentry` gate is un-passable (SPA shell) | Fixed. Bundle-hash check, recorded Tuesday, asserted changed Wednesday and Monday. |
| The plan burns today | Fixed. Today is a full working pre-day. |
| Play gates never enumerated; discovered Sunday | Fixed. Draft upload rehearses `edits:commit` on Monday. The dossier's `--promote 208` command is dropped; it needs an AAB or a versionCode Play actually holds. |
| Sideload fallback untested, signature unknown | Fixed. `apksigner verify --print-certs` before anything is built; the fallback is the release-signed universal APK. |
| Pinning test asserts a nonexistent `User` field | Fixed. Source-grep test plus three real asset asserts. |
| No slack day | Fixed. Saturday and Sunday are empty. |
| Fast rollback unverified against native init | Fixed. Verified Thursday with `aapt2 dump xmltree`. |
| Duplicate BCL assembly collision | Fixed and re-scoped. NuGet copies are Editor-excluded and `build-android.sh:347` hides the folder, so the risk is player-build-only. The "drop the MCP package" fix is wrong and is not used. |
| Two builds, first proves nothing | Fixed. One release build only. |
| Breadcrumbs bypass `CaptureLogErrorEvents` and `BeforeSend`; native crashes bypass `BeforeSend` entirely | Fixed. `MaxBreadcrumbs = 0` plus source fixes at five log sites and two unguarded parses. |
| Scrub keyed on field names misses values and suppresses wanted crashes | Fixed. No scrub hook ships. Root-cause source edits replace it. |
| Data safety "Health info: No" is a false declaration | Fixed. Health info YES / not shared, and Device or other IDs covers Firebase's FID too. |
| Internal-track exemption assumed, not checked | Fixed. Track-exclusivity check is item 1 today. |
| Retention text "nothing there to find and delete" is a new falsehood | Fixed. Replaced with a practice claim. |
| IP suppression and server-side scrubbing scheduled after the cut | Fixed. Both today, before the DSN exists. |
| Editor and PlayMode events pollute the baseline and burn quota | Fixed. Inbound filter on the `editor` environment. |
| Go/no-go demands "no user id", which a correct build cannot satisfy | Fixed. The installation id is expected and named. |
| Symbolication as a hard STOP throws away 80% of the value | Fixed. File-and-line is a soft condition. |
| Policy live *after* the first event | Fixed. Policy merges today, two days before any event. |
| No legal-basis or right-to-object section | Fixed. Sixth policy edit adds both. |
| `AutoSessionTracking = false` discards crash-free session rate | Fixed. Left at default and disclosed. |
| `SentryCliAuthFromEnv.cs` serves a CI path deferred to week two | Fixed. Deleted. The asset is gitignored instead. |
| Six policy edits where three are load-bearing | Partly accepted. The HTTPS edit is dropped as over-disclosure; the short-version and background edits stay because omission there is the honesty risk. |
| FGS declaration is not a Sentry item and blocks nothing that matters | Fixed. Parallel ticket. Crash reporting is decoupled from Play. |
| `release.yml` on the critical path | Fixed. Local build is primary. |
| Vitals / `DemoBuild.cs` symbol pin is scope creep | Fixed. Cut to week two; the asmdef cost is named. |
| `ProjectSettings.asset` change outside the squash | Fixed. Committed on the branch, rides the squash. |
| Probe branch created then deleted then re-created | Fixed. `feat/obs-sentry` is created today and kept. |
| PR review on `ProjectSettings/` blocks the merge | Accepted and stated: `main` is unprotected, CODEOWNERS is inert, merge is self-approved. |
| Solo owner, everything serialised | Accepted. Mitigated by two empty days and by front-loading the unbounded items. |
