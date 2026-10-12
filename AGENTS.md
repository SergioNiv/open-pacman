# AGENTS.md

Pac-Man clone in vanilla JS/HTML/CSS. No framework, no bundler. Learning project for spec-driven development (see `spec` skill in `.agents/skills/spec/`; specs belong in `specs/NN-slug.md` — folder does not exist yet).

## Running & verifying

- No package.json, no tests, no lint. There is nothing to "npm run".
- Play by opening `src/index.html` in a browser, or serve the `src/` dir statically (e.g. `python3 -m http.server -d src`).
- Verify changes in the browser; `src/js` is the only executable source of truth.

## Architecture (src/)

- Plain scripts, **no ES modules**. Load order in `index.html` matters: `maze.js` → `game.js` → `render.js` → `main.js`; they communicate via `window` globals (`MAZE`, `createGame`, `update`, `draw`).
- `maze.js` — level data only: `MAZE_STR` parsed to a number grid. Tiles: `1` wall, `2` dot, `3` ghost door, `0` empty.
- `game.js` — state + rules. `createGame()` copies pristine `MAZE` into `game.grid` (dots are eaten on the copy; `MAZE` is never mutated). `update(game)` is one tick; `game.state` is `start` | `playing` | `won` | `lost`.
- `render.js` — canvas drawing; reads `game.grid`, never `MAZE`.
- `main.js` — rAF loop, arrow-key input, win/lose overlay.

## Maze invariants (breaking these fails silently)

- `MAZE_STR` must be 31 strings × 28 chars, symmetric about the vertical axis (between cols 13–14).
- Row 14 (`TUNNEL_ROW`) is the tunnel: its ends are open (no `#`) and `wrapTunnel` wraps actors on that row.
- Starts are hardcoded: `PACMAN_START = {x:13, y:23}` (that cell is dot-less in `createGame`), `GHOST_STARTS` at (13,14), (14,14), (13,13), (14,13) — a 2×2 block inside the pen above the door, in order hunter / ambush / cooperator / shy. Keep them consistent with any maze edit.
- Canvas is 560×620 = 28×31 cells at `TILE=20`. If maze dimensions change, also update the `width`/`height` attrs in `index.html`.

## Movement model

- Positions are floats; actors only make decisions (turn, eat, choose dir) when `aligned()` within 1e-3 of a cell center. Pacman speed 0.125 (aligns every 8 frames), ghost speed 0.1.
- Pacman is blocked by walls (1) and doors (3); ghosts only by walls (1) — that is how ghosts cross the pen door.

## Conventions

- Repo docs, comments, and in-game UI strings are in Spanish; keep it that way.
