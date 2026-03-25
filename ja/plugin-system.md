---
title: Wasmプラグインシステムの構築
slug: plugin-system
difficulty: advanced
tags: [getting-started]
order: 38
description: WebAssemblyを使ったプラグインアーキテクチャを設計します — サンドボックス化、ポータブル、言語非依存 — トレイトベースのホスト-ゲスト通信とリソース制限付き。
starter_code: |
  use std::collections::HashMap;

  // 実際のWasmプラグインアーキテクチャ（Figma、VS Code、Envoy）で
  // 使われるパターンを純粋なRustでデモするプラグインシステム。

  // --- Pluginトレイト: すべてのプラグインが実装すべきインターフェース ---
  trait Plugin {
      fn name(&self) -> &str;
      fn version(&self) -> &str;
      fn on_init(&mut self, config: &HashMap<String, String>);
      fn on_event(&mut self, event: &str, payload: &str) -> Option<String>;
      fn on_shutdown(&mut self);
  }

  // --- ホストランタイム: プラグインを読み込み管理する ---
  struct PluginHost {
      plugins: Vec<Box<dyn Plugin>>,
      event_log: Vec<String>,
      memory_limit: usize,   // バイト
      call_limit: u64,       // イベントごとの最大呼び出し回数
  }

  impl PluginHost {
      fn new(memory_limit: usize, call_limit: u64) -> Self {
          PluginHost {
              plugins: Vec::new(),
              event_log: Vec::new(),
              memory_limit,
              call_limit,
          }
      }

      fn register(&mut self, plugin: Box<dyn Plugin>) {
          println!("[ホスト] プラグインを登録: {} v{}", plugin.name(), plugin.version());
          self.plugins.push(plugin);
      }

      fn init_all(&mut self, config: &HashMap<String, String>) {
          println!("[ホスト] {}個のプラグインを初期化中...", self.plugins.len());
          for plugin in &mut self.plugins {
              plugin.on_init(config);
              println!("[ホスト]   {} 初期化完了", plugin.name());
          }
      }

      fn dispatch_event(&mut self, event: &str, payload: &str) {
          println!("[ホスト] イベントをディスパッチ: '{}'", event);
          self.event_log.push(event.to_string());

          for plugin in &mut self.plugins {
              match plugin.on_event(event, payload) {
                  Some(response) => {
                      println!("[ホスト]   {} が応答: {}", plugin.name(), response);
                  }
                  None => {
                      println!("[ホスト]   {} はイベントを無視", plugin.name());
                  }
              }
          }
      }

      fn shutdown_all(&mut self) {
          println!("[ホスト] プラグインをシャットダウン中...");
          for plugin in &mut self.plugins {
              plugin.on_shutdown();
          }
      }
  }

  // --- プラグイン例: ワードカウンター ---
  struct WordCountPlugin {
      total_words: usize,
  }

  impl WordCountPlugin {
      fn new() -> Self {
          WordCountPlugin { total_words: 0 }
      }
  }

  impl Plugin for WordCountPlugin {
      fn name(&self) -> &str { "word-counter" }
      fn version(&self) -> &str { "1.0.0" }

      fn on_init(&mut self, _config: &HashMap<String, String>) {
          self.total_words = 0;
      }

      fn on_event(&mut self, event: &str, payload: &str) -> Option<String> {
          if event == "text:input" {
              let count = payload.split_whitespace().count();
              self.total_words += count;
              Some(format!("words={} total={}", count, self.total_words))
          } else {
              None
          }
      }

      fn on_shutdown(&mut self) {
          println!("  [word-counter] 最終カウント: {}語", self.total_words);
      }
  }

  // --- プラグイン例: キーワードハイライター ---
  struct HighlightPlugin {
      keywords: Vec<String>,
  }

  impl HighlightPlugin {
      fn new(keywords: Vec<&str>) -> Self {
          HighlightPlugin {
              keywords: keywords.into_iter().map(|s| s.to_string()).collect(),
          }
      }
  }

  impl Plugin for HighlightPlugin {
      fn name(&self) -> &str { "highlighter" }
      fn version(&self) -> &str { "0.2.0" }

      fn on_init(&mut self, config: &HashMap<String, String>) {
          if let Some(extra) = config.get("highlight_extra") {
              self.keywords.push(extra.clone());
          }
      }

      fn on_event(&mut self, event: &str, payload: &str) -> Option<String> {
          if event == "text:input" {
              let found: Vec<&String> = self.keywords.iter()
                  .filter(|kw| payload.to_lowercase().contains(&kw.to_lowercase()))
                  .collect();
              if found.is_empty() {
                  None
              } else {
                  Some(format!("キーワード検出: {:?}", found))
              }
          } else {
              None
          }
      }

      fn on_shutdown(&mut self) {
          println!("  [highlighter] シャットダウン中");
      }
  }

  fn main() {
      println!("=== Wasmプラグインシステム デモ ===\n");

      // リソース制限付きのホストを作成
      let mut host = PluginHost::new(1024 * 1024, 1000);

      // プラグインを登録
      host.register(Box::new(WordCountPlugin::new()));
      host.register(Box::new(HighlightPlugin::new(vec!["rust", "wasm"])));

      // 設定で初期化
      let mut config = HashMap::new();
      config.insert("highlight_extra".to_string(), "plugin".to_string());
      host.init_all(&config);

      println!();

      // イベントをディスパッチ
      host.dispatch_event("text:input", "Rust and Wasm make great plugin hosts");
      println!();
      host.dispatch_event("text:input", "This has no keywords");
      println!();
      host.dispatch_event("file:save", "/path/to/file.txt");
      println!();

      // シャットダウン
      host.shutdown_all();
  }
