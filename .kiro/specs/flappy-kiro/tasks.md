# Implementation Plan: Flappy Kiro

## Overview

All game code lives in a single `index.html` file at the project root. Implementation follows a bottom-up approach: constants and scaffolding first, then core systems (physics, pipes, scoring), then UI and polish. Each task is self-contained and verifiable before moving on.

---

## Tasks

- [x] 1. Scaffold index.html with canvas, CSS, and constants
  - Create `index.html` at project root with an HTML5 `<canvas>` element (id `gameCanvas`).
  - Add a `<style>` block: dark body background `#1a1a2e`, canvas centered with `display:block; margin:auto; image-rendering:pixelated`.
  - Add a `<script>` block with all game constants: `LOGICAL_W=480`, `LOGICAL_H=640`, `GRAVITY=1800`, `FLAP_VY=-550`, `SCROLL_SPEED=200`, `PIPE_WIDTH=60`, `PIPE_GAP=160`, `PIPE_SPAWN_INTERVAL=1.8`, `GAP_MIN_Y=120`, `GAP_MAX_Y`, `COLLISION_INSET=6`, `COLORS` object.
  - Set canvas `width` and `height` attributes to `LOGICAL_W` and `LOGICAL_H`.
  - Implement `resize()` function that computes CSS scale and applies it; attach to `window.onresize` and call once on load.
  - _Requirements: 1.1, 1.2, 9.1, 10.1, 10.2, 10.3_

  - [ ]* 1.1 Write property test for canvas scale factor
    - **Property 10: Canvas scale factor preserves aspect ratio**
    - **Validates: Requirements 10.1, 10.2, 10.3**

- [x] 2. Implement asset loader (image + audio)
  - [x] 2.1 Load player sprite image
    - Create `const ghostImg = new Image(); ghostImg.src = 'assets/ghosty.png';` with an `onerror` handler that sets a `imgFailed` flag.
    - _Requirements: 1.3, 9.4_
  - [x] 2.2 Load audio assets
    - Create `jumpSfx` and `gameOverSfx` as `new Audio(...)` with error handling.
    - Implement `playSound(sfx)` that wraps `sfx.currentTime = 0; sfx.play()` in a try/catch for silent failure.
    - _Requirements: 1.3, 8.1, 8.2, 8.3_

- [x] 3. Implement player physics module
  - [x] 3.1 Define player object and reset function
    - Define `player` object with fields `x`, `y`, `vy`, `rotation`, `width=48`, `height=48`.
    - Implement `player.reset()` to restore `y = LOGICAL_H * 0.45`, `vy = 0`, `rotation = 0`.
    - _Requirements: 2.1, 2.3, 7.6_

  - [ ]* 3.2 Write property test for physics step determinism
    - **Property 1: Physics step is additive and deterministic**
    - **Validates: Requirements 2.1, 2.3**

  - [x] 3.3 Implement player.flap()
    - Set `player.vy = FLAP_VY`.
    - _Requirements: 2.2_

  - [ ]* 3.4 Write property test for flap impulse
    - **Property 2: Flap always resets vertical velocity to the flap impulse**
    - **Validates: Requirement 2.2**

  - [x] 3.5 Implement player.update(dt)
    - Apply `vy += GRAVITY * dt`, then `y += vy * dt`.
    - Lerp `rotation` toward target: if `vy < 0` target is `-0.4` rad; otherwise map `vy` to `[0, 1.2]` rad clamped.
    - _Requirements: 2.1, 2.3, 2.4_

  - [x] 3.6 Implement player.draw(ctx)
    - Save context, translate to player center, rotate, draw sprite (or fallback rect if `imgFailed`), restore.
    - _Requirements: 9.4_

- [x] 4. Checkpoint — verify player physics
  - Ensure all tests pass. Manually verify in browser: player falls under gravity, flap sends it upward, rotation tilts correctly.

