---
title: "Wasm + IndexedDB"
slug: wasm-indexeddb
difficulty: intermediate
tags: [api]
order: 32
description: web-sysを通じてRustからIndexedDBにアクセスする — データベースの作成、CRUD操作、トランザクション管理、インデックスの構築、非同期データベース呼び出しのためのJsFutureの使用。
starter_code: |
  use std::collections::HashMap;
  use std::fmt;

  fn main() {
      println!("=== Wasm + IndexedDBシミュレーション ===\n");

      // --- IndexedDBスタイルのキーバリューストアのシミュレーション ---

      #[derive(Debug, Clone)]
      struct Record {
          id: u32,
          name: String,
          email: String,
          age: u32,
      }

      impl fmt::Display for Record {
          fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
              write!(f, "{{ id: {}, name: {:?}, email: {:?}, age: {} }}",
                  self.id, self.name, self.email, self.age)
          }
      }

      struct ObjectStore {
          name: String,
          records: HashMap<u32, Record>,
          auto_increment: u32,
          indexes: HashMap<String, HashMap<String, Vec<u32>>>,
      }

      impl ObjectStore {
          fn new(name: &str) -> Self {
              println!("  オブジェクトストア作成: {:?}", name);
              ObjectStore {
                  name: name.to_string(),
                  records: HashMap::new(),
                  auto_increment: 1,
                  indexes: HashMap::new(),
              }
          }

          fn create_index(&mut self, index_name: &str) {
              self.indexes.insert(index_name.to_string(), HashMap::new());
              println!("  インデックス作成: {:?}（ストア {:?} 上）", index_name, self.name);
          }

          fn put(&mut self, mut record: Record) -> u32 {
              if record.id == 0 {
                  record.id = self.auto_increment;
                  self.auto_increment += 1;
              }
              let id = record.id;

              // インデックスの更新
              if self.indexes.contains_key("name") {
                  self.indexes.get_mut("name").unwrap()
                      .entry(record.name.clone())
                      .or_insert_with(Vec::new)
                      .push(id);
              }
              if self.indexes.contains_key("email") {
                  self.indexes.get_mut("email").unwrap()
                      .entry(record.email.clone())
                      .or_insert_with(Vec::new)
                      .push(id);
              }

              self.records.insert(id, record);
              id
          }

          fn get(&self, id: u32) -> Option<&Record> {
              self.records.get(&id)
          }

          fn delete(&mut self, id: u32) -> bool {
              self.records.remove(&id).is_some()
          }

          fn get_by_index(&self, index_name: &str, value: &str) -> Vec<&Record> {
              if let Some(index) = self.indexes.get(index_name) {
                  if let Some(ids) = index.get(value) {
                      return ids.iter()
                          .filter_map(|id| self.records.get(id))
                          .collect();
                  }
              }
              Vec::new()
          }

          fn count(&self) -> usize {
              self.records.len()
          }

          fn get_all(&self) -> Vec<&Record> {
              let mut records: Vec<&Record> = self.records.values().collect();
              records.sort_by_key(|r| r.id);
              records
          }
      }

      struct Database {
          name: String,
          version: u32,
          stores: HashMap<String, ObjectStore>,
      }

      impl Database {
          fn open(name: &str, version: u32) -> Self {
              println!("--- データベースを開く ---");
              println!("  データベース: {:?}, バージョン: {}", name, version);
              Database {
                  name: name.to_string(),
                  version,
                  stores: HashMap::new(),
              }
          }

          fn create_object_store(&mut self, name: &str) -> &mut ObjectStore {
              self.stores.insert(name.to_string(), ObjectStore::new(name));
              self.stores.get_mut(name).unwrap()
          }

          fn transaction(&mut self, store_name: &str) -> Option<&mut ObjectStore> {
              self.stores.get_mut(store_name)
          }
      }

      // --- データベースを開いてスキーマを作成 ---
      let mut db = Database::open("my_app_db", 1);

      println!("\n--- スキーマ設定（onupgradeneeded） ---");
      let store = db.create_object_store("users");
      store.create_index("name");
      store.create_index("email");

      // --- CREATE: レコードの追加 ---
      println!("\n--- CREATE操作 ---");
      let store = db.transaction("users").unwrap();

      let users = vec![
          Record { id: 0, name: "Alice".into(), email: "alice@example.com".into(), age: 30 },
          Record { id: 0, name: "Bob".into(), email: "bob@example.com".into(), age: 25 },
          Record { id: 0, name: "Charlie".into(), email: "charlie@example.com".into(), age: 35 },
          Record { id: 0, name: "Alice".into(), email: "alice2@example.com".into(), age: 28 },
      ];

      for user in users {
          let id = store.put(user.clone());
          println!("  PUT -> id={}: {} <{}>", id, user.name, user.email);
      }
      println!("  ストアのレコード数: {}", store.count());

      // --- READ: キーで取得 ---
      println!("\n--- READ操作 ---");
      if let Some(record) = store.get(1) {
          println!("  GET(1): {}", record);
      }
      if let Some(record) = store.get(3) {
          println!("  GET(3): {}", record);
      }
      match store.get(99) {
          Some(r) => println!("  GET(99): {}", r),
          None => println!("  GET(99): 見つかりません"),
      }

      // --- READ: インデックスで取得 ---
      println!("\n--- INDEXクエリ ---");
      let alices = store.get_by_index("name", "Alice");
      println!("  インデックス 'name'='Alice': {}件", alices.len());
      for r in &alices {
          println!("    {}", r);
      }

      let bob_emails = store.get_by_index("email", "bob@example.com");
      println!("  インデックス 'email'='bob@example.com': {}件", bob_emails.len());
      for r in &bob_emails {
          println!("    {}", r);
      }

      // --- UPDATE: レコードの更新 ---
      println!("\n--- UPDATE操作 ---");
      let updated = Record { id: 2, name: "Bob".into(), email: "bob.new@example.com".into(), age: 26 };
      store.put(updated);
      if let Some(record) = store.get(2) {
          println!("  レコード2を更新: {}", record);
      }

      // --- DELETE: レコードの削除 ---
      println!("\n--- DELETE操作 ---");
      let deleted = store.delete(3);
      println!("  DELETE(3): 成功={}", deleted);
      println!("  削除後のストアレコード数: {}", store.count());

      let deleted_again = store.delete(99);
      println!("  DELETE(99): 成功={}（見つかりません）", deleted_again);

      // --- 全レコードの一覧 ---
      println!("\n--- 残りの全レコード ---");
      for record in store.get_all() {
          println!("  {}", record);
      }

      // --- トランザクションのまとめ ---
      println!("\n--- トランザクションまとめ ---");
      println!("  データベース: {:?} v{}", db.name, db.version);
      println!("  オブジェクトストア: {:?}", db.stores.keys().collect::<Vec<_>>());
      println!("  合計レコード数: {}", db.transaction("users").unwrap().count());
  }
