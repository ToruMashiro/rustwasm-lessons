---
title: What is WebAssembly?
slug: what-is-wasm
difficulty: beginner
tags: [getting-started]
order: 1
description: Understand what WebAssembly is, why it exists, and where Rust fits in — before writing a single line of code.
starter_code: |
  // This is plain Rust — no Wasm-specific code yet.
  // It runs the same whether compiled to native or to Wasm.

  fn main() {
      let languages = ["C", "C++", "Rust", "Go", "AssemblyScript"];

      println!("Languages that compile to WebAssembly:");
      for (i, lang) in languages.iter().enumerate() {
          println!("  {}. {}", i + 1, lang);
      }

      println!("\nRust is the most popular choice because:");
      println!("  - No garbage collector (small binary)");
      println!("  - Memory safe without runtime cost");
      println!("  - First-class Wasm support via wasm-pack");
  }
expected_output: |
  Languages that compile to WebAssembly:
    1. C
    2. C++
    3. Rust
    4. Go
    5. AssemblyScript

  Rust is the most popular choice because:
    - No garbage collector (small binary)
    - Memory safe without runtime cost
    - First-class Wasm support via wasm-pack
---

## What is WebAssembly?

WebAssembly (Wasm) is a binary instruction format that runs in the browser at near-native speed. Think of it as a portable compilation target — you write code in Rust, C++, or Go, compile it to `.wasm`, and the browser executes it alongside JavaScript.

## Why does it exist?

JavaScript is powerful, but it has limits:

- **CPU-intensive tasks** (crypto, image processing, physics) are slow in JS
- **Large codebases** (games, CAD tools) need predictable performance
- **Existing code** in C/C++/Rust can't run in browsers without porting to JS

Wasm solves all three. It's not a replacement for JavaScript — it's a complement.

## How it works

```
┌──────────┐     compile      ┌──────────┐     run       ┌──────────┐
│  Rust    │ ──────────────▶ │  .wasm   │ ────────────▶ │ Browser  │
│  source  │   (wasm-pack)   │  binary  │  (JS import)  │  engine  │
└──────────┘                 └──────────┘               └──────────┘
```

1. You write Rust code
2. `wasm-pack` compiles it to a `.wasm` binary + JS glue code
3. JavaScript imports and calls your Rust functions
4. The browser runs Wasm at near-native speed

## Why Rust?

Many languages can target Wasm, but Rust is the best fit:

- **No garbage collector** — Wasm binaries are small (often < 50KB)
- **Memory safety** — compiler catches bugs at build time, not at runtime
- **Zero-cost abstractions** — high-level code compiles to fast machine code
- **First-class tooling** — `wasm-pack`, `wasm-bindgen`, and `web-sys` are mature

## What Wasm can't do (yet)

- No direct DOM access — must go through JavaScript bindings
- No threads by default — `SharedArrayBuffer` is opt-in
- No file system access — runs in a sandboxed environment
- Can't replace JavaScript — it augments it

## Try It

The code on the right is plain Rust. Click **Run** to see it execute. In the next lessons, you'll learn how to make this code run *inside the browser* via WebAssembly.
