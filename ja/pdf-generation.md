---
title: "Wasm + PDF生成"
slug: pdf-generation
difficulty: advanced
tags: [api]
order: 47
description: Rust/Wasmでクライアントサイドのファイル生成 — PDFファイルフォーマットの内部構造を学び、最小限のPDFジェネレーターをゼロから構築します。
starter_code: |
  /// 有効なPDFファイルを生成する最小限のPDFジェネレーター
  /// PDFはテキストベースのフォーマットで、ストリーム用のバイナリセクションを含む

  struct PdfDocument {
      objects: Vec<String>,
      pages: Vec<usize>,  // ページオブジェクトのオブジェクトインデックス
  }

  impl PdfDocument {
      fn new() -> Self {
          PdfDocument {
              objects: Vec::new(),
              pages: Vec::new(),
          }
      }

      /// オブジェクトを追加し、そのオブジェクト番号（1始まり）を返す
      fn add_object(&mut self, content: String) -> usize {
          self.objects.push(content);
          self.objects.len() // 1始まりのオブジェクト番号
      }

      /// テキストコンテンツを持つページを追加する
      fn add_page(&mut self, lines: &[&str], font_size: u32) {
          // コンテンツストリーム: ページ上にテキストを配置
          let mut stream = String::new();
          stream.push_str("BT\n"); // テキスト開始
          stream.push_str(&format!("/F1 {} Tf\n", font_size));

          let mut y = 750; // ページ上部付近から開始
          for line in lines {
              stream.push_str(&format!("1 0 0 1 50 {} Tm\n", y));
              // PDF特殊文字をエスケープ
              let escaped = line
                  .replace('\\', "\\\\")
                  .replace('(', "\\(")
                  .replace(')', "\\)");
              stream.push_str(&format!("({}) Tj\n", escaped));
              y -= (font_size as i32) + 4;
          }
          stream.push_str("ET\n"); // テキスト終了

          let stream_length = stream.len();

          // ストリームオブジェクト（ページ描画コマンドを含む）
          let stream_obj = format!(
              "<< /Length {} >>\nstream\n{}endstream",
              stream_length, stream
          );
          let stream_num = self.add_object(stream_obj);

          // ページオブジェクト（ストリームとフォントを参照）
          // Parentはレンダリング時に設定される
          let page_obj = format!(
              "<< /Type /Page /Parent {{PAGES}} /MediaBox [0 0 612 792] \
               /Contents {} 0 R /Resources << /Font << /F1 {{FONT}} >> >> >>",
              stream_num
          );
          let page_num = self.add_object(page_obj);
          self.pages.push(page_num);
      }

      /// 完全なPDFをバイトベクトルとしてレンダリングする
      fn render(&self) -> Vec<u8> {
          let mut output = Vec::new();

          // PDFヘッダー
          output.extend_from_slice(b"%PDF-1.4\n");

          // フォントオブジェクト（組み込みHelveticaを使用 -- 埋め込み不要）
          let font_obj_num = self.objects.len() + 1;
          let pages_obj_num = self.objects.len() + 2;
          let catalog_obj_num = self.objects.len() + 3;

          // 相互参照テーブル用のバイトオフセットを記録
          let mut offsets: Vec<usize> = Vec::new();

          // ドキュメントオブジェクトを書き込む
          for (i, obj) in self.objects.iter().enumerate() {
              offsets.push(output.len());
              let obj_num = i + 1;
              let rendered = obj
                  .replace("{PAGES}", &format!("{} 0 R", pages_obj_num))
                  .replace("{FONT}", &format!("{} 0 R", font_obj_num));
              let line = format!("{} 0 obj\n{}\nendobj\n", obj_num, rendered);
              output.extend_from_slice(line.as_bytes());
          }

          // フォントオブジェクト（Helveticaはすべてのpdfリーダーに組み込み済み）
          offsets.push(output.len());
          let font = format!(
              "{} 0 obj\n<< /Type /Font /Subtype /Type1 /BaseFont /Helvetica >>\nendobj\n",
              font_obj_num
          );
          output.extend_from_slice(font.as_bytes());

          // Pagesオブジェクト（すべてのページオブジェクトの親）
          offsets.push(output.len());
          let page_refs: Vec<String> = self.pages.iter()
              .map(|p| format!("{} 0 R", p))
              .collect();
          let pages = format!(
              "{} 0 obj\n<< /Type /Pages /Kids [{}] /Count {} >>\nendobj\n",
              pages_obj_num,
              page_refs.join(" "),
              self.pages.len()
          );
          output.extend_from_slice(pages.as_bytes());

          // Catalogオブジェクト（PDFのルート）
          offsets.push(output.len());
          let catalog = format!(
              "{} 0 obj\n<< /Type /Catalog /Pages {} 0 R >>\nendobj\n",
              catalog_obj_num, pages_obj_num
          );
          output.extend_from_slice(catalog.as_bytes());

          // 相互参照テーブル
          let xref_offset = output.len();
          let total_objects = offsets.len();
          let mut xref = format!("xref\n0 {}\n", total_objects + 1);
          xref.push_str("0000000000 65535 f \n");
          for offset in &offsets {
              xref.push_str(&format!("{:010} 00000 n \n", offset));
          }
          output.extend_from_slice(xref.as_bytes());

          // トレーラー
          let trailer = format!(
              "trailer\n<< /Size {} /Root {} 0 R >>\nstartxref\n{}\n%%EOF\n",
              total_objects + 1,
              catalog_obj_num,
              xref_offset
          );
          output.extend_from_slice(trailer.as_bytes());

          output
      }
  }

  fn main() {
      let mut doc = PdfDocument::new();

      // ページ1: タイトルページ
      doc.add_page(&[
          "Rust + WebAssembly PDF Generator",
          "",
          "Generated entirely client-side",
          "No server round-trip needed!",
          "",
          "This PDF was created by a minimal",
          "PDF generator written in pure Rust.",
      ], 16);

      // ページ2: コンテンツページ
      doc.add_page(&[
          "Chapter 1: Why Client-Side PDF?",
          "",
          "Benefits:",
          "  - Privacy: data never leaves the browser",
          "  - Speed: no network latency",
          "  - Offline: works without internet",
          "  - Cost: no server infrastructure needed",
          "",
          "The PDF format uses a tree of objects:",
          "  Catalog -> Pages -> Page -> Content Stream",
          "",
          "Each page has a MediaBox defining its size",
          "and a Content Stream with drawing commands.",
      ], 12);

      let pdf_bytes = doc.render();

      println!("=== PDF生成完了 ===");
      println!("PDFサイズ: {} バイト", pdf_bytes.len());
      println!("ページ数: {}", doc.pages.len());
      println!();

      // 構造を表示
      let pdf_str = String::from_utf8_lossy(&pdf_bytes);
      println!("=== PDF構造（最初の500文字） ===");
      let preview: String = pdf_str.chars().take(500).collect();
      println!("{}", preview);
      println!("...");
      println!();

      // 先頭と末尾が正しいか検証
      println!("=== バリデーション ===");
      println!("%PDF-1.4で始まる: {}",
          pdf_str.starts_with("%PDF-1.4"));
      println!("%%EOFで終わる: {}",
          pdf_str.trim().ends_with("%%EOF"));
      println!("xrefテーブルを含む: {}",
          pdf_str.contains("xref"));
      println!("trailerを含む: {}",
          pdf_str.contains("trailer"));
  }
