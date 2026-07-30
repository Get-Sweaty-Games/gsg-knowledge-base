# Unity-as-base topology — inversion analysis

> **Status: ADOPTED 2026-07-28.** Written 2026-07-27 asking whether the client topology could be
> inverted — Unity as the app process, Kotlin as a native plugin underneath it — instead of the then
> UaaL model (Kotlin host embedding Unity as a library). Reviewed when the engine decision closed in
> Unity's favour (2026-07-28) and **rejected**; reversed **the same day** when a partner studio took
> ownership of the whole Unity client. **We are on Unity-as-base.**
>
> Two superseded layers are preserved below rather than deleted — this doc's standing convention, and
> the reason it survived the first rejection: a reasoned position is worth more than a gap, and the
> analysis of *what would have to move* stayed accurate through both reversals. Read in order:
> the original 2026-07-27 argument → § *Decision (2026-07-28): DECLINED* (superseded) →
> **§ *Reversal (2026-07-28): ADOPTED*, which carries the current position.** Nothing between here
> and the reversal section has been edited to match the outcome.

## The two topologies

**Today (UaaL — Unity-as-a-Library):** the Kotlin host is the Android process. `UnityHostActivity`
subclasses Unity's `UnityPlayerGameActivity` and embeds the Unity player as a component the host
boots and owns the lifecycle of. Structurally, Kotlin is on top; logically, it's inverted —
Unity was deliberately given the *app* layer (networking, rendering, evidence-bundle assembly,
all game logic per `docs/PLAN-unity.md` L26/L75), while Kotlin is scoped to platform-only
primitives (Health Connect reads, OS permissions, Keystore, token refresh, push). See
`docs/unity-bridge-contract.md` L18-20: *"The host never decides anything — it supplies raw
signals + a stored token and fulfils permission/push requests."*

**The inversion (Unity-as-base):** Unity's own build generates the whole Android app —
`AndroidManifest.xml`, the player Activity, the APK/AAB. Kotlin code ships as a native Android
library (`.aar`) that Unity's Gradle project depends on as a plugin — the traditional way Unity
mobile games have integrated native SDKs (Firebase, AppsFlyer, ironSource) for years, predating
UaaL. UaaL exists specifically for the opposite need: embedding game content inside a *larger*
native app that has substantial native-only screens outside the game.

## Why this project is a candidate for the inversion

Per `docs/SPEC-behavior-derived.md` § Screen surface, this app has **18 screens, all of them the
game** (Home/Roster/Game/Guild tab bar) — there is no native-only screen surface sitting alongside
Unity content. That's exactly the case UaaL isn't built for. `docs/PLAN-unity.md` L10 only ever
evaluated UaaL against third-party wrapper libraries (`flutter_unity_widget`,
`@azesmway/react-native-unity` — rejected for maintenance-cliff risk); "Unity as base + Kotlin
plugin" was never on the table as a labeled alternative in that decision.

## Complexity estimate: moderate, bounded — not a rebuild

The wire protocol doesn't care which side owns the process. Both bridge directions
(`docs/unity-bridge-contract.md` L11-20) already use engine-agnostic Android mechanisms:

| Direction | Mechanism | Changes under inversion? |
|---|---|---|
| Unity → Kotlin | JNI (`AndroidJavaClass`/`AndroidJavaObject`) | No — works identically reaching into a plugin AAR |
| Kotlin → Unity | `UnityPlayer.UnitySendMessage` → `HostBridgeReceiver` GameObject | No — Unity-SDK-provided, unaffected by process ownership |

None of the following need to change: the bridge contract itself, `toUnityJson()` /
`toUnitySourcesJson()` (`RawHealthSignalsJson.kt`), `HealthConnectReader.kt`,
`SecureTokenStore.kt`, `TokenRefresher.kt`, or any Unity-side C# game/networking logic.

### What actually has to move

Only the pieces that assume Kotlin owns the Activity lifecycle and the manifest:

- **`UnityHostActivity.kt`** (204 lines) — subclasses `UnityPlayerGameActivity` today; its
  `onCreate` is where `UnityBridge.register(...)` fires, where deep links are parsed
  (`handleDeepLink`), and where Google Sign-In (`beginGoogleSignIn`) kicks off. Under inversion,
  Unity's own generated Activity becomes the entry point, so this logic gets re-homed as calls
  Unity's C# side makes into the plugin at the right lifecycle moments, rather than the plugin
  owning `onCreate`.
- **`AndroidManifest.xml`** already telegraphs this exact seam. The OAuth and invite-link
  `<intent-filter>`s currently live on `DevDriverActivity`, with a code comment noting they need
  to move back to `UnityHostActivity` once the dev-launcher swap-back happens. Under inversion they
  move once more — onto whatever Activity Unity's build generates — via the plugin AAR's manifest
  merge, the standard mechanism every Unity native plugin already uses.