- [x] 5. Implement pipe manager
  - [x] 5.1 Implement pipeManager.spawn()
    - Generate random `gapY` in `[GAP_MIN_Y, GAP_MAX_Y]`.
    - Push `{ x: LOGICAL_W + 10, gapY, scored: false }` to `pipes[]`.
    - _Requirements: 4.1, 4.2_

  - [ ]* 5.2 Write property test for pipe gap bounds
    - **Property 4: All generated pipe gaps are within safe bounds**
    - **Validates: Requirement 4.2**

  - [x] 5.3 Implement pipeManager.update(dt)
    - Decrement `pipe.x` by `SCROLL_SPEED * dt` for each pipe.
    - Remove pipes where `pipe.x + PIPE_WIDTH < 0`.
    - Accumulate spawn timer; call `spawn()` when timer exceeds `PIPE_SPAWN_INTERVAL`, then reset timer.
    - _Requirements: 4.1, 4.3, 4.4_

  - [ ]* 5.4 Write property test for pipe scrolling linearity
    - **Property 5: Pipe scrolling is linear and consistent**
    - **Validates: Requirement 4.3**

  - [ ]* 5.5 Write property test for off-screen pipe removal
    - **Property 6: Off-screen pipes are always removed**
    - **Validates: Requirement 4.4**

  - [x] 5.6 Implement pipeManager.draw(ctx)
    - Draw each pipe pair: top pipe as `(x, 0, PIPE_WIDTH, gapY)` rect in `COLORS.pipePrimary`.
    - Draw bottom pipe as `(x, gapY+PIPE_GAP, PIPE_WIDTH, LOGICAL_H - gapY - PIPE_GAP)` rect in `COLORS.pipePrimary`.
    - Add a `4px` teal (`COLORS.pipeHighlight`) border on each pipe for the Kiro accent look.
    - Add pipe caps (slightly wider rect at the edge facing the gap) for the classic Flappy Bird silhouette.
    - _Requirements: 4.5, 9.2, 9.5_

  - [x] 5.7 Implement pipeManager.reset()
    - Set `pipes = []` and `spawnTimer = 0`.
    - _Requirements: 7.6_

- [x] 6. Implement collision detection
  - [x] 6.1 Implement aabbOverlap(r1, r2) pure function
    - Returns `true` if two `{x,y,w,h}` rects overlap.
    - _Requirements: 5.1, 5.4_

  - [ ]* 6.2 Write property test for AABB collision correctness
    - **Property 7: AABB collision detection is symmetric and correct**
    - **Validates: Requirements 5.1, 5.2, 5.3, 5.4**

  - [x] 6.3 Implement checkCollisions() function
    - Build player rect inset by `COLLISION_INSET` on all sides.
    - Check against each pipe's top and bottom rect.
    - Check boundary: `player.y < 0 || player.y + player.height > LOGICAL_H`.
    - On collision: call `triggerGameOver()`.
    - _Requirements: 5.1, 5.2, 5.3_

- [x] 7. Implement scoring system
  - [x] 7.1 Implement scoreManager object
    - Define `current = 0` and `high = parseInt(localStorage.getItem('flappyKiroHigh') || '0')`.
    - Implement `increment()`: `current++`; if `current > high`, set `high = current`.
    - Implement `save()`: wrap `localStorage.setItem('flappyKiroHigh', high)` in try/catch.
    - Implement `reset()`: `current = 0`.
    - _Requirements: 6.1, 6.3, 6.4, 6.5_

  - [ ]* 7.2 Write property test for high-score max invariant
    - **Property 9: High score is always the maximum of current and stored**
    - **Validates: Requirements 6.3, 6.4**

  - [x] 7.3 Implement checkScoring() function
    - For each pipe where `!pipe.scored` and `player.x >= pipe.x + PIPE_WIDTH / 2`:
      - Call `scoreManager.increment()`, set `pipe.scored = true`.
    - _Requirements: 6.1_

  - [ ]* 7.4 Write property test for score idempotence
    - **Property 8: Score increments exactly once per pipe pair**
    - **Validates: Requirement 6.1**

