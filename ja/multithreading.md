---
title: "Wasmでのマルチスレッド"
slug: multithreading
difficulty: advanced
tags: [concurrency]
order: 40
description: ブラウザで並列計算を実現 — SharedArrayBuffer、Web Workers、wasm-bindgen-rayonがWebAssemblyにマルチスレッドをもたらし、実際の高速化を実現する方法を学びます。
starter_code: |
  use std::sync::{Arc, Mutex, Barrier};
  use std::thread;
  use std::time::Instant;

  // プレーンRustで並列計算パターンをデモします。
  // Wasmでは、これらのスレッドはSharedArrayBufferを使ったWeb Workersになります。

  // --- 逐次的な二乗和 ---
  fn sum_of_squares_sequential(data: &[u64]) -> u64 {
      data.iter().map(|x| x * x).sum()
  }

  // --- スレッドを使った並列二乗和 ---
  fn sum_of_squares_parallel(data: &[u64], num_threads: usize) -> u64 {
      let chunk_size = (data.len() + num_threads - 1) / num_threads;
      let data = Arc::new(data.to_vec());
      let mut handles = Vec::new();

      for i in 0..num_threads {
          let data = Arc::clone(&data);
          let start = i * chunk_size;
          let end = (start + chunk_size).min(data.len());

          handles.push(thread::spawn(move || {
              if start >= data.len() {
                  return 0u64;
              }
              data[start..end].iter().map(|x| x * x).sum::<u64>()
          }));
      }

      handles.into_iter().map(|h| h.join().unwrap()).sum()
  }

  // --- 並列map: 各要素に関数を適用 ---
  fn parallel_map(data: &[f64], num_threads: usize, f: fn(f64) -> f64) -> Vec<f64> {
      let chunk_size = (data.len() + num_threads - 1) / num_threads;
      let data = Arc::new(data.to_vec());
      let mut handles = Vec::new();

      for i in 0..num_threads {
          let data = Arc::clone(&data);
          let start = i * chunk_size;
          let end = (start + chunk_size).min(data.len());

          handles.push(thread::spawn(move || {
              if start >= data.len() {
                  return Vec::new();
              }
              data[start..end].iter().map(|&x| f(x)).collect::<Vec<f64>>()
          }));
      }

      let mut result = Vec::new();
      for h in handles {
          result.extend(h.join().unwrap());
      }
      result
  }

  // --- Mutexを使った共有カウンター（アトミック操作） ---
  fn parallel_count_primes(limit: u64, num_threads: usize) -> u64 {
      let count = Arc::new(Mutex::new(0u64));
      let chunk_size = (limit + num_threads as u64 - 1) / num_threads as u64;
      let barrier = Arc::new(Barrier::new(num_threads));
      let mut handles = Vec::new();

      for t in 0..num_threads {
          let count = Arc::clone(&count);
          let barrier = Arc::clone(&barrier);
          let start = t as u64 * chunk_size + 2;
          let end = ((t as u64 + 1) * chunk_size + 2).min(limit + 1);

          handles.push(thread::spawn(move || {
              let mut local_count = 0u64;
              for n in start..end {
                  if is_prime(n) {
                      local_count += 1;
                  }
              }
              // バリア: すべてのスレッドがカウントを終えるまで待機
              barrier.wait();
              // ローカル結果を共有カウンターにマージ
              let mut total = count.lock().unwrap();
              *total += local_count;
          }));
      }

      for h in handles {
          h.join().unwrap();
      }

      Arc::try_unwrap(count).unwrap().into_inner().unwrap()
  }

  fn is_prime(n: u64) -> bool {
      if n < 2 { return false; }
      if n < 4 { return true; }
      if n % 2 == 0 || n % 3 == 0 { return false; }
      let mut i = 5;
      while i * i <= n {
          if n % i == 0 || n % (i + 2) == 0 { return false; }
          i += 6;
      }
      true
  }

  fn main() {
      println!("=== マルチスレッドデモ ===\n");

      // 1. 二乗和: 逐次 vs 並列
      let data: Vec<u64> = (1..=10_000).collect();

      let start = Instant::now();
      let seq_result = sum_of_squares_sequential(&data);
      let seq_time = start.elapsed();

      let start = Instant::now();
      let par_result = sum_of_squares_parallel(&data, 4);
      let par_time = start.elapsed();

      println!("--- 二乗和 (1..=10,000) ---");
      println!("逐次: {} ({:?})", seq_result, seq_time);
      println!("並列(4): {} ({:?})", par_result, par_time);
      println!("結果一致: {}", seq_result == par_result);

      // 2. 並列map
      println!("\n--- 並列Map（最初の8つの値の平方根） ---");
      let floats: Vec<f64> = (1..=8).map(|x| x as f64).collect();
      let roots = parallel_map(&floats, 4, f64::sqrt);
      for (i, (val, root)) in floats.iter().zip(roots.iter()).enumerate() {
          println!("  sqrt({}) = {:.4}", val, root);
      }

      // 3. 並列素数カウント
      println!("\n--- 並列素数カウント ---");
      let limit = 10_000u64;
      let prime_count = parallel_count_primes(limit, 4);
      println!("{}以下の素数: {}", limit, prime_count);

      // 4. スレッドスケーリング
      println!("\n--- スレッドスケーリング（二乗和, 1..=50,000） ---");
      let big_data: Vec<u64> = (1..=50_000).collect();
      for threads in [1, 2, 4, 8] {
          let start = Instant::now();
          let _result = sum_of_squares_parallel(&big_data, threads);
          let elapsed = start.elapsed();
          println!("  {}スレッド: {:?}", threads, elapsed);
      }
  }
