---
title: "Wasm + SVG操作"
slug: svg-manipulation
difficulty: intermediate
tags: [graphics]
order: 43
description: Rustからのグラフィクスの生成と操作 — ベジェ曲線の計算、チャートパスの構築、SVG出力の最適化、Wasmで動かす動的ビジュアライゼーションの作成方法を学びます。
starter_code: |
  use std::fmt;

  // プレーンRustでSVG生成とベジェ曲線の数学をデモします。
  // 実際のWasmプロジェクトでは、SVG文字列をweb-sys経由でDOMに渡すか、
  // JavaScriptに返して挿入します。

  // --- 2D点 ---
  #[derive(Debug, Clone, Copy)]
  struct Point {
      x: f64,
      y: f64,
  }

  impl Point {
      fn new(x: f64, y: f64) -> Self {
          Point { x, y }
      }

      fn distance(&self, other: &Point) -> f64 {
          ((self.x - other.x).powi(2) + (self.y - other.y).powi(2)).sqrt()
      }

      fn lerp(&self, other: &Point, t: f64) -> Point {
          Point {
              x: self.x + (other.x - self.x) * t,
              y: self.y + (other.y - self.y) * t,
          }
      }
  }

  impl fmt::Display for Point {
      fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
          write!(f, "({:.1}, {:.1})", self.x, self.y)
      }
  }

  // --- SVGパスビルダー ---
  struct SvgPath {
      commands: Vec<String>,
  }

  impl SvgPath {
      fn new() -> Self {
          SvgPath { commands: Vec::new() }
      }

      fn move_to(&mut self, p: Point) -> &mut Self {
          self.commands.push(format!("M {:.1} {:.1}", p.x, p.y));
          self
      }

      fn line_to(&mut self, p: Point) -> &mut Self {
          self.commands.push(format!("L {:.1} {:.1}", p.x, p.y));
          self
      }

      fn cubic_to(&mut self, cp1: Point, cp2: Point, end: Point) -> &mut Self {
          self.commands.push(format!(
              "C {:.1} {:.1}, {:.1} {:.1}, {:.1} {:.1}",
              cp1.x, cp1.y, cp2.x, cp2.y, end.x, end.y
          ));
          self
      }

      fn quad_to(&mut self, cp: Point, end: Point) -> &mut Self {
          self.commands.push(format!(
              "Q {:.1} {:.1}, {:.1} {:.1}",
              cp.x, cp.y, end.x, end.y
          ));
          self
      }

      fn close(&mut self) -> &mut Self {
          self.commands.push("Z".to_string());
          self
      }

      fn to_string(&self) -> String {
          self.commands.join(" ")
      }
  }

  // --- ベジェ曲線の評価 ---

  fn quadratic_bezier(p0: Point, p1: Point, p2: Point, t: f64) -> Point {
      let a = p0.lerp(&p1, t);
      let b = p1.lerp(&p2, t);
      a.lerp(&b, t)
  }

  fn cubic_bezier(p0: Point, p1: Point, p2: Point, p3: Point, t: f64) -> Point {
      let a = p0.lerp(&p1, t);
      let b = p1.lerp(&p2, t);
      let c = p2.lerp(&p3, t);
      let d = a.lerp(&b, t);
      let e = b.lerp(&c, t);
      d.lerp(&e, t)
  }

  fn bezier_length(p0: Point, p1: Point, p2: Point, p3: Point, segments: usize) -> f64 {
      let mut length = 0.0;
      let mut prev = p0;
      for i in 1..=segments {
          let t = i as f64 / segments as f64;
          let curr = cubic_bezier(p0, p1, p2, p3, t);
          length += prev.distance(&curr);
          prev = curr;
      }
      length
  }

  // --- チャート生成器 ---

  fn generate_line_chart(data: &[f64], width: f64, height: f64, padding: f64) -> String {
      if data.is_empty() { return String::new(); }

      let min_val = data.iter().cloned().fold(f64::INFINITY, f64::min);
      let max_val = data.iter().cloned().fold(f64::NEG_INFINITY, f64::max);
      let range = if (max_val - min_val).abs() < 1e-10 { 1.0 } else { max_val - min_val };

      let chart_w = width - 2.0 * padding;
      let chart_h = height - 2.0 * padding;

      let points: Vec<Point> = data.iter().enumerate().map(|(i, &v)| {
          Point::new(
              padding + (i as f64 / (data.len() - 1) as f64) * chart_w,
              padding + chart_h - ((v - min_val) / range) * chart_h,
          )
      }).collect();

      let mut path = SvgPath::new();
      path.move_to(points[0]);
      for p in &points[1..] {
          path.line_to(*p);
      }

      path.to_string()
  }

  fn generate_smooth_line_chart(data: &[f64], width: f64, height: f64, padding: f64) -> String {
      if data.len() < 2 { return String::new(); }

      let min_val = data.iter().cloned().fold(f64::INFINITY, f64::min);
      let max_val = data.iter().cloned().fold(f64::NEG_INFINITY, f64::max);
      let range = if (max_val - min_val).abs() < 1e-10 { 1.0 } else { max_val - min_val };

      let chart_w = width - 2.0 * padding;
      let chart_h = height - 2.0 * padding;

      let points: Vec<Point> = data.iter().enumerate().map(|(i, &v)| {
          Point::new(
              padding + (i as f64 / (data.len() - 1) as f64) * chart_w,
              padding + chart_h - ((v - min_val) / range) * chart_h,
          )
      }).collect();

      let mut path = SvgPath::new();
      path.move_to(points[0]);

      // Catmull-Romから3次ベジェへの変換で滑らかな曲線を生成
      for i in 0..points.len() - 1 {
          let p0 = if i > 0 { points[i - 1] } else { points[i] };
          let p1 = points[i];
          let p2 = points[i + 1];
          let p3 = if i + 2 < points.len() { points[i + 2] } else { points[i + 1] };

          let tension = 6.0;
          let cp1 = Point::new(
              p1.x + (p2.x - p0.x) / tension,
              p1.y + (p2.y - p0.y) / tension,
          );
          let cp2 = Point::new(
              p2.x - (p3.x - p1.x) / tension,
              p2.y - (p3.y - p1.y) / tension,
          );

          path.cubic_to(cp1, cp2, p2);
      }

      path.to_string()
  }

  // --- SVG要素ビルダー ---

  fn svg_circle(cx: f64, cy: f64, r: f64, fill: &str) -> String {
      format!(r#"<circle cx="{:.1}" cy="{:.1}" r="{:.1}" fill="{}"/>"#, cx, cy, r, fill)
  }

  fn svg_rect(x: f64, y: f64, w: f64, h: f64, fill: &str) -> String {
      format!(r#"<rect x="{:.1}" y="{:.1}" width="{:.1}" height="{:.1}" fill="{}"/>"#, x, y, w, h, fill)
  }

  fn svg_text(x: f64, y: f64, content: &str, size: f64) -> String {
      format!(r#"<text x="{:.1}" y="{:.1}" font-size="{:.0}">{}</text>"#, x, y, size, content)
  }

  // --- 棒グラフ生成器 ---

  fn generate_bar_chart(labels: &[&str], values: &[f64], width: f64, height: f64) -> String {
      let padding = 40.0;
      let chart_w = width - 2.0 * padding;
      let chart_h = height - 2.0 * padding;
      let max_val = values.iter().cloned().fold(f64::NEG_INFINITY, f64::max);
      let bar_width = chart_w / values.len() as f64 * 0.8;
      let gap = chart_w / values.len() as f64 * 0.2;

      let colors = ["#4e79a7", "#f28e2b", "#e15759", "#76b7b2", "#59a14f"];
      let mut elements = Vec::new();

      for (i, (&val, &label)) in values.iter().zip(labels.iter()).enumerate() {
          let bar_h = (val / max_val) * chart_h;
          let x = padding + i as f64 * (bar_width + gap);
          let y = padding + chart_h - bar_h;
          let color = colors[i % colors.len()];

          elements.push(svg_rect(x, y, bar_width, bar_h, color));
          elements.push(svg_text(x + bar_width / 2.0 - 10.0, height - 10.0, label, 12.0));
      }

      elements.join("\n  ")
  }

  fn main() {
      println!("=== SVG操作デモ ===\n");

      // 1. ベジェ曲線の評価
      println!("--- 3次ベジェ曲線 ---");
      let p0 = Point::new(0.0, 0.0);
      let p1 = Point::new(30.0, 100.0);
      let p2 = Point::new(70.0, 100.0);
      let p3 = Point::new(100.0, 0.0);

      println!("制御点: P0={}, P1={}, P2={}, P3={}", p0, p1, p2, p3);
      println!("曲線上の点:");
      for i in 0..=5 {
          let t = i as f64 / 5.0;
          let pt = cubic_bezier(p0, p1, p2, p3, t);
          println!("  t={:.1}: {}", t, pt);
      }

      let length = bezier_length(p0, p1, p2, p3, 100);
      println!("曲線の近似長さ: {:.2}", length);

      // 2. 折れ線グラフのパス
      println!("\n--- 折れ線グラフ SVGパス ---");
      let data = vec![10.0, 45.0, 28.0, 80.0, 55.0, 90.0, 35.0];
      let path = generate_line_chart(&data, 400.0, 200.0, 20.0);
      println!("データ: {:?}", data);
      println!("パス: {}", path);

      // 3. 滑らかな折れ線グラフのパス
      println!("\n--- 滑らかな折れ線グラフ（ベジェ） ---");
      let smooth_path = generate_smooth_line_chart(&data, 400.0, 200.0, 20.0);
      println!("滑らかなパス: {}", smooth_path);

      // 4. 棒グラフ
      println!("\n--- 棒グラフ要素 ---");
      let labels = vec!["Rust", "JS", "Go", "Py"];
      let values = vec![95.0, 70.0, 80.0, 60.0];
      let bars = generate_bar_chart(&labels, &values, 400.0, 250.0);
      println!("{}", bars);

      // 5. 完全なSVGドキュメント
      println!("\n--- 完全なSVGドキュメント ---");
      let svg = format!(
          r#"<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 400 200">
    <path d="{}" fill="none" stroke="#4e79a7" stroke-width="2"/>
    {}
    {}
  </svg>"#,
          path,
          svg_circle(20.0, 180.0, 4.0, "#e15759"),
          svg_circle(380.0, 42.5, 4.0, "#e15759"),
      );
      println!("{}", svg);
  }
expected_output: |
  === SVG操作デモ ===

  --- 3次ベジェ曲線 ---
  制御点: P0=(0.0, 0.0), P1=(30.0, 100.0), P2=(70.0, 100.0), P3=(100.0, 0.0)
  曲線上の点:
    t=0.0: (0.0, 0.0)
    t=0.2: (19.0, 48.0)
    t=0.4: (39.5, 72.0)
    t=0.6: (60.5, 72.0)
    t=0.8: (81.0, 48.0)
    t=1.0: (100.0, 0.0)
  曲線の近似長さ: 189.34

  --- 折れ線グラフ SVGパス ---
  データ: [10.0, 45.0, 28.0, 80.0, 55.0, 90.0, 35.0]
  パス: M 20.0 180.0 L 80.0 110.0 L 140.0 144.0 L 200.0 40.0 L 260.0 90.0 L 320.0 20.0 L 380.0 130.0

  --- 滑らかな折れ線グラフ（ベジェ） ---
  滑らかなパス: M 20.0 180.0 C 30.0 168.3, 60.0 116.0, 80.0 110.0 C 100.0 104.0, 120.0 155.7, 140.0 144.0 C 160.0 132.3, 180.0 49.0, 200.0 40.0 C 220.0 31.0, 240.0 93.3, 260.0 90.0 C 280.0 86.7, 300.0 13.3, 320.0 20.0 C 340.0 26.7, 370.0 111.7, 380.0 130.0

  --- 棒グラフ要素 ---
  <rect x="40.0" y="40.0" width="64.0" height="170.0" fill="#4e79a7"/>
  <text x="62.0" y="240.0" font-size="12">Rust</text>
  <rect x="120.0" y="84.7" width="64.0" height="125.3" fill="#f28e2b"/>
  <text x="142.0" y="240.0" font-size="12">JS</text>
  <rect x="200.0" y="66.8" width="64.0" height="143.2" fill="#e15759"/>
  <text x="222.0" y="240.0" font-size="12">Go</text>
  <rect x="280.0" y="102.6" width="64.0" height="107.4" fill="#76b7b2"/>
  <text x="302.0" y="240.0" font-size="12">Py</text>

  --- 完全なSVGドキュメント ---
  <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 400 200">
    <path d="M 20.0 180.0 L 80.0 110.0 L 140.0 144.0 L 200.0 40.0 L 260.0 90.0 L 320.0 20.0 L 380.0 130.0" fill="none" stroke="#4e79a7" stroke-width="2"/>
    <circle cx="20.0" cy="180.0" r="4.0" fill="#e15759"/>
    <circle cx="380.0" cy="42.5" r="4.0" fill="#e15759"/>
  </svg>
---

## なぜWasmからSVGを生成するのか?

SVG（Scalable Vector Graphics）はWebネイティブのベクター形式です。Rust/Wasmから生成することで、両方の長所を活かせます：

```
  ┌───────────────────────────────────────────────────┐
  │  Wasm (Rust)                                      │
  │                                                   │
  │  ● 計算負荷の高いパス生成                           │
  │  ● ベジェ曲線の数学                                │
  │  ● データ → チャート変換                            │
  │  ● SVGパスの最適化                                 │
  │  ● 座標変換                                       │
  │                                                   │
  │  出力: SVG文字列またはパスデータ                     │
  └─────────────────────┬─────────────────────────────┘
                        │
                        ▼
  ┌───────────────────────────────────────────────────┐
  │  ブラウザ (JavaScript + DOM)                       │
  │                                                   │
  │  ● SVGをDOMに挿入                                 │
  │  ● マウス/タッチイベントの処理                       │
  │  ● CSSスタイリングとアニメーション                   │
  │  ● アクセシビリティ（ARIAラベル）                    │
  │  ● レスポンシブレイアウト                            │
  └───────────────────────────────────────────────────┘
```

| アプローチ            | 長所                              | 短所                             |
|---------------------|---------------------------------|--------------------------------|
| JSチャートライブラリ    | 簡単なAPI、多くの選択肢            | 重いバンドル、大データで遅い        |
| Canvas (Wasm)       | ピクセル単位の精度、非常に高速      | スケーラブルでない、DOMイベントなし  |
| WasmからのSVG        | スケーラブル、アクセシブル、高速な計算| 文字列構築のオーバーヘッド          |
| WasmからのWebGL      | 最速の描画                        | 複雑、テキスト/アクセシビリティなし  |

## SVGパスコマンドリファレンス

SVGパスはコマンドのミニ言語を使用します：

```
  コマンド  パラメータ         説明
  ──────── ──────────────── ──────────────────────────────────
  M x y    移動             新しいサブパスを開始
  L x y    線を引く          直線を描画
  H x      水平線           水平方向のショートカット
  V y      垂直線           垂直方向のショートカット
  Q cx cy x y  2次ベジェ    制御点1つのベジェ曲線
  C c1x c1y c2x c2y x y    3次ベジェ（制御点2つ）
  A rx ry rot large sweep x y  楕円弧
  Z        パスを閉じる      始点に戻る線

  小文字 = 相対座標 (dx, dy)
  大文字 = 絶対座標 (x, y)
```

```
  3次ベジェ: C c1x c1y, c2x c2y, x y

       cp1●
          ╲
           ╲        ●cp2
  始点●     ╲      ╱
       ╲    ╲    ╱
        ╲    ╲  ╱
         ╲    ╲╱
          ╲   ╱
           ╲ ╱
            ●終点
```

## ベジェ曲線の数学

### 2次ベジェ（3点）

```
  B(t) = (1-t)^2 * P0 + 2(1-t)t * P1 + t^2 * P2

          P1（制御点）
          ●
         ╱ ╲
        ╱   ╲
  P0 ●╱     ╲● P2
     (始点)   (終点)

  t=0.0 → P0上
  t=0.5 → P1付近（しかしP1上ではない!）
  t=1.0 → P2上
```

### 3次ベジェ（4点）

```
  B(t) = (1-t)^3 * P0 + 3(1-t)^2 * t * P1
       + 3(1-t) * t^2 * P2 + t^3 * P3

  P1 ●─────● P2
     │       │
     │  曲線 │
  P0 ●       ● P3

  性質:
  ● 常にP0とP3を通過する
  ● P0での接線はP1方向を向く
  ● P3での接線はP2方向を向く
  ● 曲線は制御点の凸包に含まれる
```

### ド・カステリョのアルゴリズム

ベジェ曲線を評価する最も数値的に安定した方法です：

```
  レベル0:  P0          P1          P2          P3
              \        / \        / \        /
  レベル1:   A=lerp    B=lerp    C=lerp
                \      / \      /
  レベル2:     D=lerp   E=lerp
                  \    /
  レベル3:       結果=lerp(D,E,t)

  各レベル: new_point = (1-t) * left + t * right
```

## チャートライブラリの構築

### データポイントとグリッド付き折れ線グラフ

```rust
fn generate_chart_svg(
    data: &[(f64, f64)],
    width: f64,
    height: f64,
) -> String {
    let mut svg = format!(
        r#"<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 {} {}">"#,
        width, height
    );

    // 背景
    svg.push_str(&format!(
        r#"<rect width="{}" height="{}" fill="#fafafa"/>"#,
        width, height
    ));

    // グリッド線
    for i in 0..=4 {
        let y = 20.0 + (height - 40.0) * i as f64 / 4.0;
        svg.push_str(&format!(
            r#"<line x1="20" y1="{:.0}" x2="{:.0}" y2="{:.0}" stroke="#eee"/>"#,
            y, width - 20.0, y
        ));
    }

    // データパス
    let path = data_to_path(data, width, height, 20.0);
    svg.push_str(&format!(
        r#"<path d="{}" fill="none" stroke="#4e79a7" stroke-width="2"/>"#,
        path
    ));

    // データポイント
    for (x, y) in normalize_points(data, width, height, 20.0) {
        svg.push_str(&format!(
            r#"<circle cx="{:.1}" cy="{:.1}" r="3" fill="#4e79a7"/>"#,
            x, y
        ));
    }

    svg.push_str("</svg>");
    svg
}
```

### 円弧パスによる円グラフ

```rust
fn pie_chart(values: &[f64], colors: &[&str], radius: f64) -> String {
    let total: f64 = values.iter().sum();
    let cx = radius + 10.0;
    let cy = radius + 10.0;
    let mut angle = -std::f64::consts::FRAC_PI_2; // 上部から開始

    let mut paths = Vec::new();
    for (i, &val) in values.iter().enumerate() {
        let sweep = 2.0 * std::f64::consts::PI * val / total;
        let x1 = cx + radius * angle.cos();
        let y1 = cy + radius * angle.sin();
        let x2 = cx + radius * (angle + sweep).cos();
        let y2 = cy + radius * (angle + sweep).sin();
        let large_arc = if sweep > std::f64::consts::PI { 1 } else { 0 };

        paths.push(format!(
            r#"<path d="M {:.1} {:.1} L {:.1} {:.1} A {:.1} {:.1} 0 {} 1 {:.1} {:.1} Z" fill="{}"/>"#,
            cx, cy, x1, y1, radius, radius, large_arc, x2, y2, colors[i % colors.len()]
        ));
        angle += sweep;
    }
    paths.join("\n")
}
```

## SVGの最適化

生成されたSVGは大きくなることがあります。一般的な最適化手法を示します：

```
  最適化前                      最適化後
  ────────────────────        ────────────────────
  M 100.000000 200.000000     M100 200
  L 150.000000 180.000000     L150 180 200 160
  L 200.000000 160.000000     250 170 300 190Z
  L 250.000000 170.000000
  L 300.000000 190.000000
  Z

  削減量: 約40%小さく
```

| 最適化手法                            | サイズ削減    | 実装方法                  |
|--------------------------------------|-------------|--------------------------|
| 不要な小数の削除                       | 10-30%      | 小数第1位に丸める           |
| 相対座標（小文字）の使用                | 5-15%       | 差分エンコーディング         |
| 連続する同種コマンドの統合              | 5-10%       | "L x y"の連結             |
| 冗長な空白の除去                       | 5-10%       | ミニフィケーション          |
| パスの簡略化（Douglas-Peucker法）      | 10-50%      | 点の削減                  |

### パス簡略化（Douglas-Peuckerアルゴリズム）

```rust
fn simplify_path(points: &[Point], epsilon: f64) -> Vec<Point> {
    if points.len() < 3 { return points.to_vec(); }

    // 直線（始点→終点）から最大距離の点を見つける
    let (max_dist, max_idx) = points[1..points.len()-1].iter().enumerate()
        .map(|(i, p)| (point_line_distance(p, &points[0], points.last().unwrap()), i + 1))
        .max_by(|a, b| a.0.partial_cmp(&b.0).unwrap())
        .unwrap();

    if max_dist > epsilon {
        let mut left = simplify_path(&points[..=max_idx], epsilon);
        let right = simplify_path(&points[max_idx..], epsilon);
        left.pop(); // 重複点を除去
        left.extend(right);
        left
    } else {
        vec![points[0], *points.last().unwrap()]
    }
}
```

## 動的ビジュアライゼーション

Wasmからのリアルタイムなシーンの更新パターン：

```javascript
import { generate_chart_path } from './pkg/my_crate';

function animate() {
    const data = getLatestData(); // 例: WebSocketから
    const pathData = generate_chart_path(
        new Float64Array(data),
        canvas.width,
        canvas.height
    );

    // 既存のSVGパスを更新（DOM再作成なし!）
    pathElement.setAttribute('d', pathData);
    requestAnimationFrame(animate);
}
```

```
  データフロー:

  WebSocket ──→ JavaScript ──→ Wasm（パス計算） ──→ SVG DOM更新
                (データ収集)    (generate_chart_path) (setAttribute)
                                                     60 fps
```

SVG全体を再作成するよりも高速な理由：
1. `d`属性のみが変わり、DOM構造は変わらない
2. Wasmがパス座標をマイクロ秒単位で計算
3. ブラウザのSVGレンダラーがアンチエイリアシングとスケーリングを処理

## まとめ

WasmからのSVG生成により、Rustの計算能力とブラウザのネイティブベクターレンダラーを組み合わせることができます。計算負荷の高い部分（ベジェ曲線、座標変換、パス生成）にはWasmを使い、描画、イベント処理、スタイリングはブラウザに任せましょう。SVGパスのミニ言語はコンパクトで効率的であり、Douglas-Peucker法などの簡略化手法により出力を小さく保てます。リアルタイムビジュアライゼーションでは、DOM要素を再作成するのではなく、パスの`d`属性を直接更新しましょう。
