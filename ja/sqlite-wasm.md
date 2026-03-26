---
title: Wasm経由でブラウザ内SQLite
slug: sqlite-wasm
difficulty: advanced
tags: [api]
order: 48
description: Rust/WasmでシンプルなインメモリSQLエンジンを構築 — SQLパース、クエリ実行、インデックス作成、そしてブラウザで完全なデータベースが動作する理由を学びます。
starter_code: |
  use std::collections::HashMap;

  /// 行はカラム名 -> 値のマッピング
  type Row = HashMap<String, Value>;

  #[derive(Debug, Clone, PartialEq)]
  enum Value {
      Integer(i64),
      Text(String),
      Null,
  }

  impl std::fmt::Display for Value {
      fn fmt(&self, f: &mut std::fmt::Formatter) -> std::fmt::Result {
          match self {
              Value::Integer(n) => write!(f, "{}", n),
              Value::Text(s) => write!(f, "{}", s),
              Value::Null => write!(f, "NULL"),
          }
      }
  }

  /// シンプルなインメモリテーブル
  struct Table {
      name: String,
      columns: Vec<String>,
      rows: Vec<Row>,
  }

  impl Table {
      fn new(name: &str, columns: Vec<String>) -> Self {
          Table {
              name: name.to_string(),
              columns,
              rows: Vec::new(),
          }
      }

      fn insert(&mut self, values: Vec<(&str, Value)>) {
          let mut row = HashMap::new();
          for (col, val) in values {
              if self.columns.contains(&col.to_string()) {
                  row.insert(col.to_string(), val);
              }
          }
          // 欠落カラムをNULLで埋める
          for col in &self.columns {
              if !row.contains_key(col) {
                  row.insert(col.clone(), Value::Null);
              }
          }
          self.rows.push(row);
      }

      fn select(&self, columns: &[&str], where_clause: Option<WhereClause>) -> Vec<Vec<Value>> {
          let mut results = Vec::new();

          for row in &self.rows {
              // WHEREフィルターを適用
              if let Some(ref clause) = where_clause {
                  if !clause.matches(row) {
                      continue;
                  }
              }

              // 選択されたカラムを射影
              let mut result_row = Vec::new();
              let cols: Vec<&str> = if columns.is_empty() || columns[0] == "*" {
                  self.columns.iter().map(|s| s.as_str()).collect()
              } else {
                  columns.to_vec()
              };

              for col in &cols {
                  let val = row.get(*col).cloned().unwrap_or(Value::Null);
                  result_row.push(val);
              }
              results.push(result_row);
          }

          results
      }

      fn update(&mut self, set_col: &str, set_val: Value, where_clause: Option<WhereClause>) -> usize {
          let mut count = 0;
          for row in &mut self.rows {
              let matches = match &where_clause {
                  Some(clause) => clause.matches(row),
                  None => true,
              };
              if matches {
                  row.insert(set_col.to_string(), set_val.clone());
                  count += 1;
              }
          }
          count
      }

      fn delete(&mut self, where_clause: Option<WhereClause>) -> usize {
          let before = self.rows.len();
          self.rows.retain(|row| {
              match &where_clause {
                  Some(clause) => !clause.matches(row),
                  None => false, // WHEREなしのDELETEはすべて削除
              }
          });
          before - self.rows.len()
      }
  }

  /// シンプルなWHERE句: column = value
  #[derive(Debug, Clone)]
  struct WhereClause {
      column: String,
      op: CompareOp,
      value: Value,
  }

  #[derive(Debug, Clone)]
  enum CompareOp {
      Equal,
      NotEqual,
      GreaterThan,
      LessThan,
  }

  impl WhereClause {
      fn new(column: &str, op: CompareOp, value: Value) -> Self {
          WhereClause {
              column: column.to_string(),
              op,
              value,
          }
      }

      fn matches(&self, row: &Row) -> bool {
          let row_val = match row.get(&self.column) {
              Some(v) => v,
              None => return false,
          };

          match (&self.op, row_val, &self.value) {
              (CompareOp::Equal, a, b) => a == b,
              (CompareOp::NotEqual, a, b) => a != b,
              (CompareOp::GreaterThan, Value::Integer(a), Value::Integer(b)) => a > b,
              (CompareOp::LessThan, Value::Integer(a), Value::Integer(b)) => a < b,
              _ => false,
          }
      }
  }

  /// 複数のテーブルを保持するシンプルなデータベース
  struct Database {
      tables: HashMap<String, Table>,
  }

  impl Database {
      fn new() -> Self {
          Database { tables: HashMap::new() }
      }

      fn create_table(&mut self, name: &str, columns: Vec<String>) {
          let table = Table::new(name, columns);
          self.tables.insert(name.to_string(), table);
      }

      fn get_table(&self, name: &str) -> Option<&Table> {
          self.tables.get(name)
      }

      fn get_table_mut(&mut self, name: &str) -> Option<&mut Table> {
          self.tables.get_mut(name)
      }
  }

  fn print_results(columns: &[&str], rows: &[Vec<Value>]) {
      if rows.is_empty() {
          println!("  (行なし)");
          return;
      }

      // ヘッダーを表示
      println!("  {}", columns.join(" | "));
      println!("  {}", columns.iter()
          .map(|c| "-".repeat(c.len().max(6)))
          .collect::<Vec<_>>()
          .join("-+-"));

      // 行を表示
      for row in rows {
          let formatted: Vec<String> = row.iter()
              .map(|v| format!("{}", v))
              .collect();
          println!("  {}", formatted.join(" | "));
      }
      println!("  ({}行)", rows.len());
  }

  fn main() {
      let mut db = Database::new();

      // CREATE TABLE
      println!("=== CREATE TABLE users ===");
      db.create_table("users", vec![
          "id".into(), "name".into(), "email".into(), "age".into()
      ]);
      println!("テーブル'users'をカラム: id, name, email, age で作成しました");
      println!();

      // INSERT
      println!("=== INSERT INTO users ===");
      let users = db.get_table_mut("users").unwrap();
      users.insert(vec![
          ("id", Value::Integer(1)),
          ("name", Value::Text("Alice".into())),
          ("email", Value::Text("alice@example.com".into())),
          ("age", Value::Integer(30)),
      ]);
      users.insert(vec![
          ("id", Value::Integer(2)),
          ("name", Value::Text("Bob".into())),
          ("email", Value::Text("bob@example.com".into())),
          ("age", Value::Integer(25)),
      ]);
      users.insert(vec![
          ("id", Value::Integer(3)),
          ("name", Value::Text("Charlie".into())),
          ("email", Value::Text("charlie@example.com".into())),
          ("age", Value::Integer(35)),
      ]);
      users.insert(vec![
          ("id", Value::Integer(4)),
          ("name", Value::Text("Diana".into())),
          ("email", Value::Text("diana@example.com".into())),
          ("age", Value::Integer(28)),
      ]);
      println!("4行を挿入しました");
      println!();

      // SELECT *
      println!("=== SELECT * FROM users ===");
      let users = db.get_table("users").unwrap();
      let all = users.select(&["*"], None);
      print_results(&["id", "name", "email", "age"], &all);
      println!();

      // WHEREを使ったSELECT
      println!("=== SELECT name, age FROM users WHERE age > 27 ===");
      let filtered = users.select(
          &["name", "age"],
          Some(WhereClause::new("age", CompareOp::GreaterThan, Value::Integer(27)))
      );
      print_results(&["name", "age"], &filtered);
      println!();

      // UPDATE
      println!("=== UPDATE users SET age = 31 WHERE name = 'Alice' ===");
      let users = db.get_table_mut("users").unwrap();
      let updated = users.update(
          "age",
          Value::Integer(31),
          Some(WhereClause::new("name", CompareOp::Equal, Value::Text("Alice".into())))
      );
      println!("{}行を更新しました", updated);
      println!();

      // 更新を確認
      println!("=== SELECT * FROM users WHERE name = 'Alice' ===");
      let users = db.get_table("users").unwrap();
      let alice = users.select(
          &["*"],
          Some(WhereClause::new("name", CompareOp::Equal, Value::Text("Alice".into())))
      );
      print_results(&["id", "name", "email", "age"], &alice);
      println!();

      // DELETE
      println!("=== DELETE FROM users WHERE name = 'Bob' ===");
      let users = db.get_table_mut("users").unwrap();
      let deleted = users.delete(
          Some(WhereClause::new("name", CompareOp::Equal, Value::Text("Bob".into())))
      );
      println!("{}行を削除しました", deleted);
      println!();

      // 最終状態
      println!("=== SELECT * FROM users（最終状態） ===");
      let users = db.get_table("users").unwrap();
      let final_state = users.select(&["*"], None);
      print_results(&["id", "name", "email", "age"], &final_state);
  }
