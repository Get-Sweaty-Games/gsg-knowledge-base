# Every clip of every hero, fully dyed, measured for shimmer

**2026-09-20/21.** Answering the owner: *"warrior's hit animation has a LOT of shimmer - it looks like
teal - in my instance probably because of the scarf color. seems to have happened on some other
animations but to a lesser degree. deploy a subagent to go through each and every animation for every
hero (fully colored) and check for shimmer"*

## How this was measured

All **100 clips** (warrior 32, rogue 31, mage 37) rendered through the real wardrobe shader, dressed and
at rest, at 1:1 and at the fight's own 0.43x - `WardrobeSheet.Run` with a new colour set
**`plausibleTeal`**: the showcase loadout (`plausible`) with the one change the owner described, his
**scarf dyed Teal**. Sheets in `Logs/audit-plausibleTeal/` (400 files).

| column | tool | what it is |
|---|---|---|
| flicker 1:1 / 0.43x | `tools/masks/rendershimmer.js` | THE GATE. Share of pixels the shipped art held still between two frames (abs diff <= 12 at rest) that the DRESSED render moved (abs diff > 48). Nothing in the drawing explains those. |
| patch | same | biggest connected blob of those pixels on any pair. **This is the column that matches what a person sees.** A scattered 1% is invisible; a 350-pixel patch is a garment coming and going. |
| worst part (patch) | `tools/masks/renderparts.js` | the same pixels attributed to the garment the mask gives them, so the flicker names an object. |
| reversion | `tools/masks/revertscan.js` | garment-painted pixels whose dressed render did not move - a garment failing to take its dye. Warrior band hue 195-245 (his navy), rogue 160-205 (teal leftovers). The mage has no single band worth quoting, so his column is blank rather than wrong. |

Totals over the 100 clips: **1:1 0.564%, 0.43x 0.421%, biggest patch 351** (that patch is the warrior's
`hurt`). Label-level, mask only (`shimmer.js`): warrior 2.491%, rogue 1.930%, mage 3.707%.

⚠ **Two orderings, and the first one is the useful one.** The gate sorts by share, which puts the mage's
clips on top - his numbers are a fine speckle spread over the whole figure (patches of 8-26 px). The
owner's report is about a **patch**, so the main table is sorted by patch size.

---

## Worst first, by the size of the flickering patch

**Every number here is the state AS SHIPPED ON MAIN**, before the repair in this branch. Row 1 -
`warrior hurt` - is the owner's report, and after the repair it reads 0.385% / patch 12 / 0.274% at
0.43x; see "Before -> after" below.

