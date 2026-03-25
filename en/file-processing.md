---
title: File Processing
slug: file-processing
difficulty: intermediate
tags: [api]
order: 13
description: Read, process, and generate files client-side using Rust/Wasm — CSV parsing, image manipulation, and file downloads.
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

      // Calculate column widths
      let num_cols = rows.first().map(|r| r.len()).unwrap_or(0);
      let mut widths = vec![0usize; num_cols];
      for row in &rows {
          for (i, col) in row.iter().enumerate() {
              if i < num_cols {
                  widths[i] = widths[i].max(col.len());
              }
          }
      }

      // Format as aligned table
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

## Introduction

File processing is an excellent Wasm use case — CPU-intensive parsing and transformation runs faster in Rust, and everything stays client-side (no server upload needed).

## Reading Files from JavaScript

Users select files via `<input type="file">`, then pass the bytes to Rust:

```js
const input = document.getElementById('file-input');
input.addEventListener('change', async (e) => {
    const file = e.target.files[0];
    const buffer = await file.arrayBuffer();
    const bytes = new Uint8Array(buffer);

    // Pass to Rust
    const result = process_file(bytes);
});
```

In Rust, accept `&[u8]`:

```rust
#[wasm_bindgen]
pub fn process_file(data: &[u8]) -> String {
    // data is a Uint8Array from JS
    format!("Received {} bytes", data.len())
}
```

## CSV Processing

```rust
#[wasm_bindgen]
pub fn parse_csv(input: &str) -> JsValue {
    let mut records = Vec::new();

    for line in input.lines().skip(1) {  // skip header
        let fields: Vec<&str> = line.split(',').collect();
        records.push(fields);
    }

    // Convert to JS array
    serde_wasm_bindgen::to_value(&records).unwrap()
}
```

## Image Processing

Process image pixels in Rust for speed:

```rust
#[wasm_bindgen]
pub fn grayscale(pixels: &mut [u8]) {
    // pixels is RGBA format: [r, g, b, a, r, g, b, a, ...]
    for chunk in pixels.chunks_exact_mut(4) {
        let gray = (0.299 * chunk[0] as f64
                  + 0.587 * chunk[1] as f64
                  + 0.114 * chunk[2] as f64) as u8;
        chunk[0] = gray;  // R
        chunk[1] = gray;  // G
        chunk[2] = gray;  // B
        // chunk[3] = alpha (unchanged)
    }
}
```

```js
// Get image data from canvas
const imageData = ctx.getImageData(0, 0, width, height);
grayscale(imageData.data);  // Mutate in-place!
ctx.putImageData(imageData, 0, 0);
```

## Generating File Downloads

Create files in Rust and trigger downloads from JavaScript:

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

## Performance: Rust vs JavaScript

| Operation | JS | Rust/Wasm |
|-----------|-----|----------|
| Parse 10MB CSV | ~800ms | ~150ms |
| Image grayscale (4K) | ~50ms | ~8ms |
| JSON parse (1MB) | ~15ms | ~5ms |

Rust/Wasm wins significantly for data-intensive operations.

## Try It

Click **Run** to see the CSV formatter in action. It parses comma-separated input and outputs an aligned table.
