# Skill tree — mechanic and screen

**Status: BUILT, except steps 2 and 6.** Written 2026-08-04 as a plan; steps 1, 3, 4 and 5 landed the
same day. What remains is recorded at the end of section 9 - read that before picking this up, because
one of the two is a balance change and the other is a refactor with no player-visible effect.

Read `docs/BACKEND-INTEGRATION-PLAN.md` §4.5 before writing code here. It constrains where this
feature's state may live, and getting that wrong is expensive to undo.

---

## 1. The design, as the owner specified it

> Workouts verified by the backend provide **skill points** — a token for the relevant character, based
> on the workout logged. **1 workout = 1 token = 1 skill or 1 upgrade.** A new player starts with **one
> skill per character**; every logged workout buys an extra skill or an improvement. The screen is a
> **node tree per character**: you may only buy a skill **connected to one you already own**, or
> **upgrade a skill you already have**.
>
> Separately, hitting a **walked-steps goal** (threshold to be dictated later) mints a **joker coin**,
> spendable on **any** hero. Most people do not do all three workout types, so the joker is the
> placeholder that keeps a neglected hero moving.
>
> **At launch every player starts with three skills per class — an offensive, a defensive (a heal for
> the mage), and the whiff** — so the dead card in the opening hand is the thing that makes them want
> more skills immediately. The deck is eventually always five cards.
>
> **The dev build needs to ignore all of this**: developers must be able to set levels and ownership
> directly rather than being forced to test the game through three skills.

Three things follow immediately, and they are why this is cheap rather than speculative:

**Tokens are per-class, and the classes are already workout types.** *Warrior — strength/gym*,
*Rogue — cardio/running*, *Mage — anything studio or class based*. The authored flavour in
`Editor/GameDataGenerator.cs:75` says "boxing + classes" for the mage, which is narrower than the owner
means — but the skills themselves are already the right umbrella: Spin Cycle, Namast-heal, Pilates-cea,
Downward Dog-de, Tree Pose-itive are spin, yoga and pilates, not boxing. **Widen that comment to
"studio / class based" when the mapping lands**, or the next person will read the code and narrow the
category back down.

So "a token for the relevant character" needs no new taxonomy: a lifting session mints a Warrior token,
a run a Rogue token, a studio class a Mage token. Training one way only advances one hero — the fitness
incentive doing gameplay work for free, and exactly the pressure the joker exists to relieve.

**The whiff cards are a starting gift, not a purchase.** `w8` Rest-in-Pieces, `r8` Hit the Wall and
`m8` Cramp-ed Style are owned from the first launch and can never be bought, upgraded or removed. They
exist to be diluted: the only way to see fewer dead draws is to own more real skills. That is the
incentive, and it is why they must be *granted* rather than left out of the data.

**The dev build is a first-class requirement, not a convenience.** Testing the fight through a
three-card deck is testing a different game. See §9 step 1.

**This settles the open question in `VERIFY-WITH-OWNER.md:88-90`.** The workout hook is skill unlocks +
upgrades. `SkillDefSO.Level` (`Definitions/SkillDefSO.cs:30`) — reserved with the note *"the workout
hook will eventually unlock NEW skills and UPGRADE existing ones"* — is exactly the field this uses.

---

## 2. The consequence that has to be designed, not discovered

**Starting with three skills breaks the deck draft.** `DeckDraft.DeckSize = 5`
(`Runtime/DeckDraft.cs:28`), and `DeckValid` rejects anything that is not *exactly* five **distinct**
cards from the class pool (`:59-74`). A launch player owns three, cannot form a legal deck, and the run
cannot start. Nothing else about this feature is blocked; this is.

### The rule — the deck is `min(5, owned)`, distinct  *(owner-confirmed)*

> *"Once 5 skills are acquired, it needs to be 5 skills for every run. Until then — however many skills
> the player has unlocked (3 at launch, 4 after 1 workout, etc)."*

Your deck is everything you own until you own more than five, and a five-card choice thereafter. Note
that only an **unlock** grows the deck; an **upgrade** spends a token without adding a card, which is
the trade-off that makes the two kinds of purchase feel different.