| # | hero | clip | flicker 1:1 | patch | worst pair | flicker 0.43x | worst part (patch) | reversion clip / worst frame |
|---:|---|---|---:|---:|---|---:|---|---|
| 1 | warrior | `hurt` | 0.640% | **351** | f22->f23 2.37% | 0.554% | Scarf 5.83% (349) | 1.11% / 7.7% @f4 |
| 2 | warrior | `roll` | 0.617% | **127** | f18->f19 4.60% | 0.576% | Tunic 2.39% (127) | 1.40% / 11.1% @f27 |
| 3 | rogue | `death` | 0.552% | **53** | f17->f18 1.40% | 0.400% | Skin 1.51% (40) | 0.89% / 2.7% @f3 |
| 4 | rogue | `arrow` | 0.544% | **52** | f15->f16 1.33% | 0.423% | Leathers 0.56% (36) | 0.83% / 3.6% @f22 |
| 5 | warrior | `death` | 0.395% | **52** | f25->f26 1.03% | 0.268% | Scarf 1.02% (51) | 0.21% / 1.1% @f20 |
| 6 | warrior | `scared` | 0.277% | **39** | f24->f25 0.68% | 0.150% | Tunic 0.72% (39) | 0.24% / 1.1% @f26 |
| 7 | warrior | `spinStrike` | 0.566% | **34** | f14->f15 2.15% | 0.385% | Tunic 2.84% (34) | 0.90% / 13.7% @f17 |
| 8 | rogue | `roll` | 0.719% | **30** | f18->f19 6.48% | 0.577% | Weapons 8.91% (30) | 0.96% / 2.7% @f20 |
| 9 | rogue | `jump` | 1.502% | **27** | f14->f15 5.65% | 1.419% | Leathers 1.37% (27) | 1.13% / 8.4% @f5 |
| 10 | rogue | `runAndSlide` | 0.504% | **27** | f16->f17 1.20% | 0.369% | Skin 2.19% (24) | 0.89% / 2.8% @f22 |
| 11 | mage | `standingwave` | 1.255% | **26** | f25->f26 2.90% | 0.971% | Skin 0.81% (26) | - |
| 12 | mage | `shoryuken` | 1.372% | **25** | f10->f11 7.86% | 1.074% | Robes 1.49% (25) | - |
| 13 | warrior | `shieldShove` | 0.396% | **25** | f22->f23 1.20% | 0.279% | Scarf 1.97% (25) | 0.65% / 14.2% @f8 |
| 14 | warrior | `stab` | 0.235% | **25** | f13->f14 0.75% | 0.159% | Tunic 0.49% (25) | 0.23% / 1.1% @f26 |
| 15 | rogue | `topSlash` | 0.510% | **24** | f12->f13 1.14% | 0.402% | Skin 1.61% (23) | 0.79% / 2.6% @f14 |
| 16 | mage | `staffAttack` | 0.746% | **23** | f12->f13 4.00% | 0.532% | Hair 0.71% (23) | - |
| 17 | rogue | `block` | 0.457% | **23** | f6->f7 0.82% | 0.322% | Skin 1.18% (22) | 0.48% / 1.3% @f22 |
| 18 | warrior | `sideSlash` | 0.330% | **23** | f15->f16 0.95% | 0.239% | Tunic 0.76% (23) | 0.33% / 1.5% @f2 |
| 19 | rogue | `shoryuken` | 0.611% | **20** | f16->f17 6.42% | 0.365% | Tunic 0.73% (12) | 0.71% / 2.2% @f8 |
| 20 | mage | `standingWave2` | 0.930% | **19** | f2->f3 2.40% | 0.741% | Trim 5.75% (16) | - |
| 21 | mage | `staffAttack2` | 0.857% | **19** | f5->f6 5.96% | 0.688% | Leathers 1.95% (19) | - |
| 22 | rogue | `dodge` | 0.436% | **19** | f12->f13 1.17% | 0.318% | Hair 0.35% (18) | 1.18% / 4.3% @f17 |
| 23 | mage | `spinAttack` | 1.651% | **18** | f8->f9 7.20% | 1.516% | Robes 1.58% (18) | - |
| 24 | mage | `hurt` | 0.839% | **18** | f9->f10 2.15% | 0.517% | Robes 0.88% (18) | - |
| 25 | rogue | `jump2` | 0.546% | **18** | f9->f10 2.62% | 0.446% | Hair 0.45% (18) | 1.32% / 3.2% @f20 |
| 26 | rogue | `idle` | 0.508% | **18** | f16->f17 0.84% | 0.343% | Hair 0.38% (18) | 0.71% / 2.0% @f27 |
| 27 | warrior | `duck` | 0.362% | **18** | f8->f9 0.76% | 0.293% | Tunic 1.15% (18) | 0.33% / 1.9% @f10 |
| 28 | warrior | `spinKick` | 0.336% | **18** | f26->f27 8.11% | 0.238% | Tunic 1.33% (17) | 0.43% / 4.7% @f14 |
| 29 | warrior | `swordAndShieldAttack` | 0.204% | **18** | f14->f15 0.90% | 0.127% | Tunic 0.58% (18) | 0.45% / 4.9% @f23 |
| 30 | mage | `roll` | 1.550% | **17** | f18->f19 4.22% | 1.288% | Staff 2.95% (13) | - |
| 31 | mage | `push` | 0.979% | **17** | f1->f2 3.23% | 0.767% | Leathers 1.56% (17) | - |
| 32 | rogue | `dance` | 0.419% | **17** | f17->f18 0.83% | 0.304% | Skin 0.66% (13) | 0.69% / 2.3% @f9 |
| 33 | warrior | `push` | 0.347% | **17** | f17->f18 0.60% | 0.188% | Hair 0.15% (12) | 2.56% / 8.9% @f19 |
| 34 | warrior | `explosionStab` | 0.276% | **17** | f10->f11 0.72% | 0.189% | Tunic 0.66% (15) | 0.18% / 1.6% @f10 |
| 35 | mage | `death` | 1.324% | **16** | f13->f14 11.89% | 1.215% | Robes 1.73% (16) | - |
| 36 | mage | `palmBlast` | 1.270% | **16** | f22->f23 4.36% | 0.981% | Robes 1.33% (16) | - |
| 37 | rogue | `walk` | 0.593% | **16** | f10->f11 0.96% | 0.482% | Hair 0.46% (16) | 0.85% / 1.8% @f2 |
| 38 | mage | `idleBreak` | 0.313% | **16** | f12->f13 0.74% | 0.276% | Robes 0.38% (16) | - |
| 39 | mage | `hadouken` | 1.199% | **15** | f14->f15 3.35% | 0.832% | Robes 1.28% (15) | - |
| 40 | rogue | `spinAttack` | 0.398% | **15** | f10->f11 1.90% | 0.244% | Weapons 31.85% (15) | 0.80% / 7.1% @f13 |
| 41 | warrior | `shieldShove2` | 0.350% | **15** | f13->f14 2.85% | 0.225% | Tunic 0.82% (15) | 0.55% / 15.9% @f14 |
| 42 | mage | `energyAttack` | 1.023% | **14** | f9->f10 3.62% | 0.679% | Robes 1.11% (14) | - |
| 43 | mage | `idle` | 0.613% | **14** | f10->f11 1.53% | 0.492% | Robes 0.59% (14) | - |
| 44 | rogue | `energyAttack` | 0.416% | **14** | f3->f4 0.71% | 0.339% | Accessories 0.33% (9) | 41.14% / 75.8% @f14 |
| 45 | rogue | `flipKick` | 0.406% | **14** | f9->f10 1.08% | 0.338% | Skin 2.10% (14) | 0.72% / 1.7% @f3 |
| 46 | warrior | `dancing` | 0.382% | **14** | f14->f15 0.99% | 0.284% | Tunic 1.04% (14) | 0.30% / 2.0% @f13 |
| 47 | mage | `touchGround` | 1.647% | **13** | f10->f11 8.34% | 1.309% | Leathers 3.07% (13) | - |
| 48 | mage | `shake` | 1.216% | **13** | f7->f8 2.37% | 0.848% | Robes 1.09% (12) | - |
| 49 | rogue | `stab` | 0.402% | **13** | f13->f14 0.90% | 0.327% | Skin 1.33% (11) | 0.64% / 2.0% @f3 |
| 50 | rogue | `runAndJump` | 0.362% | **13** | f2->f3 0.87% | 0.265% | Skin 1.51% (12) | 0.78% / 2.2% @f3 |
| 51 | warrior | `shoryuken` | 0.329% | **13** | f13->f14 0.81% | 0.357% | Weapons 0.65% (11) | 0.35% / 1.9% @f7 |
| 52 | mage | `dance` | 1.268% | **12** | f13->f14 2.40% | 0.963% | Leathers 1.27% (12) | - |
| 53 | mage | `dodge` | 1.177% | **12** | f13->f14 3.93% | 0.816% | Robes 1.17% (12) | - |
| 54 | mage | `magicStaffOnGround` | 1.154% | **12** | f15->f16 3.18% | 0.878% | Leathers 1.34% (12) | - |
| 55 | mage | `floatingQuickStrike` | 1.085% | **12** | f14->f15 7.54% | 0.841% | Leathers 1.69% (12) | - |
| 56 | mage | `run2` | 0.965% | **12** | f15->f16 1.35% | 0.641% | Trim 5.81% (12) | - |
| 57 | mage | `magicThrow` | 0.816% | **12** | f14->f15 2.73% | 0.700% | Robes 0.77% (12) | - |
| 58 | mage | `jab` | 0.651% | **12** | f6->f7 2.25% | 0.551% | Leathers 0.98% (12) | - |
| 59 | mage | `idle4` | 0.607% | **12** | f19->f20 1.63% | 0.409% | Robes 0.61% (11) | - |
| 60 | rogue | `pulledbackPunch` | 0.502% | **12** | f13->f14 0.98% | 0.273% | Skin 1.45% (9) | 0.76% / 1.8% @f26 |
| 61 | mage | `idle3` | 0.460% | **12** | f18->f19 0.83% | 0.436% | Robes 0.50% (12) | - |
| 62 | rogue | `quickAttack` | 0.423% | **12** | f23->f24 0.81% | 0.286% | Hair 0.31% (11) | 1.02% / 3.6% @f19 |
| 63 | rogue | `spinKick` | 0.375% | **12** | f17->f18 1.37% | 0.281% | Skin 0.77% (12) | 0.75% / 1.4% @f12 |
| 64 | rogue | `slash` | 0.336% | **12** | f6->f7 0.65% | 0.214% | Hair 0.36% (10) | 1.19% / 3.9% @f15 |
| 65 | warrior | `dodge` | 0.278% | **12** | f13->f14 0.88% | 0.193% | Hair 0.15% (12) | 0.10% / 0.5% @f17 |
| 66 | rogue | `push` | 0.511% | **11** | f13->f14 0.75% | 0.314% | Hair 0.31% (11) | 0.64% / 1.3% @f1 |
| 67 | rogue | `soccerKick` | 0.430% | **11** | f14->f15 1.07% | 0.282% | Skin 0.88% (9) | 1.06% / 4.8% @f9 |
| 68 | rogue | `jumpkick` | 0.389% | **11** | f6->f7 2.16% | 0.316% | Weapons 16.43% (11) | 0.71% / 3.2% @f8 |
| 69 | rogue | `thrust` | 0.338% | **11** | f13->f14 0.75% | 0.280% | Skin 1.58% (11) | 0.77% / 1.9% @f5 |
| 70 | warrior | `swordSlash` | 0.271% | **11** | f11->f12 0.88% | 0.171% | Leathers 0.37% (11) | 0.11% / 0.6% @f20 |
| 71 | warrior | `topSlash` | 0.190% | **11** | f13->f14 0.67% | 0.102% | Leathers 0.14% (11) | 0.25% / 1.5% @f8 |
| 72 | mage | `walk` | 0.967% | **10** | f8->f9 1.52% | 0.614% | Leathers 1.12% (10) | - |
| 73 | rogue | `hurt` | 0.383% | **10** | f1->f2 0.79% | 0.271% | Skin 0.97% (10) | 0.55% / 1.4% @f24 |
| 74 | mage | `jump` | 0.233% | **10** | f5->f6 2.09% | 0.259% | - | - |
| 75 | warrior | `sheath` | 0.194% | **10** | f24->f25 0.49% | 0.099% | Leathers 0.19% (10) | 0.45% / 4.8% @f24 |
| 76 | mage | `throw` | 1.573% | **9** | f13->f14 9.24% | 1.203% | Robes 1.25% (9) | - |
| 77 | mage | `quickStrike` | 1.293% | **9** | f27->f28 5.99% | 0.978% | Leathers 2.69% (8) | - |
| 78 | mage | `energyGather` | 0.778% | **9** | f2->f3 1.08% | 0.583% | Trim 4.82% (9) | - |
| 79 | rogue | `somersaultKick` | 0.508% | **9** | f2->f3 1.62% | 0.412% | Skin 2.21% (9) | 0.61% / 2.3% @f14 |
| 80 | warrior | `headbutt` | 0.348% | **9** | f9->f10 2.45% | 0.234% | Leathers 0.56% (9) | 0.67% / 2.0% @f4 |
| 81 | warrior | `swordBlast` | 0.308% | **9** | f10->f11 1.67% | 0.197% | Tunic 0.74% (9) | 3.48% / 18.0% @f12 |
| 82 | warrior | `run` | 0.252% | **9** | f7->f8 0.34% | 0.165% | Tunic 0.82% (9) | 0.19% / 0.8% @f5 |
| 83 | warrior | `energySwordSwipe` | 0.191% | **9** | f13->f14 2.11% | 0.128% | gold 0.31% (9) | 21.91% / 46.0% @f10 |
| 84 | mage | `twirl` | 1.215% | **8** | f16->f17 2.29% | 0.899% | Leathers 3.01% (8) | - |
| 85 | mage | `pockets` | 0.885% | **8** | f14->f15 1.86% | 0.643% | Robes 1.06% (8) | - |
| 86 | mage | `spin` | 0.871% | **8** | f5->f6 2.66% | 0.617% | Robes 0.69% (8) | - |
| 87 | mage | `crystalToss` | 0.642% | **8** | f4->f5 1.33% | 0.544% | Robes 0.87% (8) | - |
| 88 | rogue | `knifeThrow` | 0.479% | **8** | f12->f13 2.16% | 0.451% | Leathers 0.43% (8) | 0.83% / 3.0% @f17 |
| 89 | warrior | `kick` | 0.244% | **8** | f20->f21 0.95% | 0.213% | Leathers 0.34% (8) | 0.41% / 1.7% @f9 |
| 90 | mage | `block` | 0.731% | **7** | f7->f8 1.83% | 0.569% | Leathers 0.72% (7) | - |
| 91 | mage | `energyBlastFromHand` | 0.704% | **7** | f9->f10 1.69% | 0.442% | Staff 0.39% (7) | - |
| 92 | rogue | `duck` | 0.345% | **7** | f17->f18 1.17% | 0.205% | Tunic 0.46% (7) | 1.01% / 2.7% @f1 |
| 93 | warrior | `groundAttack` | 0.283% | **7** | f13->f14 1.69% | 0.198% | Tunic 0.58% (7) | 0.36% / 2.0% @f3 |
| 94 | warrior | `idle` | 0.155% | **7** | f12->f13 0.27% | 0.095% | Skin 0.04% (7) | 0.09% / 0.3% @f5 |
| 95 | warrior | `armRaise` | 0.118% | **7** | f9->f10 0.57% | 0.059% | Hair 0.07% (7) | 0.18% / 1.4% @f16 |
| 96 | rogue | `spin` | 1.174% | **6** | f9->f10 5.52% | 0.649% | Leathers 1.22% (5) | 0.87% / 1.8% @f12 |
| 97 | warrior | `block` | 0.197% | **6** | f18->f19 0.48% | 0.116% | Tunic 0.41% (6) | 0.12% / 0.4% @f20 |
| 98 | warrior | `auraFarm` | 0.143% | **6** | f2->f3 0.23% | 0.066% | Scarf 1.05% (5) | 0.10% / 0.3% @f16 |
| 99 | warrior | `jump` | 0.066% | **6** | f14->f15 0.61% | 0.067% | - | 99.58% / 100.0% @f3 |
| 100 | warrior | `shieldRaise` | 0.161% | **5** | f2->f3 0.34% | 0.083% | Tunic 0.31% (5) | 0.17% / 2.4% @f2 |

