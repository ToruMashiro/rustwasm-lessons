---
title: "3D Graphics with WebGL"
slug: webgl-3d
difficulty: advanced
tags: [graphics]
order: 33
description: Build 3D graphics with WebGL from Rust — the rendering pipeline, vertex and fragment shaders, matrix math for rotation and projection, and rendering a spinning cube with camera and lighting.
starter_code: |
  use std::f64::consts::PI;

  fn main() {
      println!("=== 3D Graphics Math for WebGL ===\n");

      // --- 4x4 Matrix type ---
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

      // --- Rotation matrices ---
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

      // --- Translation matrix ---
      fn translate(tx: f64, ty: f64, tz: f64) -> Mat4 {
          [
              [1.0, 0.0, 0.0,  tx],
              [0.0, 1.0, 0.0,  ty],
              [0.0, 0.0, 1.0,  tz],
              [0.0, 0.0, 0.0, 1.0],
          ]
      }

      // --- Scale matrix ---
      fn scale(sx: f64, sy: f64, sz: f64) -> Mat4 {
          [
              [ sx, 0.0, 0.0, 0.0],
              [0.0,  sy, 0.0, 0.0],
              [0.0, 0.0,  sz, 0.0],
              [0.0, 0.0, 0.0, 1.0],
          ]
      }

      // --- Perspective projection ---
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

      // --- LookAt (view matrix) ---
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

      // --- Transform a 3D point ---
      fn transform_point(m: &Mat4, p: [f64; 3]) -> [f64; 3] {
          let w = m[3][0]*p[0] + m[3][1]*p[1] + m[3][2]*p[2] + m[3][3];
          [
              (m[0][0]*p[0] + m[0][1]*p[1] + m[0][2]*p[2] + m[0][3]) / w,
              (m[1][0]*p[0] + m[1][1]*p[1] + m[1][2]*p[2] + m[1][3]) / w,
              (m[2][0]*p[0] + m[2][1]*p[1] + m[2][2]*p[2] + m[2][3]) / w,
          ]
      }

      // === Demo ===

      println!("--- Identity Matrix ---");
      print_matrix("I", &identity());

      println!("\n--- Rotation Matrices ---");
      let angle = PI / 4.0; // 45 degrees
      println!("  Angle: {:.1} degrees ({:.4} rad)", 45.0, angle);
      print_matrix("Rotate_Y(45°)", &rotate_y(angle));

      println!("\n--- Perspective Projection ---");
      let fov = 60.0_f64.to_radians();
      let aspect = 16.0 / 9.0;
      let proj = perspective(fov, aspect, 0.1, 100.0);
      println!("  FOV: 60°, Aspect: 16:9, Near: 0.1, Far: 100.0");
      print_matrix("Projection", &proj);

      println!("\n--- Camera (LookAt) ---");
      let eye = [0.0, 2.0, 5.0];
      let target = [0.0, 0.0, 0.0];
      let up = [0.0, 1.0, 0.0];
      let view = look_at(eye, target, up);
      println!("  Eye: {:?}", eye);
      println!("  Target: {:?}", target);
      print_matrix("View", &view);

      // --- Cube vertices ---
      println!("\n--- Cube Vertex Transformation ---");
      let cube_verts: [[f64; 3]; 8] = [
          [-1.0, -1.0, -1.0], [ 1.0, -1.0, -1.0],
          [ 1.0,  1.0, -1.0], [-1.0,  1.0, -1.0],
          [-1.0, -1.0,  1.0], [ 1.0, -1.0,  1.0],
          [ 1.0,  1.0,  1.0], [-1.0,  1.0,  1.0],
      ];

      // Model transform: rotate 30° on Y, then translate back
      let model = multiply(&translate(0.0, 0.0, -3.0), &rotate_y(30.0_f64.to_radians()));
      let mvp = multiply(&proj, &multiply(&view, &model));

      println!("  Model: rotate_y(30°) + translate(0,0,-3)");
      println!("  MVP = Projection * View * Model\n");

      println!("  {:>5}  {:>10} {:>10} {:>10}  ->  {:>8} {:>8} {:>8}",
          "Vert", "X", "Y", "Z", "X'", "Y'", "Z'");
      println!("  {}", "-".repeat(70));

      for (i, v) in cube_verts.iter().enumerate() {
          let t = transform_point(&mvp, *v);
          println!("  {:>5}  {:>10.2} {:>10.2} {:>10.2}  ->  {:>8.4} {:>8.4} {:>8.4}",
              i, v[0], v[1], v[2], t[0], t[1], t[2]);
      }

      // --- Lighting calculation ---
      println!("\n--- Diffuse Lighting ---");
      let light_dir = normalize([0.5, 1.0, 0.3]);
      let normals: [[f64; 3]; 3] = [
          [0.0, 1.0, 0.0],   // top face
          [0.0, 0.0, 1.0],   // front face
          [1.0, 0.0, 0.0],   // right face
      ];
      let face_names = ["Top", "Front", "Right"];

      println!("  Light direction: [{:.2}, {:.2}, {:.2}]",
          light_dir[0], light_dir[1], light_dir[2]);
      println!();

      for (i, normal) in normals.iter().enumerate() {
          let intensity = dot(&light_dir, normal).max(0.0);
          let ambient = 0.1;
          let final_light = ambient + (1.0 - ambient) * intensity;
          let gray = (final_light * 255.0) as u8;
          println!("  {} face: normal={:?}, intensity={:.3}, brightness={}/255",
              face_names[i], normal, final_light, gray);
      }
  }
