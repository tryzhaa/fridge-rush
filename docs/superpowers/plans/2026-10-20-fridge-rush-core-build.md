# Fridge Rush Core Build Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build Fridge Rush end-to-end — single-player core, local 2-player, and Supabase-backed online race — as a deployable static browser game.

**Architecture:** Pure game-logic modules (ingredient/recipe data, spoil timing, fridge grid, conveyor sequencing, pot matching, scoring) are built and unit-tested independently of Phaser, then wired into Phaser scenes that render and handle input. Networking is a thin broadcast-only wrapper around Supabase Realtime so only three event types ever cross the wire, keeping single-player and local 2-player fully playable even if networking is never reached.

**Tech Stack:** Phaser 4, TypeScript, Vite, Vitest, @supabase/supabase-js (Realtime broadcast only, no custom server).

**Spec:** `docs/superpowers/specs/2026-10-20-fridge-rush-core-design.md`

## Global Constraints

- Match length: 180000 ms (3 minutes) per match.
- Fridge grid: 5×4 = 20 cells, fixed.
- Spoil curve, identical for every ingredient: `fresh` 0–45000 ms, `wilted` 45000–90000 ms, `rotten` ≥90000 ms since delivery.
- Conveyor delivers one item every 4000–6000 ms, drawn from a seeded PRNG shared by both players in a match.
- Sprites are 16×16 only; no meat/dairy sprites even if a source pack includes them.
- Online messages are limited to exactly three broadcast event types — `match_start`, `score_update`, `attack` — no ingredient-level data crosses the wire.
- `CO2_PER_KM = 0.17` kg CO₂e/km for the driving-equivalent stat (placeholder pending a cited source).
- Build output is a static bundle; no backend beyond Supabase.
- Local 2-player must run with zero network connectivity.

## Review Focus

- Dragging an item onto an already-occupied fridge cell: `placeItem` must reject it, not overwrite the existing item.
- Trying to cook with a rotten ingredient in the pot: `canAddToPot` must block it before `matchRecipe` ever sees it.
- Two recipes sharing an identical ingredient multiset: ambiguous for the pot to resolve — the shipped recipe data must have none.
- An item crossing into `rotten` getting counted as waste more than once if the per-frame check runs repeatedly: each cell index must surface at most once.
- An online match where the host's `match_start` broadcast never arrives (dropped Wi-Fi): the joining client must time out with a visible message and a path back to local 2-player, not hang silently.

---

### Task 1: Project scaffold

**Files:**
- Create: `package.json`, `vite.config.ts`, `tsconfig.json`, `vitest.config.ts`, `index.html`
- Create: `src/main.ts`

**Interfaces:**
- Produces: a Vite dev server and a Vitest runner later tasks' tests run under.

- [ ] **Step 1: Initialize the project**

`npm init -y`, then install `phaser@4.0.0`, `vite@7.x`, `typescript@5.x`, `vitest@3.x`, `@supabase/supabase-js@2.x` (exact patch versions: latest at install time). Add `package.json` scripts: `dev` → `vite`, `build` → `vite build`, `preview` → `vite preview`, `test` → `vitest run`.

- [ ] **Step 2: Create config files**

`vite.config.ts` (default Vite + TS setup), `tsconfig.json` (`strict: true`, `target: ES2020`), `vitest.config.ts` (node environment, no Phaser/DOM needed since only logic modules get automated tests), `index.html` with a single `#game` div and a `<script type="module" src="/src/main.ts">`.

- [ ] **Step 3: Boot a blank Phaser game**

In `src/main.ts`, construct `new Phaser.Game({ type: Phaser.AUTO, width: 960, height: 600, parent: 'game', scene: [] })`.

- [ ] **Step 4: Verify dev server runs**

Run: `npm run dev`
Expected: server starts, browser shows a blank canvas with no console errors.

- [ ] **Step 5: Commit**

```bash
git add package.json package-lock.json vite.config.ts tsconfig.json vitest.config.ts index.html src/main.ts
git commit -m "chore: scaffold Vite + Phaser 4 + TypeScript + Vitest project"
```

---

### Task 2: Ingredient data module

**Files:**
- Create: `src/data/types.ts`
- Create: `src/data/ingredients.ts`
- Test: `tests/data/ingredients.test.ts`

**Interfaces:**
- Produces: `interface Ingredient { id: string; name: string; spriteKey: string; co2PerKg: number; waterPerKg: number }`, and `export const ingredients: Ingredient[]`.

- [ ] **Step 1: Write the failing test**

