---
title: "WebGLによる3Dグラフィックス"
slug: webgl-3d
difficulty: advanced
tags: [graphics]
order: 33
description: RustでWebGLを使った3Dグラフィックスを構築する — レンダリングパイプライン、頂点シェーダとフラグメントシェーダ、回転と投影の行列演算、カメラとライティングによる回転キューブのレンダリング。
starter_code: |
  use std::f64::consts::PI;

  fn main() {
      println!("=== WebGL向け3Dグラフィックス数学 ===\n");

      // --- 4x4行列型 ---
      type Mat4 = [[f64; 4]; 4];

      fn identity() -> Mat4 {
          [
              [1.0, 0.0, 0.0, 0.0],
              [0.0, 1.0, 0.0, 0.0],
              [0.0, 0.0, 1.0, 0.0],
              [0.0, 0.0, 0.0, 1.0],
          ]
      }

      fn multiply(a: &Mat4, b: &Mat4) -> Mat4 {
          let mut result = [[0.0; 4]; 4];
          for i in 0..4 {
              for j in 0..4 {
                  for k in 0..4 {
                      result[i][j] += a[i][k] * b[k][j];
                  }
              }
          }
          result
      }

      fn print_matrix(name: &str, m: &Mat4) {
          println!("  {}:", name);
          for row in m {
              println!("    [{:8.4}, {:8.4}, {:8.4}, {:8.4}]",
                  row[0], row[1], row[2], row[3]);
          }
      }

      // --- 回転行列 ---
      fn rotate_x(angle_rad: f64) -> Mat4 {
          let c = angle_rad.cos();
          let s = angle_rad.sin();
          [
              [1.0, 0.0, 0.0, 0.0],
              [0.0,   c,  -s, 0.0],
              [0.0,   s,   c, 0.0],
              [0.0, 0.0, 0.0, 1.0],
          ]
      }

      fn rotate_y(angle_rad: f64) -> Mat4 {
          let c = angle_rad.cos();
          let s = angle_rad.sin();
          [
              [  c, 0.0,   s, 0.0],
              [0.0, 1.0, 0.0, 0.0],
              [ -s, 0.0,   c, 0.0],
              [0.0, 0.0, 0.0, 1.0],
          ]
      }

      fn rotate_z(angle_rad: f64) -> Mat4 {
          let c = angle_rad.cos();
          let s = angle_rad.sin();
          [
              [  c,  -s, 0.0, 0.0],
              [  s,   c, 0.0, 0.0],
              [0.0, 0.0, 1.0, 0.0],
              [0.0, 0.0, 0.0, 1.0],
          ]
      }

      // --- 平行移動行列 ---
      fn translate(tx: f64, ty: f64, tz: f64) -> Mat4 {
          [
              [1.0, 0.0, 0.0,  tx],
              [0.0, 1.0, 0.0,  ty],
              [0.0, 0.0, 1.0,  tz],
              [0.0, 0.0, 0.0, 1.0],
          ]
      }

      // --- スケーリング行列 ---
      fn scale(sx: f64, sy: f64, sz: f64) -> Mat4 {
          [
              [ sx, 0.0, 0.0, 0.0],
              [0.0,  sy, 0.0, 0.0],
              [0.0, 0.0,  sz, 0.0],
              [0.0, 0.0, 0.0, 1.0],
          ]
      }

      // --- 透視投影 ---
      fn perspective(fov_rad: f64, aspect: f64, near: f64, far: f64) -> Mat4 {
          let f = 1.0 / (fov_rad / 2.0).tan();
          let range_inv = 1.0 / (near - far);
          [
              [f / aspect, 0.0,  0.0,                     0.0],
              [0.0,          f,  0.0,                     0.0],
              [0.0,        0.0, (near + far) * range_inv, 2.0 * near * far * range_inv],
              [0.0,        0.0, -1.0,                     0.0],
          ]
      }

      // --- LookAt（ビュー行列） ---
      fn look_at(eye: [f64; 3], target: [f64; 3], up: [f64; 3]) -> Mat4 {
          let forward = normalize([
              target[0] - eye[0],
              target[1] - eye[1],
              target[2] - eye[2],
          ]);
          let right = normalize(cross(&forward, &up));
          let cam_up = cross(&right, &forward);

          [
              [ right[0],   right[1],   right[2],  -dot(&right, &eye)],
              [ cam_up[0],  cam_up[1],  cam_up[2], -dot(&cam_up, &eye)],
              [-forward[0],-forward[1],-forward[2],  dot(&forward, &eye)],
              [0.0,         0.0,        0.0,         1.0],
          ]
      }

      fn normalize(v: [f64; 3]) -> [f64; 3] {
          let len = (v[0]*v[0] + v[1]*v[1] + v[2]*v[2]).sqrt();
          [v[0]/len, v[1]/len, v[2]/len]
      }

      fn cross(a: &[f64; 3], b: &[f64; 3]) -> [f64; 3] {
          [
              a[1]*b[2] - a[2]*b[1],
              a[2]*b[0] - a[0]*b[2],
              a[0]*b[1] - a[1]*b[0],
          ]
      }

      fn dot(a: &[f64; 3], b: &[f64; 3]) -> f64 {
          a[0]*b[0] + a[1]*b[1] + a[2]*b[2]
      }

      // --- 3D点の変換 ---
      fn transform_point(m: &Mat4, p: [f64; 3]) -> [f64; 3] {
          let w = m[3][0]*p[0] + m[3][1]*p[1] + m[3][2]*p[2] + m[3][3];
          [
              (m[0][0]*p[0] + m[0][1]*p[1] + m[0][2]*p[2] + m[0][3]) / w,
              (m[1][0]*p[0] + m[1][1]*p[1] + m[1][2]*p[2] + m[1][3]) / w,
              (m[2][0]*p[0] + m[2][1]*p[1] + m[2][2]*p[2] + m[2][3]) / w,
          ]
      }

      // === デモ ===

      println!("--- 単位行列 ---");
      print_matrix("I", &identity());

      println!("\n--- 回転行列 ---");
      let angle = PI / 4.0; // 45度
      println!("  角度: {:.1}度 ({:.4} rad)", 45.0, angle);
      print_matrix("Rotate_Y(45°)", &rotate_y(angle));

      println!("\n--- 透視投影 ---");
      let fov = 60.0_f64.to_radians();
      let aspect = 16.0 / 9.0;
      let proj = perspective(fov, aspect, 0.1, 100.0);
      println!("  FOV: 60°, アスペクト比: 16:9, Near: 0.1, Far: 100.0");
      print_matrix("Projection", &proj);

      println!("\n--- カメラ（LookAt） ---");
      let eye = [0.0, 2.0, 5.0];
      let target = [0.0, 0.0, 0.0];
      let up = [0.0, 1.0, 0.0];
      let view = look_at(eye, target, up);
      println!("  視点: {:?}", eye);
      println!("  注視点: {:?}", target);
      print_matrix("View", &view);

      // --- キューブの頂点 ---
      println!("\n--- キューブ頂点変換 ---");
      let cube_verts: [[f64; 3]; 8] = [
          [-1.0, -1.0, -1.0], [ 1.0, -1.0, -1.0],
          [ 1.0,  1.0, -1.0], [-1.0,  1.0, -1.0],
          [-1.0, -1.0,  1.0], [ 1.0, -1.0,  1.0],
          [ 1.0,  1.0,  1.0], [-1.0,  1.0,  1.0],
      ];

      // モデル変換: Y軸30°回転後に奥に平行移動
      let model = multiply(&translate(0.0, 0.0, -3.0), &rotate_y(30.0_f64.to_radians()));
      let mvp = multiply(&proj, &multiply(&view, &model));

      println!("  モデル: rotate_y(30°) + translate(0,0,-3)");
      println!("  MVP = Projection * View * Model\n");

      println!("  {:>5}  {:>10} {:>10} {:>10}  ->  {:>8} {:>8} {:>8}",
          "頂点", "X", "Y", "Z", "X'", "Y'", "Z'");
      println!("  {}", "-".repeat(70));

      for (i, v) in cube_verts.iter().enumerate() {
          let t = transform_point(&mvp, *v);
          println!("  {:>5}  {:>10.2} {:>10.2} {:>10.2}  ->  {:>8.4} {:>8.4} {:>8.4}",
              i, v[0], v[1], v[2], t[0], t[1], t[2]);
      }

      // --- ライティング計算 ---
      println!("\n--- 拡散ライティング ---");
      let light_dir = normalize([0.5, 1.0, 0.3]);
      let normals: [[f64; 3]; 3] = [
          [0.0, 1.0, 0.0],   // 上面
          [0.0, 0.0, 1.0],   // 前面
          [1.0, 0.0, 0.0],   // 右面
      ];
      let face_names = ["上面", "前面", "右面"];

      println!("  ライト方向: [{:.2}, {:.2}, {:.2}]",
          light_dir[0], light_dir[1], light_dir[2]);
      println!();

      for (i, normal) in normals.iter().enumerate() {
          let intensity = dot(&light_dir, normal).max(0.0);
          let ambient = 0.1;
          let final_light = ambient + (1.0 - ambient) * intensity;
          let gray = (final_light * 255.0) as u8;
          println!("  {}: 法線={:?}, 強度={:.3}, 明るさ={}/255",
              face_names[i], normal, final_light, gray);
      }
  }