expected_output: |
  === Wasmプラグインシステム デモ ===

  [ホスト] プラグインを登録: word-counter v1.0.0
  [ホスト] プラグインを登録: highlighter v0.2.0
  [ホスト] 2個のプラグインを初期化中...
  [ホスト]   word-counter 初期化完了
  [ホスト]   highlighter 初期化完了

  [ホスト] イベントをディスパッチ: 'text:input'
  [ホスト]   word-counter が応答: words=7 total=7
  [ホスト]   highlighter が応答: キーワード検出: ["rust", "wasm", "plugin"]

  [ホスト] イベントをディスパッチ: 'text:input'
  [ホスト]   word-counter が応答: words=4 total=11
  [ホスト]   highlighter はイベントを無視

  [ホスト] イベントをディスパッチ: 'file:save'
  [ホスト]   word-counter はイベントを無視
  [ホスト]   highlighter はイベントを無視

  [ホスト] プラグインをシャットダウン中...
    [word-counter] 最終カウント: 11語
    [highlighter] シャットダウン中
---

## なぜWasmでプラグインなのか？

WebAssemblyは、安全性、パフォーマンス、ポータビリティのユニークな組み合わせを提供するため、プラグインシステムの標準ランタイムになりつつあります：

```
  従来のプラグイン              Wasmプラグイン
  （ネイティブコード）          （サンドボックス化）

  ┌─────────────────┐       ┌─────────────────────────────┐
  │  ホストプロセス   │       │  ホストプロセス               │
  │  ┌─────────────┐│       │  ┌────────────────────────┐ │
  │  │ プラグイン    ││       │  │ Wasmランタイム          │ │
  │  │ (DLL)       ││       │  │ ┌──────────┐           │ │
  │  │ メモリ、     ││       │  │ │ プラグイン │ サンドボックス│ │
  │  │ ファイル     ││       │  │ │ .wasm    │ リニア    │ │
  │  │ システム、   ││       │  │ │          │ メモリ    │ │
  │  │ ネットワーク ││       │  │ └──────────┘           │ │
  │  │ への完全な   ││       │  │  fs、net、生メモリ     │ │
  │  │ アクセス     ││       │  │  アクセスなし           │ │
  │  └─────────────┘│       │  └────────────────────────┘ │
  │                  │       │                             │
  │ ⚠ クラッシュ =  │       │  ✓ クラッシュ = プラグインのみ │
  │   ホストも停止   │       │                             │
  └─────────────────┘       └─────────────────────────────┘
```

