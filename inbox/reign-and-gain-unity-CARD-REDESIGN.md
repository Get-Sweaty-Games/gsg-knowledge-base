# Card redesign — branded cards and an interactive draw

Owner-specified 2026-08-04. **Items 4–6 not built.** This is the brief for them; items 1–3 were the
heart-upgrade model and are **done** — see the bottom of this file.

Read `COMBAT-RULES.md` before starting item 6 — it is a rules change, not presentation.

---

## What already exists, so it is not rebuilt

- **Per-class card borders are DONE.** `CombatView.cs` sets `CardFrame.color` and `CardRibbon.color`
  from `RgTheme.ForClass`, dimmed via `RgTheme.Dim(cls, 0.55f)` when the card is unplayable. Warrior
  blue `3f9be8`, rogue red `e8524f`, mage green `5cc46a` — note warrior is BLUE and mage is GREEN, which
  were swapped until #56. If the border reads weakly on screen the fix is **weight or thickness**, not
  wiring.
- **Tween handles exist.** Each `HeroRow` carries `CardWrapper` (the rect `CardDealFx` tweens — the
  card's own anchors stay untouched) and `CardGroup` (a `CanvasGroup` for deal/discard fades).
  `CardDealFx` already animates dealing.
- `CardFace` (parchment), `CardPlate` (outlined kit plate carrying the outline), `CardIcon` +
  `CardIconShadow`, `CardLabel`, `CardValue`, `CardFocusPick` (the "Focus would change THIS one" rim).

---

## Item 4 — a card back with the game logo — **DONE**

Each skill should read as a **branded card**, with the reverse showing the game logo.

**The logo constraint is absolute and is the one thing to get right first.** Use the whole lockup,
as delivered, unaltered — no cropping, rescaling of parts, or extracting elements. The owner has
confirmed one specific allowance: **the logo may be OCCLUDED by another card overlapping it**, because
that is natural stacking rather than an edit to the asset. So design the back around the full lockup and
let the fan/stack overlap do the rest.

Everything else on the back goes through `RgSkin` / `ProcTex` like the rest of the chrome — see the
no-flat-colour invariant in `CLAUDE.md`.

### How it was built

`CombatView.Build.BuildCardBack` — a **full-bleed overlay on the same card object**, not a second card.
One card, one `Button`, one tap target, and "face-down" is a single `SetActive`. A second card would
double the hand's objects and hand `AutoPilot` another `Button` to find.

- **`Branding.cs` now owns the logo.** It was loaded by literal string in two places and the card back
  made three; Resources resolve *by name*, so a rename breaks all three silently at runtime. `Branding.Place`
  drops the lockup in aspect-preserved, so "don't distort it" is not a decision retaken per call site.
  `BrandingTests` pins that the asset exists and is still the 762×648 lockup rather than a crop.
- **The back carries no `TextMeshProUGUI`, and that is load-bearing.** `AutoPilot` identifies a button by
  the first TMP label beneath it and searches *inactive* children — a wordmark set as text here would
  rename every card in the pilot's eyes even while the back is hidden.
- **The class rim is on the back too**, from the *hero's* class rather than the card's: the back has to be
  right while face-down, which is exactly when there is no skill to ask. Item 6 asks for one card per
  class, so a back that hid the class would make the choice unreadable.
- `CombatView.SetCardFaceDown(hero, faceDown)` / `IsCardFaceDown` are the state. **The flip animation is
  item 5's job** — an animation that owned the state would have no safe interruption point, and this card
  can be reshuffled or focused out from under a tween at any moment.

**It ships switched off:** nothing turns a card face-down yet, because the moments that should are items
5 and 6. Verified by forcing all three face-down on the running game and screenshotting.

## Item 5 — deal, flip and shuffle animation — **DONE**

New skills arriving should be animated and interactive rather than appearing. Three moments:

- **Between rounds** — the hand is dealt in, staggered.
- **On reshuffle** — the discard visibly goes somewhere and the deck reads as a finite object.
- **On focus** — the replacement is dealt in.

Build on `CardDealFx` + `CardWrapper` / `CardGroup`. This is the half of #131 that is pure presentation
and carries no rules risk; it can land independently of item 6.

### What was already there, and what was actually missing

`CardDealFx` already dealt in staggered, swapped on reshuffle and focus, and squashed on consume — all
three *moments* were wired, driven by a state diff `CombatView` takes against `CombatEngine` with no
controller involvement. What was missing was the **flip** and the **deck**:

- **The flip is real now.** `FlipOut` used to be an x-shrink standing in for a turn, because there was
  nothing on the other side to show. Item 4 built the back, so a card now deals out **face-down** and
  turns over on arrival, and a reshuffled card turns face-down *before* it leaves. The face is swapped
  at the flip's midpoint, where the card is edge-on and there is nothing to see; `|cos|` rather than a
  triangle so it slows through the turn, which is what makes a scale read as a rotation.
- **Face-up is part of the rest pose.** This is the interrupt safety that matters most now: every other
  settled property recovers something cosmetic, but a card left face-down by a rebind, a hidden canvas
  or a Refresh mid-flip is one the player cannot read and has no way to turn over.
- **There is a visible deck.** Cards fly out of it and discards fly back into it.

### The deck point moved, and where it went was measured

From off the bottom-left edge to the gap on the right, because there is now something to see there. Off
the live board: the hand ends at x 322 and End Turn starts at x 634, so that span is the only clear room
on the cards' own row — the left has RESHUFFLE and FOCUS with **24px** between them and the first card.
`CombatView.BuildPiles` positions each pile *from* its own `CardDealFx` offset so the props and the
trajectories cannot drift apart. **Both figures above are pre-2026-08-28**: the hand moved left, END TURN
moved right to 0.862, and the span now holds TWO piles at 206px each.

> ~~**"The deck reads as a finite object" is drawn, not modelled, and that is deliberate.**~~
> **SUPERSEDED 2026-08-28 — the deck is finite for real now.** The note read: "`CombatEngine.Draw` takes
> `h.Deck[rng.Range(Count)]` every round — a draw **with replacement**... Nothing is ever used up, so a
> card count would be a lie and a discard pile would say the opposite of the rules... A genuinely
> depleting draw pile is a `COMBAT-RULES.md` change and belongs with item 6, not here."
>
> **That change was made, and it went where this note said it would**: `docs/COMBAT-RULES.md` §1b. So the
> card count is now the truth rather than a lie, the discard flies into a **bin** rather than back into
> the deck, and there are two icons on the row instead of one.
>
> One pile serves three heroes, which is a simplification worth naming — each hero has their own deck.
> The existing deal already converged all three trajectories on a single point because that is what reads
> as dealing, so the pile just stands where they already met.

## Item 6 — the player actively picks their cards — **DONE**

> **Owner revised the spec while it was being built, and the revision is the point.** It was specified
> face-down everywhere. Building it made the consequence concrete: a pick between *face-down* cards from
> your own deck **carries no information** — you cannot tell one back from another — so it is the draw
> the engine already made with three taps in front of it, ~36 taps a run that cannot be played well or
> badly. Owner's call, 2026-08-04:
>
> | When | Face | Why |
> |---|---|---|
> | **Between rounds** | **UP** | Intents are rolled at step 1.4, *before* the draw, precisely so the player can see what is coming. Face-up the pick is a real read of the board. |
> | **Reshuffle** | **DOWN** | The gamble — and now the only blind draw in the fight, which is what gives Reshuffle a texture of its own again once every round opens with a choice. |
> | **Focus** | **UP** | Already shipped with #60 and untouched. |
>
> **Focus was already done and always was**, which halved this item: `OnFocusTapped` → pick the hero →
> `ShowFocusPicker` shows that hero's deck face-up → you pick.

### How it is built

- **The fan is a MODE, not a coroutine parked on a callback.** `CombatView.SyncRoundPicker` is driven
  from `Refresh` against the engine's phase, so there is no promise anyone can fail to keep — the same
  shape the Focus pick settled on after `QteView`'s dropped callback once froze a fight permanently.
  It also means Reshuffle re-opening the fan *mid*-player-phase needs no special handling: the fan
  reappears because the phase says so.
- **A low band, not a centred modal.** The player is choosing while reading enemy intents above it, so
  the fan sits roughly where the hand sits and the scrim is light. That is what makes the choice a read.
- **Rebuilt only when the offer changes**, keyed on a hero+cards signature — `Refresh` runs constantly,
  and a `Button` destroyed in the frame it is clicked drops the click.
- **No cancel.** There is nothing to cancel back to; the round cannot start until a card is taken.

### The landmine, cleared

`AutoPilot` matches the fan **by GameObject name** (`CombatView.RoundPickOptionPrefix`) at **top
priority**, above the QTE. Two reasons it is not matched by label: during `Picking` the fan is the *only*
legal action, so a pilot that did not know it would fall through to "anything unrecognised must be a
card" — and face-up these buttons are labelled with **skill names**, which are authored content that
could one day collide with a bucket (a shop boon is already called "Endurance Training"). It always takes
the leftmost card, the same fixed policy `CombatEngine.AutoPick` and the headless harness use, and draws
no randomness of its own.

**Verified:** 497/497 EditMode, and a full piloted run at **231 actions** against ~164 before — the
difference is exactly the three extra taps a round.



**On reshuffle and between rounds:** cards shuffle **face-down** and the player actively picks one card
for each class. **On focus:** the cards are shown **face-up** and the player picks.

### Landmine 1 — this will hang `AutoPilot`

`AutoPilot` **deliberately never taps Reshuffle or Focus** — they are finite and spending them is
off-limits for the pilot — and it actively backs out of half-finished Focus picks
(`BackOutOfFocusPick`). A **mandatory** pick step every round gives it nothing it knows how to satisfy,
so `[RUN] CLEARED` stops appearing.

That matters more than it sounds: the pilot is what catches the "soft-locks at the boss" class of bug
that unit tests cannot. **Decide the pilot's strategy before writing the flow**, not after. Also read
the invariant in `CLAUDE.md` about `AutoPilot.CombatStep` treating any unrecognised `Button` as a
playable card — a new picker full of buttons walks straight into it, and needs an explicit skip the way
`MuteToggle` does.

### Landmine 2 — it changes the rules

The hand goes from **dealt** to **drafted every round**. That is a `COMBAT-RULES.md` change with real
balance consequences (the draft is currently a once-per-run event at deck build), so it needs the spec
updated and the owner's sign-off on the balance, not just an implementation.

### Confirmed: the pick happens EVERY round

Owner-confirmed 2026-08-04. Not only on reshuffle and focus — the between-rounds draw is a pick too, so
an ordinary round costs three picks (one per class) before anything else happens.

**This turns landmine 1 from a caveat into a prerequisite.** The pilot's escape route today is that it
simply never taps Reshuffle or Focus, so it never meets a picker at all. An every-round pick removes that
route: the picker now sits on the unavoidable path through a turn. So `AutoPilot` does not merely need
"a strategy" — **it has to be taught to pick, and that work gates the feature rather than following it.**
Build the pilot's picking alongside the picker, not after it, or the smoke test goes dark for as long as
the two are out of step, and it is the only thing covering the soft-lock class of bug.

Two consequences worth weighing while designing. Both are the owner's call and neither is settled:

- **Input cost.** Three picks per round is a lot of taps on a mobile turn that currently needs one per
  hero. If it drags, the honest lever is making each pick fast — one tap on a fanned spread — rather than
  quietly reducing how often it happens.
- **Reshuffle and Focus lose their distinctiveness.** Their texture today is that they interrupt the
  normal flow with a choice. If every round already opens with a choice, they have to differ in some other
  way than "now you pick". Focus at least still shows its cards face-up, which reshuffle and the
  between-rounds draw do not.

---

## Items 1–3, for context — the heart-upgrade model

Three quantities where the code had two:

| | |
|---|---|
| **Ceiling** | **12**, shared by every hero (`HeartBar.MaxHearts`) |
| **Unlocked** | per hero, starts **11** warrior / **8** rogue / **7** mage — today's values, so nothing rebalances. `HeroState.MaxHp` already *is* this quantity |
| **Current HP** | what the mage restores |

Unlocking grants an **empty** slot, so a shop heart and a shop heal stay distinct purchases. Locked
slots are drawn as faint shells, and locked is a third state, distinct from drained: drained is a heart
you had and lost, locked is one you never had.

**All three are now DONE.** What each turned out to be:

1. **Node grants** — done. The premise recorded when this was written was wrong in one respect worth
   knowing: `MaxHpParty`, `MaxHpClass` and Train by the fire *already* raised the max. What did not
   exist was the **ceiling** — those grants pushed the warrior to 14 or 15 and `HeartBar` then quietly
   compressed the row to 2 HP a heart to fit, so buying a heart made the row *shorter*. The ceiling now
   lives in the rules (`HeroState.HeartCeiling`) and `HeartBar.MaxHearts` reads it, every grant clamps
   through the one method `HeroState.UnlockHearts`, and `MaxHpClass` no longer heals as well.
   **Read VERIFY-WITH-OWNER #69 before tuning anything here** — a shared ceiling of 12 with the warrior
   starting at 11 means he can gain exactly one heart per run, which makes the three per-class boons
   worth very different amounts for the same price.
2. **Persistence** — done, and it was already true: `RunState.Heroes` and `CombatState.Heroes` hold the
   *same* `HeroState` objects by deliberate design, so a raised max already survived every fight. There
   was no copy-back step to notice if that ever stopped being true, so it is now pinned by
   `InterludeTests.AHeartUnlockedAtANode_SurvivesTheNextFight` rather than left as an accident. A second
   "unlocked max" field on `RunState` was considered and rejected: two sources of truth for one number.
3. **Mage heal at full health** — done, as card legality. `CombatEngine.CanPlay` refuses a heal with no
   legal target, so the card dims and cannot be spent. `CanSupportTarget` is the same rule applied
   per-target and is what the view lights target overlays with, so a target that would be refused is
   never offered — which is also what keeps AutoPilot out of a tap-refuse loop. **Cramp is the
   exception**: any heal zeroes Cramp, so a cramped ally at full HP is still a legal target.

### Where the shells appear, and how the rows are sized — owner-decided 2026-08-04

I flagged that the locked shells are drawn in **combat only**, and that between fights a bought heart
just makes the row longer. **The owner's answer was to keep it that way and make the bar grow:** "make
the bar width/lengths dynamic based on the number of hearts". So:

- **In combat**, every hero draws the full 12-slot ceiling with the unbought slots as locked shells.
- **On the map rail and the result screen**, the row is only as long as the hero's unlocked hearts, and
  the plate lengthens as they buy more.

**Nothing is sized by eye any more, in either direction.** That was the actual defect, and it had bitten
in two opposite ways at once:

| | Band | 12-heart row | Fix |
|---|---|---|---|
| Result plate | 286px fixed | 297px | plate **grows** with the hero: `PlateWidthFor(hero.MaxHp)`, 286 floor → 333 at the ceiling |
| Combat plate | 352.8px, **cannot** grow | 369px at 28px hearts | **hearts shrink**: `HeartMath.FitHeartSize` picks 26.65 |
| Map rail | 455px | 261px at 19px hearts | already fits — the "~230px wide past the portrait" comment in `MapView.Build.cs` is stale, from when the purse took a quarter of the rail |

The combat overflow had been shipping unnoticed: the row hung half a heart past its band, inside the
plate, so nothing was visibly clipped. It was found by measuring `HeartBar.Width` against
`rect.width` on the live scene, not by looking — which is the argument for the two maths helpers
(`HeartMath.RowWidth`, `HeartMath.FitHeartSize`) existing at all. A plate that has to hold a heart row
now asks how wide one is instead of carrying a literal that was true for eleven hearts.

---

## Verification note

A previous session recorded that the Unity MCP `tests-run` throws "An unexpected error happened while
running tests" on launch and orphans an in-progress flag, giving **about one suite run per editor
restart**. On 2026-08-04, after exiting a long-idle play session, it ran **twice in a row cleanly**
(6.1s and 6.5s). So the fault is real but not the hard once-per-restart limit it was taken for — try a
second run before assuming you cannot have one. What did precede both good runs: stopping play mode
first, then `assets-refresh`, then `scene-save`.

PR #139's suite debt is **cleared**: 477/477 EditMode green as of the heart-model commit, covering the
three commits that had never been through it.