expected_output: |
  === CREATE TABLE users ===
  テーブル'users'をカラム: id, name, email, age で作成しました

  === INSERT INTO users ===
  4行を挿入しました

  === SELECT * FROM users ===
    id | name | email | age
    ...
    (4行)

  === SELECT name, age FROM users WHERE age > 27 ===
    name | age
    ...
    (3行)

  === UPDATE users SET age = 31 WHERE name = 'Alice' ===
  1行を更新しました

  === DELETE FROM users WHERE name = 'Bob' ===
  1行を削除しました

  === SELECT * FROM users（最終状態） ===
    (3行)
---

## はじめに

ブラウザ内で完全なSQLデータベースを実行するのは不可能に思えますが、WebAssemblyにコンパイルされたSQLiteがそれを現実にします。このレッスンでは核となる概念を理解するために純粋なRustで簡略化したインメモリSQLエンジンを構築し、実際のSQLiteがWasmでどのように動作するかを解説します。

## なぜブラウザ内SQLiteなのか？

従来のウェブアプリはデータクエリをすべてサーバーに送信します：

```
従来の方式:
┌──────────┐  HTTPリクエスト ┌──────────┐  SQLクエリ  ┌──────────┐
│  ブラウザ  │──────────────>│  サーバー  │────────────>│データベース│
│           │<──────────────│           │<────────────│          │
│           │  HTTPレスポンス│           │  結果セット │          │
└──────────┘               └──────────┘              └──────────┘
   レイテンシ: クエリあたり50-200ms

WasmでSQLiteを使う場合:
┌──────────────────────────────┐
│  ブラウザ                     │
│  ┌──────────┐  ┌───────────┐│
│  │ アプリ(JS)│──│SQLite Wasm││
│  │          │  │(インメモリ) ││
│  └──────────┘  └───────────┘│
│   レイテンシ: クエリあたり<1ms│
└──────────────────────────────┘
```