| 特性                  | ネイティブプラグイン (DLL/SO) | Wasmプラグイン            |
|-----------------------|------------------------|-------------------------|
| サンドボックス化       | なし                   | あり                    |
| 言語非依存            | なし（ABI依存）        | あり（任意の言語 -> Wasm）|
| ポータブル            | なし（OS別ビルド）     | あり（単一バイナリ）     |
| ネイティブに近い速度   | あり                   | あり（ネイティブの約0.8-0.95倍）|
| メモリ安全            | 言語に依存             | あり（リニアメモリ）     |
| ホットリロード可能     | 困難                   | 容易（再インスタンス化） |
| リソース制限           | 困難                   | 組み込み（fuel、メモリ） |

## 実際の使用例

| 製品          | Wasmプラグインの使い方                                     |
|-------------|-------------------------------------------------------------|
| **Figma**    | アプリ内のWasmサンドボックスでコミュニティプラグインを実行   |
| **VS Code**  | 拡張ホストがWasm拡張機能を実行可能（Web & デスクトップ）    |
| **Envoy**    | Wasmモジュールとしてプロキシフィルタ（ネットワークポリシー、認証など） |
| **Shopify**  | Functions（割引、配送）がWasmとして実行                    |
| **Zed**      | エディタ拡張機能（テーマ、言語）がWasmとして実行            |
| **Fermyon**  | Spinフレームワーク: マイクロサービス全体をWasmコンポーネントとして |

## Wasmプラグインシステムのアーキテクチャ

```
  ┌───────────────────────────────────────────────────────┐
  │                     ホストアプリケーション               │
  │                                                       │
  │  ┌─────────────┐  ┌────────────┐  ┌───────────────┐  │
  │  │ プラグイン    │  │ プラグイン  │  │ プラグイン     │  │
  │  │ レジストリ    │  │ ローダー   │  │ ライフサイクル  │  │
  │  │             │  │            │  │ マネージャ     │  │
  │  │ - マニフェスト│  │ - コンパイル│  │ - init()      │  │
  │  │ - バージョン │  │ - 検証     │  │ - on_event()  │  │
  │  │ - 依存関係  │  │ - インスタンス化│ │ - shutdown()  │  │
  │  └─────────────┘  └────────────┘  └───────────────┘  │
  │                         │                             │
  │                         ▼                             │
  │  ┌────────────────────────────────────────────────┐   │
  │  │          Wasmランタイム (wasmtime / wasmer)      │   │
  │  │                                                │   │
  │  │  ┌──────────┐  ┌──────────┐  ┌──────────┐     │   │
  │  │  │Plugin A  │  │Plugin B  │  │Plugin C  │     │   │
  │  │  │ .wasm    │  │ .wasm    │  │ .wasm    │     │   │
  │  │  │          │  │          │  │          │     │   │
  │  │  │ 1MBメモリ │  │ 2MBメモリ │  │ 1MBメモリ │     │   │
  │  │  │ 制限     │  │ 制限     │  │ 制限     │     │   │
  │  │  └──────────┘  └──────────┘  └──────────┘     │   │
  │  └────────────────────────────────────────────────┘   │
  └───────────────────────────────────────────────────────┘
```

## ホスト-ゲスト通信

ホストとプラグインは、**インポートおよびエクスポートされた関数**を通じて通信します：

```
  ホスト                              プラグイン (.wasm)
  ┌──────────────────┐             ┌──────────────────┐
  │                  │  エクスポート │                  │
  │  プラグイン関数を ─┼────────────>│ on_event()       │
  │  呼び出す        │             │ on_init()        │
  │                  │             │ alloc() / free() │
  │                  │  インポート   │                  │
  │  host_log()  <───┼────────────│ ホスト関数を      │
  │  host_fetch()<───┼────────────│ 呼び出す          │
  │  host_kv_get()<──┼────────────│                  │
  └──────────────────┘             └──────────────────┘
```

### wasmtimeの使用（Rustホスト）