---

## The same 100 clips sorted by the gate's own share

| # | hero | clip | flicker 1:1 | patch | worst pair | flicker 0.43x | worst part (patch) | reversion clip / worst frame |
|---:|---|---|---:|---:|---|---:|---|---|
| 1 | mage | `spinAttack` | 1.651% | **18** | f8->f9 7.20% | 1.516% | Robes 1.58% (18) | - |
| 2 | mage | `touchGround` | 1.647% | **13** | f10->f11 8.34% | 1.309% | Leathers 3.07% (13) | - |
| 3 | mage | `throw` | 1.573% | **9** | f13->f14 9.24% | 1.203% | Robes 1.25% (9) | - |
| 4 | mage | `roll` | 1.550% | **17** | f18->f19 4.22% | 1.288% | Staff 2.95% (13) | - |
| 5 | rogue | `jump` | 1.502% | **27** | f14->f15 5.65% | 1.419% | Leathers 1.37% (27) | 1.13% / 8.4% @f5 |
| 6 | mage | `shoryuken` | 1.372% | **25** | f10->f11 7.86% | 1.074% | Robes 1.49% (25) | - |
| 7 | mage | `death` | 1.324% | **16** | f13->f14 11.89% | 1.215% | Robes 1.73% (16) | - |
| 8 | mage | `quickStrike` | 1.293% | **9** | f27->f28 5.99% | 0.978% | Leathers 2.69% (8) | - |
| 9 | mage | `palmBlast` | 1.270% | **16** | f22->f23 4.36% | 0.981% | Robes 1.33% (16) | - |
| 10 | mage | `dance` | 1.268% | **12** | f13->f14 2.40% | 0.963% | Leathers 1.27% (12) | - |
| 11 | mage | `standingwave` | 1.255% | **26** | f25->f26 2.90% | 0.971% | Skin 0.81% (26) | - |
| 12 | mage | `shake` | 1.216% | **13** | f7->f8 2.37% | 0.848% | Robes 1.09% (12) | - |
| 13 | mage | `twirl` | 1.215% | **8** | f16->f17 2.29% | 0.899% | Leathers 3.01% (8) | - |
| 14 | mage | `hadouken` | 1.199% | **15** | f14->f15 3.35% | 0.832% | Robes 1.28% (15) | - |
| 15 | mage | `dodge` | 1.177% | **12** | f13->f14 3.93% | 0.816% | Robes 1.17% (12) | - |
| 16 | rogue | `spin` | 1.174% | **6** | f9->f10 5.52% | 0.649% | Leathers 1.22% (5) | 0.87% / 1.8% @f12 |
| 17 | mage | `magicStaffOnGround` | 1.154% | **12** | f15->f16 3.18% | 0.878% | Leathers 1.34% (12) | - |
| 18 | mage | `floatingQuickStrike` | 1.085% | **12** | f14->f15 7.54% | 0.841% | Leathers 1.69% (12) | - |
| 19 | mage | `energyAttack` | 1.023% | **14** | f9->f10 3.62% | 0.679% | Robes 1.11% (14) | - |
| 20 | mage | `push` | 0.979% | **17** | f1->f2 3.23% | 0.767% | Leathers 1.56% (17) | - |
| 21 | mage | `walk` | 0.967% | **10** | f8->f9 1.52% | 0.614% | Leathers 1.12% (10) | - |
| 22 | mage | `run2` | 0.965% | **12** | f15->f16 1.35% | 0.641% | Trim 5.81% (12) | - |
| 23 | mage | `standingWave2` | 0.930% | **19** | f2->f3 2.40% | 0.741% | Trim 5.75% (16) | - |
| 24 | mage | `pockets` | 0.885% | **8** | f14->f15 1.86% | 0.643% | Robes 1.06% (8) | - |
| 25 | mage | `spin` | 0.871% | **8** | f5->f6 2.66% | 0.617% | Robes 0.69% (8) | - |
| 26 | mage | `staffAttack2` | 0.857% | **19** | f5->f6 5.96% | 0.688% | Leathers 1.95% (19) | - |
| 27 | mage | `hurt` | 0.839% | **18** | f9->f10 2.15% | 0.517% | Robes 0.88% (18) | - |
| 28 | mage | `magicThrow` | 0.816% | **12** | f14->f15 2.73% | 0.700% | Robes 0.77% (12) | - |
| 29 | mage | `energyGather` | 0.778% | **9** | f2->f3 1.08% | 0.583% | Trim 4.82% (9) | - |
| 30 | mage | `staffAttack` | 0.746% | **23** | f12->f13 4.00% | 0.532% | Hair 0.71% (23) | - |
| 31 | mage | `block` | 0.731% | **7** | f7->f8 1.83% | 0.569% | Leathers 0.72% (7) | - |
| 32 | rogue | `roll` | 0.719% | **30** | f18->f19 6.48% | 0.577% | Weapons 8.91% (30) | 0.96% / 2.7% @f20 |
| 33 | mage | `energyBlastFromHand` | 0.704% | **7** | f9->f10 1.69% | 0.442% | Staff 0.39% (7) | - |
| 34 | mage | `jab` | 0.651% | **12** | f6->f7 2.25% | 0.551% | Leathers 0.98% (12) | - |
| 35 | mage | `crystalToss` | 0.642% | **8** | f4->f5 1.33% | 0.544% | Robes 0.87% (8) | - |
| 36 | warrior | `hurt` | 0.640% | **351** | f22->f23 2.37% | 0.554% | Scarf 5.83% (349) | 1.11% / 7.7% @f4 |
| 37 | warrior | `roll` | 0.617% | **127** | f18->f19 4.60% | 0.576% | Tunic 2.39% (127) | 1.40% / 11.1% @f27 |
| 38 | mage | `idle` | 0.613% | **14** | f10->f11 1.53% | 0.492% | Robes 0.59% (14) | - |
| 39 | rogue | `shoryuken` | 0.611% | **20** | f16->f17 6.42% | 0.365% | Tunic 0.73% (12) | 0.71% / 2.2% @f8 |
| 40 | mage | `idle4` | 0.607% | **12** | f19->f20 1.63% | 0.409% | Robes 0.61% (11) | - |
| 41 | rogue | `walk` | 0.593% | **16** | f10->f11 0.96% | 0.482% | Hair 0.46% (16) | 0.85% / 1.8% @f2 |
| 42 | warrior | `spinStrike` | 0.566% | **34** | f14->f15 2.15% | 0.385% | Tunic 2.84% (34) | 0.90% / 13.7% @f17 |
| 43 | rogue | `death` | 0.552% | **53** | f17->f18 1.40% | 0.400% | Skin 1.51% (40) | 0.89% / 2.7% @f3 |
| 44 | rogue | `jump2` | 0.546% | **18** | f9->f10 2.62% | 0.446% | Hair 0.45% (18) | 1.32% / 3.2% @f20 |
| 45 | rogue | `arrow` | 0.544% | **52** | f15->f16 1.33% | 0.423% | Leathers 0.56% (36) | 0.83% / 3.6% @f22 |
| 46 | rogue | `push` | 0.511% | **11** | f13->f14 0.75% | 0.314% | Hair 0.31% (11) | 0.64% / 1.3% @f1 |
| 47 | rogue | `topSlash` | 0.510% | **24** | f12->f13 1.14% | 0.402% | Skin 1.61% (23) | 0.79% / 2.6% @f14 |
| 48 | rogue | `idle` | 0.508% | **18** | f16->f17 0.84% | 0.343% | Hair 0.38% (18) | 0.71% / 2.0% @f27 |
| 49 | rogue | `somersaultKick` | 0.508% | **9** | f2->f3 1.62% | 0.412% | Skin 2.21% (9) | 0.61% / 2.3% @f14 |
| 50 | rogue | `runAndSlide` | 0.504% | **27** | f16->f17 1.20% | 0.369% | Skin 2.19% (24) | 0.89% / 2.8% @f22 |
| 51 | rogue | `pulledbackPunch` | 0.502% | **12** | f13->f14 0.98% | 0.273% | Skin 1.45% (9) | 0.76% / 1.8% @f26 |
| 52 | rogue | `knifeThrow` | 0.479% | **8** | f12->f13 2.16% | 0.451% | Leathers 0.43% (8) | 0.83% / 3.0% @f17 |
| 53 | mage | `idle3` | 0.460% | **12** | f18->f19 0.83% | 0.436% | Robes 0.50% (12) | - |
| 54 | rogue | `block` | 0.457% | **23** | f6->f7 0.82% | 0.322% | Skin 1.18% (22) | 0.48% / 1.3% @f22 |
| 55 | rogue | `dodge` | 0.436% | **19** | f12->f13 1.17% | 0.318% | Hair 0.35% (18) | 1.18% / 4.3% @f17 |
| 56 | rogue | `soccerKick` | 0.430% | **11** | f14->f15 1.07% | 0.282% | Skin 0.88% (9) | 1.06% / 4.8% @f9 |
| 57 | rogue | `quickAttack` | 0.423% | **12** | f23->f24 0.81% | 0.286% | Hair 0.31% (11) | 1.02% / 3.6% @f19 |
| 58 | rogue | `dance` | 0.419% | **17** | f17->f18 0.83% | 0.304% | Skin 0.66% (13) | 0.69% / 2.3% @f9 |
| 59 | rogue | `energyAttack` | 0.416% | **14** | f3->f4 0.71% | 0.339% | Accessories 0.33% (9) | 41.14% / 75.8% @f14 |
| 60 | rogue | `flipKick` | 0.406% | **14** | f9->f10 1.08% | 0.338% | Skin 2.10% (14) | 0.72% / 1.7% @f3 |
| 61 | rogue | `stab` | 0.402% | **13** | f13->f14 0.90% | 0.327% | Skin 1.33% (11) | 0.64% / 2.0% @f3 |
| 62 | rogue | `spinAttack` | 0.398% | **15** | f10->f11 1.90% | 0.244% | Weapons 31.85% (15) | 0.80% / 7.1% @f13 |
| 63 | warrior | `shieldShove` | 0.396% | **25** | f22->f23 1.20% | 0.279% | Scarf 1.97% (25) | 0.65% / 14.2% @f8 |
| 64 | warrior | `death` | 0.395% | **52** | f25->f26 1.03% | 0.268% | Scarf 1.02% (51) | 0.21% / 1.1% @f20 |
| 65 | rogue | `jumpkick` | 0.389% | **11** | f6->f7 2.16% | 0.316% | Weapons 16.43% (11) | 0.71% / 3.2% @f8 |
| 66 | rogue | `hurt` | 0.383% | **10** | f1->f2 0.79% | 0.271% | Skin 0.97% (10) | 0.55% / 1.4% @f24 |
| 67 | warrior | `dancing` | 0.382% | **14** | f14->f15 0.99% | 0.284% | Tunic 1.04% (14) | 0.30% / 2.0% @f13 |
| 68 | rogue | `spinKick` | 0.375% | **12** | f17->f18 1.37% | 0.281% | Skin 0.77% (12) | 0.75% / 1.4% @f12 |
| 69 | warrior | `duck` | 0.362% | **18** | f8->f9 0.76% | 0.293% | Tunic 1.15% (18) | 0.33% / 1.9% @f10 |
| 70 | rogue | `runAndJump` | 0.362% | **13** | f2->f3 0.87% | 0.265% | Skin 1.51% (12) | 0.78% / 2.2% @f3 |
| 71 | warrior | `shieldShove2` | 0.350% | **15** | f13->f14 2.85% | 0.225% | Tunic 0.82% (15) | 0.55% / 15.9% @f14 |
| 72 | warrior | `headbutt` | 0.348% | **9** | f9->f10 2.45% | 0.234% | Leathers 0.56% (9) | 0.67% / 2.0% @f4 |
| 73 | warrior | `push` | 0.347% | **17** | f17->f18 0.60% | 0.188% | Hair 0.15% (12) | 2.56% / 8.9% @f19 |
| 74 | rogue | `duck` | 0.345% | **7** | f17->f18 1.17% | 0.205% | Tunic 0.46% (7) | 1.01% / 2.7% @f1 |
| 75 | rogue | `thrust` | 0.338% | **11** | f13->f14 0.75% | 0.280% | Skin 1.58% (11) | 0.77% / 1.9% @f5 |
| 76 | rogue | `slash` | 0.336% | **12** | f6->f7 0.65% | 0.214% | Hair 0.36% (10) | 1.19% / 3.9% @f15 |
| 77 | warrior | `spinKick` | 0.336% | **18** | f26->f27 8.11% | 0.238% | Tunic 1.33% (17) | 0.43% / 4.7% @f14 |
| 78 | warrior | `sideSlash` | 0.330% | **23** | f15->f16 0.95% | 0.239% | Tunic 0.76% (23) | 0.33% / 1.5% @f2 |
| 79 | warrior | `shoryuken` | 0.329% | **13** | f13->f14 0.81% | 0.357% | Weapons 0.65% (11) | 0.35% / 1.9% @f7 |
| 80 | mage | `idleBreak` | 0.313% | **16** | f12->f13 0.74% | 0.276% | Robes 0.38% (16) | - |
| 81 | warrior | `swordBlast` | 0.308% | **9** | f10->f11 1.67% | 0.197% | Tunic 0.74% (9) | 3.48% / 18.0% @f12 |
| 82 | warrior | `groundAttack` | 0.283% | **7** | f13->f14 1.69% | 0.198% | Tunic 0.58% (7) | 0.36% / 2.0% @f3 |
| 83 | warrior | `dodge` | 0.278% | **12** | f13->f14 0.88% | 0.193% | Hair 0.15% (12) | 0.10% / 0.5% @f17 |
| 84 | warrior | `scared` | 0.277% | **39** | f24->f25 0.68% | 0.150% | Tunic 0.72% (39) | 0.24% / 1.1% @f26 |
| 85 | warrior | `explosionStab` | 0.276% | **17** | f10->f11 0.72% | 0.189% | Tunic 0.66% (15) | 0.18% / 1.6% @f10 |
| 86 | warrior | `swordSlash` | 0.271% | **11** | f11->f12 0.88% | 0.171% | Leathers 0.37% (11) | 0.11% / 0.6% @f20 |
| 87 | warrior | `run` | 0.252% | **9** | f7->f8 0.34% | 0.165% | Tunic 0.82% (9) | 0.19% / 0.8% @f5 |
| 88 | warrior | `kick` | 0.244% | **8** | f20->f21 0.95% | 0.213% | Leathers 0.34% (8) | 0.41% / 1.7% @f9 |
| 89 | warrior | `stab` | 0.235% | **25** | f13->f14 0.75% | 0.159% | Tunic 0.49% (25) | 0.23% / 1.1% @f26 |
| 90 | mage | `jump` | 0.233% | **10** | f5->f6 2.09% | 0.259% | - | - |
| 91 | warrior | `swordAndShieldAttack` | 0.204% | **18** | f14->f15 0.90% | 0.127% | Tunic 0.58% (18) | 0.45% / 4.9% @f23 |
| 92 | warrior | `block` | 0.197% | **6** | f18->f19 0.48% | 0.116% | Tunic 0.41% (6) | 0.12% / 0.4% @f20 |
| 93 | warrior | `sheath` | 0.194% | **10** | f24->f25 0.49% | 0.099% | Leathers 0.19% (10) | 0.45% / 4.8% @f24 |
| 94 | warrior | `energySwordSwipe` | 0.191% | **9** | f13->f14 2.11% | 0.128% | gold 0.31% (9) | 21.91% / 46.0% @f10 |
| 95 | warrior | `topSlash` | 0.190% | **11** | f13->f14 0.67% | 0.102% | Leathers 0.14% (11) | 0.25% / 1.5% @f8 |
| 96 | warrior | `shieldRaise` | 0.161% | **5** | f2->f3 0.34% | 0.083% | Tunic 0.31% (5) | 0.17% / 2.4% @f2 |
| 97 | warrior | `idle` | 0.155% | **7** | f12->f13 0.27% | 0.095% | Skin 0.04% (7) | 0.09% / 0.3% @f5 |
| 98 | warrior | `auraFarm` | 0.143% | **6** | f2->f3 0.23% | 0.066% | Scarf 1.05% (5) | 0.10% / 0.3% @f16 |
| 99 | warrior | `armRaise` | 0.118% | **7** | f9->f10 0.57% | 0.059% | Hair 0.07% (7) | 0.18% / 1.4% @f16 |
| 100 | warrior | `jump` | 0.066% | **6** | f14->f15 0.61% | 0.067% | - | 99.58% / 100.0% @f3 |

