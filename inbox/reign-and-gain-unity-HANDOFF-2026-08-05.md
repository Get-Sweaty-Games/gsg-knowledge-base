# Handoff — 2026-08-05

Written at the end of a long session so the next agent can pick up without re-deriving. Read this
before touching the card system or the skill tree.

**Branch:** `fix/knight-art-and-heal` → **PR #151** (open, not merged, wants review — it touches the
art pipeline, which is shared).
**Tests:** 551/551 EditMode green at `3d68296`. Working tree clean apart from three files of editor
churn that were dirty before this session (`Rubik SDF.asset`, the TMP fallback, and
`ProjectSettings.asset`, where Unity keeps adding `SENTIS_ANALYTICS_ENABLED;APP_UI_EDITOR_ONLY` to the
WebGL defines). None of those are ours; decide deliberately rather than sweeping them in.

---

## 1. The thing that is NOT done: card-system polish

**The owner's verdict on the current state is "we are getting there but it's far from polished to my
standards."** That is the live task. Do not treat the card system as finished.

### What exists now

The between-rounds deal has been through **three shapes in one day**, each answering a real complaint:

| | shape | why it was replaced |
|---|---|---|
| 1 | per-hero fan, face-up | three sequential prompts pretending to be a choice |
| 2 | static table of 3 rows | all 15 visible but a dialog; the taps did nothing |
| 3 | **one line of backs crossing the screen** | current |

Today's shape: the party's whole deck (15 cards, or 12 on a reshuffle) sweeps across the screen as a
single horizontal stream of face-down backs. One card per hero peels out **while the line is still
moving** and flies into that hero's slot, turning over on the way. No panel, no plaque. Border colour
per hero is the only thing distinguishing cards. Tap anywhere to skip.

Code: `CombatView.BuildDealLine` / `SweepDealLine` / `PluckDealCard` / `PluckRoutine`, driven by
`CombatController.DealPicks`.

### What was unfinished — and what a later session on the same day did about it

Both of the items below were the top two entries on this list. They are **done**; the list is kept
because the reasoning is still the best summary of what the deal and the picker are for.

- **~~THE FOCUS PICKER WAS ASKED FOR AND IS NOT BUILT.~~ BUILT.** The ask was: cards float face-down,
  then **rotate to face the player** so a card can be chosen knowingly. `CombatView.ShowFocusPicker` is
  now that — a fan of five real cards that floats up face-down, holds a beat, and turns over left to
  right, readable in ~1.2s. The vertical list of labelled buttons and its plaque are gone. Its numbers
  are `FocusCard*` / `FocusRise*` / `FocusFlip*` in `CombatView`. See VERIFY-WITH-OWNER #85-#87.
- **~~I never watched the flying line run.~~ WATCHED, AND IT WAS BROKEN.** The warning was right and the
  bug was worse than a wrong-looking animation: **the deck never crossed the screen at all.** Probed
  live, the picker was switched off at t+1.0s with all fifteen cards still spanning screen x 2184..3432
  on a 2560-wide view — off the RIGHT edge, never having arrived. Two independent causes:
  1. `SyncRoundPicker`'s signature included `Picked[i]` and `Cards[i].SkillId`, both of which change the
     instant the engine takes a card — and the controller calls `Refresh` right after every `AutoPick`.
     So the line was destroyed and rebuilt from off-screen **three times inside one 1.3s deal**.
  2. The view's sweep was a flat 2.1s while the controller's beats summed to 1.33s. The comment claiming
     the beats "sit inside the view's sweep" was true and was exactly the bug.
- **The two clocks are now one.** `CombatView.DealWatchBeat` / `DealPickBeat` are the single source and
  `CombatController` reads them; `CombatView.SweepDurationFor(pickers)` derives the crossing from them
  plus an exit tail. Verified across four consecutive deals: on screen at t+0.55 while cards are plucked,
  fully off the LEFT edge (-1203..-128) by the close, every deal closing at t+1.73s = `SweepDurationFor(3)`.
- Still guessed, not tuned: the beats themselves (0.55 / 0.34 / 0.5), `PluckDuration`, `PluckArc`, the
  deal's vertical band, and the card overlap. VERIFY-WITH-OWNER #81-#84 lists them for the owner.

### How to look at it

