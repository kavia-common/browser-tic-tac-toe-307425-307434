# React Tic Tac Toe (Ocean Professional) — Implementation Plan

## Scope and goals

This project will deliver a single-container, frontend-only React Tic Tac Toe application designed for two players playing locally on the same device. The v1 goal is a polished, responsive UI aligned with the Ocean Professional theme, with clean separation between pure game logic and presentational components.

The app must support standard Tic Tac Toe rules with the following user-facing features:

The game must allow two-player local play with alternating turns (X then O) and must prevent illegal moves. The game must detect a win (three in a row) and a draw, and it must stop accepting moves once the game is over. The game must provide a reset / new round action. The game must display a scoreboard that tracks wins for X and O and optionally draws across rounds within the current session (in-memory for v1).

## Architecture overview

The app will use a small “game engine” layer of pure utilities separated from UI components. The engine layer should contain deterministic, testable functions with no React or DOM dependencies. The UI layer will be a set of React components that render the game state, handle user input, and call engine functions to derive game status.

This repository currently contains a lightweight Create React App template in `frontend_app/` with basic theme toggling via CSS variables and `data-theme` in `src/App.js` and `src/App.css`. The Tic Tac Toe app can either reuse this theme toggle mechanism or replace it with Ocean Professional fixed styling for v1. The plan below assumes Ocean Professional as the default theme, optionally retaining the light/dark toggling pattern if desired.

A suggested folder layout (under `frontend_app/src/`) is:

- `game/` for engine utilities (pure functions and types).
- `components/` for reusable UI pieces (Board, Square, StatusBar, Controls).
- `pages/` for top-level screens (GamePage).
- `styles/` optional, or continue using `App.css` for global styles and component-level CSS modules/partials if preferred.

## Component structure and responsibilities

### GamePage (page container)
`GamePage` is the top-level game screen that owns the game state and orchestrates the child components. It should:
- Hold board state, turn state, computed game status, winner line, and scoreboard state.
- Provide event handlers such as `onSquareClick(index)`, `onResetRound()`, `onResetScores()`, and optional `onUndo()`.
- Render the layout described by the style guide: centered board, scoreboard and status above, controls below.

`GamePage` should be responsible for calling the game engine utility to evaluate board outcomes after each move.

### Board (presentational + event delegation)
`Board` renders the 3x3 grid and delegates clicks to its parent via an `onSquareClick(index)` callback. It should:
- Receive `squares` as an array of 9 values (`'X' | 'O' | null`).
- Receive `winnerLine` (indices of winning squares) to visually highlight the winning combination.
- Render nine `Square` children in a grid.

### Square (atomic interactive control)
`Square` renders a single cell. It should:
- Display the value (`X`, `O`, or empty).
- Be keyboard accessible (use a `<button>`).
- Apply hover/focus styles, and show disabled state when not clickable.
- Apply a highlight style if its index is part of `winnerLine`.

### StatusBar (status + scoreboard summary)
`StatusBar` shows the current game status and a compact scoreboard section. It should:
- Display “Next player: X/O” while the game is active.
- Display “Winner: X/O” when a win is detected.
- Display “Draw” when the game ends in a draw.
- Display scoreboard values (X wins, O wins, draws).

This component should remain presentational, receiving computed strings/data from `GamePage`.

### Controls (round and session actions)
`Controls` provides actions:
- “New Round” / “Reset Board” to clear the board and start a fresh round (keeping scores).
- “Reset Scores” to clear the scoreboard.
- Optional: “Undo” to step back one move (only when history exists).

Controls should be presentational and take callbacks and disabled states from `GamePage`.

## State management approach

All state can live in `GamePage` using React `useState` and derived computation via the game engine utilities. React Context is unnecessary for this small app.

### Core state
The following state variables are recommended:

- `squares: Array<'X' | 'O' | null>` length 9.
- `currentPlayer: 'X' | 'O'`.
- `gameStatus: 'playing' | 'won' | 'draw'`.
- `winner: 'X' | 'O' | null` (null unless won).
- `winnerLine: number[] | null` (indices for styling).
- `scores: { x: number; o: number; draws: number }`.

### Optional undo state
If undo is included in v1, store:
- `history: Array<Array<'X' | 'O' | null>>` (snapshots of the board), or
- `moves: number[]` + reconstruct boards (more complex, not necessary).

Undo behavior should:
- Pop history, set `squares` to previous snapshot.
- Toggle `currentPlayer`.
- Recompute `gameStatus`, `winner`, and `winnerLine`.

### Derived values
Avoid duplicating derived state when possible. For example, `gameStatus`, `winner`, and `winnerLine` can be computed from `squares` after each move using a utility function. If you store them, ensure they are updated in exactly one place (a single move handler and reset handlers) to prevent inconsistencies.

## Game engine utilities (pure logic)

The game engine should be implemented as pure functions in `frontend_app/src/game/` and designed to be easily unit-tested.

Recommended utilities:

- `getNextPlayer(currentPlayer): 'X' | 'O'`.
- `isMoveLegal(squares, index): boolean` to ensure a square is empty and game is not over (or check game status in caller).
- `calculateWinner(squares): { winner: 'X' | 'O' | null; line: number[] | null }`.
- `calculateGameStatus(squares): { status: 'playing' | 'won' | 'draw'; winner: 'X' | 'O' | null; line: number[] | null }`.

