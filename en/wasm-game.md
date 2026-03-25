---
title: Building a Wasm Game
slug: wasm-game
difficulty: advanced
tags: [graphics, simulation]
order: 24
description: Build a complete browser game with Rust/Wasm — game loop, input handling, collision detection, and Canvas rendering.
starter_code: |
  use wasm_bindgen::prelude::*;

  #[wasm_bindgen]
  pub struct Game {
      player_x: f64,
      player_y: f64,
      player_size: f64,
      targets: Vec<(f64, f64)>,
      score: u32,
      width: f64,
      height: f64,
  }

  #[wasm_bindgen]
  impl Game {
      #[wasm_bindgen(constructor)]
      pub fn new(width: f64, height: f64) -> Self {
          let mut targets = Vec::new();
          for i in 0..5 {
              targets.push((
                  100.0 + (i as f64) * 120.0,
                  100.0 + (i as f64) * 60.0,
              ));
          }
          Self {
              player_x: width / 2.0,
              player_y: height / 2.0,
              player_size: 20.0,
              targets,
              score: 0,
              width,
              height,
          }
      }

      pub fn move_player(&mut self, dx: f64, dy: f64) {
          self.player_x = (self.player_x + dx).clamp(0.0, self.width);
          self.player_y = (self.player_y + dy).clamp(0.0, self.height);
      }

      pub fn update(&mut self) {
          // Check collisions with targets
          self.targets.retain(|&(tx, ty)| {
              let dist = ((self.player_x - tx).powi(2) + (self.player_y - ty).powi(2)).sqrt();
              if dist < self.player_size + 15.0 {
                  self.score += 10;
                  false // Remove collected target
              } else {
                  true
              }
          });

          // Respawn targets if all collected
          if self.targets.is_empty() {
              for i in 0..5 {
                  self.targets.push((
                      (i as f64 * 137.0) % self.width,
                      (i as f64 * 97.0) % self.height,
                  ));
              }
          }
      }

      pub fn score(&self) -> u32 { self.score }
      pub fn player_x(&self) -> f64 { self.player_x }
      pub fn player_y(&self) -> f64 { self.player_y }
      pub fn player_size(&self) -> f64 { self.player_size }

      pub fn target_count(&self) -> usize { self.targets.len() }

      pub fn target_positions(&self) -> Vec<f64> {
          self.targets.iter().flat_map(|&(x, y)| vec![x, y]).collect()
      }
  }
expected_output: |
  Game created: 800x600
  Player at (400, 300)
  5 targets spawned
  Score: 0
  Collecting targets...
  Score: 50
---

## Game Architecture

A Wasm game splits responsibility between Rust and JavaScript:

```
Rust (game logic)              JavaScript (rendering + input)
┌────────────────────┐         ┌────────────────────────┐
│ Game state         │         │ Canvas 2D rendering    │
│ Physics/collision  │◀───────▶│ Keyboard/mouse input   │
│ Entity management  │  data   │ requestAnimationFrame  │
│ Score/rules        │         │ Audio playback         │
└────────────────────┘         └────────────────────────┘
```

**Rust owns the state, JS handles I/O.**

## The Game Loop

```js
import init, { Game } from './pkg/game.js';

await init();

const canvas = document.getElementById('game');
const ctx = canvas.getContext('2d');
const game = new Game(canvas.width, canvas.height);

// Track pressed keys
const keys = new Set();
document.addEventListener('keydown', (e) => keys.add(e.key));
document.addEventListener('keyup', (e) => keys.delete(e.key));

function gameLoop() {
    // 1. Input
    const speed = 4;
    if (keys.has('ArrowLeft'))  game.move_player(-speed, 0);
    if (keys.has('ArrowRight')) game.move_player(speed, 0);
    if (keys.has('ArrowUp'))    game.move_player(0, -speed);
    if (keys.has('ArrowDown'))  game.move_player(0, speed);

    // 2. Update (Rust)
    game.update();

    // 3. Render (JS)
    ctx.clearRect(0, 0, canvas.width, canvas.height);

    // Draw player
    ctx.fillStyle = '#654ff0';
    ctx.beginPath();
    ctx.arc(game.player_x(), game.player_y(), game.player_size(), 0, Math.PI * 2);
    ctx.fill();

    // Draw targets
    const positions = game.target_positions();
    ctx.fillStyle = '#ce422b';
    for (let i = 0; i < positions.length; i += 2) {
        ctx.beginPath();
        ctx.arc(positions[i], positions[i + 1], 15, 0, Math.PI * 2);
        ctx.fill();
    }

    // Draw score
    ctx.fillStyle = '#f1f5f9';
    ctx.font = '20px monospace';
    ctx.fillText(`Score: ${game.score()}`, 10, 30);

    requestAnimationFrame(gameLoop);
}

requestAnimationFrame(gameLoop);
```

## Collision Detection

### Circle-circle collision

```rust
fn circles_collide(x1: f64, y1: f64, r1: f64, x2: f64, y2: f64, r2: f64) -> bool {
    let dx = x1 - x2;
    let dy = y1 - y2;
    let dist_sq = dx * dx + dy * dy;
    let radii = r1 + r2;
    dist_sq < radii * radii  // Avoid sqrt for performance
}
```

### AABB (rectangle) collision

```rust
fn aabb_collide(
    x1: f64, y1: f64, w1: f64, h1: f64,
    x2: f64, y2: f64, w2: f64, h2: f64,
) -> bool {
    x1 < x2 + w2 && x1 + w1 > x2 && y1 < y2 + h2 && y1 + h1 > y2
}
```

## Entity Component System (ECS) Pattern

For complex games, use an ECS pattern:

```rust
#[wasm_bindgen]
pub struct World {
    positions: Vec<f64>,    // [x0, y0, x1, y1, ...]
    velocities: Vec<f64>,   // [vx0, vy0, vx1, vy1, ...]
    sizes: Vec<f64>,        // [r0, r1, ...]
    alive: Vec<bool>,
    count: usize,
}

#[wasm_bindgen]
impl World {
    pub fn update(&mut self, dt: f64) {
        for i in 0..self.count {
            if !self.alive[i] { continue; }
            let idx = i * 2;
            self.positions[idx] += self.velocities[idx] * dt;
            self.positions[idx + 1] += self.velocities[idx + 1] * dt;
        }
    }

    pub fn positions(&self) -> Vec<f64> {
        self.positions.clone()
    }
}
```

## Performance Tips for Games

| Tip | Why |
|-----|-----|
| Use flat arrays (`Vec<f64>`) for bulk data | Minimizes Wasm↔JS boundary crossing |
| Avoid allocating per frame | Reuse buffers with `.clear()` |
| Use `f64` not `f32` in JS-facing code | JS numbers are f64 |
| Check collisions with spatial partitioning | Grid or quadtree for O(n) instead of O(n²) |
| Keep game logic in Rust, rendering in JS | Canvas API isn't accessible from Wasm |

## Try It

The starter code implements a simple collectible game — a player that moves and collects targets. In a real project, you'd wire this to keyboard input and a Canvas rendering loop.
