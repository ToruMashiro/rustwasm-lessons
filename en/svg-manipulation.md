---
title: "Wasm + SVG Manipulation"
slug: svg-manipulation
difficulty: intermediate
tags: [graphics]
order: 43
description: Generate and manipulate SVG graphics from Rust — compute Bezier curves, build chart paths, optimize SVG output, and create dynamic visualizations powered by Wasm.
starter_code: |
  use std::fmt;

  // Demonstrating SVG generation and Bezier math in plain Rust.
  // In a real Wasm project, you'd pass the SVG string to the DOM
  // via web-sys or return it to JavaScript for insertion.

  // --- 2D Point ---
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

  // --- SVG Path Builder ---
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

  // --- Bezier Curve Evaluation ---

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

  // --- Chart generators ---

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

      // Catmull-Rom to cubic Bezier conversion for smooth curves
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

  // --- SVG element builder ---

  fn svg_circle(cx: f64, cy: f64, r: f64, fill: &str) -> String {
      format!(r#"<circle cx="{:.1}" cy="{:.1}" r="{:.1}" fill="{}"/>"#, cx, cy, r, fill)
  }

  fn svg_rect(x: f64, y: f64, w: f64, h: f64, fill: &str) -> String {
      format!(r#"<rect x="{:.1}" y="{:.1}" width="{:.1}" height="{:.1}" fill="{}"/>"#, x, y, w, h, fill)
  }

  fn svg_text(x: f64, y: f64, content: &str, size: f64) -> String {
      format!(r#"<text x="{:.1}" y="{:.1}" font-size="{:.0}">{}</text>"#, x, y, size, content)
  }

  // --- Bar chart generator ---

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
      println!("=== SVG Manipulation Demo ===\n");

      // 1. Bezier curve evaluation
      println!("--- Cubic Bezier Curve ---");
      let p0 = Point::new(0.0, 0.0);
      let p1 = Point::new(30.0, 100.0);
      let p2 = Point::new(70.0, 100.0);
      let p3 = Point::new(100.0, 0.0);

      println!("Control points: P0={}, P1={}, P2={}, P3={}", p0, p1, p2, p3);
      println!("Points along curve:");
      for i in 0..=5 {
          let t = i as f64 / 5.0;
          let pt = cubic_bezier(p0, p1, p2, p3, t);
          println!("  t={:.1}: {}", t, pt);
      }

      let length = bezier_length(p0, p1, p2, p3, 100);
      println!("Approximate curve length: {:.2}", length);

      // 2. Line chart path
      println!("\n--- Line Chart SVG Path ---");
      let data = vec![10.0, 45.0, 28.0, 80.0, 55.0, 90.0, 35.0];
      let path = generate_line_chart(&data, 400.0, 200.0, 20.0);
      println!("Data: {:?}", data);
      println!("Path: {}", path);

      // 3. Smooth line chart path
      println!("\n--- Smooth Line Chart (Bezier) ---");
      let smooth_path = generate_smooth_line_chart(&data, 400.0, 200.0, 20.0);
      println!("Smooth path: {}", smooth_path);

      // 4. Bar chart
      println!("\n--- Bar Chart Elements ---");
      let labels = vec!["Rust", "JS", "Go", "Py"];
      let values = vec![95.0, 70.0, 80.0, 60.0];
      let bars = generate_bar_chart(&labels, &values, 400.0, 250.0);
      println!("{}", bars);

      // 5. Complete SVG document
      println!("\n--- Complete SVG Document ---");
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
  === SVG Manipulation Demo ===

  --- Cubic Bezier Curve ---
  Control points: P0=(0.0, 0.0), P1=(30.0, 100.0), P2=(70.0, 100.0), P3=(100.0, 0.0)
  Points along curve:
    t=0.0: (0.0, 0.0)
    t=0.2: (19.0, 48.0)
    t=0.4: (39.5, 72.0)
    t=0.6: (60.5, 72.0)
    t=0.8: (81.0, 48.0)
    t=1.0: (100.0, 0.0)
  Approximate curve length: 189.34

  --- Line Chart SVG Path ---
  Data: [10.0, 45.0, 28.0, 80.0, 55.0, 90.0, 35.0]
  Path: M 20.0 180.0 L 80.0 110.0 L 140.0 144.0 L 200.0 40.0 L 260.0 90.0 L 320.0 20.0 L 380.0 130.0

  --- Smooth Line Chart (Bezier) ---
  Smooth path: M 20.0 180.0 C 30.0 168.3, 60.0 116.0, 80.0 110.0 C 100.0 104.0, 120.0 155.7, 140.0 144.0 C 160.0 132.3, 180.0 49.0, 200.0 40.0 C 220.0 31.0, 240.0 93.3, 260.0 90.0 C 280.0 86.7, 300.0 13.3, 320.0 20.0 C 340.0 26.7, 370.0 111.7, 380.0 130.0

  --- Bar Chart Elements ---
  <rect x="40.0" y="40.0" width="64.0" height="170.0" fill="#4e79a7"/>
  <text x="62.0" y="240.0" font-size="12">Rust</text>
  <rect x="120.0" y="84.7" width="64.0" height="125.3" fill="#f28e2b"/>
  <text x="142.0" y="240.0" font-size="12">JS</text>
  <rect x="200.0" y="66.8" width="64.0" height="143.2" fill="#e15759"/>
  <text x="222.0" y="240.0" font-size="12">Go</text>
  <rect x="280.0" y="102.6" width="64.0" height="107.4" fill="#76b7b2"/>
  <text x="302.0" y="240.0" font-size="12">Py</text>

  --- Complete SVG Document ---
  <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 400 200">
    <path d="M 20.0 180.0 L 80.0 110.0 L 140.0 144.0 L 200.0 40.0 L 260.0 90.0 L 320.0 20.0 L 380.0 130.0" fill="none" stroke="#4e79a7" stroke-width="2"/>
    <circle cx="20.0" cy="180.0" r="4.0" fill="#e15759"/>
    <circle cx="380.0" cy="42.5" r="4.0" fill="#e15759"/>
  </svg>
---

## Why Generate SVG from Wasm?

SVG (Scalable Vector Graphics) is the web's native vector format. Generating it from Rust/Wasm combines the best of both worlds:

```
  ┌───────────────────────────────────────────────────┐
  │  Wasm (Rust)                                      │
  │                                                   │
  │  ● Compute-heavy path generation                  │
  │  ● Bezier curve math                              │
  │  ● Data → chart conversion                        │
  │  ● SVG path optimization                          │
  │  ● Coordinate transformations                     │
  │                                                   │
  │  Output: SVG string or path data                  │
  └─────────────────────┬─────────────────────────────┘
                        │
                        ▼
  ┌───────────────────────────────────────────────────┐
  │  Browser (JavaScript + DOM)                       │
  │                                                   │
  │  ● Insert SVG into DOM                            │
  │  ● Handle mouse/touch events                      │
  │  ● CSS styling and animations                     │
  │  ● Accessibility (ARIA labels)                    │
  │  ● Responsive layout                              │
  └───────────────────────────────────────────────────┘
```

| Approach              | Pros                            | Cons                          |
|-----------------------|---------------------------------|-------------------------------|
| JS chart library      | Easy API, many options          | Heavy bundle, slow for big data|
| Canvas (Wasm)         | Pixel-perfect, very fast        | Not scalable, no DOM events   |
| SVG from Wasm         | Scalable, accessible, fast math | String building overhead      |
| WebGL from Wasm       | Fastest rendering               | Complex, no text/accessibility|

## SVG Path Commands Reference

SVG paths use a mini-language of commands:

```
  Command  Parameters       Description
  ──────── ──────────────── ──────────────────────────────────
  M x y    Move to          Start a new sub-path
  L x y    Line to          Draw straight line
  H x      Horizontal line  Horizontal shortcut
  V y      Vertical line    Vertical shortcut
  Q cx cy x y  Quadratic    Bezier with 1 control point
  C c1x c1y c2x c2y x y    Cubic Bezier with 2 control points
  A rx ry rot large sweep x y  Elliptical arc
  Z        Close path       Line back to start

  Lowercase = relative coordinates (dx, dy)
  Uppercase = absolute coordinates (x, y)
```

```
  Cubic Bezier: C c1x c1y, c2x c2y, x y

       cp1●
          ╲
           ╲        ●cp2
  start●    ╲      ╱
        ╲    ╲    ╱
         ╲    ╲  ╱
          ╲    ╲╱
           ╲   ╱
            ╲ ╱
             ●end
```

## Bezier Curves: The Math

### Quadratic Bezier (3 points)

```
  B(t) = (1-t)^2 * P0 + 2(1-t)t * P1 + t^2 * P2

          P1 (control)
          ●
         ╱ ╲
        ╱   ╲
  P0 ●╱     ╲● P2
     (start)  (end)

  t=0.0 → at P0
  t=0.5 → near P1 (but not on it!)
  t=1.0 → at P2
```

### Cubic Bezier (4 points)

```
  B(t) = (1-t)^3 * P0 + 3(1-t)^2 * t * P1
       + 3(1-t) * t^2 * P2 + t^3 * P3

  P1 ●─────● P2
     │       │
     │  curve│
  P0 ●       ● P3

  Properties:
  ● Always passes through P0 and P3
  ● Tangent at P0 points toward P1
  ● Tangent at P3 points toward P2
  ● Curve is contained in convex hull of control points
```

### De Casteljau's Algorithm

The most numerically stable way to evaluate a Bezier curve:

```
  Level 0:  P0          P1          P2          P3
              \        / \        / \        /
  Level 1:   A=lerp    B=lerp    C=lerp
                \      / \      /
  Level 2:     D=lerp   E=lerp
                  \    /
  Level 3:       result=lerp(D,E,t)

  Each level: new_point = (1-t) * left + t * right
```

## Building Chart Libraries

### Line chart with data points and grid

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

    // Background
    svg.push_str(&format!(
        r#"<rect width="{}" height="{}" fill="#fafafa"/>"#,
        width, height
    ));

    // Grid lines
    for i in 0..=4 {
        let y = 20.0 + (height - 40.0) * i as f64 / 4.0;
        svg.push_str(&format!(
            r#"<line x1="20" y1="{:.0}" x2="{:.0}" y2="{:.0}" stroke="#eee"/>"#,
            y, width - 20.0, y
        ));
    }

    // Data path
    let path = data_to_path(data, width, height, 20.0);
    svg.push_str(&format!(
        r#"<path d="{}" fill="none" stroke="#4e79a7" stroke-width="2"/>"#,
        path
    ));

    // Data points
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

### Pie chart with arc paths

```rust
fn pie_chart(values: &[f64], colors: &[&str], radius: f64) -> String {
    let total: f64 = values.iter().sum();
    let cx = radius + 10.0;
    let cy = radius + 10.0;
    let mut angle = -std::f64::consts::FRAC_PI_2; // Start at top

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

## SVG Optimization

Generated SVGs can be large. Common optimizations:

```
  Before optimization         After optimization
  ────────────────────        ────────────────────
  M 100.000000 200.000000     M100 200
  L 150.000000 180.000000     L150 180 200 160
  L 200.000000 160.000000     250 170 300 190Z
  L 250.000000 170.000000
  L 300.000000 190.000000
  Z

  Savings: ~40% smaller
```

| Optimization                          | Size Reduction | Implementation          |
|---------------------------------------|----------------|-------------------------|
| Remove unnecessary decimals           | 10-30%         | Round to 1 decimal      |
| Use relative coordinates (lowercase)  | 5-15%          | Delta encoding          |
| Merge consecutive same-type commands  | 5-10%          | "L x y" chaining        |
| Remove redundant whitespace           | 5-10%          | Minification            |
| Simplify paths (Douglas-Peucker)      | 10-50%         | Point reduction         |

### Path simplification (Douglas-Peucker algorithm)

```rust
fn simplify_path(points: &[Point], epsilon: f64) -> Vec<Point> {
    if points.len() < 3 { return points.to_vec(); }

    // Find point with maximum distance from line (first→last)
    let (max_dist, max_idx) = points[1..points.len()-1].iter().enumerate()
        .map(|(i, p)| (point_line_distance(p, &points[0], points.last().unwrap()), i + 1))
        .max_by(|a, b| a.0.partial_cmp(&b.0).unwrap())
        .unwrap();

    if max_dist > epsilon {
        let mut left = simplify_path(&points[..=max_idx], epsilon);
        let right = simplify_path(&points[max_idx..], epsilon);
        left.pop(); // Remove duplicate point
        left.extend(right);
        left
    } else {
        vec![points[0], *points.last().unwrap()]
    }
}
```

## Dynamic Visualizations

The pattern for real-time SVG updates from Wasm:

```javascript
import { generate_chart_path } from './pkg/my_crate';

function animate() {
    const data = getLatestData(); // e.g., from WebSocket
    const pathData = generate_chart_path(
        new Float64Array(data),
        canvas.width,
        canvas.height
    );

    // Update existing SVG path (no DOM recreation!)
    pathElement.setAttribute('d', pathData);
    requestAnimationFrame(animate);
}
```

```
  Data Flow:

  WebSocket ──→ JavaScript ──→ Wasm (path math) ──→ SVG DOM update
                (collect      (generate_chart_path)  (setAttribute)
                 data)                                 60 fps
```

This is faster than recreating the entire SVG because:
1. Only the `d` attribute changes, not the DOM structure
2. Wasm computes the path coordinates in microseconds
3. The browser's SVG renderer handles anti-aliasing and scaling

## Summary

Generating SVG from Wasm lets you combine Rust's computational power with the browser's native vector renderer. Use Wasm for the math-heavy parts (Bezier curves, coordinate transforms, path generation) and let the browser handle rendering, events, and styling. The SVG path mini-language is compact and efficient, and techniques like Douglas-Peucker simplification keep the output small. For real-time visualizations, update path `d` attributes directly rather than recreating DOM elements.
