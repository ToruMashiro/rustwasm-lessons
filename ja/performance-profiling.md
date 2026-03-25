---
title: Wasmパフォーマンスプロファイリング
slug: performance-profiling
difficulty: intermediate
tags: [getting-started]
order: 26
description: Wasmのパフォーマンスを測定、プロファイル、最適化する — バイナリサイズの削減、ランタイムプロファイリング、ベンチマーク手法。
starter_code: |
  use std::time::Instant;

  fn benchmark<F: Fn() -> R, R>(name: &str, iterations: u32, f: F) -> R {
      let start = Instant::now();
      let mut result = f();
      for _ in 1..iterations {
          result = f();
      }
      let elapsed = start.elapsed();
      let per_iter = elapsed / iterations;
      println!("{}: {:?} total, {:?} per iteration ({} runs)",
          name, elapsed, per_iter, iterations);
      result
  }

  fn sum_loop(n: u64) -> u64 {
      let mut sum = 0u64;
      for i in 0..n {
          sum += i;
      }
      sum
  }

  fn sum_formula(n: u64) -> u64 {
      n * (n - 1) / 2
  }

  fn fibonacci(n: u32) -> u64 {
      let mut a = 0u64;
      let mut b = 1u64;
      for _ in 0..n {
          let temp = b;
          b = a + b;
          a = temp;
      }
      a
  }

  fn main() {
      let n = 10_000_000;

      let r1 = benchmark("sum_loop    ", 10, || sum_loop(n));
      let r2 = benchmark("sum_formula ", 10, || sum_formula(n));
      println!("Both equal: {}\n", r1 == r2);

      benchmark("fibonacci(80)", 1000, || fibonacci(80));
  }
expected_output: |
  sum_loop    : 142ms total, 14ms per iteration (10 runs)
  sum_formula : 0µs total, 0µs per iteration (10 runs)
  Both equal: true

  fibonacci(80): 12µs total, 0µs per iteration (1000 runs)
---

## なぜWasmをプロファイルするのか？

Wasmは高速ですが、自動的に最適というわけではありません。よくある問題:
- **バイナリが大きすぎる** — 初回ロードが遅い
- **過度な境界越え** — JS↔Wasm間で文字列やデータをコピーしすぎ
- **不必要なアロケーション** — フレームごとにVec/Stringを生成
- **最適化されていないアルゴリズム** — O(n)で済むところをO(n²)

## バイナリサイズの測定

```bash
# 生のサイズを確認
ls -lh pkg/my_app_bg.wasm

# 何がスペースを占めているか確認
cargo install twiggy
twiggy top pkg/my_app_bg.wasm

# 出力例:
#  Shallow Bytes │ Shallow % │ Item
# ───────────────┼───────────┼──────────────
#         12,458 │   15.32%  │ data[0]
#          8,234 │   10.13%  │ wasm_bindgen::convert
#          5,120 │    6.30%  │ core::fmt
```

## バイナリサイズの削減

### Cargo.tomlの最適化

```toml
[profile.release]
opt-level = "z"       # サイズ最適化（最小）
lto = true            # リンク時最適化
codegen-units = 1     # より良い最適化
strip = true          # デバッグシンボルを除去
panic = "abort"       # パニック処理を小さく
```

### wasm-opt（さらに10-30%削減）

```bash
# binaryenをインストール
npm install -g binaryen

# 最適化
wasm-opt -Oz -o small.wasm pkg/my_app_bg.wasm

# 比較
ls -lh pkg/my_app_bg.wasm small.wasm
```

### 重い依存関係を避ける

| クレート | サイズへの影響 | 代替案 |
|-------|-----------|-------------|
| `serde_json` | +50-100KB | `serde-wasm-bindgen`（0KB、JsValueを使用） |
| `regex` | +100-200KB | 手動の文字列パース |
| `chrono` | +50KB | `js_sys::Date`（無料、JS Dateを使用） |
| `rand` | +30KB | `js_sys::Math::random()`（無料） |

## ランタイムプロファイリング

### console.time（シンプル）

```rust
#[wasm_bindgen]
extern "C" {
    #[wasm_bindgen(js_namespace = console)]
    fn time(label: &str);

    #[wasm_bindgen(js_namespace = console, js_name = "timeEnd")]
    fn time_end(label: &str);
}

#[wasm_bindgen]
pub fn expensive_operation() {
    time("expensive_operation");
    // ... 処理コード ...
    time_end("expensive_operation");
}
```

### performance.now（精密）

```js
const start = performance.now();
wasm_function();
const elapsed = performance.now() - start;
console.log(`Took ${elapsed.toFixed(2)}ms`);
```

### ブラウザDevToolsプロファイラー

1. DevToolsを開く → **Performance** タブ
2. **Record** をクリック
3. アプリを操作する
4. 記録を停止
5. フレームチャートで `wasm-function[N]` を探す
6. 「Self Time」でソートしてボトルネックを特定

## ベンチマークのベストプラクティス

```js
// 悪い例: 1回の測定（ノイズが多い）
const t = performance.now();
result = wasm_fn();
console.log(performance.now() - t);

// 良い例: 複数回の反復、ウォームアップ
function bench(fn, iterations = 1000) {
    // ウォームアップ（JIT、キャッシュ）
    for (let i = 0; i < 10; i++) fn();

    const start = performance.now();
    for (let i = 0; i < iterations; i++) fn();
    const elapsed = performance.now() - start;

    return elapsed / iterations;
}
```

## よくある最適化パターン

### 1. 境界越えをバッチ処理

```rust
// 悪い例: N回の境界越え
for item in items {
    let result = process(item);  // JSがWasmをN回呼び出す
    display(result);             // WasmがJSにN回戻る
}

// 良い例: 1回の境界越え
let results = process_all(items);  // 1回の呼び出し、1回の返却
```

### 2. アロケーションの再利用

```rust
// 悪い例: 呼び出しのたびにアロケーション
pub fn get_data(&self) -> Vec<f64> {
    self.items.iter().map(|i| i.value).collect()
}

// 良い例: バッファを再利用
pub fn get_data(&mut self, output: &mut [f64]) {
    for (i, item) in self.items.iter().enumerate() {
        output[i] = item.value;
    }
}
```

## サイズの目標値

| カテゴリ | サイズ | 評価 |
|----------|------|---------|
| < 50KB | 優秀 | 100ms未満でロード |
| 50-150KB | 良好 | ほとんどのアプリで許容範囲 |
| 150-500KB | まずまず | 遅延読み込みを検討 |
| > 500KB | 大きい | 最適化が必要 |
| > 1MB | 大きすぎる | 分割または依存関係を削減 |

## 試してみよう

**Run** をクリックして、ベンチマークの実例を確認しましょう。ループと公式のアプローチを比較し、フィボナッチを測定します — パフォーマンスの違いを特定する方法を実演します。