```ts
import { ingredients } from '../../src/data/ingredients';

test('has exactly 15 ingredients with unique ids and positive footprint values', () => {
  expect(ingredients).toHaveLength(15);
  const ids = ingredients.map(i => i.id);
  expect(new Set(ids).size).toBe(15);
  for (const ing of ingredients) {
    expect(ing.co2PerKg).toBeGreaterThan(0);
    expect(ing.waterPerKg).toBeGreaterThan(0);
  }
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run tests/data/ingredients.test.ts`
Expected: FAIL — module not found.

- [ ] **Step 3: Implement the data file**

Define `Ingredient` in `src/data/types.ts`. In `src/data/ingredients.ts`, export the 15-row table from the spec's Ingredient Data section verbatim (id, name, spriteKey = id, co2PerKg, waterPerKg). Add a top-of-file comment: `// PLACEHOLDER: co2PerKg/waterPerKg are estimates — replace with cited OWID / Water Footprint Network figures before the event.`

- [ ] **Step 4: Run test to verify it passes**

Run: `npx vitest run tests/data/ingredients.test.ts`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add src/data/types.ts src/data/ingredients.ts tests/data/ingredients.test.ts
git commit -m "feat: add ingredient data with footprint placeholders"
```

---

### Task 3: Recipe data module

**Files:**
- Modify: `src/data/types.ts`
- Create: `src/data/recipes.ts`
- Test: `tests/data/recipes.test.ts`

**Interfaces:**
- Consumes: `ingredients: Ingredient[]` from Task 2.
- Produces: `interface Recipe { id: string; name: string; ingredientIds: string[]; points: number; tip: string }`, `export const recipes: Recipe[]`, `interface RecipeWithDerived extends Recipe { co2Saved: number; waterSaved: number }`, `export function buildRecipeCatalog(ingredients: Ingredient[], recipes: Recipe[]): RecipeWithDerived[]`.

- [ ] **Step 1: Write the failing test**

```ts
import { ingredients } from '../../src/data/ingredients';
import { recipes, buildRecipeCatalog } from '../../src/data/recipes';

test('has 10 recipes, all referencing real ingredients, no duplicate ingredient sets', () => {
  expect(recipes).toHaveLength(10);
  const ingredientIds = new Set(ingredients.map(i => i.id));
  const seenSets = new Set<string>();
  for (const r of recipes) {
    for (const id of r.ingredientIds) expect(ingredientIds.has(id)).toBe(true);
    const key = [...r.ingredientIds].sort().join(',');
    expect(seenSets.has(key)).toBe(false);
    seenSets.add(key);
  }
});