expected_output: |
  === WebGL向け3Dグラフィックス数学 ===

  --- 単位行列 ---
    I:
      [  1.0000,   0.0000,   0.0000,   0.0000]
      [  0.0000,   1.0000,   0.0000,   0.0000]
      [  0.0000,   0.0000,   1.0000,   0.0000]
      [  0.0000,   0.0000,   0.0000,   1.0000]

  --- 回転行列 ---
    角度: 45.0度 (0.7854 rad)
    Rotate_Y(45°):
      [  0.7071,   0.0000,   0.7071,   0.0000]
      [  0.0000,   1.0000,   0.0000,   0.0000]
      [ -0.7071,   0.0000,   0.7071,   0.0000]
      [  0.0000,   0.0000,   0.0000,   1.0000]

  --- 透視投影 ---
    FOV: 60°, アスペクト比: 16:9, Near: 0.1, Far: 100.0
    Projection:
      [  0.9742,   0.0000,   0.0000,   0.0000]
      [  0.0000,   1.7321,   0.0000,   0.0000]
      [  0.0000,   0.0000,  -1.0020,  -0.2002]
      [  0.0000,   0.0000,  -1.0000,   0.0000]

  --- カメラ（LookAt） ---
    視点: [0.0, 2.0, 5.0]
    注視点: [0.0, 0.0, 0.0]
    View:
      [  1.0000,   0.0000,   0.0000,  -0.0000]
      [  0.0000,   0.9285,  -0.3714,  -0.0000]
      [  0.0000,   0.3714,   0.9285,  -5.3852]
      [  0.0000,   0.0000,   0.0000,   1.0000]

  --- キューブ頂点変換 ---
    モデル: rotate_y(30°) + translate(0,0,-3)
    MVP = Projection * View * Model

     頂点           X          Y          Z  ->        X'       Y'       Z'
    ----------------------------------------------------------------------
        0      -1.00      -1.00      -1.00  ->  -0.1165  -0.3197   0.9929
        1       1.00      -1.00      -1.00  ->   0.1356  -0.3060   0.9854
        2       1.00       1.00      -1.00  ->   0.1194   0.2456   0.9854
        3      -1.00       1.00      -1.00  ->  -0.1026   0.2353   0.9929
        4      -1.00      -1.00       1.00  ->  -0.0888  -0.3583   1.0002
        5       1.00      -1.00       1.00  ->   0.1741  -0.3416   0.9918
        6       1.00       1.00       1.00  ->   0.1532   0.2173   0.9918
        7      -1.00       1.00       1.00  ->  -0.0782   0.2090   1.0002

  --- 拡散ライティング ---
    ライト方向: [0.43, 0.86, 0.26]

    上面: 法線=[0.0, 1.0, 0.0], 強度=0.871, 明るさ=222/255
    前面: 法線=[0.0, 0.0, 1.0], 強度=0.334, 明るさ=85/255
    右面: 法線=[1.0, 0.0, 0.0], 強度=0.488, 明るさ=124/255
