---
title: 本番環境へのデプロイ
slug: deploying
difficulty: advanced
tags: [getting-started]
order: 16
description: Rust/Wasmアプリケーションをバンドル、最適化し、Cloudflare Pages、Vercel、Netlifyなどのプラットフォームにデプロイします。
starter_code: |
  fn main() {
      println!("=== デプロイチェックリスト ===\n");

      let checks = [
          ("Build mode", "wasm-pack build --release --target web"),
          ("Optimize binary", "wasm-opt -Oz -o opt.wasm pkg/app_bg.wasm"),
          ("Check size", "ls -lh pkg/*.wasm"),
          ("Test locally", "npx serve ."),
          ("Deploy", "npx wrangler pages deploy dist/"),
      ];

      for (i, (check, cmd)) in checks.iter().enumerate() {
          println!("  {}. {}", i + 1, check);
          println!("     $ {}\n", cmd);
      }

      println!("サイズ目標:");
      println!("  < 50KB  - 優秀");
      println!("  < 100KB - 良好");
      println!("  < 500KB - 許容範囲");
      println!("  > 1MB   - 最適化が必要");
  }
expected_output: |
  === デプロイチェックリスト ===

    1. Build mode
       $ wasm-pack build --release --target web

    2. Optimize binary
       $ wasm-opt -Oz -o opt.wasm pkg/app_bg.wasm

    3. Check size
       $ ls -lh pkg/*.wasm

    4. Test locally
       $ npx serve .

    5. Deploy
       $ npx wrangler pages deploy dist/

  サイズ目標:
    < 50KB  - 優秀
    < 100KB - 良好
    < 500KB - 許容範囲
    > 1MB   - 最適化が必要
---

## 本番用ビルド

```bash
# サイズ最適化付きリリースビルド
wasm-pack build --release --target web
```

`Cargo.toml`にリリース最適化設定を追加します：

```toml
[profile.release]
opt-level = "s"       # "s" = 小さい、"z" = 最小
lto = true            # リンク時最適化
codegen-units = 1     # より良い最適化
strip = true          # デバッグシンボルを除去
panic = "abort"       # パニック処理をコンパクトに
```

## バイナリサイズの最適化

### ステップ1：wasm-opt

```bash
# binaryenのインストール
npm install -g binaryen

# 最適化（サイズを10〜30%削減可能）
wasm-opt -Oz -o optimized.wasm pkg/my_app_bg.wasm
```

### ステップ2：サイズの内訳を確認

```bash
# twiggyのインストール
cargo install twiggy

# サイズの大きい関数を表示
twiggy top pkg/my_app_bg.wasm

# コールグラフを表示
twiggy paths pkg/my_app_bg.wasm
```

### ステップ3：不要なコードを削除

```rust
// #[cfg]を使ってWasmビルドからコードを除外
#[cfg(not(target_arch = "wasm32"))]
fn native_only() { /* ... */ }

// 小さな機能のために大きなクレートを取り込まない
// フィーチャーフラグを使って依存関係を最小化
```

## Wasmの正しい配信

サーバーは正しいMIMEタイプを送信する必要があります：

| ファイル | Content-Type |
|---------|-------------|
| `.wasm` | `application/wasm` |
| `.js` | `application/javascript` |

最近のホスティングサービスはほとんどが自動的に処理します。処理されない場合は、ヘッダーを追加します：

```
# _headersファイル（Cloudflare Pages、Netlify）
/*.wasm
  Content-Type: application/wasm
  Cache-Control: public, max-age=31536000, immutable
```

## Cloudflare Pagesへのデプロイ

```bash
# ビルド
wasm-pack build --release --target web

# HTML + pkg/を含むdist/フォルダを作成
mkdir -p dist
cp index.html dist/
cp -r pkg dist/

# デプロイ
npx wrangler pages deploy dist/ --project-name my-wasm-app
```

または、GitHubリポジトリを接続して、プッシュごとに自動デプロイすることもできます。

## Vercelへのデプロイ

```json
// vercel.json
{
  "headers": [
    {
      "source": "/(.*).wasm",
      "headers": [
        { "key": "Content-Type", "value": "application/wasm" },
        { "key": "Cache-Control", "value": "public, max-age=31536000, immutable" }
      ]
    }
  ]
}
```

## Netlifyへのデプロイ

```toml
# netlify.toml
[[headers]]
  for = "/*.wasm"
  [headers.values]
    Content-Type = "application/wasm"
    Cache-Control = "public, max-age=31536000, immutable"
```

## キャッシュ戦略

Wasmバイナリはビルド後は不変です — 積極的なキャッシュを使用しましょう：

```
# .wasmを1年間キャッシュ（コンテンツハッシュ付きファイル名）
Cache-Control: public, max-age=31536000, immutable

# JSローダーはキャッシュしない（新しい.wasmを参照する可能性）
Cache-Control: public, max-age=3600
```

## 読み込み戦略

### ストリーミングコンパイル（最速）

```js
// ブラウザがダウンロード中にWasmをコンパイル — 最速の方法
const response = await fetch('app_bg.wasm');
const module = await WebAssembly.compileStreaming(response);
```

### 遅延読み込み

```js
// 必要な時だけWasmを読み込む
let wasmModule = null;

async function getWasm() {
    if (!wasmModule) {
        wasmModule = await import('./pkg/my_app.js');
        await wasmModule.default();
    }
    return wasmModule;
}

// 初回使用時に読み込み
button.onclick = async () => {
    const wasm = await getWasm();
    wasm.process(data);
};
```

## 本番デプロイチェックリスト

| ステップ | コマンド | 理由 |
|---------|---------|------|
| リリースビルド | `wasm-pack build --release` | デバッグ情報を除去 |
| wasm-opt | `wasm-opt -Oz` | バイナリを10〜30%縮小 |
| サイズ確認 | `ls -lh pkg/*.wasm` | 100KB未満を目標 |
| MIMEタイプ確認 | Networkタブを確認 | `application/wasm`であること |
| HTTPS確認 | Wasmに必須 | HTTPでは読み込まれない |
| キャッシュ追加 | Cache-Controlヘッダーを設定 | .wasmファイルに1年間 |
| エラーレポート | `console_error_panic_hook`を追加 | 本番でのクラッシュをデバッグ |
| フォールバック | `WebAssembly`の存在を確認 | 非対応の場合メッセージを表示 |

## 試してみよう

**Run**をクリックして、デプロイチェックリストを確認しましょう。Rust/Wasmプロジェクトを公開する準備ができたら、これらのステップに従ってください。
