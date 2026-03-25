---
title: "Wasm Streams & Async Iterators"
slug: wasm-streams
difficulty: advanced
tags: [api]
order: 30
description: Work with ReadableStream, WritableStream, and TransformStream from Rust — streaming large files, handling backpressure, and building async iterator pipelines with the wasm-streams crate.
starter_code: |
  use std::collections::VecDeque;

  fn main() {
      println!("=== Wasm Streams & Async Iterators ===\n");

      // --- Simulating a ReadableStream ---
      println!("--- ReadableStream Simulation ---");

      struct ReadableStream {
          chunks: VecDeque<Vec<u8>>,
          position: usize,
      }

      impl ReadableStream {
          fn new(data: &[u8], chunk_size: usize) -> Self {
              let mut chunks = VecDeque::new();
              for chunk in data.chunks(chunk_size) {
                  chunks.push_back(chunk.to_vec());
              }
              println!("  Created stream: {} bytes in {} chunks of {}",
                  data.len(), chunks.len(), chunk_size);
              ReadableStream { chunks, position: 0 }
          }

          fn read(&mut self) -> Option<Vec<u8>> {
              self.chunks.pop_front().map(|chunk| {
                  self.position += chunk.len();
                  chunk
              })
          }

          fn is_done(&self) -> bool {
              self.chunks.is_empty()
          }
      }

      let data = b"Hello, this is streaming data flowing through Wasm!";
      let mut stream = ReadableStream::new(data, 10);

      let mut total_read = 0;
      while let Some(chunk) = stream.read() {
          total_read += chunk.len();
          let text = String::from_utf8_lossy(&chunk);
          println!("  Read chunk ({:2} bytes): {:?}", chunk.len(), text);
      }
      println!("  Stream done. Total: {} bytes\n", total_read);

      // --- Simulating a WritableStream with backpressure ---
      println!("--- WritableStream with Backpressure ---");

      struct WritableStream {
          buffer: Vec<u8>,
          capacity: usize,
          high_water_mark: usize,
          total_written: usize,
      }

      impl WritableStream {
          fn new(capacity: usize, high_water_mark: usize) -> Self {
              println!("  Created writable: capacity={}, high_water_mark={}",
                  capacity, high_water_mark);
              WritableStream {
                  buffer: Vec::with_capacity(capacity),
                  capacity,
                  high_water_mark,
                  total_written: 0,
              }
          }

          fn write(&mut self, data: &[u8]) -> Result<bool, String> {
              if self.buffer.len() + data.len() > self.capacity {
                  return Err(format!("Buffer full ({}/{})", self.buffer.len(), self.capacity));
              }
              self.buffer.extend_from_slice(data);
              self.total_written += data.len();
              let backpressure = self.buffer.len() >= self.high_water_mark;
              Ok(backpressure)
          }

          fn flush(&mut self) -> Vec<u8> {
              let flushed = self.buffer.clone();
              self.buffer.clear();
              flushed
          }
      }

      let mut writable = WritableStream::new(30, 20);
      let chunks_to_write = vec![b"chunk1_" as &[u8], b"chunk2_", b"chunk3_", b"chunk4_"];

      for chunk in &chunks_to_write {
          match writable.write(chunk) {
              Ok(backpressure) => {
                  println!("  Wrote {} bytes, backpressure: {}", chunk.len(), backpressure);
                  if backpressure {
                      let flushed = writable.flush();
                      println!("  Flushed {} bytes (backpressure relief)", flushed.len());
                  }
              }
              Err(e) => println!("  Write error: {}", e),
          }
      }
      println!("  Total written: {} bytes\n", writable.total_written);

      // --- Simulating a TransformStream ---
      println!("--- TransformStream Simulation ---");

      struct TransformStream {
          transform_fn: Box<dyn Fn(&[u8]) -> Vec<u8>>,
          name: String,
      }

      impl TransformStream {
          fn new(name: &str, f: Box<dyn Fn(&[u8]) -> Vec<u8>>) -> Self {
              println!("  Created transform: '{}'", name);
              TransformStream { transform_fn: f, name: name.to_string() }
          }

          fn transform(&self, chunk: &[u8]) -> Vec<u8> {
              (self.transform_fn)(chunk)
          }
      }

      let uppercase = TransformStream::new("uppercase", Box::new(|data| {
          data.iter().map(|b| b.to_ascii_uppercase()).collect()
      }));

      let prefix = TransformStream::new("prefix", Box::new(|data| {
          let mut result = b"[OUT] ".to_vec();
          result.extend_from_slice(data);
          result
      }));

      let input_chunks = vec![b"hello " as &[u8], b"world ", b"from wasm"];
      println!("\n  Pipeline: input -> uppercase -> prefix -> output");
      for chunk in &input_chunks {
          let step1 = uppercase.transform(chunk);
          let step2 = prefix.transform(&step1);
          println!("  {:?} -> {:?}",
              String::from_utf8_lossy(chunk),
              String::from_utf8_lossy(&step2));
      }

      // --- Async iterator pattern ---
      println!("\n--- Async Iterator Pattern ---");

      struct AsyncIter {
          items: Vec<String>,
          index: usize,
      }

      impl AsyncIter {
          fn new(items: Vec<String>) -> Self {
              AsyncIter { items, index: 0 }
          }
      }

      impl Iterator for AsyncIter {
          type Item = String;
          fn next(&mut self) -> Option<String> {
              if self.index < self.items.len() {
                  let item = self.items[self.index].clone();
                  self.index += 1;
                  Some(item)
              } else {
                  None
              }
          }
      }

      let iter = AsyncIter::new(vec![
          "chunk_a".into(), "chunk_b".into(), "chunk_c".into(),
      ]);

      for (i, item) in iter.enumerate() {
          println!("  yield[{}]: {}", i, item);
      }

      println!("\n--- Stream Size Estimation ---");
      let file_size: u64 = 10 * 1024 * 1024; // 10 MB
      let chunk_size: u64 = 64 * 1024;        // 64 KB
      let num_chunks = (file_size + chunk_size - 1) / chunk_size;
      println!("  File: {} MB", file_size / (1024 * 1024));
      println!("  Chunk size: {} KB", chunk_size / 1024);
      println!("  Total chunks: {}", num_chunks);
      println!("  Memory per chunk: {} KB", chunk_size / 1024);
      println!("  Peak memory (3 buffered): {} KB", 3 * chunk_size / 1024);
  }
