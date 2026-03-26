---
title: "Wasmでの正規表現とテキスト処理"
slug: regex-text-processing
difficulty: intermediate
tags: [data-structures]
order: 42
description: RustのWebAssemblyで超高速な正規表現エンジンを活用 — 入力検証、テキスト検索、ログ解析、CSVデータ処理、Unicode正規処理の方法を学びます。
starter_code: |
  use std::collections::HashMap;

  // プレーンRustで正規表現の概念とテキスト処理をデモします。
  // ここで使うパターンは、Wasmにコンパイルされた`regex`クレートで
  // 使うものと同じです。内部構造を示すためにミニマッチャーを実装します。

  // --- シンプルなパターンマッチャー（正規表現の概念をシミュレーション） ---

  #[derive(Debug, Clone)]
  enum Pattern {
      Literal(String),
      AnyChar,                     // .
      Digit,                       // \d
      Word,                        // \w
      Whitespace,                  // \s
      OneOrMore(Box<Pattern>),     // +
      ZeroOrMore(Box<Pattern>),    // *
      Optional(Box<Pattern>),      // ?
      Group(Vec<Pattern>),         // (...)
  }

  fn matches_char(pat: &Pattern, ch: char) -> bool {
      match pat {
          Pattern::Literal(s) => s.len() == 1 && s.chars().next() == Some(ch),
          Pattern::AnyChar => ch != '\n',
          Pattern::Digit => ch.is_ascii_digit(),
          Pattern::Word => ch.is_alphanumeric() || ch == '_',
          Pattern::Whitespace => ch.is_whitespace(),
          _ => false,
      }
  }

  // --- メールバリデータ（一般的な正規表現パターンのデモ） ---
  fn is_valid_email(email: &str) -> bool {
      // パターン: [word+.-]+@[word.-]+\.[a-zA-Z]{2,}
      let parts: Vec<&str> = email.splitn(2, '@').collect();
      if parts.len() != 2 { return false; }

      let local = parts[0];
      let domain = parts[1];

      // ローカル部: 1文字以上、英数字、ドット、ハイフン、プラスのみ
      if local.is_empty() { return false; }
      if !local.chars().all(|c| c.is_alphanumeric() || c == '.' || c == '-' || c == '_' || c == '+') {
          return false;
      }

      // ドメイン: ドットを含む必要あり、有効な文字
      let domain_parts: Vec<&str> = domain.rsplitn(2, '.').collect();
      if domain_parts.len() != 2 { return false; }

      let tld = domain_parts[0];
      let host = domain_parts[1];

      if tld.len() < 2 || !tld.chars().all(|c| c.is_ascii_alphabetic()) { return false; }
      if host.is_empty() || !host.chars().all(|c| c.is_alphanumeric() || c == '.' || c == '-') {
          return false;
      }

      true
  }

  // --- ログパーサー ---
  #[derive(Debug)]
  struct LogEntry {
      timestamp: String,
      level: String,
      message: String,
  }

  fn parse_log_line(line: &str) -> Option<LogEntry> {
      // 形式: [2024-01-15 09:23:45] ERROR: Something went wrong
      let line = line.trim();
      if !line.starts_with('[') { return None; }

      let bracket_end = line.find(']')?;
      let timestamp = line[1..bracket_end].to_string();

      let rest = line[bracket_end + 1..].trim();
      let colon_pos = rest.find(':')?;
      let level = rest[..colon_pos].trim().to_string();
      let message = rest[colon_pos + 1..].trim().to_string();

      Some(LogEntry { timestamp, level, message })
  }

  // --- CSVパーサー ---
  fn parse_csv_row(line: &str) -> Vec<String> {
      let mut fields = Vec::new();
      let mut current = String::new();
      let mut in_quotes = false;

      let chars: Vec<char> = line.chars().collect();
      let mut i = 0;
      while i < chars.len() {
          let ch = chars[i];
          if in_quotes {
              if ch == '"' {
                  if i + 1 < chars.len() && chars[i + 1] == '"' {
                      current.push('"');
                      i += 1;
                  } else {
                      in_quotes = false;
                  }
              } else {
                  current.push(ch);
              }
          } else if ch == '"' {
              in_quotes = true;
          } else if ch == ',' {
              fields.push(current.trim().to_string());
              current = String::new();
          } else {
              current.push(ch);
          }
          i += 1;
      }
      fields.push(current.trim().to_string());
      fields
  }

  // --- コンテキスト付きテキスト検索（grep -C と同様） ---
  fn search_with_context(text: &str, query: &str, context_lines: usize) -> Vec<String> {
      let lines: Vec<&str> = text.lines().collect();
      let mut results = Vec::new();
      let query_lower = query.to_lowercase();

      for (i, line) in lines.iter().enumerate() {
          if line.to_lowercase().contains(&query_lower) {
              let start = i.saturating_sub(context_lines);
              let end = (i + context_lines + 1).min(lines.len());

              let mut block = Vec::new();
              for j in start..end {
                  let marker = if j == i { ">>>" } else { "   " };
                  block.push(format!("{} {:>3}: {}", marker, j + 1, lines[j]));
              }
              results.push(block.join("\n"));
          }
      }
      results
  }

  // --- テキスト置換（sed s/pattern/replacement/g と同様） ---
  fn replace_all(text: &str, find: &str, replace: &str) -> String {
      text.replace(find, replace)
  }

  // --- 単語頻度カウンター ---
  fn word_frequency(text: &str) -> Vec<(String, usize)> {
      let mut freq: HashMap<String, usize> = HashMap::new();
      for word in text.split_whitespace() {
          let cleaned: String = word.chars()
              .filter(|c| c.is_alphanumeric() || *c == '\'')
              .collect::<String>()
              .to_lowercase();
          if !cleaned.is_empty() {
              *freq.entry(cleaned).or_insert(0) += 1;
          }
      }
      let mut sorted: Vec<(String, usize)> = freq.into_iter().collect();
      sorted.sort_by(|a, b| b.1.cmp(&a.1).then(a.0.cmp(&b.0)));
      sorted
  }

  // --- Unicode処理 ---
  fn unicode_stats(text: &str) -> (usize, usize, usize, usize) {
      let chars = text.chars().count();
      let bytes = text.len();
      let words = text.split_whitespace().count();
      let lines = text.lines().count();
      (chars, bytes, words, lines)
  }

  fn main() {
      println!("=== 正規表現 & テキスト処理デモ ===\n");

      // 1. メール検証
      println!("--- メール検証 ---");
      let emails = vec![
          "user@example.com",
          "john.doe+tag@company.co.uk",
          "invalid@",
          "@no-local.com",
          "spaces in@email.com",
          "valid_addr@sub.domain.org",
      ];
      for email in &emails {
          println!("  {:<35} {}", email, if is_valid_email(email) { "有効" } else { "無効" });
      }

      // 2. ログ解析
      println!("\n--- ログ解析 ---");
      let log_lines = vec![
          "[2024-01-15 09:23:45] ERROR: Database connection failed",
          "[2024-01-15 09:23:46] INFO: Retrying connection...",
          "[2024-01-15 09:23:47] WARN: Connection slow (2.3s)",
          "malformed line without brackets",
      ];
      for line in &log_lines {
          match parse_log_line(line) {
              Some(entry) => println!("  [{:>5}] {} - {}", entry.level, entry.timestamp, entry.message),
              None => println!("  スキップ: {}", line),
          }
      }

      // 3. CSV解析
      println!("\n--- CSV解析 ---");
      let csv_data = r#"Name,Age,City
  Alice,30,"New York"
  Bob,25,"San Francisco"
  "O'Brien, Charlie",40,London"#;
      println!("  入力CSV:");
      for line in csv_data.lines() {
          println!("    {}", line.trim());
      }
      println!("  解析結果:");
      for (i, line) in csv_data.lines().enumerate() {
          let fields = parse_csv_row(line.trim());
          println!("    行 {}: {:?}", i, fields);
      }

      // 4. コンテキスト付きテキスト検索
      println!("\n--- テキスト検索（クエリ: 'error'） ---");
      let document = "Line 1: All systems normal\nLine 2: Warning detected\nLine 3: Error in module X\nLine 4: Recovery started\nLine 5: System restored";
      let matches = search_with_context(document, "error", 1);
      for m in &matches {
          println!("{}", m);
      }

      // 5. 単語頻度
      println!("\n--- 単語頻度 ---");
      let text = "the quick brown fox jumps over the lazy dog the fox";
      let freq = word_frequency(text);
      println!("  テキスト: \"{}\"", text);
      println!("  上位の単語:");
      for (word, count) in freq.iter().take(5) {
          println!("    {:<10} {}", word, count);
      }

      // 6. Unicode統計
      println!("\n--- Unicode処理 ---");
      let unicode_text = "Hello, world!\nRust + Wasm = fast";
      let (chars, bytes, words, lines) = unicode_stats(unicode_text);
      println!("  テキスト: {:?}", unicode_text);
      println!("  文字数: {}, バイト数: {}, 単語数: {}, 行数: {}", chars, bytes, words, lines);
  }
