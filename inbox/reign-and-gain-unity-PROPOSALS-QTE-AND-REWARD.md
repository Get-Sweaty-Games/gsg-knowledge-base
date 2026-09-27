# Two proposals: QTE clarity, and reward gating

**Status: nothing here is built.** These were the two items on the 2026-08-11 punch list that were too
vague to guess at, and the owner's instruction was to write them up rather than implement a guess. Every
section below ends in a decision for the owner, and nothing lands until one is taken.

Everything factual here was read out of the code on 2026-08-11 and is cited, so this can be argued with
rather than taken on trust.

---

## The owner's answers, 2026-08-12

**Section 1 is approved in full. Section 2's premise was wrong and is re-scoped below** — read this
before acting on anything further down, because section 2's proposals A and B were both declined.

- **D1 — QTE clarity: A, B and C all go ahead.** No wording change was asked for, so `IMP IS DEBUFFED`
  stands as the label until it is seen on screen.
- **D2 — "dopamine gates" does NOT mean pacing the payoffs.** The owner, verbatim: *"rewards, payoffs,
  successes etc need to be celebrated more clearly and more often. wins, perfect blocks, perfect strikes,
  jackpot roulette spins all need to be mentioned clearly to trigger an emotional response from the
  player."* So the ask is **amplify the moments the game already has**, not add new beats at the
  structural milestones that lack them. The "smallest unit pays hardest, largest pays nothing" finding is
  still true and still cited, but it is **not** what was being asked for and must not be built as if it
  were.
- **Reward proposals A and B: NEITHER, for now.** The floor-cleared beat and the hub count-up are both
  answers to the wrong question — they place *new* beats where there are none, where the ask is to make
  the *existing* ones land harder. They are not rejected on merit; they are simply not this job.
- **D3 — the "would not do" list is wanted, but NOT NOW.** The owner: *"we will have rewards for daily
  logins, weekly training rewards and consecutive/weekly playing. these WILL be in the game but we dont
  have to bring them in now."* So the fitness-framing objection recorded below is **overruled as a
  design position** — retention rewards are planned — and only the timing is deferred. Do not re-raise
  the objection; it has been answered.

**What a re-scoped section 2 has to cover:** the win, the perfect block, the perfect strike and the
jackpot roulette spin, each celebrated more loudly and more often than it is today. That list is the
owner's own and is the scope.

---

# 1. QTE clarity

## Where this actually stands

The QTE is called *"the game's strongest identity lever"* in `QteView`'s own header, and it is already
well drawn: a framed gauge, a lit and bracketed sweet window that breathes, a real needle, and a verdict
word — `PERFECT!` / `GOOD!` / `MISS!` (`QteView.cs:445-447`).

So this is **not** a "make the bar prettier" problem. The bar is fine. What is unclear is everything
*around* it — when it happens, why it happens, and what it was worth.

## The three gaps, in the order a player meets them

### Gap 1 — "why didn't I get a bar that time?"

This is the big one, and the spec already knows it. A QTE does not always run. It is gated
(`docs/COMBAT-RULES.md` §QTE, `CombatEngine.AttackTapApplies` / `DefenceQteApplies`):

| | Gate | If open | If shut |
|---|---|---|---|
| **Offence** | the target enemy carries a status, **or** the attacking hero is buffed | the speed-tap | no QTE; resolves `Plain` (1.0×, no crit) |
| **Defence** | the struck hero carries a status, **or** is buffed | the timing bar | no QTE; the blow lands at **full** value |

`COMBAT-RULES.md` records that the previous version of this gate failed **specifically on readability**:

> *"It was unreadable. The defence gate needed two things from two different rounds of play and neither
> was ever drawn, so the mechanic arrived and left with no visible cause."*

The current gate is a genuine improvement, because it keys off **status**, and status *is* drawn on both
sides of the board. But drawn-somewhere is not the same as connected. Nothing on screen says *"you get a
bar here **because** that enemy is debuffed"*. The player sees a chip on an enemy, and separately sees a
bar appear on some attacks and not others, and is left to correlate two things across many rounds.

That is a smaller version of the exact failure the spec says it just fixed.

### Gap 2 — "why was the window bigger that time?"

`Empowered` widens both bands: `SweetWith(qteWide) => Sweet + qteWide * 2f` and
`GoodWith(qteWide) => Good + qteWide * 3f` (`QteProfileSO.cs:38-39`). Defaults are Sweet 4, Good 16
(`:20`, `:23`), so one stack of buff is a **+50% sweet window** and a fairly large change to the Good band.

That is a big, earned, *invisible* change. The bar just silently has different proportions. A player who
does not already know the rule cannot tell a buffed QTE from an unbuffed one — they only feel that
sometimes it seems easier, which reads as inconsistency rather than as reward.

### Gap 3 — "what did that actually buy me?"

