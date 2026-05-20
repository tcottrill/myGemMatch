# Bejeweled Clone — Design

**Date:** 2026-05-17
**Deliverable:** Single self-contained `index.html` that plays a Bejeweled-style match-3 game in any modern browser, on desktop or mobile, and installs as a PWA.

## Goals

- Classic Bejeweled-2-style match-3 with cascades and three power gems (fire, star, hyper).
- Endless mode only in v1 (Blitz/timed mode deferred).
- Reuse the gem visuals already present in `bejeweled.html` (the existing file in this repo).
- Ship as one HTML file + PWA sidecars (icons, manifest), playable from `file://` and installable from HTTPS.

## Non-goals

- No level progression, no campaign, no in-app purchases.
- No Blitz/timed mode (deferred).
- No multiplayer or leaderboards.
- No server, no build pipeline, no framework.

## Architecture

DOM + canvas hybrid:

- **DOM (HTML/CSS):** outer chrome — title bar, current score, best score, "New Game" button, mute toggle, menu overlay, and any dialogs (paused, confirm restart). Uses CSS for layout, transitions, and accessibility-friendly buttons.
- **Canvas:** the 8×8 game board, all gem rendering, swap/match/cascade animations, and ambient effects on power gems. DPR clamped to 2. Resize handler debounced (120 ms; 260 ms on `orientationchange`).

The board lives inside a flex-centered container; canvas backing-store is sized off the CSS box.

## Asset reuse

`bejeweled.html` contains `<symbol>` definitions for the seven colored gems (`tile_red`, `tile_orange`, `tile_yellow`, `tile_green`, `tile_blue`, `tile_purple`, `tile_white`) and for power-up gems (`tile_hyper`, plus overlay groups for fire/star effects). We will:

1. Extract those seven color symbols and the hyper/fire/star visuals as standalone SVG strings.
2. Embed them in a hidden `<svg id="gem-sprite">` in `index.html` (one combined sprite, with internal `id`s namespaced per the skill's sprite guidance — though in this case the source IDs don't collide so namespacing is a safety pass).
3. At boot, rasterize each gem variant to an `Image` (data URL of a standalone SVG document wrapping the symbol's geometry) for fast `ctx.drawImage` blits.
4. The original `<animate>` gradient tags don't tick once rasterized, so the "alive" feel of power gems is reproduced in canvas:
   - **Fire:** pulsing radial-gradient glow ring (sin-wave alpha).
   - **Star:** rotating spark cross overlay.
   - **Hyper:** hue-shifting outline drawn via `ctx` (HSL with time-based hue).

## Data model

```js
// board[r][c] = a Gem, or null while animating
type Gem = {
  color: 0..6,            // index into PALETTE
  special: null | 'fire' | 'star' | 'hyper',
  // transient render state (set during animations):
  dx, dy,                 // pixel offset from grid cell
  scale, alpha,           // for clear/spawn anims
  id,                     // stable id for tracking through swaps
};
```

Board is `Gem[8][8]`. Animations are a flat array of `{kind, gemId, t0, dur, ...}` objects driven by a single self-stopping `requestAnimationFrame` loop (per skill's animation-loop reference).

## Game rules

- **Init:** fill the board randomly; if a cell would complete a line of 3+ at placement, reroll its color. After fill, verify at least one legal swap exists; if not, shuffle and re-verify.
- **Input:** Pointer Events with `setPointerCapture`. Tap a gem to select; tap an orthogonally-adjacent gem to swap. A drag/swipe on a selected gem also swaps in the direction released.
- **Swap rule:** swap commits only if the resulting board has at least one match including one of the swapped cells. Otherwise the gems animate back to origin (with a soft error blip).
- **Power-gem spawn (on the committing swap):**
  - Match-4 in a single line → spawn a **fire gem** at the swap-origin cell.
  - Match-5+ in a single line → spawn a **star gem** at the swap-origin cell.
  - Two matches that share a cell (T or L) → spawn a **hyper cube** at the shared cell.
- **Power-gem activation (when matched as part of a future swap):**
  - **Fire:** removes its 3×3 neighborhood (gems in that area are matched too, contributing to cascade).
  - **Star:** removes its full row and column.
  - **Hyper:** when swapped with a regular gem, removes every gem of that color from the board.
- **Resolve loop:** after a committing swap → detect matches → award score → animate clears → apply gravity → spawn new gems falling from the top → repeat until no matches. Each cascade step multiplies score ×1.5.
- **Scoring:** 3 = 30, 4 = 60 (+ 30 for fire spawn), 5+ = 100 (+ 50 for star spawn). T/L spawns hyper (+ 100). Cascade multiplier ×1.5 compounds per chain.
- **Deadlock handling:** after every settle, scan for any legal swap. If none exists, display a brief "Shuffling…" toast and reshuffle the board (preserving score). Repeat until a legal board appears.

## State machine

```
LOADING → MENU
MENU → PLAY            ("New Game" or "Continue")
PLAY → PLAY            (gameplay loop; internal substates: IDLE, SWAP, RESOLVE)
PLAY → MENU            ("New Game" button — confirm if score > 0)
```

A `CONFIRM_RESTART` dialog protects an in-progress game. No win condition in v1; the game is endless.

## Polish

- **Hint system:** track time since last input. After 5 s idle in IDLE substate, pick any legal swap and pulse its two gems with a soft scale animation until the next input.
- **Audio:** synthesized via WebAudio, lazy-initialized on first user gesture. Sounds: click (short blip), swap-commit (rising pair), match (pleasant chord), big-match (4+ → brighter chord), power-gem-activate (sweep), cascade-rising-pitch (each chain bumps the pitch), error/illegal-swap (low buzz). Mute toggle persists.
- **Best score:** persisted to `localStorage` under `bejeweled.bestScore.v1`. Writes debounced 250 ms, flushed on `pagehide`. Mute under `bejeweled.mute.v1`. All access wrapped in try/catch.
- **PWA:** `site.webmanifest` with `start_url: "./"`, scope `"./"`, display `"standalone"`, theme color matching board background. Apple-touch-icons at 180/167/152. Maskable 192 + 512. Generated from a single `icon-master.svg` of a stylized jewel. `viewport-fit=cover`, `apple-mobile-web-app-capable`, suitable status-bar-style. Canvas has `touch-action: none`; body has `overscroll-behavior: none` and `-webkit-tap-highlight-color: transparent`.

## Errors and edge cases

- Pointer cancelled (e.g., browser overlay) → cancel current drag, revert any in-flight swap visuals.
- Resize mid-animation → recompute pixel positions from grid coords (animations are time-based, not pixel-based).
- localStorage unavailable (private mode) → degrade silently; best score becomes session-only.
- Rapid input during RESOLVE substate → ignored; queue is not built up.

## File layout

```
myBejeweled/
├── index.html                  # the entire game
├── bejeweled.html              # original source, left untouched
├── site.webmanifest
├── icon-master.svg
├── icon-192.png
├── icon-512.png
├── apple-touch-icon.png        # 180×180
├── apple-touch-icon-167x167.png
├── apple-touch-icon-152x152.png
├── favicon.ico
├── favicon-16x16.png
├── favicon-32x32.png
├── README.md
└── docs/superpowers/specs/2026-05-17-bejeweled-clone-design.md
```

## Out of scope (future)

- Blitz / timed mode with 60-second timer.
- Moves-limited mode.
- Coin/treasure/dirt obstacle tiles (the source has SVGs for these).
- Daily challenge or seeded boards.
