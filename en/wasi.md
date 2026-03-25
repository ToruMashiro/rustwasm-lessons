---
title: Wasm Outside the Browser (WASI)
slug: wasi
difficulty: advanced
tags: [api]
order: 25
description: Run WebAssembly outside the browser with WASI — server-side Wasm, edge computing, and portable binaries.
starter_code: |
  fn main() {
      println!("=== WASI (WebAssembly System Interface) ===\n");

      println!("What WASI provides:");
      let capabilities = [
          ("File I/O", "Read/write files via capability-based security"),
          ("Env vars", "Access environment variables"),
          ("Args", "Read command-line arguments"),
          ("Clocks", "Wall clock and monotonic timers"),
          ("Random", "Cryptographic random numbers"),
          ("Stdout/Stderr", "Standard output streams"),
      ];

      for (cap, desc) in &capabilities {
          println!("  • {}: {}", cap, desc);
      }

      println!("\nRuntimes that support WASI:");
      let runtimes = ["Wasmtime", "Wasmer", "WasmEdge", "Spin", "Cloudflare Workers"];
      for rt in &runtimes {
          println!("  - {}", rt);
      }

      println!("\nCompile for WASI:");
      println!("  rustup target add wasm32-wasip1");
      println!("  cargo build --target wasm32-wasip1 --release");
      println!("  wasmtime ./target/wasm32-wasip1/release/my_app.wasm");
  }
expected_output: |
  === WASI (WebAssembly System Interface) ===

  What WASI provides:
    • File I/O: Read/write files via capability-based security
    • Env vars: Access environment variables
    • Args: Read command-line arguments
    • Clocks: Wall clock and monotonic timers
    • Random: Cryptographic random numbers
    • Stdout/Stderr: Standard output streams

  Runtimes that support WASI:
    - Wasmtime
    - Wasmer
    - WasmEdge
    - Spin
    - Cloudflare Workers

  Compile for WASI:
    rustup target add wasm32-wasip1
    cargo build --target wasm32-wasip1 --release
    wasmtime ./target/wasm32-wasip1/release/my_app.wasm
---

## What is WASI?

WASI (WebAssembly System Interface) lets Wasm run **outside the browser** — on servers, edge networks, IoT devices, and CLIs. It provides a standardized API for system capabilities that browsers don't need (file I/O, networking, etc.).

```
Browser Wasm                    WASI Wasm
┌───────────────┐               ┌───────────────┐
│ Web APIs      │               │ System APIs   │
│ (DOM, Canvas, │               │ (Files, Net,  │
│  Fetch, etc.) │               │  Env, Clock)  │
├───────────────┤               ├───────────────┤
│ Wasm Runtime  │               │ Wasm Runtime  │
│ (V8, Spider-  │               │ (Wasmtime,    │
│  Monkey)      │               │  Wasmer)      │
└───────────────┘               └───────────────┘
```

## Two Wasm Targets

| Target | Where it runs | APIs |
|--------|--------------|------|
| `wasm32-unknown-unknown` | Browser | Web APIs via web-sys |
| `wasm32-wasip1` | Server/CLI | WASI (files, env, args) |

## Getting Started

```bash
# Add the WASI target
rustup target add wasm32-wasip1

# Install a WASI runtime
cargo install wasmtime-cli

# Create a project
cargo new --bin my-wasi-app
cd my-wasi-app
```

```rust
// src/main.rs — works on WASI just like native Rust
use std::fs;
use std::env;

fn main() {
    // Command-line args
    let args: Vec<String> = env::args().collect();
    println!("Args: {:?}", args);

    // File I/O (sandboxed)
    fs::write("output.txt", "Hello from WASI!").unwrap();
    let content = fs::read_to_string("output.txt").unwrap();
    println!("Read: {}", content);

    // Environment variables
    if let Ok(val) = env::var("MY_VAR") {
        println!("MY_VAR = {}", val);
    }
}
```

```bash
# Build
cargo build --target wasm32-wasip1 --release

# Run with wasmtime (grant file access)
wasmtime --dir=. ./target/wasm32-wasip1/release/my-wasi-app.wasm
```

## Capability-Based Security

WASI uses a **sandbox model** — the Wasm module has NO access by default. You explicitly grant capabilities:

```bash
# No access to anything
wasmtime app.wasm

# Grant access to current directory
wasmtime --dir=. app.wasm

# Grant access to specific path
wasmtime --dir=/data::/app/data app.wasm

# Grant environment variables
wasmtime --env MY_VAR=hello app.wasm
```

This is fundamentally more secure than native executables — a Wasm module can't read your files, network, or environment unless you explicitly allow it.

## Use Cases

| Use Case | Why WASI? |
|----------|----------|
| **Edge computing** (Cloudflare Workers) | Cold start ~1ms vs ~100ms for containers |
| **Plugin systems** | Safely run untrusted code in your app |
| **CLI tools** | Write once, run on any platform with a Wasm runtime |
| **Serverless functions** | Smaller, faster, more secure than containers |
| **Embedded/IoT** | Tiny runtime footprint (~5MB) |

## WASI vs Browser Wasm

| Feature | Browser | WASI |
|---------|---------|------|
| DOM access | ✓ | ✗ |
| File system | ✗ | ✓ (sandboxed) |
| Network sockets | ✗ (Fetch only) | ✓ (sandboxed) |
| Startup time | ~100ms | ~1ms |
| Sandbox | Browser sandbox | Capability-based |

## Cloudflare Workers Example

Cloudflare Workers can run WASI Wasm:

```rust
// A simple HTTP handler compiled to WASI
fn main() {
    let body = "Hello from Rust + WASI on Cloudflare!";
    println!("Content-Type: text/plain");
    println!("Content-Length: {}", body.len());
    println!();
    print!("{}", body);
}
```

## The Future: Component Model

WASI is evolving toward the **Component Model** — composable Wasm modules that can import/export typed interfaces (not just bytes). This will enable:
- Wasm modules calling each other
- Language-agnostic interfaces
- Package managers for Wasm components

## Try It

Click **Run** to see WASI capabilities. The code lists what WASI provides and how to compile for it. Try running it locally with `wasmtime` to see file I/O and env vars in action.