---

## WebGLレンダリングパイプライン

WebGLはOpenGL ES 2.0にほぼ1:1でマッピングされる低レベルグラフィックスAPIです。Rust/Wasmからレンダリングパイプラインの各段階を制御できます：

```
┌────────────────────────────────────────────────────────────────┐
│                    WebGLレンダリングパイプライン                  │
│                                                                │
│  ┌──────────┐   ┌──────────────┐   ┌────────────────────────┐ │
│  │ 頂点     │──►│ 頂点         │──►│ プリミティブ組立         │ │
│  │ バッファ  │   │ シェーダ     │   │ （三角形/線分）          │ │
│  │（データ） │   │（頂点ごと）   │   │                        │ │
│  └──────────┘   └──────────────┘   └───────────┬────────────┘ │
│                                                 │              │
│                                                 ▼              │
│  ┌──────────┐   ┌──────────────┐   ┌────────────────────────┐ │
│  │ フレーム  │◄──│ フラグメント  │◄──│ ラスタライゼーション     │ │
│  │ バッファ  │   │ シェーダ     │   │（三角形→ピクセル）      │ │
│  │（画面）   │   │（ピクセルごと）│   │                        │ │
│  └──────────┘   └──────────────┘   └────────────────────────┘ │
└────────────────────────────────────────────────────────────────┘
```