test('buildRecipeCatalog derives co2Saved as the sum of ingredient co2PerKg', () => {
  const catalog = buildRecipeCatalog(ingredients, recipes);
  const rootVegSoup = catalog.find(r => r.id === 'root_veg_soup')!;
  expect(rootVegSoup.co2Saved).toBeCloseTo(1.2, 5); // carrot 0.4 + onion 0.5 + potato 0.3
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run tests/data/recipes.test.ts`
Expected: FAIL — module not found.

- [ ] **Step 3: Implement the data file**

Add `Recipe`/`RecipeWithDerived` to `src/data/types.ts`. In `src/data/recipes.ts`, export the 10-row table from the spec's Recipe Data section verbatim, with `tip: 'PLACEHOLDER — add real tip from UNEP Food Waste Index'` for every entry. Implement `buildRecipeCatalog` by summing matched ingredients' `co2PerKg`/`waterPerKg` for each recipe's `ingredientIds`.

- [ ] **Step 4: Run test to verify it passes**

Run: `npx vitest run tests/data/recipes.test.ts`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add src/data/types.ts src/data/recipes.ts tests/data/recipes.test.ts
git commit -m "feat: add recipe data with derived footprint totals"
```

---

### Task 4: Spoil stage logic

**Files:**
- Create: `src/systems/spoil.ts`
- Test: `tests/systems/spoil.test.ts`

**Interfaces:**
- Produces: `export const FRESH_MS = 45000`, `export const ROTTEN_MS = 90000`, `export type SpoilStage = 'fresh' | 'wilted' | 'rotten'`, `export function getSpoilStage(deliveredAtMs: number, nowMs: number): SpoilStage`.

- [ ] **Step 1: Write the failing test**

```ts
import { getSpoilStage } from '../../src/systems/spoil';

test.each([
  [0, 0, 'fresh'], [0, 44999, 'fresh'],
  [0, 45000, 'wilted'], [0, 89999, 'wilted'],
  [0, 90000, 'rotten'], [0, 999999, 'rotten'],
])('deliveredAt=%i now=%i -> %s', (deliveredAt, now, stage) => {
  expect(getSpoilStage(deliveredAt, now)).toBe(stage);
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run tests/systems/spoil.test.ts`
Expected: FAIL — module not found.

- [ ] **Step 3: Implement `getSpoilStage`**

`elapsed = nowMs - deliveredAtMs`; return `'fresh'` if `elapsed < FRESH_MS`, `'wilted'` if `elapsed < ROTTEN_MS`, else `'rotten'`.

- [ ] **Step 4: Run test to verify it passes**

Run: `npx vitest run tests/systems/spoil.test.ts`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add src/systems/spoil.ts tests/systems/spoil.test.ts
git commit -m "feat: add spoil stage timing logic"
```

---

### Task 5: Fridge grid

**Files:**
- Create: `src/systems/FridgeGrid.ts`
- Test: `tests/systems/FridgeGrid.test.ts`

**Interfaces:**
- Consumes: `getSpoilStage`, `SpoilStage` from Task 4.
- Produces: `export const CELL_COUNT = 20`, `export class FridgeGrid { placeItem(cellIndex: number, ingredientId: string, deliveredAtMs: number): boolean; removeItem(cellIndex: number): { ingredientId: string; deliveredAtMs: number } | null; getItem(cellIndex: number): { ingredientId: string; deliveredAtMs: number } | null; getStage(cellIndex: number, nowMs: number): SpoilStage | 'empty'; collectNewlyRotten(nowMs: number): number[] }`. Later tasks (Pot UI, GameScene, LocalMatchScene) call these methods by these exact names.

- [ ] **Step 1: Write the failing test**

```ts
import { FridgeGrid, CELL_COUNT } from '../../src/systems/FridgeGrid';

test('placeItem occupies an empty cell and rejects an occupied one', () => {
  const grid = new FridgeGrid();
  expect(grid.placeItem(0, 'carrot', 0)).toBe(true);
  expect(grid.placeItem(0, 'onion', 1000)).toBe(false);
  expect(grid.getItem(0)?.ingredientId).toBe('carrot');
});

test('getStage reflects spoil.ts thresholds', () => {
  const grid = new FridgeGrid();
  grid.placeItem(1, 'carrot', 0);
  expect(grid.getStage(1, 0)).toBe('fresh');
  expect(grid.getStage(1, 45000)).toBe('wilted');
  expect(grid.getStage(1, 90000)).toBe('rotten');
  expect(grid.getStage(2, 0)).toBe('empty');
});

test('collectNewlyRotten reports each cell at most once', () => {
  const grid = new FridgeGrid();
  grid.placeItem(3, 'carrot', 0);
  expect(grid.collectNewlyRotten(90000)).toEqual([3]);
  expect(grid.collectNewlyRotten(95000)).toEqual([]);
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run tests/systems/FridgeGrid.test.ts`
Expected: FAIL — module not found.

- [ ] **Step 3: Implement `FridgeGrid`**

Back it with a `(Array<{ ingredientId, deliveredAtMs, rottenCounted: boolean } | null>)(CELL_COUNT).fill(null)`. `collectNewlyRotten` iterates cells, and for any cell at `getSpoilStage(...) === 'rotten'` whose `rottenCounted` is still false, sets it true and includes that index in the returned array.

- [ ] **Step 4: Run test to verify it passes**

Run: `npx vitest run tests/systems/FridgeGrid.test.ts`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add src/systems/FridgeGrid.ts tests/systems/FridgeGrid.test.ts
git commit -m "feat: add fridge grid with spoil-aware cell state"
```

---

### Task 6: Conveyor delivery sequence

**Files:**
- Create: `src/systems/conveyor.ts`
- Test: `tests/systems/conveyor.test.ts`

**Interfaces:**
- Produces: `export interface DeliveryPlanEntry { ingredientId: string; offsetMs: number }`, `export function generateDeliveryPlan(seed: number, count: number, ingredientIds: string[]): DeliveryPlanEntry[]`.

- [ ] **Step 1: Write the failing test**

```ts
import { generateDeliveryPlan } from '../../src/systems/conveyor';

const ids = ['carrot', 'onion', 'potato'];

test('same seed produces an identical plan', () => {
  const a = generateDeliveryPlan(42, 10, ids);
  const b = generateDeliveryPlan(42, 10, ids);
  expect(a).toEqual(b);
});

test('offsets are strictly increasing with 4000-6000ms gaps', () => {
  const plan = generateDeliveryPlan(42, 10, ids);
  expect(plan).toHaveLength(10);
  for (let i = 1; i < plan.length; i++) {
    const gap = plan[i].offsetMs - plan[i - 1].offsetMs;
    expect(gap).toBeGreaterThanOrEqual(4000);
    expect(gap).toBeLessThanOrEqual(6000);
  }
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run tests/systems/conveyor.test.ts`
Expected: FAIL — module not found.

- [ ] **Step 3: Implement `generateDeliveryPlan`**

Implement a small seeded PRNG (e.g. mulberry32) seeded from `seed`, returning floats in `[0, 1)`. For each of `count` entries, pick `ingredientIds[Math.floor(rand() * ingredientIds.length)]` and an offset gap of `4000 + rand() * 2000` added to a running cumulative total (first entry's `offsetMs` is its own gap from match start, i.e. cumulative starts at the first gap, not 0).

- [ ] **Step 4: Run test to verify it passes**

Run: `npx vitest run tests/systems/conveyor.test.ts`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add src/systems/conveyor.ts tests/systems/conveyor.test.ts
git commit -m "feat: add seeded conveyor delivery sequence"
```

---

### Task 7: Pot matcher

**Files:**
- Create: `src/systems/potMatcher.ts`
- Test: `tests/systems/potMatcher.test.ts`

**Interfaces:**
- Consumes: `RecipeWithDerived` from Task 3, `SpoilStage` from Task 4.
- Produces: `export function matchRecipe(potIngredientIds: string[], recipes: RecipeWithDerived[]): RecipeWithDerived | null`, `export function canAddToPot(stage: SpoilStage | 'empty'): boolean`.

- [ ] **Step 1: Write the failing test**

```ts
import { ingredients } from '../../src/data/ingredients';
import { recipes, buildRecipeCatalog } from '../../src/data/recipes';
import { matchRecipe, canAddToPot } from '../../src/systems/potMatcher';

const catalog = buildRecipeCatalog(ingredients, recipes);

test('matchRecipe finds a recipe regardless of ingredient order', () => {
  expect(matchRecipe(['onion', 'carrot', 'potato'], catalog)?.id).toBe('root_veg_soup');
  expect(matchRecipe(['carrot', 'onion'], catalog)).toBeNull();
});

test('canAddToPot blocks only rotten items', () => {
  expect(canAddToPot('fresh')).toBe(true);
  expect(canAddToPot('wilted')).toBe(true);
  expect(canAddToPot('rotten')).toBe(false);
  expect(canAddToPot('empty')).toBe(false);
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run tests/systems/potMatcher.test.ts`
Expected: FAIL — module not found.

- [ ] **Step 3: Implement `matchRecipe` and `canAddToPot`**

`matchRecipe` compares the sorted `potIngredientIds` against each recipe's sorted `ingredientIds` for an exact multiset match, returning the first match or `null`. `canAddToPot` returns `stage === 'fresh' || stage === 'wilted'`.

- [ ] **Step 4: Run test to verify it passes**

Run: `npx vitest run tests/systems/potMatcher.test.ts`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add src/systems/potMatcher.ts tests/systems/potMatcher.test.ts
git commit -m "feat: add pot recipe matcher with rotten-item guard"
```

---

### Task 8: Score tracker

**Files:**
- Create: `src/systems/scoreTracker.ts`
- Test: `tests/systems/scoreTracker.test.ts`

**Interfaces:**
- Consumes: `RecipeWithDerived` from Task 3.
- Produces: `export const CO2_PER_KM = 0.17`, `export interface ScoreSummary { points: number; co2SavedKg: number; wasteCount: number; drivingKmEquivalent: number }`, `export class ScoreTracker { addRecipe(recipe: RecipeWithDerived): void; addWaste(): void; getSummary(): ScoreSummary }`.

- [ ] **Step 1: Write the failing test**

```ts
import { ingredients } from '../../src/data/ingredients';
import { recipes, buildRecipeCatalog } from '../../src/data/recipes';
import { ScoreTracker, CO2_PER_KM } from '../../src/systems/scoreTracker';

const catalog = buildRecipeCatalog(ingredients, recipes);

test('accumulates points, co2 saved, and waste across events', () => {
  const tracker = new ScoreTracker();
  const r1 = catalog.find(r => r.id === 'root_veg_soup')!;
  const r2 = catalog.find(r => r.id === 'garden_salad')!;
  tracker.addRecipe(r1);
  tracker.addRecipe(r2);
  tracker.addWaste();
  const summary = tracker.getSummary();
  expect(summary.points).toBe(r1.points + r2.points);
  expect(summary.co2SavedKg).toBeCloseTo(r1.co2Saved + r2.co2Saved, 5);
  expect(summary.wasteCount).toBe(1);
  expect(summary.drivingKmEquivalent).toBeCloseTo(summary.co2SavedKg / CO2_PER_KM, 5);
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run tests/systems/scoreTracker.test.ts`
Expected: FAIL — module not found.

- [ ] **Step 3: Implement `ScoreTracker`**

Track running totals for `points`, `co2SavedKg`, `wasteCount` as instance fields; `getSummary()` returns them plus `drivingKmEquivalent = co2SavedKg / CO2_PER_KM`.

- [ ] **Step 4: Run test to verify it passes**

Run: `npx vitest run tests/systems/scoreTracker.test.ts`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add src/systems/scoreTracker.ts tests/systems/scoreTracker.test.ts
git commit -m "feat: add score tracker with CO2 driving-equivalent stat"
```

---

### Task 9: Boot scene and asset loading

**Files:**
- Create: `src/scenes/BootScene.ts`
- Create: `public/assets/sprites/` (one 16×16 image per ingredient's `spriteKey`, plus `fridge-cell.png`, `conveyor-belt.png`, `pot.png` background art)
- Modify: `src/main.ts:` register `BootScene` in the `scene` array

**Interfaces:**
- Consumes: `ingredients` from Task 2 (for the list of `spriteKey`s to preload).

- [ ] **Step 1: Source and place the sprites**

Download Karsiori's, FrozenTaurus's, and Alex Kovacs's packs; pick one 16×16 sprite per ingredient id in `src/data/ingredients.ts`; save each as `public/assets/sprites/<spriteKey>.png`. Exclude any meat/dairy sprites included in a pack.

- [ ] **Step 2: Implement `BootScene`**

In `preload()`, loop over `ingredients` and call `this.load.image(ing.spriteKey, `assets/sprites/${ing.spriteKey}.png`)`; also load the three background images. In `create()`, start the next scene (placeholder `MenuScene` key until Task 16 creates it — use a temporary direct transition to a stub scene if `MenuScene` doesn't exist yet).

- [ ] **Step 3: Manual verification**

Run: `npm run dev`
Expected: browser console shows no 404s; Phaser's load-complete event fires (check via `this.load.on('complete', ...)` log).

- [ ] **Step 4: Commit**

```bash
git add src/scenes/BootScene.ts public/assets/sprites src/main.ts
git commit -m "feat: add boot scene and ingredient sprite loading"
```

---

### Task 10: Single-player conveyor and fridge grid rendering

**Files:**
- Create: `src/scenes/GameScene.ts`
- Modify: `src/main.ts`

**Interfaces:**
- Consumes: `FridgeGrid`, `CELL_COUNT` (Task 5), `generateDeliveryPlan` (Task 6), `ingredients` (Task 2).
- Produces: `GameScene` constructed with `{ playerId: string, seed: number }` (later tasks extend this config object — do not replace it).

- [ ] **Step 1: Render the grid and start deliveries**

In `GameScene.create()`, draw a 5×4 grid of cell sprites (`CELL_COUNT` from Task 5) using `fridge-cell.png`. Start a 180000 ms countdown (`this.time.delayedCall`). Call `generateDeliveryPlan(this.seed, 30, ingredients.map(i => i.id))` and schedule each entry as a sprite that appears on the conveyor belt at its `offsetMs` via `this.time.delayedCall`.

- [ ] **Step 2: Implement drag-and-drop into the grid**

Make each conveyor sprite draggable (`this.input.setDraggable`). On `drop` over a grid cell, call `fridgeGrid.placeItem(cellIndex, ingredientId, this.time.now)`; if it returns `false`, tween the sprite back to the conveyor position; if `true`, destroy the conveyor sprite and spawn a fridge-cell sprite in its place.

- [ ] **Step 3: Render spoil state**

On every `update(time)`, for each occupied cell call `fridgeGrid.getStage(cellIndex, time)` and set that cell's sprite tint: `fresh` → no tint, `wilted` → yellow tint, `rotten` → dark/brown tint.

- [ ] **Step 4: Manual verification**

Run: `npm run dev`
Expected: drag three different conveyor items into the grid; after 45 real seconds the first one visibly tints yellow, after 90 seconds it tints dark.

- [ ] **Step 5: Commit**

```bash
git add src/scenes/GameScene.ts src/main.ts
git commit -m "feat: render conveyor deliveries and spoil-aware fridge grid"
```

---

### Task 11: Pot UI and recipe discovery

**Files:**
- Modify: `src/scenes/GameScene.ts`

**Interfaces:**
- Consumes: `matchRecipe`, `canAddToPot` (Task 7), `ScoreTracker` (Task 8), `buildRecipeCatalog` (Task 3).

- [ ] **Step 1: Implement the pot drop zone**

Add a pot sprite as a drop target accepting 2–3 dragged fridge items. On each drop, check `canAddToPot(fridgeGrid.getStage(cellIndex, this.time.now))`; if `false`, tween the item back and flash the pot red; if `true`, remove it from the grid (`fridgeGrid.removeItem`) and add its `ingredientId` to an in-scene `potContents: string[]`.

- [ ] **Step 2: Resolve the pot on a "cook" action**

On a cook button press (or automatically once `potContents.length >= 2`), call `matchRecipe(potContents, recipeCatalog)`. On a match: call `scoreTracker.addRecipe(recipe)`, show a toast with `recipe.name` and `recipe.tip`, clear `potContents`. On no match: show a "no recipe" toast and clear `potContents`.

- [ ] **Step 3: Manual verification**

Run: `npm run dev`
Expected: deliver and drag carrot, onion, and potato into the pot, cook, and see a "Root Veg Soup" toast plus an increased score.

- [ ] **Step 4: Commit**

```bash
git add src/scenes/GameScene.ts
git commit -m "feat: add pot UI and recipe discovery feedback"
```

---

### Task 12: HUD, match timer, and waste tracking

**Files:**
- Modify: `src/scenes/GameScene.ts`

**Interfaces:**
- Consumes: `ScoreTracker.getSummary()` (Task 8), `FridgeGrid.collectNewlyRotten()` (Task 5).

- [ ] **Step 1: Render the HUD**

Add text objects for points, CO₂ saved (kg), waste count, and the countdown timer (mm:ss), updated from `scoreTracker.getSummary()` each frame.

- [ ] **Step 2: Wire waste collection**

On every `update(time)`, call `fridgeGrid.collectNewlyRotten(time)` and call `scoreTracker.addWaste()` once per returned cell index.

- [ ] **Step 3: Transition to EndScene at timeout**

When the 180000 ms countdown reaches zero, call `this.scene.start('EndScene', { summary: scoreTracker.getSummary() })` (`EndScene` is created in Task 13).

- [ ] **Step 4: Manual verification**

Run: `npm run dev`
Expected: deliver one item and leave it untouched for 90+ seconds; waste count increases by exactly 1, not continuously.

- [ ] **Step 5: Commit**

```bash
git add src/scenes/GameScene.ts
git commit -m "feat: add HUD, match timer, and waste tracking"
```

---

### Task 13: End scene

**Files:**
- Create: `src/scenes/EndScene.ts`
- Modify: `src/main.ts`

**Interfaces:**
- Consumes: `ScoreSummary` shape from Task 8, via `scene.start('EndScene', { summary })`.

- [ ] **Step 1: Implement `EndScene`**

Read `data.summary` in `init(data)`. Render final points, `co2SavedKg`, and the line `"You saved {co2SavedKg} kg of CO₂ — about the same as {drivingKmEquivalent} km of driving."` Render a credits block listing Karsiori's, FrozenTaurus's, and Alex Kovacs's pack names and links (from `STRATEGY.md`'s Sources section).

- [ ] **Step 2: Manual verification**

Run: `npm run dev`
Expected: finish a match; EndScene's numbers match the HUD's last-displayed values exactly.

- [ ] **Step 3: Commit**

```bash
git add src/scenes/EndScene.ts src/main.ts
git commit -m "feat: add end scene with CO2 comparison and asset credits"
```

---

### Task 14: Local 2-player mode

**Files:**
- Create: `src/scenes/LocalMatchScene.ts`
- Modify: `src/scenes/GameScene.ts` (extend its config to `{ playerId: string, seed: number, viewportX: number, viewportWidth: number, controlScheme: 'wasd' | 'arrows' }`, and confine rendering/input to the given viewport slice)
- Modify: `src/main.ts`

**Interfaces:**
- Consumes: `FridgeGrid` (Task 5), `ScoreTracker` (Task 8), `generateDeliveryPlan` (Task 6).
- Produces: `LocalMatchScene` exposes a method both `GameScene` instances call on a recipe match: `triggerAttack(targetPlayerId: 'p1' | 'p2'): void`.

- [ ] **Step 1: Split the viewport and share one seed**

In `LocalMatchScene.create()`, pick one shared `seed`, then launch two `GameScene` instances (`this.scene.launch`) with `playerId: 'p1'`/`'p2'`, the same `seed`, and `viewportX`/`viewportWidth` set to the left and right halves of the canvas respectively, and distinct `controlScheme`s.

- [ ] **Step 2: Wire the attack trigger**

Add `LocalMatchScene.triggerAttack(targetPlayerId)`: look up the target `GameScene` instance and call three `fridgeGrid.placeItem(randomEmptyCell, randomIngredientId, time.now)` calls spaced 2000 ms apart via `this.time.delayedCall`. In each `GameScene`'s recipe-match handler (Task 11), after a successful `matchRecipe`, call `this.scene.get('LocalMatchScene').triggerAttack(otherPlayerId)`.

- [ ] **Step 3: Manual verification**

Run: `npm run dev`, select local 2-player.
Expected: cooking a recipe as player 1 causes 3 new items to land in player 2's grid over about 2 seconds.

- [ ] **Step 4: Commit**

```bash
git add src/scenes/LocalMatchScene.ts src/scenes/GameScene.ts src/main.ts
git commit -m "feat: add local 2-player mode with shared seed and attack trigger"
```

---

### Task 15: Supabase Realtime client

**Files:**
- Create: `src/net/realtimeClient.ts`
- Test: `tests/net/realtimeClient.test.ts`
- Create: `.env.example`
- Modify: `.gitignore` (add `.env`)

**Interfaces:**
- Produces: `export interface RealtimeChannelLike { on(event: string, filter: any, cb: (payload: any) => void): any; send(args: any): Promise<any>; subscribe(cb?: (status: string) => void): any }` (the subset of the Supabase channel API this class uses, so tests can mock it), `export class RealtimeClient { constructor(channel: RealtimeChannelLike); joinRoom(code: string): Promise<void>; broadcastMatchStart(seed: number): void; broadcastScoreUpdate(summary: ScoreSummary): void; broadcastAttack(): void; onMatchStart(cb: (seed: number) => void): void; onScoreUpdate(cb: (summary: ScoreSummary) => void): void; onAttack(cb: () => void): void }`.

- [ ] **Step 1: Write the failing test**

```ts
import { RealtimeClient } from '../../src/net/realtimeClient';

function makeMockChannel() {
  const handlers: Record<string, (payload: any) => void> = {};
  return {
    on: vi.fn((_event, filter, cb) => { handlers[filter.event] = cb; return this; }),
    send: vi.fn(() => Promise.resolve('ok')),
    subscribe: vi.fn((cb) => { cb?.('SUBSCRIBED'); return this; }),
    __emit: (event: string, payload: any) => handlers[event]?.(payload),
  };
}

test('broadcastAttack sends an attack event with no ingredient payload', async () => {
  const channel = makeMockChannel();
  const client = new RealtimeClient(channel as any);
  await client.joinRoom('ABC123');
  client.broadcastAttack();
  expect(channel.send).toHaveBeenCalledWith(expect.objectContaining({ type: 'broadcast', event: 'attack' }));
});

test('onAttack fires when the channel receives an attack event', async () => {
  const channel = makeMockChannel();
  const client = new RealtimeClient(channel as any);
  await client.joinRoom('ABC123');
  const spy = vi.fn();
  client.onAttack(spy);
  channel.__emit('attack', {});
  expect(spy).toHaveBeenCalled();
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run tests/net/realtimeClient.test.ts`
Expected: FAIL — module not found.

- [ ] **Step 3: Implement `RealtimeClient`**

Wrap the injected `channel`: `joinRoom` calls `channel.subscribe()` and resolves once the `'SUBSCRIBED'` status callback fires. `broadcastMatchStart`/`broadcastScoreUpdate`/`broadcastAttack` each call `channel.send({ type: 'broadcast', event: '<name>', payload: ... })` with `attack`'s payload empty (per the spec's "no ingredient data crosses the wire" rule). `onMatchStart`/`onScoreUpdate`/`onAttack` each call `channel.on('broadcast', { event: '<name>' }, (msg) => cb(msg.payload))`.

- [ ] **Step 4: Run test to verify it passes**

Run: `npx vitest run tests/net/realtimeClient.test.ts`
Expected: PASS

- [ ] **Step 5: Create the Supabase project (prerequisite, not code)**

Create a project in the Supabase dashboard; copy its URL and anon key into a local `.env` (never committed) as `VITE_SUPABASE_URL` and `VITE_SUPABASE_ANON_KEY`. Add both as empty placeholders to `.env.example`. Confirm `.env` is covered by `.gitignore`.

- [ ] **Step 6: Commit**

```bash
git add src/net/realtimeClient.ts tests/net/realtimeClient.test.ts .env.example .gitignore
git commit -m "feat: add Supabase Realtime broadcast client"
```

---

### Task 16: Menu scene and room-code flow

**Files:**
- Create: `src/scenes/MenuScene.ts`
- Modify: `src/main.ts` (register `MenuScene` as the scene `BootScene` starts into, replacing Task 9's temporary stub transition)

**Interfaces:**
- Consumes: `RealtimeClient` (Task 15), `LocalMatchScene` (Task 14), `GameScene` (Task 10).

- [ ] **Step 1: Render mode-select UI**

Two buttons: "Local 2-Player" (starts `LocalMatchScene`) and "Online Race" (reveals a room-code text input plus "Create Room" and "Join Room" buttons).

- [ ] **Step 2: Implement create-room flow**

On "Create Room": generate a short alphanumeric code, construct a `supabase.channel(code)`-backed `RealtimeClient`, call `joinRoom(code)`, then once a second peer is detected (Supabase presence or a simple "peer joined" broadcast), call `broadcastMatchStart(seed)` with a freshly-generated seed and start `GameScene` with `{ seed, playerId: 'p1', client }` — extend `GameScene`'s config type (first defined in Task 10, extended in Task 14) with an optional `client?: RealtimeClient` field; its presence is how Task 17's online-sync code distinguishes an online match from local play.

- [ ] **Step 3: Implement join-room flow with a timeout fallback**

On "Join Room": construct the `RealtimeClient` for the entered code, call `joinRoom(code)`, then `onMatchStart(seed => this.scene.start('GameScene', { seed, playerId: 'p2', client }))` (same `client` field as Step 2). Start a 10000 ms timer alongside; if `onMatchStart` hasn't fired when it elapses, show "Couldn't connect — reconnect or play local 2-player instead" and return to the mode-select buttons instead of hanging.

- [ ] **Step 4: Manual verification**

Run: `npm run dev` in two browser tabs.
Expected: create a room in tab A, join with its code in tab B; both reach `GameScene` with the same seed logged to console. Then block tab B's network (devtools offline) before joining and confirm the 10-second timeout message appears.

- [ ] **Step 5: Commit**

```bash
git add src/scenes/MenuScene.ts src/main.ts
git commit -m "feat: add menu scene with room-code online matchmaking and timeout fallback"
```

---

### Task 17: Online score and attack sync

**Files:**
- Modify: `src/scenes/GameScene.ts`

**Interfaces:**
- Consumes: `RealtimeClient.broadcastScoreUpdate/onScoreUpdate/broadcastAttack/onAttack` (Task 15).

- [ ] **Step 1: Broadcast on recipe match and heartbeat**

When `playerId` indicates an online match (a `RealtimeClient` instance was passed into the scene's config), call `client.broadcastScoreUpdate(scoreTracker.getSummary())` and `client.broadcastAttack()` immediately after every successful `matchRecipe` (Task 11). Also call `broadcastScoreUpdate` every 2000 ms as a heartbeat via `this.time.addEvent`.

- [ ] **Step 2: Subscribe to opponent updates**

In `create()`, call `client.onScoreUpdate(summary => this.opponentSummary = summary)` and update an "opponent score" HUD line from it each frame. Call `client.onAttack(() => this.applyForcedDeliveries())`, reusing the same three-items-over-2-seconds routine Task 14 built for local 2-player (extract it into a shared helper both `LocalMatchScene` and this online path call, rather than duplicating it).

- [ ] **Step 3: Manual verification**

Run: `npm run dev` in two tabs, start an online match.
Expected: cooking a recipe in tab A shows 3 forced deliveries land in tab B's grid within ~2 seconds, and tab A's opponent-score HUD line updates with tab B's totals.

- [ ] **Step 4: Commit**

```bash
git add src/scenes/GameScene.ts
git commit -m "feat: sync score and attack events over Supabase Realtime"
```

---

### Task 18: Deploy config and licensing pass

**Files:**
- Create: `vercel.json`
- Modify: `src/scenes/EndScene.ts` (finalize credit wording once Alex Kovacs's license is confirmed)

- [ ] **Step 1: Confirm the Alex Kovacs pack's license**

Re-check `https://alexkovacsart.itch.io/free-pixel-art-foods/purchase` for its exact terms (the spec flagged this as unconfirmed). Update `EndScene`'s credit line wording to match whatever it actually requires (attribution text, link, or none).

- [ ] **Step 2: Configure the static build for hosting**

Add a minimal `vercel.json` (`{ "buildCommand": "npm run build", "outputDirectory": "dist" }`) for a Vercel deploy; alternatively note in this task's commit message that itch.io packaging just zips `dist/` after `npm run build`.

- [ ] **Step 3: Manual verification**

Run: `npm run build && npm run preview`
Expected: the built bundle serves and plays a full match standalone, with no dev-only console errors.

- [ ] **Step 4: Commit**

```bash
git add vercel.json src/scenes/EndScene.ts
git commit -m "chore: add static deploy config and finalize asset credits"
```
