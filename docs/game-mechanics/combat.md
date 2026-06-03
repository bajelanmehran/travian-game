# Combat Mechanics

> Research based on custom-built battle implementation inspired by Travian's observable combat behavior.  
> All code referenced is original work by the author.

---

## Overview

Combat in Travian is resolved server-side at the moment a movement arrives at its destination. The `BattleModel` handles all combat resolution including troop losses, wall damage, catapult/ram effects, loot calculation, and battle report generation.

---

## Unit Stats

Each tribe has 10 unit slots. Stats are stored in three arrays:

- `$off` — offensive power per unit
- `$def_i` — defensive power vs infantry
- `$def_c` — defensive power vs cavalry

### Tribe IDs
| ID | Tribe |
|---|---|
| 1 | Romans |
| 2 | Teutons |
| 3 | Gauls |
| 4 | Huns (Egyptians) |
| 5 | Egyptians (Spartans) |
| 6 | Spartans |
| 7 | (Additional tribe) |

### Example — Roman unit offensive values (tribe 1):
| Unit slot | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 |
|---|---|---|---|---|---|---|---|---|---|---|
| Off | 40 | 30 | 70 | 0 | 120 | 180 | 60 | 75 | 50 | 0 |

Unit 4 (scout) has 0 offense — it uses a separate scout combat system.

---

## Combat Constants

```
BASE_VILLAGE_DEF    = 10     // Minimum defense every village has
SCOUT_DEF           = 20     // Scout defensive power
SCOUT_OFF           = 35     // Scout offensive power
LONE_ATTACKER_THRESHOLD = 84 // Below this total offense → attacker dies alone
```

---

## Wall Effects

Each tribe's wall has different durability and base defense multiplier:

| Tribe | Wall durability | Wall base multiplier | Wall extra bonus |
|---|---|---|---|
| Romans | 1 | 1.03 | 10 |
| Teutons | 5 | 1.02 | 6 |
| Gauls | 2 | 1.025 | 8 |
| Huns | 1 | 1.00 | 0 |
| Egyptians | 1 | 1.03 | 10 |
| Spartans | 5 | 1.025 | 8 |
| Tribe 7 | 1 | 1.015 | 6 |

**Wall defense formula:**
```
wall_defense_multiplier = wall_base ^ wall_level
```

Example: Roman wall level 20 → `1.03^20 ≈ 1.806` — adds ~80% to total defense.

---

## Combat Resolution

### Step 1 — Compute total offense

```
total_offense = Σ (unit_count[i] × unit_off[i] × smithy_upgrade_bonus[i])
total_offense × morale_modifier
total_offense × artifact_effect
total_offense × atk_bonus_rate (if attacker has server bonus)
```

### Step 2 — Compute total defense

Defense is split into **infantry defense** and **cavalry defense**:

```
infantry_def = Σ (unit_count[i] × def_i[tribe][i] × smithy_upgrade_bonus[i])
cavalry_def  = Σ (unit_count[i] × def_c[tribe][i] × smithy_upgrade_bonus[i])
```

The effective defense is a **weighted average** based on attacker's cavalry ratio:
```
cavalry_ratio = attacker_cavalry_units / attacker_total_units
effective_def = infantry_def × (1 - cavalry_ratio) + cavalry_def × cavalry_ratio
```

Then apply wall and bonuses:
```
effective_def × wall_base^wall_level
effective_def × artifact_effect
effective_def × def_bonus_rate (if defender has server bonus)
```

### Step 3 — Compute losses

The core loss formula:
```
x = (total_offense / effective_defense) ^ 1.5

attacker_loss_ratio = min(1 / x, 1)
defender_loss_ratio = min(x,     1)  ... simplified
```

This is a **power law** — being 2× stronger means the opponent loses ~2.83× as many troops (`2^1.5 = 2.83`).

### Step 4 — Lone attacker check

If the total offense after morale × `common_morale(pop_ratio)` falls below `LONE_ATTACKER_THRESHOLD` (84), the attacker loses all troops automatically regardless of defense value.