expected_output: |
  === Wasm + IndexedDBシミュレーション ===

  --- データベースを開く ---
    データベース: "my_app_db", バージョン: 1

  --- スキーマ設定（onupgradeneeded） ---
    オブジェクトストア作成: "users"
    インデックス作成: "name"（ストア "users" 上）
    インデックス作成: "email"（ストア "users" 上）

  --- CREATE操作 ---
    PUT -> id=1: Alice <alice@example.com>
    PUT -> id=2: Bob <bob@example.com>
    PUT -> id=3: Charlie <charlie@example.com>
    PUT -> id=4: Alice <alice2@example.com>
    ストアのレコード数: 4

  --- READ操作 ---
    GET(1): { id: 1, name: "Alice", email: "alice@example.com", age: 30 }
    GET(3): { id: 3, name: "Charlie", email: "charlie@example.com", age: 35 }
    GET(99): 見つかりません

  --- INDEXクエリ ---
    インデックス 'name'='Alice': 2件
      { id: 1, name: "Alice", email: "alice@example.com", age: 30 }
      { id: 4, name: "Alice", email: "alice2@example.com", age: 28 }
    インデックス 'email'='bob@example.com': 1件
      { id: 2, name: "Bob", email: "bob@example.com", age: 25 }

  --- UPDATE操作 ---
    レコード2を更新: { id: 2, name: "Bob", email: "bob.new@example.com", age: 26 }

  --- DELETE操作 ---
    DELETE(3): 成功=true
    削除後のストアレコード数: 3
    DELETE(99): 成功=false（見つかりません）

  --- 残りの全レコード ---
    { id: 1, name: "Alice", email: "alice@example.com", age: 30 }
    { id: 2, name: "Bob", email: "bob.new@example.com", age: 26 }
    { id: 4, name: "Alice", email: "alice2@example.com", age: 28 }

  --- トランザクションまとめ ---
    データベース: "my_app_db" v1
    オブジェクトストア: ["users"]
    合計レコード数: 3
