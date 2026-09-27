# Reign & Gain — Build Plan

Written after a full pass over `UNITY HANDOVER` + `RPGym Game Art` (both Google Drive exports).

**Status: the full loop is PLAYABLE end to end in the Editor. 314 EditMode tests passing.**

Verified by `AutoPilot`, which drives the real UI with no human input:

```
[RUN] CLEARED (beat Sir Loin)!  seed 12345, gold 37
[AUTO] autopilot: finished at run-cleared after 80 actions     max floor 8/8, 0 exceptions
```

That covers Character Select → Deck Draft → the 8-floor climb → real fights with QTE, cards, intents
and status → shop / event / treasure / rest → mini-boss → Sir Loin → victory.

**Everything is committed to a local git repo** (no remote configured — see `VERIFY-WITH-OWNER.md` #14).
Autonomous decisions are logged in `VERIFY-WITH-OWNER.md` for review.

## Confirmed decisions (owner, this session)

| | |
|---|---|
| Title | **Reign & Gain** (not a working title) |
| Studio | **Get Sweaty Games** — "RPGym" is the game's tagline, not the company |
| Bundle id | `com.getsweatygames.reignandgain` (Android + iOS) — reverse-domain of getsweatygames.com |
| Extra clips | `disappointed` / `bored` / `idle 2` are **alternate idles** — played occasionally, and after player inactivity. Not one-shots. |
| Project | Build in place in the existing project — Unity 6000.5.5f1, URP 17.5.0 |
| UI | uGUI + TextMeshPro |
| Animation | **24 fps** |
| Missing clips | Reuse nearest existing clip + distinct VFX; swap real clips in later via `IArtProvider` |
| Target device | **1080p landscape, mid-range Android.** Source frames downscale ~0.7x in M-Art |
| Data authoring | An editor script generates the ~50 SOs from the Data-Reference tables; design tunes in the inspector |
| Hero names | **Player names all three at Character Select — no defaults.** Ada/Yarin/Mira were artist-facing labels in the Animation Matrix only, not characters |
| Enemy names | goblin=Dadbod Goblin, rat=Gymrat, demon=Skinny Pete the Imp, zombie=Zombinge, skeleton=Bone Appé-tit, spider=Spidercurl Girl, ghost=Winded Wraith, knight=Sir Loin |
| Source of truth | Sprite sheets > PRDs > demo (see §0) |

Standing instruction: **ask for each enemy's name rather than inferring it from the demo.**

## Owner overrides to the enemy design (these beat the PRD — use these, not the docs)

**Archetypes.** Stated in hero-class terms; `Warrior=Bruiser, Rogue=Skirmisher, Mage=Caster`. Encoded
in `EnemyArchetypes.cs` and locked by `EnemyArchetypeTests`.

> ⚠ **The class triangle these archetypes fed was REMOVED on 2026-08-23** (owner: "this feature needs
> to be removed or archived and inactive for now ... the mechanic as a whole is still there and needs to
> be removed entirely"). The mapping below is still correct and still locked, because the encounter
> composer uses an archetype for VARIETY. Nothing reads it as a strength or a weakness any more.

| Enemy | Owner | Fight PRD §6.3 said | |
|---|---|---|---|
| Dadbod Goblin | Warrior | Bruiser | same |
| Gymrat | **Warrior** | Skirmisher | **changed** |
| Skinny Pete | Rogue | Skirmisher | same |
| Zombinge | Mage | Caster | same |
| Bone Appé-tit | **Rogue** | Bruiser | **changed** |
| Spidercurl Girl | **Rogue** | Caster | **changed** |
| Winded Wraith | Mage | Caster | same |
| Sir Loin | **Warrior** | `Boss` (not a triangle class) | **changed** |

Distribution: 3 warrior-class, 3 rogue-class, 2 mage-class. Sir Loin being Warrior-class means **Rogue
heroes counter the final fight**.

**Sir Loin is a boss, never a minion.** He appears in no spawn pool; a test asserts he can only ever
appear in a Boss encounter.

**A mini-boss is a buffed-up ordinary minion**, not a dedicated creature. This retires two things from
the handoff: the hard-coded "Mini-boss = Winded Wraith", and the fixed Elite pool (BoneAppetit +
Spidercurl). All eight non-boss enemies are ordinary minions that any Fight or Miniboss node can draw.
Scaling lives in `EncounterBuffs.cs`, tunable on `GameConfigSO`:

| Encounter | HP | Bonus intents | Foes | Source |
|---|---|---|---|---|
| Fight | ×1.0 | +0 | 2–3 | — |
| **Mini-boss** | **×2.0** | **+1** | **1** | owner-stated |
| Boss (Sir Loin) | ×1.0 | +0 | 1 | authored stats, never scaled |

So a mini-boss Dadbod Goblin is 16 HP with 2 intents.

### Elite was merged into Mini-boss (2026-08-14)

Owner: *"what is the difference between the purple fight node and the orange fight node? they seem
redundant — both trigger the same fight"*, and, asked to choose, *"merge them — one buffed-fight node"*.

There used to be a third row here: **Elite, ×1.5 HP, +0 intents, 1–2 foes**, and its source column read
*"proposed reading of 'lightly buffed'"* — a guess that was never confirmed. **Mini-boss survived because
its numbers are the owner's**, so merging that way kept a decision and dropped a guess.

`NodeType.Elite` and `EncounterKind.Elite` are **deleted**, not retired-in-place, so every reference had
to be found by the compiler. The Elite weight in `MapGenerator.BandTable` became the Mini-boss weight one
for one, so **the combat share of every band is unchanged** — what changed is what a buffed fight *is*,
not how often one appears.

**It barely moved the difficulty**, measured on the 80-seed fixed-policy sweep either side of the change:

| loop | 0 | 2 | 4 | 6 | 8 |
|---|---|---|---|---|---|
| before | 73% | 25% | 15% | 0% | 0% |
| after | 70% | 30% | 13% | 0% | 0% |

Three points at loop 0, and loop 2 went *up* — noise on 80 seeds. **The foe count is why:** a mini-boss is
exactly *one* foe where an Elite was one or two, so doubling the HP and halving the head count very nearly
cancels. The encounter got **deeper, not harder** — one tougher opponent instead of a pair of middling
ones. Do not read that as "the magnitudes don't matter": raising the multiplier without also thinking
about the foe count would land the full increase, with nothing left to cancel it.

Five of the thirteen pinned seeds in `SeedBaselineTests` moved and were re-recorded; the movement is
written up there.

## Owner override to map fight density (2026-07-28)

Owner, verbatim: *"the dungeon run map is a bit too cluttered with none-fight steps making it easy to
almost finish a run with just 1-2 fights."* This **beats the demo BAND_TABLE and Interlude PRD §4.2**
wherever they disagree — including the earlier band-2 "elite 3 / event 2 vs event 3 / elite 2" ruling,
which is now moot where it conflicts with density.

Measured on the demo-faithful weights (seeds 1000–1599): the combat-minimizing route averaged **2.4**
combat nodes pre-boss, **87/600** seeds allowed reaching the boss with the Miniboss as the *only*
fight, and only 47.7% of choice nodes were combat.

The fix, in `MapGenerator.cs`:

- **Structural floor** (`EnforceCombatFloor` + `MinChoiceCombatFloor`): every start→boss route now
  crosses ≥ 3 combat choice nodes, i.e. **≥ 4 combat encounters pre-boss** counting the Miniboss gate
  (≥ 5 with the boss). A post-pass converts non-combat nodes lying on the most combat-minimal routes
  into Fights until the floor holds; the guaranteed Shop/Rest are never converted. Weights alone
  cannot bound the worst-case route — the floor is the actual guarantee, and `Validate` now enforces
  it as a hard invariant.
- **Weights** leaned combat-ward (band 0: fight 5→6, event 3→2; band 2: treasure 2→1; band 1
  unchanged) so the post-pass intervenes sparingly.

Result over the same 600 seeds: worst-case route = **4** combat pre-boss (was 1), choice-node combat
share **61.5%** (owner target ~55–65%), Shop+Rest still guaranteed per map, Events in 579/600 and
Treasures in 442/600 maps. Determinism untouched — same seed, same map, `SeededRng` only. Pinned by
`CombatFloor_HoldsAlongEveryRoute_AcrossManySeeds`, `CombatShare_LandsInOwnerBand`, and
`Interludes_SurviveTheCombatFloor` (600 seeds each); all 322 EditMode tests pass.

**Winded Wraith base HP = 8** (was 18 as the dedicated mini-boss, wildly out of line with the 5–10
minion range). In line with Spidercurl, fitting a Lair-biome mage-class minion. Its 3/4 damage intents
are unchanged, so it reads as a hard-hitting late minion rather than a wall.

## M0 result (verified through the live Unity bridge)

- `productName` = "Reign & Gain", `companyName` = "Get Sweaty Games", bundle id
  `com.getsweatygames.reignandgain`, **landscape-locked** (portrait off, both landscapes on).
- `ReignAndGain.dll` + `ReignAndGain.EditModeTests.dll` compile clean — **0 errors**.
- **19/19 EditMode tests pass.** 600 seeds validated against every map invariant; seed determinism
  proven for both the map and the whole run; 300 seeded runs all reach and clear the boss.
- `Assets/Scenes/Bootstrap.unity` + `Assets/ScriptableObjects/GameConfig.asset` created; Bootstrap is
  build scene 0.
- Headless play-mode run walks Start→Boss with correct biome progression and correct roster ids.

Known environment wart: the project path contains spaces (`C:\Users\Nitai Zweig\RNG test`), which the
MCP plugin warns about. Everything works so far; flagging in case tooling misbehaves later.

---

## 0. Source-of-truth hierarchy

Established by the owner in this session, and it overrides what the docs say:

1. **Sprite sheets** (`RPGym Game Art`) — authoritative for enemy identity, names, models, and which
   attacks exist. Created *after* the demo and after the docs.
2. **PRDs / Data-Reference** — authoritative for rules, numbers, and structure.
3. **`combat_core3.SOURCE.js`** (the GDevelop demo) — authoritative for *feel* and for any formula the
   docs leave ambiguous. Frozen; still uses retired enemy ids.

Two doc statements are now **obsolete** and I am treating them as superseded:

- CLAUDE.md agreement #1 ("art = placeholders, do NOT invent final art, real sheets arrive later") —
  the real sheets are here. Placeholders are now only a bootstrap scaffold, not the deliverable.
- Fight PRD §7.2 / Enemy-Roster "**2–3 animation clips per enemy, hard cap**" — the delivered art has
  4–5 clips per enemy. The cap was an art-budget constraint that the art has already exceeded.

---

## 1. What the handoff actually contains

### 1.1 The `UNITY HANDOVER` export is filename-scrambled

Every filename in that folder is attached to the wrong file's contents. `combat_core3.SOURCE.js` is a
PNG, `Enemy-Roster.md` is the demo HTML, `male_mage_idle_animation.png` is markdown, and the two
`.xlsx` matrices are swapped with two `.cs` files. All content is intact and I have de-scrambled it by
content inspection into a clean set.

It is **not** a clean permutation — the export mixed in files from a wider set, so a few original
filenames are unrecoverable. Concretely, **3 of the 10 C# skeleton stubs are missing** (their name
slots are occupied by an extra markdown file, the demo HTML, and a third PNG). The survivors are:
enums, GameRoot, IArtProvider, PlaceholderArt, StubCombatController, and two small boundary types.
The missing ones are trivial stubs (`SeededRng`, `MapGenerator`, `QteProfileSO`-ish, ~0.5–3 KB each)
that get rewritten in M1 anyway, so this is **not a blocker** — but the nested `RG_Handoff.zip` does
not contain them either, so if a canonical copy exists elsewhere it is worth grabbing.

### 1.2 Art inventory

| | |
|---|---|
| Sprite sheets | 99 files, 199 MB on disk |
| Characters | 6 hero skins (male/female × warrior/rogue/mage) + 8 enemies |
| Idle poses | 20 single-frame stills (772×450), separate from the sheets |
| `references/` | **empty** |

**Sheet grid (verified by alpha projection, not guessed):** every sheet is **4 columns wide**, cell
**768×448**, rows = `height / 448`. Every sheet ends with **exactly 3 blank cells**. So:

| Sheet height | Rows | Frames |
|---|---|---|
| 3584 | 8 | **29** |
| 3136 | 7 | **25** |
| 2688 | 6 | **21** |
| 2240 | 5 | **17** |

Produced by God Mode AI (per the matrix), side-scrolling facing, one facing per character. Every
action starts from the idle pose — so idle is the blend anchor for everything.

### 1.3 Enemy mapping (confirmed by owner)

| Art folder | Enemy | Tier | Clips delivered |
|---|---|---|---|
| goblin | Dadbod Goblin | Basic | idle, attack, hit, death |
| rat | Gymrat | Basic | idle, idle2, attack\*, hit, death, curling-disappointed |
| demon | Skinny Pete the Imp | Basic | idle, attack, hit, death |
| zombie | Zombinge | Basic | idle, attack, hit, death, run |
| skeleton | Bone Appé-tit | Elite | idle, idle2\*, punch, hit, death |
| spider | Spidercurl Girl | Elite | idle, idle2, attack, hit, death |
| ghost | Winded Wraith | Mini-boss | idle, idle2, attack, hit, death |
| knight | Sir Loin | BOSS | idle, attack, **attack2**, hit, death |

**The Tier column is what the art was DELIVERED as, not what the game does with it.** Those tiers were
retired above — every non-boss here is an ordinary minion that any Fight or Mini-boss node can draw, and
the Elite tier does not exist at all since 2026-08-14. The column is kept because it is a record of the
delivery, and re-labelling it would make the art drop harder to trace, not easier.

Every enemy has idle + attack + hit + death — better than the spec's 2–3 cap. Sir Loin's `attack2`
for his telegraphed special is present as required.

\* Two file-format defects: `rat attack clean.webp` is **actually a PNG** with a wrong extension (safe
rename). `skeleton idle 2 clean.webp` is a genuine WebP at 24 KB vs ~1.5 MB for its siblings — a
degraded export. Unity does not import `.webp` natively; both need converting. The skeleton one is
redundant since `skeleton idle clean.png` is intact.

---

## 2. THE critical technical risk: art memory

The sheets are **3.71 GB as uncompressed RGBA32 textures.** That is not shippable on mobile — it is
not shippable anywhere. The cause is padding: character content fills only **14–41%** of each
768×448 cell.

The art therefore **cannot be imported as-is**. It needs an offline conditioning pass, which I plan to
build as an Editor tool so it is repeatable when art is re-exported:

1. **Slice** each sheet on the verified 4×N grid, dropping the 3 trailing blanks.
2. **Trim** every frame to its alpha bounding box, recording the offset so the pivot stays stable
   (trim without offset compensation = jittering animation; this is the step that makes or breaks
   "smooth").
3. **Compute one shared pivot per character** from the union of its idle bounding boxes, so all clips
   for a character register against the same ground point and the sprite does not swim between
   animations.
4. **Repack** into per-character atlases (target 2048×2048, one or two per character) via Sprite Atlas.
5. **Compress** — ASTC 6×6 on mobile.

### MEASURED RESULT (actual, not projected)

| | |
|---|---|
| Characters / frames | 14 / **2,634** |
| Atlas pages | **85** (≤2048², grouped per character) |
| **Raw, sheets imported as-is** | **3.38 GB** RGBA32 — the cost avoided |
| After trim + 0.75× scale | **881 MB** RGBA32 (**3.92×** reduction) |
| **After ASTC 6×6** | **98 MB** |
| Atlas PNGs on disk | 107.8 MB |
| **Realistic worst-case resident set** | **~51 MB** |

That last row is the number that matters: a fight loads one warrior + one rogue + one mage plus up to
three enemy types, never all 14 characters. Worst case is the heaviest skins (male warrior 18.1 +
female mage 14.9 + female rogue 8.8) plus three Lair enemies (~9.3) ≈ **51 MB**. Comfortable on a
mid-range phone.

**A defect this pass caught:** `female mage` first trimmed to 768×448 — i.e. not at all — because
those sheets carry hairline 1-pixel seams at the exact cell boundaries, and one stray pixel per edge
defeats an alpha bounding box entirely. Fixed by requiring ≥3 opaque pixels in a column/row before it
counts as content: that character went 227 MB → 134 MB and the total 108 MB → 98 MB.

**Optimisation lever deliberately NOT taken:** male/female warrior boxes are 628×405 and 627×370 versus
a ~350×300 typical, because sword-blast and spin attacks reach far wider than the idle and the shared
box sizes every frame to the widest. Per-clip boxes would reclaim ~15 MB but forfeit the
one-box-per-character rule that makes pivots stable by construction. Not worth it at 51 MB resident.

This is also why "smooth gameplay + beautiful visuals" is mostly an *import pipeline* problem here,
not a rendering problem.

---

## 3. Hero animation coverage vs the 24 cards

The design needs 7 skill animations + idle/walk/hurt/defeat per class (the Animation Matrix says 11
per hero, 71 total). Mapping delivered art onto the card list surfaces **real gaps**:

| Class | Attacks needed | Attacks delivered | Blocks | Heals | Hurt | Verdict |
|---|---|---|---|---|---|---|
| Warrior ♂ | 4 (w1–w4) | 4 (shoryuken, slash, spinning, sword-blast) | ✅ block | n/a | ✅ (+hit2) | **complete** |
| Warrior ♀ | 4 | 4 (slash, spin, spin2, energy) | ❌ **none** | n/a | ✅ | block missing |
| Rogue ♂ | 5 (r1–r5) | 4 (slash, stab, spin-kick, slide) | ✅ dodge | n/a | ❌ **none** | 1 attack + hurt missing |
| Rogue ♀ | 5 | 5 (punch, kick, kick2, shuriken, slide) | ✅ dodge | n/a | ✅ | **complete** |
| Mage ♂ | 3 (m1–m3) | 2 (strike, energy-strike) | ✅ block | ❌ **none** | ✅ | 1 attack + heals missing |
| Mage ♀ | 3 | 6 (punch, punch2, staff, staff2, energy, energy2) | ❌ **none** | ❌ **none** | ✅ | block + heals missing |

**The biggest gap is Heal.** Mage is the only healer and has two heal cards (Namast-heal, Pilates-cea),
and there is no heal/yoga/namaste clip for either gender. `male mage energy gather` / `energy disperse`
and `female mage energy gather` are the plausible stand-ins but they read as charge-up, not restoration.

Three blocks per Warrior mapping onto one `block` clip is fine and explicitly anticipated by the docs
("blocks/heals can share a per-type clip"). Missing-entirely is the problem.

---

## 4. Exact combat math (extracted from the demo, to be matched)

```
dmg = round((eff + atkBonus) * qteMult) + comboBonus*comboMult + sunderStacks
      + critBonus (if crit) + 2 (if exec boon && target.hp <= ceil(target.max/2))

eff          = Sore ? max(1, floor(base/2)) : base
comboBonus   = min(3, combo-1)          // combo resets each round; miss breaks it
dealDamage   = block absorbs first, remainder hits HP

QTE attack   : red end -> mult 0 (whiff, no status, breaks combo)
               |pos-c| <= sweet -> mult 2.0 + CRIT
               |pos-c| <= good  -> mult 1.4
               else             -> mult 1.0
QTE defense  : red end -> 150% taken ("CRUSHED")
               sweet   -> 0 damage (perfect block)
               good    -> half
               else    -> full
marker speed = (attack 150 | defense 165) * profile.spd - qteWide*20, floor 70
qteWide boon : sweet += 2, good += 3
auto-resolve : 2000 ms attack / 1800 ms defense; tap listener armed after 60 ms
status caps  : Winded min(12, +2)  Burn min(9, +2)  Fatigue min(4, +1)
               Cramp min(4, +2)    Sore/Stiff min(2, +1)   affliction chance 0.30
```

Hero HP: Warrior 11 / Rogue 8 / Mage 7. Deck 5 of 8. Reshuffle 2, Focus 1 per battle.

---

## 5. Architecture

Following `Unity-Architecture-Spec.md` as written:

```
RunController (owns run: seed, party, gold, boons, current node)
   ├── MapModule    : MapGenerator, MapView, Shop/Event/Rest/Treasure
   └── CombatModule : turn engine, DeckSystem, QteSystem, StatusSystem, ProcFightGen
        boundary    : ICombat.EnterEncounter(encounter, onWin, onLose)   <- the ONLY coupling
   Data layer       : ScriptableObjects (Skill, Enemy, Reward, QteProfile, GameConfig)
   Art layer        : IArtProvider -> SpriteSheetArtProvider (Placeholder impl as fallback)
```

Non-negotiables carried from the spec: one seeded RNG per run (never `UnityEngine.Random` in
generation or content rolls); all content data-driven via SOs; all visuals through `IArtProvider`;
map-generation invariants unit-tested across 500+ seeds.

The existing `RNG test` project already matches the recommended stack — Unity **6000.5.5f1**, URP
**17.5.0** (Mobile + PC renderers), Input System **1.19.0** (new-only), uGUI **2.5.0**, Test Framework
**1.7.0**. It needs landscape lock (currently all 4 orientations allowed) and a product rename.

---

## 6. Milestones

Following `Build-Plan-and-Milestones.md`, with one insertion — **M-Art before M2**, because the art
conditioning pipeline is on the critical path for every visual milestone and is where the memory risk
lives.

| | Milestone | Gate |
|---|---|---|
| ✅ M0 | Project setup, landscape lock, folder scaffold | **DONE** — compiles clean; headless run clears the boss |
| ✅ M1 | Data layer + seeded MapGenerator + 70 data assets + tests | **DONE** — 56/56 tests; 600 seeds; fixed seed reproduces |
| ✅ **M-Art** | Slice/trim/pivot/atlas pipeline; all 99 sheets conditioned | **DONE** — 2,634 sprites, 23 pages, 98 MB ASTC, every clip resolves |
| ✅ M2 | Run shell, Character Select, Deck Draft, MapView | **DONE** — wired, themed, screenshot-verified |
| ✅ M3 | Combat core turn engine | **DONE** — pure engine, 50 tests, round order and every formula matched |
| ✅ M4 | QTE system | **DONE** — `QteRunner` + `QteView`; bar difficulty verified by sampling |
| ✅ M5 | Status effects ~~+ class triangle~~ | **DONE inside M3** — cramp gates, sore halves, stiff zeroes block. **The triangle (±50%/−33%) was removed on 2026-08-23** at the owner's request |
| ✅ M6 | Shop / Event / Treasure / Rest | **DONE** — logic + overlays, 70 tests |
| ✅ M7 | Procedural fight generation | **DONE** — depth-driven budget, archetype variety, 15 tests |
| 🔄 M8 | Juice, audio, device perf pass | audio system + placeholder SFX done; **visual polish ongoing, no device build yet** |

**M3 + M4 + M5 share ONE combat gate** (owner, to cut stall points) — they are one system and are more
meaningful to review together.
| M7 | Procedural fight generation + balance | scales by floor, seeded run reproducible |
| M8 | Juice, audio, device perf pass | smooth on mid-range phone in landscape |

Stop for sign-off at every gate.

---

## 7. Where "smooth + beautiful + identity" actually comes from

Recording this so it drives implementation rather than being a late polish pass:

- **Smooth** — the trim/pivot discipline in M-Art (no sub-pixel swim between clips); atlas batching so
  a 3-hero / 3-enemy board is a handful of draw calls; impact frame of each attack clip synced to QTE
  resolution so a crit *lands* on the sweet-spot hit rather than trailing it.
- **Beautiful** — 2D URP lights for biome grading (dungeon purple → forest green → crypt grey → lair
  red per the Interlude PRD); the delivered art is high-res enough to hold up; hit-stop + screen shake
  + flash already specified in the demo's `zoomHit`/`shake`/`flash`.
- **Identity** — the fitness theme is carried by the *QTE shapes* more than anything: Swolecano's
  2.5-wide sweet spot at 78% with an 18-wide miss zone at 1.6× speed genuinely feels like a max-effort
  lift. Per-skill bar shape is the single strongest identity lever and it is fully specified.

---

## 8. Open questions

The M0 stack gate is answered (see the top of this file). These remain, listed against the milestone
that first needs them — none block M1 or M-Art:
### RESOLVED (owner, design sweep)

| Question | Decision |
|---|---|
| Map presentation (Interlude Q1) | **Fit-to-screen board**, whole climb visible at once, as the demo does |
| Class triangle (Fight Q1) | ~~Damage only: +50% advantage / −33% disadvantage~~ — **the question is moot: the mechanic was removed entirely on 2026-08-23** (owner). Note the magnitudes were never confirmed in the whole time it shipped |
| Cramp harshness (Fight Q4) | **Keep the demo's real behaviour: decays 1/round**, cap `min(4, +2)`. The docs' "persists until cured" was never true in code |
| Run structure (Interlude Q2) | **One continuous 8-floor climb.** No acts — matches the generator already validated over 600 seeds |

### Still open

| Needed by | Question |
|---|---|
| M7 | `L` (player level) → fight budget curve, and where the real-workout hook feeds in (Fight PRD Q2, Interlude PRD Q5). The last real design unknown. |
| M2 | ~~Character Select — is the gender skin a free choice per class, or fixed per class?~~ Moot as of `b6635f3`: the female skins were removed, leaving one skin per class and no choice to make. |
| any | Git repo + branch — nothing is under version control yet. |
| — | `RPGym Game Art/references/` is empty. Was something meant to ship in it? |

### Process (owner, to move faster)

- **Batched gate:** M3 + M4 + M5 are reviewed together as ONE combat gate rather than three. They're a
  single coherent system and are more meaningful to judge together.
- **Parallel tracks via subagents**, partitioned by file boundary so agents cannot collide. M6
  (interlude node logic) has zero overlap with the combat engine and runs alongside it.
- **Do not cut the test suite.** It has already caught the wrong `blockChance` values, a generator that
  silently no-op'd against a stale assembly, and 45 lost sprite frames — all silent failures.