The deal is only up for a couple of seconds per round, so racing it with screenshots does not work.
Hook `EditorApplication.update`, watch `CombatView.IsRoundPickerOpen`, count game frames from when it
opens, then set `EditorApplication.isPaused = true`. That is how the invisible-card bug was found.

**Three traps in that technique, all paid for on 2026-08-05. Read these before using it.**

1. **`editor-application-set-state` with only `isPaused` STOPS PLAY MODE.** `isPlaying` defaults to
   false, so unpausing that way kills the session. Always pass both flags.
2. **Never take more than one sample per animation.** The first frame after a resume carries up to
   `maximumDeltaTime` (0.333s) of game time, so stepping through marks inside a single deal
   fast-forwards the very thing being measured — it made a working sweep look 20x too slow. Take ONE
   sample per deal and let the next round provide the next sample.
3. **Editor-side `EditorApplication.update` hooks installed by `script-execute` SURVIVE entering and
   leaving play mode**, because this project has domain reload on play disabled. Old probes keep firing,
   pausing the editor and fighting the new one. Recompiling a `.cs` file is what actually clears them.

**And the trap specific to looking at the Focus picker: `AutoPilot` cancels it on sight**, by name, which
is correct and documented. Destroying the `AutoPilot` *component* does not stop it — `AutoPilot.Run` is
started with `StartCoroutine` on **GameRoot**, so the coroutine keeps running on a live host with a
destroyed `this`. `GameRoot.StopAllCoroutines()` is what stops the pilot. An hour went into "why does the
picker close after 0.1s" before that was the answer.

---

## 2. Also shipped today (all in PR #151)

- **The fat knight's death.** `fat knight death.png` was a **byte-for-byte copy of `fat knight hit.png`** —
  a real 29-frame clip that simply was not a death, so the boss stood upright through his own defeat and
  every frame-counting test passed. Owner supplied the real sheet. `ArtSourceSheets.FindDuplicateSheets`
  now hashes sheets per folder and the conditioner warns on a collision.
- **His attacks ship under real names** (`attack`/`attack2`/`attack3`) instead of runtime aliases; `hit 2`
  had been importing as a hurt *variant* and was completely unreachable. His walk was importing at 25 of
  28 frames (art in the padding cells).
- **`AtlasImporter` no longer re-compresses the whole roster.** It dirtied every page unconditionally;
  one character's re-import cost ~4 minutes of blocked editor. Now compares first: **80ms**.
- **Statuses kept, not removed** (owner reversed the earlier "delete them" decision). Enemy DoTs now tick
  **before** the enemy phase, so a burning enemy loses its last swing. Ticks are named (`BURN 3`), at
  normal damage size, one at a time.
- Target ring lights on target *selection*, not after the QTE. Locked heart slots are dots, not faint
  hearts. Mage heal plays into full health. Boss is "Evil Knight" on both costumes. Armed cards wear a
  cream rim. Deck pile opens a browser of the three 5-card decks.

---

## 3. Balance: the fight got much easier today, by accident

Three changes each favoured the player and they stack. Measured over 80 seeds
(`SeedBaselineTests.TheClearRate_StaysInAPlausibleBand` logs it):

