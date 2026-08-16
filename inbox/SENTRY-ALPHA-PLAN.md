# Sentry crash reporting — alpha rollout

**Status: shipped, released, and verified. All 5 GO/NO-GO conditions passed 2026-08-13.** GO was
called the same day; versionCode 296 (`65fc859`) is **live on Play's internal track**, confirmed by a
direct read of the track state (`status: "completed"`, not just uploaded).

## What's live

- Sentry Unity SDK 4.8.0, pinned via git URL in `Packages/manifest.json` (not OpenUPM — tops out at
  1.8.0; not the `.unitypackage`, which drags in ~65MB of `sentry-cli` binaries).
- Org in the **EU region** (`de.sentry.io`) — fixed at org creation, not reversible. DPA accepted.
- DSN configured in `Assets/Resources/Sentry/SentryOptions.asset` (committed), verified end to end
  with a real crash.
- Ships to the team by **sideload**, not Play — the app is already installed there and Google's
  review queue must not gate crash reporting.
- **Also live on Play's internal track**, versionCode 296 (`65fc859`), release notes "Introducing
  monitoring by sentry. Bunch of UI and game improvements." First `edits:commit` attempt 403'd on the
  unanswered Foreground Service declaration, exactly as predicted by the GO/NO-GO plan; the
  declaration was then answered directly in Play Console (no git trace by nature) and a second
  `scripts/upload-play.js` run committed cleanly. **Verified independently**, not just from the
  script's own output — a read-only `edits.tracks.get` call returned
  `{"track":"internal","releases":[{"versionCodes":["296"],"status":"completed"}]}`.

  **Open discrepancy:** issue #363 (tracks capturing the two `adb screenrecord` demo videos Google's
  form can ask for) is still **open** on GitHub even though the console declaration was clearly
  answered — the commit would not have succeeded otherwise. Either the video evidence wasn't required
  this time, or it's still owed. Don't close #363 on the strength of this alone — check what was
  actually submitted in Play Console first.

### Configuration (`Assets/Resources/Sentry/SentryOptions.asset`)

| Setting | Value | Why |
|---|---|---|
| `CaptureLogErrorEvents` | `false` | Load-bearing. Gates only `LogType.Error`/`LogType.Assert` — unhandled exceptions and `Debug.LogException` use a separate integration and are unaffected. Left on, the 5,000-event/month free tier empties from routine `Debug.LogError` noise. |
| `MaxBreadcrumbs` | `100` | Not zeroed. The leak this would have prevented (log strings riding in breadcrumbs) is closed at the source instead, by the repo's `BreadcrumbsFor*` flags — so the lifecycle/scene/system breadcrumbs survive. Verified on a real payload: 8 breadcrumbs, all lifecycle/scene/system, zero log strings. |
| `SendDefaultPii` | `false` (default) | No email, username, device unique id, machine name. |
| `AutoSessionTracking` | `true` (default) | Crash-free session rate; disclosed in the policy rather than turned off. Not billed against the error quota. |
| `TracesSampleRate` | `0` | No performance quota spent. |
| `AttachScreenshot` / `AttachViewHierarchy` | `false` | A screenshot of a fitness app can contain health values. |
| `Il2CppLineNumberSupportEnabled` | `true` (default) | Adds `--emit-source-mapping` to `ProjectSettings.asset` — the one line a release build is expected to dirty. |

**No user is ever set.** Nothing calls `SentrySdk.ConfigureScope`/`.User =`, enforced by a source-grep
test — the SDK mints a random per-installation id instead (expected, not a leak).

**No `BeforeSend` hook ships.** A culprit-keyed hook risks suppressing real crashes and doesn't run on
native crashes at all; the source edits below close the leak instead.

**Editor events never reach Sentry**, two layers: `CaptureInEditor=0` (load-bearing — the SDK doesn't
init at all in the Editor) plus `Assets/Scripts/Bootstrap/SentryOptionsConfig.cs`'s `SetBeforeSend`,
which drops anything tagged `environment=editor` as a second guard.

