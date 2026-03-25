---
title: Hello WebAssembly
slug: hello-wasm
difficulty: beginner
tags: [getting-started]
order: 3
description: Your first Rust + WebAssembly project. Learn how wasm-bindgen bridges Rust and JavaScript.
starter_code: |
  use wasm_bindgen::prelude::*;

  // Export a function to JavaScript
  #[wasm_bindgen]
  pub fn greet(name: &str) -> String {
      format!("Hello, {}! Welcome to WebAssembly.", name)
  }

  // Export a simple math function
  #[wasm_bindgen]
  pub fn add(a: i32, b: i32) -> i32 {
      a + b
  }
expected_output: |
  Hello, World! Welcome to WebAssembly.
  2 + 3 = 5
---

## Introduction

WebAssembly (Wasm) lets you run compiled code in the browser at near-native speed. Rust is one of the best languages for targeting Wasm because of its zero-cost abstractions, no garbage collector, and small binary sizes.

## How It Works

1. You write Rust code and annotate functions with `#[wasm_bindgen]`
2. The `wasm-pack` tool compiles your Rust code to a `.wasm` binary
3. JavaScript can import and call your exported functions directly

## Key Concepts

- **`#[wasm_bindgen]`** — marks functions to be accessible from JavaScript
- **`&str`** — Rust's string slice type, automatically converted from JS strings
- **Return types** — Rust types like `String`, `i32`, `f64` are converted to JS equivalents

## Calling from JavaScript

Once compiled with `wasm-pack`, you can import and call your Rust functions directly:

```js
import init, { greet, add } from './pkg/hello_wasm.js';

async function run() {
    await init();
    console.log(greet("World"));
    console.log("2 + 3 =", add(2, 3));
}
run();
```

## Why two imports? `init` vs `{ greet, add }`

Every Wasm module has **two kinds of imports**:

```js
import init, { greet, add } from './pkg/hello_wasm.js';
//     ^^^^  ^^^^^^^^^^^^^^
//     │     └── Named exports: your Rust functions and structs
//     └── Default export: the Wasm initializer
```

**`init`** (default export):
- Downloads and compiles the `.wasm` binary file
- Must be called **once** before using any Rust function
- Returns a `Promise` — use `await init()` or `.then()`
- After this call, all named exports are ready to use

**`{ greet, add }`** (named exports):
- Your `#[wasm_bindgen] pub fn` functions
- Your `#[wasm_bindgen] pub struct` types (as JS classes)
- These are **not usable** until `init()` completes

This is why Wasm code always follows this pattern:

```js
// Step 1: Initialize (downloads + compiles .wasm)
await init();

// Step 2: Now use your Rust functions
greet("World");    // ✓ works
add(2, 3);         // ✓ works
```

If you try to call a function before `init()`:

```js
greet("World");    // ✗ Error: Wasm not initialized
await init();
greet("World");    // ✓ Now it works
```

## Try It

Modify the `greet` function to include a personalized message, or add a new exported function like `multiply`.