expected_output: |
  === 正規表現 & テキスト処理デモ ===

  --- メール検証 ---
    user@example.com                    有効
    john.doe+tag@company.co.uk          有効
    invalid@                            無効
    @no-local.com                       無効
    spaces in@email.com                 無効
    valid_addr@sub.domain.org           有効

  --- ログ解析 ---
    [ERROR] 2024-01-15 09:23:45 - Database connection failed
    [ INFO] 2024-01-15 09:23:46 - Retrying connection...
    [ WARN] 2024-01-15 09:23:47 - Connection slow (2.3s)
    スキップ: malformed line without brackets

  --- CSV解析 ---
    入力CSV:
      Name,Age,City
      Alice,30,"New York"
      Bob,25,"San Francisco"
      "O'Brien, Charlie",40,London
    解析結果:
      行 0: ["Name", "Age", "City"]
      行 1: ["Alice", "30", "New York"]
      行 2: ["Bob", "25", "San Francisco"]
      行 3: ["O'Brien, Charlie", "40", "London"]

  --- テキスト検索（クエリ: 'error'） ---
      2: Line 2: Warning detected
  >>>  3: Line 3: Error in module X
      4: Line 4: Recovery started

  --- 単語頻度 ---
    テキスト: "the quick brown fox jumps over the lazy dog the fox"
    上位の単語:
      the        3
      fox        2
      brown      1
      dog        1
      jumps      1

  --- Unicode処理 ---
    テキスト: "Hello, world!\nRust + Wasm = fast"
    文字数: 32, バイト数: 32, 単語数: 7, 行数: 2