**Seven source edits close the log-message leak** (a `JsonException` message embeds the offending
JSON token — could be a health field name or value):
- 5 log sites changed `{e.Message}` → `{e.GetType().Name}`: `Host/HealthReadSession.cs:106`,
  `Host/PresenceSnapshotSession.cs:148`, `Host/RunSession.cs:182`, `AppUI/AppRoot.Activity.cs:87` and
  `:103`.
- 2 unguarded parses wrapped in try/catch, same rule: `Host/HostBridgeReceiver.cs:48`,
  `Net/SupabaseAuthClient.cs:217`.

Pinned by `Assets/Tests/EditMode/SentryConfigTests.cs`.

### Compliance

- IP suppression works (org-level scrub on `$user.ip_address`).
- **Geo scrubbing needed a fix.** `$user.geo` names an object — Sentry applies a rule matching zero
  nodes *silently*, with no error. The working selector is **`$user.geo.**`** (matches the leaf
  fields). Verified on a real payload after the fix: `country_code`/`city`/`subdivision`/`region` all
  `null`. **Lesson: a Sentry Console control isn't verified until a real payload agrees with it.**
- The auth token lives only in the gitignored `Assets/Plugins/Sentry/SentryCliOptions.asset`
  (per-machine — carries the token and the project slug together, no way to split them). **CI has no
  token, so a CI build uploads no symbols** — accepted for now, every build this week is local.
- **The Sentry project slug in that same asset must read `reign-and-gain-unity`.** A stale slug (from
  before the repo rename) fails the release build ~35 minutes in, at the very end, in Gradle.

### Privacy policy

`gsg-landingpage/app/src/sites/reign-and-gain/pages/PrivacyPage.tsx` — six edits (analytics/SDK
clause, sub-processor list, retention wording, short-version summary, background-sync sentence, new
legal-basis/right-to-object section). Live, verified by fetching the hashed JS bundle directly
(grepping the page root returns nothing — it's a bare Vite SPA shell). `LAST_UPDATED` bumped to
August 11, 2026.

**Vercel builds from a personal fork** (`yarins0/get-sweaty-games-landing-pg`), synced by
`.github/workflows/sync-fork-on-merge.yml`. That sync has silently failed before while the dashboard
read "Ready" — always re-verify with the bundle-hash fetch, never trust the dashboard alone:
```bash
curl -s https://getsweatygames.com | grep -o '/assets/index-[^"]*\.js'
```

## GO/NO-GO — all 5 conditions passed 2026-08-13

| # | Condition | Result |
|---|---|---|
| 1 | Privacy policy live | Verified against the deployed bundle. |
| 2 | EditMode suite green | 1526/1526 on `514b057` originally; **re-verified 1534/1534 on `65fc859`**, Editor closed. |
| 3 | AAB under 150MB, both native Sentry libs present | v288, 128.8MB, `libsentry.so` + `libsentry-android.so`, arm64-v8a only. |
| 4 | A real managed C# exception reaches Sentry with a stack trace | Proven on a throwaway branch (never merged), event `8c631290…3661`: full managed stack trace, `in_app` frames, symbolicated to `GameRoot.cs`. `user.email`/`username` absent, `user.id` = installation id only, no health data. |
| 5 | Compliance controls verified against a real payload, not just the Console | IP + geo scrubbing both confirmed null on a real event; DPA accepted; editor events filtered. |

**Soft condition, also met:** file-and-line symbolication works — confirmed on the same
throwaway-branch event, resolved to `GameRoot.cs:11746`.

## What's NOT done

- **PlayMode suite never run.** AutoPilot (full run, inside the Editor) passed instead — stronger
  evidence for the runtime init-order question this was meant to answer, but the PlayMode suite itself
  is still outstanding.
- **Not yet sideloaded to the team's devices.** Only the owner's device has the release build.
- **Foreground Service Play-Console declaration itself is answered** — the release commit above would
  have 403'd otherwise, and the second attempt didn't. **Issue #363 (the `adb screenrecord` demo
  captures) is still open regardless** — see the discrepancy note under "What's live." Don't assume
  it's safe to close without checking what was actually submitted.