```
EffectiveDeckSize(owned) = min(DeckSize, owned.Count)
```

| Skills owned | Deck | What the draft screen is |
|---|---|---|
| 3 (launch) | 3 | Your whole roster, whiff included. **1 draw in 3 is dead.** |
| 4–5 | 4–5 | Still everything. The whiff dilutes as you buy. |
| 6 | 5 | The first real cut — and the whiff is the obvious thing to drop. |
| 7–8 | 5 | A genuine 5-of-N draft, reached by training. |

**This is what makes the whiff bite.** At three owned it is unavoidable — a third of every hero's draws
do nothing, and the only cure is more skills. At six owned the player finally gets to cut it, which is a
small, earned moment. It also means the draft screen *becomes* a choice as you progress, where today it
is a fixed 5-of-8 forever.

**`DeckDraft.DeckSize` stays 5** and becomes the cap rather than the count. Everything downstream —
`DeckValid`, `CanToggle`, `AutoDraft`, `DeckDraftView`'s "3/5 picked" counter — reads
`EffectiveDeckSize(owned)` instead.

### The starting three  *(owner-confirmed)*

| | Attack | Defence / heal | Whiff |
|---|---|---|---|
| Warrior | `w1` Curlified (Atk 2) | `w7` Brace Yourself (Blk 2) | `w8` Rest-in-Pieces |
| Rogue | `r1` Jog On (Atk 1) | `r6` Side-Step-Up (Blk 2, self) | `r8` Hit the Wall |
| Mage | `m1` Jab-racadabra (Atk 2) | `m4` Namast-heal (Heal 3) | `m8` Cramp-ed Style |

The mage's middle slot is its heal, not a block — the heal is the mage's identity and the only one in
the game. The cheap reliable attack is the offensive starter in every case: with a three-card deck
`CombatEngine.Draw` deals it constantly, so it has to be something that always has a use.

---

## 3. The trees

**Three open roots per class — including the whiff — and five buyable nodes.** All three starters show
as owned and each grows a branch, so the first token is a three-way choice rather than a queue.

```
WARRIOR (strength/gym)          ROGUE (cardio/running)          MAGE (studio / classes)

★w1 Curlified                  ★r1 Jog On                     ★m1 Jab-racadabra
  Atk 2                          Atk 1                          Atk 2
  └ w2 Bench-Pressure  Atk 3     └ r2 Sprinterrupt    Atk 2      └ m2 Spin Cycle      Atk 3
    └ w3 Deadlift-Off  Atk 4       └ r3 Joggernaut    Atk 3        └ m3 Kick-box-alot Atk 4

★w7 Brace Yourself             ★r6 Side-Step-Up               ★m4 Namast-heal
  Blk 2                          Blk 2 self                     Heal 3
  └ w5 Plankenstein    Blk 3     └ r7 Zig-Zag-nificent          └ m6 Downward Dog-de Blk 3
    └ w6 Ab-solute Unit Blk 5         Blk 3 self                  └ m7 Tree Pose-itive Blk 4

★w8 Rest-in-Pieces             ★r8 Hit the Wall               ★m8 Cramp-ed Style
  whiff                          whiff                          whiff
  └ w4 Swolecano       Atk 5     └ r4 Dash-ing        Atk 2      └ m5 Pilates-cea     Heal 5
                                   └ r5 Marathon-ster Atk 4

★ = owned from the start, shown open, never purchasable and never upgradeable
```

Adjacency is the owner's rule: a node is buyable only when a **connected** node is owned. The layout is
a proposal, not a constraint — edges live in data (`SkillNodeSO.Prerequisites`), so re-shaping a tree is
an editor-tool change rather than a code change.

**The whiff branches into each class's recovery payoff**, which is the one bit of the layout worth
keeping if the rest is re-cut. *Rest-in-Pieces* → **Swolecano**, the all-or-nothing biggest lift: you
rested, now move the big weight. *Hit the Wall* → **Dash-ing** → **Marathon-ster**: out of gas, second
wind, then the distance. *Cramp-ed Style* → **Pilates-cea**, the big heal — and heals cleanse Cramp, so
the card that represents seizing up literally opens the cure for it.

