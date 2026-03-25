---
title: Wasm + Web Workers
slug: web-workers
difficulty: advanced
tags: [concurrency]
order: 19
description: Run Wasm in Web Workers for parallel computation without blocking the main thread — multi-threaded performance in the browser.
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
  All computed without blocking the UI!
---

## Why Web Workers?

Wasm runs on the main thread by default. Heavy computation blocks the UI — buttons freeze, animations stop. Web Workers let you run Wasm on a **separate thread**.

```
Main Thread (UI)              Worker Thread (Computation)
┌──────────────┐              ┌──────────────┐
│ DOM, events  │  postMessage │ Wasm module  │
│ animations   │ ◀──────────▶│ heavy math   │
│ responsive   │              │ doesn't block│
└──────────────┘              └──────────────┘
```

## Basic Worker Setup

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

// This doesn't block the UI
worker.postMessage({
    type: 'compute',
    data: { iterations: 10_000_000 }
});
```

## Multiple Workers (Parallel)

Split work across multiple Workers for true parallelism:

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

## SharedArrayBuffer (Advanced)

Share memory between Workers without copying:

```js
// Requires these headers on your server:
// Cross-Origin-Opener-Policy: same-origin
// Cross-Origin-Embedder-Policy: require-corp

const shared = new SharedArrayBuffer(1024);
const view = new Float64Array(shared);

// Both main thread and worker can read/write
worker.postMessage({ buffer: shared });
```

```rust
// In Rust, access shared memory via pointer
#[wasm_bindgen]
pub fn process_shared(ptr: *mut f64, len: usize) {
    let data = unsafe { std::slice::from_raw_parts_mut(ptr, len) };
    for x in data.iter_mut() {
        *x *= 2.0;
    }
}
```

## When to Use Web Workers

| Use Case | Main Thread | Web Worker |
|----------|-------------|------------|
| DOM manipulation | ✓ | ✗ (no DOM access) |
| Event handlers | ✓ | ✗ |
| Heavy computation | ✗ (blocks UI) | ✓ |
| Image processing | ✗ | ✓ |
| Crypto operations | ✗ | ✓ |
| Physics simulation | ✗ | ✓ |
| Data parsing | Depends on size | ✓ for large files |

## Performance Comparison

| Task | Main Thread | 1 Worker | 4 Workers |
|------|------------|----------|-----------|
| Count primes to 1M | 800ms (UI frozen) | 800ms (UI responsive) | 200ms |
| Image blur (4K) | 120ms (UI frozen) | 120ms (UI responsive) | 35ms |

Workers don't make individual tasks faster — they keep the UI responsive and enable parallelism.

## Try It

The starter code shows CPU-intensive functions (trigonometric computation, prime counting) that would block the UI if run on the main thread. In production, wrap these in a Web Worker.