expected_output: |
  === Wasm Streams & Async Iterators ===

  --- ReadableStream Simulation ---
    Created stream: 51 bytes in 6 chunks of 10
    Read chunk (10 bytes): "Hello, thi"
    Read chunk (10 bytes): "s is strea"
    Read chunk (10 bytes): "ming data "
    Read chunk (10 bytes): "flowing th"
    Read chunk (10 bytes): "rough Wasm"
    Read chunk ( 1 bytes): "!"
    Stream done. Total: 51 bytes

  --- WritableStream with Backpressure ---
    Created writable: capacity=30, high_water_mark=20
    Wrote 7 bytes, backpressure: false
    Wrote 7 bytes, backpressure: false
    Wrote 7 bytes, backpressure: true
    Flushed 21 bytes (backpressure relief)
    Wrote 7 bytes, backpressure: false
    Total written: 28 bytes

  --- TransformStream Simulation ---
    Created transform: 'uppercase'
    Created transform: 'prefix'

    Pipeline: input -> uppercase -> prefix -> output
    "hello " -> "[OUT] HELLO "
    "world " -> "[OUT] WORLD "
    "from wasm" -> "[OUT] FROM WASM"

  --- Async Iterator Pattern ---
    yield[0]: chunk_a
    yield[1]: chunk_b
    yield[2]: chunk_c

  --- Stream Size Estimation ---
    File: 10 MB
    Chunk size: 64 KB
    Total chunks: 160
    Memory per chunk: 64 KB
    Peak memory (3 buffered): 192 KB
