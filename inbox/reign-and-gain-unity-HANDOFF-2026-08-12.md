# Handoff — 2026-08-12

Written for whoever picks this up next. It assumes you have read `CLAUDE.md`; everything here is either
**newer than that file** or **a decision the owner made today that is not written down anywhere else**.

Read the "Do not re-ask" section before talking to the owner. Several of these were settled today at the
cost of a whole proposal being written against the wrong premise.

---

## 1. Where things stand

`main` is green. (The count in this line was 1453 when written and is 1469 at the end of the day - do not
trust a written figure, `tests-run` reports the total.)

**Merged today** (nine PRs, all squash-merged):

| PR | What |
|---|---|
| #353 | The two open proposals got answers; `docs/PROPOSALS-QTE-AND-REWARD.md` header carries them |
| #354 | Mobile aspect scaling — `UiCanvas`, `ScreenMatchMode.Expand` |
| #352 | Combat readability — focus-fan art, target affordance, layering, flinch timing |
| #355 | QTE clarity — the gate reason, the widened-window bracket, the multiplier in the verdict |
| #356 | Roulette — two-phase deceleration + the payoff burst |
| #357 | Damage numbers driven by magnitude |
| #359 | Screen transitions |
| #360 | Cinzel display face + Hebrew typed names |
| #361 | `CLAUDE.md`: there are two dynamic font atlases now |

**Open and mine — needs review:**

- **#364 is MERGED** (owner's call, 2026-08-12). It was the blocker on three separate things: the buff
  stars, the card text, and the strip beside the healthbar the status icons want.

**Open and not mine:** #362 and #184, both `yarins0`. The owner asked me to check whether **#184** has open
tasks needing doing — *I did not get to it.* That is the first thing on the list below.

---

## 2. Do NOT re-ask the owner about these

Settled today. Re-opening them wastes their time and mine already cost a full proposal.

- **"Dopamine gates" does not mean pacing.** It means **amplify the moments that already exist** — wins,
  perfect blocks, perfect strikes, jackpot spins. A whole proposal was written on the "pace the payoffs"
  reading and declined. The floor-cleared beat and the hub count-up are **declined**, not deferred.
- **Retention rewards (daily login, weekly training, streaks) WILL ship.** The fitness-framing objection
  ("a game that pays you for opening the app argues with one that pays you for training") is **answered and
  overruled**. Do not raise it again. Timing only is deferred.
- **Hebrew scope is typed names only.** UI copy stays English. There is no localisation system and none is
  wanted yet.
- **Font is Cinzel**, display roles only, and the owner has confirmed it "is good".
- **The draft screen splits into one hero at a time.** Not "reduce density".
- **Difficulty:** hard-grant a second attack card per class **and** give 1 class token per class (no joker),
  plus deepen the loop-0 forgiveness.
- **Status marks stay OFF the character art.** The `StatusIcon.cs` "not on asset is absolute" ruling was
  *not* overturned — the owner asked for all markings to be *clearer*, which is a legibility pass.

---

## 3. Outstanding work, in priority order

### 3.1 From the owner's 2026-08-12 test run

> **Status as of the end of 2026-08-12.** Items 2, 3 and 4 have LANDED on `main`; the rest are open.
> Item 4 changed shape on owner feedback and item 2 turned out to be two separate renderers - both are
> noted in place below rather than deleted, because the reasoning is the useful part.

Five were done in #364 (card text, effect highlighting, the star, the focus fan, strike connection), which
is now MERGED - so the buff stars the owner kept seeing are gone.

1. **Google registration screen** — "trees are floating in the air. overall, overhaul the screen so it's a
   bit better looking, more dynamic, more textured." The sign-in screen draws `RgSkin`'s outdoor backdrop;
   the floating trees are the treeline band sitting above the horizon with nothing under it.
2. **Card borders still read as "stacked."** #364 fixed the *text* half of that note only. The owner
   confirmed it is "a mix" of the text and the borders. `CardFaceArt.Build` draws **three** nested edges —
   `Frame` (class colour), `Bevel` (dimmed class colour, inset 9), `Body` (inset 16). That is what reads as
   two cards offset. Collapse to one clear edge.
   > **LANDED, and it was TWO renderers rather than one.** #367 collapsed `CardFaceArt`'s three edges to
   > one - that is the big face (skill tree, Buildcraft, focus fan). The owner then reported the cards in
   > HAND still doubling: those are built by `CombatView.BuildCard`, which had its own version of the same
   > fault (plate in darkened class colour under a rim in full-strength class colour). #377 removed that
   > rim. If a card ever looks wrong again, check WHICH of the two renderers drew it first.
