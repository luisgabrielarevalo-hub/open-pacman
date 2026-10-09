# AGENTS.md — open-pacman

Vanilla JS + HTML + CSS Pac-Man clone. No bundler, no npm, no tests, no lint.

## Run

- No build step. Open `src/index.html` directly, or serve: `python3 -m http.server` from `src/`.
- All game code lives in `src/js/` + `src/css/style.css`. No other entrypoints.

## Architecture (load order matters)

`src/index.html` loads scripts in fixed order — keep it: `maze.js` → `game.js` → `render.js` → `main.js`. They share globals via `window.*`, not modules.

- `js/maze.js` — pristine 28×31 level (`MAZE_STR` → `MAZE`), `TUNNEL_ROW=14`, `PACMAN_START={13,23}`, `GHOST_STARTS`. Tile codes: `#`=1 wall, `.`=2 dot, ` `=0 empty, `-`=3 pen door. Never mutate `MAZE`; each game copies it to `game.grid`.
- `js/game.js` — state + rules: `createGame()`, `update(game)`. Movement uses fractional cell coords with `aligned()` epsilon `1e-3`; `PACMAN_SPEED=0.125`, `GHOST_SPEED=0.1`. `isWall`/`canMove` are actor-specific: pacman blocked by wall(1)+door(3), ghost only by wall(1). Tunnel wraps only on row 14. Ghosts: `hunter` chases via Manhattan distance, `random` picks randomly; both avoid 180° turns except in dead ends. Collision radius `<0.5`, 3 lives, states `start|playing|won|lost`.
- `js/render.js` — canvas drawing, reads `game.grid` (not `MAZE`) so eaten dots disappear. `TILE=20`, canvas 560×620.
- `js/main.js` — loop, arrow-key input (`nextDir` applied at cell alignment), overlay start/win/lose.

## Conventions

- Spec-driven learning project: new features start as a spec via the `spec` skill, implemented via `spec-impl`. Template: `.agents/skills/spec/template.md` (blockquote header with Status/Depends/Date/Objective, mandatory In/Out scope, boolean acceptance criteria).
- UI text and comments are in Spanish; keep it.
- Verify by loading the page and checking the console — there is no test suite.