```rust
use wasmtime::*;

fn load_plugin(engine: &Engine, path: &str) -> Result<Instance> {
    let module = Module::from_file(engine, path)?;
    let mut store = Store::new(engine, ());
    let mut linker = Linker::new(engine);

    // プラグインにホスト関数を提供
    linker.func_wrap("env", "host_log", |caller: Caller<'_, ()>, ptr: i32, len: i32| {
        // プラグインのリニアメモリから文字列を読み取り
        let memory = caller.get_export("memory").unwrap().into_memory().unwrap();
        let data = &memory.data(&caller)[ptr as usize..(ptr + len) as usize];
        let msg = std::str::from_utf8(data).unwrap();
        println!("[プラグイン ログ] {}", msg);
    })?;

    let instance = linker.instantiate(&mut store, &module)?;
    Ok(instance)
}
```

### プラグイン関数の呼び出し

```rust
fn call_plugin_event(
    store: &mut Store<()>,
    instance: &Instance,
    event: &str,
) -> Result<i32> {
    let memory = instance.get_memory(&mut *store, "memory").unwrap();
    let alloc = instance.get_typed_func::<i32, i32>(&mut *store, "alloc")?;

    // プラグインメモリにスペースをアロケートしイベント文字列をコピー
    let ptr = alloc.call(&mut *store, event.len() as i32)?;
    memory.data_mut(&mut *store)[ptr as usize..ptr as usize + event.len()]
        .copy_from_slice(event.as_bytes());

    // プラグインのon_event関数を呼び出し
    let on_event = instance
        .get_typed_func::<(i32, i32), i32>(&mut *store, "on_event")?;
    let result = on_event.call(&mut *store, (ptr, event.len() as i32))?;

    Ok(result)
}
```

## WIT: WebAssembly Interface Types

**WIT**（Wasm Interface Type）は、言語非依存の方法でホストとゲスト間の契約を定義します：

```wit
// plugin.wit — インターフェース定義

package my-app:plugin@1.0.0;

interface host {
    /// ホストのコンソールにメッセージをログ出力
    log: func(message: string);

    /// ホストストレージからキーバリューペアを読み取り
    kv-get: func(key: string) -> option<string>;

    /// ホストストレージにキーバリューペアを書き込み
    kv-set: func(key: string, value: string);
}

interface plugin {
    /// プラグイン読み込み時に呼ばれる
    init: func(config: list<tuple<string, string>>);

    /// 各イベントで呼ばれる
    on-event: func(name: string, payload: string) -> option<string>;

    /// プラグインのアンロード前に呼ばれる
    shutdown: func();
}

world my-plugin {
    import host;
    export plugin;
}
```

### wit-bindgenがグルーコードを生成：

```rust
// プラグインクレート内（ゲスト側）
wit_bindgen::generate!({
    world: "my-plugin",
    exports: {
        "my-app:plugin/plugin": MyPlugin,
    },
});

struct MyPlugin;

impl exports::my_app::plugin::plugin::Guest for MyPlugin {
    fn init(config: Vec<(String, String)>) {
        host::log(&format!("{}個の設定エントリで初期化", config.len()));
    }

    fn on_event(name: String, payload: String) -> Option<String> {
        host::log(&format!("イベント受信: {}", name));
        Some(format!("processed: {}", payload))
    }

    fn shutdown() {
        host::log("さようなら！");
    }
}
```

## リソース制限とサンドボックス化

Wasmランタイムはプラグインリソースに対するきめ細かな制御を提供します：

### メモリ制限

```rust
let mut config = Config::new();
let engine = Engine::new(&config)?;

let memory_type = MemoryType::new(1, Some(16)); // 最小1ページ、最大16ページ (1 MB)
let memory = Memory::new(&mut store, memory_type)?;
```

### Fuel（CPU制限）

```rust
let mut config = Config::new();
config.consume_fuel(true);
let engine = Engine::new(&config)?;

let mut store = Store::new(&engine, ());
store.set_fuel(1_000_000)?;  // 100万命令

// プラグインはfuelが尽きるまで実行
let result = on_event.call(&mut store, (ptr, len));
match result {
    Err(e) if e.to_string().contains("fuel") => {
        println!("プラグインがCPU予算を超過しました！");
    }
    _ => {}
}
```

### 機能テーブル