ユースケース：
- **オフラインファーストアプリ** -- ネットワークなしで完全なSQL機能
- **ローカルデータ処理** -- クライアント側で大きなデータセットをフィルタ/ソート/集計
- **プライバシー重視のアプリ** -- データがデバイスの外に出ない
- **プロトタイピング** -- 開発中にバックエンドが不要

## SQLエンジンの仕組み

簡略化したエンジンは4つのコア操作を実装しています：

```
SQL操作          │  メソッド        │  説明
─────────────────┼─────────────────┼──────────────────────────
CREATE TABLE     │  create_table() │  テーブルのカラムを定義
INSERT INTO      │  insert()       │  値を持つ行を追加
SELECT ... WHERE │  select()       │  行の読み取り、フィルタ、射影
UPDATE ... WHERE │  update()       │  条件に合う行を変更
DELETE ... WHERE │  delete()       │  条件に合う行を削除
```

### データモデル

```
Database
└── tables: HashMap<String, Table>
    └── Table
        ├── name: String
        ├── columns: Vec<String>
        └── rows: Vec<Row>
            └── Row = HashMap<String, Value>
                └── Value: Integer(i64) | Text(String) | Null
```

各行は`HashMap<String, Value>`であり、カラムアクセスはO(1)ですが、カラム型レイアウトよりメモリ使用量が多くなります。プロダクション向けデータベースはより効率的な表現を使用します。

