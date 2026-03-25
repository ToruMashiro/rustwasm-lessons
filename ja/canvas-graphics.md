---
title: Canvas と 2Dグラフィックス
slug: canvas-graphics
difficulty: intermediate
tags: [graphics, dom]
order: 11
description: Rust/Wasmから制御するHTML Canvasを使って、図形の描画、スプライトのアニメーション、インタラクティブなグラフィックスを構築します。
starter_code: |
  use wasm_bindgen::prelude::*;
  use web_sys::{CanvasRenderingContext2d, HtmlCanvasElement, window};

  #[wasm_bindgen]
  pub struct Canvas {
      ctx: CanvasRenderingContext2d,
      width: f64,
      height: f64,
  }

  #[wasm_bindgen]
  impl Canvas {
      #[wasm_bindgen(constructor)]
      pub fn new(canvas_id: &str) -> Result<Canvas, JsValue> {
          let document = window().unwrap().document().unwrap();
          let canvas = document
              .get_element_by_id(canvas_id)
              .unwrap()
              .dyn_into::<HtmlCanvasElement>()?;

          let ctx = canvas
              .get_context("2d")?
              .unwrap()
              .dyn_into::<CanvasRenderingContext2d>()?;

          let width = canvas.width() as f64;
          let height = canvas.height() as f64;

          Ok(Canvas { ctx, width, height })
      }

      pub fn clear(&self) {
          self.ctx.clear_rect(0.0, 0.0, self.width, self.height);
      }

      pub fn draw_circle(&self, x: f64, y: f64, radius: f64, color: &str) {
          self.ctx.begin_path();
          self.ctx.arc(x, y, radius, 0.0, std::f64::consts::PI * 2.0).unwrap();
          self.ctx.set_fill_style_str(color);
          self.ctx.fill();
      }

      pub fn draw_rect(&self, x: f64, y: f64, w: f64, h: f64, color: &str) {
          self.ctx.set_fill_style_str(color);
          self.ctx.fill_rect(x, y, w, h);
      }
  }
expected_output: |
  Canvasを初期化: 800x600
  円を描画 (400, 300) r=50
  矩形を描画 (100, 100) 200x150
  フレームの描画に成功
---

## はじめに

HTML Canvasは、Rust/Wasmの最適なユースケースの一つです。Rustが計算処理（物理演算、ゲームロジック、プロシージャル生成）を担当し、Canvas APIがレンダリングを処理します。この分担により、重い処理部分でネイティブに近いパフォーマンスが得られます。

## セットアップ

```toml
[dependencies.web-sys]
version = "0.3"
features = [
    "Window",
    "Document",
    "HtmlCanvasElement",
    "CanvasRenderingContext2d",
]
```

## Canvasコンテキストの取得

```rust
use web_sys::{CanvasRenderingContext2d, HtmlCanvasElement};

fn get_context(canvas_id: &str) -> Result<CanvasRenderingContext2d, JsValue> {
    let document = web_sys::window().unwrap().document().unwrap();
    let canvas = document
        .get_element_by_id(canvas_id)
        .unwrap()
        .dyn_into::<HtmlCanvasElement>()?;

    canvas
        .get_context("2d")?
        .unwrap()
        .dyn_into::<CanvasRenderingContext2d>()
}
```

## 基本図形の描画

```rust
// 矩形
ctx.set_fill_style_str("#654ff0");
ctx.fill_rect(10.0, 10.0, 100.0, 50.0);

// 円
ctx.begin_path();
ctx.arc(200.0, 200.0, 40.0, 0.0, std::f64::consts::PI * 2.0)?;
ctx.set_fill_style_str("#ce422b");
ctx.fill();

// 線
ctx.begin_path();
ctx.move_to(0.0, 0.0);
ctx.line_to(300.0, 150.0);
ctx.set_stroke_style_str("#34d399");
ctx.set_line_width(2.0);
ctx.stroke();

// テキスト
ctx.set_font("24px Inter");
ctx.set_fill_style_str("#f1f5f9");
ctx.fill_text("Hello Wasm!", 50.0, 50.0)?;
```

## アニメーションループ

アニメーションはJavaScript側で`requestAnimationFrame`を使って実行し、各フレームでRustを呼び出します：

```js
import init, { Canvas } from './pkg/my_app.js';

async function run() {
    await init();
    const canvas = new Canvas("game-canvas");

    let x = 0;
    function frame() {
        canvas.clear();
        canvas.draw_circle(x, 300, 20, "#654ff0");
        x = (x + 2) % 800;
        requestAnimationFrame(frame);
    }
    requestAnimationFrame(frame);
}
run();
```

## マウス入力

```rust
use web_sys::MouseEvent;
use wasm_bindgen::closure::Closure;

pub fn setup_mouse(canvas: &HtmlCanvasElement) -> Result<(), JsValue> {
    let closure = Closure::wrap(Box::new(move |event: MouseEvent| {
        let x = event.offset_x() as f64;
        let y = event.offset_y() as f64;
        // (x, y) でのクリックを処理
    }) as Box<dyn FnMut(_)>);

    canvas.add_event_listener_with_callback(
        "click",
        closure.as_ref().unchecked_ref(),
    )?;
    closure.forget();
    Ok(())
}
```

## 画像の描画

```rust
use web_sys::HtmlImageElement;

pub fn draw_image(ctx: &CanvasRenderingContext2d, src: &str, x: f64, y: f64)
    -> Result<(), JsValue>
{
    let img = HtmlImageElement::new()?;
    img.set_src(src);

    let ctx = ctx.clone();
    let closure = Closure::once(move || {
        ctx.draw_image_with_html_image_element(&img, x, y).unwrap();
    });
    img.set_onload(Some(closure.as_ref().unchecked_ref()));
    closure.forget();
    Ok(())
}
```

## パフォーマンス比較：Canvas vs WebGL

| 特徴 | Canvas 2D | WebGL |
|------|----------|-------|
| セットアップの複雑さ | 低い | 高い |
| 描画呼び出し | 約1,000回/フレーム | 約10,000回/フレーム |
| シェーダー | なし | あり |
| 3D | なし | あり |
| 最適な用途 | 2Dゲーム、チャート、UI | 3D、大量パーティクルシステム |

ほとんどの2D Rust/Wasmプロジェクトでは、Canvas 2Dで十分であり、はるかにシンプルです。

## 試してみよう

スターターコードは、基本的な描画メソッドを持つCanvasラッパー構造体を示しています。実際のプロジェクトでは、JavaScriptのアニメーションループからこれらのメソッドを呼び出します。
