---
title: Canvas & 2D Graphics
slug: canvas-graphics
difficulty: intermediate
tags: [graphics, dom]
order: 11
description: Draw shapes, animate sprites, and build interactive graphics using HTML Canvas controlled from Rust/Wasm.
starter_code: |
  use wasm_bindgen::prelude::*;
  use web_sys::{CanvasRenderingContext2d, HtmlCanvasElement, window};

  #[wasm_bindgen]
  pub struct Canvas {
      ctx: CanvasRenderingContext2d,
      width: f64,
      height: f64,
  }

  #[wasm_bindgen]
  impl Canvas {
      #[wasm_bindgen(constructor)]
      pub fn new(canvas_id: &str) -> Result<Canvas, JsValue> {
          let document = window().unwrap().document().unwrap();
          let canvas = document
              .get_element_by_id(canvas_id)
              .unwrap()
              .dyn_into::<HtmlCanvasElement>()?;

          let ctx = canvas
              .get_context("2d")?
              .unwrap()
              .dyn_into::<CanvasRenderingContext2d>()?;

          let width = canvas.width() as f64;
          let height = canvas.height() as f64;

          Ok(Canvas { ctx, width, height })
      }

      pub fn clear(&self) {
          self.ctx.clear_rect(0.0, 0.0, self.width, self.height);
      }

      pub fn draw_circle(&self, x: f64, y: f64, radius: f64, color: &str) {
          self.ctx.begin_path();
          self.ctx.arc(x, y, radius, 0.0, std::f64::consts::PI * 2.0).unwrap();
          self.ctx.set_fill_style_str(color);
          self.ctx.fill();
      }

      pub fn draw_rect(&self, x: f64, y: f64, w: f64, h: f64, color: &str) {
          self.ctx.set_fill_style_str(color);
          self.ctx.fill_rect(x, y, w, h);
      }
  }
expected_output: |
  Canvas initialized: 800x600
  Drawing circle at (400, 300) r=50
  Drawing rectangle at (100, 100) 200x150
  Frame rendered successfully
---

## Introduction

HTML Canvas is one of the best use cases for Rust/Wasm — Rust handles the computation (physics, game logic, procedural generation), while the Canvas API handles rendering. This split gives you near-native performance for the heavy parts.

## Setup

```toml
[dependencies.web-sys]
version = "0.3"
features = [
    "Window",
    "Document",
    "HtmlCanvasElement",
    "CanvasRenderingContext2d",
]
```

## Getting the Canvas Context

```rust
use web_sys::{CanvasRenderingContext2d, HtmlCanvasElement};

fn get_context(canvas_id: &str) -> Result<CanvasRenderingContext2d, JsValue> {
    let document = web_sys::window().unwrap().document().unwrap();
    let canvas = document
        .get_element_by_id(canvas_id)
        .unwrap()
        .dyn_into::<HtmlCanvasElement>()?;

    canvas
        .get_context("2d")?
        .unwrap()
        .dyn_into::<CanvasRenderingContext2d>()
}
```

## Drawing Primitives

```rust
// Rectangle
ctx.set_fill_style_str("#654ff0");
ctx.fill_rect(10.0, 10.0, 100.0, 50.0);

// Circle
ctx.begin_path();
ctx.arc(200.0, 200.0, 40.0, 0.0, std::f64::consts::PI * 2.0)?;
ctx.set_fill_style_str("#ce422b");
ctx.fill();

// Line
ctx.begin_path();
ctx.move_to(0.0, 0.0);
ctx.line_to(300.0, 150.0);
ctx.set_stroke_style_str("#34d399");
ctx.set_line_width(2.0);
ctx.stroke();

// Text
ctx.set_font("24px Inter");
ctx.set_fill_style_str("#f1f5f9");
ctx.fill_text("Hello Wasm!", 50.0, 50.0)?;
```

## Animation Loop

The animation runs in JavaScript using `requestAnimationFrame`, calling Rust for each frame:

```js
import init, { Canvas } from './pkg/my_app.js';

async function run() {
    await init();
    const canvas = new Canvas("game-canvas");

    let x = 0;
    function frame() {
        canvas.clear();
        canvas.draw_circle(x, 300, 20, "#654ff0");
        x = (x + 2) % 800;
        requestAnimationFrame(frame);
    }
    requestAnimationFrame(frame);
}
run();
```

## Mouse Input

```rust
use web_sys::MouseEvent;
use wasm_bindgen::closure::Closure;

pub fn setup_mouse(canvas: &HtmlCanvasElement) -> Result<(), JsValue> {
    let closure = Closure::wrap(Box::new(move |event: MouseEvent| {
        let x = event.offset_x() as f64;
        let y = event.offset_y() as f64;
        // Handle click at (x, y)
    }) as Box<dyn FnMut(_)>);

    canvas.add_event_listener_with_callback(
        "click",
        closure.as_ref().unchecked_ref(),
    )?;
    closure.forget();
    Ok(())
}
```

## Image Drawing

```rust
use web_sys::HtmlImageElement;

pub fn draw_image(ctx: &CanvasRenderingContext2d, src: &str, x: f64, y: f64)
    -> Result<(), JsValue>
{
    let img = HtmlImageElement::new()?;
    img.set_src(src);

    let ctx = ctx.clone();
    let closure = Closure::once(move || {
        ctx.draw_image_with_html_image_element(&img, x, y).unwrap();
    });
    img.set_onload(Some(closure.as_ref().unchecked_ref()));
    closure.forget();
    Ok(())
}
```

## Performance: Canvas vs WebGL

| Feature | Canvas 2D | WebGL |
|---------|----------|-------|
| Setup complexity | Low | High |
| Draw calls | ~1,000/frame | ~10,000/frame |
| Shaders | No | Yes |
| 3D | No | Yes |
| Best for | 2D games, charts, UI | 3D, heavy particle systems |

For most 2D Rust/Wasm projects, Canvas 2D is sufficient and much simpler.

## Try It

The starter code shows a Canvas wrapper struct with basic drawing methods. In a real project, you'd call these from a JavaScript animation loop.