## クエリ実行パイプライン

実際のSQLデータベースは複数のステージでクエリを処理します：

```
SQL文字列
    │
    ▼
┌──────────┐
│  字句解析 │  トークンに分割: SELECT, *, FROM, users, WHERE, ...
└────┬─────┘
     ▼
┌──────────┐
│  構文解析 │  抽象構文木（AST）を構築
└────┬─────┘
     ▼
┌──────────┐
│ プランナー│  実行戦略を選択（どのインデックスを使うか？）
└────┬─────┘
     ▼
┌──────────┐
│ エグゼキュータ│  テーブルスキャン、フィルタ適用、カラム射影
└────┬─────┘
     ▼
結果セット
```

簡略化したエンジンでは字句解析/構文解析をスキップしてメソッドを直接呼び出しますが、実行ロジック（スキャン、フィルタ、射影）は同じです。

## インデックス作成：クエリを高速化する

インデックスなしでは、すべてのクエリがすべての行をスキャンする必要があります（フルテーブルスキャン）。インデックスは高速な検索を可能にするソート済みのデータ構造です：

```
フルテーブルスキャン（インデックスなし）:
  SELECT * FROM users WHERE age = 30
  → 行1をチェック: age=30? はい ✓
  → 行2をチェック: age=25? いいえ
  → 行3をチェック: age=35? いいえ
  → 行4をチェック: age=28? いいえ
  時間: O(n) — すべての行をチェックする必要がある

ageにB-treeインデックスがある場合:
         [28]
        /    \
     [25]   [30, 35]

  → ツリーを探索: 30 > 28 → 右へ → 30を発見
  時間: O(log n) — ほとんどの行をスキップ
```

| 行数 | フルスキャン | B-treeインデックス |
|------|-----------|--------------|
| 100 | 100回チェック | 約7回チェック |
| 10,000 | 10,000回チェック | 約14回チェック |
| 1,000,000 | 1,000,000回チェック | 約20回チェック |

## トランザクション：ACID保証

SQLiteはWasm内でもACIDトランザクションを提供します：

```
BEGIN TRANSACTION;
  INSERT INTO accounts (id, balance) VALUES (1, 1000);
  UPDATE accounts SET balance = balance - 100 WHERE id = 1;
  UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;
-- 3つの操作がすべて成功するか、すべてが取り消される
```

- **原子性（Atomicity）** -- すべての操作が成功するか、すべてがロールバックされる
- **一貫性（Consistency）** -- データベースの制約が常に維持される
- **分離性（Isolation）** -- 同時実行トランザクションが互いに干渉しない
- **永続性（Durability）** -- コミットされたデータはクラッシュ後も存続する（Wasmではストレージに永続化）

## データの永続化：IndexedDBとOPFS

Wasm SQLiteデータベースはデフォルトでインメモリです。ページ再読み込み後もデータを保持するにはストレージバックエンドが必要です：

### IndexedDBバックエンド

```
┌──────────┐      ┌──────────┐      ┌────────────┐
│ SQLite   │ VFS  │ JSシム    │      │ IndexedDB  │
│ (Wasm)   │─────>│(アダプタ)  │─────>│ (ブラウザ)  │
│          │      │          │      │            │
└──────────┘      └──────────┘      └────────────┘
```

SQLiteは仮想ファイルシステム（VFS）レイヤーを使用します。Wasmでは、このVFSがJavaScriptで実装され、IndexedDBへページの読み書きを行います。

### OPFSバックエンド（モダンブラウザ）

Origin Private File System（OPFS）は真のファイルシステムアクセスを提供します：

```
┌──────────┐      ┌──────────┐
│ SQLite   │ VFS  │  OPFS    │
│ (Wasm)   │─────>│(ネイティブ)│
│          │      │ ファイルI/O│
└──────────┘      └──────────┘
```

