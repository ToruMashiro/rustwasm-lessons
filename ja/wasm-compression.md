---
title: "Wasm + 圧縮"
slug: wasm-compression
difficulty: intermediate
tags: [api]
order: 45
description: Rust/Wasmでクライアントサイドのデータ圧縮アルゴリズムを実装 — ランレングス符号化、LZ77、ハフマン符号化の基礎。
starter_code: |
  /// ランレングス符号化: 連続するバイトを圧縮する
  /// 例: [A, A, A, B, B, C] -> [(A, 3), (B, 2), (C, 1)]
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

  /// RLEデコード: ペアを元のデータに展開する
  fn rle_decode(encoded: &[(u8, u8)]) -> Vec<u8> {
      let mut result = Vec::new();
      for &(byte, count) in encoded {
          for _ in 0..count {
              result.push(byte);
          }
      }
      result
  }

  /// シンプルなLZ77圧縮
  /// 繰り返しシーケンスを見つけて(offset, length)ペアに置換する
  /// 出力: Literal(byte)またはMatch(offset, length)のシーケンス
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

          // 検索ウィンドウ: window_sizeバイト分だけ前を探す
          let search_start = if pos > window_size { pos - window_size } else { 0 };

          for offset_start in search_start..pos {
              let mut length = 0;
              while pos + length < data.len()
                  && data[offset_start + length] == data[pos + length]
                  && length < 255
              {
                  length += 1;
                  // 検索位置を超えて読み取らないようにする
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
              // 長さが3以上の場合のみマッチとしてエンコード（スペース節約）
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

  /// LZ77トークンを元のデータに展開する
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
      // === RLEデモ ===
      println!("=== ランレングス符号化 ===");
      let data = b"AAAAAABBBCCDDDDDDDDEE";
      println!("元データ:    {:?}", std::str::from_utf8(data).unwrap());
      println!("元のサイズ: {} バイト", data.len());

      let encoded = rle_encode(data);
      println!("RLEエンコード済み: {:?}", encoded);
      println!("エンコード後のサイズ:  {} バイト（ペアとして）", encoded.len() * 2);

      let decoded = rle_decode(&encoded);
      println!("デコード済み:     {:?}", std::str::from_utf8(&decoded).unwrap());
      println!("可逆:    {}", data.as_slice() == decoded.as_slice());

      let ratio = (encoded.len() * 2) as f64 / data.len() as f64;
      println!("圧縮率: {:.1}%", ratio * 100.0);
      println!();

      // === LZ77デモ ===
      println!("=== LZ77圧縮 ===");
      let text = b"ABCABCABCXYZXYZXYZ";
      println!("元データ:    {:?}", std::str::from_utf8(text).unwrap());
      println!("元のサイズ: {} バイト", text.len());

      let compressed = lz77_compress(text, 32);
      println!("LZ77トークン:");
      for token in &compressed {
          match token {
              LZ77Token::Literal(b) => print!("  Lit('{}') ", *b as char),
              LZ77Token::Match { offset, length } =>
                  print!("  Match(off={}, len={}) ", offset, length),
          }
      }
      println!();

      let decompressed = lz77_decompress(&compressed);
      println!("展開後: {:?}", std::str::from_utf8(&decompressed).unwrap());
      println!("可逆:     {}", text.as_slice() == decompressed.as_slice());

      // 実効圧縮サイズを計算
      let compressed_size: usize = compressed.iter().map(|t| match t {
          LZ77Token::Literal(_) => 1,
          LZ77Token::Match { .. } => 3, // offset(2) + length(1)
      }).sum();
      println!("実効圧縮サイズ: 約{} バイト", compressed_size);
      println!("圧縮率: {:.1}%",
          compressed_size as f64 / text.len() as f64 * 100.0);
  }
expected_output: |
  === ランレングス符号化 ===
  元データ:    "AAAAAABBBCCDDDDDDDDEE"
  元のサイズ: 21 バイト
  RLEエンコード済み: [(65, 6), (66, 3), (67, 2), (68, 8), (69, 2)]
  エンコード後のサイズ:  10 バイト（ペアとして）
  デコード済み:     "AAAAAABBBCCDDDDDDDDEE"
  可逆:    true
  圧縮率: 47.6%

  === LZ77圧縮 ===
  元データ:    "ABCABCABCXYZXYZXYZ"
  ...
  可逆:     true
---

## はじめに

データ圧縮はWebAssemblyに自然に適したユースケースです。ファイル、APIペイロード、ユーザーデータをクライアントサイドで圧縮することで、アップロード帯域幅とレイテンシを削減できます。Rustの低レベル制御とゼロコスト抽象化により、ブラウザ内で効率的に動作する圧縮アルゴリズムの実装に最適です。

## なぜクライアントサイドで圧縮するのか？

```
クライアントサイド圧縮なしの場合:
┌──────────┐   100MB 生データ  ┌──────────┐
│  ブラウザ  │──────────────>│  サーバー  │
│           │   遅いアップロード│           │
└──────────┘               └──────────┘

クライアントサイドWasm圧縮ありの場合:
┌──────────┐   圧縮        ┌──────────┐   30MB     ┌──────────┐
│  ブラウザ  │─────────────>│  Wasm    │──────────>│  サーバー  │
│           │  (CPU処理)   │  エンジン │  高速!    │           │
└──────────┘              └──────────┘           └──────────┘
```

メリット：
- **帯域幅の削減** -- ペイロードが小さくなり転送が高速化
- **サーバーコストの削減** -- サーバーが展開や圧縮を行う必要がない
- **オフライン対応** -- 圧縮はすべてブラウザ内で実行される
- **プライバシー** -- データを暗号化前に圧縮でき、生データが送信されない

## 圧縮アルゴリズムの概要

| アルゴリズム | 種類 | 圧縮率 | 速度 | 用途 |
|-----------|------|-------|-------|---------|
| RLE | 可逆 | 低 | 非常に高速 | BMP, FAX |
| LZ77 | 可逆 | 中 | 中程度 | gzip, PNG |
| Huffman | 可逆 | 中 | 中程度 | JPEG, zip |
| DEFLATE | 可逆 | 高 | 中程度 | gzip, zlib, PNG |
| LZ4 | 可逆 | 中 | 非常に高速 | データベース, リアルタイム |
| Brotli | 可逆 | 非常に高 | 低速 | HTTP圧縮 |
| zstd | 可逆 | 非常に高 | 高速 | Facebook, カーネル |

## ランレングス符号化（RLE）

RLEは最もシンプルな圧縮アルゴリズムです。連続する同じバイトを(バイト, カウント)ペアに置き換えます：

```
入力:  A A A A A B B C C C C
出力: (A,5) (B,2) (C,4)

バイト数:  11 → 6  (45%削減)
```

RLEは連続して繰り返される値が多いデータ（単色の画像、スパースデータ）に効果的です。繰り返しのないデータでは性能が悪く、最悪の場合サイズが*2倍*になります：

```
入力:  A B C D E F    (6バイト、繰り返しなし)
出力: (A,1)(B,1)(C,1)(D,1)(E,1)(F,1)  (12バイト!)
```

## LZ77：辞書ベースの圧縮

LZ77は現代の圧縮（gzip、DEFLATE、zlib）の基盤です。繰り返しシーケンスを見つけて後方参照に置き換えることで動作します：

```
スライディングウィンドウの概念:

位置:     0 1 2 3 4 5 6 7 8 9 ...
データ:   A B C A B C A B C X

位置3で"ABC"が位置0にマッチ:
  → Match(offset=3, length=3)

位置6で"ABC"が位置3にマッチ:
  → Match(offset=3, length=3)

結果: Lit(A) Lit(B) Lit(C) Match(3,3) Match(3,3) Lit(X)
       3バイト          +   6バイト    +  1バイト  = 10バイト
       元の10バイトに対して → ただしオーバーヘッドあり...
```

スライディングウィンドウは重要です -- アルゴリズムが後方を検索する範囲を制限し、圧縮率と速度のトレードオフを行います：

| ウィンドウサイズ | 検索時間 | 圧縮率 |
|-------------|-------------|-------------------|
| 256バイト | 非常に高速 | 低い |
| 4KB | 高速 | 中程度 |
| 32KB (gzip) | 中程度 | 良好 |
| 64KB | 低速 | より良好 |

## ハフマン符号化の基礎

ハフマン符号化は、頻度の高いバイトに短いビット列を、頻度の低いバイトに長いビット列を割り当てます：

```
例: "AAABBC"
頻度: A=3, B=2, C=1

固定符号化（各8ビット）: 6 × 8 = 48ビット
ハフマン符号化:
  A → 0       (1ビット)    × 3 = 3ビット
  B → 10      (2ビット)    × 2 = 4ビット
  C → 11      (2ビット)    × 1 = 2ビット
                            合計 = 9ビット

ハフマン木:
        *
       / \
      A   *
         / \
        B   C
```

DEFLATE（gzipで使用）はLZ77 + ハフマン符号化を2段階の圧縮パイプラインとして組み合わせます。

## プロダクション用途のflate2クレート

実際のアプリケーションでは、`flate2`クレートがDEFLATE/gzip/zlib圧縮を提供し、Wasmにクリーンにコンパイルできます：

```toml
[dependencies]
flate2 = "1.0"
wasm-bindgen = "0.2"
```

```rust
// プロダクションコード（flate2クレートが必要）
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

## クライアントサイド圧縮 vs サーバーサイド圧縮の使い分け

| シナリオ | 推奨 | 理由 |
|----------|---------------|--------|
| 大きなファイルのアップロード | クライアントサイド | アップロード時間の短縮 |
| APIレスポンス | サーバーサイド | サーバーの方がCPUが豊富 |
| オフラインファーストアプリ | クライアントサイド | サーバーが利用不可 |
| リアルタイムデータ | クライアントサイド | レイテンシの低減 |
| 静的アセット | サーバーサイド（事前圧縮） | 一度だけのコスト |
| 機密データ | クライアントサイドで圧縮後に暗号化 | プライバシー |

## パフォーマンスベンチマーク：Wasm vs JavaScript

1MBのJSONファイルの圧縮：

```
┌──────────────────────────┬────────┬────────────┐
│ 実装                      │ 時間   │ 出力サイズ │
├──────────────────────────┼────────┼────────────┤
│ JS pako (zlib)           │ 85ms   │ 198KB      │
│ Rust flate2 (Wasm)       │ 32ms   │ 195KB      │
│ CompressionStream API    │ 28ms   │ 196KB      │
│ Rust lz4_flex (Wasm)     │ 8ms    │ 310KB      │
└──────────────────────────┴────────┴────────────┘
```

Rust/Wasmは同等のアルゴリズムに対してJavaScriptより2〜3倍高速です。ネイティブのCompressionStream APIは競争力がありますが、gzip/deflateのみサポートしており、すべてのブラウザで利用可能ではありません。

## DEFLATE：gzipの実際の仕組み

DEFLATEはgzipとzlibの内部で使われるアルゴリズムです。LZ77とHuffmanをパイプラインで組み合わせます：

```
生データ
   │
   ▼
┌──────────┐
│  LZ77    │  繰り返しシーケンスを検索し、
│  パス    │  リテラルとマッチを出力
└────┬─────┘
     │
     ▼
┌──────────┐
│ Huffman  │  LZ77の出力を可変長
│  パス    │  ビットコードでエンコード
└────┬─────┘
     │
     ▼
圧縮ビットストリーム
```

2つのパスは相互に補完します：LZ77は*空間的*冗長性（繰り返しパターン）を除去し、Huffmanは*統計的*冗長性（バイト頻度の偏り）を除去します。

## 試してみよう

コードを拡張してみましょう：
- 圧縮でサイズが増加する場合に生データの格納にフォールバックするRLEの**ワーストケース検出器**を追加する
- バイト頻度をカウントしてプレフィックスコードテーブルを構築するシンプルな**ハフマンエンコーダー**を実装する
- ウィンドウサイズを制御するLZ77の**圧縮レベル**パラメータを追加する
- 異なるデータパターン（ランダムバイト vs 繰り返しパターン）でRLEとLZ77のベンチマークを比較する