---

## なぜWasmでRustの正規表現を使うのか?

JavaScriptには組み込みの正規表現がありますが、なぜRustの`regex`クレートをWasmにコンパイルするのでしょうか？パフォーマンス、正確性、そして機能面で優れているからです：

```
  ベンチマーク: 10,000件のメールアドレスをマッチ

  JS RegExp          ████████████████████████  24ms
  Rust regex (Wasm)  ████████  8ms
  Rust regex (native)████  4ms

  ← 高速                         低速 →
```

| 機能                     | JavaScript RegExp         | Rustの`regex`クレート       |
|--------------------------|---------------------------|----------------------------|
| エンジン種類              | バックトラッキング（NFA/DFA）| 保証されたDFA/NFAハイブリッド |
| 壊滅的バックトラッキング    | 発生しうる（ReDoS）         | 不可能（線形時間）           |
| Unicodeサポート           | 部分的（ES2018以降）        | 完全（Unicodeカテゴリ）      |
| 名前付きキャプチャ         | あり（ES2018）             | あり                        |
| 先読み/後読み             | あり                       | なし（O(n)を保証）           |
| コンパイル速度            | 高速                       | やや遅い（キャッシュ可能）    |
| マッチ速度               | 良好                       | 優秀（2-5倍高速）            |