3. **The roulette should be a real spinning wheel**, "like in a casino game". #356 made the *travel* feel
   right but it is still a flat seven-cell row with a light running across it. This is a rebuild, not a
   retune — and note the prize is drawn by the engine before the view exists and **must stay that way**
   (`CombatMath.RouletteCoins`); the animation derives from the result, never the reverse.
   > **LANDED (#368), and then went further.** It is a real wheel - seven wedges, a hub and a fixed
   > pointer - and the owner then asked for it to be **100% manual**: it waits to be started and waits to
   > be stopped. That moved the soft-lock guard onto `CombatController.RouletteTimeoutSeconds`, now 90s,
   > because the overlay can no longer end itself. Do not retune that back toward "one spin" - a test
   > fails if you do, and it would reintroduce the auto-stop.
4. **Instructions must pop, and be BUTTONS not text.** "make instructions (such as the spin roulette) a
   button, not just text", animated to draw attention. Applies to the roulette's "Tap to spin", the new
   `CHOOSE A TARGET` prompt, and the FOCUS prompt.
   > **LANDED (#371), BUT THE OWNER NARROWED THE RULE ON SEEING IT:** "buttons should only be for things
   > that actually trigger - the text 'pick an enemy' shouldn't be a button but a clear legible text".
   > So the two combat prompts are legible TYPE (near-white on a dark plate, heading size), and only the
   > roulette's prompt is a real `Button` - because with a manual wheel, pressing it does something.
   > A live `Button` in the combat view also needs an AutoPilot skip; `InstructionChipTests` pins that
   > these carry none.
5. **PERFECT / VICTORY much larger and more noticeable.** **Visual only** — the owner explicitly chose to
   skip VO plumbing rather than have an empty cue wired up. Raise it as its own job if recordings appear.
6. **The ROGUE should walk before he leaps.** An approach clip is missing on that recipe; see
   `AnimLocomotion.StepIn` and `SkillAnimRecipes`.
   > **Corrected 2026-08-12 by the owner.** This entry said *mage*. It is the rogue. Anyone who went
   > looking at the mage's recipes for a missing approach was sent to the wrong character by this file.
7. **Change the warrior's GETTING-HIT sound — the scream.** Owner's words: it is wrong. Remember `SfxId`
   is **append-only** — both `ProceduralSfx.SeedFor` and every `AudioClipTableSO` row read the ordinal,
   so a new cue goes at the BOTTOM and nothing is reordered.
   > **Corrected 2026-08-12 by the owner.** This entry said "the warrior's swing sound" and pointed at
   > `SfxId.Swing`. It is the hurt/pain cue, not the swing. Sir Loin's costumes carry authored Effort,
   > Pain and Death takes (see `Assets/Audio/README.md`); the warrior's Pain is the one to look at, and
   > `HeroHurt` is still the procedural synth because no pack had material for it.

### 3.2 Older, still open

- **PR #184** — check whether it has open tasks needing doing. *Explicitly asked for and not started.*
- **Split the draft into one hero at a time.** `DeckDraftView.Initialise` builds three columns side by side.
  **`AutoPilot` will hang without a matching change** — it clicks every button labelled RANDOM inside the
  draft view and then `START RUN` (`AutoPilot.cs:152-171`); a three-step flow needs it taught to advance.
- **Balance / the empty hand.** Hard-grant a second attack per class, grant 1 class token per class in
  `LocalSkillProgression.Reset` (**not** `GrantTokens`, which is marked DEV ONLY and whose file says nothing
  in the client may mint a token), and lower `EncounterBuffs.FirstLoopMultiplier` below 0.70. This
  **re-baselines all 13 rows of `SeedBaselineTests`** — expected, and that file documents the procedure. The
  canary is `RunInvariantTests.SameSeed_ReproducesAnIdenticalRun`: if that is red *too*, it is a determinism
  bug, not a balance change.
- **"Make all markings clearer."** Block is already drawn **four** ways (shield+numeral on the corner plate,
  one pip per point after the hearts, status chips, on-stage icons beside the name). This is a size and
  contrast pass, not a missing feature.

---

## 4. Landmines found today

Not in `CLAUDE.md` (except where noted) and each one cost real time.

### The font role is inferred from SIZE, and autosizing defeats it
`UiFactory.CreateText` gives the display face to anything created at ≥ `RgTheme.SubheadingSize`.
`CardFaceArt` creates its description at **30pt and autosizes DOWN to 17** — so card body text became
Cinzel, which has **no lower case**, and every card in the game rendered "One clean rep." as
"ONE CLEAN REP.". **If a label autosizes, pass `display:` explicitly.** The size it is born at is a
ceiling, not a role. This is written into `RgFonts` as well.

### There are now TWO dynamic font atlases, and the new one is worse
`Rubik SDF` and `Cinzel SDF`. The Cinzel one lives under `Resources/`, so **any** play session that draws a
title dirties it — and it will block a `git checkout` mid-rebase with "local changes would be overwritten".
When git refuses a branch switch, these two are the first thing to check. Now in `CLAUDE.md`.

### `tests-run` hangs silently if the Editor is in PLAY MODE
This cost ~15 minutes and a wedged runner. `CLAUDE.md` says "stop play → save scene → run tests"; what it
does not say is that skipping it does not error, it **hangs**, and the MCP call then times out client-side
while the Editor keeps a stale "run in progress" flag that survives a domain reload.

**The escape hatch:** drive Unity's own API from `script-execute` — `TestRunnerApi.Execute` with an
`ICallbacks` listener that writes results to a file, then poll the file. That bypasses the plugin entirely
and is how the final green run was obtained. Worth keeping in your pocket.

### `×` is tofu, and so is any lowercase in Cinzel
The Rubik atlas holds only what Rubik ships. `CombatUiMath.QteVerdictSuffix` writes the letter `x`
deliberately. Cinzel additionally has no Hebrew — it carries Rubik in its TMP fallback table, which is what
makes typed Hebrew names survive a Cinzel nameplate. **Do not clear that fallback.**

### The 150px strike floor was overriding measured pairings
`StrikeGapPx` applied `StrikeGapMinPx` to *both* branches. It exists for the **unmeasured** fallback, where
the arithmetic degrades to nothing and two figures walk into the same spot. On the measured path it just
pushed every close pairing back out to 150px — which is what "enemies attack the air" was. Fixed in #364.

### `console-get-logs` caps at ~100 entries, oldest-first
Already in `CLAUDE.md`, but worth repeating because it bit again: during a long play session the buffer is
full of boot logs and you will not see the thing you just logged. **Probe live state with `script-execute`
instead** — reading `GameRoot.RunEnded` / `FlowState` directly is faster and always current.

### AutoPilot verdicts
`DEFEATED` is a **pass**. What the pilot checks is that a run *terminates*. The launch party has a measured
0% clear rate (`SeedBaselineTests.TheLaunchClearRate_IsMeasuredSeparately`) — that is the balance problem in
§3.2, not a regression.

---

## 5. Useful recipes

**Get into a fight fast in play mode** — the drive script pattern used all session: tap `SIGN IN`, dismiss
the claim dialog, `PLAY`, `CONTINUE`, every `RANDOM`, `START RUN`, then walk nodes until `GameRoot.InCombat`,
stepping past any `NodeOverlayBase` that appears. Interlude nodes (treasure/shop) will swallow taps — check
for the overlay first.

**Capture a frame mid-animation:** `EditorApplication.isPaused = true` from a runtime coroutine after a
measured delay. One pause per occurrence — `CLAUDE.md`'s warning about repeated pausing is about *timing
measurement*, and a single frame grab is fine.

**Screen alpha probe** (caught a real bug in #359): walk every `CanvasGroup` whose parent is named `Canvas`
and assert any *inactive* screen sits at alpha 1. A screen stranded mid-fade is invisible until a path shows
it without re-entering.

---

## 6. One judgement call worth revisiting

Removing the QTE star (#364) loses information in exactly one case, and it is written at the call site: a
**buffed hero opens the gate on every enemy**, including ones carrying no chip of their own. The hero's buff
chip is still on screen and the QTE's reason line names the cause when the bar opens — but if the owner ever
reports "I can't tell which enemy gives me a bar", that is the case, and the answer is probably to put the
mark back for the buffed-attacker case only rather than to revert the whole thing.
