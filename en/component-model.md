---
title: Wasm Component Model
slug: component-model
difficulty: advanced
tags: [api]
order: 28
description: The future of WebAssembly — composable modules, typed interfaces, and language-agnostic components that work together.
starter_code: |
  fn main() {
      println!("=== Wasm Component Model ===\n");

      println!("Current Wasm (Core Modules):");
      println!("  • Only numeric types (i32, i64, f32, f64)");
      println!("  • Strings passed as ptr + len (manual)");
      println!("  • Each module is standalone");
      println!("  • Language-specific glue code needed\n");

      println!("Component Model (the future):");
      println!("  • Rich types: strings, lists, records, variants");
      println!("  • Typed interfaces (WIT files)");
      println!("  • Modules can import/export components");
      println!("  • Language-agnostic — Rust, Go, Python, JS");
      println!("  • Package manager (warg registry)\n");

      println!("WIT (Wasm Interface Type) example:");
      println!("  interface my-api {{");
      println!("    record user {{");
      println!("      name: string,");
      println!("      age: u32,");
      println!("    }}");
      println!("    greet: func(user: user) -> string");
      println!("  }}\n");

      println!("Tools:");
      println!("  • cargo-component — build Wasm components from Rust");
      println!("  • wasm-tools     — manipulate Wasm binaries");
      println!("  • wasmtime        — runtime with component support");
      println!("  • jco             — JS tooling for components");
  }
expected_output: |
  === Wasm Component Model ===

  Current Wasm (Core Modules):
    • Only numeric types (i32, i64, f32, f64)
    • Strings passed as ptr + len (manual)
    • Each module is standalone
    • Language-specific glue code needed

  Component Model (the future):
    • Rich types: strings, lists, records, variants
    • Typed interfaces (WIT files)
    • Modules can import/export components
    • Language-agnostic — Rust, Go, Python, JS
    • Package manager (warg registry)

  WIT (Wasm Interface Type) example:
    interface my-api {
      record user {
        name: string,
        age: u32,
      }
      greet: func(user: user) -> string
    }

  Tools:
    • cargo-component — build Wasm components from Rust
    • wasm-tools     — manipulate Wasm binaries
    • wasmtime        — runtime with component support
    • jco             — JS tooling for components
---

## The Problem with Current Wasm

Today's WebAssembly only supports 4 types: `i32`, `i64`, `f32`, `f64`. Everything else (strings, structs, arrays) requires manual encoding:

```
Current: Rust String → UTF-8 bytes → pointer + length → JS decode
Future:  Rust String → string (native Wasm type) → JS string
```

`wasm-bindgen` solves this for Rust↔JS, but it's **Rust-specific**. A Go module can't call a Rust module's functions without custom glue.

## What the Component Model Fixes

| Problem | Current | Component Model |
|---------|---------|----------------|
| Types | Only i32/i64/f32/f64 | Strings, lists, records, variants, options, results |
| Interop | Language-specific glue | Language-agnostic WIT interfaces |
| Composition | Monolithic modules | Composable components that import/export |
| Dependencies | Manual linking | Package registry (warg) |

## WIT: WebAssembly Interface Types

WIT is the interface definition language for components:

```wit
// my-api.wit
package example:my-api;

interface types {
    record user {
        name: string,
        age: u32,
        email: option<string>,
    }

    enum role {
        admin,
        editor,
        viewer,
    }

    variant error {
        not-found(string),
        permission-denied,
        internal(string),
    }
}

interface users {
    use types.{user, role, error};

    create-user: func(name: string, age: u32) -> result<user, error>;
    get-user: func(id: u32) -> option<user>;
    list-users: func() -> list<user>;
}

world my-app {
    import wasi:http/outgoing-handler;
    export users;
}
```

## WIT Types

| WIT type | Description | Example |
|----------|------------|---------|
| `string` | UTF-8 string | `"hello"` |
| `bool` | Boolean | `true` |
| `u8`..`u64`, `s8`..`s64` | Integers | `42` |
| `f32`, `f64` | Floats | `3.14` |
| `list<T>` | Dynamic array | `list<string>` |
| `option<T>` | Nullable | `option<u32>` |
| `result<T, E>` | Success or error | `result<user, error>` |
| `record` | Named fields (struct) | `record { name: string }` |
| `variant` | Tagged union (enum) | `variant { a(u32), b }` |
| `tuple<T, U>` | Anonymous tuple | `tuple<string, u32>` |

## Building a Component with Rust

```bash
# Install cargo-component
cargo install cargo-component

# Create a new component project
cargo component new my-component
cd my-component
```

```rust
// src/lib.rs
#[allow(warnings)]
mod bindings;

use bindings::Guest;

struct Component;

impl Guest for Component {
    fn greet(name: String) -> String {
        format!("Hello, {}!", name)
    }
}

bindings::export!(Component with_types_in bindings);
```

```bash
# Build the component
cargo component build --release

# The output is a .wasm component (not a core module)
```

## Composing Components

Multiple components can be composed together:

```
┌─────────────────┐     ┌─────────────────┐
│  Auth Component  │────▶│  API Component   │
│  (Rust)          │     │  (Go)            │
└─────────────────┘     └────────┬────────┘
                                 │
                        ┌────────▼────────┐
                        │  DB Component    │
                        │  (Python)        │
                        └─────────────────┘
```

Each component:
- Exposes a typed WIT interface
- Can be written in **any language**
- Is sandboxed — can only access what it imports
- Can be replaced without changing other components

## Timeline

| Status | Feature |
|--------|---------|
| ✓ Stable | Core Wasm (modules, memory, functions) |
| ✓ Stable | WASI preview 1 (file I/O, env, args) |
| ✓ Stable | Component Model specification |
| ✓ Available | cargo-component, jco, wasmtime support |
| 🔄 In progress | WASI preview 2 (HTTP, sockets, key-value) |
| 🔄 In progress | warg package registry |
| 📋 Planned | Browser support for components |
| 📋 Planned | Component linking in bundlers |

## Why This Matters

The Component Model turns Wasm from "fast code in the browser" into a **universal software component format**:

- Write a Rust image processor → use it from Python, Go, JS
- Build a plugin system → users write plugins in any language
- Microservices → each service is a sandboxed Wasm component
- Edge computing → compose components at the CDN edge

## Try It

Click **Run** to see the Component Model overview — what it provides, WIT syntax, and the tools available. This is the future direction of WebAssembly.
