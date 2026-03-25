---
title: Custom Allocators in Wasm
slug: custom-allocators
difficulty: advanced
tags: [data-structures]
order: 35
description: Understand how Wasm memory allocation works and build your own bump allocator, arena allocator, and GlobalAlloc implementation in Rust.
starter_code: |
  use std::alloc::{GlobalAlloc, Layout};
  use std::cell::UnsafeCell;
  use std::ptr;

  // A simple bump allocator
  // Allocates by moving a pointer forward, never frees individual objects.
  // All memory is reclaimed at once via reset().

  const ARENA_SIZE: usize = 4096;

  struct BumpAllocator {
      arena: UnsafeCell<[u8; ARENA_SIZE]>,
      offset: UnsafeCell<usize>,
  }

  // Safety: our allocator is single-threaded (like Wasm)
  unsafe impl Sync for BumpAllocator {}

  impl BumpAllocator {
      const fn new() -> Self {
          BumpAllocator {
              arena: UnsafeCell::new([0u8; ARENA_SIZE]),
              offset: UnsafeCell::new(0),
          }
      }

      fn reset(&self) {
          unsafe {
              *self.offset.get() = 0;
          }
      }

      fn used(&self) -> usize {
          unsafe { *self.offset.get() }
      }
  }

  unsafe impl GlobalAlloc for BumpAllocator {
      unsafe fn alloc(&self, layout: Layout) -> *mut u8 {
          let offset = &mut *self.offset.get();
          let arena = &mut *self.arena.get();

          // Align the current offset
          let align = layout.align();
          let aligned = (*offset + align - 1) & !(align - 1);
          let new_offset = aligned + layout.size();

          if new_offset > ARENA_SIZE {
              return ptr::null_mut(); // Out of memory
          }

          *offset = new_offset;
          arena.as_mut_ptr().add(aligned)
      }

      unsafe fn dealloc(&self, _ptr: *mut u8, _layout: Layout) {
          // Bump allocator: individual dealloc is a no-op.
          // Memory is reclaimed only via reset().
      }
  }

  // Normally you'd use: #[global_allocator]
  // static ALLOC: BumpAllocator = BumpAllocator::new();
  // But for this demo we drive it manually.

  fn main() {
      let alloc = BumpAllocator::new();

      println!("=== Bump Allocator Demo ===");
      println!("Arena size: {} bytes", ARENA_SIZE);
      println!("Used after init: {} bytes", alloc.used());

      // Allocate some objects
      let layout_u32 = Layout::new::<u32>();
      let layout_u64 = Layout::new::<u64>();
      let layout_arr = Layout::array::<u8>(100).unwrap();

      unsafe {
          let p1 = alloc.alloc(layout_u32);
          println!("\nAllocated u32  (4 bytes,  align 4): used = {}", alloc.used());

          let p2 = alloc.alloc(layout_u64);
          println!("Allocated u64  (8 bytes,  align 8): used = {}", alloc.used());

          let p3 = alloc.alloc(layout_arr);
          println!("Allocated [u8;100]      (align 1): used = {}", alloc.used());

          // Write to and read from the allocated memory
          *(p1 as *mut u32) = 42;
          *(p2 as *mut u64) = 123456789;
          ptr::write_bytes(p3, 0xAB, 100);

          println!("\nValues:");
          println!("  *p1 (u32) = {}", *(p1 as *const u32));
          println!("  *p2 (u64) = {}", *(p2 as *const u64));
          println!("  *p3 [0]   = 0x{:02X}", *p3);
      }

      // Reset reclaims all memory
      alloc.reset();
      println!("\nAfter reset: used = {} bytes", alloc.used());
  }
expected_output: |
  === Bump Allocator Demo ===
  Arena size: 4096 bytes
  Used after init: 0 bytes

  Allocated u32  (4 bytes,  align 4): used = 4
  Allocated u64  (8 bytes,  align 8): used = 16
  Allocated [u8;100]      (align 1): used = 116

  Values:
    *p1 (u32) = 42
    *p2 (u64) = 123456789
    *p3 [0]   = 0xAB

  After reset: used = 0 bytes
---

## How Wasm Memory Allocation Works

WebAssembly has a single linear memory — a contiguous, growable byte array. There is no built-in heap, no garbage collector, and no `malloc`. Your Rust program must bring its own allocator.

```
  Wasm Linear Memory
  ┌──────────────────────────────────────────────────┐
  │ Stack │ Static Data │         Heap →             │
  │ (grows│ (.data,     │   (managed by allocator)   │
  │  down)│  .rodata)   │                            │
  └──────────────────────────────────────────────────┘
  0x0000                                      memory.size
                                              (pages × 64KB)
```

