---
title: Wasm + Web Workers
slug: web-workers
difficulty: advanced
tags: [concurrency]
order: 19
description: Web Workersを使ってWasmを並列実行し、メインスレッドをブロックせずにブラウザでマルチスレッドパフォーマンスを実現します。
starter_code: |
  use wasm_bindgen::prelude::*;

  #[wasm_bindgen]
  pub fn heavy_computation(iterations: u32) -> f64 {
      let mut sum = 0.0f64;
      for i in 0..iterations {
          sum += (i as f64).sin() * (i as f64).cos();
      }
      sum
  }

  #[wasm_bindgen]
  pub fn is_prime(n: u64) -> bool {
      if n < 2 { return false; }
      if n < 4 { return true; }
      if n % 2 == 0 || n % 3 == 0 { return false; }
      let mut i = 5u64;
      while i * i <= n {
          if n % i == 0 || n % (i + 2) == 0 {
              return false;
          }
          i += 6;
      }
      true
  }

  #[wasm_bindgen]
  pub fn count_primes(up_to: u64) -> u64 {
      (2..=up_to).filter(|&n| is_prime(n)).count() as u64
  }
expected_output: |
  heavy_computation(10_000_000): 0.3427...
  count_primes(100_000): 9592
  UIをブロックせずに全て計算完了！
---

## なぜWeb Workersが必要なのか？

Wasmはデフォルトでメインスレッドで実行されます。重い計算はUIをブロックし、ボタンがフリーズしたりアニメーションが停止したりします。Web Workersを使えば、Wasmを**別のスレッド**で実行できます。

```
メインスレッド（UI）              ワーカースレッド（計算）
┌──────────────┐              ┌──────────────┐
│ DOM、イベント │  postMessage │ Wasmモジュール│
│ アニメーション│ ◀──────────▶│ 重い計算処理  │
│ レスポンシブ  │              │ ブロックしない│
└──────────────┘              └──────────────┘
```

## 基本的なWorkerのセットアップ

### worker.js
```js
import init, { heavy_computation } from './pkg/my_wasm.js';

self.onmessage = async (e) => {
    await init();

    const { type, data } = e.data;

    if (type === 'compute') {
        const result = heavy_computation(data.iterations);
        self.postMessage({ type: 'result', result });
    }
};
```

### main.js
```js
const worker = new Worker('./worker.js', { type: 'module' });

worker.onmessage = (e) => {
    if (e.data.type === 'result') {
        console.log('Result:', e.data.result);
    }
};

// UIをブロックしない
worker.postMessage({
    type: 'compute',
    data: { iterations: 10_000_000 }
});
```

## 複数のWorker（並列処理）

複数のWorkerに作業を分割して真の並列処理を実現します：

```js
function createWorkerPool(size) {
    const workers = [];
    for (let i = 0; i < size; i++) {
        workers.push(new Worker('./worker.js', { type: 'module' }));
    }
    return workers;
}

async function parallelPrimeCount(max, numWorkers = 4) {
    const pool = createWorkerPool(numWorkers);
    const chunkSize = Math.ceil(max / numWorkers);

    const promises = pool.map((worker, i) => {
        const start = i * chunkSize + 1;
        const end = Math.min((i + 1) * chunkSize, max);

        return new Promise((resolve) => {
            worker.onmessage = (e) => resolve(e.data.count);
            worker.postMessage({ type: 'count_primes', start, end });
        });
    });

    const counts = await Promise.all(promises);
    return counts.reduce((a, b) => a + b, 0);
}
```

## SharedArrayBuffer（上級）

コピーなしでWorker間でメモリを共有します：

```js
// サーバーに以下のヘッダーが必要です：
// Cross-Origin-Opener-Policy: same-origin
// Cross-Origin-Embedder-Policy: require-corp

const shared = new SharedArrayBuffer(1024);
const view = new Float64Array(shared);

// メインスレッドとワーカーの両方が読み書き可能
worker.postMessage({ buffer: shared });
```

```rust
// Rustでは共有メモリにポインタ経由でアクセス
#[wasm_bindgen]
pub fn process_shared(ptr: *mut f64, len: usize) {
    let data = unsafe { std::slice::from_raw_parts_mut(ptr, len) };
    for x in data.iter_mut() {
        *x *= 2.0;
    }
}
```

## Web Workersを使うべき場面

| ユースケース | メインスレッド | Web Worker |
|----------|-------------|------------|
| DOM操作 | ✓ | ✗（DOMアクセス不可） |
| イベントハンドラ | ✓ | ✗ |
| 重い計算 | ✗（UIがブロックされる） | ✓ |
| 画像処理 | ✗ | ✓ |
| 暗号化処理 | ✗ | ✓ |
| 物理シミュレーション | ✗ | ✓ |
| データ解析 | サイズによる | ✓（大きなファイル向け） |

## パフォーマンス比較

| タスク | メインスレッド | Worker 1つ | Worker 4つ |
|------|------------|----------|-----------|
| 100万までの素数カウント | 800ms（UIフリーズ） | 800ms（UIレスポンシブ） | 200ms |
| 画像ぼかし（4K） | 120ms（UIフリーズ） | 120ms（UIレスポンシブ） | 35ms |

Workerは個々のタスクを高速化するわけではありません。UIをレスポンシブに保ち、並列処理を可能にするものです。

## 試してみよう

スターターコードには、メインスレッドで実行するとUIをブロックしてしまうCPU集約型の関数（三角関数の計算、素数カウント）が含まれています。本番環境では、これらをWeb Workerでラップして使用します。
