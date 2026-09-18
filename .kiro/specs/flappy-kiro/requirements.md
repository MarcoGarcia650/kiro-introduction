# Requirements Document

## Introduction

Flappy Kiro is a retro-styled, browser-based endless-scroller game inspired by Flappy Bird. The player controls Kiro's ghost (a ghost sprite) through an infinite series of pipe obstacles. The game runs entirely in a single HTML file using the Canvas API — no build step, no external dependencies. It targets both desktop and mobile browsers and features Kiro-themed visuals (dark background, purple/teal accent colors) and audio feedback for flap and death events.

## Glossary

- **Game**: The Flappy Kiro application running in the browser.
- **Player**: The ghost character (`assets/ghosty.png`) controlled by the user.
- **Pipe Pair**: A vertically-aligned pair of pipes (top pipe + bottom pipe) with a gap through which the player must pass.
- **Gap**: The vertical opening between the top and bottom pipe of a Pipe Pair.
- **Score**: The integer count of Pipe Pairs the player has successfully passed through in the current run.
- **High Score**: The highest Score achieved in the current browser session (stored in `localStorage`).
- **Game Over**: The state entered when the Player collides with a pipe or the canvas boundary.
- **Canvas**: The HTML5 `<canvas>` element that renders the game world.
- **Frame**: A single rendered update cycle, targeting 60 frames per second.
- **Gravity**: A constant downward acceleration applied to the Player each Frame.
- **Flap**: A jump impulse that temporarily overrides Gravity by applying an upward velocity to the Player.

---

## Requirements

### Requirement 1: Single-File Delivery

**User Story:** As a developer, I want the game delivered as a single HTML file, so that I can open it in a browser with no build step or server.

#### Acceptance Criteria

1. THE Game SHALL be fully contained within a single `index.html` file including all CSS, JavaScript, and Canvas rendering code.
2. THE Game SHALL load and run correctly when opened via `file://` protocol in a modern browser (Chrome, Firefox, Safari, Edge).
3. THE Game SHALL reference asset files (`assets/ghosty.png`, `assets/jump.wav`, `assets/game_over.wav`) using relative paths from the file's location.

---

### Requirement 2: Player Physics

**User Story:** As a player, I want the ghost character to fall under gravity and jump when I input a command, so that I have precise control over its vertical position.

#### Acceptance Criteria

1. THE Player SHALL fall downward each Frame due to a constant Gravity acceleration.
2. WHEN a Flap input is received, THE Player SHALL immediately receive an upward velocity impulse, overriding any current downward velocity.
3. WHILE the Game is in the Playing state, THE Player SHALL continuously update its vertical position based on current velocity and Gravity.
4. THE Player's rotation SHALL tilt downward as it falls and tilt upward briefly after a Flap, providing visual feedback.

---

### Requirement 3: Input Handling

**User Story:** As a player, I want to control the ghost by clicking, pressing Space, or tapping on mobile, so that the game is accessible on any device.

#### Acceptance Criteria

1. WHEN the player presses the Space key, THE Game SHALL trigger a Flap action if the Game is in the Playing state.
2. WHEN the player clicks or taps the Canvas, THE Game SHALL trigger a Flap action if the Game is in the Playing state.
3. WHEN the player presses Space, clicks, or taps while in the Ready state, THE Game SHALL transition to the Playing state and trigger the first Flap.
4. WHEN the player presses Space, clicks, or taps while in the Game Over state, THE Game SHALL restart the game.

---

### Requirement 4: Pipe Obstacles

**User Story:** As a player, I want to navigate through a continuous stream of pipe obstacles, so that the game provides a meaningful and increasing challenge.

#### Acceptance Criteria

1. THE Game SHALL spawn Pipe Pairs at regular horizontal intervals as the game scrolls.
2. EACH Pipe Pair SHALL have a randomised Gap position within safe vertical bounds, ensuring the Gap is always reachable.
3. THE Game SHALL scroll Pipe Pairs leftward at a constant speed each Frame.
4. WHEN a Pipe Pair exits the left edge of the Canvas, THE Game SHALL remove it from the active set.
5. THE Pipes SHALL be rendered using Kiro-themed accent colors (purple and/or teal).