The verdict word says how well you did. It never says what it was worth. `PERFECT!` and `GOOD!` land in
gold and cream, and the multiplier that follows is left to be inferred from the damage number.

## What I'd propose

Three changes, each independently shippable, in the order I'd do them. **They are deliberately all
presentation** — no rule moves, so nothing here can shift balance or the seed baselines.

**A. Name the reason on the bar.** When a QTE opens, print the cause in the gauge's own frame — one short
line, e.g. `ADA IS BUFFED` or `IMP IS DEBUFFED`. The gate already computes exactly this to decide the bar
should exist, so it is available at the call site and costs nothing to surface. This closes Gap 1 with one
label and no new mechanic.

**B. Mark the widened window.** When `qteWide > 0`, draw the *base* sweet window as a thin outline behind
the live one, so the player can see the band they'd normally have had and the extra they were given. It
makes the buff a visible reward at the exact instant it pays out, rather than a number in a chip.

**C. Put the multiplier in the verdict.** `PERFECT! ×2.0` rather than `PERFECT!`. One string change,
reading from the profile that already owns the number.

**What I would NOT do:** widen or narrow any band, change any multiplier, or add a tutorial pop-up. The
bands are tuned and pinned by the seed baselines; a tutorial is what you write when the screen cannot
explain itself, and after A–C it can.

> ### Decision needed
> **Do A–C go ahead?** They are presentation-only and I can land them in one PR. Say if you want the
> wording different — `ADA IS BUFFED` is a guess at your voice, and the label is the whole of change A.

---

# 2. Reward gating ("dopamine gates")

## The honest caveat first

**"Dopamine gates" was never defined beyond the phrase**, and it is the one item on the list where I
could build something plausible and entirely wrong. So this section proposes *less* than the one above,
and asks more.

My reading: **the game's payoffs are not paced.** Rewards mostly arrive as numbers changing, and the few
real *moments* are unevenly spread. A "gate" would be a deliberate beat where the game stops and pays the
player, placed where the run needs one.

If that is not what you meant, stop here and tell me — everything below is downstream of that reading.

## What the run already has

Worth being precise, because the answer is "more than you'd think, unevenly placed":

| Beat | Where | How strong |
|---|---|---|
| **Coin roulette** | after every fight (`CoinRouletteView`) | **Strong.** A drawn prize, a decelerating light, a legible 2–8 range, and since this week a tap that arms and stops it. This is the model the others should be measured against. |
| Treasure / shop / event / rest | interlude nodes (`Assets/Scripts/InterludeUI/`) | Medium. Real screens, but they are transactions — you choose and move on. |
| QTE verdict | mid-fight | Short and frequent, which is right for its slot. |
| Level-up, gold, XP | hub, on sync | **Weak.** Numbers change on a panel. |
| Clearing a floor | map | **Nothing.** The run's biggest structural milestone passes silently. |

The shape of that table is the finding: the game pays out *hardest* for the smallest unit (one fight) and
*not at all* for the largest (a floor). That is backwards, and it is the thing I would fix first.

## What I'd propose

**A. A floor-cleared beat.** The one clear hole. When a floor is cleared, stop for ~2 s and say so, with
the run's shape drawn — floor N of M, party alive, gold carried. It costs one new view and no rules.

**B. Make the weak payouts *land* rather than adding new ones.** The hub's gold and XP arrive as text that
is already correct by the time you look at it. Counting them up, with the existing coin cue under them,
converts an already-happened fact into an event. No new content, no new art.

**C. Leave the roulette alone.** It works, you designed it, and it is the reference. The risk with a pass
like this is diluting the one strong beat by putting siblings next to it.

**What I would NOT do without you saying so:** add loot boxes, streak counters, daily-login rewards, or
anything that gates *content* behind a reward loop. Those are a different genre of decision from pacing
the payoffs the game already gives, and they touch the fitness framing directly — this game pays you for
training, and a mechanic that pays you for *opening the app* argues with that. Not my call to make.

> ### Decisions needed
> 1. **Is "pace the payoffs" the right reading of "dopamine gates"?** If not, describe the itch and I'll
>    re-scope.
> 2. **If yes — A only, or A + B?** A is self-contained. B touches the hub, which is the backend-facing
>    screen, so it wants your eye before it wants my code.
> 3. **Is anything in the "would not do" list actually wanted?** I have assumed not, on the fitness
>    framing. That assumption is easy to overturn and I'd rather you overturned it than that I guessed.

---

## Why neither of these was just built

Both are *taste* calls dressed as feature work. The QTE one has a defensible answer that falls out of the
code (the gate is computed and not drawn — surface it), which is why section 1 proposes specifics.
Section 2 does not have that, and building a reward loop nobody specified is how a game ends up with
mechanics that argue with each other. The measurable parts of this week's list are done and merged; these
two are the parts where your answer is the input.