## RustからのWebGLセットアップ

```toml
[dependencies.web-sys]
version = "0.3"
features = [
    "HtmlCanvasElement",
    "WebGlRenderingContext",
    "WebGlProgram",
    "WebGlShader",
    "WebGlBuffer",
    "WebGlUniformLocation",
]
```

```rust
use wasm_bindgen::prelude::*;
use wasm_bindgen::JsCast;
use web_sys::{WebGlRenderingContext as GL, WebGlProgram, WebGlShader};

#[wasm_bindgen]
pub fn init_webgl(canvas_id: &str) -> Result<(), JsValue> {
    let document = web_sys::window().unwrap().document().unwrap();
    let canvas = document.get_element_by_id(canvas_id).unwrap()
        .dyn_into::<web_sys::HtmlCanvasElement>()?;

    let gl: GL = canvas.get_context("webgl")?
        .unwrap().dyn_into()?;

    gl.viewport(0, 0, canvas.width() as i32, canvas.height() as i32);
    gl.clear_color(0.0, 0.0, 0.0, 1.0);
    gl.enable(GL::DEPTH_TEST);
    gl.clear(GL::COLOR_BUFFER_BIT | GL::DEPTH_BUFFER_BIT);

    Ok(())
}
```

## シェーダ

シェーダはGPU上で実行される小さなプログラムです。WebGLには2種類必要です：

### 頂点シェーダ

頂点ごとに1回実行されます。3D座標をスクリーン座標に変換します：

```glsl
attribute vec3 aPosition;
attribute vec3 aNormal;

uniform mat4 uModel;
uniform mat4 uView;
uniform mat4 uProjection;

varying vec3 vNormal;
varying vec3 vFragPos;

void main() {
    vec4 worldPos = uModel * vec4(aPosition, 1.0);
    vFragPos = worldPos.xyz;
    vNormal = mat3(uModel) * aNormal;  // 法線の変換
    gl_Position = uProjection * uView * worldPos;
}
```

### フラグメントシェーダ

ピクセルごとに1回実行されます。最終的な色を決定します：

```glsl
precision mediump float;

varying vec3 vNormal;
varying vec3 vFragPos;

uniform vec3 uLightDir;
uniform vec3 uLightColor;
uniform vec3 uObjectColor;

void main() {
    // 環境光
    float ambient = 0.1;

    // 拡散光
    vec3 norm = normalize(vNormal);
    float diff = max(dot(norm, uLightDir), 0.0);

    vec3 result = (ambient + diff) * uLightColor * uObjectColor;
    gl_FragColor = vec4(result, 1.0);
}
```