It also means the dead card is not merely dead weight in the deck: it is a live root on the screen with
somewhere good to go, which reads far better than a greyed-out orphan.

**The whiff is still never bought and never levelled.** `SkillType.None` ignores `Value`, so a level on
one buys nothing. It is granted, it is drawn, and it is a prerequisite — that is all.

### Upgrades

Default: **+1 to the skill's `Value` per level, capped at level 3** (so +2 at most). Reads plainly on
the card — *Curlified 2 → 3* — and needs no new concept.

The whiff is not upgradeable — `SkillType.None` ignores `Value` entirely, so a level on one buys nothing.

Budget per class: 5 unlocks + 7 upgradeable skills × 2 levels = **19 tokens**, so **57 workouts** to max
all three heroes. At one workout per token that is a months-long arc, the right order of magnitude for a
fitness app. Numbers are a tuning question, not a structural one.

**Flagging the balance risk honestly:** `w4` Swolecano at Value 7 instead of 5, on a QTE profile tuned
for 5, is a large swing — and the enemy roster is not on a progression curve at all
(`ProcGen/ProcFightGen.cs:25`: fight budget is run depth only, explicitly awaiting this hook). A fully
upgraded party will flatten the existing content. That is a real balance pass, and it is the owner's
call whether the fight budget should scale with total tokens spent — which is precisely the "player
level → fight budget curve" that `ProcGenConfigSO.cs:6-9` calls the last real design unknown.

---

## 4. The two currencies

| | Minted by | Spendable on |
|---|---|---|
| **Class token** ×3 | one verified workout of that class's type | that class's tree only |
| **Joker coin** | hitting the walked-steps goal | any class's tree |

**When the player holds both coins, ASK which to spend** *(owner-decided 2026-08-04, revised — this
supersedes the earlier "spend the class token first" rule)*.

The earlier rule was right about the danger and wrong about the remedy. A joker is strictly more
valuable — it does everything a class token does and more — so spending one needlessly is a trap the
player only notices later, when the joker they were saving for a neglected hero is gone. But *choosing
for them* is still choosing: a player deliberately hoarding a class token to pour into one hero has no
way to say so, and the auto-spend takes the decision silently. **Asking is strictly more informative
than any default, and it costs a tap only in the case where the decision is real.**

So `ProgressionRules.Affordable(snapshot, cls)` returns a `CoinOptions` — *both* flags, plus
`NeedsChoice` when the player holds both. `TryUnlock` / `TryUpgrade` **require the caller to name the
coin**, and there is deliberately no overload that picks one: a convenience default would be exactly the
silent spend this rule exists to prevent, and it would be reached for the moment it existed.
`CoinOptions.Only` returns `None` when there are two, for the same reason — treating a choice as an
answer is the bug.

Screen consequence: tapping BUY on a node where `NeedsCoinChoice` is true opens a two-option prompt
naming both coins; with one coin it buys directly and the button names what it will spend.

**The steps signal already exists.** `HostSignals.OnTodayStepsRead` (`Host/HostSignals.cs:40`) delivers
an int. The threshold is deliberately unset ("we will dictate later") and is a server-side number
regardless — see §5: the client must not decide when a joker has been earned.

---

## 5. Where the state lives

`BACKEND-INTEGRATION-PLAN.md:451-461` sets a boundary enforced by grep, not by discipline:

- **R1 — simulation state.** Anything in `RunState`: local, seeded, test-pinned, never shown with reward
  framing. `RunState.Gold` is in-dungeon coin, discarded at the end of a run.
- **R2 — account state.** Persists between runs. **Only ever assigned from a parsed server response.
  Never computed, never incremented client-side.**
- **R3 — one bridge.** Only `IRunSettlement` may touch both.

**Tokens, jokers and owned nodes are R2, unambiguously** — the owner's wording is "workouts *verified by
the backend* provide skill points". The client never mints a token, never decides a node's cost, and
never decides that a step goal has been met. It renders a snapshot and sends an intent; the server
returns the next snapshot. Anything else is an exploit: a client that can grant itself a token is a
client that can skip the workout, which is the one thing this product cannot allow. **The joker raises
the stakes on that** — it is the universal currency, so it is the first thing anyone would try to forge.

