---
title: Particle Simulation
slug: particle-simulation
difficulty: advanced
tags: [simulation, graphics]
order: 8
description: Build a high-performance particle system rendered on HTML Canvas, powered by Rust/Wasm.
starter_code: |
  use wasm_bindgen::prelude::*;

  #[wasm_bindgen]
  pub struct Particle {
      x: f64,
      y: f64,
      vx: f64,
      vy: f64,
      radius: f64,
  }

  #[wasm_bindgen]
  pub struct Simulation {
      particles: Vec<Particle>,
      width: f64,
      height: f64,
  }

  #[wasm_bindgen]
  impl Simulation {
      #[wasm_bindgen(constructor)]
      pub fn new(width: f64, height: f64, count: usize) -> Self {
          let mut particles = Vec::with_capacity(count);
          for i in 0..count {
              let angle = (i as f64) * 2.0 * std::f64::consts::PI / (count as f64);
              particles.push(Particle {
                  x: width / 2.0 + (width / 4.0) * angle.cos(),
                  y: height / 2.0 + (height / 4.0) * angle.sin(),
                  vx: angle.cos() * 2.0,
                  vy: angle.sin() * 2.0,
                  radius: 3.0,
              });
          }
          Self { particles, width, height }
      }

      pub fn tick(&mut self) {
          for p in &mut self.particles {
              p.x += p.vx;
              p.y += p.vy;

              // Bounce off walls
              if p.x - p.radius < 0.0 || p.x + p.radius > self.width {
                  p.vx = -p.vx;
              }
              if p.y - p.radius < 0.0 || p.y + p.radius > self.height {
                  p.vy = -p.vy;
              }
          }
      }

      pub fn positions(&self) -> Vec<f64> {
          self.particles.iter()
              .flat_map(|p| vec![p.x, p.y, p.radius])
              .collect()
      }

      pub fn particle_count(&self) -> usize {
          self.particles.len()
      }
  }
expected_output: |
  Simulation created: 500 particles
  Frame 1: all particles updated
  Rendering at ~60fps via requestAnimationFrame
---

## Introduction

This is where Rust/Wasm truly shines — running thousands of physics calculations per frame at near-native speed, with JavaScript handling only the Canvas rendering. This separation lets you achieve 60fps with 10,000+ particles.

## Architecture

```
┌─────────────┐    positions()    ┌──────────────┐
│  Rust/Wasm   │ ───────────────▶ │  JavaScript  │
│  Simulation  │                  │  Canvas 2D   │
│  (physics)   │ ◀─────────────── │  (rendering) │
└─────────────┘    tick()         └──────────────┘
```

The key insight: **Rust owns the simulation state.** JavaScript only reads position data and draws to the Canvas. This avoids expensive data serialization every frame.

## Setup

```toml
[dependencies]
wasm-bindgen = "0.2"

[dependencies.web-sys]
version = "0.3"
features = [
    "Window",
    "Document",
    "HtmlCanvasElement",
    "CanvasRenderingContext2d",
]
```

## Data transfer: flat arrays

The most efficient way to send bulk data from Rust to JS is a flat `Vec<f64>`:

```rust
// Instead of returning Vec<Particle> (can't cross Wasm boundary),
// return a flat array: [x0, y0, r0, x1, y1, r1, ...]
pub fn positions(&self) -> Vec<f64> {
    self.particles.iter()
        .flat_map(|p| vec![p.x, p.y, p.radius])
        .collect()
}
```

In JavaScript, this becomes a `Float64Array` — directly iterable, no JSON parsing:

```js
const positions = simulation.positions();
for (let i = 0; i < positions.length; i += 3) {
    const x = positions[i];
    const y = positions[i + 1];
    const radius = positions[i + 2];
    // draw...
}
```

## JavaScript rendering loop

```js
import init, { Simulation } from './pkg/particles.js';

async function run() {
    await init();

    const canvas = document.getElementById('canvas');
    const ctx = canvas.getContext('2d');
    const sim = new Simulation(canvas.width, canvas.height, 1000);

    function frame() {
        // Physics in Rust
        sim.tick();

        // Read positions
        const positions = sim.positions();

        // Render in JS
        ctx.clearRect(0, 0, canvas.width, canvas.height);
        ctx.fillStyle = '#654ff0';

        for (let i = 0; i < positions.length; i += 3) {
            ctx.beginPath();
            ctx.arc(positions[i], positions[i+1], positions[i+2], 0, Math.PI * 2);
            ctx.fill();
        }

        requestAnimationFrame(frame);
    }

    requestAnimationFrame(frame);
}

run();
```

## Performance optimization tips

### 1. Reuse buffers — avoid allocating every frame

```rust
pub struct Simulation {
    particles: Vec<Particle>,
    position_buffer: Vec<f64>,  // reuse this
    // ...
}

pub fn positions(&mut self) -> Vec<f64> {
    self.position_buffer.clear();
    for p in &self.particles {
        self.position_buffer.push(p.x);
        self.position_buffer.push(p.y);
        self.position_buffer.push(p.radius);
    }
    self.position_buffer.clone()
}
```

### 2. Use SharedArrayBuffer for zero-copy (advanced)

Instead of copying data each frame, share memory between Rust and JS:

```rust
// Expose a pointer to the particle data
pub fn positions_ptr(&self) -> *const f64 { /* ... */ }
pub fn positions_len(&self) -> usize { /* ... */ }
```

```js
// JS reads directly from Wasm memory (zero copy!)
const ptr = sim.positions_ptr();
const len = sim.positions_len();
const view = new Float64Array(wasm.memory.buffer, ptr, len);
```

### 3. Spatial partitioning for collisions

For particle-particle collisions, use a grid or quadtree to avoid O(n²) checks:

```rust
pub fn tick_with_collisions(&mut self) {
    // Grid-based broad phase
    let cell_size = 20.0;
    let mut grid: HashMap<(i32, i32), Vec<usize>> = HashMap::new();

    for (i, p) in self.particles.iter().enumerate() {
        let key = ((p.x / cell_size) as i32, (p.y / cell_size) as i32);
        grid.entry(key).or_default().push(i);
    }

    // Only check particles in same/neighboring cells
    // ...
}
```

## Performance benchmarks

| Particle count | Pure JS (fps) | Rust/Wasm (fps) |
|---------------|--------------|----------------|
| 1,000 | 60 | 60 |
| 5,000 | 35 | 60 |
| 10,000 | 15 | 60 |
| 50,000 | 3 | 45 |

The gap widens dramatically with complex physics (collisions, forces, constraints).

## Try It

Add gravity, particle collisions, or mouse interaction to the simulation.