- **`GoogleSignIn.kt`** (63 lines) — activity-result handling needs to attach to Unity's activity
  instead of a custom one; the sign-in logic itself is unchanged.
- **`DeepLink.kt`** (88 lines, `android-host/hostlogic/`) — **does not move.** It's pure Kotlin
  with no Android dependency, already isolated from any Activity — exactly why the project put it
  there in the first place.

### What doesn't change, and why it matters

The host-owned-refresh-token trust design (`docs/SPEC-behavior-derived.md` § Auth, `CLAUDE.md`) is
untouched. Its whole point is that the refresh token never enters Unity's managed/decompilable
(IL2CPP/Mono) memory. A Kotlin plugin AAR is still native code outside that runtime, so
`SecureTokenStore`/`TokenRefresher` keep the same guarantee, unmodified, regardless of which side
owns the process.

## Decision (2026-07-28): DECLINED — we stay on UaaL *(SUPERSEDED same day — see § Reversal)*

The engine question closed in Unity's favour on 2026-07-28, which is when this proposal got its
review. It was rejected. The reasoning matters more than the verdict, because the original argument
above is not merely unproven — **its central mechanism claim is wrong**, and that is worth recording
so the same reasoning doesn't get re-derived later.

### The memory argument doesn't hold

The § Relevance argument as originally written attributed the ~80–180MB floor to *"UaaL's
dual-runtime embedding (a full native host process on top of a full Unity player as a subordinate
library)."* **There is no dual runtime and no second process.** IL2CPP is ahead-of-time-compiled
native code, not a VM; the Kotlin/AndroidX classes load into the same ART instance that Unity's own
Java layer requires under *either* topology. `libil2cpp.so` (~78MB) and
`src/main/assets/bin/Data/data.unity3d` (16,730,126 bytes) are resident identically either way.

