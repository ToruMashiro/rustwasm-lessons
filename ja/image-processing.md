---
title: 画像処理
slug: image-processing
difficulty: intermediate
tags: [graphics]
order: 20
description: Rust/Wasmを使ってクライアントサイドで画像を処理します — グレースケール、ぼかし、明るさ、コントラスト、ピクセル操作をネイティブに近い速度で実行します。
starter_code: |
  use wasm_bindgen::prelude::*;

  /// 画像をグレースケールに変換
  /// pixels: RGBA形式 [r, g, b, a, r, g, b, a, ...]
  #[wasm_bindgen]
  pub fn grayscale(pixels: &mut [u8]) {
      for chunk in pixels.chunks_exact_mut(4) {
          let gray = (0.299 * chunk[0] as f64
                    + 0.587 * chunk[1] as f64
                    + 0.114 * chunk[2] as f64) as u8;
          chunk[0] = gray;
          chunk[1] = gray;
          chunk[2] = gray;
          // chunk[3] = alpha（変更なし）
      }
  }

  /// 明るさを調整（-255から255）
  #[wasm_bindgen]
  pub fn brightness(pixels: &mut [u8], amount: i32) {
      for chunk in pixels.chunks_exact_mut(4) {
          for i in 0..3 {
              let val = chunk[i] as i32 + amount;
              chunk[i] = val.clamp(0, 255) as u8;
          }
      }
  }

  /// 色を反転
  #[wasm_bindgen]
  pub fn invert(pixels: &mut [u8]) {
      for chunk in pixels.chunks_exact_mut(4) {
          chunk[0] = 255 - chunk[0];
          chunk[1] = 255 - chunk[1];
          chunk[2] = 255 - chunk[2];
      }
  }

  /// セピアトーンフィルター
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
  グレースケール: 3,145,728ピクセルを処理（1024x768）
  明るさ+50: 2msで適用
  反転: 1msで適用
  セピア: 3msで適用
---

## はじめに

画像処理はWasmの最も強力なユースケースの1つです。ピクセル単位の操作はCPU集約的であり、Rust/WasmはこれらのJavaScript処理より5〜10倍高速です。すべての処理はクライアントサイドで行われるため、画像がユーザーのブラウザから外に出ることはありません。

## 仕組み

```
┌───────────┐    getImageData()   ┌──────────┐   putImageData()   ┌───────────┐
│  <canvas>  │ ────────────────▶ │ Rust/Wasm │ ────────────────▶ │  <canvas>  │
│  （ソース） │    Uint8Array     │ （処理）   │    Uint8Array     │  （結果）   │
└───────────┘                    └──────────┘                    └───────────┘
```

1. 画像をCanvasに読み込む
2. ピクセルデータを `Uint8Array`（RGBA形式）として取得
3. Rustに渡してインプレースで変更
4. 変更されたピクセルをCanvasに戻す

## JavaScriptとの統合

```js
import init, { grayscale, brightness, sepia } from './pkg/image_wasm.js';

await init();

const canvas = document.getElementById('canvas');
const ctx = canvas.getContext('2d');

// 画像を読み込む
const img = new Image();
img.onload = () => {
    canvas.width = img.width;
    canvas.height = img.height;
    ctx.drawImage(img, 0, 0);
};
img.src = 'photo.jpg';

// フィルターを適用
function applyFilter(filterFn) {
    const imageData = ctx.getImageData(0, 0, canvas.width, canvas.height);
    filterFn(imageData.data);  // インプレースで変更！
    ctx.putImageData(imageData, 0, 0);
}

// 使用例
applyFilter(grayscale);
applyFilter((pixels) => brightness(pixels, 30));
applyFilter(sepia);
```

## ピクセル形式

Canvasの `ImageData` はRGBA形式のフラットな `Uint8Array` です：

```
Index:  0    1    2    3    4    5    6    7    ...
Data:  [R₀] [G₀] [B₀] [A₀] [R₁] [G₁] [B₁] [A₁] ...
        └── pixel 0 ──┘     └── pixel 1 ──┘
```

Rustでは `chunks_exact_mut(4)` を使って4バイトずつ処理します。

## ボックスブラー

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

## パフォーマンス比較

| フィルター | JavaScript | Rust/Wasm | 高速化 |
|--------|-----------|----------|---------|
| グレースケール（4K画像） | 45ms | 5ms | 9倍 |
| 明るさ（4K） | 40ms | 4ms | 10倍 |
| ボックスブラー r=3（4K） | 2,500ms | 280ms | 9倍 |
| セピア（4K） | 50ms | 6ms | 8倍 |

## 重要な最適化：インプレース変更

Rustは `Uint8Array` を**インプレース**で変更します。データのコピーは不要です：

```rust
pub fn grayscale(pixels: &mut [u8]) {  // &mut = インプレースで変更
    // pixelsはCanvasのImageDataバッファそのもの
    // 変更はJS側に即座に反映される
}
```

これは最も高速なアプローチです。アロケーションもコピーもゼロです。

## 試してみよう

スターターコードには4つの画像フィルターが含まれています。実際のプロジェクトでは、Canvasの `imageData.data` をこれらの関数に直接渡して、結果を即座に確認できます。