expected_output: |
  === マルチスレッドデモ ===

  --- 二乗和 (1..=10,000) ---
  逐次: 333383335000 ({seq_time})
  並列(4): 333383335000 ({par_time})
  結果一致: true

  --- 並列Map（最初の8つの値の平方根） ---
    sqrt(1) = 1.0000
    sqrt(2) = 1.4142
    sqrt(3) = 1.7321
    sqrt(4) = 2.0000
    sqrt(5) = 2.2361
    sqrt(6) = 2.4495
    sqrt(7) = 2.6458
    sqrt(8) = 2.8284

  --- 並列素数カウント ---
  10000以下の素数: 1229

  --- スレッドスケーリング（二乗和, 1..=50,000） ---
    1スレッド: {time}
    2スレッド: {time}
    4スレッド: {time}
    8スレッド: {time}
---

## WebAssemblyにおけるマルチスレッドの仕組み

ブラウザはデフォルトでシングルスレッドです。並列Wasmコードを実行するために、プラットフォームは2つのビルディングブロックを提供します：

```
  メインスレッド                Web Workers（スレッドプール）
  ┌─────────────────┐        ┌─────────────────┐
  │  JavaScript      │        │  Worker 1        │
  │  + Wasmインスタンス│       │  Wasmインスタンス  │
  │                  │  生成  │  （共有メモリ）    │
  │  postMessage() ──┼───────>│                  │
  │                  │        └─────────────────┘
  │                  │        ┌─────────────────┐
  │                  │        │  Worker 2        │
  │  SharedArray  ───┼───────>│  Wasmインスタンス  │
  │  Buffer          │        │  （共有メモリ）    │
  │                  │        └─────────────────┘
  │                  │        ┌─────────────────┐
  │                  │        │  Worker 3        │
  │  Atomics.wait()  │        │  Wasmインスタンス  │
  │  Atomics.notify()│<──────>│  Atomics操作      │
  └─────────────────┘        └─────────────────┘
```

### SharedArrayBuffer

`SharedArrayBuffer`は重要なプリミティブです。`ArrayBuffer`とは異なり、コピーなしでメインスレッドとWorker間で共有できます。Wasmのリニアメモリは`SharedArrayBuffer`をバッキングとして使用でき、すべてのスレッドが同じメモリにアクセスできます：

```javascript
// 共有Wasmメモリの作成
const memory = new WebAssembly.Memory({
  initial: 256,   // 256ページ（16 MB）
  maximum: 4096,  // 4096ページ（256 MB）
  shared: true     // ← SharedArrayBufferバッキングを有効化
});
```

### Atomics

`Atomics` APIは、Wasmの`memory.atomic.*`命令に直接マッピングされる低レベル同期プリミティブを提供します：

| Atomicsメソッド          | 目的                              | Wasm相当                   |
|-------------------------|-----------------------------------|---------------------------|
| `Atomics.load()`        | 共有値の読み取り                    | `i32.atomic.load`         |
| `Atomics.store()`       | 共有値の書き込み                    | `i32.atomic.store`        |
| `Atomics.add()`         | アトミックインクリメント             | `i32.atomic.rmw.add`      |
| `Atomics.compareExchange()` | CAS操作                        | `i32.atomic.rmw.cmpxchg`  |
| `Atomics.wait()`        | 通知まで待機（futex）               | `memory.atomic.wait32`    |
| `Atomics.notify()`      | 待機中のスレッドを起床               | `memory.atomic.notify`    |

## wasm-bindgen-rayon: Wasmでの並列イテレータ

