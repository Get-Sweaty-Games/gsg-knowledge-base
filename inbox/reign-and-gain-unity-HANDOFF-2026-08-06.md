# Handoff — 2026-08-06

A long session covering combat timing, the whole audio layer, an art import, and a card rebalance.
Read the **Open decisions** section first: three things are waiting on the owner and one piece of work is
deliberately half-finished.

---

## 1. Merged to `main`

| PR | What |
|---|---|
| **#173** | Combat animation sync (#168) — flinches on the real beat; `ActionSettleTimeout` 2x → 2.5x |
| **#175** | Matchup arrow drawn by `IconFactory` (#174) instead of a `↑` in a TMP label |
| **#176** | Audio quality pass — new synthesis primitives, rebuilt combat cues, alternate takes |
| **#182** | The fight was silent at every impact (#179) — four dead outcome-to-cue methods wired |
| **#151** | (earlier) The card deal set-piece, 63 animated cards, fat knight |

## 2. On `feat/audio-voice-profiles` → **PR #183, open**

- Per-character **voice profiles** (13 characters) and per-weapon **attack styles**
- **Swing** and **Effort/Pain** cues — formant-synthesised voices
- Deal animation: last card can be seen to land; blank leaves removed
- Whiffs **dance**
- **39 new hero clips imported** (78 total across the three heroes)
- Rebalance (see §4) — *uncommitted at time of writing, see §6*

---

## 3. Three traps that cost time. All the same shape.

**An artefact claimed the work was done, and only measuring disagreed.** This happened four times today
and is worth internalising before touching anything below.

1. **The art import is THREE stages, not one.** `ArtConditioner.Run` packs pages and writes the manifest;
   `AtlasImporter.Import` writes the sprite rects into each page's `.meta`; `CharacterArtBuilder.Build`
   reads those sprites into the `CharacterArtSO` bundles the runtime loads. Driving the API directly
   skips the middle stage **silently** — the manifest listed 78 clips and the game saw 39. Use the menu
   (`Reign & Gain > Import Art Drop`) unless you have a reason not to.
   *Tell:* a page `.meta` of ~3 KB with `internalIDToNameTable: []` is unsliced. A sliced one is ~45 KB.

2. **`GameDataGenerator` is an editor tool.** Editing its source changes nothing until
   `Reign & Gain > Generate Design Data` runs and rewrites `Assets/ScriptableObjects/Skills/*.asset`.
   The suite passed 709/709 against **stale card data** until the generator was run.

3. **A comment asserted a sound existed that never did.** `ResolveEnemyAction` listed three channels
   carrying a guard — the `+N` pop, the intent line, "and `SfxId.Block` from AudioSystem" — ending
   *"Verified on screen"*. Two of the three were real. `PlayAttackOutcome`, `PlayDefenceOutcome`,
   `PlaySupportOutcome` and `PlayQteResult` had **no callers anywhere**, so the only combat audio a player
   heard was damage-over-time ticks.

4. **Unity's busy state is the `busy for` suffix**, not a phase name. Matching `Compiling|Importing`
   produced three false "settled" readings; the title also goes through `Running managed callbacks` and
   plain `Compiling Scripts`. Match `busy for|Compiling|Importing|Hold on|Reloading` and require several
   consecutive clear reads.

---

## 4. The rebalance — PARTIALLY APPLIED

Owner's rule (2026-08-06): **offence = high damage, defence = high defence, specials = status effects
(+ small offence/defence if relevant). Only the mage heals.**

An audit against every card found **11 violations**, not the 5 first reported:

| card | was | now |
|---|---|---|
| w11 Protein Shake | DMG 4 + lifesteal 2 | DMG 3 ×2 (flurry) |
| r11 Shin Splints | DMG 4 + Fatigue 2 | DMG 2 ×3 (flurry) |
| m11 Slow Burn | DMG 4 + Burn 3 | DMG 6 single |
| m12 Chakra Chug | DMG 3 + lifesteal 3 | DMG 2 ×3 (flurry) |
| w12 Spot Me | BLOCK 3 + empower | BLOCK 4 self |
| w14 Foam Roll-er | BLOCK 3 + cleanse | BLOCK 4 ally |
| m15 Deep Stretch | BLOCK 4 + cleanse | BLOCK 4 party |
| m16 Power Pose | BLOCK 3 + empower | BLOCK 5 self |
| w18 Superset | DMG 2 ×3 | DMG 2 + Fatigue 3 |
| w20 Drop Set | DMG 2 ×2 all | DMG 2 all + Fatigue 2 |
| r17 Pace Setter | DMG 1 ×3 | DMG 1 + Winded 3 |

**Design principle applied:** siblings differ by **shape** (single / flurry / sweep, or self / ally /
party) rather than by status, because the generator's own note says a tier whose cards differ only by
number is not a choice.

