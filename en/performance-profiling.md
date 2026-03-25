---
title: Wasm Performance Profiling
slug: performance-profiling
difficulty: intermediate
tags: [getting-started]
order: 26
description: Measure, profile, and optimize Wasm performance — binary size reduction, runtime profiling, and benchmarking techniques.
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

## Why Profile Wasm?

Wasm is fast, but not automatically optimal. Common issues:
- **Binary too large** — slow initial load
- **Excessive boundary crossings** — copying strings/data JS↔Wasm
- **Unnecessary allocations** — creating Vec/String per frame
- **Unoptimized algorithms** — O(n²) when O(n) is possible

## Measuring Binary Size

```bash
# Check raw size
ls -lh pkg/my_app_bg.wasm

# See what's taking space
cargo install twiggy
twiggy top pkg/my_app_bg.wasm

# Example output:
#  Shallow Bytes │ Shallow % │ Item
# ───────────────┼───────────┼──────────────
#         12,458 │   15.32%  │ data[0]
#          8,234 │   10.13%  │ wasm_bindgen::convert
#          5,120 │    6.30%  │ core::fmt
```

## Reducing Binary Size

### Cargo.toml optimizations

```toml
[profile.release]
opt-level = "z"       # Optimize for size (smallest)
lto = true            # Link-time optimization
codegen-units = 1     # Better optimization
strip = true          # Remove debug symbols
panic = "abort"       # Smaller panic handling
```

### wasm-opt (10-30% further reduction)

```bash
# Install binaryen
npm install -g binaryen

# Optimize
wasm-opt -Oz -o small.wasm pkg/my_app_bg.wasm

# Compare
ls -lh pkg/my_app_bg.wasm small.wasm
```

### Avoid heavy dependencies

| Crate | Size impact | Alternative |
|-------|-----------|-------------|
| `serde_json` | +50-100KB | `serde-wasm-bindgen` (0KB, uses JsValue) |
| `regex` | +100-200KB | Manual string parsing |
| `chrono` | +50KB | `js_sys::Date` (free, uses JS Date) |
| `rand` | +30KB | `js_sys::Math::random()` (free) |

## Runtime Profiling

### console.time (simple)

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
    // ... your code ...
    time_end("expensive_operation");
}
```

### performance.now (precise)

```js
const start = performance.now();
wasm_function();
const elapsed = performance.now() - start;
console.log(`Took ${elapsed.toFixed(2)}ms`);
```

### Browser DevTools Profiler

1. Open DevTools → **Performance** tab
2. Click **Record**
3. Interact with your app
4. Stop recording
5. Look for `wasm-function[N]` in the flame chart
6. Sort by "Self Time" to find bottlenecks

## Benchmarking Best Practices

```js
// BAD: single measurement (noisy)
const t = performance.now();
result = wasm_fn();
console.log(performance.now() - t);

// GOOD: multiple iterations, warm up
function bench(fn, iterations = 1000) {
    // Warm up (JIT, caches)
    for (let i = 0; i < 10; i++) fn();

    const start = performance.now();
    for (let i = 0; i < iterations; i++) fn();
    const elapsed = performance.now() - start;

    return elapsed / iterations;
}
```

## Common Optimization Patterns

### 1. Batch boundary crossings

```rust
// BAD: N boundary crossings
for item in items {
    let result = process(item);  // JS calls Wasm N times
    display(result);             // Wasm returns to JS N times
}

// GOOD: 1 boundary crossing
let results = process_all(items);  // One call, one return
```

### 2. Reuse allocations

```rust
// BAD: allocates every call
pub fn get_data(&self) -> Vec<f64> {
    self.items.iter().map(|i| i.value).collect()
}

// GOOD: reuse buffer
pub fn get_data(&mut self, output: &mut [f64]) {
    for (i, item) in self.items.iter().enumerate() {
        output[i] = item.value;
    }
}
```

## Size Targets

| Category | Size | Quality |
|----------|------|---------|
| < 50KB | Excellent | Loads in < 100ms |
| 50-150KB | Good | Acceptable for most apps |
| 150-500KB | Okay | Consider lazy loading |
| > 500KB | Large | Needs optimization |
| > 1MB | Too large | Split or reduce dependencies |

## Try It

Click **Run** to see benchmarking in action. The code compares a loop vs formula approach and measures fibonacci — demonstrating how to identify performance differences.
