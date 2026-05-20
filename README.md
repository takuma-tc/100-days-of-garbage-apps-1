# Day 1 — Tetris

A single-file Tetris clone built with pure HTML, CSS, and JavaScript. No libraries, no build tools, no bundler — just open `tetris.html` in a browser.

## How to play

| Key | Action |
|-----|--------|
| `Space` | Start / Hard drop |
| `←` `→` | Move left / right |
| `↑` | Rotate clockwise (SRS) |
| `↓` | Soft drop (+1pt per row) |
| Hard drop | +2pt per row fallen |

## Scoring

| Lines cleared | Base points |
|---------------|-------------|
| 1 | 100 |
| 2 | 300 |
| 3 | 500 |
| 4 (Tetris) | 800 |

Base points are multiplied by the current level. Every 10 lines = level up, drop speed increases.

### Cascade chain bonus

After clearing lines, blocks fall column-by-column (not row-by-row). If new lines form from the collapse, a chain triggers with a score multiplier:

| Chain depth | Multiplier |
|-------------|------------|
| Initial clear | ×1.0 |
| CHAIN! | ×1.5 |
| ×2 CHAIN! | ×2.0 |
| ×3 CHAIN! | ×2.5 |

Chains are capped at 3 to keep board strategy meaningful.

High score is saved in `localStorage` and persists across sessions.

---

## What I learned

### Canvas API — stateless drawing

There is no retained scene graph. Every frame you clear the canvas and redraw everything from scratch. The rendering function just reads game state and emits draw calls (`fillRect`, `beginPath`, `stroke`). Once I stopped thinking "update an element" and started thinking "describe the current state visually", it became straightforward.

### Game loop with `requestAnimationFrame` and delta time

`setInterval` drifts. `requestAnimationFrame` ties the loop to the display refresh and pauses automatically when the tab is hidden. The key pattern is accumulating `dt` rather than acting on every frame:

```js
function loop(timestamp) {
  const dt = Math.min(timestamp - lastTime, 50); // cap to avoid spiral after tab switch
  lastTime = timestamp;
  dropAcc += dt;
  if (dropAcc >= dropInterval) {
    dropAcc -= dropInterval;
    current.y++;
  }
  requestAnimationFrame(loop);
}
```

Capping `dt` at 50ms prevents the piece from teleporting through the board after the tab was hidden.

### State machine for multi-phase animation

Once line clears needed a flash animation and a cascade pause, a simple boolean `running` wasn't enough. I introduced a `phase` string:

```
'idle' → 'playing' → 'flash' → 'cascade' → 'playing' → ...
```

Each phase has its own update and draw logic. Input is only accepted during `'playing'`. Transitions are explicit. Without this structure, the timing logic would have become an unmaintainable mess of flags and timeouts.

### Column-independent gravity (cascade mechanic)

Standard Tetris uses uniform row gravity: when a line is cleared, everything above drops by one row. For the cascade, I replaced this with column-by-column compaction:

```js
function applyCascadeGravity() {
  for (let c = 0; c < COLS; c++) {
    const cells = [];
    for (let r = 0; r < ROWS; r++)
      if (board[r][c]) cells.push(board[r][c]);
    for (let r = ROWS - 1; r >= 0; r--)
      board[r][c] = cells.length > 0 ? cells.pop() : 0;
  }
}
```

Each column collects its non-empty cells top-to-bottom and packs them to the bottom. This can cause isolated blocks from adjacent columns to form new complete lines — the chain. The cap of 3 chains exists because unlimited cascades would let a player set up a board that clears itself, removing all strategic tension.

### SRS wall kicks

Without wall kicks, rotating near a wall silently fails. With SRS (Super Rotation System), the game tries up to 5 offset positions on every rotation:

```js
const KICKS_JLSTZ = [
  [[0,0],[-1,0],[-1,-1],[0,2],[-1,2]],  // 0→R
  [[0,0],[1,0],[1,1],[0,-2],[1,-2]],     // R→2
  ...
];
```

The tables come from the Tetris Guideline spec in y-up coordinates. Since canvas uses y-down, all y values are negated. I piece has its own kick table because its rotation center is different.

### Lock delay with reset cap

Without lock delay, pieces lock instantly on touch — punishing any attempt to slide or rotate at the bottom. With a 500ms delay, the player has a window to adjust. But unlimited resets would let a player stall forever, so resets are capped at 15:

```js
function resetLock() {
  if (onGround && lockResets < MAX_LOCK_RESETS) {
    lockAcc = 0;
    lockResets++;
  }
}
```

The piece pulses (alpha oscillates) while lock delay is ticking, and a thin progress bar along its bottom edge shows how much time is left. Feedback without UI clutter.

### Web Audio API — synthesized sound effects

No audio files. All sounds are generated programmatically using oscillators:

```js
function beep(freq, dur, type = 'square', vol = 0.12, offset = 0) {
  const osc = ac.createOscillator();
  const gain = ac.createGain();
  osc.connect(gain); gain.connect(ac.destination);
  osc.type = type;
  osc.frequency.setValueAtTime(freq, t);
  gain.gain.setValueAtTime(vol, t);
  gain.gain.exponentialRampToValueAtTime(0.001, t + dur);
  osc.start(t); osc.stop(t + dur);
}
```

`exponentialRampToValueAtTime` creates a natural fade-out. Chaining multiple `beep()` calls with staggered `offset` values creates arpeggios for line clears, level ups, and chain sounds. The `AudioContext` is created lazily on the first user gesture to satisfy browser autoplay policy.

### Matrix rotation — index reasoning

Clockwise rotation of a 2D matrix is transpose + horizontal flip, which collapses to:

```
result[c][N - 1 - r] = shape[r][c]
```

It's 5 lines of code but took real pencil-and-paper reasoning to get the indices right. The resulting array has dimensions `M × N` (swapped from `N × M`), which matters for non-square pieces like I.

### Ghost piece

Calculated by incrementing `y` until `valid()` returns false, then drawn at low `globalAlpha`. Costs almost nothing computationally and makes the game dramatically more playable. The ghost also doubles as a visual check that the collision detection is working correctly.

### Collision detection as a single gate

Every movement, rotation, and gravity step goes through one function:

```js
function valid(shape, ox, oy) {
  for (let r = 0; r < shape.length; r++)
    for (let c = 0; c < shape[r].length; c++)
      if (shape[r][c]) {
        const x = ox + c, y = oy + r;
        if (x < 0 || x >= COLS || y >= ROWS) return false;
        if (y >= 0 && board[y][x]) return false;
      }
  return true;
}
```

`y >= 0` in the board check allows pieces to spawn above the visible area without immediately colliding. Centralizing this meant I never had to write ad-hoc boundary checks elsewhere — every feature that moves a piece just calls `valid()` first.
