---
title: State Management
slug: state-management
difficulty: intermediate
tags: [data-structures]
order: 12
description: Share and synchronize state between Rust/Wasm and JavaScript using structs, shared memory, and message passing patterns.
starter_code: |
  use wasm_bindgen::prelude::*;

  #[wasm_bindgen]
  pub struct AppState {
      items: Vec<String>,
      counter: u32,
      modified: bool,
  }

  #[wasm_bindgen]
  impl AppState {
      #[wasm_bindgen(constructor)]
      pub fn new() -> Self {
          Self {
              items: Vec::new(),
              counter: 0,
              modified: false,
          }
      }

      pub fn add_item(&mut self, item: &str) {
          self.items.push(item.to_string());
          self.counter += 1;
          self.modified = true;
      }

      pub fn remove_item(&mut self, index: usize) -> bool {
          if index < self.items.len() {
              self.items.remove(index);
              self.modified = true;
              true
          } else {
              false
          }
      }

      pub fn get_items(&self) -> String {
          self.items.join(",")
      }

      pub fn count(&self) -> u32 { self.counter }

      pub fn is_modified(&self) -> bool { self.modified }

      pub fn clear_modified(&mut self) { self.modified = false; }
  }
expected_output: |
  State created
  Added: "apple" → count: 1
  Added: "banana" → count: 2
  Items: apple,banana
  Modified: true
---

## The Challenge

In Rust/Wasm apps, state lives in two places:
- **Rust** (Wasm linear memory) — game state, data models, computation results
- **JavaScript** — DOM state, UI state, browser APIs

You need patterns to keep them in sync without excessive copying.

## Pattern 1: Rust Owns State (Recommended)

The simplest and most performant pattern — Rust owns all application state, JavaScript only reads it:

```rust
#[wasm_bindgen]
pub struct GameState {
    score: u32,
    level: u32,
    entities: Vec<Entity>,
}

#[wasm_bindgen]
impl GameState {
    // JS calls methods to mutate state
    pub fn update(&mut self, dt: f64) { /* physics */ }
    pub fn add_entity(&mut self, x: f64, y: f64) { /* ... */ }

    // JS reads state via getters
    pub fn score(&self) -> u32 { self.score }
    pub fn level(&self) -> u32 { self.level }
    pub fn entity_positions(&self) -> Vec<f64> { /* flat array */ }
}
```

```js
const state = new GameState();

function gameLoop() {
    state.update(16.67); // mutate in Rust
    render(state.entity_positions()); // read from JS
    requestAnimationFrame(gameLoop);
}
```

## Pattern 2: Message Passing

For complex apps, use a command pattern — JS sends actions, Rust processes them:

```rust
#[wasm_bindgen]
pub enum Action {
    AddItem,
    RemoveItem,
    UpdateScore,
    Reset,
}

#[wasm_bindgen]
impl AppState {
    pub fn dispatch(&mut self, action: Action, payload: &str) -> String {
        match action {
            Action::AddItem => {
                self.items.push(payload.to_string());
                format!("Added: {}", payload)
            }
            Action::RemoveItem => {
                // parse index from payload
                "Removed".to_string()
            }
            Action::Reset => {
                self.items.clear();
                "Reset".to_string()
            }
            _ => "Unknown action".to_string(),
        }
    }
}
```

## Pattern 3: Shared Memory (Advanced)

For bulk data (game worlds, image buffers), share memory directly:

```rust
#[wasm_bindgen]
impl World {
    /// Returns a pointer to the pixel buffer in Wasm memory
    pub fn pixels_ptr(&self) -> *const u8 {
        self.pixels.as_ptr()
    }

    pub fn pixels_len(&self) -> usize {
        self.pixels.len()
    }
}
```

```js
// JS reads directly from Wasm memory — zero copy!
const ptr = world.pixels_ptr();
const len = world.pixels_len();
const pixels = new Uint8Array(wasm.memory.buffer, ptr, len);

// Draw to canvas
const imageData = new ImageData(
    new Uint8ClampedArray(pixels.buffer, pixels.byteOffset, pixels.byteLength),
    width, height
);
ctx.putImageData(imageData, 0, 0);
```

## Anti-Patterns to Avoid

| Don't | Why | Do Instead |
|-------|-----|-----------|
| Serialize to JSON every frame | Slow, allocates strings | Return flat arrays or use shared memory |
| Clone entire state to JS | Expensive copy | Expose getters for what JS needs |
| Let JS modify Rust memory directly | Unsafe, can corrupt state | Use methods with validation |
| Store DOM references in Rust | Complex lifetime issues | Let JS handle DOM, Rust handles data |

## Try It

The starter code shows a state struct with add/remove/query methods. This "Rust owns state" pattern works for most applications.