**The endpoint does not exist**, and the plan says explicitly not to invent it (`:463-466`). So build
against an interface with a local fake, the pattern Phase 7 already uses for `LocalRunSettlement`. Note
this would be the **first save system in the project** — there is no `PlayerPrefs` or `JsonUtility`
persistence anywhere today — so the fake must be unmistakably marked as a stand-in rather than quietly
becoming the real store.

```
Assets/Scripts/Progression/            (new area, existing ReignAndGain assembly — no new asmdef)
  SkillNodeSO.cs              id, class, prerequisites, unlock-or-upgrade, cost
  SkillTreeSO.cs              the authored tree per class, generated by an editor tool
  ProgressionSnapshot.cs      R2 DTO. GET-ONLY: tokens per class, jokers, owned nodes, skill levels
  ISkillProgression.cs        Snapshot() / Purchase(nodeId, currency) -> new snapshot
  LocalSkillProgression.cs    the marked stand-in, until the endpoint exists
  ProgressionRules.cs         PURE: is this node buyable, and with which coin? Shared with the server
```

`ProgressionSnapshot` must use **get-only properties** — `BACKEND-INTEGRATION-PLAN.md` §2.2 makes that
the structural guard, so client-side mutation is a compile error rather than a review comment.

**`ProgressionRules` is pure and testable, and that is the point.** The client needs the adjacency rule
to grey out unbuyable nodes; the server needs it to reject an illegal purchase. Same rule, one file, no
Unity types — the same split as `CombatEngine` (rules) versus `CombatController` (presentation).

**Levels must never be written into `SkillDefSO`.** It is a shared ScriptableObject asset: assigning
`Level` at runtime writes through to the asset in-editor and applies to every hero in every run. Levels
live in the snapshot as a `skillId → level` map and are applied by lookup when a fight starts. This is
the single easiest way to get this feature badly wrong.

---

## 6. The screen

Procedural C#, no prefab, `RgSkin`/`ProcTex` for every surface, `IconFactory` for every glyph — **never
an emoji in a TMP label** (CLAUDE.md).

```
┌───────────────────────────── ⛊3   ⛊1   ⛊0   ★2 ─────────┐  3 class pills + jokers
│                      SKILL TREE                           │
├────────────────┬────────────────┬─────────────────────────┤
│    WARRIOR     │     ROGUE      │        MAGE             │
│  (medallion)   │  (medallion)   │    (medallion)          │
│       ●        │       ●        │        ●        ● owned │
│      ╱ ╲       │      ╱ ╲       │       ╱ ╲       ◐ buyable
│     ◐   ○      │     ●   ○      │      ○   ○      ○ locked│
│     │   │      │     │   │      │      │   │              │
│     ○   ○      │     ◐   ○      │      ○   ○              │
├────────────────┴────────────────┴─────────────────────────┤
│  Bench-Pressure · Attack 3 · costs 1 Warrior token        │
│                                     [ BUY ]  [ CONTINUE ] │
└───────────────────────────────────────────────────────────┘
```

- **Three columns**, one per class, borrowed from `DeckDraftView`. Class colour on the rim via
  `RgTheme.ForClass` — warrior blue, rogue red, mage green (they were swapped once; verify).
- **Nodes** are kit chips (`ProcTex.KitDisc` / `BevelledOctagon`, both exist) carrying
  `IconFactory.SkillGlyph(type)`, so sword/shield/cross says what the node *is* before it is read.
- **Three states, drawn distinctly**, following `HeartBar`'s locked-versus-drained rule
  (`UI/HeartBar.cs:134-155`): *owned* full class colour, *buyable* dimmed with a lit rim, *locked* a
  faint shell. **Buyable must not resemble owned** — that is the same class of lie as a drained heart
  reading as one you never had.
- **An owned node shows its level** (pips, or 2/3) so the upgrade path is visible without tapping.
- **Edges** reuse the map's trail drawing (`UI/MapView.Build.cs`, `UI/MapLayout.cs`). A prerequisite is
  the same picture as a map edge, and the map already draws reachable-versus-not.
