---
title: Rust Basics for Wasm
slug: rust-basics
difficulty: beginner
tags: [getting-started]
order: 2
description: Learn the essential Rust syntax you need for WebAssembly — functions, types, structs, and error handling. No prior Rust experience needed.
starter_code: |
  // === Functions and Types ===
  fn add(a: i32, b: i32) -> i32 {
      a + b  // no semicolon = return value
  }

  // === Structs ===
  struct Point {
      x: f64,
      y: f64,
  }

  impl Point {
      fn new(x: f64, y: f64) -> Self {
          Self { x, y }
      }

      fn distance(&self, other: &Point) -> f64 {
          ((self.x - other.x).powi(2) + (self.y - other.y).powi(2)).sqrt()
      }
  }

  // === Strings ===
  fn greet(name: &str) -> String {
      format!("Hello, {}!", name)
  }

  // === Option and Result ===
  fn safe_divide(a: f64, b: f64) -> Option<f64> {
      if b == 0.0 {
          None
      } else {
          Some(a / b)
      }
  }

  fn main() {
      // Functions
      println!("2 + 3 = {}", add(2, 3));

      // Structs
      let p1 = Point::new(0.0, 0.0);
      let p2 = Point::new(3.0, 4.0);
      println!("Distance: {}", p1.distance(&p2));

      // Strings
      println!("{}", greet("Rustacean"));

      // Option
      match safe_divide(10.0, 3.0) {
          Some(result) => println!("10 / 3 = {:.2}", result),
          None => println!("Cannot divide by zero!"),
      }
  }
expected_output: |
  2 + 3 = 5
  Distance: 5
  Hello, Rustacean!
  10 / 3 = 3.33
---

## Why learn Rust basics?

You don't need to master all of Rust to use WebAssembly. This lesson covers the **minimal subset** you'll use in every Wasm project.

## Functions

Rust functions use `fn`, explicit types, and the last expression (without `;`) is the return value:

```rust
fn multiply(a: i32, b: i32) -> i32 {
    a * b  // returned implicitly
}
```

## Imports: `use`, `::`, and `*`

You'll see this line at the top of every Wasm file:

```rust
use wasm_bindgen::prelude::*;
```

Let's break it down piece by piece:

### `use` — bring items into scope

`use` is like `import` in JavaScript. Without it, you'd have to write the full path every time:

```rust
// Without use: verbose
wasm_bindgen::prelude::wasm_bindgen

// With use: clean
use wasm_bindgen::prelude::*;
#[wasm_bindgen]  // now available directly
```

### `::` — path separator

`::` navigates through modules (like `/` in a file path or `.` in JavaScript):

```rust
// crate::module::submodule::item
use wasm_bindgen::prelude::*;
//  ^^^^^^^^^^^^   ^^^^^^^   ^
//  crate name     module    everything in it

use std::collections::HashMap;
//  ^^^  ^^^^^^^^^^^  ^^^^^^^
//  std   module      specific type

use web_sys::Document;
//  ^^^^^^^  ^^^^^^^^
//  crate    specific type
```

Think of it as: `crate_name::module::Item`

### `*` — glob import (everything)

`*` imports all public items from a module:

```rust
use wasm_bindgen::prelude::*;  // imports everything from prelude
```

This is common for "prelude" modules — they're curated sets of the most-used items. `wasm_bindgen::prelude::*` gives you:
- `#[wasm_bindgen]` attribute
- `JsValue` type
- `JsCast` trait (for `dyn_into()`)
- And other essentials

You can also import specific items instead:

```rust
// Import only what you need
use wasm_bindgen::prelude::wasm_bindgen;
use wasm_bindgen::JsValue;

// Or group imports with braces
use web_sys::{Document, Element, Window};
//            ^^^^^^^^  ^^^^^^^  ^^^^^^
//            three specific types from web_sys
```

### Common import patterns in Wasm

```rust
// The essentials (in almost every file)
use wasm_bindgen::prelude::*;

// Specific web APIs
use web_sys::{Document, Element, HtmlCanvasElement};

// JavaScript built-in types
use js_sys::{Array, Object, JSON};

// Closures for event handlers
use wasm_bindgen::closure::Closure;

// Async support
use wasm_bindgen_futures::JsFuture;
```

## Complete Wasm Type Reference

These are all the types you can pass between Rust and JavaScript through `wasm-bindgen`:

### Primitives (zero-cost — passed directly through Wasm)

