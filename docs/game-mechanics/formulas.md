# Hero & Building Mechanics

> Research based on custom-built implementation inspired by Travian's observable behavior.

---

## Hero Regeneration Cost

Hero resurrection cost scales with hero level and varies by tribe.

### Base resource costs per tribe (at level 0):

| Tribe ID | Wood | Clay | Iron | Crop |
|---|---|---|---|---|
| 1 (Romans) | 130 | 115 | 180 | 75 |
| 2 (Teutons) | 180 | 130 | 115 | 75 |
| 3 (Gauls) | 115 | 180 | 130 | 75 |
| 5 | 130 | 115 | 180 | 75 |
| 6 | 115 | 180 | 130 | 75 |
| 7 | 125 | 125 | 125 | 125 |

### Cost formula:

```
cost[resource] = tribe_base[resource] × (1 + level/24) × (1 + level)
```

Level is capped at 100 for cost calculation purposes.

### Rounding rules (`trickyRounding`):

| Hero level | Rounding |
|---|---|
| < 5 | Round to nearest 10 |
| 5 – 9 | Round to nearest 50 |
| ≥ 10 | Round to nearest 100 |

### Example costs (Romans, tribe 1):

| Level | Wood | Clay | Iron | Crop |
|---|---|---|---|---|
| 0 | 130 | 115 | 180 | 75 |
| 10 | ~2,730 | ~2,415 | ~3,780 | ~1,575 |
| 50 | ~41,600 | ~36,800 | ~57,600 | ~24,000 |
| 100 | ~109,200 | ~96,600 | ~151,200 | ~63,000 |

---

## Hero Adventure Duration

```
duration = round(min(level + 1, 24) / floor(gameSpeed / 3 + 1) × 3600 × rate)
```

- `rate` = 1000 if milliseconds mode enabled, else 1
- Duration is capped at hero level 23 (min(level+1, 24) maxes out at 24)
- Higher game speed → shorter adventures

---

## Hero Item Tiers

Items available in adventures scale with server age:

| Tier | Server day range (1x speed) |
|---|---|
| 1 | Days 0 – 75 |
| 2 | Days 75 – 165 |
| 3 | Day 165+ |

Formula (adjusts proportionally for server length):
```
tier1_threshold = 75/350 × round_length_days × 86400 seconds
tier2_threshold = 165/350 × round_length_days × 86400 seconds
```

---

## Building Construction

### Upgrade cost formula

Each building level cost is calculated from base cost using a growth factor `k`:

```
cost[level] = base_cost × k^(level - 1)
```

### Build time formula

```
time[level] = (a × k^(level-1) + b) / gameSpeed
```

Where `a`, `k`, `b` are building-specific constants.

### Resource field parameters:

| Building | Base cost (W/C/I/Crop) | k | Max level |
|---|---|---|---|
| Woodcutter | 40/100/50/60 | 1.67 | 20 |
| Clay pit | 80/40/80/50 | 1.67 | 21 |
| Iron mine | 100/80/30/60 | 1.67 | 20 |
| Cropland | 70/90/70/20 | 1.67 | 21 |

### Processing building parameters:

| Building | Base cost (W/C/I/Crop) | k | Max level | Requires |
|---|---|---|---|---|
| Sawmill | 520/380/290/90 | 1.80 | 5 | Woodcutter L10, Main Bldg L5 |
| Brickyard | 440/480/320/50 | 1.80 | 5 | Clay pit L10, Main Bldg L5 |
| Iron foundry | 200/450/510/120 | 1.80 | 5 | Iron mine L10, Main Bldg L5 |

Note: Processing buildings use a higher growth factor (1.80 vs 1.67) — they become exponentially more expensive per level.

---

## Festival

### Duration:
```
duration = 72 hours (3 days)

if gameSpeed > 20:
    if gameSpeed ≤ 100: duration /= gameSpeed
    else: duration /= (gameSpeed / 2)

minimum duration = 300 seconds (5 minutes)
```

### Resource cost:
```
base = [3870 Wood, 1680 Clay, 5940 Iron, 1340 Crop]

if gameSpeed > 20:
    multiply each by ceil(gameSpeed / 20)
```

---

## Alliance Bonus

### Contribution thresholds for each level:

| Level | Additional contributions needed | Total cumulative |
|---|---|---|
| 1 | 2,400,000 | 2,400,000 |
| 2 | 19,200,000 | 21,600,000 |
| 3 | 38,400,000 | 60,000,000 |
| 4 | 76,800,000 | 136,800,000 |
| 5 | 153,600,000 | 290,400,000 |

On high-speed servers (`gameSpeed > 20`): all thresholds multiplied by `ceil(gameSpeed / 1000)`.

### Donation limits per level:

| Level | Daily donation limit (1x) |
|---|---|
| 0 | 300,000 |
| 1 | 400,000 |
| 2 | 550,000 |
| 3 | 750,000 |
| 4–5 | 1,000,000 |

Limit scales with `gameSpeed`: `limit × gameSpeed`.

### Upgrade duration:
```
duration = round((level × 86400) / gameSpeed)
```

### New player cooldown:
```
cooldown = ceil((level - 2) × 86400 / gameSpeed)
```

---

## Oasis Storage

```
max_storage = 1000 × storage_multiplier

if oasis_type in [3, 7, 11, 15] (crop oases):
    max_storage × 2
```

---

## Artifacts — Release Time

```
base_rate = 14/365 × round_length_days × 86400

if gameSpeed ≤ 1: rate = 14 days
if gameSpeed ≤ 2: rate = 10 days
if gameSpeed ≤ 10: rate = 7 days

release_time = server_start_time + rate
```

---

## Village Field Layouts

Each field type (`fieldtype`) defines the resource type of each of the 18 field slots.

Resource type mapping:
- 1 = Wood
- 2 = Clay
- 3 = Iron
- 4 = Crop

### Summary of field distributions:

| Field type | Wood | Clay | Iron | Crop |
|---|---|---|---|---|
| 1 | 3 | 3 | 3 | 9 |
| 2 | 3 | 4 | 5 | 6 |
| 3 | 4 | 4 | 4 | 6 |
| 4 | 4 | 5 | 3 | 6 |
| 5 | 5 | 3 | 4 | 6 |
| 6 | 1 | 1 | 1 | 15 (15-crop) |
| 7 | 4 | 4 | 3 | 7 |
| 8 | 3 | 4 | 4 | 7 |
| 9 | 4 | 3 | 4 | 7 |
| 10 | 3 | 5 | 4 | 6 |
| 11 | 4 | 3 | 5 | 6 |
| 12 | 5 | 4 | 3 | 6 |

Field type 6 is the legendary **15-crop village** — prized for feeding large armies.

---

## Oasis Bonus Types

| Oasis type ID | Bonus |
|---|---|
| 2 | +25% Wood |
| 3 | +25% Wood, +25% Crop |
| 4 | +50% Wood |
| 6 | +25% Clay |
| 7 | +25% Clay, +25% Crop |
| 8 | +50% Clay |
| 10 | +25% Iron |
| 11 | +25% Iron, +25% Crop |
| 12 | +50% Iron |
| 14 | +25% Crop |
| 15 | +50% Crop |
