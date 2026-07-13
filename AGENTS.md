# ResearchOutput — Word Puzzle Game

A client-side browser game (HTML + CSS + vanilla JS) served as static files from `www/`. There is no backend, database, build step, package manager, or automated tests/linters. All state (selected mode, difficulty, level, score, completed levels) is persisted in the browser via `localStorage`.

## Cursor Cloud specific instructions

- The application code lives on the `create-game-modes` branch. The `main` branch currently contains only `README.md`, so check out `create-game-modes` (or a branch based on it) to work with the game.
- Run it as a static site — there is nothing to install or build. Serve the `www/` directory and open it in a browser:
  - `cd www && python3 -m http.server 8000` then open `http://localhost:8000/index.html` (`python3` is preinstalled).
- No test runner or linter is configured (no `package.json`, no config files), so there are no lint/test/build commands to run.
- App flow: `index.html` (pick mode) → `difficulty.html` → for Game Mode 1, `level-selection.html` → `game-with-question.html`; for Game Mode 2, directly to `game-without-question.html`.
- Valid answers for manual testing are in the data files, not derivable from the UI: Game Mode 1 answers are in `www/js/questions.js` (e.g. Easy Level 1 = `money, food, traffic, noise, prices`); Game Mode 2 words are in `www/js/word-bank.js`.
- `www/rules.html` and `www/settings.html` are intentionally empty stubs linked from the main menu — they are not yet implemented.
- Progress is gated by `localStorage` (e.g. levels beyond 1 unlock only after completing the prior level). To retest from a clean state, clear the site's `localStorage`.