---

## IndexedDBとは？

IndexedDBは、すべてのブラウザに組み込まれた低レベルのトランザクショナルデータベースです。localStorage（文字列のみ最大約5MBまで）とは異なり、IndexedDBは構造化データ、バイナリブロブ、ファイルをほぼ無制限のサイズで保存できます。

```
┌─────────────────────────────────────────────────────┐
│                    ブラウザ                           │
│                                                      │
│  ┌─────────────┐   ┌────────────────────────────┐   │
│  │  Wasmモジュール│──►│       IndexedDB             │   │
│  │  （Rust）     │   │                            │   │
│  │              │   │  データベース: "my_app"       │   │
│  │  web-sys     │   │  ├── ストア: "users"        │   │
│  │  バインディング│   │  │   ├── インデックス: "email"│   │
│  │              │   │  │   └── インデックス: "name" │   │
│  │  JsFuture    │   │  └── ストア: "settings"     │   │
│  └─────────────┘   └────────────────────────────┘   │
└─────────────────────────────────────────────────────┘
```

| 機能 | localStorage | IndexedDB |
|---------|-------------|-----------|
| データ型 | 文字列のみ | 構造化データ、バイナリ、ブロブ |
| サイズ制限 | 約5〜10 MB | 数百MB以上 |
| 非同期 | いいえ（ブロッキング） | はい |
| インデックス | なし | あり |
| トランザクション | なし | あり（ACID準拠） |
| カーソル | なし | あり |

## IndexedDB用のweb-sys設定

```toml
[dependencies]
wasm-bindgen = "0.2"
wasm-bindgen-futures = "0.4"
js-sys = "0.3"

[dependencies.web-sys]
version = "0.3"
features = [
    "Window",
    "IdbFactory",
    "IdbOpenDbRequest",
    "IdbDatabase",
    "IdbTransaction",
    "IdbTransactionMode",
    "IdbObjectStore",
    "IdbObjectStoreParameters",
    "IdbRequest",
    "IdbKeyRange",
    "IdbIndex",
    "IdbIndexParameters",
    "IdbCursorWithValue",
    "IdbCursorDirection",
    "DomException",
    "Event",
    "IdbVersionChangeEvent",
]
```

## データベースを開く

IndexedDBの操作はJavaScriptではコールバックベースです。Rustでは`JsFuture`や手動の`Promise`処理でラップします：

```rust
use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use web_sys::{IdbDatabase, IdbOpenDbRequest, IdbObjectStoreParameters};
use wasm_bindgen_futures::JsFuture;

pub async fn open_db(name: &str, version: u32) -> Result<IdbDatabase, JsValue> {
    let window = web_sys::window().unwrap();
    let idb_factory = window.indexed_db()?.unwrap();

    let open_request: IdbOpenDbRequest = idb_factory.open_with_u32(name, version)?;

    // スキーマアップグレードの処理
    let on_upgrade = Closure::wrap(Box::new(move |event: web_sys::IdbVersionChangeEvent| {
        let request: IdbOpenDbRequest = event.target().unwrap().dyn_into().unwrap();
        let db: IdbDatabase = request.result().unwrap().dyn_into().unwrap();

        // アップグレード中にオブジェクトストアを作成
        if !db.object_store_names().contains("users") {
            let mut params = IdbObjectStoreParameters::new();
            params.key_path(Some(&JsValue::from_str("id")));
            params.auto_increment(true);

            let store = db.create_object_store_with_optional_parameters("users", &params)?;
            store.create_index_with_str("email", "email")?;
            store.create_index_with_str("name", "name")?;
        }
    }) as Box<dyn FnMut(_)>);

    open_request.set_onupgradeneeded(Some(on_upgrade.as_ref().unchecked_ref()));
    on_upgrade.forget();

    // データベースが開くのを待つ
    let result = JsFuture::from(open_request.into()).await?;
    Ok(result.dyn_into()?)
}
```

