# AGENTS.md

Pac-Man clone (original level-1 geometry) in vanilla JS / HTML / CSS. **No build step, no `package.json`, no dependencies, no tests, no linter, no formatter config.** Everything below is verified against the current tree.

## Run and verify

- Open `src/index.html` in a browser. `file://` works — no modules, no `fetch`. There is nothing to install.
- There is no `npm install` / `npm run dev` / `npm test` / `npm run build`. Do not introduce a toolchain unless explicitly asked.
- Verification is manual: load the page, use the arrow keys, check the console for errors, and confirm the HUD (`SCORE`, `VIDAS`) and the win/lose overlay.
- Optional static server (not part of the repo): `npx serve src`.
- The app root is `src/`, not the repo root. There is no `index.html` at the repo root.

## Architecture

Four classic `<script>` files, loaded in this exact order from `src/index.html`:

```
src/js/maze.js  ->  src/js/game.js  ->  src/js/render.js  ->  src/js/main.js
```

There is no module system and no bundler. The files talk through `window` globals:

| File | Exports | Responsibility |
| --- | --- | --- |
| `maze.js` | `window.MAZE`, `window.TUNNEL_ROW`, `window.PACMAN_START`, `window.GHOST_STARTS` | level geometry only |
| `game.js` | `window.createGame`, `window.update`, `window.DIRS` | state and rules |
| `render.js` | `window.draw` | canvas drawing |
| `main.js` | — (entrypoint) | `requestAnimationFrame` loop, keyboard, overlay |

Rules that follow from this:

- To add a file: add a `<script>` tag in `src/index.html` in **dependency order** (not alphabetical) and expose its API with `window.X = X` — that is the existing convention.
- `render.js` uses `DIRS` from `game.js`; `game.js` reads `MAZE`/`TUNNEL_ROW`/`PACMAN_START`/`GHOST_STARTS` from `maze.js`. Load order is load-bearing.

## Maze and grid model (easy to break)

- Grid is **28 cols x 31 rows**. Coordinates are cells `(x, y)`, origin top-left, `x` in `[0,27]`, `y` in `[0,30]`. Symmetric about the vertical axis between cols 13 and 14.
- Tile values: `1` wall, `2` dot, `0` empty/walkable, `3` pen door.
- Edit `MAZE_STR` in `maze.js` (31 strings of 28 chars: `#` wall, `.` dot, `-` door, space = empty) — the numeric `MAZE` is parsed from it. Leading/trailing spaces on the tunnel row (row 14) are meaningful.
- `MAZE` is pristine and never mutated. `createGame()` copies it into `game.grid`; eating a dot sets `game.grid[y][x] = 0`. Render reads `game.grid`, not `MAZE`, so eaten dots disappear.
- The pen door (tile `3`) blocks Pac-Man but **not** ghosts — see `isWall( grid, x, y, actor )` in `game.js`.
- Canvas must stay `cols * TILE` x `rows * TILE` with `TILE = 20` → 560x620. If the maze dimensions change, update the `width`/`height` on `<canvas id="game">` in `src/index.html` **and** `#game-wrap` width/height in `src/css/style.css`.

## Movement model

- Positions are fractional cells; speed is cells/frame (Pac-Man 1/8, ghosts 1/10), chosen so actors land on a cell center every 8 / 10 frames.
- Turns are buffered: `keydown` sets `pacman.nextDir`; the turn is applied only when Pac-Man is aligned on a cell center **and** the target cell is not a wall.
- Ghost decisions happen only at cell centers: `hunter` minimizes Manhattan distance to Pac-Man; `random` picks uniformly among legal directions and never reverses unless it is a dead end.
- Collision tolerance is 0.5 cell.
- `showOverlay()` rewrites `#overlay.innerHTML`, so the `#action-btn` const in `main.js` becomes stale after the first win/lose screen (harmless, but do not rely on it).

## Style conventions (differ from prettier/eslint defaults)

- 2-space indent, single quotes, semicolons, and **spaces inside parentheses**: `if ( x ) {`, `foo( arg )`. Running prettier/eslint with default settings would reformat the entire codebase — do not.
- Identifiers are English; **comments and all user-facing strings are Spanish** (`SCORE`, `VIDAS`, `GANASTE`, `PERDISTE`, "Flechas para moverte"). Keep new comments and UI text in Spanish.
- Keep the arcade look in `render.js`: walls drawn as rounded strokes connecting the centers of adjacent wall cells, fixed color constants, single `TILE` constant.

## Spec-driven workflow (the point of this repo)

Vendored skills live in `.agents/skills/` (upstream `klerith/fernando-skills`, pinned by `skills-lock.json`; both have `disable-model-invocation: true`, so they only run when invoked explicitly as `/spec` and `/spec-impl` — do not auto-trigger them).

- `/spec` → writes `specs/NN-slug.md`. States: `Draft` → `In review` → `Approved` → `Implemented` → `Obsolete`. The `specs/` folder does not exist yet; the first `/spec` run creates it and seeds `specs/.spec-config.yml`.
- `/spec-impl NN-slug` → requires the spec state to **mean "Approved"** (the human flips that word; the agent must never approve on the user's behalf), creates/switches to branch `spec-NN-slug`, then implements one plan step at a time and pauses for review.
- `specs/.spec-config.yml` `AutoCreateBranch: true` is the default (auto-create the branch); set it to `false` to be asked first.
- Never commit automatically during `/spec-impl` — commit only when the user asks.
- Reply in the same language as the user's request. Existing repo docs (README, code comments) are in Spanish, so specs should follow suit unless the user says otherwise.
