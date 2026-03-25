---
title: Wasm Size Optimization
slug: wasm-size-optimization
difficulty: intermediate
tags: [getting-started]
order: 34
description: Learn techniques to shrink your Wasm binary — from Cargo.toml tweaks and wasm-opt to tree shaking, avoiding bloat, and measuring with twiggy.
starter_code: |
  // Demonstrating size-conscious Rust patterns
  // (Plain Rust — concepts apply to Wasm builds)

  use std::mem;

  // --- Technique 1: Prefer fixed-size types over dynamic ones ---
  struct CompactPoint {
      x: i16,
      y: i16,
  }

  struct HeavyPoint {
      x: f64,
      y: f64,
      label: String,  // String pulls in allocator + format machinery
  }

  // --- Technique 2: Avoid format! / to_string() in hot paths ---
  fn status_code_message(code: u16) -> &'static str {
      match code {
          200 => "OK",
          404 => "Not Found",
          500 => "Internal Server Error",
          _   => "Unknown",
      }
  }

  // --- Technique 3: Use enums instead of strings for state ---
  #[repr(u8)]
  enum Color {
      Red   = 0,
      Green = 1,
      Blue  = 2,
  }

  // --- Technique 4: Manual itoa instead of format!("{}", n) ---
  fn u32_to_buf(mut n: u32, buf: &mut [u8; 10]) -> usize {
      if n == 0 {
          buf[0] = b'0';
          return 1;
      }
      let mut i = 0;
      while n > 0 {
          buf[i] = b'0' + (n % 10) as u8;
          n /= 10;
          i += 1;
      }
      buf[..i].reverse();
      i
  }

  fn main() {
      // Size comparison
      println!("CompactPoint size: {} bytes", mem::size_of::<CompactPoint>());
      println!("HeavyPoint  size: {} bytes", mem::size_of::<HeavyPoint>());
      println!("Color enum  size: {} bytes", mem::size_of::<Color>());

      // Static strings — no allocator needed
      println!("HTTP 200 -> {}", status_code_message(200));
      println!("HTTP 404 -> {}", status_code_message(404));

      // Manual number conversion
      let mut buf = [0u8; 10];
      let len = u32_to_buf(42, &mut buf);
      let s = std::str::from_utf8(&buf[..len]).unwrap();
      println!("u32_to_buf(42) -> \"{}\"", s);

      // Show that repr(u8) enums are 1 byte
      println!("Color::Red   as u8 = {}", Color::Red as u8);
      println!("Color::Green as u8 = {}", Color::Green as u8);
  }
expected_output: |
  CompactPoint size: 4 bytes
  HeavyPoint  size: 40 bytes
  Color enum  size: 1 bytes
  HTTP 200 -> OK
  HTTP 404 -> Not Found
  u32_to_buf(42) -> "42"
  Color::Red   as u8 = 0
  Color::Green as u8 = 1
---

## Why Binary Size Matters

Every byte of your `.wasm` file must be downloaded, parsed, and compiled by the browser before your application starts. Unlike JavaScript, Wasm is a compact binary format — but a naive Rust build can still produce multi-megabyte files.

```
                    Download        Parse         Compile
  ┌──────┐       ┌──────────┐    ┌─────────┐    ┌─────────┐
  │ .wasm│──────>│ Network  │───>│ Decoder │───>│ JIT/AOT │──> Run
  │ file │       │ transfer │    │         │    │ compile │
  └──────┘       └──────────┘    └─────────┘    └─────────┘
     ^                ^
     │                │
   Smaller file = faster startup everywhere
```

| File Size | 3G (2 Mbps) | 4G (20 Mbps) | Wi-Fi (50 Mbps) |
|-----------|-------------|--------------|-----------------|
| 100 KB    | 0.4 s       | 0.04 s       | 0.02 s          |
| 500 KB    | 2.0 s       | 0.2 s        | 0.08 s          |
| 2 MB      | 8.0 s       | 0.8 s        | 0.32 s          |
| 10 MB     | 40.0 s      | 4.0 s        | 1.6 s           |

## Cargo.toml Release Profile

The single most impactful change is tuning your `[profile.release]` section:

```toml
[profile.release]
opt-level = "z"       # Optimize for size (s = size, z = even smaller)
lto = true            # Link-Time Optimization — whole-program analysis
codegen-units = 1     # Single codegen unit = better inlining / dead code
strip = true          # Remove debug symbols
panic = "abort"       # No unwinding machinery (saves ~10-20 KB)
```