expected_output: |
  === PDF生成完了 ===
  PDFサイズ: 約1200バイト
  ページ数: 2

  === PDF構造（最初の500文字） ===
  %PDF-1.4
  1 0 obj
  << /Length ... >>
  stream
  BT
  /F1 16 Tf
  ...

  === バリデーション ===
  %PDF-1.4で始まる: true
  %%EOFで終わる: true
  xrefテーブルを含む: true
  trailerを含む: true
---

## はじめに

Rust/Wasmでクライアントサイドにてpdfを生成することで、ドキュメント作成のためにサーバーへ機密データを送信する必要がなくなります。請求書、レポート、証明書、領収書をすべてブラウザ内で直接生成できます。このレッスンでは最小限ながら有効なPDFジェネレーターをゼロから構築し、PDFファイルフォーマットの内部構造を解説します。

## なぜクライアントサイドでPDF生成するのか？

| アプローチ | レイテンシ | プライバシー | オフライン | サーバーコスト |
|----------|---------|---------|---------|-------------|
| サーバーサイド (wkhtmltopdf) | 200-500ms + ネットワーク | データがサーバーに送信される | 不可 | 高い |
| サーバーサイド (Puppeteer) | 500-2000ms + ネットワーク | データがサーバーに送信される | 不可 | 非常に高い |
| クライアントJS (jsPDF) | 50-200ms | データはローカルに留まる | 可能 | なし |
| クライアントWasm (Rust) | 10-50ms | データはローカルに留まる | 可能 | なし |

