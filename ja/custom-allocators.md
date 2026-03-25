---
title: Wasmのカスタムアロケータ
slug: custom-allocators
difficulty: advanced
tags: [data-structures]
order: 35
description: Wasmのメモリアロケーションの仕組みを理解し、独自のバンプアロケータ、アリーナアロケータ、GlobalAlloc実装をRustで構築します。
starter_code: |
  use std::alloc::{GlobalAlloc, Layout};
  use std::cell::UnsafeCell;
  use std::ptr;

  // シンプルなバンプアロケータ
  // ポインタを前方に移動してアロケートし、個々のオブジェクトは解放しない。
  // すべてのメモリはreset()で一括回収される。

  const ARENA_SIZE: usize = 4096;

  struct BumpAllocator {
      arena: UnsafeCell<[u8; ARENA_SIZE]>,
      offset: UnsafeCell<usize>,
  }

  // 安全性: アロケータはシングルスレッド（Wasmと同様）
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

          // 現在のオフセットをアライメント
          let align = layout.align();
          let aligned = (*offset + align - 1) & !(align - 1);
          let new_offset = aligned + layout.size();

          if new_offset > ARENA_SIZE {
              return ptr::null_mut(); // メモリ不足
          }

          *offset = new_offset;
          arena.as_mut_ptr().add(aligned)
      }

      unsafe fn dealloc(&self, _ptr: *mut u8, _layout: Layout) {
          // バンプアロケータ: 個別のdeallocはno-op。
          // メモリはreset()でのみ回収される。
      }
  }

  // 通常は次のように使用: #[global_allocator]
  // static ALLOC: BumpAllocator = BumpAllocator::new();
  // ただし、このデモでは手動で操作する。

  fn main() {
      let alloc = BumpAllocator::new();

      println!("=== バンプアロケータ デモ ===");
      println!("アリーナサイズ: {}バイト", ARENA_SIZE);
      println!("初期化後の使用量: {}バイト", alloc.used());

      // いくつかのオブジェクトをアロケート
      let layout_u32 = Layout::new::<u32>();
      let layout_u64 = Layout::new::<u64>();
      let layout_arr = Layout::array::<u8>(100).unwrap();

      unsafe {
          let p1 = alloc.alloc(layout_u32);
          println!("\nu32をアロケート (4バイト, アライメント4): 使用量 = {}", alloc.used());

          let p2 = alloc.alloc(layout_u64);
          println!("u64をアロケート (8バイト, アライメント8): 使用量 = {}", alloc.used());

          let p3 = alloc.alloc(layout_arr);
          println!("[u8;100]をアロケート (アライメント1): 使用量 = {}", alloc.used());

          // アロケートしたメモリに書き込みと読み取り
          *(p1 as *mut u32) = 42;
          *(p2 as *mut u64) = 123456789;
          ptr::write_bytes(p3, 0xAB, 100);

          println!("\n値:");
          println!("  *p1 (u32) = {}", *(p1 as *const u32));
          println!("  *p2 (u64) = {}", *(p2 as *const u64));
          println!("  *p3 [0]   = 0x{:02X}", *p3);
      }

      // リセットですべてのメモリを回収
      alloc.reset();
      println!("\nリセット後: 使用量 = {}バイト", alloc.used());
  }
expected_output: |
  === バンプアロケータ デモ ===
  アリーナサイズ: 4096バイト
  初期化後の使用量: 0バイト

  u32をアロケート (4バイト, アライメント4): 使用量 = 4
  u64をアロケート (8バイト, アライメント8): 使用量 = 16
  [u8;100]をアロケート (アライメント1): 使用量 = 116

  値:
    *p1 (u32) = 42
    *p2 (u64) = 123456789
    *p3 [0]   = 0xAB

  リセット後: 使用量 = 0バイト
---

## Wasmのメモリアロケーションの仕組み

WebAssemblyは単一のリニアメモリを持ちます — 連続した、拡張可能なバイト配列です。組み込みのヒープ、ガベージコレクタ、`malloc`はありません。Rustプログラムは独自のアロケータを持ち込む必要があります。

```
  Wasmリニアメモリ
  ┌──────────────────────────────────────────────────┐
  │ スタック │ 静的データ │          ヒープ →          │
  │ (下方に │ (.data,    │  （アロケータが管理）       │
  │  成長)  │  .rodata)  │                            │
  └──────────────────────────────────────────────────┘
  0x0000                                      memory.size
                                              (ページ × 64KB)
```

重要な事実：
- メモリは**64 KB**のページ単位で計測されます
- `memory.grow(n)`は`n`ページを追加します。ホストが制限を課している場合は失敗する可能性があります
- **メモリ保護はありません** — 範囲内のどのオフセットにもアクセス可能です
- アロケータは`.wasm`バイナリにリンクされ、サイズに加算されます

