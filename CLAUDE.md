# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project status

This repo currently contains only the game design document (`STRATEGY.md`). No source code, package manifest, build tooling, or tests exist yet — this is a hackathon project in the planning stage. There are no build/lint/test commands to run until the codebase is scaffolded.

When scaffolding begins, update this file with the actual commands (`npm run dev`, `npm run build`, test runner, etc.) and remove this section.

## What this project is

**Fridge Rush** — a 3-minute, head-to-head browser game built for a 24-hour hackathon. See `STRATEGY.md` for the full design doc (art assets, scope, open questions, sources). Key points for implementation:

- **Core loop:** groceries arrive on a conveyor belt → drag into a fridge grid → items spoil over time (fresh → wilted → rotten, color-coded) → drag items into a pot to discover recipes → recipes score points, saved CO₂, and a real-world tip.
- **Head-to-head twist:** finishing a recipe sends the opponent a surprise bulk delivery that clogs their fridge.
- **Two match modes, in priority order:**
  1. Local 2-player on one screen/keyboard (simpler, no network dependency — build and keep this working as the demo fallback).
  2. Online race: each player runs their own fridge/kitchen locally with the same delivery sequence; only scores and "attack" events are broadcast between clients via Supabase Realtime (room code based, no custom server).
- **Build order matters:** the plan is to get a complete single-player game working first (fridge grid, spoiling, cooking, recipe data) before adding networking, so a Supabase/networking issue never leaves the demo with nothing to show.

## Planned architecture

- **Game engine:** Phaser 4 + TypeScript + Vite. Phaser 4 API is close to Phaser 3, so Phaser 3 patterns (scenes, drag-and-drop input, tweens for spoil-state transitions) mostly apply.
- **Realtime sync:** Supabase Realtime broadcast only (room code pairing + score/attack messages), not full state sync — deliberately avoided because full real-time world sync was judged infeasible for a solo 24-hour build.
- **Data-driven recipes/ingredients:** ingredients and their CO₂/water-footprint stats are meant to live in a single data file (~15 ingredients), sourced from Our World in Data (Poore & Nemecek emissions study) and the Water Footprint Network, so recipe matching and the CO₂-savings end screen both read from one source of truth rather than hardcoded values scattered across scenes.
- **Hosting:** static deploy target (Vercel or itch.io) — no backend beyond Supabase, so the build output should remain a static bundle.

## Asset constraints

Sprites are pulled from free itch.io packs (see `STRATEGY.md` for links/licenses), not hand-drawn. When adding art:
- Keep to a single sprite size (16×16) — do not mix sizes from different packs.
- Exclude meat/dairy sprites from the packs that include them (plant-forward theme).
- Track which packs require attribution; credits belong on the end screen.
