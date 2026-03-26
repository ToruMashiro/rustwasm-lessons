---
title: "Wasm + Compression"
slug: wasm-compression
difficulty: intermediate
tags: [api]
order: 45
description: Implement client-side data compression algorithms in Rust/Wasm — Run-Length Encoding, LZ77, and Huffman coding fundamentals.
starter_code: |
  /// Run-Length Encoding: compress repeated bytes
  /// e.g., [A, A, A, B, B, C] -> [(A, 3), (B, 2), (C, 1)]
  fn rle_encode(data: &[u8]) -> Vec<(u8, u8)> {
      if data.is_empty() {
          return Vec::new();
      }

      let mut result = Vec::new();
      let mut current = data[0];
      let mut count: u8 = 1;

      for &byte in &data[1..] {
          if byte == current && count < 255 {
              count += 1;
          } else {
              result.push((current, count));
              current = byte;
              count = 1;
          }
      }
      result.push((current, count));
      result
  }

  /// RLE decode: expand pairs back to original data
  fn rle_decode(encoded: &[(u8, u8)]) -> Vec<u8> {
      let mut result = Vec::new();
      for &(byte, count) in encoded {
          for _ in 0..count {
              result.push(byte);
          }
      }
      result
  }

  /// Simple LZ77 compression
  /// Finds repeated sequences and replaces them with (offset, length) pairs
  /// Output: sequence of Literal(byte) or Match(offset, length)
  #[derive(Debug, Clone)]
  enum LZ77Token {
      Literal(u8),
      Match { offset: usize, length: usize },
  }

  fn lz77_compress(data: &[u8], window_size: usize) -> Vec<LZ77Token> {
      let mut tokens = Vec::new();
      let mut pos = 0;

      while pos < data.len() {
          let mut best_offset = 0;
          let mut best_length = 0;

          // Search window: look back up to window_size bytes
          let search_start = if pos > window_size { pos - window_size } else { 0 };

          for offset_start in search_start..pos {
              let mut length = 0;
              while pos + length < data.len()
                  && data[offset_start + length] == data[pos + length]
                  && length < 255
              {
                  length += 1;
                  // Prevent reading past the search position
                  if offset_start + length >= pos {
                      break;
                  }
              }

              if length > best_length {
                  best_length = length;
                  best_offset = pos - offset_start;
              }
          }

          if best_length >= 3 {
              // Only encode as match if length >= 3 (saves space)
              tokens.push(LZ77Token::Match {
                  offset: best_offset,
                  length: best_length,
              });
              pos += best_length;
          } else {
              tokens.push(LZ77Token::Literal(data[pos]));
              pos += 1;
          }
      }

      tokens
  }

  /// Decompress LZ77 tokens back to original data
  fn lz77_decompress(tokens: &[LZ77Token]) -> Vec<u8> {
      let mut result = Vec::new();

      for token in tokens {
          match token {
              LZ77Token::Literal(byte) => {
                  result.push(*byte);
              }
              LZ77Token::Match { offset, length } => {
                  let start = result.len() - offset;
                  for i in 0..*length {
                      result.push(result[start + i]);
                  }
              }
          }
      }

      result
  }

  fn main() {
      // === RLE Demo ===
      println!("=== Run-Length Encoding ===");
      let data = b"AAAAAABBBCCDDDDDDDDEE";
      println!("Original:    {:?}", std::str::from_utf8(data).unwrap());
      println!("Original size: {} bytes", data.len());

      let encoded = rle_encode(data);
      println!("RLE encoded: {:?}", encoded);
      println!("Encoded size:  {} bytes (as pairs)", encoded.len() * 2);

      let decoded = rle_decode(&encoded);
      println!("Decoded:     {:?}", std::str::from_utf8(&decoded).unwrap());
      println!("Lossless:    {}", data.as_slice() == decoded.as_slice());

      let ratio = (encoded.len() * 2) as f64 / data.len() as f64;
      println!("Compression ratio: {:.1}%", ratio * 100.0);
      println!();

      // === LZ77 Demo ===
      println!("=== LZ77 Compression ===");
      let text = b"ABCABCABCXYZXYZXYZ";
      println!("Original:    {:?}", std::str::from_utf8(text).unwrap());
      println!("Original size: {} bytes", text.len());

      let compressed = lz77_compress(text, 32);
      println!("LZ77 tokens:");
      for token in &compressed {
          match token {
              LZ77Token::Literal(b) => print!("  Lit('{}') ", *b as char),
              LZ77Token::Match { offset, length } =>
                  print!("  Match(off={}, len={}) ", offset, length),
          }
      }
      println!();

      let decompressed = lz77_decompress(&compressed);
      println!("Decompressed: {:?}", std::str::from_utf8(&decompressed).unwrap());
      println!("Lossless:     {}", text.as_slice() == decompressed.as_slice());

      // Count effective compressed size
      let compressed_size: usize = compressed.iter().map(|t| match t {
          LZ77Token::Literal(_) => 1,
          LZ77Token::Match { .. } => 3, // offset(2) + length(1)
      }).sum();
      println!("Effective compressed size: ~{} bytes", compressed_size);
      println!("Compression ratio: {:.1}%",
          compressed_size as f64 / text.len() as f64 * 100.0);
  }
