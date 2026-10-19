# Fridge Rush: Hackathon Game Plan

Oct 19, 2026 · @trish

## Art assets

Hand-drawn pixel art is not needed. Free asset packs are normal at hackathons. For one person in 24 hours, time spent drawing is time not spent on gameplay.

| Pack | Size | License | Notes |
| --- | --- | --- | --- |
| [Karsiori's Free Pixel Art Food Pack](https://karsiori.itch.io/free-pixel-art-food-pack) | Individual sprites | CC0 | Credit appreciated, not required |
| [FrozenTaurus's Food Assets](https://luizatorello.itch.io/food-assets) | 16×16, 45 sprites | Free, credit required | Best fit: fruit and vegetable sheets (eggplant, carrot, potato, leek, onion, broccoli, bell peppers) |
| [Alex Kovacs's 100 Free Pixel Art Foods](https://alexkovacsart.itch.io/free-pixel-art-foods/purchase) | 16×16, 100 items | Free (check the page for exact terms) | Food and food-related items |

- Several packs include meat and dairy sprites. Leave those out.
- Stick to one sprite size. Mixing 16×16 and 32×32 looks messy.
- Check the hackathon rules. Some events require everything to be made during the event, or ask you to list outside assets. Credits on the end screen usually cover it.

## Head-to-head matches

Build an online race, and keep local 2-player on one screen as the backup. Head-to-head is the riskiest part of the plan for one person in 24 hours.

| Option | Difficulty | How it works |
| --- | --- | --- |
| Online race (recommended) | Medium | Each player runs their own fridge and kitchen on their own laptop. Both get the same deliveries in the same order. Only scores and "attacks" are sent between players, via Supabase Realtime broadcast with a room code, so no server code is needed. |
| Local 2-player on one screen | Easy | Both players share one keyboard and screen. No internet needed, so it is safe if venue Wi-Fi is bad on demo day. |
| Full real-time sync | Very hard | Both players share one game world. Not realistic for one person in 24 hours. |

## Scope: Fridge Rush

Drop the farming part. The core game is a 3-minute head-to-head match built from the Zero-Waste Fridge and Leftover Alchemist ideas.

1. **Deliveries:** Groceries arrive on a conveyor belt. You drag them into your fridge grid.
2. **Spoiling:** Items slowly go bad, shown by a color shift from fresh to wilted to rotten. Anything that rots adds to your waste.
3. **Cooking:** Drag 2–3 items into the pot to discover a recipe. Each recipe earns points, saved CO₂, and a short real-world tip.
4. **Head-to-head twist:** Each recipe you finish sends your opponent a surprise bulk delivery that clogs their fridge.
5. **End screen:** Shows who won, plus "you saved X kg of CO₂, about the same as Y km of driving", using Our World in Data numbers.

**Real data sources:**
- Our World in Data: greenhouse-gas emissions per kg of food (Poore & Nemecek study)
- Water Footprint Network: water use per food
- UNEP Food Waste Index: waste statistics for "did you know" pop-ups

## Tech stack

| Layer | Choice | Why |
| --- | --- | --- |
| Game framework | [Phaser 4](https://phaser.io/download/phaser4) + TypeScript + Vite | Real 2D framework with pixel-art mode, scenes, animations and drag-and-drop. Phaser 4.0 shipped in April 2026. Code is written almost the same as Phaser 3, so most Phaser 3 tutorials still apply. |
| Matches | Supabase Realtime | Room codes and score/attack broadcast with no server code |
| Hosting | Vercel or itch.io | Free and quick to deploy a browser game |

## 24-hour build plan

**TODO:** The 6-phase plan was not included in the pasted notes. Add it here.

Hours 0–12 produce a complete single-player game. The online race, polish and deploy come after, so a networking problem never leaves you with nothing to demo.

Hours 0–2 setup covers loading the asset packs and a data file of about 15 ingredients with real CO₂ and water numbers.

## Open questions

- [ ] Do the rules allow pre-made assets, or code written before the event starts?
- [ ] Add the AI chef (suggests a recipe from what's in your fridge)? If yes, treat it as a bonus for hours 17–21, not core.
- [ ] Build a working starter before the event (fridge grid, spoil timers, cooking, ingredient data file), or a step-by-step build guide if code must be written during the event?

## Sources

- [Phaser 4 releases (phaser.io)](https://phaser.io/download/phaser4)
- [Phaser v4.0.0 announcement (GitHub)](https://github.com/phaserjs/phaser/discussions/7274)
- [Free Pixel Art Food Pack, karsiori](https://karsiori.itch.io/free-pixel-art-food-pack)
- [FrozenTaurus's Food Assets](https://luizatorello.itch.io/food-assets)
- [100 Free Pixel Art Foods, Alex Kovacs](https://alexkovacsart.itch.io/free-pixel-art-foods/purchase)