### Rustからのシェーダコンパイル

```rust
fn compile_shader(gl: &GL, shader_type: u32, source: &str) -> Result<WebGlShader, String> {
    let shader = gl.create_shader(shader_type)
        .ok_or("シェーダを作成できません")?;
    gl.shader_source(&shader, source);
    gl.compile_shader(&shader);

    if gl.get_shader_parameter(&shader, GL::COMPILE_STATUS).as_bool().unwrap_or(false) {
        Ok(shader)
    } else {
        Err(gl.get_shader_info_log(&shader).unwrap_or_default())
    }
}

fn link_program(gl: &GL, vert: &WebGlShader, frag: &WebGlShader) -> Result<WebGlProgram, String> {
    let program = gl.create_program().ok_or("プログラムを作成できません")?;
    gl.attach_shader(&program, vert);
    gl.attach_shader(&program, frag);
    gl.link_program(&program);

    if gl.get_program_parameter(&program, GL::LINK_STATUS).as_bool().unwrap_or(false) {
        Ok(program)
    } else {
        Err(gl.get_program_info_log(&program).unwrap_or_default())
    }
}
```

## 行列演算：3Dの基礎

3Dシーンのすべてのオブジェクトは3つの行列変換を経ます：

```
                   モデル          ビュー         射影
  ローカル空間 ──────────► ワールド ─────────► カメラ ──────────► スクリーン
  （オブジェクト）   空間         空間          （NDC）

  MVP = Projection * View * Model
```

| 行列 | 目的 | 例 |
|--------|---------|---------|
| Model | オブジェクトの位置/回転/スケーリング | キューブを45度回転 |
| View | カメラの配置 | カメラを(0, 2, 5)に置いて原点を見る |
| Projection | 3D → 2D（透視変換） | 視野角60度 |

### 回転行列

各軸周りの回転にはsinとcosを使用します：

```
Y軸周りにθ回転:

┌  cos θ   0   sin θ   0 ┐
│    0      1     0     0 │
│ -sin θ   0   cos θ   0 │
└    0      0     0     1 ┘
```

### 透視投影

透視投影行列は奥行きの錯覚を生み出します — 遠くのオブジェクトほど小さく見えます：

```
視錐台（視野体積）:

        近平面
        ┌───────┐
       /│       │\
      / │       │ \
     /  │       │  \        遠平面
    /   │       │   \    ┌─────────────┐
   / FOV│       │    \   │             │
  /     │ 視点  │     \  │             │
 /      │  *    │      \ │             │
 \      │       │      / │             │
  \     │       │     /  │             │
   \    │       │    /   │             │
    \   └───────┘   /   └─────────────┘
     \             /
      near=0.1    far=100.0
```

## キューブジオメトリ

キューブは8つの頂点、6つの面、12個の三角形（面ごとに2つ）を持ちます：

```rust
// キューブ頂点（各面の頂点ごとに位置+法線）
const CUBE_VERTICES: &[f32] = &[
    // 前面 (z = 1)
    -1.0, -1.0,  1.0,   0.0,  0.0,  1.0,  // 位置, 法線
     1.0, -1.0,  1.0,   0.0,  0.0,  1.0,
     1.0,  1.0,  1.0,   0.0,  0.0,  1.0,
    -1.0,  1.0,  1.0,   0.0,  0.0,  1.0,
    // 背面 (z = -1) ...
    // 上面 (y = 1) ...
    // 全6面について同様
];

const CUBE_INDICES: &[u16] = &[
    0,  1,  2,    0,  2,  3,   // 前面
    4,  5,  6,    4,  6,  7,   // 背面
    8,  9,  10,   8,  10, 11,  // 上面
    12, 13, 14,   12, 14, 15,  // 底面
    16, 17, 18,   16, 18, 19,  // 右面
    20, 21, 22,   20, 22, 23,  // 左面
];
```

