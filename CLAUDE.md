# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Vanilla Tetris. Plain HTML5 Canvas + CSS + JS. No dependencies, no build step, no `package.json`, no test suite.

## Commands

No build/lint/test tooling exists. To run the game, serve the directory and open in browser:

```bash
npx serve .
# or
python3 -m http.server 8000
```

Then open `index.html` / `http://localhost:8000`. Opening `index.html` directly (`file://`) also works.

## Architecture

Three files, all logic lives in `game.js` (~300 lines, single script, no modules):

- **`index.html`** — DOM shell: `<canvas id="board">` (300×600, 10×20 grid at `BLOCK=30`), `<canvas id="next-canvas">` for the next-piece preview, HUD spans (`#score`, `#lines`, `#level`), and `#overlay` for pause/game-over states.
- **`style.css`** — dark/retro arcade visual theme.
- **`game.js`** — state, game loop, rendering, input, all in module-level globals (`board`, `current`, `next`, `score`, `lines`, `level`, `paused`, `gameOver`, `dropInterval`, etc.) mutated directly by functions.

Key mechanics in `game.js`:

- **Board model**: `ROWS × COLS` matrix; each cell is `0` (empty) or a color index `1–7` identifying which piece locked there.
- **Pieces** (`PIECES`): defined as small square matrices. Rotation (`rotateCW`) is a transpose + row-reverse producing a new matrix — pieces are not stored with rotation state, just re-rotated from the current shape.
- **Collision** (`collide`): checks board bounds and existing locked cells for a given shape/offset.
- **Wall kicks** (`tryRotate`): after rotating, tries offsets `[0, -1, 1, -2, 2]` until one doesn't collide, else rotation is discarded.
- **Ghost piece** (`ghostY`): projects `current` straight down until collision, drawn at `globalAlpha = 0.2`.
- **Lock → clear → spawn pipeline**: `lockPiece()` calls `merge()` (bakes piece into `board`) → `clearLines()` (removes full rows bottom-up, unshifts empty rows, updates score/level/`dropInterval`) → `spawn()` (promotes `next` to `current`, generates new `next`, checks spawn collision → `endGame()` if it collides).
- **Game loop** (`loop`, driven by `requestAnimationFrame`): accumulates `dt` in `dropAccum`; once it exceeds `dropInterval`, drops the piece one row or locks it; then redraws every frame.
- **Speed curve**: `dropInterval = max(100, 1000 - (level - 1) * 90)`; level increases every 10 cleared lines.
- **Scoring**: `LINE_SCORES = [0, 100, 300, 500, 800]` (by lines-cleared-at-once) `× level`; hard drop adds 2 pts/row traveled, soft drop adds 1 pt/row.
- **Input**: single `keydown` listener switches on `e.code` (arrows, `KeyX` for rotate, `Space` for hard drop, `KeyP` for pause), guarded by `paused`/`gameOver` flags.

## Tuning knobs

All at the top of `game.js`: `COLS`, `ROWS`, `BLOCK` (px per cell), `COLORS` (7-color palette), `LINE_SCORES`, initial `dropInterval`. If `COLS`/`ROWS`/`BLOCK` change, also update the `#board` canvas `width`/`height` in `index.html` to match (`COLS × BLOCK`, `ROWS × BLOCK`).
