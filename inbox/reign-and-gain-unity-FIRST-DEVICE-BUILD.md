# Phase 8 — the first real device build

This is the last engineering-owned item in issue #34. Nobody has ever run this project on a phone.
You are the first, so every step below fails in its own way and each has a named symptom.

Every fact here is read from source.

**Citations name a symbol when the file is one we edit** — `DemoBuild.Run`, not `DemoBuild.cs:<line>`.
Line numbers are used only for data files that are not reordered by hand: `ProjectSettings.asset`,
`Bootstrap.unity`, `AndroidManifest.xml`. Every line citation in this document was wrong once, because
adding forty lines to `DemoBuild.cs` moved all six of them at once. Issue #256 is that bug.

---

## Before you start — what you need

1. An Android phone. `minSdk` is 26 (`ProjectSettings.asset:184`).
2. A USB cable, and `adb` on your PATH.
3. **Health Connect installed on the phone** (`com.google.android.apps.healthdata`), holding real data.
4. A real Google account signed in on the phone.
5. The Unity Editor **closed** if you build headless. Unity locks the project (`DemoBuild`'s header).

You do **not** need a keystore. `androidUseCustomKeystore: 0` (`ProjectSettings.asset:292`), so Unity
signs with its debug keystore and `adb install` accepts it.

You do **not** need `Assets/Resources/devtoken.json`. That file is for the Editor's simulate button only.

---

## Warning — read this before you touch anything

**The MCP bridge must be unplugged before any player build.** This is not optional and it is not a style
preference. It pulls an Editor-only assembly and a set of colliding BCL DLLs into the player, and the
build dies on them ten to twenty minutes in, naming none of them. Step 2 lists all three artifacts.

None of it is our code. None of it can be fixed from this repo.

**`./scripts/build-android.sh` does the unplug for you, and always restores.** Use it. Build by hand
only when it will not run, and then read its "Doing it by hand" list in full — it is longer than the
procedure people remember.

**If the build fails and the first errors name `UnityEditor` in `MainThreadDispatcher`, or a duplicate
`System.*` assembly, the bridge was not unplugged.** `DemoBuild.Run` logs that symptom itself when a build fails.

---

## Step 1 — Confirm the scene is shippable

**You do not need to check this by eye.** The build gate reads all six developer flags out of the scene
and refuses on any one of them. This step is here so you recognise the refusal, not so you pre-empt it.

The six flags sit in the `GameRoot` block at `Bootstrap.unity:153-159` and are all `0` on `main`.
That range is seven lines, not six — `progressionMode` is interleaved with them and the gate does not
read it:

```
headless: 0
autoPilot: 0
fakeGoogleSignIn: 0
fakeBrowserFallback: 0
simulateActivity: 0
validateSeedCount: 0
```

`DemoBuild.RefuseReason` reads the list in `DemoBuild.ShippableFlags`. A refusal names the flag and says
what shipping it armed would do. `Assets/Tests/EditMode/ShippableFlagsTests.cs` pins all six against its
own copy of the names, so removing one from the gate turns the suite red.

**The gate fails closed.** A flag it cannot read — renamed, deleted, or holding a value that will not
parse — is refused, not approved. Before `2026-08-09` an absent field read as "not set" and the build
went ahead.

Three of the six have **no runtime guard**, so the gate is the only thing stopping them:

| Flag | Guarded at runtime on device? |
|---|---|
| `fakeGoogleSignIn` | No. `SignInView.cs:198` is `Application.isEditor \|\| _fakeGoogleSignIn`. |
| `fakeBrowserFallback` | No, and `SignInView.cs:158` gives it precedence over `fakeGoogleSignIn`. |
| `simulateActivity` | Yes. `AppRoot.cs:652-656` forces it back to `Off` outside the Editor. |

---

## Step 2 — Build the APK

Close the Unity Editor. Then run one command:

```bash
./scripts/build-android.sh
```

That script replaces what used to be four manual steps: unplug the bridge, switch the build target,
build, and restore the repo afterwards. It restores on **every** exit path — a failed build, and a
Ctrl-C. Read its header before changing it; each thing it moves has a reason.

**Measured on the first real run, 2026-08-09:** `Succeeded in 00:43:06, 0 error(s), 1019 warning(s)`,
producing a **119 MB** APK. Most of that is the one-time asset reimport the platform switch forces.
Later builds are far shorter.

The build writes `Builds/Android/ReignAndGain.apk` (`DemoBuild.Android`). `Builds/` and `Logs/` are
both gitignored, so neither appears in `git status`.

It produces an **APK, not an AAB** — `buildAppBundle = false` (`DemoBuild.ApplyDemoSettings`). That is what
sideloading wants. Play will want the AAB, and that is not built yet.

Its exit code is Unity's own:

| Code | Meaning |
|---|---|
| 0 | Built. The path and size are printed. |
| 1 | The build failed. The `[BUILD]` and `error CS` lines are printed from the log. |
| 2 | `DemoBuild` refused the scene — a developer flag is armed. See step 1. |
| 3 | A precondition failed. The message says which. |

The most common code 3 is the Editor still being open. Unity locks a project it has open, and a headless
build against a locked project fails without ever mentioning the Editor, so the script checks
`Temp/UnityLockfile` first and refuses.

### What it moves, and why each one fails the build

Three artifacts, not the two the old procedure named:

| Artifact | If left in place |
|---|---|
| `com.ivanmurzak.unity.mcp` in `Packages/manifest.json` | Its Runtime asmdef has `defineConstraints: ["UNITY_MCP_READY"]`, and that define is set for all 20 player targets, so the assembly compiles into the player. `Runtime/Utils/MainThreadDispatcher.cs` has an unguarded `using UnityEditor;`. Hard `CS0246`. |
| `Assets/Plugins/NuGet/` | 42 DLLs, 27 marked `Any: enabled: 1, Exclude Editor: 1` — ship to the player, hide from the Editor. `System.Text.Json.dll` and the `Microsoft.Bcl.*` set collide with Unity's own BCL facades. |
| `Assets/packages-merged-link/link.xml` | Generated per-machine by the plugin. It names `ReflectorNet`, `McpPlugin` and `McpPlugin.Common` for preservation, and IL2CPP reads it during managed stripping — after those assemblies have stopped existing. |

The third was in neither this document nor `DemoBuild`'s header. A hand-run of the old procedure misses it.

**`Packages/packages-lock.json` is tracked, and Unity rewrites it** when it resolves without the package.
The script restores that file too. Restoring only the manifest would leave a tracked file dirty.

### Doing it by hand

Only if the script will not run.

1. Delete the `"com.ivanmurzak.unity.mcp"` line from `Packages/manifest.json`.
2. Rename `Assets/Plugins/NuGet` to `Assets/Plugins/NuGet~`. A trailing `~` makes Unity ignore a folder.
3. Rename `Assets/packages-merged-link` to `Assets/packages-merged-link~`.
4. Switch the build target: `File > Build Profiles`, select **Android**, click **Switch Platform**.
   `DemoBuild.Run` never calls `SwitchActiveBuildTarget`, so this is on you. It
   reimports every asset and takes a long time on a first switch.
5. Choose `Reign & Gain > Build > Android (sideload)`.
6. Undo steps 1 to 3, and `git checkout -- Packages/packages-lock.json`.

---

## Step 3 — Install on the phone

1. Connect the phone by USB.
2. Enable USB debugging on the phone.
3. Run this:

```bash
adb install -r Builds/Android/ReignAndGain.apk
```

4. Start logcat in a second terminal before you open the app:

```bash
adb logcat -s Unity:V
```

Every rung below reports itself to logcat. Watch it the whole time.

---

## Step 4 — Climb the ladder

The ladder is the rung list under **It is not device-proven** in `Assets/Plugins/Android/README.md`. Each rung fails differently, so climb them in
order and stop at the first one that fails.

### Rung 0 — read the *merged* manifest

Confirm `android:screenOrientation` and `android:configChanges` survived the manifest merge onto
`UnityHostActivity`.

1. Open `Temp/gradleOut/launcher/build/intermediates/merged_manifest/release/AndroidManifest.xml`.
   Android Studio's APK Analyzer reads the shipped one, which is better evidence.
2. Confirm `configChanges` still reads
   `orientation|screenLayout|screenSize|smallestScreenSize|density`.

Source declares it at `AndroidManifest.xml:12`. "Declared in source" and "survives the merge" are
different claims, and only the first has ever been checked.

**If it is stripped:** an orientation change destroys and recreates the Activity instead of absorbing
it. To the player that looks like the app relaunching.

### Rung 1 — the `.aar` is in the APK

**This rung needs no phone.** It was verified on 2026-08-09 and it passed:

```bash
python -c "
import zipfile
z = zipfile.ZipFile('Builds/Android/ReignAndGain.apk')
for n in [x for x in z.namelist() if x.endswith('.dex')]:
    if b'com/getsweatygames/reignandgain/UnityBridge' in z.read(n):
        print('FOUND in', n)
"
```

Android Studio's APK Analyzer shows the same thing with a UI. The first build reported 1133 entries,
two dex files and 6 `arm64-v8a` libraries, with `UnityBridge` in `classes.dex`.

If it is absent, the plugin never imported. Nothing below can work.

### Rung 2 — `UnityBridge.configure(...)` succeeds

`HostBootstrap.cs:101-102` calls it with four arguments read from `Resources/appconfig.json`.

**Failure symptom:** logcat reads
`[HOST] no appconfig, skipping configure() - silent refresh and native sign-in will both fail.`
(`HostBootstrap.cs:96-98`).

### Rung 3 — `UnityBridge.get()` returns non-null

This is the most likely first-session failure and it is silent.

**Failure symptom:** logcat reads, exactly once:

```
[HOST] com.getsweatygames.reignandgain.UnityBridge.get() is null - UnityHostActivity never registered
a bridge. Check the manifest fragment still names it LAUNCHER. Every host call is a no-op until it does.
```

That is `AndroidHostBridge.cs:173-175`. A null here means the manifest fragment did not merge. Every
host call degrades to a no-op afterwards, so the app keeps running and reports nothing.

### Rung 4 — `requestTodaySteps()` fires `OnTodayStepsRead`

This is the end-to-end proof: C# → JNI → Kotlin → Health Connect → `UnitySendMessage` → C#.

Before it can pass, grant the permissions:

1. Answer the runtime prompts for `POST_NOTIFICATIONS`, `ACTIVITY_RECOGNITION` and
   `ACCESS_FINE_LOCATION`. Each needs one tap.
2. Answer the Health Connect permission dialog.

**On Android 14 and above, the Health Connect dialog only reaches you through the
`ViewPermissionUsageActivity` alias** (`AndroidManifest.xml:69-78`). If that alias did not merge, the
permission screen never engages, `getGrantedPermissions()` stays empty forever, and `requestHealthRead`
becomes a permanent no-op with no error and nothing in logcat.

**If steps read as empty, check Health Connect's own UI before you suspect the bridge.** An empty
Health Connect returns nothing, which looks identical to a broken plugin.

---

## Step 5 — Test sign-in

**Caution — do not repeatedly dismiss the Google account sheet.** Credential Manager throttles repeated
dismissals into a silent, permanent sign-in failure (`SignInSession.cs:11-14`). If you dismiss it
several times while testing, sign-in stops working on that device with no error.

1. Tap sign in.
2. Choose a real Google account.

The Editor fakes this whenever `Application.isEditor` is true (`SignInView.cs:198`), so a device is the
only place the real path runs.

---

## Step 6 — Test a guild invite link

**App Link verification does not work on a sideloaded build.** The deployed `assetlinks.json` carries
three SHA-256 fingerprints, and your debug keystore is not one of them.

Test the custom-scheme filter instead (`AndroidManifest.xml:49-54`):

```bash
adb shell am start -a android.intent.action.VIEW -d "com.getsweatygames.reignandgain://invite?code=TEST"
```

The `https://getsweatygames.com/reignandgain/invite` filter (`AndroidManifest.xml:40-45`) will not
resolve, and it fails quietly because the custom-scheme fallback still works.

---

## Step 7 — Check the repo

The script already restored the bridge, on whichever exit path it took. It prints each thing it puts
back, ending with `Bridge restored.` Two things remain for you.

1. Reopen the Editor and let it re-resolve packages.
2. Run `git status --short`.

**An Android build rewrites tracked files that nobody authored.** The script reverts the three it knows
about and prints each one, then lists any other tracked file the build touched **without** reverting it
— an unrecognised change is something to look at, not something to throw away.

| Reverted automatically | What the build does to it |
|---|---|
| `ProjectSettings/UnityConnectSettings.asset` | Flips `m_Enabled` to `1`, silently switching **Unity Analytics on** for the whole project. |
| `Assets/Settings/Mobile_RPAsset.asset` | URP recomputes shader-prefiltering flags for the target platform. |
| `Assets/Settings/UniversalRenderPipelineGlobalSettings.asset` | Same — serialized shader-stripping state. |

A byproduct that was **already dirty before the build** is never reverted. The script cannot tell your
work from Unity's by content, so it uses "was this clean when I started" instead.

If `Assets/Fonts/Rubik SDF.asset` is dirty, run `git checkout -- "Assets/Fonts/Rubik SDF.asset"`. Play
mode wipes and rebuilds that dynamic atlas, producing a huge meaningless diff. The build itself never
enters play mode, so this only appears if you played the game.

**If `Packages/manifest.json` or `Packages/packages-lock.json` is dirty, the restore did not finish.**
That means the script was killed outright — `kill -9`, or the machine went down — because every other
exit path restores. Run `git checkout -- Packages/`, and rename `Assets/Plugins/NuGet~` and
`Assets/packages-merged-link~` back by hand.

---

## What to report back

For each rung, one line: passed, or the exact logcat text.

Rung 3's error line and rung 4's silence are the two outcomes worth capturing verbatim. They are the
failures this whole ladder exists to distinguish.

---

## Known-good baseline, for comparison

| Setting | Value | Source |
|---|---|---|
| Application id | `com.getsweatygames.reignandgain` | `ProjectSettings.asset:174` |
| minSdk | 26 | `ProjectSettings.asset:184` |
| targetSdk | 36 (pinned) | `ProjectSettings.asset:185` |
| Architectures | ARM64 only | `ProjectSettings.asset:276` |
| Scripting backend | IL2CPP | `ProjectSettings.asset:909-910` |
| Orientation | landscape only | `ProjectSettings.asset:63-66` |
| Bundle version | `0.1.0-demo` | forced by `DemoBuild.ApplyDemoSettings` |
| Backend | `https://api.reignandgain.getsweatygames.com` | `appconfig.json:2` |
| Scene | `Assets/Scenes/Bootstrap.unity`, the only one | `EditorBuildSettings.asset:8-10` |

`google-services.json` lives at `Assets/Plugins/Android/` and is gitignored (`.gitignore:24`), so it is
absent on a fresh clone and on CI. It is **not** required to build — `launcherTemplate.gradle` guards the
plugin behind an existence check, and without it push registration degrades to `OnPushTokenError`.

**Being present in `Assets/` was never enough, and that was the bug behind "this device could not
register for notifications".** The guard resolves inside the **launcher module**, and Unity copies
`Assets/Plugins/Android/` into `unityLibrary` — and treats a `.json` with a `TextScriptImporter` meta as
a TextAsset, so it reached neither module. Measured on the 2026-08-09 build: zero `google-services.json`
anywhere under `Library/Bee/Android/Prj`, and zero `google_app_id` entries in the launcher's `R.txt`. The
guard therefore failed **open** on every build ever made here — green log, dead push.

`GoogleServicesCopier` (an `IPostGenerateGradleAndroidProject` step) now copies it into `launcher/` and
**logs a warning when it cannot**, so a push-dead build no longer reads exactly like a working one.

---

## One thing that ships that you should know about

Naming any hero `$devskill$` on character select unlocks the entire skill tree. It is in
`Assets/Scripts/Bootstrap/DevCheats.cs`, behind `#if UNITY_EDITOR || DEVELOPMENT_BUILD`.

`DemoBuild.Run` builds with `BuildOptions.None` and never `Development`, so a release
player compiles an empty method and the cheat is **not** in this APK. It is in any development build.