### ReDoS防護

Rustの`regex`クレートはパターンに関係なく**O(n)**のマッチング時間を保証します。これにより正規表現によるサービス拒否攻撃（ReDoS）に対して安全です：

```
  パターン: (a+)+$     入力: "aaaaaaaaaaaaaaaaX"

  JavaScript RegExp:
    2^n通りの組み合わせを試行 → 数秒間ハング
    これはReDoS脆弱性です!

  Rustのregexクレート:
    線形スキャン → 即座に「マッチなし」を返す
    設計上安全（バックトラッキングなし）
```

## Wasmでのregexクレートの使用

```rust
use regex::Regex;
use wasm_bindgen::prelude::*;

#[wasm_bindgen]
pub struct TextProcessor {
    email_re: Regex,
    url_re: Regex,
    phone_re: Regex,
}

#[wasm_bindgen]
impl TextProcessor {
    #[wasm_bindgen(constructor)]
    pub fn new() -> Self {
        TextProcessor {
            email_re: Regex::new(r"[\w.+-]+@[\w-]+\.[\w.]+").unwrap(),
            url_re: Regex::new(r"https?://[\w\-._~:/?#\[\]@!$&'()*+,;=%]+").unwrap(),
            phone_re: Regex::new(r"\+?1?[-.\s]?\(?\d{3}\)?[-.\s]?\d{3}[-.\s]?\d{4}").unwrap(),
        }
    }

    pub fn find_emails(&self, text: &str) -> Vec<String> {
        self.email_re.find_iter(text)
            .map(|m| m.as_str().to_string())
            .collect()
    }

    pub fn validate_email(&self, email: &str) -> bool {
        self.email_re.is_match(email)
    }

    pub fn redact_phones(&self, text: &str) -> String {
        self.phone_re.replace_all(text, "[秘匿]").to_string()
    }
}
```

### regexをWasmにコンパイル — サイズの考慮事項

`regex`クレートはWasmバイナリに約200-400 KBを追加します（Unicodeテーブルの包含状況による）。サイズを削減するには：

```toml
# Cargo.toml
[dependencies]
regex = { version = "1", default-features = false, features = ["std"] }
# Unicodeテーブルを削除: 約100 KB節約だが\p{...}パターンが使えなくなる
```

## WasmアプリでよくあるRegexパターン

```
  パターン                      マッチ対象                  ユースケース
  ─────────────────────────── ───────────────────────── ──────────────────
  [\w.+-]+@[\w-]+\.[\w.]+     user@example.com          メール検証
  https?://[^\s]+             http://example.com/path   URL抽出
  \d{4}-\d{2}-\d{2}          2024-01-15                ISO日付
  \d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}  192.168.1.1     IPアドレス
  #[0-9a-fA-F]{6}            #ff00aa                   16進カラー
  \b[A-Z]{2,}\b              NASA, FBI                 頭字語
  ^\s*$                       （空行/空白行）              空白行
  <[^>]+>                     <div class="x">           HTMLタグ
```

## テキスト検索と置換

### コンパイル済みregexによる効率的な検索

```rust
use regex::Regex;

// 一度コンパイルして何度も使用
lazy_static! {
    static ref LOG_PATTERN: Regex = Regex::new(
        r"\[(\d{4}-\d{2}-\d{2} \d{2}:\d{2}:\d{2})\] (ERROR|WARN|INFO|DEBUG): (.+)"
    ).unwrap();
}

fn parse_log(line: &str) -> Option<(String, String, String)> {
    LOG_PATTERN.captures(line).map(|caps| (
        caps[1].to_string(),  // タイムスタンプ
        caps[2].to_string(),  // レベル
        caps[3].to_string(),  // メッセージ
    ))
}
```

### キャプチャを使った検索と置換

