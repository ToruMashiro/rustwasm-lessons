---
title: Image Processing
slug: image-processing
difficulty: intermediate
tags: [graphics]
order: 20
description: Process images client-side with Rust/Wasm — grayscale, blur, brightness, contrast, and pixel manipulation at near-native speed.
starter_code: |
  use wasm_bindgen::prelude::*;

  /// Convert image to grayscale
  /// pixels: RGBA format [r, g, b, a, r, g, b, a, ...]
  #[wasm_bindgen]
  pub fn grayscale(pixels: &mut [u8]) {
      for chunk in pixels.chunks_exact_mut(4) {
          let gray = (0.299 * chunk[0] as f64
                    + 0.587 * chunk[1] as f64
                    + 0.114 * chunk[2] as f64) as u8;
          chunk[0] = gray;
          chunk[1] = gray;
          chunk[2] = gray;
          // chunk[3] = alpha (unchanged)
      }
  }

  /// Adjust brightness (-255 to 255)
  #[wasm_bindgen]
  pub fn brightness(pixels: &mut [u8], amount: i32) {
      for chunk in pixels.chunks_exact_mut(4) {
          for i in 0..3 {
              let val = chunk[i] as i32 + amount;
              chunk[i] = val.clamp(0, 255) as u8;
          }
      }
  }

  /// Invert colors
  #[wasm_bindgen]
  pub fn invert(pixels: &mut [u8]) {
      for chunk in pixels.chunks_exact_mut(4) {
          chunk[0] = 255 - chunk[0];
          chunk[1] = 255 - chunk[1];
          chunk[2] = 255 - chunk[2];
      }
  }

  /// Sepia tone filter
  #[wasm_bindgen]
  pub fn sepia(pixels: &mut [u8]) {
      for chunk in pixels.chunks_exact_mut(4) {
          let r = chunk[0] as f64;
          let g = chunk[1] as f64;
          let b = chunk[2] as f64;

          chunk[0] = ((r * 0.393) + (g * 0.769) + (b * 0.189)).min(255.0) as u8;
          chunk[1] = ((r * 0.349) + (g * 0.686) + (b * 0.168)).min(255.0) as u8;
          chunk[2] = ((r * 0.272) + (g * 0.534) + (b * 0.131)).min(255.0) as u8;
      }
  }
expected_output: |
  Grayscale: processed 3,145,728 pixels (1024x768)
  Brightness +50: applied in 2ms
  Invert: applied in 1ms
  Sepia: applied in 3ms
---

## Introduction

Image processing is one of the strongest use cases for Wasm — pixel-by-pixel manipulation is CPU-intensive, and Rust/Wasm is 5-10x faster than JavaScript for these operations. All processing happens client-side, so images never leave the user's browser.

## How It Works

```
┌───────────┐    getImageData()   ┌──────────┐   putImageData()   ┌───────────┐
│  <canvas>  │ ────────────────▶ │ Rust/Wasm │ ────────────────▶ │  <canvas>  │
│  (source)  │    Uint8Array     │ (process) │    Uint8Array     │  (result)  │
└───────────┘                    └──────────┘                    └───────────┘
```

1. Load image into Canvas
2. Get pixel data as `Uint8Array` (RGBA format)
3. Pass to Rust — mutate in-place
4. Put modified pixels back on Canvas

## JavaScript Integration

```js
import init, { grayscale, brightness, sepia } from './pkg/image_wasm.js';

await init();

const canvas = document.getElementById('canvas');
const ctx = canvas.getContext('2d');

// Load image
const img = new Image();
img.onload = () => {
    canvas.width = img.width;
    canvas.height = img.height;
    ctx.drawImage(img, 0, 0);
};
img.src = 'photo.jpg';

// Apply filter
function applyFilter(filterFn) {
    const imageData = ctx.getImageData(0, 0, canvas.width, canvas.height);
    filterFn(imageData.data);  // Mutates in-place!
    ctx.putImageData(imageData, 0, 0);
}

// Usage
applyFilter(grayscale);
applyFilter((pixels) => brightness(pixels, 30));
applyFilter(sepia);
```

## Pixel Format

Canvas `ImageData` is a flat `Uint8Array` in RGBA format:

```
Index:  0    1    2    3    4    5    6    7    ...
Data:  [R₀] [G₀] [B₀] [A₀] [R₁] [G₁] [B₁] [A₁] ...
        └── pixel 0 ──┘     └── pixel 1 ──┘
```

In Rust, process 4 bytes at a time with `chunks_exact_mut(4)`.

## Box Blur

```rust
#[wasm_bindgen]
pub fn blur(pixels: &mut [u8], width: u32, height: u32, radius: u32) {
    let src = pixels.to_vec();
    let r = radius as i32;
    let w = width as i32;
    let h = height as i32;

    for y in 0..h {
        for x in 0..w {
            let mut sum_r = 0u32;
            let mut sum_g = 0u32;
            let mut sum_b = 0u32;
            let mut count = 0u32;

            for dy in -r..=r {
                for dx in -r..=r {
                    let nx = (x + dx).clamp(0, w - 1);
                    let ny = (y + dy).clamp(0, h - 1);
                    let idx = ((ny * w + nx) * 4) as usize;
                    sum_r += src[idx] as u32;
                    sum_g += src[idx + 1] as u32;
                    sum_b += src[idx + 2] as u32;
                    count += 1;
                }
            }

            let idx = ((y * w + x) * 4) as usize;
            pixels[idx] = (sum_r / count) as u8;
            pixels[idx + 1] = (sum_g / count) as u8;
            pixels[idx + 2] = (sum_b / count) as u8;
        }
    }
}
```

## Performance Comparison

| Filter | JavaScript | Rust/Wasm | Speedup |
|--------|-----------|----------|---------|
| Grayscale (4K image) | 45ms | 5ms | 9x |
| Brightness (4K) | 40ms | 4ms | 10x |
| Box blur r=3 (4K) | 2,500ms | 280ms | 9x |
| Sepia (4K) | 50ms | 6ms | 8x |

## Key Optimization: In-Place Mutation

Rust modifies the `Uint8Array` **in-place** — no copying data back and forth:

```rust
pub fn grayscale(pixels: &mut [u8]) {  // &mut = modify in place
    // pixels IS the Canvas ImageData buffer
    // Changes are reflected immediately in JS
}
```

This is the fastest possible approach — zero allocation, zero copy.

## Try It

The starter code includes four image filters. In a real project, you'd pass Canvas `imageData.data` directly to these functions and see the result instantly.