```
  ┌──────────────────┬─────────┬─────────┬─────────┐
  │ 機能             │ デフォルト│ 許可    │ 拒否    │
  ├──────────────────┼─────────┼─────────┼─────────┤
  │ リニアメモリ      │ 1ページ │ ≤ 16 pg │ > 16 pg │
  │ CPU (fuel)       │ 100万   │ カスタム │ 0       │
  │ ホスト関数       │ なし    │ リンク済 │ 省略    │
  │ ファイルシステム  │ なし    │ WASI    │ デフォルト│
  │ ネットワーク      │ なし    │ WASI    │ デフォルト│
  │ システムクロック  │ なし    │ WASI    │ デフォルト│
  └──────────────────┴─────────┴─────────┴─────────┘
```

## プラグインの発見と読み込み

実用的なシステムには、プラグインを発見、検証、読み込む方法が必要です：

```rust
struct PluginManifest {
    name: String,
    version: String,
    wasm_path: String,
    permissions: Vec<String>,
    min_host_version: String,
}

struct PluginRegistry {
    manifests: HashMap<String, PluginManifest>,
    loaded: HashMap<String, Instance>,
}

impl PluginRegistry {
    fn discover(&mut self, plugin_dir: &str) -> Vec<String> {
        // ディレクトリをスキャンしてplugin.tomlファイルを探す
        // マニフェストをパースし、署名を検証
        // 発見されたプラグイン名のリストを返す
        vec![]
    }

    fn load(&mut self, name: &str, engine: &Engine) -> Result<(), String> {
        let manifest = self.manifests.get(name).ok_or("見つかりません")?;

        // パーミッションの検証
        for perm in &manifest.permissions {
            if !self.is_permission_allowed(perm) {
                return Err(format!("パーミッション拒否: {}", perm));
            }
        }

        // コンパイルとインスタンス化
        // ...
        Ok(())
    }

    fn is_permission_allowed(&self, _perm: &str) -> bool { true }
}
```

### ホットリロード

Wasmの強みの1つは簡単なホットリロードです — 古いインスタンスを破棄し、新しいものを読み込みます：

```rust
fn hot_reload(registry: &mut PluginRegistry, name: &str, engine: &Engine) {
    // 1. 古いインスタンスでshutdownを呼ぶ
    // 2. 古いInstanceを破棄（すべてのメモリを回収）
    registry.loaded.remove(name);

    // 3. 新しい.wasmファイルを読み込み
    registry.load(name, engine).unwrap();

    // 4. 新しいインスタンスでinitを呼ぶ
    // プラグイン状態は新鮮 — 古いポインタやメモリリークなし
}
```

## セキュリティに関する考慮事項

```
  攻撃面の比較:

  ネイティブプラグイン           Wasmプラグイン
  ───────────────            ────────────────
  ✗ バッファオーバーフロー      ✓ リニアメモリの境界チェック
  ✗ 解放後使用                 ✓ 生ポインタが露出しない
  ✗ 任意のシステムコール        ✓ インポートされた関数のみ
  ✗ ファイルシステムアクセス     ✓ 明示的なWASI機能
  ✗ ネットワークアクセス        ✓ 許可が必要
  ✗ 共有メモリの破損            ✓ 分離されたアドレス空間
```

| 脅威                      | 緩和策                                        |
|---------------------------|---------------------------------------------|
| 無限ループ                | fuel / 命令数制限                             |
| メモリ枯渇                | メモリページ制限                              |
| 悪意あるホスト呼び出し     | プラグインからのすべての引数を検証             |
| データ漏洩                | 明示的に許可されない限りネットワークなし       |
| サイドチャネルタイミング    | クロックアクセスの無効化またはジッターの追加    |

## まとめ

Wasmプラグインシステムは、ネイティブコードのパフォーマンスとサンドボックス実行の安全性を兼ね備えています。ホストはトレイトのようなインターフェース（またはWIT契約）を定義し、実行時に`.wasm`モジュールを読み込み、リソース制限を適用します。Figma、Envoy、VS Codeなどの実際の製品が、このモデルが大規模に機能することを証明しています。ランタイムには`wasmtime`または`wasmer`を、人間工学的なインターフェースには`wit-bindgen`を、安全性にはfuel/メモリ制限を使用しましょう。
