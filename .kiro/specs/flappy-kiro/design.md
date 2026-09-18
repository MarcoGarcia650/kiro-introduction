# Design Document: Flappy Kiro

## Overview

Flappy Kiro is implemented as a **single `index.html` file** containing all HTML structure, CSS styling, and JavaScript game logic. Rendering is done via the HTML5 Canvas API. No build tools, bundlers, or external libraries are required.

The game targets a **logical resolution of 480 × 640 px** and scales uniformly to fill any viewport, with all physics and collision logic operating in logical coordinates.

---

## Architecture

The game follows a simple **game-loop architecture** with three distinct states managed by a central state machine:

```
READY ──[input]──▶ PLAYING ──[collision]──▶ GAME_OVER
                              ◀──[input]────────────
```

The JavaScript is organized into the following functional sections within `index.html`:

1. **Constants** — physics, dimensions, colors, asset paths
2. **Asset Loader** — loads image and audio with graceful failure
3. **State Machine** — `READY | PLAYING | GAME_OVER`
4. **Player Module** — position, velocity, rotation, draw, reset
5. **Pipe Manager** — spawn, scroll, remove, draw, collision check
6. **Score Manager** — increment, high-score persistence via `localStorage`
7. **Input Handler** — keyboard (Space) + pointer (click/touch)
8. **Renderer** — background, pipes, player, UI overlays
9. **Game Loop** — `requestAnimationFrame` delta-time loop

```
┌─────────────────────────────────────────────┐
│                  index.html                 │
│  ┌──────────┐   ┌──────────┐   ┌─────────┐ │
│  │  Player  │   │  Pipes   │   │  Score  │ │
│  └────┬─────┘   └────┬─────┘   └────┬────┘ │
│       │              │               │      │
│  ┌────▼──────────────▼───────────────▼────┐ │
│  │            Game Loop (rAF)             │ │
│  │  update(dt) → check collisions → draw  │ │
│  └────────────────────────────────────────┘ │
│  ┌─────────────────────────────────────────┐ │
│  │    Input Handler (keyboard + pointer)  │ │
│  └─────────────────────────────────────────┘ │
└─────────────────────────────────────────────┘
```

---

## Components and Interfaces

### Constants

```js
const LOGICAL_W = 480;
const LOGICAL_H = 640;
const GRAVITY    = 1800;    // px/s² in logical coords
const FLAP_VY    = -550;    // px/s upward impulse
const SCROLL_SPEED = 200;   // px/s leftward
const PIPE_WIDTH   = 60;    // px
const PIPE_GAP     = 160;   // px vertical gap
const PIPE_SPAWN_INTERVAL = 1.8; // seconds between spawns
const GAP_MIN_Y = 120;      // top of gap, minimum
const GAP_MAX_Y = LOGICAL_H - 120 - PIPE_GAP; // top of gap, maximum
const COLLISION_INSET = 6;  // px inset for AABB leniency
const COLORS = {
  background: '#1a1a2e',
  pipePrimary:   '#7c3aed',  // purple
  pipeHighlight: '#0d9488',  // teal
  scoreText: '#ffffff',
  overlay: 'rgba(0,0,0,0.55)',
};
```

### Player

```js
player = {
  x: LOGICAL_W * 0.25,  // fixed horizontal position
  y: number,            // vertical position (top of sprite)
  vy: number,           // vertical velocity px/s (positive = down)
  rotation: number,     // radians
  width: 48,            // sprite render width
  height: 48,           // sprite render height
}

// Methods:
player.reset()        // restore starting y, vy=0, rotation=0
player.flap()         // set vy = FLAP_VY
player.update(dt)     // vy += GRAVITY*dt; y += vy*dt; update rotation
player.draw(ctx)      // draw rotated ghosty.png sprite
```

Player rotation is clamped:
- After flap: lerp toward `-0.4` rad (nose up)
- Falling: lerp toward `+1.2` rad (nose down, max)

### Pipe Manager

```js
pipes = []  // array of pipe objects

pipe = {
  x: number,       // left edge in logical coords
  gapY: number,    // top of gap
  scored: boolean, // has this pipe pair been counted?
}

// Top pipe: rect { x, 0, PIPE_WIDTH, gapY }
// Bottom pipe: rect { x, gapY+PIPE_GAP, PIPE_WIDTH, LOGICAL_H - (gapY+PIPE_GAP) }

pipeManager.spawnTimer  // accumulates dt, spawns when >= PIPE_SPAWN_INTERVAL
pipeManager.update(dt)  // scroll + spawn + remove off-screen
pipeManager.draw(ctx)   // draw all pipes with Kiro colors + border/highlight
pipeManager.reset()     // clear pipes[], reset spawnTimer
```