Implementation notes:
- `calculateWinner` should check the 8 possible winning lines for 3-in-a-row.
- A draw occurs when there is no winner and all squares are non-null.
- The utilities must not mutate the input array.

## UI/UX design notes (Ocean Professional theme)

The UI should follow a modern, clean aesthetic with subtle shadows, rounded corners, and minimalist composition. The Ocean Professional palette from the work item should be applied consistently:

- Primary: `#2563EB` (blue) for primary actions, focus rings, and player X (optional mapping).
- Secondary: `#F59E0B` (amber) for highlights, accent borders, and player O (optional mapping).
- Error: `#EF4444` for destructive actions or invalid states (rare in v1).
- Background: `#f9fafb` for the app canvas.
- Surface: `#ffffff` for cards/panels.
- Text: `#111827` for primary typography.

The background should use a subtle gradient similar to the style guide guidance (for example, a faint blue tint blended into near-white). The game should be centered both vertically and horizontally, and should remain usable on narrow mobile viewports.

Component styling guidelines:
- Use rounded corners for the board container and buttons.
- Use subtle box-shadow on the main card/surface and on hover for interactive elements.
- Provide clear hover and focus-visible states for squares and buttons.
- The board grid should have consistent gaps and a strong but minimal border treatment.
- The winning line should be highlighted using an accent background or outline, while keeping text legible.

Responsiveness:
- The board should scale down on smaller screens; using CSS `clamp()` for square size is recommended.
- Controls can stack vertically on narrow widths.
- Typography should scale modestly and avoid overflow.

Accessibility:
- Each `Square` must be a real button with an `aria-label` like “Place X at row 1 column 2”.
- The status region should be announced politely using `aria-live="polite"` so screen reader users are informed of turn changes and game end.
- Ensure color contrast meets WCAG AA for text and interactive elements.
- Use `:focus-visible` to show keyboard focus without distracting mouse users.

## Implementation steps and milestones

### Milestone 1: Game engine utilities
1. Create `src/game/` with `calculateWinner` and `calculateGameStatus`.
2. Add unit tests for win detection, draw detection, and “playing” status.
3. Confirm utilities do not mutate input.

Exit criteria: all unit tests pass and cover all win lines and draw scenarios.

### Milestone 2: Core UI components (static)
1. Create `Square`, `Board`, `StatusBar`, and `Controls` components.
2. Render a static 3x3 board and basic layout in `GamePage`.
3. Apply Ocean Professional styling (global tokens + component styles).

Exit criteria: UI matches intended layout, is responsive, and is keyboard navigable.

### Milestone 3: Gameplay wiring
1. Implement `GamePage` state: `squares`, `currentPlayer`, `scores`, and derived status.
2. Wire `onSquareClick` to place marks, recompute status, and stop on end state.
3. Implement “New Round” (reset board, keep scores).
4. Implement automatic score updates when transitioning to won/draw.

Exit criteria: full local two-player play works with win/draw detection and correct score updates.

### Milestone 4: Polish and optional undo
1. Add winner line highlight.
2. Add optional undo support (history + button states).
3. Improve micro-interactions (hover transitions, subtle animations, focus rings).
4. Ensure lint clean and tests stable.

Exit criteria: polished interactions, no console errors, and stable test coverage for core flows.

## Testing strategy

Testing should be split between game logic unit tests and UI interaction tests.

### Unit tests (game engine)
Use Jest (already part of CRA) for pure function tests:
- `calculateWinner` should correctly identify all eight win lines for both X and O.
- `calculateGameStatus` should return:
  - `playing` when there are empty squares and no winner,
  - `won` with correct `winner` and `line`,
  - `draw` when board full with no winner.
- Include regression tests for “almost win” cases.

These tests should live under `src/game/__tests__/` or alongside modules, depending on repository conventions.

### Component tests (React Testing Library)
Use React Testing Library (available via CRA defaults in the template) to test:
- Clicking a square places the current player mark.
- Turns alternate between X and O.
- Clicking an occupied square does nothing.
- After a win, further clicks do nothing and status displays winner.
- After a draw, status displays draw and further clicks do nothing.
- “New Round” clears board but retains scores.
- “Reset Scores” clears scores.
- Optional undo: “Undo” restores previous board and current player.

If the existing `src/App.test.js` is updated during implementation, ensure tests reflect the new UI rather than the template “learn react” link.

## Future extensibility notes

The design should anticipate enhancements without requiring major rewrites.

Potential v2+ features:
- AI opponent: add an AI move selector (minimax or heuristic) implemented as another pure utility. Keep UI unchanged; only add a game mode toggle and AI turn triggering.
- Online play: introduce a backend and WebSocket synchronization. Keep engine pure and update state via events from the network layer.
- Variable board sizes: generalize board representation and win-checking to NxN with a configurable win condition (e.g., 4 in a row). The UI should render a dynamic grid and `Square` size should be computed from N.
- Persistence: store scoreboard and last mode in `localStorage` with a simple hydration on mount.

To enable these, keep the engine API stable and avoid mixing engine logic into components.

## Repository integration notes

This repository’s React template currently uses `src/App.js` and `src/App.css` for global layout and theme toggling. The Tic Tac Toe app can be implemented by replacing the `App` contents to render `GamePage` and by updating the CSS variables and component styles to match the Ocean Professional palette. Ensure the final app still runs via `npm start` in `frontend_app/` on port 3000.

