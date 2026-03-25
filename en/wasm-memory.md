---
title: Wasm Memory Model
slug: wasm-memory
difficulty: intermediate
tags: [data-structures]
order: 21
description: Understand how WebAssembly linear memory works — allocation, growth, views, and sharing data between Rust and JavaScript efficiently.
starter_code: |
  fn main() {
      // Simulating Wasm linear memory concepts

      // 1. Linear memory is a contiguous byte array
      let mut memory: Vec<u8> = vec![0; 65536]; // 1 page = 64KB
      println!("Memory size: {} bytes ({} pages)", memory.len(), memory.len() / 65536);

      // 2. Writing data at specific offsets
      let message = "Hello, Wasm!";
      let offset = 1024;
      for (i, byte) in message.bytes().enumerate() {
          memory[offset + i] = byte;
      }

      // 3. Reading data back
      let read: String = memory[offset..offset + message.len()]
          .iter()
          .map(|&b| b as char)
          .collect();
      println!("Read from offset {}: \"{}\"", offset, read);

      // 4. Growing memory
      let old_size = memory.len();
      memory.resize(old_size + 65536, 0); // Grow by 1 page
      println!("Grew to: {} bytes ({} pages)", memory.len(), memory.len() / 65536);

      // 5. Storing numbers (little-endian, like Wasm)
      let value: u32 = 42;
      let num_offset = 2048;
      memory[num_offset..num_offset + 4].copy_from_slice(&value.to_le_bytes());
      let read_value = u32::from_le_bytes(memory[num_offset..num_offset + 4].try_into().unwrap());
      println!("Stored u32 at offset {}: {}", num_offset, read_value);
  }
expected_output: |
  Memory size: 65536 bytes (1 pages)
  Read from offset 1024: "Hello, Wasm!"
  Grew to: 131072 bytes (2 pages)
  Stored u32 at offset 2048: 42
---

## What is Linear Memory?

WebAssembly has a single, contiguous block of memory called **linear memory**. Think of it as a giant `Vec<u8>` that both Rust and JavaScript can read and write to.

```
Address:  0x0000    0x0100    0x0200    ...    0xFFFF
         ┌─────────┬─────────┬─────────┬──────┐
Memory:  │ stack   │ heap    │ data    │ ...  │
         │ data    │ allocs  │ section │      │
         └─────────┴─────────┴─────────┴──────┘
         └────────── 1 page = 64KB ───────────┘
```

Key properties:
- **Byte-addressable** — every byte has an index (0, 1, 2, ...)
- **Pages** — memory grows in 64KB pages (1 page = 65,536 bytes)
- **Shared** — both Rust and JS can access the same bytes
- **Bounds-checked** — accessing out-of-bounds memory traps (doesn't crash the browser)

## How Rust Uses Linear Memory

When you compile Rust to Wasm, the compiler lays out memory like this:

```
┌──────────────┬──────────────┬──────────────┬──────────────┐
│ Static data  │ Stack        │ Heap         │ Free space   │
│ (strings,    │ (local vars, │ (Vec, String,│ (grows →)    │
│  constants)  │  call frames)│  Box allocs) │              │
└──────────────┴──────────────┴──────────────┴──────────────┘
0x0000                                                  0xFFFF...
```

## Accessing Memory from JavaScript

JavaScript sees Wasm memory as an `ArrayBuffer`:

```js
// After init(), wasm.memory is available
const memory = wasm.memory;

// Create typed views into the buffer
const bytes = new Uint8Array(memory.buffer);
const u32s = new Uint32Array(memory.buffer);
const f64s = new Float64Array(memory.buffer);

// Read a byte at address 1024
const value = bytes[1024];

// Write a float at address 2048 (byte offset)
f64s[2048 / 8] = 3.14;  // Float64Array index = byte offset / 8
```

## Passing Data Between Rust and JS

### Approach 1: Copy via wasm-bindgen (simple, safe)

```rust
#[wasm_bindgen]
pub fn process(data: &[u8]) -> Vec<u8> {
    // data is copied from JS to Wasm memory
    let mut result = data.to_vec();
    // ... modify result ...
    result  // copied from Wasm back to JS
}
```

### Approach 2: Pointer + length (zero-copy, advanced)

```rust
#[wasm_bindgen]
pub struct Buffer {
    data: Vec<u8>,
}

#[wasm_bindgen]
impl Buffer {
    #[wasm_bindgen(constructor)]
    pub fn new(size: usize) -> Self {
        Self { data: vec![0; size] }
    }

    pub fn ptr(&self) -> *const u8 {
        self.data.as_ptr()
    }

    pub fn len(&self) -> usize {
        self.data.len()
    }
}
```

```js
const buf = new Buffer(1024);
const ptr = buf.ptr();
const len = buf.len();

// Create a view directly into Wasm memory — zero copy!
const view = new Uint8Array(wasm.memory.buffer, ptr, len);

// Modify in place — Rust sees the changes
view[0] = 42;
```

## Memory Growth

Wasm memory starts small and grows on demand:

```rust
// Rust triggers growth automatically when Vec/String needs more space
let mut big_vec = Vec::with_capacity(1_000_000); // may grow memory
```

```js
// JS can check and grow manually
console.log(wasm.memory.buffer.byteLength); // current size
wasm.memory.grow(10); // grow by 10 pages (640KB)

// IMPORTANT: After grow(), all ArrayBuffer views are invalidated!
// Re-create views after growing:
const freshView = new Uint8Array(wasm.memory.buffer);
```

## Common Pitfalls

| Pitfall | Why | Fix |
|---------|-----|-----|
| Stale ArrayBuffer views after memory growth | `memory.grow()` detaches old buffer | Re-create views after any operation that might grow |
| Alignment errors | Float64Array needs 8-byte aligned offset | Use `ptr as usize % 8 == 0` check |
| Memory leaks | Rust allocations not freed | Drop structs properly, avoid `forget()` overuse |
| Reading freed memory | Use-after-free via stale JS reference | Copy data out before freeing Rust structs |

## Memory Size Budget

| Content | Typical size |
|---------|-------------|
| Small Wasm module | 50-200KB |
| Wasm + web-sys bindings | 200KB-1MB |
| Default linear memory | 1 page (64KB) |
| Large application | 10-50MB |
| Browser limit | ~2-4GB |

## Try It

Click **Run** to see linear memory concepts in action — creating memory, writing/reading at offsets, growing pages, and storing typed values.
