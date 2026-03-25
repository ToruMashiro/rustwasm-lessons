---
title: "Wasmストリームと非同期イテレータ"
slug: wasm-streams
difficulty: advanced
tags: [api]
order: 30
description: RustからReadableStream、WritableStream、TransformStreamを操作する — 大容量ファイルのストリーミング、バックプレッシャーの処理、wasm-streamsクレートによる非同期イテレータパイプラインの構築。
starter_code: |
  use std::collections::VecDeque;

  fn main() {
      println!("=== Wasmストリームと非同期イテレータ ===\n");

      // --- ReadableStreamのシミュレーション ---
      println!("--- ReadableStreamシミュレーション ---");

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
              println!("  ストリーム作成: {}バイトを{}チャンク（各{}バイト）に分割",
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
          println!("  チャンク読み取り ({:2}バイト): {:?}", chunk.len(), text);
      }
      println!("  ストリーム完了。合計: {}バイト\n", total_read);

      // --- バックプレッシャー付きWritableStreamのシミュレーション ---
      println!("--- バックプレッシャー付きWritableStream ---");

      struct WritableStream {
          buffer: Vec<u8>,
          capacity: usize,
          high_water_mark: usize,
          total_written: usize,
      }

      impl WritableStream {
          fn new(capacity: usize, high_water_mark: usize) -> Self {
              println!("  書き込みストリーム作成: capacity={}, high_water_mark={}",
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
                  return Err(format!("バッファ満杯 ({}/{})", self.buffer.len(), self.capacity));
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
                  println!("  {}バイト書き込み、バックプレッシャー: {}", chunk.len(), backpressure);
                  if backpressure {
                      let flushed = writable.flush();
                      println!("  {}バイトをフラッシュ（バックプレッシャー解消）", flushed.len());
                  }
              }
              Err(e) => println!("  書き込みエラー: {}", e),
          }
      }
      println!("  書き込み合計: {}バイト\n", writable.total_written);

      // --- TransformStreamのシミュレーション ---
      println!("--- TransformStreamシミュレーション ---");

      struct TransformStream {
          transform_fn: Box<dyn Fn(&[u8]) -> Vec<u8>>,
          name: String,
      }

      impl TransformStream {
          fn new(name: &str, f: Box<dyn Fn(&[u8]) -> Vec<u8>>) -> Self {
              println!("  変換ストリーム作成: '{}'", name);
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
      println!("\n  パイプライン: input -> uppercase -> prefix -> output");
      for chunk in &input_chunks {
          let step1 = uppercase.transform(chunk);
          let step2 = prefix.transform(&step1);
          println!("  {:?} -> {:?}",
              String::from_utf8_lossy(chunk),
              String::from_utf8_lossy(&step2));
      }

      // --- 非同期イテレータパターン ---
      println!("\n--- 非同期イテレータパターン ---");

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

      println!("\n--- ストリームサイズ見積もり ---");
      let file_size: u64 = 10 * 1024 * 1024; // 10 MB
      let chunk_size: u64 = 64 * 1024;        // 64 KB
      let num_chunks = (file_size + chunk_size - 1) / chunk_size;
      println!("  ファイル: {} MB", file_size / (1024 * 1024));
      println!("  チャンクサイズ: {} KB", chunk_size / 1024);
      println!("  合計チャンク数: {}", num_chunks);
      println!("  チャンクあたりメモリ: {} KB", chunk_size / 1024);
      println!("  ピークメモリ（3チャンクバッファ）: {} KB", 3 * chunk_size / 1024);
  }
expected_output: |
  === Wasmストリームと非同期イテレータ ===

  --- ReadableStreamシミュレーション ---
    ストリーム作成: 51バイトを6チャンク（各10バイト）に分割
    チャンク読み取り (10バイト): "Hello, thi"
    チャンク読み取り (10バイト): "s is strea"
    チャンク読み取り (10バイト): "ming data "
    チャンク読み取り (10バイト): "flowing th"
    チャンク読み取り (10バイト): "rough Wasm"
    チャンク読み取り ( 1バイト): "!"
    ストリーム完了。合計: 51バイト

  --- バックプレッシャー付きWritableStream ---
    書き込みストリーム作成: capacity=30, high_water_mark=20
    7バイト書き込み、バックプレッシャー: false
    7バイト書き込み、バックプレッシャー: false
    7バイト書き込み、バックプレッシャー: true
    21バイトをフラッシュ（バックプレッシャー解消）
    7バイト書き込み、バックプレッシャー: false
    書き込み合計: 28バイト

  --- TransformStreamシミュレーション ---
    変換ストリーム作成: 'uppercase'
    変換ストリーム作成: 'prefix'

    パイプライン: input -> uppercase -> prefix -> output
    "hello " -> "[OUT] HELLO "
    "world " -> "[OUT] WORLD "
    "from wasm" -> "[OUT] FROM WASM"

  --- 非同期イテレータパターン ---
    yield[0]: chunk_a
    yield[1]: chunk_b
    yield[2]: chunk_c

  --- ストリームサイズ見積もり ---
    ファイル: 10 MB
    チャンクサイズ: 64 KB
    合計チャンク数: 160
    チャンクあたりメモリ: 64 KB
    ピークメモリ（3チャンクバッファ）: 192 KB
---

## Webストリームとは？

[Streams API](https://developer.mozilla.org/en-US/docs/Web/API/Streams_API)を使うと、ファイル全体をメモリに読み込む代わりに、データをインクリメンタルに処理できます。大容量ファイルを扱うWasmでは不可欠です — 500MBの動画をリニアメモリに一度に全部コピーしたくはありません。

```
┌──────────────────────────────────────────────────────────┐
│                  ストリームアーキテクチャ                   │
│                                                          │
│  ソース ──→ ReadableStream ──→ TransformStream ──→ シンク │
│             （生成）           （変換）       WritableStream │
│                                               （消費）     │
│                                                          │
│  ◄────────── バックプレッシャー信号が上流に流れる ────────►│
└──────────────────────────────────────────────────────────┘
```

ストリームには3種類あります：

| ストリーム種別 | 目的 | 方向 |
|------------|---------|-----------|
| `ReadableStream` | データチャンクを生成 | ソース → コンシューマ |
| `WritableStream` | データチャンクを消費 | プロデューサ → シンク |
| `TransformStream` | データを転送中に変換 | 入力 → 出力 |

## wasm-streamsクレート

[wasm-streams](https://crates.io/crates/wasm-streams)クレートはWeb Streams APIのRustバインディングを提供し、JSストリームを`futures`クレートのRust `Stream`/`Sink`型に変換します。

```toml
[dependencies]
wasm-streams = "0.4"
futures = "0.3"
wasm-bindgen = "0.2"
wasm-bindgen-futures = "0.4"
web-sys = { version = "0.3", features = ["ReadableStream", "WritableStream"] }
```

## Fetchレスポンスをストリームとして読む

最も一般的なユースケースは`fetch()`レスポンスのストリーミングです：

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

    // レスポンスボディからReadableStreamを取得
    let raw_body = resp.body().unwrap();
    let body = ReadableStream::from_raw(raw_body);

    // RustのStreamに変換
    let mut reader = body.into_stream();

    let mut total_bytes = 0u64;
    while let Some(chunk) = reader.next().await {
        let chunk = chunk?;
        let array: js_sys::Uint8Array = chunk.dyn_into()?;
        total_bytes += array.length() as u64;
        // ここでチャンクを処理...
    }

    web_sys::console::log_1(
        &format!("{}バイトをストリーミング", total_bytes).into()
    );
    Ok(())
}
```

## バックプレッシャー

バックプレッシャーは、高速なプロデューサが低速なコンシューマを圧倒するのを防ぐ仕組みです。これがないと、メモリ内に無制限にデータがバッファリングされます。

```
高速プロデューサ         低速コンシューマ
     │                      │
     │──── チャンク1 ───────►│ （処理中...）
     │──── チャンク2 ───────►│ （まだチャンク1を処理中）
     │──── チャンク3 ──► 待機│ ← バックプレッシャー信号
     │     （一時停止）       │
     │◄── 準備完了 ─────────│ （チャンク1の処理完了）
     │──── チャンク3 ───────►│ （再開）
```

Streams APIは**ハイウォーターマーク**と**キューサイズ**を通じてバックプレッシャーを自動的に処理します：

```rust
// wasm-streamsでは、バックプレッシャーは透過的に処理される：
let mut writer = writable_stream.into_sink();

// 内部キューが満杯の場合、自動的に待機する
writer.send(chunk).await?;  // バックプレッシャー時にサスペンドする可能性あり
```

### ハイウォーターマークの設定

```js
// JavaScript側 — キューイング戦略の設定
const stream = new ReadableStream({
    start(controller) { /* ... */ },
    pull(controller) { /* ... */ },
}, {
    highWaterMark: 3,  // バックプレッシャー前に最大3チャンクをバッファ
});
```

## TransformStream：データの転送中処理

`TransformStream`はReadableStreamとWritableStreamの間に位置し、通過するデータを変換します：

```rust
use wasm_streams::TransformStream;

#[wasm_bindgen]
pub fn create_uppercase_transform() -> web_sys::TransformStream {
    let transform = TransformStream::new(
        // 変換関数
        |chunk: JsValue, controller: &TransformStreamDefaultController| {
            let input: js_sys::Uint8Array = chunk.dyn_into().unwrap();
            let mut data = vec![0u8; input.length() as usize];
            input.copy_to(&mut data);

            // 変換: ASCIIを大文字に
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

## ストリームのパイプ接続

ストリームをパイプラインとして連結できます：

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

        // Wasmリニアメモリ内で処理
        let mut data = vec![0u8; len];
        array.copy_to(&mut data);

        // 例: チェックサムの計算
        let checksum: u32 = data.iter().map(|&b| b as u32).sum();

        processed_bytes += len as u64;
        chunk_count += 1;
    }

    Ok(())
}
```

## 大容量ファイルのストリーミング：完全な例

```
┌─────────┐    ┌───────────┐    ┌──────────────┐    ┌────────┐
│  ファイル │───►│ Readable  │───►│  Transform   │───►│ DBに   │
│  入力     │    │  Stream   │    │  （Wasm内）   │    │ 書き込み│
│ （ディスク）│    │ 64KB      │    │  圧縮/       │    │        │
│          │    │ チャンク   │    │  暗号化       │    │        │
└─────────┘    └───────────┘    └──────────────┘    └────────┘
                                       │
                              同時にメモリ内にあるのは
                              1〜3チャンクだけ！
```

```js
// JavaScript側: パイプラインの接続
const fileStream = file.stream();  // File APIからのReadableStream
const transform = wasm.create_compression_transform();

const compressed = fileStream.pipeThrough(transform);
const writer = getStorageWriter();
await compressed.pipeTo(writer);
```

## Wasm向けRustの非同期イテレータ

Rustの`Stream`トレイト（`futures`クレート）は`Iterator`の非同期版です：

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

## メモリに関する考慮事項

| アプローチ | メモリ使用量 | レイテンシ |
|----------|-------------|---------|
| ファイル全体を読み込み | O(ファイルサイズ) | 高い（全ダウンロード完了を待つ） |
| 64KBチャンクでストリーミング | O(64KB * バッファ数) | 低い（データ到着時に処理） |
| バックプレッシャー付きストリーミング | O(ハイウォーターマーク) | 適応的 |

100MBのファイルを64KBチャンクで処理する場合：

- **ストリーミングなし：** Wasmリニアメモリに100MB
- **ストリーミングあり：** Wasmリニアメモリに約192KB（3チャンクバッファ）

## まとめ

1. **Webストリーム**はデータをインクリメンタルに処理でき、大きなメモリ割り当てを回避する
2. **wasm-streams**はJS Streams APIとRustの`futures::Stream`および`futures::Sink`を橋渡しする
3. **バックプレッシャー**は高速プロデューサが低速コンシューマを圧倒するのを防ぐ — Streams APIが自動的に処理する
4. **TransformStream**はWasmが入力全体をバッファリングせずに転送中のデータを処理できる
5. **64KBチャンク**は良いデフォルトチャンクサイズ — Wasmページサイズと一致し、スループットとメモリのバランスが取れている
6. Wasmアプリケーションでは数MB以上のファイルには常にストリーミングを使用すること
