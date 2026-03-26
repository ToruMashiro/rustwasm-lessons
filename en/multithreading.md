---
title: "Multithreading with Wasm"
slug: multithreading
difficulty: advanced
tags: [concurrency]
order: 40
description: Unlock parallel computation in the browser — learn how SharedArrayBuffer, Web Workers, and wasm-bindgen-rayon bring multithreading to WebAssembly with real speedups.
starter_code: |
  use std::sync::{Arc, Mutex, Barrier};
  use std::thread;
  use std::time::Instant;

  // Demonstrating parallel computation patterns in plain Rust.
  // In Wasm, these threads become Web Workers with SharedArrayBuffer.

  // --- Sequential sum of squares ---
  fn sum_of_squares_sequential(data: &[u64]) -> u64 {
      data.iter().map(|x| x * x).sum()
  }

  // --- Parallel sum of squares using threads ---
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

  // --- Parallel map: apply a function to each element ---
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

  // --- Shared counter with Mutex (atomic operations) ---
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
              // Barrier: wait for all threads to finish counting
              barrier.wait();
              // Merge local result into shared counter
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
      println!("=== Multithreading Demo ===\n");

      // 1. Sum of squares: sequential vs parallel
      let data: Vec<u64> = (1..=10_000).collect();

      let start = Instant::now();
      let seq_result = sum_of_squares_sequential(&data);
      let seq_time = start.elapsed();

      let start = Instant::now();
      let par_result = sum_of_squares_parallel(&data, 4);
      let par_time = start.elapsed();

      println!("--- Sum of Squares (1..=10,000) ---");
      println!("Sequential: {} ({:?})", seq_result, seq_time);
      println!("Parallel(4): {} ({:?})", par_result, par_time);
      println!("Results match: {}", seq_result == par_result);

      // 2. Parallel map
      println!("\n--- Parallel Map (sqrt of first 8 values) ---");
      let floats: Vec<f64> = (1..=8).map(|x| x as f64).collect();
      let roots = parallel_map(&floats, 4, f64::sqrt);
      for (i, (val, root)) in floats.iter().zip(roots.iter()).enumerate() {
          println!("  sqrt({}) = {:.4}", val, root);
      }

      // 3. Parallel prime counting
      println!("\n--- Parallel Prime Counting ---");
      let limit = 10_000u64;
      let prime_count = parallel_count_primes(limit, 4);
      println!("Primes up to {}: {}", limit, prime_count);

      // 4. Thread scaling
      println!("\n--- Thread Scaling (sum of squares, 1..=50,000) ---");
      let big_data: Vec<u64> = (1..=50_000).collect();
      for threads in [1, 2, 4, 8] {
          let start = Instant::now();
          let _result = sum_of_squares_parallel(&big_data, threads);
          let elapsed = start.elapsed();
          println!("  {} thread(s): {:?}", threads, elapsed);
      }
  }
expected_output: |
  === Multithreading Demo ===

  --- Sum of Squares (1..=10,000) ---
  Sequential: 333383335000 ({seq_time})
  Parallel(4): 333383335000 ({par_time})
  Results match: true

  --- Parallel Map (sqrt of first 8 values) ---
    sqrt(1) = 1.0000
    sqrt(2) = 1.4142
    sqrt(3) = 1.7321
    sqrt(4) = 2.0000
    sqrt(5) = 2.2361
    sqrt(6) = 2.4495
    sqrt(7) = 2.6458
    sqrt(8) = 2.8284

  --- Parallel Prime Counting ---
  Primes up to 10000: 1229

  --- Thread Scaling (sum of squares, 1..=50,000) ---
    1 thread(s): {time}
    2 thread(s): {time}
    4 thread(s): {time}
    8 thread(s): {time}
---

## How Multithreading Works in WebAssembly

Browsers are single-threaded by default. To run parallel Wasm code, the platform provides two building blocks:

