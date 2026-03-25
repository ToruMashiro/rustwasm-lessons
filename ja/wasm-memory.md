---
title: Wasmメモリモデル
slug: wasm-memory
difficulty: intermediate
tags: [data-structures]
order: 21
description: WebAssemblyのリニアメモリの仕組みを理解する — メモリの確保、拡張、ビュー、そしてRustとJavaScript間の効率的なデータ共有。
starter_code: |
  fn main() {
      // Wasmリニアメモリの概念をシミュレーション

      // 1. リニアメモリは連続したバイト配列
      let mut memory: Vec<u8> = vec![0; 65536]; // 1ページ = 64KB
      println!("Memory size: {} bytes ({} pages)", memory.len(), memory.len() / 65536);

      // 2. 特定のオフセットにデータを書き込む
      let message = "Hello, Wasm!";
      let offset = 1024;
      for (i, byte) in message.bytes().enumerate() {
          memory[offset + i] = byte;
      }

      // 3. データを読み戻す
      let read: String = memory[offset..offset + message.len()]
          .iter()
          .map(|&b| b as char)
          .collect();
      println!("Read from offset {}: \"{}\"", offset, read);

      // 4. メモリの拡張
      let old_size = memory.len();
      memory.resize(old_size + 65536, 0); // 1ページ分拡張
      println!("Grew to: {} bytes ({} pages)", memory.len(), memory.len() / 65536);

      // 5. 数値の格納（リトルエンディアン、Wasmと同様）
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

## リニアメモリとは？

WebAssemblyには**リニアメモリ**と呼ばれる単一の連続メモリブロックがあります。RustとJavaScriptの両方が読み書きできる巨大な `Vec<u8>` と考えてください。

```
Address:  0x0000    0x0100    0x0200    ...    0xFFFF
         ┌─────────┬─────────┬─────────┬──────┐
Memory:  │ stack   │ heap    │ data    │ ...  │
         │ data    │ allocs  │ section │      │
         └─────────┴─────────┴─────────┴──────┘
         └────────── 1 page = 64KB ───────────┘
```

主な特徴:
- **バイトアドレス指定** — すべてのバイトにインデックスがある（0, 1, 2, ...）
- **ページ単位** — メモリは64KBページ単位で拡張（1ページ = 65,536バイト）
- **共有可能** — RustとJSの両方が同じバイトにアクセスできる
- **境界チェック付き** — 範囲外メモリアクセスはトラップする（ブラウザはクラッシュしない）

## Rustがリニアメモリをどう使うか

RustをWasmにコンパイルすると、コンパイラは以下のようにメモリを配置します:

```
┌──────────────┬──────────────┬──────────────┬──────────────┐
│ Static data  │ Stack        │ Heap         │ Free space   │
│ (strings,    │ (local vars, │ (Vec, String,│ (grows →)    │
│  constants)  │  call frames)│  Box allocs) │              │
└──────────────┴──────────────┴──────────────┴──────────────┘
0x0000                                                  0xFFFF...
```

## JavaScriptからメモリにアクセスする

JavaScriptはWasmメモリを `ArrayBuffer` として参照します:

```js
// init()の後、wasm.memoryが利用可能になる
const memory = wasm.memory;

// バッファへの型付きビューを作成
const bytes = new Uint8Array(memory.buffer);
const u32s = new Uint32Array(memory.buffer);
const f64s = new Float64Array(memory.buffer);

// アドレス1024のバイトを読み取る
const value = bytes[1024];

// アドレス2048（バイトオフセット）にfloatを書き込む
f64s[2048 / 8] = 3.14;  // Float64Arrayのインデックス = バイトオフセット / 8
```

## RustとJS間のデータ受け渡し

### 方法1: wasm-bindgenによるコピー（シンプルで安全）

```rust
#[wasm_bindgen]
pub fn process(data: &[u8]) -> Vec<u8> {
    // dataはJSからWasmメモリにコピーされる
    let mut result = data.to_vec();
    // ... resultを加工 ...
    result  // WasmからJSにコピーして返される
}
```

### 方法2: ポインタ + 長さ（ゼロコピー、上級者向け）

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

// Wasmメモリへの直接ビュー — ゼロコピー！
const view = new Uint8Array(wasm.memory.buffer, ptr, len);

// その場で変更 — Rust側にも反映される
view[0] = 42;
```

## メモリの拡張

Wasmメモリは小さく始まり、必要に応じて拡張されます:

```rust
// Vec/Stringがより多くの領域を必要とすると、Rustが自動的にメモリを拡張する
let mut big_vec = Vec::with_capacity(1_000_000); // メモリが拡張される可能性がある
```

```js
// JSから手動でチェック・拡張することも可能
console.log(wasm.memory.buffer.byteLength); // 現在のサイズ
wasm.memory.grow(10); // 10ページ（640KB）拡張

// 重要: grow()の後、すべてのArrayBufferビューは無効になる！
// 拡張後にビューを再作成する:
const freshView = new Uint8Array(wasm.memory.buffer);
```

## よくある落とし穴

| 落とし穴 | 原因 | 対策 |
|---------|-----|-----|
| メモリ拡張後のArrayBufferビューの失効 | `memory.grow()` が古いバッファを切り離す | メモリ拡張の可能性がある操作の後にビューを再作成する |
| アライメントエラー | Float64Arrayは8バイトアライメントが必要 | `ptr as usize % 8 == 0` チェックを使う |
| メモリリーク | Rustのアロケーションが解放されない | 構造体を適切にドロップし、`forget()` の多用を避ける |
| 解放済みメモリの読み取り | 古いJS参照によるuse-after-free | Rust構造体を解放する前にデータをコピーしておく |

## メモリサイズの目安

| 内容 | 一般的なサイズ |
|---------|-------------|
| 小さなWasmモジュール | 50-200KB |
| Wasm + web-sysバインディング | 200KB-1MB |
| デフォルトのリニアメモリ | 1ページ（64KB） |
| 大規模アプリケーション | 10-50MB |
| ブラウザの上限 | 約2-4GB |

## 試してみよう

**Run** をクリックして、リニアメモリの概念を実際に確認しましょう — メモリの作成、オフセットでの読み書き、ページの拡張、型付き値の格納を体験できます。