WasmはPDF生成においてJavaScriptより3〜5倍高速です。大きなバイト配列の構築時にGC負荷を回避し、効率的な文字列フォーマットを行うためです。

## PDFファイルフォーマットの構造

PDFファイルは4つの主要セクションで構成されています：

```
┌──────────────────────────────┐
│  ヘッダー                     │  %PDF-1.4
├──────────────────────────────┤
│  ボディ                       │  オブジェクト（ページ、フォント、
│  （番号付きオブジェクト）      │   画像、コンテンツストリーム）
├──────────────────────────────┤
│  相互参照テーブル              │  ランダムアクセスのための
│  (xref)                      │  各オブジェクトのバイトオフセット
├──────────────────────────────┤
│  トレーラー                   │  ルートオブジェクトと
│                              │  xrefテーブルへのポインタ
└──────────────────────────────┘
```

### オブジェクト階層

すべてのPDFはこのツリー構造を持ちます：

```
Catalog（ルート）
└── Pages（コレクション）
    ├── Page 1
    │   ├── MediaBox [0 0 612 792]  (USレターサイズ、ポイント単位)
    │   ├── Resources
    │   │   └── Font /F1 → Helvetica
    │   └── Contents → Streamオブジェクト
    │       └── "BT /F1 12 Tf (Hello) Tj ET"
    ├── Page 2
    │   └── ...
    └── Page N
```

### PDFオブジェクト

オブジェクトは番号付き（1 0 obj、2 0 objなど）で、辞書、配列、ストリーム、プリミティブ値のいずれかです：

```
% 辞書オブジェクト
1 0 obj
<< /Type /Catalog /Pages 2 0 R >>
endobj

% ストリームオブジェクト（描画コマンドを含む）
3 0 obj
<< /Length 44 >>
stream
BT
/F1 12 Tf
1 0 0 1 50 750 Tm
(Hello World) Tj
ET
endstream
endobj
```

`2 0 R`の`R`は「オブジェクト2、世代0への参照」を意味します -- これがオブジェクト間のリンク方法です。

## コンテンツストリームコマンド

PDFコンテンツストリームはPostScriptに似たスタックベースの描画言語を使用します：

| コマンド | 意味 | 例 |
|---------|---------|---------|
| `BT` | テキストブロック開始 | `BT` |
| `ET` | テキストブロック終了 | `ET` |
| `Tf` | フォントとサイズを設定 | `/F1 12 Tf` |
| `Tm` | テキスト行列（位置）を設定 | `1 0 0 1 50 750 Tm` |
| `Tj` | テキスト文字列を表示 | `(Hello) Tj` |
| `Td` | テキスト位置を移動 | `0 -14 Td` |
| `m` | 点に移動 | `50 700 m` |
| `l` | 点まで線を引く | `550 700 l` |
| `S` | パスをストローク | `S` |
| `re` | 矩形 | `50 50 500 700 re` |
| `rg` | 塗りつぶし色を設定 (RGB) | `1 0 0 rg`（赤） |

座標系は左下隅から始まります。USレターは612 x 792ポイントです（1ポイント = 1/72インチ）。

## 組み込みフォント

PDFはすべてのPDFリーダーがサポートしなければならない14の標準フォントを定義しています -- 埋め込み不要：

