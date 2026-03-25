---
title: How wasm-bindgen Works
slug: wasm-bindgen
difficulty: beginner
tags: [getting-started]
order: 4
description: Understand the bridge between Rust and JavaScript — how wasm-bindgen converts types, exports functions, and generates glue code.
starter_code: |
  use wasm_bindgen::prelude::*;

  // === Exporting to JavaScript ===

  // Simple function: primitives pass directly
  #[wasm_bindgen]
  pub fn add(a: i32, b: i32) -> i32 {
      a + b
  }

  // String conversion: &str (from JS) → String (to JS)
  #[wasm_bindgen]
  pub fn greet(name: &str) -> String {
      format!("Hello, {}!", name)
  }

  // Struct exported as a JS class
  #[wasm_bindgen]
  pub struct Counter {
      count: u32,
  }

  #[wasm_bindgen]
  impl Counter {
      #[wasm_bindgen(constructor)]
      pub fn new() -> Self {
          Self { count: 0 }
      }

      pub fn increment(&mut self) {
          self.count += 1;
      }

      pub fn value(&self) -> u32 {
          self.count
      }
  }

  // === Importing from JavaScript ===

  #[wasm_bindgen]
  extern "C" {
      // Import console.log
      #[wasm_bindgen(js_namespace = console)]
      fn log(s: &str);

      // Import Math.random
      #[wasm_bindgen(js_namespace = Math)]
      fn random() -> f64;
  }
expected_output: |
  // In JavaScript:
  import init, { add, greet, Counter } from './pkg/my_wasm.js';

  await init();

  add(2, 3);           // → 5
  greet("World");      // → "Hello, World!"

  const c = new Counter();
  c.increment();
  c.value();           // → 1
---

## What is wasm-bindgen?

`wasm-bindgen` is the bridge between Rust and JavaScript. WebAssembly natively only supports four numeric types: `i32`, `i64`, `f32`, and `f64`. Everything else — strings, structs, arrays, objects — needs conversion. `wasm-bindgen` generates the glue code that handles this automatically.

## The `#[wasm_bindgen]` attribute

This single attribute tells `wasm-bindgen` what to expose or import:

```rust
#[wasm_bindgen]          // on a function → export to JS
pub fn my_func() {}

#[wasm_bindgen]          // on a struct → create a JS class
pub struct MyStruct {}

#[wasm_bindgen]          // on an impl block → export methods
impl MyStruct {}

#[wasm_bindgen]          // on extern "C" → import from JS
extern "C" {
    fn alert(s: &str);
}
```

## How types cross the boundary

| Rust | JavaScript | Cost | How it works |
|------|-----------|------|-------------|
| `i32`, `f64` | `number` | Free | Native Wasm numeric types |
| `i64`, `u64` | `BigInt` | Free | Wasm i64 maps to BigInt |
| `bool` | `boolean` | Free | Represented as i32 (0/1) in Wasm |
| `&str` | `string` | Copy | JS encodes string to UTF-8 into Wasm linear memory |
| `String` | `string` | Copy | Rust writes UTF-8 bytes, JS decodes them |
| `&[u8]` | `Uint8Array` | Copy | Bytes copied between JS and Wasm memory |
| `Vec<f64>` | `Float64Array` | Copy | Bulk numeric data transfer |
| `Option<T>` | `T \| undefined` | Varies | `None` becomes `undefined` in JS |
| `Result<T, E>` | `T` or throws | Varies | `Err` becomes a JS exception |
| `JsValue` | `any` | Ref | Opaque handle to any JS value |
| `#[wasm_bindgen] struct` | `class` | Pointer | JS holds a pointer into Wasm memory |
| `Closure<dyn FnMut()>` | `Function` | Alloc | Rust closure wrapped as JS callback |

**Key insight**: Primitives are free. Strings are copied. Structs are pointers. Closures allocate.

## What wasm-pack generates

When you run `wasm-pack build`, it creates:

```
pkg/
├── my_wasm_bg.wasm       # Compiled Wasm binary
├── my_wasm_bg.wasm.d.ts  # TypeScript types for the Wasm binary
├── my_wasm.js            # JavaScript glue code (handles type conversion)
├── my_wasm.d.ts          # TypeScript types for the public API
└── package.json          # Ready to publish to npm
```

The `.js` glue file handles:
- Loading and instantiating the `.wasm` binary
- Encoding/decoding strings between JS (UTF-16) and Wasm (UTF-8)
- Managing Wasm memory allocation for strings and arrays
- Preventing use-after-free for exported structs
- Providing TypeScript-friendly APIs with full type safety

## Importing JavaScript functions

You can call any JavaScript API from Rust using `extern "C"` blocks:

```rust
#[wasm_bindgen]
extern "C" {
    // window.alert()
    fn alert(s: &str);

    // console.log() — specify the JS namespace
    #[wasm_bindgen(js_namespace = console)]
    fn log(s: &str);

    // Rename: Rust name differs from JS name
    #[wasm_bindgen(js_name = "setTimeout")]
    fn set_timeout(closure: &Closure<dyn FnMut()>, ms: u32);

    // Import a global variable
    #[wasm_bindgen(js_name = "document")]
    static DOCUMENT: web_sys::Document;
}
```

## Common attribute patterns

### Constructor — lets JS use `new MyStruct()`

```rust
#[wasm_bindgen(constructor)]
pub fn new() -> Self { Self { count: 0 } }
```

### Getter / Setter — lets JS use `obj.name`

```rust
#[wasm_bindgen(getter)]
pub fn name(&self) -> String { self.name.clone() }

#[wasm_bindgen(setter)]
pub fn set_name(&mut self, name: &str) { self.name = name.to_string(); }
```

### js_name — rename for JS convention

```rust
#[wasm_bindgen(js_name = "calculateTotal")]
pub fn calculate_total(&self) -> f64 { /* ... */ }
```

### skip — hide a public method from JS

```rust
#[wasm_bindgen(skip)]
pub fn internal_method(&self) { /* not exported */ }
```

## Cargo.toml setup

Every Rust/Wasm project needs these dependencies:

```toml
[lib]
crate-type = ["cdylib"]

[dependencies]
wasm-bindgen = "0.2"
# Optional: browser API bindings
web-sys = { version = "0.3", features = ["Window", "Document"] }
# Optional: JS built-in types
js-sys = "0.3"
# Optional: async/await support
wasm-bindgen-futures = "0.4"
```

## Try It

The starter code shows the three main patterns: exporting functions, exporting structs, and importing JS functions. These three patterns cover 90% of real-world Wasm usage.