---

## What the warrior's teal shimmer actually is

Not the dye, not the art, not a rim: **a ~500-800 pixel piece of his scarf changes which garment it
belongs to between frames.**

His scarf and his tunic are painted in **one navy** (`tools/masks/README.md` lists the pair at 0.037
separation - "navy on navy; why `set1` measured nothing"), so no colour test can tell them apart and
the mask decides it per frame. On `hurt` it decides differently on different frames:

```
Scarf label, texels per frame (shipped):
 2217 2263 2290 2305 2331 2277 1966 1725 1414 1494 1377 1296 1388  882 1441
 1653 1650 1575 1650 1775 1877 1612 1627 1095 1125 1707 1302 1318 1337
```

f22 -> f23 loses 529 of them to the Tunic label and f24 -> f25 takes 579 back. In the owner's loadout
the Scarf dial is **Teal** and the Tunic dial is not, so that piece flashes teal / not-teal / teal. The
gate says the same thing in numbers: on `hurt` the **Scarf's own still pixels flicker 5.83%, the worst
pair f22->f23 moves 68.7% of them, and the patch is 349 px - the biggest single flicker patch in all
100 clips.**

The 2026-09-15 repair (`scarfflood.js`, VERIFY #748) had already been applied to this clip and is still
in place; it could not reach this. Two reasons, both measured:

1. its flood travels through **candidates only** (Tunic-labelled navy above value 45), and on every frame
   of f20-f26 the number of candidates 4-adjacent to a texel already labelled Scarf is **zero** - what
   sits between them is the scarf's own dark fold, painted below 45 and labelled Tunic;
2. it could seed from the scarf a frame already had, but only when that frame had lost **more than half**
   of it (`areas[f] < 0.5 * ref`), so it fired on the frames that had almost no scarf and skipped the
   ones that had most of one - a fix applied to some frames of a clip and not others, which is the
   flicker rather than a repair of it.

### The repair

`scarfflood.js --bridge 25`, added here. Navy at or above value 25 - whatever the mask currently calls
it - is split into connected **regions**, and a region whose top sits at or above the head's bottom is
the scarf, whole. Height rather than "touches the head", because on f27 and f28 the far end of the drape
touches neither the head nor a Scarf texel (the sword crosses between them) and a head-touch test
claimed it on f23-f26 and refused it on f27-f28 - the same partial-frame fix in a new shape. The skirt
is the same navy but never reaches above the head, so it is never claimed; `--cap` still refuses a
region that runs away.

**Run twice, the second run moves 0 texels and the pages are byte-identical.**

### Before -> after, warrior `hurt`

Same render harness, same `plausibleTeal` loadout, sheets in `Logs/audit-after/`:

| measure | before | after |
|---|---:|---:|
| `hurt` render-flicker, 1:1 | 0.640% | **0.385%** |
| `hurt` biggest patch, 1:1 | **351** | **12** |
| `hurt` worst pair | f22->f23 2.37% | f15->f16 0.73% |
| `hurt` render-flicker, 0.43x (the fight's own scale) | 0.554% | **0.274%** |
| `hurt` biggest patch, 0.43x | 65 | **8** |
| **Scarf**, share of its own still pixels | **5.83%** | **2.05%** |
| **Scarf**, worst pair | f22->f23 **68.7%** | f12->f13 13.6% |
| **Scarf**, biggest patch | **349** | **8** |
| Tunic, share / patch | 1.32% / 273 | 0.79% / 7 |
| reversion (hue 195-245), clip / worst frame | 1.11% / 7.7% @f4 | 1.11% / 7.7% @f4 |
| label-level `shimmer.js`, share / patch | 2.877% / 560 | 2.554% / 209 |
| `headbutt` (same page, cells untouched) | 0.348% / 9 | 0.348% / 9 |

The 351-pixel patch was the biggest single flicker patch in all 100 clips; it is now 12. The second
gate (reversion) did not move, which is the point - the texels moved between two parts that BOTH take a
dial, so nothing was handed to a part that does not dye. `headbutt` shares page 3 and is byte-identical,
which is the check that only `hurt`'s cells were written.

**Pictures** (`Logs/audit-png/`, rendered with his teal scarf through the real shader):

- `warrior-hurt-f22-26-BEFORE-over-AFTER.png` - the one to look at. Top row before, bottom row after,
  frames 22-26. The scarf's drape over his right shoulder is **dark crimson on f23, f24, f25 and f26**
  before, with only a thin teal curl left at the neck, and **teal on every frame** after.
- `warrior-hurt-tealscarf-BEFORE.png` / `warrior-hurt-tealscarf-AFTER.png` - all 29 frames.
- `warrior-hurt-f20-28-BEFORE-over-AFTER.png` - the wider strip.


### Applied to `hurt` only, and why

The same rule was run over the eight other warrior clips whose Scarf or Tunic flicker is in the same
band (`roll spinStrike shieldShove shieldShove2 headbutt duck death spinKick`) and judged at label level
with `shimmer.js` before rendering, as the README requires. Only `hurt` improved:

| clip | flicker before -> after | patch before -> after | verdict |
|---|---|---|---|
| `hurt` | 2.877% -> **2.554%** | 560 -> **209** | kept |
| `roll` | 6.119% -> 6.153% | 490 -> 759 | rejected |
| `death` | 2.653% -> 2.841% | 320 -> 631 | rejected |
| `duck` | 2.475% -> 2.621% | 168 -> 168 | rejected |
| `headbutt` | 3.486% -> 3.528% | 208 -> 208 | rejected |
| `shieldShove` | 3.187% -> 3.211% | 262 -> 262 | rejected |
| `shieldShove2` | 3.144% -> 3.158% | 248 -> 248 | rejected |
| `spinStrike` | 4.586% -> 4.581% | 539 -> 539 | neutral, not taken |
| `spinKick` | 3.635% -> 3.624% | 542 -> 542 | neutral, not taken |

A clip that does not measure better is not touched. On `roll` and `death` the scarf genuinely passes in
front of the skirt, so a region above the head's bottom is not only the scarf there.

---

## Other things this audit found and did NOT fix

- **`male warrior jump` and `male mage jump` have completely EMPTY masks.** Every one of the warrior's
  580,162 ink pixels on that clip carries label 0, and the mage's the same. Those two clips render
  **entirely undyed** - the whole outfit snaps back to the drawn colours for the length of the jump.
  `revertscan` reads the warrior's `jump` at **99.58% undyed, 100% on f3**. This is known
  (`tools/masks/README.md`: "a clip whose committed mask is EMPTY (the two `jump` clips) cannot be
  relabelled - there is nothing to learn the colours from") and needs a mask by some other route, not a
  mask repair. It is a constant wrong colour rather than a shimmer, which is why no flicker number
  reports it.
- **`energySwordSwipe` 21.9% and `swordBlast` 3.5% "undyed"** on the warrior's navy band are the blue
  energy effect, which is correctly left alone. Not a defect.
- **The mage's fine speckle** (1.0-1.7% over 22 clips, patches of 8-26 px) is the residue of the art's
  own boil, already treated with a temporal median on his idles in #1205. It is the thing the gate ranks
  first and the least visible thing on the list.
- **The rogue's `arrow`/`jump`/`spinAttack` gold and Weapons rows** in `renderparts` are 20-54% of a
  100-300 pixel part - a thin bright object whose sub-pixel edge is re-decided each frame. Small patches
  (6-30 px), the dagger-shimmer shape already documented.

---

## For the lead

`Assets/Resources/CharacterMasks/male-warrior-3.png` is the only asset changed (`hurt` and `headbutt`
live on page 3; only `hurt`'s cells were written). **The EditMode suite was NOT run** - this worktree's
assets are stale against main in other folders and it invents phantom failures. Run
`AtlasManifestTests` / `ArtDehaloTests` on main after merging.