---

### Requirement 5: Collision Detection

**User Story:** As a player, I want the game to accurately detect when my ghost hits a pipe or the screen boundary, so that the game ends fairly.

#### Acceptance Criteria

1. WHEN the Player's bounding box overlaps any Pipe's bounding box, THE Game SHALL transition to the Game Over state.
2. WHEN the Player's vertical position exceeds the bottom Canvas boundary, THE Game SHALL transition to the Game Over state.
3. WHEN the Player's vertical position goes above the top Canvas boundary, THE Game SHALL transition to the Game Over state.
4. THE collision check SHALL use axis-aligned bounding-box (AABB) logic with a small inset margin to allow pixel-near misses and feel fair.

---

### Requirement 6: Scoring

**User Story:** As a player, I want to see my score increase as I pass through pipes, so that I have clear progress feedback.

#### Acceptance Criteria

1. WHEN the Player's horizontal center passes the horizontal center of a Pipe Pair, THE Game SHALL increment the Score by 1.
2. THE Score SHALL be displayed on screen at all times while in the Playing state.
3. WHEN the Game transitions to the Game Over state, THE Game SHALL compare the current Score to the High Score.
4. IF the current Score exceeds the High Score, THE Game SHALL update the High Score to the current Score.
5. THE High Score SHALL persist across restarts within the same browser session using `localStorage`.

---

### Requirement 7: Game States

**User Story:** As a player, I want clear transitions between Ready, Playing, and Game Over states, so that I understand what is happening at all times.

#### Acceptance Criteria

1. WHEN the Game first loads, THE Game SHALL enter the Ready state and display a start prompt.
2. WHILE in the Ready state, THE Game SHALL render the Player at a fixed starting position and animate an idle bob.
3. WHEN transitioning to the Playing state, THE Game SHALL begin scrolling pipes and applying physics.
4. WHEN the Game enters the Game Over state, THE Game SHALL stop all scrolling and physics updates.
5. WHEN the Game enters the Game Over state, THE Game SHALL display the current Score, the High Score, and a restart prompt.
6. WHEN restarting, THE Game SHALL reset the Score, Player position, velocity, and all active Pipe Pairs, then return to the Ready state.

---

### Requirement 8: Audio Feedback

**User Story:** As a player, I want sound effects for flapping and dying, so that the game feels responsive and satisfying.

#### Acceptance Criteria

1. WHEN a Flap action occurs, THE Game SHALL play `assets/jump.wav`.
2. WHEN the Game transitions to the Game Over state, THE Game SHALL play `assets/game_over.wav`.
3. IF an audio file fails to load, THE Game SHALL continue to function silently without throwing an error.

---

### Requirement 9: Visual Style

**User Story:** As a player, I want a retro, Kiro-themed visual experience so that the game feels unique and on-brand.

#### Acceptance Criteria

1. THE Canvas background SHALL use a dark color (approximately `#1a1a2e` or similar deep navy/dark purple).
2. THE Pipes SHALL be rendered in Kiro accent colors (purple `#7c3aed` and teal `#0d9488` or similar).
3. THE Score text SHALL be displayed in a large, bold, white font with a subtle drop shadow for readability.
4. THE Player sprite SHALL be rendered using `assets/ghosty.png` scaled to an appropriate gameplay size.
5. THE Game Over screen SHALL display a semi-transparent overlay with legible Score and High Score text.
6. THE Ready screen SHALL display a title and a flashing or animated start prompt.

---

### Requirement 10: Responsiveness

**User Story:** As a player on any device, I want the game canvas to scale to fit my screen, so that the experience is good on both desktop and mobile.

#### Acceptance Criteria

1. THE Canvas SHALL resize to fit the browser viewport on load and when the window is resized.
2. THE Game's internal logical resolution SHALL remain constant (e.g., 480×640) and the Canvas SHALL scale uniformly to fit.
3. WHILE the Canvas is scaled, THE Game's collision detection and physics SHALL operate on the internal logical resolution, not the CSS pixel size.
4. THE Game SHALL remain playable on a viewport as small as 320×480 CSS pixels.