| after | clear rate |
|---|---|
| start of day | 33.8% |
| DoTs kill before the enemy swings | 38.8% |
| reshuffle offers 4 (can't return the card you replaced) | **45.0%** |

**None of these were balance decisions.** An 11-point swing wants the owner's eyes before anyone does
deliberate difficulty work. Read the rate off the log, never off the twelve pinned seeds — twelve is far
too small a sample and has misled twice already.

---

## 4. Open decisions and open work

**The big one — `VERIFY-WITH-OWNER.md` #73.** The skill tree is built and reachable, but **nothing
outside it reads progression**: `GameRoot` passes it to neither the deck draft nor the combat engine, so
buying a skill changes no fight and no deck. Both seams exist and accept the data. Wiring it drops every
deck from 5 cards to 3 — a large balance change on top of §3. **OWNER DECIDED 2026-08-05: wire it, keep
the real new-player path as the default, and build a dev grant mechanism instead of defaulting to it.**

### The endless loop (OWNER DIRECTION, 2026-08-05 — design agreed, not built)

There is one map and there will not be another for the MVP, so the run becomes **a real roguelite loop**:
clearing the boss does not end the run, it **re-enters the same map with the nodes procedurally re-ordered
and the difficulty raised.** Endless, on one map. This is also what the `1-1 / 1-2 / 2-1` node labels are
for — loop number and row.

It fits the painted layout unusually well: `MatchPaintedLayout` pins the map to nine painted flat spots,
and re-ordering the nodes on a fixed board is exactly what that constraint is good at. Five things it
touches, recorded so whoever builds it does not rediscover them:

- **THE SHOP RULE HAS TO BECOME LOOP-AWARE.** `Shop_IsNeverTheFirstNode` (added 2026-08-05, VERIFY #84)
  exists because gold starts at zero. On loop 2+ the purse is *not* empty, so the rule stops being about
  fairness and starts being a pointless restriction. The honest form is "no shop on the opening node
  **while the purse is empty**", not "never on row 1". Do not just delete the test - narrow it.
- **"Clear rate" stops being a well-defined metric.** Every balance figure in this repo (the 80-seed
  43.8%, the twelve pinned baselines, `EveryClearingBaseline_ActuallyReachesTheBoss`) assumes a run ENDS
  at the boss. With a loop, the question becomes "clear rate of loop 1" and "how many loops survived".
  The baselines need re-framing before they mean anything again.
- **Determinism.** A loop's map must come from the run seed and the loop index, never a fresh random -
  `SameSeed_ReproducesAnIdenticalRun` is the invariant that has caught every reproducibility break so far.
- **What actually scales** is undecided: enemy HP, enemy damage, roster size, more Elites, fewer Rests, or
  a mix. Step 6 of the skill-tree plan (difficulty scaling with TOKENS) was declined by the owner - this
  is a different axis (scaling with LOOPS) and does not revive it.
- **Loops grant no skill points.** Progression currency comes from verified workouts and the steps goal,
  not from play, so looping is a pure in-run challenge with no meta reward attached. That separation is
  clean, but it means a player has no permanent reason to push deeper unless one is added deliberately.

- Skill tree plan **step 2 skipped**: trees are a plain-C# table (`Progression/SkillTree.cs`), not
  `SkillNodeSO` assets. Note `SkillTreeView.LayoutOf` derives layout from `StartsOwned`/`Prerequisites`,
  so a generator must preserve branch order.
- Step 6 (difficulty scaling with tokens) **declined** by the owner for now.
- Issue **#110 can be closed** — verified fixed, the mage's death clip is 17 frames with none blank.
- Issue **#77** (hero nameplates float left of their heads) is live and unowned.
- Site repo: all PRs merged, nothing open. The rain fixes (#63, #64) are deployed but **never
  reproduced in a browser** — the sub-pixel diagnosis is inferred from screenshots plus the owner's
  "only at 100% zoom" observation. Worth one confirmation.

---

## 5. Landmines learned the hard way today

- **An animation that starts a card at scale 0 makes it invisible while its Button is still live.** Two
  rows of the old table rendered as empty space that could nevertheless be tapped. If you stagger
  anything, cap the total delay and leave the object at full size when the animation cannot run.
- **A clip can be valid, correctly sized, fully tested and still be the wrong drawing.** Nothing in the
  art pipeline looked at pixels until this session. When art "does not work", render the frames out and
  *look* before debugging code.
- **C# static field initialisers run in declaration order.** `ArtFolders` threw a
  `TypeInitializationException` because a derived array was declared above the thing it derived from. It
  now uses a static constructor for exactly this reason — do not convert it back to field initialisers.
- **The MCP bridge times out on long editor work but the work still completes.** `AtlasImporter.Import`,
  `CharacterArtBuilder.Build` and the conditioner all "failed" with retry errors and had in fact run.
  Check file mtimes before re-running — re-running was what cost 40 minutes.
- `SceneWiring.SetAutoPilot` cannot be called during play mode (it opens the scene). Exit play first.
- Setting `Time.timeScale` from a script can leave `ProjectSettings/TimeManager.asset` dirty. Check
  `git status` before committing.

---

## 6. Owner decisions recorded today

Card system: picks are **automatic** (the taps carried no information); the deal is **face-down**;
**fresh 5 every round** (no depletion across rounds); reshuffle offers **4**, excluding the held card;
**Focus is the only knowing choice** and works by swapping the held card for a named one; the deck
browser shows **the 5-card decks in this fight**, not everything owned.

Statuses: **kept**, made legible, and DoTs resolve before the enemy phase.

Armed-card highlight means **the card you just tapped while choosing a target** — not a spent card.
