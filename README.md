# Day 1 — Tetris

A single-file Tetris clone built with pure HTML, CSS, and JavaScript. No libraries, no build tools.

## How to play

Open `tetris.html` in any browser. No server needed.

| Key | Action |
|-----|--------|
| `Space` | Start / Hard drop |
| `←` `→` | Move left / right |
| `↑` | Rotate clockwise |
| `↓` | Soft drop (+1pt per row) |

Hard drop scores 2pts per row. Clearing multiple lines at once multiplies the bonus by your current level.

## Scoring

| Lines cleared | Base points |
|---------------|-------------|
| 1 | 100 |
| 2 | 300 |
| 3 | 500 |
| 4 (Tetris) | 800 |

All values are multiplied by the current level. Every 10 lines clears a level up and the drop speed increases.

## What I learned

### Canvas API basics
Drawing on a `<canvas>` element is essentially just issuing a series of imperative draw calls — `fillRect`, `beginPath`, `stroke`, etc. There is no retained scene graph, so you redraw the entire board every frame. Once that clicked, the rendering loop became straightforward.

### Game loop with `requestAnimationFrame`
Instead of `setInterval`, using `requestAnimationFrame` ties the loop to the display refresh rate and pauses automatically when the tab is hidden. The pattern is:

```js
function loop(timestamp) {
  const dt = timestamp - lastTime;
  lastTime = timestamp;
  // accumulate dt, act when threshold crossed
  rafId = requestAnimationFrame(loop);
}
```

Accumulating `dt` rather than acting on every frame means the game speed stays consistent regardless of frame rate.

### Separating game state from rendering
All game state lives in plain JS variables (`board`, `current`, `score`, etc.). The rendering functions just read that state and draw it — they never mutate it. This separation made debugging much easier: I could `console.log` state without touching any drawing code.

### Matrix rotation
Rotating a Tetris piece clockwise is a classic matrix transpose + reverse:

```js
function rotate(shape) {
  const N = shape.length, M = shape[0].length;
  const result = Array.from({ length: M }, () => Array(N).fill(0));
  for (let r = 0; r < N; r++)
    for (let c = 0; c < M; c++)
      result[c][N - 1 - r] = shape[r][c];
  return result;
}
```

It's a small function but it took a moment to reason about indices correctly.

### Ghost piece
The ghost (shadow) shows where the piece will land. It's calculated by repeatedly testing `valid(shape, x, y+1)` until it fails — then rendering the piece at that `y` with low `globalAlpha`. Cheap to compute, and it makes the game dramatically more playable.

### Collision detection
Every movement or rotation goes through a single `valid()` function that checks all filled cells against walls, the floor, and the existing board. Centralizing this meant I never had to write ad-hoc boundary checks anywhere else.

### Line clearing with `splice` + `unshift`
When a full row is found, `splice` removes it and `unshift` prepends a new empty row at the top. Iterating from the bottom and re-checking the same index after a splice handles multiple consecutive clears correctly.

## What I would do differently

- Add wall kicks (SRS rotation system) so pieces don't get stuck at walls on rotate.
- Persist the high score in `localStorage`.
- Add a lock-delay so the piece doesn't instantly lock when it touches the ground.
- Sound effects with the Web Audio API.