---

## Morale System

Morale is based on **population ratio** between attacker and defender:

```
morale = f(attacker_total_pop / defender_total_pop)
```

- Large player attacking small player → morale penalty on attacker
- Defender starts with `total_pop = 500` as baseline minimum

---

## Scout Combat

Scouts use a separate resolution system:

```
scout_offense = morale × scout_count × SCOUT_OFF × smithy_bonus × artifact_effect
scout_defense = Σ (defender_spy_count × SCOUT_DEF × smithy_bonus) × wall_multiplier × artifact_effect

x = (scout_offense / scout_defense) ^ 1.5
off_losses = min(1 / x, 1)
```

---

## Administrator (Senator/Chief/Chieftain) Effect

Each tribe's administrator reduces loyalty by a random amount per wave:

| Tribe | Min loyalty reduction | Max loyalty reduction |
|---|---|---|
| Romans | 20 | 30 |
| Teutons | 20 | 25 |
| Gauls | 20 | 25 |
| Huns | 0 | 0 |
| Egyptians | 200 | 200 |
| Spartans | 20 | 25 |
| Tribe 7 | 20 | 25 |

Egyptians (tribe 5) have a drastically higher loyalty reduction — one wave can conquer a village instantly.

---

## Ram Damage

Ram damage targets the **wall** (building ID 40):
```
BuildingAction::downgrade(village, building=40, levels_lost, destroy_if_zero)
```

Wall downgrade is affected by:
- Number of surviving rams after battle
- `def_losses` ratio — rams are less effective when attacker takes heavy losses

**High-speed server adjustment** (`getGameSpeed() > 50`):
When the attacker loses all troops (`off_losses == 1`) but defender also loses troops, ram damage is heavily scaled down (multiplied by 0.003–0.015 depending on `def_losses` ratio). If `def_losses < 0.33`, ram damage is zeroed out entirely.

---

## Catapult Damage

Catapults target specific buildings by `gid` (building type ID):
- If a specific target is set → finds a slot with that building
- If target is random (gid=99) → picks any non-protected building randomly

Protected buildings (immune to catapults):
- Building IDs 31, 32, 33, 42, 43

```
BuildingAction::downgrade(village, field, levels_lost, destroy_if_zero)
```

Catapult effectiveness is also subject to the same high-speed server scaling as rams.

---

## Village Destruction

After wall/catapult damage is applied, the system checks if the village's population has reached 0:
```
if (village_pop == 0) → attempt village deletion
```

Deletion conditions are validated by `AccountDeleter::isVillageDestroyAble()` — checking attacker/defender ownership rules before permanently removing the village from the map.

---

## Troop Upkeep

Each unit consumes crop per hour (upkeep), by tribe:

| Tribe | U1 | U2 | U3 | U4 | U5 | U6 | U7 | U8 | U9 | U10 | Hero |
|---|---|---|---|---|---|---|---|---|---|---|---|
| Romans | 1 | 1 | 1 | 2 | 3 | 4 | 3 | 6 | 5 | 1 | 6 |
| Teutons | 1 | 1 | 1 | 1 | 2 | 3 | 3 | 6 | 4 | 1 | 6 |
| Gauls | 1 | 1 | 2 | 2 | 2 | 3 | 3 | 6 | 4 | 1 | 6 |
| Huns | 1 | 1 | 1 | 1 | 2 | 2 | 3 | 3 | 3 | 5 | 0 |
| Egyptians | 1 | 1 | 1 | 1 | 2 | 3 | 6 | 5 | 0 | 0 | 6 |

---

## Cavalry Classification

These unit slots are classified as cavalry (affects defense weighting):

| Tribe | Cavalry unit slots |
|---|---|
| Romans | 4, 5, 6 |
| Teutons | 4, 5, 6 |
| Gauls | 3, 4, 5, 6 |
| Huns | — (none) |
| Egyptians | 5, 6 |
| Spartans | 4, 5, 6 |
| Tribe 7 | 4, 5, 6 |
