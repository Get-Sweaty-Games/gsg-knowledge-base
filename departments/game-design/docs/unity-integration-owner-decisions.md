# Unity Integration — Owner Decisions (B1–B6)

> Filed from inbox `integration-decisions-unity-backend.md` §3 (owner decisions) and
> `partner-answers.md`'s B1 section. These are product/game-design calls made during the Unity↔
> backend integration — as opposed to the backend/API decisions, which are filed in
> `departments/engineering/docs/unity-backend-integration-decisions.md`.
>
> **This is a decision record, not a status document** — see the engineering doc's provenance note
> for why. Do not add status to this file.

## B1 — orientation — landscape everywhere

The project is landscape-locked end to end. The brief's shell is shaped like a portrait phone app, and
neither contract ever stated an orientation. Three options were on the table, plus a fourth "no bottom
nav bar at all" left unevaluated. The full option table and both sides' reasoning are preserved in the
Unity repo's issue #21.

**Game design's call: landscape everywhere, including the launch splash.** Deliberately not the
portrait-shell option, overriding the "logging a workout is one-handed, portrait" argument, on three
grounds: a second canvas configuration is a cost paid forever, since every future screen must decide
which frame it belongs to; nothing was built on the shell side, so there was no sunk portrait work to
preserve; and `RgSkin` / `ProcTex` and the whole procedural material system are authored
landscape-shaped throughout.

Reversible if a device build shows the claim flow is genuinely unusable in landscape. The seam would
move to the Game-tab boundary exactly as the portrait-shell option described, and the host absorbs it
for free either way.

**The portrait splash collapsed to a no-op.** There is no custom splash art, so the "splash" is Unity's
own logo and a portrait splash meant rotating that logo 90°. `BeforeSplashScreen` is the only hook that
runs early enough to affect it, and it renders *landscape*. Unity closed the Android rotation-flash
issue as Won't Fix (1–15 frame desync, worse under URP), and `AndroidMinSdkVersion: 26` puts Android
8/9 in the range Google confirmed cannot be fixed. Landscape-splash is the only clean outcome
available, not a compromise.

**One real gain identified and still owed:** assign the existing logo into `m_SplashScreenLogos` for an
actual branded splash. A player notices that, unlike a rotated Unity logo.

**A premise used in the discussion was wrong.** The reasoning assumed the host's `configChanges`
already absorbs a runtime `Screen.orientation` switch with no Activity restart. No manifest in the
Unity repo declared one, and the `.aar` declares no `<activity>` at all. It did not change B1's
outcome. **Do not reuse it as a premise elsewhere.**

### The full exchange behind the call (from the partner Q&A)

**Not decided unilaterally — deferred to a joint call, deliberately.** The host imposes nothing on
this choice, so it costs the backend zero either way:

- the manifest fragment declares no `android:screenOrientation` — the host takes no position and
  never has;
- its `configChanges` already lists `orientation|screenLayout|screenSize|smallestScreenSize|density`,
  so a runtime `Screen.orientation` switch is absorbed **without an Activity restart** (this premise
  about `configChanges` is the one flagged wrong above — the manifest that actually runs the host
  declares no `<activity>` at all, so this claim was never verified against the real manifest).

`docs/screen-flow.mermaid` (delivered in unity#18) does **not** answer orientation — it is a screen
inventory and navigation graph only, an omission rather than a decision. The doc now carries an
explicit note that orientation is deliberately unstated, plus a warning not to read its vertical
drawing direction as portrait intent.

**The two-postures argument that decided it:** logging a workout and claiming a reward is a
one-handed, standing-in-a-gym interaction — portrait. A dungeon run is lean-back — landscape. Backend
leaned toward the shell-boundary-switch option on this reasoning, which the studio's own
recommendation also favored; that reasoning, and "no sunk portrait work to preserve" plus "`RgSkin`/
`ProcTex` are landscape-shaped throughout," is what decided the call above in favor of landscape
everywhere instead.

## B2, B3, B5, B6

| # | Question | Decision |
|---|---|---|
| B2 | Two quantities both called "gold" (`RunState.Gold` vs `character.gold`) | **Do not rename `RunState.Gold`** — twelve baseline rows and roughly twenty call sites reference it. Document the distinction; change only player-facing copy. Already free in-dungeon: the purse displays struck coins, never the word "Gold" |
| B3 | Must the run loop work with no account and no network? | Keep a `GameRoot`-only entry path bypassing the shell, so the test-plus-AutoPilot ladder keeps working without a reachable backend. A sanctioned dev path, not a hidden bypass |
| B5 | Scope of the frozen guild surface | Build only the live routes — create/join/invite/members plus a roster fed by `/state`. **None of the frozen world-boss UI.** C10 (engineering doc) independently confirms the boss fields are permanently null |
| B6 | The workout→game hook | **Do not decide it.** The settlement seam is what makes deferring it cheap — that is the whole point of the seam |

## B4 — dissolved by C2

Brief-listed as blocked, because no bridge method existed to clear stored tokens. C2 (engineering doc)
shipped `clearAuthTokens()`. Settings can ship logout. **Do not carry B4 forward as blocked.**

## Related decisions

- "Nothing navigates between the shell pages, by design" is a game-design decision tracked alongside
  these — see `departments/engineering/docs/unity-backend-integration-decisions.md` § 4, Shell and
  REST.