## CRUD操作

### Create（put）

```rust
use serde::Serialize;

#[derive(Serialize)]
struct User {
    name: String,
    email: String,
    age: u32,
}

pub async fn add_user(db: &IdbDatabase, user: &User) -> Result<JsValue, JsValue> {
    let transaction = db.transaction_with_str_and_mode(
        "users",
        web_sys::IdbTransactionMode::Readwrite,
    )?;

    let store = transaction.object_store("users")?;
    let js_value = serde_wasm_bindgen::to_value(user)?;
    let request = store.put(&js_value)?;

    JsFuture::from(request).await
}
```

### Read（get）

```rust
pub async fn get_user(db: &IdbDatabase, id: u32) -> Result<Option<JsValue>, JsValue> {
    let transaction = db.transaction_with_str("users")?;
    let store = transaction.object_store("users")?;
    let request = store.get(&JsValue::from(id))?;

    let result = JsFuture::from(request).await?;
    if result.is_undefined() {
        Ok(None)
    } else {
        Ok(Some(result))
    }
}
```

### Update

```rust
// 既存のキーでput()するとレコードが更新される
pub async fn update_user(db: &IdbDatabase, user: &User) -> Result<JsValue, JsValue> {
    // add_userと同じ — put()はupsert操作
    add_user(db, user).await
}
```

### Delete

```rust
pub async fn delete_user(db: &IdbDatabase, id: u32) -> Result<(), JsValue> {
    let transaction = db.transaction_with_str_and_mode(
        "users",
        web_sys::IdbTransactionMode::Readwrite,
    )?;

    let store = transaction.object_store("users")?;
    let request = store.delete(&JsValue::from(id))?;
    JsFuture::from(request).await?;
    Ok(())
}
```

## トランザクション

IndexedDBのトランザクションは**ACID準拠**です：

```
┌─────────────────────────────────────────┐
│            トランザクション                │
│                                          │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐ │
│  │  put()   │  │  get()   │  │ delete() │ │
│  │ ユーザ#1 │  │ ユーザ#2 │  │ ユーザ#3 │ │
│  └─────────┘  └─────────┘  └─────────┘ │
│                                          │
│  全て成功 ──► コミット                    │
│  一つでも失敗 ──► ロールバック（全て元に戻る）│
└─────────────────────────────────────────┘
```

トランザクションモード：

| モード | 読み取り可？ | 書き込み可？ | 他をブロック？ |
|------|----------|-----------|----------------|
| `readonly` | はい | いいえ | いいえ |
| `readwrite` | はい | はい | 同じストアへの書き込みをブロック |
| `versionchange` | はい | はい（スキーマ） | 排他的 |

```rust
// 1つのトランザクション内で複数操作
pub async fn transfer_credits(
    db: &IdbDatabase,
    from_id: u32,
    to_id: u32,
    amount: u32,
) -> Result<(), JsValue> {
    let tx = db.transaction_with_str_and_mode(
        "users", web_sys::IdbTransactionMode::Readwrite,
    )?;
    let store = tx.object_store("users")?;

    // 両方の操作がアトミックに行われる
    let from = JsFuture::from(store.get(&from_id.into())?).await?;
    let to = JsFuture::from(store.get(&to_id.into())?).await?;

    // 変更してputで戻す
    js_sys::Reflect::set(&from, &"credits".into(),
        &(get_credits(&from) - amount).into())?;
    js_sys::Reflect::set(&to, &"credits".into(),
        &(get_credits(&to) + amount).into())?;

    store.put(&from)?;
    store.put(&to)?;

    // トランザクションは全リクエスト完了時に自動コミット
    Ok(())
}
```