- **`contexts.app.permissions` discloses all 7 Health Connect permission names (as `not_granted`) in
  every event payload.** Not a leak by itself (the Play listing already discloses the permissions),
  but it's a "decide deliberately" item, not acted on.
- **Lawyer review of the Article 9 health-data question — not started.** Legitimate interest is *not*
  a valid basis for special-category (health) data under Article 9; whether any of this data could
  reach a crash report at all needs a real answer before public launch. Deliberate deferral for the
  alpha (team + small invited cohort, EU hosting, no account id at Sentry, signed DPA).
- **Written legitimate-interest assessment — not written.** The owner's own task, ~2 hours.
- **No persisted consent surface.** `PermissionGate.cs` is an explainer modal with no persisted choice
  or Settings toggle — can't be retrofitted onto crashes already collected. Accepted ceiling of the
  legitimate-interest approach for now.
- **Play closed-testing (12 testers × 14 days) — not started**, queued behind the FGS review, which
  has no timeline Google will commit to.

## Team device rollout — the one gotcha

**Play-installed builds and locally-built release builds sign with different keys**, so
`adb install -r` fails (`INSTALL_FAILED_UPDATE_INCOMPATIBLE`) on any device whose app came from Play.
The device must be **uninstalled first**, which wipes local state — sign-in tokens (server-side
progress is untouched). Say this in the tester instructions before sideloading, not after someone
hits it.

## Rollback

- **Before any merge to `main`:** the work was never isolated to a single unmerged branch — it landed
  via ordinary PRs (`a525158` SDK install, `da55c49` bundled with other fixes) ahead of the gate. A
  one-SHA revert does not exist; a revert PR would need to be written deliberately.
- **Fast, no revert:** set `Enabled = false` on `SentryOptions.asset` and rebuild. **Verified this also
  stops the native layer** (A/B/A logcat test on device — 2 `nativeloader … libsentry*.so` loads when
  on, 0 when off). No need to also touch `AndroidNativeInitializationType`.
- **Full removal**, if ever needed:
  ```bash
  jq 'del(.dependencies["io.sentry.unity"])' Packages/manifest.json > tmp && mv tmp Packages/manifest.json
  rm -rf Assets/Plugins/Sentry Assets/Plugins/Sentry.meta Assets/Resources/Sentry Assets/Resources/Sentry.meta Assets/Tests/EditMode/SentryConfigTests.cs Assets/Tests/EditMode/SentryConfigTests.cs.meta
  ```
  The `rm` of `SentryConfigTests.cs` is required — it references Sentry types directly and would fail
  to compile otherwise. Confirm EditMode is still green after.
- **Nothing rolls back in `gsg-landingpage`.** A policy that discloses a crash reporter you've since
  removed is over-disclosure, not a risk.

## Traps, for next time

- The release build path always writes to `Builds/Android/ReignAndGain.aab` — back up the real
  artifact before building a throwaway one.
- An AAB can't be `adb install`ed directly. `bundletool build-apks --mode=universal` with the real
  keystore, then extract `universal.apk`, then `adb install -r`.
- A thrown managed exception from inside a `UnityEvent` listener does **not** crash the app —
  `UnityEvent.Invoke()` catches it and routes it through `Debug.LogException`, a separate SDK
  integration from `CaptureLogErrorEvents`. Check Sentry, not for a visible crash.
- MIUI's "USB debugging (Security settings)" toggle (needed for `adb shell input tap/keyevent`) stayed
  inactive despite enabling plain USB debugging — may need a time delay or a Mi account link. Manual
  taps work fine as a fallback.