Nor is there duplicated packaging to reclaim: `app/build.gradle.kts:158-171` already declares
`unity-classes.jar`, `games-activity`, and `appcompat` as `compileOnly` precisely because
`unityLibrary` ships them at runtime — the comment there says so outright ("compileOnly avoids
packaging them twice").

What the inversion actually changes is which Activity subclass is the entry point and who owns the
manifest merge. That is worth single-digit MB, not 80–180.

### Three reasons it is actively worse here

1. **It deletes the only lever we have against the floor.** `UnityHostActivity.kt:40-45` and
   `PLAN-unity.md` L107-113 both name the mitigation for a fatal memory floor: *tear-down-on-background*
   rather than always-resident embedding. That requires the host to own `onCreate` and to decide when
   the player exists. Under Unity-as-base the app **is** the player — you cannot not-load it. The
   inversion does not sidestep the floor; **it removes the escape hatch.** This is the single
   strongest argument against, and it inverts the doc's own thesis.

2. **The targetSdk asymmetry runs against it, hard.** `:app` sets compileSdk/targetSdk **36**
   (`app/build.gradle.kts:36,42`); `unityLibrary` is **35** (`unityLibrary/build.gradle:19,32`).
   Under UaaL the application module is ours, so our 36 wins the merge and we control the bump.
   Under inversion the application module is Unity-generated and targetSdk comes from Player
   Settings — i.e. from the Editor's release cadence. This is a health-data app chasing Play's annual
   targetSdk deadline, carrying an `activity-alias` whose own comment says *"Required as long as
   targetSdk >= 34"* (`AndroidManifest.xml:138-152`). Handing Play-compliance timing to the engine
   vendor is a materially worse trade than an unmeasured memory number.

3. **"What actually has to move" (above) is materially incomplete.** It names three files and omits
   the entire Gradle configuration surface that lives in `:app`: 6 `buildConfigField`s, 4
   `manifestPlaceholders`, the release signing config, the conditional `google-services` apply, the
   `assembleRelease` `GOOGLE_WEB_CLIENT_ID` guard (`app/build.gradle.kts:187-198`), the Firebase BOM,
   and `implementation(project(":hostlogic"))`. Kotlin reads `BuildConfig.*` in three files —
   `UnityHostActivity.kt` (×4), `TokenRefresher.kt` (×4), `DevDriverActivity.kt` (×15+). Inside an
   `.aar` those resolve to the **library's** `BuildConfig`, not the app's, so every read needs
   re-plumbing through `mainTemplate.gradle` / `launcherTemplate.gradle` or conversion to injected
   parameters. That is the work the doc above calls "moderate, bounded."

### What the proposal got right

The observation in § Why this project is a candidate stands and should not be dismissed: UaaL's
design intent genuinely is native-app-with-embedded-game, and this app is 18 screens all of which are
the game. **On a greenfield project, Unity-as-base would be the defensible default.** This is not a
greenfield project — the host, the bridge, the manifest, and the whole Gradle surface already exist
and already work. The proposal is right about the ideal and wrong about the migration.

### Revival trigger *(as written at the time of the rejection — superseded)*

Deliberately **not** a memory measurement. "Revisit if we measure a bad number" would be theatre: by
argument (1), a bad number sends us toward tear-down-on-background, which *requires* UaaL. A bad
memory result argues **for** the current topology, not against it.

> **Revive this only if a Unity feature or package hard-requires Unity's own generated launcher
> Activity** — i.e. something that cannot function when `UnityPlayerGameActivity` is subclassed by a
> host-owned Activity. Memory and cold-start are explicitly **not** revival triggers.

---

## Reversal (2026-07-28): ADOPTED — we move to Unity-as-base

Reversed the same day it was declined. **The trigger above is not what fired**, and saying so
plainly is the point of this section: nothing about Unity's packaging changed, and no Unity feature
turned out to require the generated launcher Activity. Anyone re-deriving the decision from the
rejection alone will reach the wrong answer, because the input that changed is not in it.

**What changed: team topology.** A partner studio now owns and builds the entire Unity client in its
own repo (`Get-Sweaty-Games/reign-and-gain` — a complete turn-based dungeon-crawler, its own CI, its
own Unity version). The application module should live where the client lives. Our own `unity/` tree
was a thin app shell of ~34 scripts, was judged disposable by the founder, and is deleted rather than
migrated. That removes the migration cost the rejection was largely protecting, because the thing
being thrown away is precisely the thing the inversion would have had to reshape.

Note what this does **not** change: the rejection's technical reasoning about `android-host/` was
correct when written and is still correct. The decision flipped on an input, not on an error.

### Disposition of the three rejection arguments

**(1) "It deletes the only lever against the memory floor" — NEUTRALISED, not accepted as a cost.**
This was rated the strongest argument, and it turns out to be avoidable. Unity supports overriding
the generated launcher via a manifest fragment in `Assets/Plugins/Android/AndroidManifest.xml` — the
same merge mechanism § What actually has to move already identified, applied one step further. So
`UnityHostActivity` ships **inside** `hostbridge.aar`, still subclassing `UnityPlayerGameActivity`,
and the partner's fragment names it as the launcher. The host keeps `onCreate`, so
tear-down-on-background remains available. `handleDeepLink`, `UnityBridge.register`,
`beginGoogleSignIn` and the notification-permission flow do not move and are not re-plumbed. The
original argument was wrong on the mechanism, not on the principle — the principle (never give up the
lever) is upheld, by a route the argument did not consider.

**(2) "targetSdk asymmetry hands Play-compliance timing to the engine vendor" — STANDS, accepted with
mitigation.** Not refuted. Under Unity-as-base the application module is Unity-generated and
targetSdk comes from Player Settings. For a health-data app on Play's annual deadline, carrying an
`activity-alias` whose own comment reads *"Required as long as targetSdk >= 34"*, this is a real
cost. Mitigation: the partner pins **targetSdk 36 explicitly** in Player Settings — their project is
`AndroidTargetSdkVersion: 0` (auto/highest) today, which is worse than either fixed value because it
moves without anyone deciding. The requirement is carried in `docs/unity-client-brief.md`. This is
recorded as a known accepted cost, not a solved problem.

**(3) "The migration is the whole `:app` Gradle surface" — mostly evaporated.** `OAUTH_REDIRECT` and
`INVITE_BASE_URL` stay as library `BuildConfig` fields: they were already hardcoded in the build file
rather than sourced from `local.properties`, and `handleDeepLink` needs them at cold start, before
any C# exists to inject anything. Only **three** values need runtime injection —
`GOOGLE_WEB_CLIENT_ID`, `SUPABASE_URL`, `SUPABASE_PUBLISHABLE_KEY` — via a new
`UnityBridge.configure(supabaseUrl, supabasePublishableKey, googleWebClientId)`. The `×15+` reads
that made this argument look heavy are all in `DevDriverActivity`, which stays in an application
module we keep as an on-device host harness. The signing config, the `google-services` guard and the
Firebase BOM go with it.

### What is not proven

Everything above is design, not evidence. At the time of writing: `hostbridge.aar` has not been
built, delivered, or loaded by anything; the manifest-fragment launcher override is the standard
documented mechanism but has not been exercised on this project; `unity-classes.jar` as a committed
`compileOnly` stub is untried; and the partner integration has not happened. Treat the topology as
decided and unverified until `docs/PLAN-closeout.md` Phase 5 closes its on-device gate.

### Revival trigger (current)

Symmetrically to the original, **not** a memory measurement — under the launcher-override the
tear-down lever is retained either way, so memory now argues for neither topology.

> **Return to UaaL only if the partner arrangement ends and we own the client again, or if the
> product grows substantial native-only screens outside the game** — the case UaaL is actually
> designed for. A targetSdk or Play-compliance incident is a reason to renegotiate who pins Player
> Settings, not a reason to move the app module back.