| Rust | JavaScript | Notes |
|------|-----------|-------|
| `i8`, `i16`, `i32` | `number` | Signed integers |
| `u8`, `u16`, `u32` | `number` | Unsigned integers |
| `i64`, `u64` | `BigInt` | 64-bit integers map to `BigInt`, not `number` |
| `isize`, `usize` | `number` | Platform-sized integers (32-bit in Wasm) |
| `f32`, `f64` | `number` | Floating point |
| `bool` | `boolean` | |
| `char` | `string` | Single Unicode character |

### Strings (copied across the Wasm boundary)

| Rust | JavaScript | Direction | Notes |
|------|-----------|-----------|-------|
| `&str` | `string` | JS → Rust | Borrowed, read-only. JS string is encoded to UTF-8 |
| `String` | `string` | Rust → JS | Owned. UTF-8 bytes decoded back to JS string |

### Typed Arrays (copied or zero-copy via views)

| Rust | JavaScript | Notes |
|------|-----------|-------|
| `Vec<u8>`, `&[u8]` | `Uint8Array` | Byte buffers, file data |
| `Vec<i8>`, `&[i8]` | `Int8Array` | |
| `Vec<u16>`, `&[u16]` | `Uint16Array` | |
| `Vec<i16>`, `&[i16]` | `Int16Array` | |
| `Vec<u32>`, `&[u32]` | `Uint32Array` | |
| `Vec<i32>`, `&[i32]` | `Int32Array` | |
| `Vec<f32>`, `&[f32]` | `Float32Array` | Graphics, audio data |
| `Vec<f64>`, `&[f64]` | `Float64Array` | Physics simulations |
| `Vec<u64>`, `&[u64]` | `BigUint64Array` | |
| `Vec<i64>`, `&[i64]` | `BigInt64Array` | |

### Complex Types

| Rust | JavaScript | Notes |
|------|-----------|-------|
| `JsValue` | `any` | Catch-all for any JS value |
| `Option<T>` | `T \| undefined` | `None` becomes `undefined` |
| `Result<T, JsValue>` | `T` (throws on `Err`) | Errors become JS exceptions |
| `#[wasm_bindgen] struct` | `class` | Exported as a JS class, pointer-based (no copy) |
| `Box<[JsValue]>` | `Array` | JS array of any values |
| `Closure<dyn FnMut()>` | `Function` | Rust closures as JS callbacks |

### js-sys Built-in Types

| Rust (`js_sys::`) | JavaScript | Notes |
|-------------------|-----------|-------|
| `js_sys::Array` | `Array` | Full Array API |
| `js_sys::Object` | `Object` | Generic JS object |
| `js_sys::Map` | `Map` | |
| `js_sys::Set` | `Set` | |
| `js_sys::Date` | `Date` | |
| `js_sys::RegExp` | `RegExp` | |
| `js_sys::Promise` | `Promise` | Use with `wasm-bindgen-futures` |
| `js_sys::Function` | `Function` | Callable JS function |
| `js_sys::Uint8Array` | `Uint8Array` | Zero-copy view into Wasm memory |

## Structs

Structs are Rust's version of classes. You'll export these to JavaScript as Wasm objects:

```rust
struct GameState {
    score: u32,
    level: u32,
}

impl GameState {
    fn new() -> Self {
        Self { score: 0, level: 1 }
    }

    fn add_points(&mut self, points: u32) {
        self.score += points;
    }
}
```

## Ownership & Borrowing (simplified)

The one Rust concept that trips people up. For Wasm, you need to know:

- **`&str`** — borrow a string (read-only, cheap)
- **`String`** — own a string (can modify, must allocate)
- **`&self`** — method borrows the struct (read-only)
- **`&mut self`** — method borrows the struct (can modify)

```rust
fn print_name(name: &str) {     // borrows, doesn't own
    println!("Name: {}", name);
}

fn make_greeting(name: &str) -> String {  // returns owned String
    format!("Hello, {}!", name)
}
```

## Error handling: Option and Result

Wasm functions that can fail should return `Result<T, JsValue>`:

```rust
// Option: value might not exist
fn find_user(id: u32) -> Option<String> {
    if id == 1 { Some("Alice".to_string()) } else { None }
}

// Result: operation might fail (in Wasm, use JsValue for errors)
fn parse_number(s: &str) -> Result<i32, String> {
    s.parse::<i32>().map_err(|e| e.to_string())
}
```

## Try It

Click **Run** to see all the examples in action. Try modifying the code — add a new method to `Point` or change the `greet` function.