```rust
use regex::Regex;

fn anonymize_data(text: &str) -> String {
    let email_re = Regex::new(r"([\w.+-]+)@([\w-]+\.[\w.]+)").unwrap();
    let phone_re = Regex::new(r"\d{3}[-.]?\d{3}[-.]?\d{4}").unwrap();

    let result = email_re.replace_all(text, "***@$2");  // ドメインは保持
    let result = phone_re.replace_all(&result, "***-***-****");
    result.to_string()
}
```

## WasmでのCSV処理

CSV解析はWasmがJavaScriptを大幅に上回る一般的なユースケースです：

```
  100,000行のCSVを処理

  Papa Parse (JS)     ████████████████████████████████  320ms
  Rust csvクレート     ████████████  120ms
  (Wasm)

  ← 高速                                    低速 →
```

```rust
use csv::ReaderBuilder;
use wasm_bindgen::prelude::*;

#[wasm_bindgen]
pub fn parse_csv(data: &str) -> JsValue {
    let mut reader = ReaderBuilder::new()
        .has_headers(true)
        .from_reader(data.as_bytes());

    let mut rows: Vec<Vec<String>> = Vec::new();
    for result in reader.records() {
        if let Ok(record) = result {
            rows.push(record.iter().map(|s| s.to_string()).collect());
        }
    }

    serde_wasm_bindgen::to_value(&rows).unwrap()
}
```

## Unicode処理

Rustの`regex`クレートは優れたUnicodeサポートを備えており、国際的なテキスト処理において重要です：

```
  文字               UTF-8バイト数  Rust char    JS char
  ────────────────── ────────────── ──────────── ─────────
  'A'                1バイト        1 char       1コード単位
  アクセント付き'e'    2バイト        1 char       1コード単位
  漢字               3バイト        1 char       1コード単位
  絵文字             4バイト        1 char       2コード単位!
```

```rust
use regex::Regex;

// Unicode対応の単語境界
let re = Regex::new(r"\b\w+\b").unwrap();  // Unicodeで動作!

// Unicodeカテゴリマッチング
let re = Regex::new(r"\p{Han}+").unwrap();       // 漢字
let re = Regex::new(r"\p{Cyrillic}+").unwrap();  // ロシア語/キリル文字
let re = Regex::new(r"\p{Emoji}+").unwrap();     // 絵文字

// Unicode対応の大文字小文字無視
let re = Regex::new(r"(?i)strasse|straße").unwrap();  // ドイツ語にマッチ
```

## パフォーマンス最適化のヒント

```
  最適化                              効果
  ──────────────────────────────────── ──────────────────────────
  regexを一度コンパイルして再利用       10-100倍高速化
  find()の代わりにis_match()を使用     より高速（キャプチャ割当てなし）
  パターンにアンカーを付ける（^...$）   テキスト全体のスキャンを回避
  複数パターンにはRegexSetを使用       N回スキャンの代わりに1回
  ASCII専用ならUnicodeを無効化          バイナリ縮小、高速化
  生バイトにはbytes::Regexを使用       UTF-8検証を回避
```

### RegexSet: 1回のパスで複数パターンをマッチ

```rust
use regex::RegexSet;

let set = RegexSet::new(&[
    r"[\w.+-]+@[\w-]+\.[\w.]+",    // メール
    r"https?://[^\s]+",             // URL
    r"\d{3}[-.]?\d{3}[-.]?\d{4}",  // 電話番号
    r"\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}",  // IP
]).unwrap();

let text = "Contact john@example.com or visit https://example.com";
let matches: Vec<usize> = set.matches(text).into_iter().collect();
// matches = [0, 1]  → メールとURLパターンを検出
```

## まとめ

WasmにコンパイルしたRustの`regex`クレートは、保証された線形時間マッチング（ReDoSなし）でJavaScriptの2-5倍高速なテキスト処理を提供します。メール検証、ログ解析、CSV処理、検索と置換、その他テキスト集約型のワークロードに活用しましょう。Wasmバイナリには約200-400 KBが追加されますが、パフォーマンスと安全性でその価値を十分に発揮します。完全なUnicodeサポートと組み合わせれば、国際的なテキスト処理がそのまま動作します。
