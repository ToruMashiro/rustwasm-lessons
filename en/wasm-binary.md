---
title: Wasm Binary Format
slug: wasm-binary
difficulty: advanced
tags: [getting-started]
order: 27
description: Understand the .wasm binary format — modules, sections, WAT text format, and how the browser loads and instantiates Wasm.
starter_code: |
  fn main() {
      println!("=== Wasm Binary Format ===\n");

      // The magic bytes that start every .wasm file
      let magic: [u8; 4] = [0x00, 0x61, 0x73, 0x6D]; // \0asm
      let version: [u8; 4] = [0x01, 0x00, 0x00, 0x00]; // version 1

      print!("Magic number: ");
      for b in &magic {
          print!("0x{:02X} ", b);
      }
      println!("(\\0asm)");

      print!("Version:      ");
      for b in &version {
          print!("0x{:02X} ", b);
      }
      println!("(1)");

      println!("\nSections in a .wasm file:");
      let sections = [
          (1, "Type", "Function signatures"),
          (2, "Import", "Imported functions/memory"),
          (3, "Function", "Function declarations"),
          (5, "Memory", "Linear memory definitions"),
          (6, "Global", "Global variables"),
          (7, "Export", "Exported functions/memory"),
          (10, "Code", "Function bodies (the actual code)"),
          (11, "Data", "Static data (strings, constants)"),
      ];

      for (id, name, desc) in &sections {
          println!("  Section {:2}: {:10} — {}", id, name, desc);
      }
  }
expected_output: |
  === Wasm Binary Format ===

  Magic number: 0x00 0x61 0x73 0x6D (\0asm)
  Version:      0x01 0x00 0x00 0x00 (1)

  Sections in a .wasm file:
    Section  1: Type       — Function signatures
    Section  2: Import     — Imported functions/memory
    Section  3: Function   — Function declarations
    Section  5: Memory     — Linear memory definitions
    Section  6: Global     — Global variables
    Section  7: Export     — Exported functions/memory
    Section 10: Code       — Function bodies (the actual code)
    Section 11: Data       — Static data (strings, constants)
---

## The .wasm Binary

Every `.wasm` file starts with 8 bytes:

```
00 61 73 6D    ← Magic number: "\0asm"
01 00 00 00    ← Version: 1
```

After the header, the file contains **sections** — each section has a type ID, size, and content.

## Sections

| ID | Name | Contains |
|----|------|----------|
| 0 | Custom | Debug info, names, metadata |
| 1 | Type | Function type signatures `(i32, i32) -> i32` |
| 2 | Import | Functions/memory/globals imported from JS |
| 3 | Function | Maps function index → type signature |
| 4 | Table | Indirect function call tables |
| 5 | Memory | Linear memory declarations (initial + max pages) |
| 6 | Global | Global variable declarations |
| 7 | Export | Functions/memory exposed to JavaScript |
| 8 | Start | Function to call on instantiation |
| 9 | Element | Table initialization data |
| 10 | Code | The actual function bytecode |
| 11 | Data | Static data (strings, constants) |

## WAT: The Text Format

WAT (WebAssembly Text) is the human-readable form of Wasm. You can convert between them:

```bash
# Binary → text
wasm2wat my_module.wasm -o my_module.wat

# Text → binary
wat2wasm my_module.wat -o my_module.wasm
```

### A simple function in WAT

```wat
(module
  ;; Type section: function signature
  (type $add_type (func (param i32 i32) (result i32)))

  ;; Function section + Code section
  (func $add (type $add_type) (param $a i32) (param $b i32) (result i32)
    local.get $a
    local.get $b
    i32.add
  )

  ;; Export section
  (export "add" (func $add))
)
```

This compiles to the same thing as:

```rust
#[wasm_bindgen]
pub fn add(a: i32, b: i32) -> i32 {
    a + b
}
```

## Wasm Instruction Set

Wasm uses a **stack machine** — instructions push/pop values on a stack:

```wat
;; Calculate (a + b) * 2
local.get $a     ;; stack: [a]
local.get $b     ;; stack: [a, b]
i32.add          ;; stack: [a+b]
i32.const 2      ;; stack: [a+b, 2]
i32.mul          ;; stack: [(a+b)*2]
```

### Common instructions

| Instruction | What it does |
|-------------|-------------|
| `local.get $x` | Push local variable |
| `local.set $x` | Pop and store to local |
| `i32.const N` | Push constant |
| `i32.add` | Pop 2, push sum |
| `i32.mul` | Pop 2, push product |
| `i32.lt_s` | Pop 2, push 1 if less |
| `if...else...end` | Conditional |
| `loop...br_if...end` | Loop with conditional break |
| `call $fn` | Call function |
| `return` | Return from function |

## How the Browser Loads Wasm

```js
// Step 1: Fetch the binary
const response = await fetch('module.wasm');

// Step 2: Compile (streaming = compiles while downloading)
const module = await WebAssembly.compileStreaming(response);

// Step 3: Instantiate (link imports, allocate memory)
const instance = await WebAssembly.instantiate(module, {
    env: {
        log: (ptr, len) => { /* JS function imported by Wasm */ }
    }
});

// Step 4: Call exported functions
const result = instance.exports.add(2, 3);
```

### Streaming compilation

```
Download:  ████████████████████░░░░░
Compile:   ░░░████████████████████░░
Execute:   ░░░░░░░░░░░░░░░░░░░░░░██
```

The browser compiles Wasm **while downloading** — so a 100KB Wasm file might be ready to execute immediately after the download finishes.

## Inspecting .wasm Files

```bash
# See what's exported
wasm-objdump -x module.wasm

# See the disassembly
wasm-objdump -d module.wasm

# See section sizes
wasm-objdump -h module.wasm

# Convert to readable text
wasm2wat module.wasm
```

## Try It

Click **Run** to see the binary format structure — magic bytes, version, and all section types that make up a .wasm file.