### NOT DONE — the rogue's defence path

`r13` Shake It Out, `r15` Catch a Breath and `r16` Coiled Spring **still carry riders**. Deliberately
left, because the rogue's blocks are all **self-only** by class design, so with riders stripped
`r12/r13/r14/r15/r16` would differ only by block value — the exact ladder the pyramid exists to prevent.

The owner proposed differentiating by QTE instead ("one has a higher baseline, one has a bigger perfect
window"). **That works for offence today and not for defence:**

- Attack cards carry `SkillDefSO.QteProfileId`; `CombatEngine.AttackQteFor` reads it off the card.
- The defence QTE comes from the **enemy** — `DefenceQte(enemyId)` — and block cards have
  `QteProfileId = ""`. Letting the played block card influence the defence profile is an **engine
  change**, and is the blocker on finishing this.

### Also not done

- `COMBAT-RULES.md` has **not** been updated for the rebalance. The owner asked for game *and*
  documentation.
- **The clear-rate ladder has not been re-measured.** `SeedBaselineTests` pins 98% at loop 0 down to 0%
  at loop 8; 11 changed cards will have moved it. Re-measure before merging.

---

## 5. The art import

39 new clips staged into `RPGym Game Art/sprite sheets/male <hero>/` (list of exactly what was copied:
the session scratchpad's `imported-files.txt`; all additive, nothing overwritten).

| | before | after |
|---|---|---|
| male warrior | 10 clips | 23 |
| male rogue | 12 | 27 |
| male mage | 17 | 28 |

**Two name collisions kept as two animations**, compared frame-by-frame first:
- `rogue stab` (low lunging dive) vs the new **`thrust`** (upright, blade extended)
- `mage spin` (quick 13-frame pirouette) vs the new **`twirl`** (full 21-frame revolution, back to camera)

**The rogue's trim box moved, 477 → 554 wide.** `SilhouetteTable_DescribesTheFramesTheManifestShips`
caught it with the remedy in the message. Re-measured: `cellW` and the right edge are the only numbers
that changed — the figure did not move or resize, the frame gained transparent space. `StageFormations`
reads the measured edges, which is why nothing shifted on screen.

**Effects available but unused:** `Desktop/game art new animations/effects` — aura / blast / explosion /
fire / slash, each as a still in blue / green / red **and** 13 animated videos. The colours already match
the class palette (blue = warrior, red = rogue, green = mage).

---

## 6. State of the tree at handoff

Uncommitted on `feat/audio-voice-profiles`:

- `Assets/Scripts/Editor/GameDataGenerator.cs` + **11 regenerated `Skills/*.asset`** — the rebalance
- `Assets/Fonts/Rubik SDF.asset` — **not ours**, TMP dynamic-atlas churn, do not commit
- `ProjectSettings/ProjectSettings.asset` — **not ours**, Unity re-adds
  `SENTIS_ANALYTICS_ENABLED;APP_UI_EDITOR_ONLY` on load. Recurs forever; revert it, or commit the defines
  deliberately (it is shared ground with `unity-devs`)

---

## 7. Open decisions — these block work

1. **Rogue defence differentiation.** Engine change to let a block card carry a QTE profile, or accept a
   value ladder, or something else?
2. **The #172 animation mapping.** Posted as a comment on issue #172, ~30 cards, hero + skill → clip.
   Nothing built. Needs the owner's strike-outs.
3. **Sound and music assets.** The owner reports downloading them; they are **not on this machine** — no
   `.unitypackage`, no Asset Store cache. They need importing via Package Manager while signed in.
   General web is unreachable from the agent environment (`kenney.nl`, `opengameart.org` reset;
   `github.com` responds), so the agent cannot fetch them.

## 8. Open issues

- **#165** sound — open on purpose; per-class tailoring done, music/assets outstanding
- **#170** dopamine gates — **parked at owner's request**, proposal in the comments
- **#172** review the 63 animations — mapping proposed, awaiting the owner
- **#163** rename the 15 draft skills — **blocks #166** (card art)
- **#167** battle UI, **#169** QTE bar — ready, not started
- **#179** closed by #182
- **#180** yarins0's combat feel pass — **closed by the owner**; gameplay polish is now recorded in
  `CONTRIBUTING.md` as the owner's own work

## 9. Owner decisions recorded today

- Gameplay polish (timing, telegraph, recoil, weight) is the owner's directly — see `CONTRIBUTING.md`
- Audio stays **synthesised** for now, but higher quality and tailored per character
- Whiffs (`w8`/`r8`/`m8`) **dance**, and deliberately still do nothing
- The deal is **three cards out, three back** — no blank leaves
- `dance` is a special; `rogue top slash` is offensive; `rogue push` is defensive