OPFSはWeb Worker内での同期読み取りをサポートするため、SQLiteの同期I/Oモデルと一致し、IndexedDBより高速です。

| ストレージバックエンド | 読み取り速度 | 書き込み速度 | 互換性 |
|----------------|------------|-------------|---------------|
| インメモリ | 最速 | 最速 | すべてのブラウザ |
| IndexedDB VFS | 約2倍遅い | 約5倍遅い | すべてのブラウザ |
| OPFS VFS | 約1.2倍遅い | 約1.5倍遅い | Chrome 102+, Firefox 111+ |

## 実際のWasm版SQLite：sql.jsと代替手段

**sql.js** -- ブラウザ内SQLiteの最も人気のあるソリューションです。EmscriptenでSQLiteのCコードをWasmにコンパイルしています：

```js
const SQL = await initSqlJs();
const db = new SQL.Database();
db.run("CREATE TABLE users (id INTEGER, name TEXT)");
db.run("INSERT INTO users VALUES (1, 'Alice')");
const results = db.exec("SELECT * FROM users");
```

**Rustの代替手段：**

| プロジェクト | アプローチ | ステータス |
|---------|----------|--------|
| sql.js | C → Emscripten → Wasm | 成熟、広く使用 |
| rusqlite + wasm | SQLite CへのRustバインディング | wasm32ターゲットで動作 |
| gluesql | 純粋RustのSQLエンジン | ネイティブWasm、C依存なし |
| limbo | SQLite互換の純粋Rust | より新しい、Wasmファースト設計 |

## パフォーマンス：SQLite Wasm vs IndexedDB

10,000行操作のベンチマーク：

```
┌──────────────────────┬──────────┬──────────┬──────────┐
│ 操作                  │ SQLite   │ IndexedDB│ 比率     │
│                      │ (Wasm)   │(ネイティブ)│          │
├──────────────────────┼──────────┼──────────┼──────────┤
│ INSERT（単一）        │ 0.02ms   │ 0.5ms    │ 25倍     │
│ INSERT（一括10K）     │ 15ms     │ 450ms    │ 30倍     │
│ SELECT（フルスキャン） │ 2ms      │ 35ms     │ 17倍     │
│ SELECT（インデックス付）│ 0.05ms   │ 0.8ms    │ 16倍     │
│ UPDATE（単一）        │ 0.03ms   │ 1.2ms    │ 40倍     │
│ 複雑なJOIN           │ 8ms      │ N/A*     │ --       │
│ 集計（COUNT）         │ 1ms      │ 30ms     │ 30倍     │
└──────────────────────┴──────────┴──────────┴──────────┘
* IndexedDBにはJOINサポートがないため、手動のJSコードが必要
```

Wasm版SQLiteはほとんどの操作でIndexedDBより劇的に高速で、JOIN、サブクエリ、ウィンドウ関数を含む完全なSQLサポートを提供します。

## Wasm版SQLiteの使いどころ

| シナリオ | 推奨 |
|----------|---------------|
| シンプルなキー・バリューストレージ | localStorageまたはIndexedDBを使用 |
| JOINを伴う複雑なクエリ | Wasm版SQLite |
| 同期付きオフラインファースト | SQLite + カスタム同期プロトコル |
| 大規模データセット（>10MB） | Wasm版SQLite（より高いパフォーマンス） |
| フォームデータ / 設定 | IndexedDBで十分 |
| 全文検索 | SQLite FTS5拡張 |

## 試してみよう

SQLエンジンを拡張してみましょう：
- カラムで結果をソートする（昇順または降順）**ORDER BY**サポートを追加する
- **COUNT、SUM、AVG**集計関数を実装する
- O(log n)ルックアップのためのカラムへのシンプルな**B-treeインデックス**を追加する
- 複数の`WhereClause`値を組み合わせてWHERE句での**AND/OR**をサポートする
- 共有カラムで行をマッチングさせて2つのテーブル間の**JOIN**を実装する