### Score Manager

```js
scoreManager = {
  current: 0,
  high: parseInt(localStorage.getItem('flappyKiroHigh') || '0'),
}

scoreManager.increment()   // current++; update high if needed
scoreManager.save()        // write high to localStorage
scoreManager.reset()       // current = 0
```

### State Machine

```js
const STATE = { READY: 'READY', PLAYING: 'PLAYING', GAME_OVER: 'GAME_OVER' };
let gameState = STATE.READY;

function handleInput() {
  if (gameState === STATE.READY)    { startGame(); player.flap(); }
  if (gameState === STATE.PLAYING)  { player.flap(); playSound(jumpSfx); }
  if (gameState === STATE.GAME_OVER){ restartGame(); }
}
```

### Audio

```js
const jumpSfx    = new Audio('assets/jump.wav');
const gameOverSfx = new Audio('assets/game_over.wav');
jumpSfx.volume    = 0.6;
gameOverSfx.volume = 0.8;

function playSound(sfx) {
  try { sfx.currentTime = 0; sfx.play(); } catch(e) { /* silent fail */ }
}
```

### Renderer (Canvas)

The Canvas is positioned with CSS:
```css
canvas { display:block; margin:auto; image-rendering:pixelated; }
```

The `resize()` function calculates a CSS scale factor:
```js
function resize() {
  const scale = Math.min(window.innerWidth / LOGICAL_W,
                         window.innerHeight / LOGICAL_H);
  canvas.style.width  = (LOGICAL_W * scale) + 'px';
  canvas.style.height = (LOGICAL_H * scale) + 'px';
}
```

The canvas `width`/`height` attributes are always `LOGICAL_W × LOGICAL_H`; only the CSS size changes.

### Game Loop

```js
let lastTime = null;

function loop(timestamp) {
  const dt = Math.min((timestamp - lastTime) / 1000, 0.05); // cap at 50ms
  lastTime = timestamp;

  if (gameState === STATE.PLAYING) {
    player.update(dt);
    pipeManager.update(dt);
    checkCollisions();
    checkScoring();
  }

  draw();
  requestAnimationFrame(loop);
}
```

Delta-time is capped at 50 ms (20 fps minimum equivalent) to prevent large physics jumps when the tab is backgrounded.

---

## Data Models

### GameState

| Value | Description |
|-------|-------------|
| `READY` | Initial state; player idles, waiting for first input |
| `PLAYING` | Physics active, pipes scrolling, score accumulating |
| `GAME_OVER` | Physics/scrolling halted; overlay displayed |

### Pipe Object

| Field | Type | Description |
|-------|------|-------------|
| `x` | `number` | Logical x position of pipe pair's left edge |
| `gapY` | `number` | Logical y of the top of the gap |
| `scored` | `boolean` | Whether this pair has already incremented score |

### Player Object

| Field | Type | Description |
|-------|------|-------------|
| `x` | `number` | Fixed horizontal position (logical) |
| `y` | `number` | Vertical position of sprite top (logical) |
| `vy` | `number` | Vertical velocity px/s; positive = downward |
| `rotation` | `number` | Tilt angle in radians |
| `width` | `number` | Sprite draw width (logical px) |
| `height` | `number` | Sprite draw height (logical px) |

### ScoreState

| Field | Type | Description |
|-------|------|-------------|
| `current` | `number` | Score for current run |
| `high` | `number` | All-time high score (session-persistent via localStorage) |

---

## Correctness Properties

*A property is a characteristic or behavior that should hold true across all valid executions of a system — essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.*

### Property 1: Physics step is additive and deterministic

*For any* player state `(y, vy)` and time delta `dt`, after one physics step:
- `vy_new = vy + GRAVITY * dt`
- `y_new = y + vy_new * dt`

The result is purely a function of inputs; the same inputs always produce the same outputs.

**Validates: Requirements 2.1, 2.3**

---

### Property 2: Flap always resets vertical velocity to the flap impulse

*For any* player vertical velocity `vy` at the moment of a Flap event, after `player.flap()` the resulting `vy` equals `FLAP_VY` (a negative constant representing upward motion), regardless of previous velocity.

**Validates: Requirement 2.2**

---

### Property 3: State machine transitions are total and deterministic

