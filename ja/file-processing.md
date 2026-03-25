---
title: ファイル処理
slug: file-processing
difficulty: intermediate
tags: [api]
order: 13
description: Rust/Wasmを使ってクライアントサイドでファイルの読み込み、処理、生成を行います — CSVパース、画像操作、ファイルダウンロード。
starter_code: |
  use wasm_bindgen::prelude::*;

  #[wasm_bindgen]
  pub fn process_csv(input: &str) -> String {
      let mut rows: Vec<Vec<&str>> = Vec::new();

      for line in input.lines() {
          if line.trim().is_empty() { continue; }
          let cols: Vec<&str> = line.split(',').map(|s| s.trim()).collect();
          rows.push(cols);
      }

      // 列幅を計算
      let num_cols = rows.first().map(|r| r.len()).unwrap_or(0);
      let mut widths = vec![0usize; num_cols];
      for row in &rows {
          for (i, col) in row.iter().enumerate() {
              if i < num_cols {
                  widths[i] = widths[i].max(col.len());
              }
          }
      }

      // 整列されたテーブルとしてフォーマット
      let mut output = String::new();
      for (ri, row) in rows.iter().enumerate() {
          for (i, col) in row.iter().enumerate() {
              if i > 0 { output.push_str(" | "); }
              output.push_str(&format!("{:<width$}", col, width = widths[i]));
          }
          output.push('\n');
          if ri == 0 {
              for (i, w) in widths.iter().enumerate() {
                  if i > 0 { output.push_str("-+-"); }
                  output.push_str(&"-".repeat(*w));
              }
              output.push('\n');
          }
      }

      output
  }
expected_output: |
  Name   | Age | City
  -------+-----+---------
  Alice  | 30  | Tokyo
  Bob    | 25  | New York
  Charlie| 35  | London
---

## はじめに

ファイル処理は、Wasmの優れたユースケースです。CPU集約的なパースや変換はRustでより高速に実行でき、すべてがクライアントサイドで完結します（サーバーへのアップロードは不要です）。

## JavaScriptからのファイル読み込み

ユーザーは`<input type="file">`でファイルを選択し、バイトデータをRustに渡します：

```js
const input = document.getElementById('file-input');
input.addEventListener('change', async (e) => {
    const file = e.target.files[0];
    const buffer = await file.arrayBuffer();
    const bytes = new Uint8Array(buffer);

    // Rustに渡す
    const result = process_file(bytes);
});
```

Rust側では`&[u8]`として受け取ります：

```rust
#[wasm_bindgen]
pub fn process_file(data: &[u8]) -> String {
    // dataはJSからのUint8Array
    format!("Received {} bytes", data.len())
}
```

## CSV処理

```rust
#[wasm_bindgen]
pub fn parse_csv(input: &str) -> JsValue {
    let mut records = Vec::new();

    for line in input.lines().skip(1) {  // ヘッダーをスキップ
        let fields: Vec<&str> = line.split(',').collect();
        records.push(fields);
    }

    // JS配列に変換
    serde_wasm_bindgen::to_value(&records).unwrap()
}
```

## 画像処理

Rustで画像のピクセルを高速に処理します：

```rust
#[wasm_bindgen]
pub fn grayscale(pixels: &mut [u8]) {
    // pixelsはRGBA形式: [r, g, b, a, r, g, b, a, ...]
    for chunk in pixels.chunks_exact_mut(4) {
        let gray = (0.299 * chunk[0] as f64
                  + 0.587 * chunk[1] as f64
                  + 0.114 * chunk[2] as f64) as u8;
        chunk[0] = gray;  // R
        chunk[1] = gray;  // G
        chunk[2] = gray;  // B
        // chunk[3] = アルファ値（変更なし）
    }
}
```

```js
// Canvasから画像データを取得
const imageData = ctx.getImageData(0, 0, width, height);
grayscale(imageData.data);  // インプレースで変更！
ctx.putImageData(imageData, 0, 0);
```

## ファイルダウンロードの生成

Rustでファイルを生成し、JavaScriptからダウンロードをトリガーします：

```rust
#[wasm_bindgen]
pub fn generate_report() -> Vec<u8> {
    let csv = "name,score\nAlice,95\nBob,87\n";
    csv.as_bytes().to_vec()
}
```

```js
const data = generate_report();
const blob = new Blob([data], { type: 'text/csv' });
const url = URL.createObjectURL(blob);

const a = document.createElement('a');
a.href = url;
a.download = 'report.csv';
a.click();
URL.revokeObjectURL(url);
```

## パフォーマンス比較：Rust vs JavaScript

| 操作 | JS | Rust/Wasm |
|------|-----|----------|
| 10MB CSVのパース | 約800ms | 約150ms |
| 画像グレースケール（4K） | 約50ms | 約8ms |
| JSONパース（1MB） | 約15ms | 約5ms |

データ集約的な操作では、Rust/Wasmが圧倒的に優れています。

## 試してみよう

**Run**をクリックして、CSVフォーマッターの動作を確認しましょう。カンマ区切りの入力をパースし、整列されたテーブルとして出力します。