Key facts:
- Memory is measured in **pages** of 64 KB each
- `memory.grow(n)` adds `n` pages; it can fail if the host imposes limits
- There is **no memory protection** — any offset within bounds is accessible
- The allocator is linked into your `.wasm` binary, adding to its size

## Default Allocator: dlmalloc

By default, Rust's `wasm32-unknown-unknown` target uses a port of **dlmalloc** (Doug Lea's malloc). It is a general-purpose allocator optimized for a mix of allocation sizes.

```
  dlmalloc internals (simplified)
  ┌────────────────────────────────────────────┐
  │  Free list bins (by size class)            │
  │  ┌─────┐ ┌─────┐ ┌──────┐ ┌───────────┐  │
  │  │ 8B  │ │ 16B │ │ 64B  │ │ 256B+     │  │
  │  │ free│ │ free│ │ free │ │ free list │  │
  │  │ list│ │ list│ │ list │ │ (tree)    │  │
  │  └─────┘ └─────┘ └──────┘ └───────────┘  │
  │                                            │
  │  On alloc: find best-fit bin, split chunk  │
  │  On free:  coalesce neighbors, return bin  │
  └────────────────────────────────────────────┘
```

| Property         | dlmalloc                |
|------------------|-------------------------|
| Binary overhead  | ~10 KB                  |
| Alloc complexity | O(1) for small, O(log n) large |
| Fragmentation    | Low (coalescing)        |
| Thread-safe      | Not needed (Wasm is single-threaded) |

## wee_alloc: The Tiny Alternative

`wee_alloc` was designed specifically for Wasm — it trades allocation speed and fragmentation resistance for a ~1 KB binary footprint.

```rust
#[global_allocator]
static ALLOC: wee_alloc::WeeAlloc = wee_alloc::WeeAlloc::INIT;
```

It works by maintaining a linked free list with minimal bookkeeping:

```
  wee_alloc free list
  ┌──────┐    ┌──────┐    ┌──────┐
  │ cell │───>│ cell │───>│ cell │───> null
  │ 32B  │    │ 16B  │    │ 64B  │
  └──────┘    └──────┘    └──────┘

  Alloc: walk list, find first fit
  Free:  prepend to list (no coalescing!)
```

> **Caveat**: `wee_alloc` does not coalesce adjacent free blocks. Over time, memory can fragment badly. It is no longer maintained — consider `lol_alloc` or `talc` for new projects.

## Bump Allocators

A bump allocator is the simplest and fastest possible allocator. It maintains a single pointer that moves forward on each allocation. Individual frees are impossible — all memory is reclaimed at once.

```
  Bump Allocator State

  After 3 allocations:
  ┌───────┬────────┬──────────────┬─────────────────────┐
  │ obj A │ obj B  │   obj C      │     free space       │
  └───────┴────────┴──────────────┴─────────────────────┘
  ^                                ^                     ^
  base                           offset                 end

  Alloc:  offset += size (after alignment)
  Free:   no-op
  Reset:  offset = base (reclaims everything)
```

### When to Use Bump Allocation

| Use case                           | Good fit? |
|------------------------------------|-----------|
| Per-frame game allocations         | Yes       |
| Request-scoped web handler data    | Yes       |
| Short-lived computation scratch    | Yes       |
| Long-running with varied lifetimes | No        |
| Need to free individual objects    | No        |

### Alignment

The allocator must respect each type's alignment requirement. On Wasm, `u32` needs 4-byte alignment, `u64` needs 8-byte alignment:

```
  Before alignment:  offset = 5
  Requested align:   8

  aligned = (5 + 8 - 1) & !(8 - 1)
          = 12 & !7
          = 12 & 0xFFFF_FFF8
          = 8

  Memory:
  ┌─┬─┬─┬─┬─┬─┬─┬─┬─────────────────┐
  │0│1│2│3│4│ │ │ │8  9 10 11 12 ... │
  └─┴─┴─┴─┴─┴─┴─┴─┴─────────────────┘
                ^   ^
            wasted  aligned offset
           (padding)
```

## Arena Allocation

An arena is a bump allocator that grows by allocating new chunks when the current one fills up:

```rust
struct Arena {
    chunks: Vec<Vec<u8>>,
    current: *mut u8,
    remaining: usize,
}

impl Arena {
    fn new() -> Self {
        let chunk = vec![0u8; 4096];
        let ptr = chunk.as_ptr() as *mut u8;
        Arena {
            chunks: vec![chunk],
            current: ptr,
            remaining: 4096,
        }
    }

    fn alloc(&mut self, layout: Layout) -> *mut u8 {
        // Align
        let align = layout.align();
        let padding = self.current.align_offset(align);
        let needed = padding + layout.size();

        if needed > self.remaining {
            // Allocate a new chunk
            let size = needed.max(4096);
            let chunk = vec![0u8; size];
            let ptr = chunk.as_ptr() as *mut u8;
            self.chunks.push(chunk);
            self.current = ptr;
            self.remaining = size;
            return self.alloc(layout); // retry
        }

        unsafe {
            let result = self.current.add(padding);
            self.current = self.current.add(needed);
            self.remaining -= needed;
            result
        }
    }

    fn reset(&mut self) {
        // Keep only the first chunk
        self.chunks.truncate(1);
        let chunk = &self.chunks[0];
        self.current = chunk.as_ptr() as *mut u8;
        self.remaining = chunk.len();
    }
}
```