expected_output: |
  === 3D Graphics Math for WebGL ===

  --- Identity Matrix ---
    I:
      [  1.0000,   0.0000,   0.0000,   0.0000]
      [  0.0000,   1.0000,   0.0000,   0.0000]
      [  0.0000,   0.0000,   1.0000,   0.0000]
      [  0.0000,   0.0000,   0.0000,   1.0000]

  --- Rotation Matrices ---
    Angle: 45.0 degrees (0.7854 rad)
    Rotate_Y(45°):
      [  0.7071,   0.0000,   0.7071,   0.0000]
      [  0.0000,   1.0000,   0.0000,   0.0000]
      [ -0.7071,   0.0000,   0.7071,   0.0000]
      [  0.0000,   0.0000,   0.0000,   1.0000]

  --- Perspective Projection ---
    FOV: 60°, Aspect: 16:9, Near: 0.1, Far: 100.0
    Projection:
      [  0.9742,   0.0000,   0.0000,   0.0000]
      [  0.0000,   1.7321,   0.0000,   0.0000]
      [  0.0000,   0.0000,  -1.0020,  -0.2002]
      [  0.0000,   0.0000,  -1.0000,   0.0000]

  --- Camera (LookAt) ---
    Eye: [0.0, 2.0, 5.0]
    Target: [0.0, 0.0, 0.0]
    View:
      [  1.0000,   0.0000,   0.0000,  -0.0000]
      [  0.0000,   0.9285,  -0.3714,  -0.0000]
      [  0.0000,   0.3714,   0.9285,  -5.3852]
      [  0.0000,   0.0000,   0.0000,   1.0000]

  --- Cube Vertex Transformation ---
    Model: rotate_y(30°) + translate(0,0,-3)
    MVP = Projection * View * Model

     Vert           X          Y          Z  ->        X'       Y'       Z'
    ----------------------------------------------------------------------
        0      -1.00      -1.00      -1.00  ->  -0.1165  -0.3197   0.9929
        1       1.00      -1.00      -1.00  ->   0.1356  -0.3060   0.9854
        2       1.00       1.00      -1.00  ->   0.1194   0.2456   0.9854
        3      -1.00       1.00      -1.00  ->  -0.1026   0.2353   0.9929
        4      -1.00      -1.00       1.00  ->  -0.0888  -0.3583   1.0002
        5       1.00      -1.00       1.00  ->   0.1741  -0.3416   0.9918
        6       1.00       1.00       1.00  ->   0.1532   0.2173   0.9918
        7      -1.00       1.00       1.00  ->  -0.0782   0.2090   1.0002

  --- Diffuse Lighting ---
    Light direction: [0.43, 0.86, 0.26]

    Top face: normal=[0.0, 1.0, 0.0], intensity=0.871, brightness=222/255
    Front face: normal=[0.0, 0.0, 1.0], intensity=0.334, brightness=85/255
    Right face: normal=[1.0, 0.0, 0.0], intensity=0.488, brightness=124/255
---

## The WebGL Rendering Pipeline

WebGL is a low-level graphics API that maps almost 1:1 to OpenGL ES 2.0. From Rust/Wasm, you control every stage of the rendering pipeline:

```
┌────────────────────────────────────────────────────────────────┐
│                    WebGL Rendering Pipeline                     │
│                                                                │
│  ┌──────────┐   ┌──────────────┐   ┌────────────────────────┐ │
│  │ Vertex   │──►│ Vertex       │──►│ Primitive Assembly     │ │
│  │ Buffer   │   │ Shader       │   │ (triangles/lines)      │ │
│  │ (data)   │   │ (per vertex) │   │                        │ │
│  └──────────┘   └──────────────┘   └───────────┬────────────┘ │
│                                                 │              │
│                                                 ▼              │
│  ┌──────────┐   ┌──────────────┐   ┌────────────────────────┐ │
│  │ Frame    │◄──│ Fragment     │◄──│ Rasterization          │ │
│  │ Buffer   │   │ Shader       │   │ (triangles → pixels)   │ │
│  │ (screen) │   │ (per pixel)  │   │                        │ │
│  └──────────┘   └──────────────┘   └────────────────────────┘ │
└────────────────────────────────────────────────────────────────┘
```

## Setting Up WebGL from Rust

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

## Shaders

Shaders are small programs that run on the GPU. WebGL requires two types:

### Vertex Shader

Runs once per vertex. Transforms 3D coordinates to screen coordinates:

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
    vNormal = mat3(uModel) * aNormal;  // transform normal
    gl_Position = uProjection * uView * worldPos;
}
```

### Fragment Shader

Runs once per pixel. Determines the final color:

```glsl
precision mediump float;

varying vec3 vNormal;
varying vec3 vFragPos;

uniform vec3 uLightDir;
uniform vec3 uLightColor;
uniform vec3 uObjectColor;

void main() {
    // Ambient
    float ambient = 0.1;

    // Diffuse
    vec3 norm = normalize(vNormal);
    float diff = max(dot(norm, uLightDir), 0.0);

    vec3 result = (ambient + diff) * uLightColor * uObjectColor;
    gl_FragColor = vec4(result, 1.0);
}
```

### Compiling Shaders from Rust

```rust
fn compile_shader(gl: &GL, shader_type: u32, source: &str) -> Result<WebGlShader, String> {
    let shader = gl.create_shader(shader_type)
        .ok_or("Unable to create shader")?;
    gl.shader_source(&shader, source);
    gl.compile_shader(&shader);

    if gl.get_shader_parameter(&shader, GL::COMPILE_STATUS).as_bool().unwrap_or(false) {
        Ok(shader)
    } else {
        Err(gl.get_shader_info_log(&shader).unwrap_or_default())
    }
}