*For any* combination of `(currentState, inputEvent)` where `inputEvent ∈ {space, click, tap}`, the resulting game state is fully determined by the transition table and never results in an undefined or invalid state.

**Validates: Requirements 3.1, 3.2, 3.3, 3.4, 7.1–7.6**

---

### Property 4: All generated pipe gaps are within safe bounds

*For any* pipe pair generated by `pipeManager.spawn()`, the `gapY` value satisfies:
`GAP_MIN_Y ≤ gapY ≤ GAP_MAX_Y`

This guarantees the gap is always reachable and never extends beyond canvas boundaries.

**Validates: Requirement 4.2**

---

### Property 5: Pipe scrolling is linear and consistent

*For any* pipe at position `x` at time `t`, after `n` physics steps each of duration `dt`:
`x_final = x - SCROLL_SPEED * n * dt`

The pipe's position is a linear function of elapsed time; no drift or compounding error occurs.

**Validates: Requirement 4.3**

---

### Property 6: Off-screen pipes are always removed

*For any* pipe array state after `pipeManager.update(dt)` completes, no pipe in the array satisfies `pipe.x + PIPE_WIDTH < 0`. Off-screen pipes are never retained.

**Validates: Requirement 4.4**

---

### Property 7: AABB collision detection is symmetric and correct

*For any* two axis-aligned rectangles `A` and `B`, the collision function returns `true` if and only if they overlap (with inset `COLLISION_INSET` applied to the player rect). The function is deterministic: same inputs always produce the same boolean output.

**Validates: Requirements 5.1, 5.2, 5.3, 5.4**

---

### Property 8: Score increments exactly once per pipe pair

*For any* pipe pair, during the run from the time the pipe is spawned to the time it exits the screen, the score is incremented exactly 0 or 1 times (never 2+). The `scored` flag on each pipe ensures idempotent counting.

**Validates: Requirement 6.1**

---

### Property 9: High score is always the maximum of current and stored

*For any* `(currentScore, storedHighScore)` pair at game-over time:
`newHighScore = max(currentScore, storedHighScore)`

The high score never decreases and always reflects the best run.

**Validates: Requirements 6.3, 6.4**

---

### Property 10: Canvas scale factor preserves aspect ratio

*For any* `(viewportWidth, viewportHeight)`:
`scale = min(viewportWidth / LOGICAL_W, viewportHeight / LOGICAL_H)`

The resulting `canvas.style.width = LOGICAL_W * scale` and `canvas.style.height = LOGICAL_H * scale` never exceed the viewport dimensions, and the aspect ratio `LOGICAL_W / LOGICAL_H` is preserved.

**Validates: Requirements 10.1, 10.2, 10.3**

---

## Error Handling

| Scenario | Handling |
|----------|----------|
| Asset image fails to load | Game continues; player rendered as a colored rect fallback |
| Audio file fails to load | `playSound()` catches the exception silently; no crash |
| `localStorage` unavailable | `try/catch` around all `localStorage` calls; high score resets to 0 for session |
| Delta-time spike (tab hidden) | `dt` capped at 50 ms to prevent physics tunneling |
| Window resize during gameplay | `resize()` recalculates CSS scale; logical coords unchanged |

---

## Testing Strategy

### Unit Tests (example-based)

Because this is a single-file browser game, unit tests are not run with a test framework during the build step. Instead, the correctness properties above guide manual testing and can be extracted into a standalone test script for CI if desired.

Key example tests to run manually or via a headless browser:

- Verify `player.update(0.016)` produces expected `y` and `vy` changes for known inputs.
- Verify `player.flap()` always sets `vy = FLAP_VY`.
- Verify `pipeManager.spawn()` always produces `gapY` in `[GAP_MIN_Y, GAP_MAX_Y]` over 1000 iterations.
- Verify AABB collision correctly detects overlap and non-overlap for edge-case rectangles.
- Verify `scoreManager.increment()` only fires once per pipe pair.
- Verify `max(current, stored)` high-score logic.

### Property-Based Tests

The mathematical properties (1, 2, 4, 5, 6, 7, 8, 9, 10) are suitable for property-based testing using a library such as [fast-check](https://fast-check.io/) in a Node.js environment, extracting the pure functions from `index.html`. Each property test should run a minimum of 100 iterations with randomly generated inputs.

### Integration / Smoke Tests

- Open `index.html` via `file://` in Chrome, Firefox, Safari, and mobile browsers.
- Verify assets load and sounds play.
- Verify `localStorage` persistence across page refreshes.
- Verify canvas scaling on viewport resize.