```
  Arena with multiple chunks:

  Chunk 0 (full)        Chunk 1 (full)        Chunk 2 (active)
  ┌──────────────┐     ┌──────────────┐     ┌───────┬────────┐
  │██████████████│     │██████████████│     │███████│  free  │
  └──────────────┘     └──────────────┘     └───────┴────────┘
                                              ^current
```

## The GlobalAlloc Trait

Rust's `GlobalAlloc` trait is the interface every allocator must implement:

```rust
use std::alloc::{GlobalAlloc, Layout};

unsafe trait GlobalAlloc {
    unsafe fn alloc(&self, layout: Layout) -> *mut u8;
    unsafe fn dealloc(&self, ptr: *mut u8, layout: Layout);

    // Optional — default implementations call alloc + copy
    unsafe fn alloc_zeroed(&self, layout: Layout) -> *mut u8 { ... }
    unsafe fn realloc(&self, ptr: *mut u8, layout: Layout, new_size: usize) -> *mut u8 { ... }
}
```

### Layout

`Layout` describes the size and alignment of a memory request:

```rust
let layout = Layout::new::<u32>();
assert_eq!(layout.size(), 4);
assert_eq!(layout.align(), 4);

let layout = Layout::array::<f64>(10).unwrap();
assert_eq!(layout.size(), 80);    // 10 * 8
assert_eq!(layout.align(), 8);

let layout = Layout::from_size_align(256, 16).unwrap();
```

### Registering Your Allocator

```rust
#[global_allocator]
static ALLOC: MyAllocator = MyAllocator::new();
```

This replaces the default allocator for **all** heap allocations in your binary — `Box`, `Vec`, `String`, `HashMap`, everything.

## Allocator Comparison

| Allocator    | Binary Size | Alloc Speed | Free Speed | Fragmentation | Best For              |
|-------------|-------------|-------------|------------|---------------|-----------------------|
| dlmalloc    | ~10 KB      | Fast        | Fast       | Low           | General purpose       |
| wee_alloc   | ~1 KB       | Medium      | Fast       | High          | Tiny Wasm binaries    |
| Bump        | ~0.1 KB     | Fastest     | N/A        | None*         | Per-frame / scoped    |
| Arena       | ~0.5 KB     | Very fast   | N/A        | None*         | Batch processing      |
| lol_alloc   | ~0.2 KB     | Medium      | Slow       | Medium        | Minimal Wasm          |
| talc        | ~5 KB       | Fast        | Fast       | Low           | General (modern)      |

*No fragmentation because individual frees don't happen.

## Pool Allocator Pattern

For types that are allocated and freed frequently (e.g., game entities, network packets), a pool allocator pre-allocates a fixed number of slots:

```rust
struct Pool<T> {
    storage: Vec<Option<T>>,
    free_indices: Vec<usize>,
}

impl<T> Pool<T> {
    fn new(capacity: usize) -> Self {
        Pool {
            storage: (0..capacity).map(|_| None).collect(),
            free_indices: (0..capacity).rev().collect(),
        }
    }

    fn alloc(&mut self, value: T) -> Option<usize> {
        let idx = self.free_indices.pop()?;
        self.storage[idx] = Some(value);
        Some(idx)
    }

    fn free(&mut self, idx: usize) {
        self.storage[idx] = None;
        self.free_indices.push(idx);
    }

    fn get(&self, idx: usize) -> Option<&T> {
        self.storage.get(idx)?.as_ref()
    }
}
```

```
  Pool<Entity> with capacity 8:

  ┌─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┐
  │  E  │     │  E  │  E  │     │  E  │     │  E  │
  │ [0] │ [1] │ [2] │ [3] │ [4] │ [5] │ [6] │ [7] │
  └─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┘
    used  free  used  used  free  used  free  used

  free_indices = [6, 4, 1]  (stack order)
```

## Summary

Wasm's linear memory model means you control allocation entirely. The default dlmalloc works well for most applications. For size-critical builds, swap in a tiny allocator. For performance-critical workloads with predictable lifetimes, bump and arena allocators give unbeatable speed by deferring all frees to a single reset.