## デフォルトアロケータ: dlmalloc

デフォルトでは、Rustの`wasm32-unknown-unknown`ターゲットは**dlmalloc**（Doug Leaのmalloc）の移植版を使用します。これは、さまざまなアロケーションサイズの混合に最適化された汎用アロケータです。

```
  dlmallocの内部構造（簡略化）
  ┌────────────────────────────────────────────┐
  │  フリーリストビン（サイズクラス別）         │
  │  ┌─────┐ ┌─────┐ ┌──────┐ ┌───────────┐  │
  │  │ 8B  │ │ 16B │ │ 64B  │ │ 256B+     │  │
  │  │ フリー│ │ フリー│ │ フリー│ │ フリーリスト│  │
  │  │ リスト│ │ リスト│ │ リスト│ │ (ツリー)  │  │
  │  └─────┘ └─────┘ └──────┘ └───────────┘  │
  │                                            │
  │  アロケート時: 最適なビンを見つけチャンクを分割 │
  │  解放時: 隣接を結合しビンに返却              │
  └────────────────────────────────────────────┘
```

| 特性             | dlmalloc                |
|------------------|-------------------------|
| バイナリオーバーヘッド | ~10 KB             |
| アロケーション複雑度 | 小さい場合O(1)、大きい場合O(log n) |
| フラグメンテーション | 低い（結合あり）     |
| スレッドセーフ     | 不要（Wasmはシングルスレッド） |

## wee_alloc: 小さな代替手段

`wee_alloc`はWasm専用に設計されました — アロケーション速度とフラグメンテーション耐性を犠牲にして、約1 KBのバイナリフットプリントを実現します。

```rust
#[global_allocator]
static ALLOC: wee_alloc::WeeAlloc = wee_alloc::WeeAlloc::INIT;
```

最小限のブックキーピングでリンクフリーリストを維持することで動作します：

```
  wee_allocフリーリスト
  ┌──────┐    ┌──────┐    ┌──────┐
  │ セル │───>│ セル │───>│ セル │───> null
  │ 32B  │    │ 16B  │    │ 64B  │
  └──────┘    └──────┘    └──────┘

  アロケート: リストを走査し、最初に適合するものを見つける
  解放: リストの先頭に追加（結合なし！）
```

> **注意**: `wee_alloc`は隣接するフリーブロックを結合しません。時間の経過とともにメモリが深刻にフラグメント化する可能性があります。現在メンテナンスされていません — 新しいプロジェクトには`lol_alloc`や`talc`を検討してください。

## バンプアロケータ

バンプアロケータは、最もシンプルで最速のアロケータです。各アロケーション時に前方に移動する単一のポインタを管理します。個別の解放は不可能で、すべてのメモリは一括で回収されます。

```
  バンプアロケータの状態

  3回のアロケーション後:
  ┌───────┬────────┬──────────────┬─────────────────────┐
  │ obj A │ obj B  │   obj C      │     空き領域         │
  └───────┴────────┴──────────────┴─────────────────────┘
  ^                                ^                     ^
  base                           offset                 end

  アロケート:  offset += size（アライメント後）
  解放:       no-op
  リセット:   offset = base（すべてを回収）
```

### バンプアロケーションを使うべき場面

| ユースケース                     | 適している？ |
|---------------------------------|-----------|
| フレームごとのゲームアロケーション | はい      |
| リクエストスコープのWebハンドラデータ | はい   |
| 短命な計算用スクラッチメモリ     | はい      |
| 多様なライフタイムの長期実行     | いいえ    |
| 個別のオブジェクト解放が必要     | いいえ    |

### アライメント

アロケータは各型のアライメント要件を尊重する必要があります。Wasmでは`u32`は4バイトアライメント、`u64`は8バイトアライメントが必要です：

```
  アライメント前:    offset = 5
  要求アライメント:  8

  aligned = (5 + 8 - 1) & !(8 - 1)
          = 12 & !7
          = 12 & 0xFFFF_FFF8
          = 8

  メモリ:
  ┌─┬─┬─┬─┬─┬─┬─┬─┬─────────────────┐
  │0│1│2│3│4│ │ │ │8  9 10 11 12 ... │
  └─┴─┴─┴─┴─┴─┴─┴─┴─────────────────┘
                ^   ^
           無駄な   アライメント済みオフセット
          (パディング)
```

## アリーナアロケーション

