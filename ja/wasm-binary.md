---
title: Wasmバイナリフォーマット
slug: wasm-binary
difficulty: advanced
tags: [getting-started]
order: 27
description: .wasmバイナリフォーマットを理解する — モジュール、セクション、WATテキストフォーマット、ブラウザがWasmをロード・インスタンス化する仕組み。
starter_code: |
  fn main() {
      println!("=== Wasm Binary Format ===\n");

      // すべての.wasmファイルの先頭にあるマジックバイト
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
          (1, "Type", "関数シグネチャ"),
          (2, "Import", "インポートされた関数/メモリ"),
          (3, "Function", "関数宣言"),
          (5, "Memory", "リニアメモリ定義"),
          (6, "Global", "グローバル変数"),
          (7, "Export", "エクスポートされた関数/メモリ"),
          (10, "Code", "関数本体（実際のコード）"),
          (11, "Data", "静的データ（文字列、定数）"),
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
    Section  1: Type       — 関数シグネチャ
    Section  2: Import     — インポートされた関数/メモリ
    Section  3: Function   — 関数宣言
    Section  5: Memory     — リニアメモリ定義
    Section  6: Global     — グローバル変数
    Section  7: Export     — エクスポートされた関数/メモリ
    Section 10: Code       — 関数本体（実際のコード）
    Section 11: Data       — 静的データ（文字列、定数）
---

## .wasmバイナリ

すべての `.wasm` ファイルは8バイトから始まります:

```
00 61 73 6D    ← マジックナンバー: "\0asm"
01 00 00 00    ← バージョン: 1
```

ヘッダーの後、ファイルには**セクション**が含まれます — 各セクションにはタイプID、サイズ、内容があります。

## セクション

| ID | 名前 | 内容 |
|----|------|----------|
| 0 | Custom | デバッグ情報、名前、メタデータ |
| 1 | Type | 関数型シグネチャ `(i32, i32) -> i32` |
| 2 | Import | JSからインポートされた関数/メモリ/グローバル |
| 3 | Function | 関数インデックス → 型シグネチャのマッピング |
| 4 | Table | 間接関数呼び出しテーブル |
| 5 | Memory | リニアメモリ宣言（初期 + 最大ページ数） |
| 6 | Global | グローバル変数宣言 |
| 7 | Export | JavaScriptに公開される関数/メモリ |
| 8 | Start | インスタンス化時に呼ばれる関数 |
| 9 | Element | テーブル初期化データ |
| 10 | Code | 実際の関数バイトコード |
| 11 | Data | 静的データ（文字列、定数） |

## WAT: テキストフォーマット

WAT（WebAssembly Text）はWasmの人間が読める形式です。相互に変換できます:

```bash
# バイナリ → テキスト
wasm2wat my_module.wasm -o my_module.wat

# テキスト → バイナリ
wat2wasm my_module.wat -o my_module.wasm
```

### WATでのシンプルな関数

```wat
(module
  ;; Typeセクション: 関数シグネチャ
  (type $add_type (func (param i32 i32) (result i32)))

  ;; Functionセクション + Codeセクション
  (func $add (type $add_type) (param $a i32) (param $b i32) (result i32)
    local.get $a
    local.get $b
    i32.add
  )

  ;; Exportセクション
  (export "add" (func $add))
)
```

これは以下のRustコードと同じものにコンパイルされます:

```rust
#[wasm_bindgen]
pub fn add(a: i32, b: i32) -> i32 {
    a + b
}
```

## Wasm命令セット

Wasmは**スタックマシン**を使います — 命令がスタック上の値をプッシュ/ポップします:

```wat
;; (a + b) * 2 を計算
local.get $a     ;; スタック: [a]
local.get $b     ;; スタック: [a, b]
i32.add          ;; スタック: [a+b]
i32.const 2      ;; スタック: [a+b, 2]
i32.mul          ;; スタック: [(a+b)*2]
```

### よく使う命令

| 命令 | 動作 |
|-------------|-------------|
| `local.get $x` | ローカル変数をプッシュ |
| `local.set $x` | ポップしてローカルに格納 |
| `i32.const N` | 定数をプッシュ |
| `i32.add` | 2つポップして合計をプッシュ |
| `i32.mul` | 2つポップして積をプッシュ |
| `i32.lt_s` | 2つポップして小さければ1をプッシュ |
| `if...else...end` | 条件分岐 |
| `loop...br_if...end` | 条件付きブレークのループ |
| `call $fn` | 関数呼び出し |
| `return` | 関数からの復帰 |

## ブラウザがWasmをロードする方法

```js
// ステップ1: バイナリを取得
const response = await fetch('module.wasm');

// ステップ2: コンパイル（ストリーミング = ダウンロード中にコンパイル）
const module = await WebAssembly.compileStreaming(response);

// ステップ3: インスタンス化（インポートのリンク、メモリの確保）
const instance = await WebAssembly.instantiate(module, {
    env: {
        log: (ptr, len) => { /* WasmがインポートしたJS関数 */ }
    }
});

// ステップ4: エクスポートされた関数を呼び出す
const result = instance.exports.add(2, 3);
```

### ストリーミングコンパイル

```
Download:  ████████████████████░░░░░
Compile:   ░░░████████████████████░░
Execute:   ░░░░░░░░░░░░░░░░░░░░░░██
```

ブラウザは**ダウンロード中に**Wasmをコンパイルします — そのため100KBのWasmファイルはダウンロード完了直後に実行可能な状態になることがあります。

## .wasmファイルの検査

```bash
# エクスポートされたものを確認
wasm-objdump -x module.wasm

# 逆アセンブリを確認
wasm-objdump -d module.wasm

# セクションサイズを確認
wasm-objdump -h module.wasm

# 読みやすいテキストに変換
wasm2wat module.wasm
```

## 試してみよう

**Run** をクリックして、バイナリフォーマットの構造を確認しましょう — マジックバイト、バージョン、そして.wasmファイルを構成するすべてのセクションタイプを確認できます。