- **Four pills** top-right, matching `MapView.Chrome.cs` (`MapArt.Pill`): three class counts plus jokers,
  the joker visually distinct. A Rogue token cannot buy a Warrior skill and the screen must never imply
  otherwise — but the joker must read as the one that can go anywhere.
- **Tap selects, BUY commits**, and the button names the coin it will spend. A purchase on first tap will
  be mis-tapped, and this currency costs a trip to the gym.

---

## 7. Flow wiring

`GameFlow` (`Runtime/GameFlow.cs:14-61`) rejects any transition not in its table, so this is three edits:

1. `GameState.SkillTree` in the enum.
2. Transitions: **`Intro → SkillTree → CharacterSelect`**, with `SkillTree → Intro` back, and
   `GameOver`/`RunCleared → SkillTree` so a finished run spends what training earned. The tree is
   account state and belongs outside the run — placing it mid-run would make it touch `RunState`, which
   is the R1/R2 line.
3. `GameRoot.BuildViews` + `ShowFor` (`Bootstrap/GameRoot.cs:228-287`), following `_select` / `_draft`.

`GameFlowTests` covers the transition table and needs the new rows.

---

## 8. Landmines

**AutoPilot fails the moment this screen exists.** `AutoPilot.Run` opens with
`WaitFor(() => FindActive<CharacterSelectView>() != null, "Character Select never appeared")`
(`Bootstrap/AutoPilot.cs:77`) — a 12-second wait that calls `Fail`. A screen in front of Character
Select means the pilot never starts, and the pilot is the only thing covering the soft-lock class of
bug. **Its step ships in the same commit as the screen, not after.** It should spend nothing and tap
CONTINUE: spending is a tactical choice a smoke test has no business making, the same reasoning that
keeps it off Reshuffle and Focus.

Add the `GetComponentInParent<SkillTreeView>()` skip in `CombatStep` alongside the existing `MapView` /
`DeckDraftView` ones. It costs one line, and that list exists because each omission cost a session.

**The pilot also needs a party it can draft.** Against a fresh progression with one skill per class,
`EffectiveDeckSize` must let it start — otherwise the smoke test dies at Deck Draft instead. Simplest:
the pilot runs against a fully-unlocked progression, which also keeps it exercising the whole card pool.

**Seed baselines.** `RunBehaviourHarness` must construct parties from a **fixed fully-unlocked, all
level-1** progression, or `SeedBaselineTests`' twelve pinned runs stop measuring the engine and start
measuring whatever progression the machine happens to hold. If a baseline moves while wiring this up,
that is a bug, not a re-baseline.

**Never `UnityEngine.Random`** — `SeededRng`, or `ProcTex`'s integer hash for anything cosmetic.

---

## 9. Build order

1. `ProgressionSnapshot` + `ISkillProgression` + `LocalSkillProgression` + `ProgressionRules`
   (adjacency, affordability, class-token-before-joker), with tests. **No UI, no Unity types in the rules.**
   **Ship the dev override in this same step, before any of the rest.** Three modes on
   `LocalSkillProgression`, selected from a `GameRoot` inspector field beside `seed` / `headless` /
   `autopilot`, plus `Reign & Gain` menu items alongside `SceneWiring`'s AutoPilot toggles:
   *Launch* (the real three-skill start), *Everything* (all nodes, max level — what the fight and the
   art are tuned against), *Custom* (set tokens, ownership and levels by hand). Testing combat through a
   three-card deck is testing a different game, and this is the step that stops that happening for the
   whole rest of the build.
2. `SkillNodeSO` / `SkillTreeSO` + the generator tool, authoring the three trees in §3. Follows
   `GameDataGenerator.cs`; hand-authored SOs are not this project's pattern.
3. **The deck-draft change** (§2) — `EffectiveDeckSize`, `DeckValid`, `AutoDraft`, `DeckDraftView`, and
   the harness. Do this *before* the screen: it is the part that can break a shipped loop, and it is
   provable entirely in EditMode.
4. Apply levels at fight start via the `skillId → level` lookup. Re-baseline deliberately if anything moves.
5. `SkillTreeView` + `GameFlow` / `GameRoot` wiring **+ the AutoPilot step, same commit**.
6. Fight-budget scaling against tokens spent — only if the owner wants it (§3).