## インデックスを使ったクエリ

インデックスを使うと、主キー以外のフィールドでレコードを検索できます：

```rust
pub async fn find_by_email(
    db: &IdbDatabase,
    email: &str,
) -> Result<Option<JsValue>, JsValue> {
    let tx = db.transaction_with_str("users")?;
    let store = tx.object_store("users")?;
    let index = store.index("email")?;

    let request = index.get(&JsValue::from_str(email))?;
    let result = JsFuture::from(request).await?;

    if result.is_undefined() { Ok(None) } else { Ok(Some(result)) }
}
```

### IdbKeyRangeによる範囲クエリ

```rust
pub async fn find_users_in_age_range(
    db: &IdbDatabase,
    min_age: u32,
    max_age: u32,
) -> Result<Vec<JsValue>, JsValue> {
    let tx = db.transaction_with_str("users")?;
    let store = tx.object_store("users")?;
    let index = store.index("age")?;

    let range = web_sys::IdbKeyRange::bound(
        &JsValue::from(min_age),
        &JsValue::from(max_age),
    )?;

    let request = index.get_all_with_key(&range)?;
    let result = JsFuture::from(request).await?;
    let array: js_sys::Array = result.dyn_into()?;

    Ok(array.iter().collect())
}
```

## JsFuture：コールバックから非同期への橋渡し

IndexedDBはPromise以前に設計されたため、イベントコールバックを使用します。`JsFuture`は`IdbRequest`をRustの`Future`に変換します：

```
┌────────────┐     ┌──────────────┐     ┌────────────┐
│ IdbRequest │────►│   Promise    │────►│  JsFuture   │
│（コールバック）│     │  （ラッパー） │     │  （.await）  │
│            │     │              │     │             │
│ onsuccess  │     │  resolve()   │     │  Ok(value)  │
│ onerror    │     │  reject()    │     │  Err(error) │
└────────────┘     └──────────────┘     └────────────┘
```

```rust
use wasm_bindgen_futures::JsFuture;

// IdbRequest -> JsFuture（暗黙のPromise変換経由）
let request: IdbRequest = store.get(&key)?;
let result: JsValue = JsFuture::from(request).await?;
```

## エラーハンドリング

```rust
pub async fn safe_db_operation(db: &IdbDatabase) -> Result<String, String> {
    let tx = db.transaction_with_str_and_mode(
        "users",
        web_sys::IdbTransactionMode::Readwrite,
    ).map_err(|e| format!("トランザクション失敗: {:?}", e))?;

    let store = tx.object_store("users")
        .map_err(|e| format!("ストアが見つかりません: {:?}", e))?;

    let request = store.get(&JsValue::from(1))
        .map_err(|e| format!("取得失敗: {:?}", e))?;

    let result = JsFuture::from(request).await
        .map_err(|e| format!("非同期エラー: {:?}", e))?;

    if result.is_undefined() {
        Err("レコードが見つかりません".to_string())
    } else {
        Ok(format!("{:?}", result))
    }
}
```

## まとめ

1. **IndexedDB**は強力なブラウザデータベース — トランザクショナルでインデックス付き、バイナリデータと大規模データセットを扱える
2. **web-sys**は完全なIndexedDBバインディングを提供するが、APIは冗長でコールバックベース
3. **JsFuture**はコールバックベースのAPIとRustのasync/awaitを橋渡しする
4. **トランザクション**はACID準拠 — 関連する操作をグループ化してアトミック性を確保する
5. **インデックス**は主キー以外のフィールドで効率的な検索を可能にする
6. **スキーマアップグレード**は`onupgradeneeded`イベント内で行う — ストアやインデックスの作成/削除はここでのみ可能
7. エラーハンドリングは慎重に — IndexedDB操作はクォータ制限、パーミッション、スキーマの不一致により失敗する可能性がある