`rayon`クレートはRustでデータ並列イテレータを提供します。`wasm-bindgen-rayon`アダプタはrayonのスレッドプールをWeb Workersに橋渡しします：

```rust
// Cargo.toml
// [dependencies]
// rayon = "1.8"
// wasm-bindgen-rayon = "1.2"

use rayon::prelude::*;
use wasm_bindgen::prelude::*;

#[wasm_bindgen]
pub fn parallel_sum(data: &[f64]) -> f64 {
    data.par_iter().sum()
}

#[wasm_bindgen]
pub fn parallel_mandelbrot(width: u32, height: u32) -> Vec<u8> {
    let pixels: Vec<u8> = (0..height)
        .into_par_iter()  // ← 並列イテレーション
        .flat_map(|y| {
            (0..width).map(move |x| compute_pixel(x, y, width, height))
        })
        .collect();
    pixels
}
```

### セットアップ

```rust
// lib.rs — rayonを使う前にこれを呼ぶ必要がある
pub use wasm_bindgen_rayon::init_thread_pool;
```

```javascript
// JavaScript側
import init, { initThreadPool, parallel_sum } from './pkg/my_crate';

async function main() {
    await init();
    await initThreadPool(navigator.hardwareConcurrency);
    // これでrayonのpar_iter()がWeb Workersを使います!
    const result = parallel_sum(new Float64Array([1, 2, 3, 4, 5]));
}
```

### ビルド設定

```toml
# .cargo/config.toml
[target.wasm32-unknown-unknown]
rustflags = ["-C", "target-feature=+atomics,+bulk-memory,+mutable-globals"]

[unstable]
build-std = ["panic_abort", "std"]
```

## COOP/COEPヘッダー（必須!）

`SharedArrayBuffer`はCross-Origin Isolationの制約を受けます。サーバーは**必ず**以下のヘッダーを送信する必要があります：

```
Cross-Origin-Opener-Policy: same-origin
Cross-Origin-Embedder-Policy: require-corp
```

```
  ブラウザセキュリティモデル:

  ヘッダーなし:                ヘッダーあり:
  ┌──────────────────┐         ┌──────────────────┐
  │ SharedArrayBuffer │         │ SharedArrayBuffer │
  │     ブロック       │         │     許可           │
  │                  │         │                  │
  │ "SecurityError:  │         │ Cross-origin     │
  │  SharedArray     │         │ isolated = true  │
  │  Buffer is not   │         │                  │
  │  defined"        │         │ COOP: same-origin│
  └──────────────────┘         │ COEP: require-corp│
                               └──────────────────┘
```

### サーバー設定の例

```
# Nginx
add_header Cross-Origin-Opener-Policy same-origin;
add_header Cross-Origin-Embedder-Policy require-corp;

# Apache (.htaccess)
Header set Cross-Origin-Opener-Policy "same-origin"
Header set Cross-Origin-Embedder-Policy "require-corp"

# Vite (vite.config.ts)
export default {
  server: {
    headers: {
      'Cross-Origin-Opener-Policy': 'same-origin',
      'Cross-Origin-Embedder-Policy': 'require-corp',
    },
  },
};
```

## スレッドが効果的な場合とそうでない場合

スレッドにはオーバーヘッド（Worker生成、同期、メッセージパッシング）が伴います。そのコストを償却できるほどの十分な作業量がある場合にのみ効果があります：

```
  高速化
  │
  │          ●  理想的（線形）
  │        ●╱
  │      ●╱    ●── 実際（アムダールの法則）
  │    ●╱    ●
  │  ●╱   ●
  │●╱  ●         ●── 収穫逓減
  │╱●
  │●
  ├───────────────────── スレッド数
  1   2   4   8  16

  アムダールの法則: 高速化 = 1 / (S + P/N)
    S = 逐次部分の割合
    P = 並列部分の割合 (S + P = 1)
    N = スレッド数
```

| ワークロード                         | スレッドは効果的? | 理由                                   |
|------------------------------------|-----------------|---------------------------------------|
| マンデルブロ描画（1024x1024）         | はい            | 完全並列、CPU集約型                      |
| 画像フィルタ（大きな画像）             | はい            | 各ピクセルが独立                         |
| 1000要素のソート                     | いいえ          | 小さすぎ、オーバーヘッド > 節約            |
| JSONパース                          | いいえ          | ほぼ逐次的                              |
| 行列乗算（大規模）                   | はい            | 行をスレッド間で分割                      |
| DOM操作                             | いいえ          | メインスレッドで実行する必要あり            |
| 物理シミュレーション（1000+物体）      | はい            | 各物体の更新が独立                       |
| 短い文字列のSHA-256                  | いいえ          | 逐次アルゴリズム、微小入力                 |