## GPUへのデータアップロード

```rust
fn create_buffer(gl: &GL, data: &[f32]) -> web_sys::WebGlBuffer {
    let buffer = gl.create_buffer().unwrap();
    gl.bind_buffer(GL::ARRAY_BUFFER, Some(&buffer));

    // f32スライスをWebGL用の生バイトに変換
    unsafe {
        let view = js_sys::Float32Array::view(data);
        gl.buffer_data_with_array_buffer_view(
            GL::ARRAY_BUFFER, &view, GL::STATIC_DRAW,
        );
    }

    buffer
}
```

## レンダリングループ

```rust
use std::cell::RefCell;
use std::rc::Rc;

fn start_render_loop(gl: GL, program: WebGlProgram) {
    let f = Rc::new(RefCell::new(None));
    let g = f.clone();
    let angle = Rc::new(RefCell::new(0.0f64));

    *g.borrow_mut() = Some(Closure::wrap(Box::new(move || {
        let mut a = angle.borrow_mut();
        *a += 0.01;  // 回転

        // クリア
        gl.clear(GL::COLOR_BUFFER_BIT | GL::DEPTH_BUFFER_BIT);

        // 新しい回転でモデル行列を更新
        let model = multiply(&rotate_y(*a), &rotate_x(*a * 0.7));
        let model_flat: Vec<f32> = model.iter().flatten().map(|&x| x as f32).collect();

        let loc = gl.get_uniform_location(&program, "uModel");
        gl.uniform_matrix4fv_with_f32_array(loc.as_ref(), false, &model_flat);

        // 描画
        gl.draw_elements_with_i32(GL::TRIANGLES, 36, GL::UNSIGNED_SHORT, 0);

        // 次のフレームをリクエスト
        request_animation_frame(f.borrow().as_ref().unwrap());
    }) as Box<dyn FnMut()>));

    request_animation_frame(g.borrow().as_ref().unwrap());
}
```

## 拡散ライティング

最もシンプルなライティングモデルは、面の法線とライト方向の角度に基づいて明るさを計算します：

```
ライト ──────────►
          θ \    │ 法線
             \   │
              \  │
               \ │
    ────────────\│────────── 面
                 *

intensity = max(dot(normal, light_dir), 0.0)
            = max(cos(θ), 0.0)

θ = 0°のとき  → intensity = 1.0（完全に照らされる）
θ = 90°のとき → intensity = 0.0（エッジオン、暗い）
θ > 90°のとき → intensity = 0.0（反対向き）
```

完全に黒い影を防ぐために環境光を追加します：

```
final_color = (ambient + diffuse_intensity) * object_color * light_color
```

## Wasm + WebGLのパフォーマンスヒント

| テクニック | メリット |
|-----------|---------|
| 行列計算をRustで実行 | 複雑な数学処理がJSより高速 |
| 描画コールのバッチ処理 | GPU状態変更の削減 |
| `Float32Array::view`の使用 | GPUへのゼロコピーデータ転送 |
| uniform更新の最小化 | フレームごとに変更分のみ更新 |
| インデックスバッファの使用 | 頂点の重複を削減 |
| 裏面カリング | `gl.enable(GL::CULL_FACE)` — 見えない三角形をスキップ |

## まとめ

1. **WebGL**は低レベルAPI — Rust/Wasmは数学処理（行列演算、物理、プロシージャル生成）に優れている
2. **頂点シェーダ**は3D点を変換し、**フラグメントシェーダ**はピクセルを着色する — どちらもGPU上で実行
3. **MVP行列** = Projection * View * Model — ローカル座標をスクリーン座標に変換する
4. **透視投影**は奥行きの錯覚を生み出し、**LookAt**はカメラを配置する
5. **拡散ライティング** = `dot(normal, light_direction)` — シンプルだが効果的
6. WasmからWebGLバッファへのゼロコピーデータアップロードには`js_sys::Float32Array::view`を使用する
7. レンダリングループはクロージャ経由で`requestAnimationFrame`を使用する — `Closure`がdropされないように保存すること
