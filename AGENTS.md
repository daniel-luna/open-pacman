# AGENTS.md

Vanilla JS/HTML/CSS Pac-Man clone. The repo exists to practice spec-driven development (README). No `package.json`, no build, no lint, no tests, no CI — do not invent commands or add tooling.

## Run / verify

- Open `src/index.html` in a browser (plain `<script>` tags, works over `file://`). No dev server needed.
- After a change, reload and check the browser console for errors. That is the whole verification loop.

## Project rules

- Keep it vanilla: no framework, no bundler, no npm dependencies (stack fixed by README).
- Spanish everywhere: README, code comments, UI strings. Match it; the `/spec` skill also replies in the language of the prompt.
- Never commit unless the user explicitly asks.

## Spec workflow (the point of this repo)

- Features start with `/spec <one-sentence description>` → writes `specs/NN-slug.md`. Never write code during `/spec`. The file lands in `Draft`; flipping it to `Approved` is the human's job.
- Implement with `/spec-impl NN-slug`: it refuses anything not meaning "Approved", creates branch `spec-NN-slug` (auto by default; controlled by `specs/.spec-config.yml`), then implements the plan one step at a time, pausing after each for diff review. Ambiguity → stop and ask, don't improvise.
- Skills are local copies in `.agents/skills/`, pinned by hash in `skills-lock.json`. Don't hand-edit them; the lock would go stale.
- `/spec` reads project memory in order: `CLAUDE.md` → `AGENTS.md` → `GEMINI.md` → `README.md`. Durable context belongs here.

## Architecture (`src/`)

- No ES modules. Files share state via `window.*` exports (`MAZE`, `createGame`, `update`, `draw`, `DIRS`, …). Load order in `src/index.html` is fixed: `maze.js` → `game.js` → `render.js` → `main.js`. New files must be added there in dependency order.
- `maze.js` — 28×31 maze as readable strings parsed to numbers; `MAZE` is pristine and never mutated.
- `game.js` — state and rules. `createGame()` copies `MAZE` into `game.grid`; all dot-eating mutates `game.grid`.
- `render.js` — draws from `game.grid`, never `MAZE`, or eaten dots reappear.
- `main.js` — RAF loop, keyboard input, overlay screens (start/win/lose).

## Game invariants

- Tile codes: `0` empty, `1` wall, `2` dot, `3` ghost-house door, `4` power pellet. Pacman is blocked by wall + door; ghosts by wall only.
- Cell (x, y), origin top-left; `TUNNEL_ROW = 14` wraps actors horizontally.
- Movement is in cells/frame (pacman 1/8, ghost 1/10) and only changes direction when `aligned()` (on a cell center). Keep that pattern for any new actor.
- Ghost `kind`: `hunter` (greedy toward Pacman) vs `random`. Decisions happen in `decideGhost()`.