- [x] 8. Implement state machine and game flow
  - [x] 8.1 Define STATE enum and gameState variable
    - `const STATE = { READY: 'READY', PLAYING: 'PLAYING', GAME_OVER: 'GAME_OVER' }; let gameState = STATE.READY;`
    - _Requirements: 7.1_

  - [x] 8.2 Implement startGame(), triggerGameOver(), restartGame() functions
    - `startGame()`: set `gameState = STATE.PLAYING`.
    - `triggerGameOver()`: set `gameState = STATE.GAME_OVER`; call `scoreManager.save()`; call `playSound(gameOverSfx)`.
    - `restartGame()`: call `player.reset()`, `pipeManager.reset()`, `scoreManager.reset()`; set `gameState = STATE.READY`.
    - _Requirements: 7.3, 7.4, 7.6, 8.2_

  - [ ]* 8.3 Write property test for state machine totality
    - **Property 3: State machine transitions are total and deterministic**
    - **Validates: Requirements 3.1–3.4, 7.1–7.6**

- [x] 9. Implement input handling
  - Attach `keydown` listener on `document`: if `event.code === 'Space'`, prevent default and call `handleInput()`.
  - Attach `pointerdown` (covers mouse click and touch tap) listener on `canvas`: call `handleInput()`.
  - Implement `handleInput()`: dispatch based on `gameState` (see design §State Machine).
  - _Requirements: 3.1, 3.2, 3.3, 3.4_

- [x] 10. Checkpoint — verify game loop integration
  - Connect player, pipes, scoring, collisions, and state machine into the game loop.
  - Implement `loop(timestamp)` with delta-time cap of 50ms; attach to `requestAnimationFrame`.
  - Ensure all tests pass. Manually play through: Ready → Playing → score → die → Game Over → restart.

- [x] 11. Implement renderer (draw functions)
  - [x] 11.1 Implement drawBackground(ctx)
    - Fill canvas with `COLORS.background`.
    - Add subtle parallax decorative elements (faint star dots or grid lines) for visual depth.
    - _Requirements: 9.1_

  - [x] 11.2 Implement drawHUD(ctx)
    - Draw current score centered at top of canvas in large bold white font with `2px` black shadow.
    - _Requirements: 6.2, 9.3_

  - [x] 11.3 Implement drawReadyScreen(ctx)
    - Draw semi-transparent overlay text: game title "FLAPPY KIRO" and a flashing prompt "TAP or SPACE to start".
    - Animate idle bob on player (small sine-wave `y` offset in Ready state).
    - _Requirements: 7.1, 7.2, 9.6_

  - [x] 11.4 Implement drawGameOverScreen(ctx)
    - Draw semi-transparent dark overlay over the full canvas.
    - Draw "GAME OVER" heading, current score, high score, and "TAP or SPACE to restart" prompt.
    - _Requirements: 7.5, 9.5_

  - [x] 11.5 Wire all draw calls into main draw() function
    - Call order: `drawBackground` → `pipeManager.draw` → `player.draw` → `drawHUD` → overlay screens.
    - _Requirements: 9.1–9.6_

- [x] 12. Final checkpoint — full game playtest and polish
  - Ensure all tests pass.
  - Verify sound effects trigger on flap and death.
  - Verify high score persists across browser refreshes.
  - Verify canvas scales correctly on mobile viewport (320×480) and landscape desktop.
  - Verify no console errors in Chrome, Firefox, and Safari.

---

## Notes

- Tasks marked with `*` are optional property/unit test tasks; they can be deferred for a faster MVP.
- All code lives in a single `index.html` — no separate `.js` or `.css` files needed.
- The logical coordinate system (480×640) is always used for physics and collision; CSS scaling is purely presentational.
- The delta-time cap (50 ms) prevents physics tunneling if the browser tab is backgrounded.

---

## Task Dependency Graph

```json
{
  "waves": [
    { "id": 0, "tasks": ["1", "1.1"] },
    { "id": 1, "tasks": ["2.1", "2.2"] },
    { "id": 2, "tasks": ["3.1", "3.3"] },
    { "id": 3, "tasks": ["3.2", "3.4", "3.5"] },
    { "id": 4, "tasks": ["3.6", "5.1", "7.1"] },
    { "id": 5, "tasks": ["5.2", "5.3", "6.1", "7.2", "7.3", "8.1"] },
    { "id": 6, "tasks": ["5.4", "5.5", "5.6", "5.7", "6.2", "6.3", "7.4", "8.2"] },
    { "id": 7, "tasks": ["8.3", "9"] },
    { "id": 8, "tasks": ["11.1", "11.2", "11.3", "11.4"] },
    { "id": 9, "tasks": ["11.5"] }
  ]
}
```
