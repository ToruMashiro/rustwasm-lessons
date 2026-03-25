---
title: Wasm + フロントエンドフレームワーク
slug: framework-integration
difficulty: advanced
tags: [dom]
order: 15
description: Rust/WasmモジュールをReact、Svelte、Vueと統合します — お気に入りのUIフレームワークを使いながら、パフォーマンスが重要なロジックにWasmを活用します。
starter_code: |
  use wasm_bindgen::prelude::*;

  // 任意のフレームワークで使える再利用可能なWasmモジュール
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
  ImageProcessorを作成: 800x600
  半径3でブラーを適用
  処理時間: 12ms
---

## パターン

Wasmは**計算処理**に、フレームワークは**UI**に使います：

```
┌─────────────────┐
│  React/Svelte   │  ← UIレンダリング、状態、イベント
│  (JavaScript)   │
├─────────────────┤
│  Wasmモジュール  │  ← 重い計算処理、データ処理
│  (Rust)         │
└─────────────────┘
```

## React統合

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

## Svelte統合

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
    // 結果でUIを更新
  }
</script>

{#if ready}
  <button on:click={processData}>Process</button>
{:else}
  <p>Wasmを読み込み中...</p>
{/if}
```

## Vue統合

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

## Vite設定

```ts
// vite.config.ts
import { defineConfig } from 'vite';

export default defineConfig({
  optimizeDeps: {
    exclude: ['my-wasm-app']  // Wasmをプリバンドルしない
  },
  server: {
    headers: {
      // SharedArrayBufferに必要（オプション）
      'Cross-Origin-Opener-Policy': 'same-origin',
      'Cross-Origin-Embedder-Policy': 'require-corp',
    }
  }
});
```

## Wasmをフレームワークと使うべき場面

| Wasmに適した処理 | JS/フレームワークに残すべき処理 |
|-----------------|-------------------------------|
| 画像・動画処理 | UIコンポーネント、ルーティング |
| 物理シミュレーション | 状態管理（Reduxなど） |
| 暗号処理 | フォーム処理、バリデーション |
| データパース（CSV、バイナリ） | DOM操作 |
| オーディオ処理 | イベントハンドリング |
| 圧縮・解凍 | HTTPリクエスト（パフォーマンスが重要でない場合） |

**経験則**: CPU集約的でJavaScriptで10ms以上かかる処理は、Wasmに移行しましょう。

## 試してみよう

スターターコードは、任意のフレームワークで動作するImageProcessor構造体を示しています。生のピクセルデータを受け取り、Rustで処理します。