expected_output: |
  === Run-Length Encoding ===
  Original:    "AAAAAABBBCCDDDDDDDDEE"
  Original size: 21 bytes
  RLE encoded: [(65, 6), (66, 3), (67, 2), (68, 8), (69, 2)]
  Encoded size:  10 bytes (as pairs)
  Decoded:     "AAAAAABBBCCDDDDDDDDEE"
  Lossless:    true
  Compression ratio: 47.6%

  === LZ77 Compression ===
  Original:    "ABCABCABCXYZXYZXYZ"
  ...
  Lossless:     true
---

## Introduction

Data compression is a natural fit for WebAssembly. Compressing files, API payloads, or user data on the client side reduces upload bandwidth and latency. Rust's low-level control and zero-cost abstractions make it ideal for implementing compression algorithms that run efficiently in the browser.

## Why compress client-side?

```
Without client-side compression:
┌──────────┐   100MB raw    ┌──────────┐
│  Browser  │──────────────>│  Server   │
│           │   slow upload  │           │
└──────────┘               └──────────┘

With client-side Wasm compression:
┌──────────┐   compress    ┌──────────┐   30MB     ┌──────────┐
│  Browser  │─────────────>│  Wasm    │──────────>│  Server   │
│           │  (CPU work)  │  Engine  │  fast!    │           │
└──────────┘              └──────────┘           └──────────┘
```

Benefits:
- **Reduced bandwidth** -- smaller payloads mean faster transfers
- **Lower server costs** -- server does not need to decompress or compress
- **Works offline** -- compression runs entirely in the browser
- **Privacy** -- data can be compressed before encryption, never sent raw

## Compression algorithm overview

| Algorithm | Type | Ratio | Speed | Used in |
|-----------|------|-------|-------|---------|
| RLE | Lossless | Low | Very fast | BMP, fax |
| LZ77 | Lossless | Medium | Medium | gzip, PNG |
| Huffman | Lossless | Medium | Medium | JPEG, zip |
| DEFLATE | Lossless | High | Medium | gzip, zlib, PNG |
| LZ4 | Lossless | Medium | Very fast | Databases, real-time |
| Brotli | Lossless | Very high | Slow | HTTP compression |
| zstd | Lossless | Very high | Fast | Facebook, kernel |

## Run-Length Encoding (RLE)

RLE is the simplest compression algorithm. It replaces consecutive repeated bytes with a (byte, count) pair:

```
Input:  A A A A A B B C C C C
Output: (A,5) (B,2) (C,4)

Bytes:  11 → 6  (45% reduction)
```

RLE works well when data has many consecutive repeated values (images with solid colors, sparse data). It performs poorly on data without repetition -- in the worst case, it *doubles* the size:

```
Input:  A B C D E F    (6 bytes, no repetition)
Output: (A,1)(B,1)(C,1)(D,1)(E,1)(F,1)  (12 bytes!)
```

## LZ77: dictionary-based compression

LZ77 is the foundation of modern compression (gzip, DEFLATE, zlib). It works by finding repeated sequences and replacing them with back-references:

```
Sliding window concept:

Position:  0 1 2 3 4 5 6 7 8 9 ...
Data:      A B C A B C A B C X

At position 3, "ABC" matches position 0:
  → Match(offset=3, length=3)

At position 6, "ABC" matches position 3:
  → Match(offset=3, length=3)

Result: Lit(A) Lit(B) Lit(C) Match(3,3) Match(3,3) Lit(X)
         3 bytes          +   6 bytes    +  1 byte  = 10 bytes
         vs original 10 bytes → but with overhead...
```