fn link_program(gl: &GL, vert: &WebGlShader, frag: &WebGlShader) -> Result<WebGlProgram, String> {
    let program = gl.create_program().ok_or("Unable to create program")?;
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

## Matrix Math: The Foundation of 3D

Every object in a 3D scene goes through three matrix transformations:

```
                   Model          View          Projection
  Local Space ──────────► World ─────────► Camera ──────────► Screen
  (object)        Space         Space          (NDC)

  MVP = Projection * View * Model
```

| Matrix | Purpose | Example |
|--------|---------|---------|
| Model | Position/rotate/scale the object | Rotate cube 45 degrees |
| View | Position the camera | Camera at (0, 2, 5) looking at origin |
| Projection | 3D → 2D (perspective) | 60 degree field of view |

### Rotation Matrices

Rotation around each axis uses sine and cosine:

```
Rotate around Y-axis by θ:

┌  cos θ   0   sin θ   0 ┐
│    0      1     0     0 │
│ -sin θ   0   cos θ   0 │
└    0      0     0     1 ┘
```

### Perspective Projection

The perspective matrix creates the illusion of depth — objects further away appear smaller:

```
Frustum (viewing volume):

        Near plane
        ┌───────┐
       /│       │\
      / │       │ \
     /  │       │  \        Far plane
    /   │       │   \    ┌─────────────┐
   / FOV│       │    \   │             │
  /     │ eye   │     \  │             │
 /      │  *    │      \ │             │
 \      │       │      / │             │
  \     │       │     /  │             │
   \    │       │    /   │             │
    \   └───────┘   /   └─────────────┘
     \             /
      near=0.1    far=100.0
```

## Cube Geometry

A cube has 8 vertices, 6 faces, and 12 triangles (2 per face):

```rust
// Cube vertices (position + normal for each face vertex)
const CUBE_VERTICES: &[f32] = &[
    // Front face (z = 1)
    -1.0, -1.0,  1.0,   0.0,  0.0,  1.0,  // pos, normal
     1.0, -1.0,  1.0,   0.0,  0.0,  1.0,
     1.0,  1.0,  1.0,   0.0,  0.0,  1.0,
    -1.0,  1.0,  1.0,   0.0,  0.0,  1.0,
    // Back face (z = -1) ...
    // Top face (y = 1) ...
    // etc. for all 6 faces
];

const CUBE_INDICES: &[u16] = &[
    0,  1,  2,    0,  2,  3,   // front
    4,  5,  6,    4,  6,  7,   // back
    8,  9,  10,   8,  10, 11,  // top
    12, 13, 14,   12, 14, 15,  // bottom
    16, 17, 18,   16, 18, 19,  // right
    20, 21, 22,   20, 22, 23,  // left
];
```

## Uploading Data to the GPU

```rust
fn create_buffer(gl: &GL, data: &[f32]) -> web_sys::WebGlBuffer {
    let buffer = gl.create_buffer().unwrap();
    gl.bind_buffer(GL::ARRAY_BUFFER, Some(&buffer));

    // Convert f32 slice to raw bytes for WebGL
    unsafe {
        let view = js_sys::Float32Array::view(data);
        gl.buffer_data_with_array_buffer_view(
            GL::ARRAY_BUFFER, &view, GL::STATIC_DRAW,
        );
    }

    buffer
}
```

## The Render Loop

```rust
use std::cell::RefCell;
use std::rc::Rc;

fn start_render_loop(gl: GL, program: WebGlProgram) {
    let f = Rc::new(RefCell::new(None));
    let g = f.clone();
    let angle = Rc::new(RefCell::new(0.0f64));

    *g.borrow_mut() = Some(Closure::wrap(Box::new(move || {
        let mut a = angle.borrow_mut();
        *a += 0.01;  // rotate

        // Clear
        gl.clear(GL::COLOR_BUFFER_BIT | GL::DEPTH_BUFFER_BIT);

        // Update model matrix with new rotation
        let model = multiply(&rotate_y(*a), &rotate_x(*a * 0.7));
        let model_flat: Vec<f32> = model.iter().flatten().map(|&x| x as f32).collect();

        let loc = gl.get_uniform_location(&program, "uModel");
        gl.uniform_matrix4fv_with_f32_array(loc.as_ref(), false, &model_flat);

        // Draw
        gl.draw_elements_with_i32(GL::TRIANGLES, 36, GL::UNSIGNED_SHORT, 0);

        // Request next frame
        request_animation_frame(f.borrow().as_ref().unwrap());
    }) as Box<dyn FnMut()>));

    request_animation_frame(g.borrow().as_ref().unwrap());
}
```

## Diffuse Lighting

The simplest lighting model computes brightness based on the angle between the surface normal and the light direction:

```
Light ──────────►
          θ \    │ Normal
             \   │
              \  │
               \ │
    ────────────\│────────── Surface
                 *

intensity = max(dot(normal, light_dir), 0.0)
            = max(cos(θ), 0.0)

When θ = 0°  → intensity = 1.0 (fully lit)
When θ = 90° → intensity = 0.0 (edge-on, dark)
When θ > 90° → intensity = 0.0 (facing away)
```

Add ambient light to prevent completely black shadows:

```
final_color = (ambient + diffuse_intensity) * object_color * light_color
```

## Performance Tips for Wasm + WebGL

| Technique | Benefit |
|-----------|---------|
| Compute matrices in Rust | Faster than JS for complex math |
| Batch draw calls | Fewer GPU state changes |
| Use `Float32Array::view` | Zero-copy data transfer to GPU |
| Minimize uniform updates | Only update what changed per frame |
| Use index buffers | Reduces vertex duplication |
| Cull back faces | `gl.enable(GL::CULL_FACE)` — skip invisible triangles |

## Key Takeaways

1. **WebGL** is a low-level API — Rust/Wasm excels at the math-heavy parts (matrix math, physics, procedural generation)
2. **Vertex shaders** transform 3D points; **fragment shaders** color pixels — both run on the GPU
3. **MVP matrix** = Projection * View * Model — transforms local coordinates to screen coordinates
4. **Perspective projection** creates depth illusion; **LookAt** positions the camera
5. **Diffuse lighting** = `dot(normal, light_direction)` — simple but effective
6. Use `js_sys::Float32Array::view` for zero-copy data uploads from Wasm to WebGL buffers
7. The render loop uses `requestAnimationFrame` via closures — store the `Closure` to prevent it from being dropped