```
Helvetica              Helvetica-Bold
Helvetica-Oblique      Helvetica-BoldOblique
Times-Roman            Times-Bold
Times-Italic           Times-BoldItalic
Courier                Courier-Bold
Courier-Oblique        Courier-BoldOblique
Symbol                 ZapfDingbats
```

これらのフォントを使用することでPDFを小さく保てます。カスタムフォントにはフォントファイルの埋め込みが必要で、数百キロバイトが追加される場合があります。

## 相互参照テーブル

xrefテーブルはファイル全体を読み込まずにオブジェクトへのランダムアクセスを可能にします。各エントリはファイル先頭からのオブジェクトのバイトオフセットを記録します：

```
xref
0 5
0000000000 65535 f     ← フリーオブジェクト（常に最初）
0000000009 00000 n     ← オブジェクト1（バイト9）
0000000058 00000 n     ← オブジェクト2（バイト58）
0000000115 00000 n     ← オブジェクト3（バイト115）
0000000258 00000 n     ← オブジェクト4（バイト258）
```

これは大きなPDFにとって重要です -- リーダーはページ1〜499をパースせずに直接ページ500にジャンプできます。

## PDFへの画像の追加

PDF内の画像は特定のフィルターを持つストリームオブジェクトとして保存されます：

```
% 画像オブジェクト（概念図）
5 0 obj
<< /Type /XObject
   /Subtype /Image
   /Width 200
   /Height 150
   /ColorSpace /DeviceRGB
   /BitsPerComponent 8
   /Length 90000
   /Filter /DCTDecode    ← JPEGデータを意味する
>>
stream
[生のJPEGバイト]
endstream
endobj
```

Wasmでは、JavaScriptからRustに画像バイトを渡し、JPEG（DCTDecode）またはPNG（FlateDecode）ストリームとして直接埋め込めます。

## テーブル生成

PDF内のテーブルは線とテキストのコマンドを使って手動で描画します：

```
シンプルなテーブルのコンテンツストリーム:
% グリッド線を描画
0.5 w                    % 線幅
50 700 m 550 700 l S     % 上罫線
50 680 m 550 680 l S     % ヘッダー区切り
50 660 m 550 660 l S     % 行1
50 700 m 50 660 l S      % 左罫線
300 700 m 300 660 l S    % 列区切り
550 700 m 550 660 l S    % 右罫線

% ヘッダーテキスト
BT /F1 10 Tf
1 0 0 1 55 685 Tm (Name) Tj
1 0 0 1 305 685 Tm (Value) Tj
ET
```

これがPDFライブラリが価値を持つ理由です -- セル位置の計算、テキスト折り返しの処理、改ページの管理を手動で行うのは非常に手間がかかります。

## プロダクション向けクレート：printpdfとgenpdf

実際のアプリケーションでは、以下のRustクレートがWasmにコンパイルできます：

**printpdf** -- 低レベルPDF生成：
```rust
// printpdfクレートが必要
use printpdf::*;

let (doc, page1, layer1) = PdfDocument::new(
    "My Document", Mm(210.0), Mm(297.0), "Layer 1"
);
let font = doc.add_builtin_font(BuiltinFont::Helvetica).unwrap();
let layer = doc.get_page(page1).get_layer(layer1);
layer.use_text("Hello from Rust!", 14.0, Mm(10.0), Mm(280.0), &font);
```

**genpdf** -- レイアウトエンジン付きの高レベルAPI：
```rust
// genpdfクレートが必要
let font = genpdf::fonts::from_files("./fonts", "Helvetica", None).unwrap();
let mut doc = genpdf::Document::new(font);
doc.push(genpdf::elements::Paragraph::new("Hello from Rust!"));
doc.push(genpdf::elements::Break::new(1));
doc.push(genpdf::elements::Paragraph::new("Second paragraph."));
```

## 試してみよう

PDFジェネレーターを拡張してみましょう：
- ページ全幅の水平線（罫線）の**描画コマンド**を追加する
- ストリーム途中で`/F2`（Helvetica-Bold）に切り替えて**太字テキスト**をサポートする
- 各ページの下部に**ページ番号**を追加する
- グリッド線とセルテキストで**テーブル**を生成する
- `rg`（塗りつぶし色）コマンドを使ってテキストに**色**を追加する