Steps 1–4 are pure C# with EditMode tests and need almost no editor time. Only step 5 needs Unity for
screenshots and an AutoPilot run.

### What actually landed, 2026-08-04

**Done: 1, 3, 4, 5.** The rules, the deck-draft change, levels reaching the fight, and the screen with
its flow wiring and AutoPilot step.

**Step 6 is declined for now, by the owner**, on the reading that skills should be a straight power
gain until there is something to judge. `ProcGenConfigSO` still has the hook.

**Step 2 was skipped and is the one piece of the plan that was simply not done.** The trees are still
the plain-C# table in `Progression/SkillTree.cs` rather than `SkillNodeSO` / `SkillTreeSO` assets. That
is a deviation from this project's pattern and it should be closed — but it changes nothing a player
sees, so it was not worth blocking the screen on. Note the layout is *derived* from the table
(`SkillTreeView.LayoutOf` reads `StartsOwned` and `Prerequisites`), so the generator has to preserve
branch order or the columns move.

### The one thing that is wired but not fed

`GameRoot` builds a `LocalSkillProgression` and hands it to the tree, and **nothing else reads it.**
The two seams exist and take the data — `CombatEngine` accepts a `SkillLevels`, and `DeckDraft` sizes
itself from an owned pool — but `GameRoot` passes neither, so today the tree screen is the only thing
progression affects. A skill bought there does not yet change a fight or a deck.

Closing that is a handful of lines and **a large balance change**, which is why it is called out rather
than done quietly: with the default `Launch` mode every hero drafts three cards instead of five. Wire
it and revisit `GameRoot.progressionMode` in the same commit, with a fresh clear-rate measurement —
`SeedBaselineTests` logs the 80-seed figure for exactly this comparison.

---

## 10. Still needs an answer

**For Yarin / the backend contract:**
- **What vocabulary does a logged workout arrive in?** `HostSignals.OnHealthSignalsRead`
  (`Host/HostSignals.cs:19`) delivers a string per workout. The three buckets are settled —
  strength/gym → Warrior, cardio/running → Rogue, studio/class → Mage — but the *strings* that map into
  them are Yarin's, and this is the one thing that makes tokens land on the right hero.
- **Which workouts count for nothing?** If a session maps to no class, it mints no token — the player
  will notice and will be right to complain. The joker softens this but does not answer it.
- **The purchase endpoint.** Does not exist. `ProgressionRules` is written to be the shared arbiter so
  the server can reject an illegal buy with the same rule the client greys out.
- **Who evaluates the steps goal, and over what window?** Daily, rolling, once-only? The client must not
  decide, but it does need to know what to display.

**For the owner, defaulted so nothing blocks:**
- **The steps threshold** — deliberately unset ("we will dictate later").
- **Upgrade curve** — defaulted to +1 Value per level, cap 3. See the balance risk in §3.
- **Should the fight budget scale with progression?** Defaulted to no, which means a maxed party
  flattens the current content. `ProcGenConfigSO` was built expecting this exact hook.
- **Is the tree ever reset?** Defaulted to never — permanence is what makes it progression rather than
  another shop.

---

## Appendix — a bug found while planning this, unrelated to the design

**The three real run boons never reach the combat engine.** `RewardEffects.BulwarkTotal` / `RegenTotal`
/ `QteWideTotal` (`Interlude/RewardEffects.cs:146-153`) are computed, unit-tested
(`InterludeTests.cs:306-335`) and read by nothing in production: `CombatController.cs:296` builds its
modifiers with `CombatModifiers.FromConfig(...)` alone, and `RunBehaviourHarness.cs:133` does the same.
Warm-Up, Second Wind and Steady Form are bought, recorded in the pickup history, and do nothing.

**It is not a prerequisite for this feature** — an earlier draft of this plan claimed it was, on the
assumption the tree would grant passive `CombatModifiers`. The owner's design unlocks and upgrades
*skills*, which touches the draft pool and `SkillDefSO.Value` instead, so the two do not meet. It is
recorded here only because this is where it was found, and it deserves its own issue.
