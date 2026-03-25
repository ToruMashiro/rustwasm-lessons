---
title: Wasm + Frontend Frameworks
slug: framework-integration
difficulty: advanced
tags: [dom]
order: 15
description: Integrate Rust/Wasm modules with React, Svelte, and Vue — use Wasm for performance-critical logic while keeping your favorite UI framework.
starter_code: |
  use wasm_bindgen::prelude::*;

  // A reusable Wasm module that works with any framework
  #[wasm_bindgen]
  pub struct ImageProcessor {
      width: u32,
      height: u32,
  }

  #[wasm_bindgen]
  impl ImageProcessor {
      #[wasm_bindgen(constructor)]
      pub fn new(width: u32, height: u32) -> Self {
          Self { width, height }
      }

      pub fn blur(&self, pixels: &mut [u8], radius: u32) {
          let r = radius as i32;
          let w = self.width as i32;
          let h = self.height as i32;
          let mut output = pixels.to_vec();

          for y in 0..h {
              for x in 0..w {
                  let mut sum_r = 0u32;
                  let mut sum_g = 0u32;
                  let mut sum_b = 0u32;
                  let mut count = 0u32;

                  for dy in -r..=r {
                      for dx in -r..=r {
                          let nx = x + dx;
                          let ny = y + dy;
                          if nx >= 0 && nx < w && ny >= 0 && ny < h {
                              let idx = ((ny * w + nx) * 4) as usize;
                              sum_r += pixels[idx] as u32;
                              sum_g += pixels[idx + 1] as u32;
                              sum_b += pixels[idx + 2] as u32;
                              count += 1;
                          }
                      }
                  }

                  let idx = ((y * w + x) * 4) as usize;
                  output[idx] = (sum_r / count) as u8;
                  output[idx + 1] = (sum_g / count) as u8;
                  output[idx + 2] = (sum_b / count) as u8;
              }
          }

          pixels.copy_from_slice(&output);
      }

      pub fn dimensions(&self) -> String {
          format!("{}x{}", self.width, self.height)
      }
  }
expected_output: |
  ImageProcessor created: 800x600
  Blur applied with radius 3
  Processing time: 12ms
---

## The Pattern

Use Wasm for **computation**, your framework for **UI**:

```
┌─────────────────┐
│  React/Svelte   │  ← UI rendering, state, events
│  (JavaScript)   │
├─────────────────┤
│  Wasm Module    │  ← Heavy computation, data processing
│  (Rust)         │
└─────────────────┘
```

## React Integration

```tsx
// hooks/useWasm.ts
import { useState, useEffect } from 'react';

export function useWasm() {
  const [wasm, setWasm] = useState(null);

  useEffect(() => {
    async function load() {
      const module = await import('../pkg/my_wasm.js');
      await module.default();  // init
      setWasm(module);
    }
    load();
  }, []);

  return wasm;
}

// components/ImageEditor.tsx
function ImageEditor() {
  const wasm = useWasm();
  const [processing, setProcessing] = useState(false);

  async function applyBlur() {
    if (!wasm) return;
    setProcessing(true);

    const processor = new wasm.ImageProcessor(800, 600);
    const imageData = ctx.getImageData(0, 0, 800, 600);
    processor.blur(imageData.data, 3);
    ctx.putImageData(imageData, 0, 0);

    setProcessing(false);
  }

  return (
    <button onClick={applyBlur} disabled={!wasm || processing}>
      {processing ? 'Processing...' : 'Apply Blur'}
    </button>
  );
}
```

## Svelte Integration

```svelte
<!-- WasmProcessor.svelte -->
<script lang="ts">
  import { onMount } from 'svelte';

  let wasm: any = null;
  let ready = false;

  onMount(async () => {
    const module = await import('../pkg/my_wasm.js');
    await module.default();
    wasm = module;
    ready = true;
  });

  function processData() {
    if (!wasm) return;
    const result = wasm.process(inputData);
    // update UI with result
  }
</script>

{#if ready}
  <button on:click={processData}>Process</button>
{:else}
  <p>Loading Wasm...</p>
{/if}
```

## Vue Integration

```vue
<!-- WasmComponent.vue -->
<script setup>
import { ref, onMounted } from 'vue';

const wasm = ref(null);
const ready = ref(false);

onMounted(async () => {
  const module = await import('../pkg/my_wasm.js');
  await module.default();
  wasm.value = module;
  ready.value = true;
});

function compute() {
  if (!wasm.value) return;
  const result = wasm.value.heavy_calculation(data);
}
</script>

<template>
  <button @click="compute" :disabled="!ready">Compute</button>
</template>
```

## Vite Configuration

```ts
// vite.config.ts
import { defineConfig } from 'vite';

export default defineConfig({
  optimizeDeps: {
    exclude: ['my-wasm-app']  // Don't pre-bundle Wasm
  },
  server: {
    headers: {
      // Required for SharedArrayBuffer (optional)
      'Cross-Origin-Opener-Policy': 'same-origin',
      'Cross-Origin-Embedder-Policy': 'require-corp',
    }
  }
});
```

## When to Use Wasm with a Framework

| Use Wasm for | Keep in JS/Framework |
|-------------|---------------------|
| Image/video processing | UI components, routing |
| Physics simulation | State management (Redux, etc.) |
| Cryptography | Form handling, validation |
| Data parsing (CSV, binary) | DOM manipulation |
| Audio processing | Event handling |
| Compression/decompression | HTTP requests (unless perf-critical) |

**Rule of thumb**: If it's CPU-bound and takes >10ms in JavaScript, move it to Wasm.

## Try It

The starter code shows an ImageProcessor struct that works with any framework — it accepts raw pixel data and processes it in Rust.