アリーナは、現在のチャンクが一杯になった時に新しいチャンクを割り当てて拡張するバンプアロケータです：

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
        // アライメント
        let align = layout.align();
        let padding = self.current.align_offset(align);
        let needed = padding + layout.size();

        if needed > self.remaining {
            // 新しいチャンクをアロケート
            let size = needed.max(4096);
            let chunk = vec![0u8; size];
            let ptr = chunk.as_ptr() as *mut u8;
            self.chunks.push(chunk);
            self.current = ptr;
            self.remaining = size;
            return self.alloc(layout); // リトライ
        }

        unsafe {
            let result = self.current.add(padding);
            self.current = self.current.add(needed);
            self.remaining -= needed;
            result
        }
    }

    fn reset(&mut self) {
        // 最初のチャンクのみ保持
        self.chunks.truncate(1);
        let chunk = &self.chunks[0];
        self.current = chunk.as_ptr() as *mut u8;
        self.remaining = chunk.len();
    }
}
```

```
  複数チャンクのアリーナ:

  チャンク0（満杯）     チャンク1（満杯）     チャンク2（アクティブ）
  ┌──────────────┐     ┌──────────────┐     ┌───────┬────────┐
  │██████████████│     │██████████████│     │███████│  空き  │
  └──────────────┘     └──────────────┘     └───────┴────────┘
                                              ^current
```

## GlobalAllocトレイト

RustのGlobalAllocトレイトは、すべてのアロケータが実装すべきインターフェースです：

```rust
use std::alloc::{GlobalAlloc, Layout};

unsafe trait GlobalAlloc {
    unsafe fn alloc(&self, layout: Layout) -> *mut u8;
    unsafe fn dealloc(&self, ptr: *mut u8, layout: Layout);

    // オプション — デフォルト実装はalloc + コピーを呼ぶ
    unsafe fn alloc_zeroed(&self, layout: Layout) -> *mut u8 { ... }
    unsafe fn realloc(&self, ptr: *mut u8, layout: Layout, new_size: usize) -> *mut u8 { ... }
}
```

### Layout

`Layout`はメモリ要求のサイズとアライメントを記述します：

```rust
let layout = Layout::new::<u32>();
assert_eq!(layout.size(), 4);
assert_eq!(layout.align(), 4);

let layout = Layout::array::<f64>(10).unwrap();
assert_eq!(layout.size(), 80);    // 10 * 8
assert_eq!(layout.align(), 8);

let layout = Layout::from_size_align(256, 16).unwrap();
```

### アロケータの登録

```rust
#[global_allocator]
static ALLOC: MyAllocator = MyAllocator::new();
```

これにより、バイナリ内の**すべての**ヒープアロケーション（`Box`、`Vec`、`String`、`HashMap`など）のデフォルトアロケータが置き換えられます。

## アロケータの比較

| アロケータ   | バイナリサイズ | アロケート速度 | 解放速度 | フラグメンテーション | 最適な用途            |
|-------------|-------------|-------------|----------|---------------|-----------------------|
| dlmalloc    | ~10 KB      | 高速        | 高速     | 低い          | 汎用                  |
| wee_alloc   | ~1 KB       | 中速        | 高速     | 高い          | 小さなWasmバイナリ     |
| バンプ       | ~0.1 KB     | 最速        | N/A      | なし*         | フレームごと/スコープ  |
| アリーナ     | ~0.5 KB     | 非常に高速  | N/A      | なし*         | バッチ処理             |
| lol_alloc   | ~0.2 KB     | 中速        | 低速     | 中程度        | 最小限のWasm           |
| talc        | ~5 KB       | 高速        | 高速     | 低い          | 汎用（モダン）         |

*個別の解放が行われないため、フラグメンテーションは発生しません。

## プールアロケータパターン

頻繁にアロケートと解放が行われる型（例：ゲームエンティティ、ネットワークパケット）には、プールアロケータが固定数のスロットを事前にアロケートします：

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
  容量8のPool<Entity>:

  ┌─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┐
  │  E  │     │  E  │  E  │     │  E  │     │  E  │
  │ [0] │ [1] │ [2] │ [3] │ [4] │ [5] │ [6] │ [7] │
  └─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┘
   使用  空き  使用  使用  空き  使用  空き  使用

  free_indices = [6, 4, 1]  （スタック順）
```

## まとめ

Wasmのリニアメモリモデルは、アロケーションを完全に制御できることを意味します。デフォルトのdlmallocはほとんどのアプリケーションで十分に動作します。サイズが重要なビルドでは、小さなアロケータに置き換えましょう。予測可能なライフタイムを持つパフォーマンス重視のワークロードでは、バンプアロケータとアリーナアロケータが、すべての解放を単一のリセットに集約することで比類なき速度を実現します。