### What Each Setting Does

```
  opt-level = "z"
  ┌──────────────────────────────────────────────────────┐
  │ "0" — no optimization          (debug builds)        │
  │ "1" — basic optimization                             │
  │ "2" — standard optimization    (default release)     │
  │ "3" — aggressive speed optimization                  │
  │ "s" — optimize for binary size                       │
  │ "z" — optimize even harder for size                  │
  └──────────────────────────────────────────────────────┘

  lto = true
  ┌────────────────────────────────────────┐
  │ Without LTO:                           │
  │   crate_a.o ──┐                        │
  │   crate_b.o ──┼── link ── binary       │
  │   crate_c.o ──┘  (dead code remains)   │
  │                                        │
  │ With LTO:                              │
  │   crate_a.o ──┐                        │
  │   crate_b.o ──┼── analyze ── optimize  │
  │   crate_c.o ──┘  all at     ── binary  │
  │                   once       (smaller!) │
  └────────────────────────────────────────┘

  codegen-units = 1
  ┌─────────────────────────────────────────────────┐
  │ Default: 16 units — faster compile, less opt.   │
  │ Setting 1: single unit — slower compile, but    │
  │ the compiler sees everything and can inline     │
  │ and eliminate more dead code.                   │
  └─────────────────────────────────────────────────┘
```

## wasm-opt: The Post-Build Optimizer

Binaryen's `wasm-opt` performs Wasm-specific optimizations the Rust compiler cannot:

```bash
# Install
cargo install wasm-opt
# or: npm install -g binaryen

# Optimize for size (-Oz) or speed (-O3)
wasm-opt -Oz -o output.wasm input.wasm
```

Typical savings from `wasm-opt -Oz`:

| Stage               | Size     |
|---------------------|----------|
| After `cargo build` | 180 KB   |
| After `wasm-opt`    | 120 KB   |
| After gzip          | 45 KB    |
| After brotli        | 38 KB    |

## Tree Shaking and Dead Code Elimination

Rust's monomorphization can create many copies of generic functions. Each instantiation of `Vec<T>` for a different `T` generates separate machine code.

```
  Your code uses:          Monomorphized into:
  Vec<u8>          ──────> Vec_u8_push, Vec_u8_pop, Vec_u8_len ...
  Vec<String>      ──────> Vec_String_push, Vec_String_pop ...
  Vec<(i32,i32)>   ──────> Vec_tuple_push, Vec_tuple_pop ...
```

### Strategies to reduce monomorphization bloat:

1. **Use trait objects** instead of generics where performance is not critical:
```rust
// Generates code per type:
fn process<T: Display>(items: &[T]) { ... }

// Single generated function:
fn process(items: &[&dyn Display]) { ... }
```

2. **Factor non-generic code** out of generic functions:
```rust
// BAD — entire function duplicated per T
fn insert<T: Ord>(vec: &mut Vec<T>, item: T) {
    let idx = vec.binary_search(&item).unwrap_or_else(|i| i);
    vec.insert(idx, item);
    log_size(vec.len()); // This line doesn't use T
}

// BETTER — log_size is compiled once
fn log_size(n: usize) { /* ... */ }
fn insert<T: Ord>(vec: &mut Vec<T>, item: T) {
    let idx = vec.binary_search(&item).unwrap_or_else(|i| i);
    vec.insert(idx, item);
    log_size(vec.len());
}
```

## Avoiding Hidden Bloat

### The `format!` and `panic!` Problem

Every `format!()`, `panic!()`, `unwrap()`, and `expect()` call pulls in Rust's formatting machinery — roughly **10–20 KB** of code.

```rust
// BAD — pulls in Display + formatting for the panic string
fn get(idx: usize) -> u8 {
    DATA[idx]  // panics with full formatting on OOB
}

// BETTER — use get() + a tiny abort
fn get(idx: usize) -> u8 {
    match DATA.get(idx) {
        Some(&v) => v,
        None => abort_with_code(1),
    }
}
```

### String Bloat Checklist