```
  Main Thread                 Web Workers (Thread Pool)
  ┌─────────────────┐        ┌─────────────────┐
  │  JavaScript      │        │  Worker 1        │
  │  + Wasm instance │        │  Wasm instance   │
  │                  │  spawn │  (shared memory) │
  │  postMessage() ──┼───────>│                  │
  │                  │        └─────────────────┘
  │                  │        ┌─────────────────┐
  │                  │        │  Worker 2        │
  │  SharedArray  ───┼───────>│  Wasm instance   │
  │  Buffer          │        │  (shared memory) │
  │                  │        └─────────────────┘
  │                  │        ┌─────────────────┐
  │                  │        │  Worker 3        │
  │  Atomics.wait()  │        │  Wasm instance   │
  │  Atomics.notify()│<──────>│  Atomics ops     │
  └─────────────────┘        └─────────────────┘
```

### SharedArrayBuffer

`SharedArrayBuffer` is the key primitive. Unlike `ArrayBuffer`, it can be shared between the main thread and workers without copying. Wasm's linear memory can be backed by a `SharedArrayBuffer`, giving all threads access to the same memory:

```javascript
// Creating shared Wasm memory
const memory = new WebAssembly.Memory({
  initial: 256,   // 256 pages (16 MB)
  maximum: 4096,  // 4096 pages (256 MB)
  shared: true     // ← This enables SharedArrayBuffer backing
});
```

### Atomics

The `Atomics` API provides low-level synchronization primitives that map directly to Wasm's `memory.atomic.*` instructions:

| Atomics Method       | Purpose                           | Wasm Equivalent         |
|----------------------|-----------------------------------|-------------------------|
| `Atomics.load()`     | Read shared value                 | `i32.atomic.load`       |
| `Atomics.store()`    | Write shared value                | `i32.atomic.store`      |
| `Atomics.add()`      | Atomic increment                  | `i32.atomic.rmw.add`    |
| `Atomics.compareExchange()` | CAS operation              | `i32.atomic.rmw.cmpxchg`|
| `Atomics.wait()`     | Block until notified (futex)      | `memory.atomic.wait32`  |
| `Atomics.notify()`   | Wake waiting threads              | `memory.atomic.notify`  |

## wasm-bindgen-rayon: Parallel Iterators in Wasm

The `rayon` crate provides data-parallel iterators in Rust. The `wasm-bindgen-rayon` adapter bridges rayon's thread pool to Web Workers:

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
        .into_par_iter()  // ← parallel iteration
        .flat_map(|y| {
            (0..width).map(move |x| compute_pixel(x, y, width, height))
        })
        .collect();
    pixels
}
```

### Setup

```rust
// lib.rs — must call this before using rayon
pub use wasm_bindgen_rayon::init_thread_pool;
```

```javascript
// JavaScript side
import init, { initThreadPool, parallel_sum } from './pkg/my_crate';

async function main() {
    await init();
    await initThreadPool(navigator.hardwareConcurrency);
    // Now rayon's par_iter() uses Web Workers!
    const result = parallel_sum(new Float64Array([1, 2, 3, 4, 5]));
}
```

### Build Configuration

```toml
# .cargo/config.toml
[target.wasm32-unknown-unknown]
rustflags = ["-C", "target-feature=+atomics,+bulk-memory,+mutable-globals"]

[unstable]
build-std = ["panic_abort", "std"]
```

## COOP/COEP Headers (Required!)

`SharedArrayBuffer` is gated behind Cross-Origin Isolation. Your server **must** send these headers:

```
Cross-Origin-Opener-Policy: same-origin
Cross-Origin-Embedder-Policy: require-corp
```

```
  Browser Security Model:

  Without headers:              With headers:
  ┌──────────────────┐         ┌──────────────────┐
  │ SharedArrayBuffer │         │ SharedArrayBuffer │
  │     BLOCKED       │         │     ALLOWED       │
  │                  │         │                  │
  │ "SecurityError:  │         │ Cross-origin     │
  │  SharedArray     │         │ isolated = true  │
  │  Buffer is not   │         │                  │
  │  defined"        │         │ COOP: same-origin│
  └──────────────────┘         │ COEP: require-corp│
                               └──────────────────┘