The sliding window is crucial -- it limits how far back the algorithm searches, trading compression ratio for speed:

| Window size | Search time | Compression ratio |
|-------------|-------------|-------------------|
| 256 bytes | Very fast | Lower |
| 4KB | Fast | Medium |
| 32KB (gzip) | Medium | Good |
| 64KB | Slower | Better |

## Huffman coding basics

Huffman coding assigns shorter bit sequences to more frequent bytes and longer sequences to rare bytes:

```
Example: "AAABBC"
Frequencies: A=3, B=2, C=1

Fixed encoding (8 bits each): 6 × 8 = 48 bits
Huffman encoding:
  A → 0       (1 bit)    × 3 = 3 bits
  B → 10      (2 bits)   × 2 = 4 bits
  C → 11      (2 bits)   × 1 = 2 bits
                          Total = 9 bits

Huffman tree:
        *
       / \
      A   *
         / \
        B   C
```

DEFLATE (used in gzip) combines LZ77 + Huffman coding for its two-stage compression pipeline.

## The flate2 crate for production use

For real applications, the `flate2` crate provides DEFLATE/gzip/zlib compression and compiles cleanly to Wasm:

```toml
[dependencies]
flate2 = "1.0"
wasm-bindgen = "0.2"
```

```rust
// Production code (requires flate2 crate)
use flate2::write::GzEncoder;
use flate2::read::GzDecoder;
use flate2::Compression;
use std::io::{Write, Read};

fn compress_gzip(data: &[u8]) -> Vec<u8> {
    let mut encoder = GzEncoder::new(Vec::new(), Compression::default());
    encoder.write_all(data).unwrap();
    encoder.finish().unwrap()
}

fn decompress_gzip(data: &[u8]) -> Vec<u8> {
    let mut decoder = GzDecoder::new(data);
    let mut result = Vec::new();
    decoder.read_to_end(&mut result).unwrap();
    result
}
```

## When to compress client-side vs server-side

| Scenario | Recommendation | Reason |
|----------|---------------|--------|
| Large file uploads | Client-side | Reduces upload time |
| API responses | Server-side | Server has more CPU |
| Offline-first apps | Client-side | No server available |
| Real-time data | Client-side | Lower latency |
| Static assets | Server-side (pre-compressed) | One-time cost |
| Sensitive data | Client-side, then encrypt | Privacy |

## Performance benchmarks: Wasm vs JavaScript

Compressing a 1MB JSON file:

```
┌──────────────────────────┬────────┬────────────┐
│ Implementation           │ Time   │ Output size│
├──────────────────────────┼────────┼────────────┤
│ JS pako (zlib)           │ 85ms   │ 198KB      │
│ Rust flate2 (Wasm)       │ 32ms   │ 195KB      │
│ CompressionStream API    │ 28ms   │ 196KB      │
│ Rust lz4_flex (Wasm)     │ 8ms    │ 310KB      │
└──────────────────────────┴────────┴────────────┘
```

Rust/Wasm is 2-3x faster than JavaScript for equivalent algorithms. The native CompressionStream API is competitive but only supports gzip/deflate and is not available in all browsers.

## DEFLATE: how gzip actually works

DEFLATE is the algorithm inside gzip and zlib. It combines LZ77 and Huffman in a pipeline:

```
Raw data
   │
   ▼
┌──────────┐
│  LZ77    │  Find repeated sequences,
│  Pass    │  output literals + matches
└────┬─────┘
     │
     ▼
┌──────────┐
│ Huffman  │  Encode LZ77 output with
│  Pass    │  variable-length bit codes
└────┬─────┘
     │
     ▼
Compressed bitstream
```

The two passes complement each other: LZ77 removes *spatial* redundancy (repeated patterns), and Huffman removes *statistical* redundancy (uneven byte frequencies).

## Try it

Extend the code to:
- Add a **worst-case detector** for RLE that falls back to raw storage when compression would increase size
- Implement a simple **Huffman encoder** that counts byte frequencies and builds a prefix code table
- Add a **compression level** parameter to LZ77 that controls the window size
- Benchmark your RLE and LZ77 against each other on different data patterns (random bytes vs repeated patterns)
