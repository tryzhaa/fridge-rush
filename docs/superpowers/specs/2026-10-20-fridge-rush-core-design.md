# Fridge Rush — Core Build Design

Builds on `STRATEGY.md` (the original game plan). This doc pins the exact
values and interfaces an implementation plan needs that the strategy doc
left open. Decisions below that aren't attributed to the human partner are
the planner's call — a reasonable default, not something asked and agreed.

## Decisions confirmed with the human partner

- Pre-made code and assets are allowed by the hackathon's rules, so this
  plan builds the real codebase now, not just a build guide.
- The event date is not yet scheduled; phases stay relative (Hour 0–2,
  etc.), not pinned to a calendar date.
- No Supabase project exists yet; the plan includes creating one.
- The AI chef feature is out of scope for this plan entirely (not even as
  a stretch task).

## Goals

- A complete single-player core (deliveries, spoiling, cooking, scoring)
  playable standalone before any networking code exists.
- Local 2-player (shared screen/keyboard) as the guaranteed, offline demo
  fallback.
- Online race (Supabase Realtime broadcast, room codes) as the preferred
  mode when both laptops have working internet.
- Real-ish CO₂/water-footprint data driving both recipe scoring and the
  end-screen comparison.

## Non-goals / anti-goals

- No farming mechanic (dropped from the original brainstorm).
- No full real-time world sync between players — only score/attack
  events cross the wire.
- No backend beyond Supabase — the build output is a static bundle.
- No meat or dairy sprites, even if a chosen pack includes them.

## Match structure

- Match length: **180 seconds** (3 minutes), countdown timer per match.
- Fridge grid: **5 columns × 4 rows = 20 cells**.
- Conveyor delivers one ingredient every **4–6 seconds** (randomized
  within that range), drawn from a **seeded** sequence so local and
  online matches are reproducible/fair.
- Spoil curve, same for every ingredient (v1 — per-ingredient rates are
  a later stretch, not in this plan):
  - 0–45s after delivery: `fresh`
  - 45–90s after delivery: `wilted`
  - 90s+: `rotten` (counts toward that player's waste total; cannot be
    used in a recipe)
- A rotten item occupies its fridge cell until the player removes it
  (removing it is a manual action, not automatic).

## Ingredient data (15 items)

All 16×16, plant-only, pulled from the three itch.io packs named in
`STRATEGY.md`. CO₂ and water-footprint values below are **placeholder
estimates** (order-of-magnitude plausible, not sourced yet) — the
implementation plan has an explicit step to replace them with cited
figures from Our World in Data (Poore & Nemecek) and the Water Footprint
Network before the event. Do not treat these numbers as final or quote
them publicly until that step is done.

| id | name | co2PerKg (kg CO₂e, placeholder) | waterPerKg (L, placeholder) |
| --- | --- | --- | --- |
| potato | Potato | 0.3 | 290 |
| carrot | Carrot | 0.4 | 195 |
| onion | Onion | 0.5 | 280 |
| leek | Leek | 0.5 | 200 |
| eggplant | Eggplant | 0.4 | 362 |
| broccoli | Broccoli | 0.4 | 285 |
| bell_pepper | Bell Pepper | 1.0 | 379 |
| tomato | Tomato | 1.4 | 214 |
| lettuce | Lettuce | 0.3 | 237 |
| mushroom | Mushroom | 0.7 | 322 |
| apple | Apple | 0.4 | 822 |
| banana | Banana | 0.9 | 790 |
| lemon | Lemon | 0.4 | 753 |
| garlic | Garlic | 0.5 | 589 |
| rice | Rice | 4.0 | 2497 |

## Recipe data (10 recipes)

Each recipe takes 2–3 ingredients in the pot (order doesn't matter, no
duplicates within one recipe). A recipe's `co2Saved` and `waterSaved` are
derived at data-load time as the sum of its ingredients' per-kg values
(1 kg assumed per ingredient) — not hand-entered, so they can never drift
from the ingredient table. `points` is a flat value per recipe, higher
for more/rarer ingredients. `tip` text is a placeholder pending real
copy sourced from UNEP's Food Waste Index — do not ship placeholder tips
to the end screen as if they were sourced facts.

| id | name | ingredients | points |
| --- | --- | --- | --- |
| baked_potato_broccoli | Baked Potato & Broccoli Mash | potato, broccoli | 10 |
| garden_salad | Garden Salad | lettuce, tomato, carrot | 15 |
| leek_potato_soup | Leek & Potato Soup | leek, potato, onion | 15 |
| ratatouille_bites | Ratatouille Bites | eggplant, tomato, bell_pepper | 20 |
| mushroom_garlic_rice | Mushroom Garlic Rice | mushroom, garlic, rice | 20 |
| pepper_onion_stirfry | Pepper & Onion Stir-Fry | bell_pepper, onion, garlic | 15 |
| apple_lemon_salad | Apple-Lemon Salad | apple, lemon, lettuce | 15 |
| banana_rice_bowl | Banana Rice Bowl | banana, rice, lemon | 15 |
| veggie_rice_medley | Veggie Rice Medley | rice, carrot, broccoli | 20 |
| root_veg_soup | Root Veg Soup | carrot, onion, potato | 15 |

Validation rule: no two recipes may share the exact same ingredient
multiset (ambiguous — the pot wouldn't know which one the player meant).

## Scoring & end screen

- `points`: sum of completed recipes' point values.
- `co2SavedKg`: sum of completed recipes' derived `co2Saved`.
- `wasteCount`: number of items that crossed into `rotten` and were
  never used in a recipe.
- End-screen driving comparison: `drivingKmEquivalent = co2SavedKg /
  CO2_PER_KM` where `CO2_PER_KM = 0.17` (kg CO₂e per km, average
  passenger car — also a placeholder pending a cited source).

## Head-to-head mechanics

- **Attack trigger:** completing any recipe immediately sends the
  opponent a "bulk delivery" — 3 extra ingredients pushed onto their
  conveyor's queue front-to-back over the next 2 seconds (not all at
  once), so their grid can realistically absorb or reject them.
- **Local 2-player:** both fridges/conveyors run in the same process;
  an attack is a direct in-memory call from one player's session to the
  other's.
- **Online race:** each client runs its own full game; only three
  message types cross via Supabase Realtime broadcast on a room-coded
  channel: `match_start` (carries the shared delivery seed), `score_update`
  (periodic, carries that player's summary), and `attack` (carries
  nothing but the sender — the receiver generates its own 3-item pull
  from its *own* next-in-sequence deliveries, so no ingredient data needs
  to cross the wire).
- Fairness: both clients derive their conveyor sequence from the same
  `match_start` seed, so "same deliveries, same order" holds without
  syncing every spawn event.

## Non-functional constraints (carried from `STRATEGY.md`)

- Single sprite size (16×16) across all assets.
- No meat/dairy sprites even where a source pack includes them.
- Local 2-player must work with zero internet connectivity.
- Static build output only (Vercel or itch.io hosting).
- Credit FrozenTaurus's pack (required by its license) and Alex Kovacs's
  pack (license terms unconfirmed — verify before shipping) on the end
  screen; Karsiori's CC0 pack doesn't require it but gets credited anyway.