```

### Server configuration examples

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

## When Threading Helps vs Hurts

Threading adds overhead (worker creation, synchronization, message passing). It only helps when the work is large enough to amortize that cost:

```
  Speedup
  │
  │          ●  ideal (linear)
  │        ●╱
  │      ●╱    ●── real (Amdahl's law)
  │    ●╱    ●
  │  ●╱   ●
  │●╱  ●         ●── diminishing returns
  │╱●
  │●
  ├───────────────────── Number of threads
  1   2   4   8  16

  Amdahl's Law: Speedup = 1 / (S + P/N)
    S = serial fraction
    P = parallel fraction (S + P = 1)
    N = number of threads
```

| Workload                           | Threads Help? | Why                                    |
|------------------------------------|---------------|----------------------------------------|
| Mandelbrot rendering (1024x1024)   | Yes           | Embarrassingly parallel, CPU-heavy     |
| Image filter (large image)         | Yes           | Each pixel independent                 |
| Sorting 1000 items                 | No            | Too small, overhead > savings          |
| JSON parsing                       | No            | Mostly sequential                      |
| Matrix multiplication (large)      | Yes           | Divide rows across threads             |
| DOM manipulation                   | No            | Must happen on main thread             |
| Physics simulation (1000+ bodies)  | Yes           | Each body update is independent        |
| SHA-256 of a small string          | No            | Sequential algorithm, tiny input       |

**Rule of thumb**: if the sequential work takes less than ~5ms, threading overhead will eat any gains.

## Thread Pool Architecture

```
  ┌─────────────────────────────────────────────────────────┐
  │  Main Thread                                            │
  │                                                         │
  │  1. initThreadPool(4)                                   │
  │     ├── spawn Worker 1 ──┐                              │
  │     ├── spawn Worker 2 ──┤  Workers load same .wasm     │
  │     ├── spawn Worker 3 ──┤  with shared memory          │
  │     └── spawn Worker 4 ──┘                              │
  │                                                         │
  │  2. parallel_sum(data)                                  │
  │     │                                                   │
  │     ▼                                                   │
  │  ┌──────────────────────────────────────┐               │
  │  │ rayon work-stealing scheduler        │               │
  │  │                                      │               │
  │  │  Task queue: [chunk1][chunk2]...     │               │
  │  │                                      │               │
  │  │  W1 ← steal ← W2 ← steal ← W3     │               │
  │  │                                      │               │
  │  │  Each worker processes chunks from   │               │
  │  │  shared memory via atomic load/store │               │
  │  └──────────────────────────────────────┘               │
  │                                                         │
  │  3. Result returned to main thread                      │
  └─────────────────────────────────────────────────────────┘
```

## Browser Support

| Browser         | SharedArrayBuffer | Wasm Threads | Status                    |
|-----------------|-------------------|--------------|---------------------------|
| Chrome 91+      | Yes               | Yes          | Full support              |
| Firefox 79+     | Yes               | Yes          | Full support              |
| Safari 15.2+    | Yes               | Yes          | Full support              |
| Edge 91+        | Yes               | Yes          | Full support (Chromium)   |
| Node.js 16+     | Yes               | Yes          | `--experimental-wasm-threads` |
| Deno 1.9+       | Yes               | Yes          | Full support              |

### Feature detection

```javascript
function supportsWasmThreads() {
    try {
        // Check SharedArrayBuffer
        new SharedArrayBuffer(1);

        // Check Atomics
        if (typeof Atomics === 'undefined') return false;

        // Check cross-origin isolation
        if (!crossOriginIsolated) return false;

        // Check Wasm threads support
        const mem = new WebAssembly.Memory({ initial: 1, maximum: 1, shared: true });
        return mem.buffer instanceof SharedArrayBuffer;
    } catch {
        return false;
    }
}
```

## Practical Example: Parallel Image Processing

```rust
use rayon::prelude::*;
use wasm_bindgen::prelude::*;

#[wasm_bindgen]
pub fn blur_image(pixels: &mut [u8], width: usize, height: usize, radius: usize) {
    let input = pixels.to_vec();

    // Process rows in parallel
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
                // Alpha unchanged
            }
        });
}
```

## Summary

Wasm multithreading uses `SharedArrayBuffer` for shared memory and Web Workers as the thread pool. The `wasm-bindgen-rayon` crate makes it feel like writing normal Rust parallel code — just use `par_iter()` instead of `iter()`. Remember to set COOP/COEP headers, build with `+atomics`, and only parallelize workloads that are large enough to overcome the threading overhead. Most modern browsers fully support Wasm threads as of 2024.
