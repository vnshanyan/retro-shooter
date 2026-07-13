# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A collection of **self-contained, single-file browser games**. Each game is one `.html`
file with all CSS in an inline `<style>` and all logic in an inline `<script>` — no build
step, no bundler, no external dependencies, no network fetches, no asset files. A game
must remain runnable by opening the file directly in a browser (`file://`).

- `shooter.html` — 2D top-down retro arcade shooter (the main, actively-developed game)
- `index.html` — tic-tac-toe (2-player + unbeatable minimax AI)

These games are independent and share no code. Keep them that way unless asked otherwise.

## Running & verifying

There are no tests, linter, or build. To verify a change:

- **Run it**: open the file in a browser. On Windows: `start "" shooter.html`.
- **Syntax-check the script without a browser** (catches parse errors fast):
  ```
  node -e "new Function(require('fs').readFileSync('shooter.html','utf8').match(/<script>([\s\S]*)<\/script>/)[1]); console.log('OK')"
  ```
  This only validates syntax — it can't exercise Canvas/DOM behavior, so still open the
  game in a browser to confirm gameplay actually works.

## Git workflow (standing instruction)

Commit and push after each meaningful unit of work — no need to ask each time.
Remote `origin` → https://github.com/vnshanyan/retro-shooter (public), branch `main`.
Use concise, descriptive commit messages.

## Shooter architecture (`shooter.html`)

The whole game lives in one IIFE. Key structure to understand before editing:

- **Fixed low-res canvas, CSS-upscaled.** The canvas is `320×180` internally and stretched
  to `960×540` via CSS with `image-rendering: pixelated` for the chunky retro look. All
  game logic works in the 320×180 coordinate space. Mouse events must be converted from
  client pixels to canvas space via `toCanvasCoords()` (uses `getBoundingClientRect`
  scale factor) — never use raw `clientX/clientY` for gameplay.

- **State machine drives everything.** `state.mode` is one of
  `menu | playing | levelComplete | gameOver | victory`. `setMode()` is the single funnel
  for transitions and shows/hides the matching DOM overlay via `showScreen()`. The
  `requestAnimationFrame` loop runs continuously but only updates the world when
  `mode === 'playing'`; other modes render a dimmed frozen frame behind an HTML overlay.
  **UI screens are DOM `<div>` overlays, not canvas-drawn** — only gameplay/sprites are on
  the canvas. The HUD overlay uses `pointer-events: none` so it never blocks canvas
  mouse aim/click.

- **Entities are plain objects from factory functions** (`createPlayer`, `createBullet`,
  `createEnemy`) held in `state.player`, `state.bullets`, `state.enemies`. No classes.
  Dead entities are removed by `filter()` at the end of each update tick.

- **Sprites are string-grids, not images.** Each sprite (`PLAYER_A`, `ENEMY_A`, etc.) is an
  array of equal-length strings where each char indexes a color in the `PALETTE` object
  (`.` = transparent). `drawSprite()` loops cells and `fillRect`s them. **All rows in a
  grid must stay the same width** (currently 8) or rendering misaligns. Aiming is done by
  rotating only the gun (via `ctx.translate`/`rotate` toward the mouse angle) and
  h-flipping the body — there are no pre-rendered directional frames. Animation is 2-frame
  walk cycles driven by per-entity `animTimer`/`animFrame`.

- **Tuning is data, not logic.** Difficulty lives in the `LEVELS` array (per-level enemy
  composition, spawn interval, speed multiplier) and `ENEMY_TYPES` (hp/speed/radius/score/
  color per type). Change these to retune waves without touching update code. Enemies
  spawn at a random angle just outside the screen and chase the player directly (no
  pathfinding). Collisions are circle-circle using squared distance (`circleHit()`).

- **Movement/animation are `dt`-scaled** (delta time from the loop, clamped to 50ms) so
  behavior is framerate-independent. Multiply per-second speeds by `dt`.