| Pattern                        | Adds to binary?         | Alternative                  |
|--------------------------------|-------------------------|------------------------------|
| `format!("x = {}", x)`        | Yes — formatting infra  | Manual conversion or &str    |
| `panic!("bad index {}", i)`    | Yes — panic + format    | `unreachable_unchecked()`*   |
| `.unwrap()`                    | Yes — panic strings     | `.unwrap_or(default)`        |
| `.expect("msg")`              | Yes — panic + string    | Match + abort                |
| `#[derive(Debug)]` on types   | Yes — Debug impls       | Only derive when needed      |
| `to_string()` on numbers      | Yes — Display trait      | itoa crate or manual buf     |

*`unreachable_unchecked` is unsafe and should only be used when you can guarantee the path is unreachable.

## wee_alloc: A Tiny Allocator

The default Rust allocator (dlmalloc in Wasm) adds ~10 KB. `wee_alloc` trades performance for size (~1 KB):

```rust
// In your lib.rs
#[global_allocator]
static ALLOC: wee_alloc::WeeAlloc = wee_alloc::WeeAlloc::INIT;
```

```toml
[dependencies]
wee_alloc = "0.4"
```

Trade-offs:

| Property          | dlmalloc     | wee_alloc    |
|-------------------|-------------|-------------|
| Binary overhead   | ~10 KB      | ~1 KB       |
| Alloc speed       | Fast        | Slower      |
| Fragmentation     | Good        | Poor        |
| Free reclaims?    | Yes         | Partially   |
| Best for          | General use | Small Wasm  |

> **Note**: `wee_alloc` is no longer actively maintained. For production use, consider the `lol_alloc` or `talc` crates, or write your own bump allocator (see Lesson 35).

## Measuring with twiggy

`twiggy` analyzes your `.wasm` binary to show what takes up space:

```bash
# Install
cargo install twiggy

# Show the top 20 largest items
twiggy top -n 20 my_app_bg.wasm

# Show a call graph of what retains what
twiggy dominators my_app_bg.wasm

# Diff two builds
twiggy diff old.wasm new.wasm
```

Example `twiggy top` output:
```
 Shallow Bytes │ Shallow % │ Item
───────────────┼───────────┼────────────────────────────
         12480 │    8.31%  │ data[0]
          6204 │    4.13%  │ "function names" subsection
          3120 │    2.08%  │ core::fmt::write
          2800 │    1.86%  │ dlmalloc::dlmalloc::Dlmalloc::malloc
          2476 │    1.65%  │ core::fmt::Formatter::pad
```

## Compression: The Final Step

After all optimizations, apply HTTP compression. Most servers support gzip and Brotli:

```
  Raw .wasm                gzip (-9)              Brotli (-11)
  ┌─────────┐            ┌─────────┐            ┌─────────┐
  │ 120 KB  │  ────────> │  45 KB  │  ────────> │  38 KB  │
  └─────────┘    ~62%    └─────────┘    ~15%    └─────────┘
                 saving                  more
```

Configure your web server:
```nginx
# Nginx
location ~ \.wasm$ {
    gzip on;
    gzip_types application/wasm;
    # Or serve pre-compressed files:
    gzip_static on;
    brotli_static on;
}
```

## Complete Optimization Pipeline

```
  cargo build --release     ← Cargo.toml profile settings
        │
        ▼
  wasm-opt -Oz              ← Binaryen post-processing
        │
        ▼
  wasm-strip                ← Remove names section (optional)
        │
        ▼
  brotli / gzip             ← HTTP transfer compression
        │
        ▼
  ✓ Minimal .wasm served to browser
```

### Quick Reference: Impact by Technique

| Technique                        | Typical Savings |
|----------------------------------|-----------------|
| `opt-level = "z"`               | 10–25%          |
| `lto = true`                    | 10–20%          |
| `codegen-units = 1`             | 5–10%           |
| `panic = "abort"`               | 5–15%           |
| `strip = true`                  | 5–15%           |
| `wasm-opt -Oz`                  | 15–30%          |
| Replace `wee_alloc`             | ~10 KB flat     |
| Remove `format!` / Debug derives | varies widely   |
| Brotli compression              | 60–70%          |

## Summary

Binary size optimization is a spectrum — you don't need every technique for every project. Start with Cargo.toml settings and `wasm-opt` (the 80/20 wins), then use `twiggy` to find remaining bloat. For sub-50 KB targets, you'll need to avoid the formatting infrastructure and consider a custom allocator.