**経験則**: 逐次処理で約5ms未満の作業であれば、スレッドのオーバーヘッドが利得を打ち消します。

## スレッドプールアーキテクチャ

```
  ┌─────────────────────────────────────────────────────────┐
  │  メインスレッド                                          │
  │                                                         │
  │  1. initThreadPool(4)                                   │
  │     ├── Worker 1を生成 ──┐                               │
  │     ├── Worker 2を生成 ──┤  Workersは同じ.wasmを         │
  │     ├── Worker 3を生成 ──┤  共有メモリでロード            │
  │     └── Worker 4を生成 ──┘                               │
  │                                                         │
  │  2. parallel_sum(data)                                  │
  │     │                                                   │
  │     ▼                                                   │
  │  ┌──────────────────────────────────────┐               │
  │  │ rayonワークスティーリングスケジューラ    │               │
  │  │                                      │               │
  │  │  タスクキュー: [chunk1][chunk2]...    │               │
  │  │                                      │               │
  │  │  W1 ← steal ← W2 ← steal ← W3     │               │
  │  │                                      │               │
  │  │  各Workerはアトミックload/storeを通じて│               │
  │  │  共有メモリのチャンクを処理            │               │
  │  └──────────────────────────────────────┘               │
  │                                                         │
  │  3. 結果がメインスレッドに返される                         │
  └─────────────────────────────────────────────────────────┘
```

## ブラウザサポート

| ブラウザ          | SharedArrayBuffer | Wasmスレッド   | 状態                       |
|-----------------|-------------------|--------------|---------------------------|
| Chrome 91+      | 対応              | 対応          | フルサポート               |
| Firefox 79+     | 対応              | 対応          | フルサポート               |
| Safari 15.2+    | 対応              | 対応          | フルサポート               |
| Edge 91+        | 対応              | 対応          | フルサポート（Chromium）    |
| Node.js 16+     | 対応              | 対応          | `--experimental-wasm-threads` |
| Deno 1.9+       | 対応              | 対応          | フルサポート               |

### 機能検出

```javascript
function supportsWasmThreads() {
    try {
        // SharedArrayBufferをチェック
        new SharedArrayBuffer(1);

        // Atomicsをチェック
        if (typeof Atomics === 'undefined') return false;

        // Cross-Origin Isolationをチェック
        if (!crossOriginIsolated) return false;

        // Wasmスレッドサポートをチェック
        const mem = new WebAssembly.Memory({ initial: 1, maximum: 1, shared: true });
        return mem.buffer instanceof SharedArrayBuffer;
    } catch {
        return false;
    }
}
```

## 実践例: 並列画像処理

```rust
use rayon::prelude::*;
use wasm_bindgen::prelude::*;

#[wasm_bindgen]
pub fn blur_image(pixels: &mut [u8], width: usize, height: usize, radius: usize) {
    let input = pixels.to_vec();

    // 行を並列処理
    pixels
        .par_chunks_mut(width * 4)
        .enumerate()
        .for_each(|(y, row)| {
            for x in 0..width {
                let (mut r, mut g, mut b, mut count) = (0u32, 0u32, 0u32, 0u32);

                for dy in -(radius as i32)..=(radius as i32) {
                    for dx in -(radius as i32)..=(radius as i32) {
                        let ny = (y as i32 + dy).clamp(0, height as i32 - 1) as usize;
                        let nx = (x as i32 + dx).clamp(0, width as i32 - 1) as usize;
                        let idx = (ny * width + nx) * 4;
                        r += input[idx] as u32;
                        g += input[idx + 1] as u32;
                        b += input[idx + 2] as u32;
                        count += 1;
                    }
                }

                let idx = x * 4;
                row[idx] = (r / count) as u8;
                row[idx + 1] = (g / count) as u8;
                row[idx + 2] = (b / count) as u8;
                // アルファは変更なし
            }
        });
}
```

## まとめ

Wasmマルチスレッドは、共有メモリに`SharedArrayBuffer`を、スレッドプールにWeb Workersを使用します。`wasm-bindgen-rayon`クレートにより、通常のRust並列コードを書く感覚で使えます — `iter()`の代わりに`par_iter()`を使うだけです。COOP/COEPヘッダーの設定、`+atomics`でのビルド、そしてスレッドのオーバーヘッドを克服するのに十分な大きさのワークロードのみを並列化することを忘れないでください。2024年現在、ほとんどのモダンブラウザがWasmスレッドを完全にサポートしています。