---

## What Are Web Streams?

The [Streams API](https://developer.mozilla.org/en-US/docs/Web/API/Streams_API) lets you process data incrementally instead of loading entire files into memory. This is essential for Wasm when handling large files — you don't want to copy a 500 MB video into linear memory all at once.

```
┌──────────────────────────────────────────────────────────┐
│                  Streams Architecture                     │
│                                                          │
│  Source ──→ ReadableStream ──→ TransformStream ──→ Sink  │
│             (produces)         (modifies)     WritableStream │
│                                               (consumes)     │
│                                                          │
│  ◄────────── Backpressure signal flows upstream ────────►│
└──────────────────────────────────────────────────────────┘
```

There are three types of streams:

| Stream Type | Purpose | Direction |
|------------|---------|-----------|
| `ReadableStream` | Produces data chunks | Source → Consumer |
| `WritableStream` | Consumes data chunks | Producer → Sink |
| `TransformStream` | Modifies data in-flight | Input → Output |

## The wasm-streams Crate

The [wasm-streams](https://crates.io/crates/wasm-streams) crate provides Rust bindings for the Web Streams API, converting JS streams into Rust `Stream`/`Sink` types from the `futures` crate.

```toml
[dependencies]
wasm-streams = "0.4"
futures = "0.3"
wasm-bindgen = "0.2"
wasm-bindgen-futures = "0.4"
web-sys = { version = "0.3", features = ["ReadableStream", "WritableStream"] }
```

## Reading a Fetch Response as a Stream

The most common use case is streaming a `fetch()` response:

```rust
use wasm_bindgen::prelude::*;
use wasm_bindgen_futures::JsFuture;
use wasm_streams::ReadableStream;
use futures::StreamExt;

#[wasm_bindgen]
pub async fn stream_fetch(url: &str) -> Result<(), JsValue> {
    let window = web_sys::window().unwrap();
    let resp: web_sys::Response = JsFuture::from(window.fetch_with_str(url))
        .await?
        .dyn_into()?;

    // Get the ReadableStream from the response body
    let raw_body = resp.body().unwrap();
    let body = ReadableStream::from_raw(raw_body);

    // Convert to a Rust Stream
    let mut reader = body.into_stream();

    let mut total_bytes = 0u64;
    while let Some(chunk) = reader.next().await {
        let chunk = chunk?;
        let array: js_sys::Uint8Array = chunk.dyn_into()?;
        total_bytes += array.length() as u64;
        // Process chunk here...
    }

    web_sys::console::log_1(
        &format!("Streamed {} bytes", total_bytes).into()
    );
    Ok(())
}
```

## Backpressure

Backpressure is the mechanism that prevents a fast producer from overwhelming a slow consumer. Without it, you'd buffer unbounded data in memory.

```
Fast Producer          Slow Consumer
     │                      │
     │──── chunk 1 ────────►│ (processing...)
     │──── chunk 2 ────────►│ (still processing chunk 1)
     │──── chunk 3 ──► WAIT │ ← backpressure signal
     │     (paused)         │
     │◄── ready ────────────│ (done with chunk 1)
     │──── chunk 3 ────────►│ (resume)
```

The Streams API handles backpressure automatically via the **high-water mark** and **queue size**:

```rust
// In wasm-streams, backpressure is handled transparently:
let mut writer = writable_stream.into_sink();

// This will automatically wait if the internal queue is full
writer.send(chunk).await?;  // may suspend if backpressured
```

### High-Water Mark Configuration

```js
// JavaScript side — configuring the queuing strategy
const stream = new ReadableStream({
    start(controller) { /* ... */ },
    pull(controller) { /* ... */ },
}, {
    highWaterMark: 3,  // Buffer up to 3 chunks before backpressure
});
```

## TransformStream: Processing Data In-Flight

A `TransformStream` sits between a readable and writable stream, modifying data as it passes through:

```rust
use wasm_streams::TransformStream;

#[wasm_bindgen]
pub fn create_uppercase_transform() -> web_sys::TransformStream {
    let transform = TransformStream::new(
        // transform function
        |chunk: JsValue, controller: &TransformStreamDefaultController| {
            let input: js_sys::Uint8Array = chunk.dyn_into().unwrap();
            let mut data = vec![0u8; input.length() as usize];
            input.copy_to(&mut data);

            // Transform: uppercase ASCII
            for byte in &mut data {
                if *byte >= b'a' && *byte <= b'z' {
                    *byte -= 32;
                }
            }

            let output = js_sys::Uint8Array::from(&data[..]);
            controller.enqueue(&output).unwrap();
            Ok(())
        },
    );

    transform.into_raw()
}
```

## Piping Streams Together

You can chain streams into a pipeline:

```rust
#[wasm_bindgen]
pub async fn process_file(input: web_sys::ReadableStream) -> Result<(), JsValue> {
    let readable = ReadableStream::from_raw(input);
    let mut stream = readable.into_stream();

    let mut processed_bytes = 0u64;
    let mut chunk_count = 0u32;

    while let Some(chunk) = stream.next().await {
        let chunk = chunk?;
        let array: js_sys::Uint8Array = chunk.dyn_into()?;
        let len = array.length() as usize;

        // Process in Wasm linear memory
        let mut data = vec![0u8; len];
        array.copy_to(&mut data);

        // Example: compute checksum
        let checksum: u32 = data.iter().map(|&b| b as u32).sum();

        processed_bytes += len as u64;
        chunk_count += 1;
    }

    Ok(())
}
```

## Streaming Large Files: A Complete Example

```
┌─────────┐    ┌───────────┐    ┌──────────────┐    ┌────────┐
│  File    │───►│ Readable  │───►│  Transform   │───►│ Write  │
│  Input   │    │  Stream   │    │  (in Wasm)   │    │ to DB  │
│  (disk)  │    │ 64KB      │    │  compress/   │    │        │
│          │    │ chunks    │    │  encrypt     │    │        │
└─────────┘    └───────────┘    └──────────────┘    └────────┘
                                       │
                              Only 1-3 chunks in
                              memory at a time!
```

```js
// JavaScript side: wire up the pipeline
const fileStream = file.stream();  // ReadableStream from File API
const transform = wasm.create_compression_transform();

const compressed = fileStream.pipeThrough(transform);
const writer = getStorageWriter();
await compressed.pipeTo(writer);
```

## Async Iterators in Rust for Wasm

Rust's `Stream` trait (from `futures`) is the async equivalent of `Iterator`:

```rust
use futures::stream::{self, StreamExt};

#[wasm_bindgen]
pub async fn process_items() {
    let items = stream::iter(vec![1, 2, 3, 4, 5]);

    items
        .map(|x| x * 2)
        .filter(|x| futures::future::ready(*x > 4))
        .for_each(|x| async move {
            web_sys::console::log_1(&format!("Item: {}", x).into());
        })
        .await;
}
```

## Memory Considerations

| Approach | Memory Usage | Latency |
|----------|-------------|---------|
| Load entire file | O(file_size) | High (wait for full download) |
| Stream with 64KB chunks | O(64KB * buffer_count) | Low (process as data arrives) |
| Stream with backpressure | O(high_water_mark) | Adaptive |

For a 100 MB file with 64 KB chunks:

- **Without streaming:** 100 MB in Wasm linear memory
- **With streaming:** ~192 KB (3 buffered chunks) in Wasm linear memory

## Key Takeaways

1. **Web Streams** let you process data incrementally, avoiding large memory allocations
2. **wasm-streams** bridges the JS Streams API with Rust's `futures::Stream` and `futures::Sink`
3. **Backpressure** prevents fast producers from overwhelming slow consumers — the Streams API handles it automatically
4. **TransformStream** lets Wasm process data in-flight without buffering the entire input
5. **64 KB chunks** are a good default chunk size — they match Wasm page size and balance throughput vs memory
6. Always use streaming for files larger than a few MB in Wasm applications
