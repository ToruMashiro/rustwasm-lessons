---
title: "Wasm + PDF Generation"
slug: pdf-generation
difficulty: advanced
tags: [api]
order: 47
description: Generate PDF files client-side with Rust/Wasm — learn the PDF file format internals and build a minimal PDF generator from scratch.
starter_code: |
  /// A minimal PDF generator that creates valid PDF files
  /// PDF is a text-based format with binary sections for streams

  struct PdfDocument {
      objects: Vec<String>,
      pages: Vec<usize>,  // object indices for page objects
  }

  impl PdfDocument {
      fn new() -> Self {
          PdfDocument {
              objects: Vec::new(),
              pages: Vec::new(),
          }
      }

      /// Add an object and return its object number (1-based)
      fn add_object(&mut self, content: String) -> usize {
          self.objects.push(content);
          self.objects.len() // 1-based object number
      }

      /// Add a page with text content
      fn add_page(&mut self, lines: &[&str], font_size: u32) {
          // Content stream: position text on the page
          let mut stream = String::new();
          stream.push_str("BT\n"); // Begin Text
          stream.push_str(&format!("/F1 {} Tf\n", font_size));

          let mut y = 750; // Start near top of page
          for line in lines {
              stream.push_str(&format!("1 0 0 1 50 {} Tm\n", y));
              // Escape special PDF characters
              let escaped = line
                  .replace('\\', "\\\\")
                  .replace('(', "\\(")
                  .replace(')', "\\)");
              stream.push_str(&format!("({}) Tj\n", escaped));
              y -= (font_size as i32) + 4;
          }
          stream.push_str("ET\n"); // End Text

          let stream_length = stream.len();

          // Stream object (contains the page drawing commands)
          let stream_obj = format!(
              "<< /Length {} >>\nstream\n{}endstream",
              stream_length, stream
          );
          let stream_num = self.add_object(stream_obj);

          // Page object (references the stream and font)
          // Parent will be filled in during render
          let page_obj = format!(
              "<< /Type /Page /Parent {{PAGES}} /MediaBox [0 0 612 792] \
               /Contents {} 0 R /Resources << /Font << /F1 {{FONT}} >> >> >>",
              stream_num
          );
          let page_num = self.add_object(page_obj);
          self.pages.push(page_num);
      }

      /// Render the complete PDF as a byte vector
      fn render(&self) -> Vec<u8> {
          let mut output = Vec::new();

          // PDF Header
          output.extend_from_slice(b"%PDF-1.4\n");

          // Font object (using built-in Helvetica -- no embedding needed)
          let font_obj_num = self.objects.len() + 1;
          let pages_obj_num = self.objects.len() + 2;
          let catalog_obj_num = self.objects.len() + 3;

          // Track byte offsets for cross-reference table
          let mut offsets: Vec<usize> = Vec::new();

          // Write document objects
          for (i, obj) in self.objects.iter().enumerate() {
              offsets.push(output.len());
              let obj_num = i + 1;
              let rendered = obj
                  .replace("{PAGES}", &format!("{} 0 R", pages_obj_num))
                  .replace("{FONT}", &format!("{} 0 R", font_obj_num));
              let line = format!("{} 0 obj\n{}\nendobj\n", obj_num, rendered);
              output.extend_from_slice(line.as_bytes());
          }

          // Font object (Helvetica is built into all PDF readers)
          offsets.push(output.len());
          let font = format!(
              "{} 0 obj\n<< /Type /Font /Subtype /Type1 /BaseFont /Helvetica >>\nendobj\n",
              font_obj_num
          );
          output.extend_from_slice(font.as_bytes());

          // Pages object (parent of all page objects)
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

          // Catalog object (root of the PDF)
          offsets.push(output.len());
          let catalog = format!(
              "{} 0 obj\n<< /Type /Catalog /Pages {} 0 R >>\nendobj\n",
              catalog_obj_num, pages_obj_num
          );
          output.extend_from_slice(catalog.as_bytes());

          // Cross-reference table
          let xref_offset = output.len();
          let total_objects = offsets.len();
          let mut xref = format!("xref\n0 {}\n", total_objects + 1);
          xref.push_str("0000000000 65535 f \n");
          for offset in &offsets {
              xref.push_str(&format!("{:010} 00000 n \n", offset));
          }
          output.extend_from_slice(xref.as_bytes());

          // Trailer
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

      // Page 1: Title page
      doc.add_page(&[
          "Rust + WebAssembly PDF Generator",
          "",
          "Generated entirely client-side",
          "No server round-trip needed!",
          "",
          "This PDF was created by a minimal",
          "PDF generator written in pure Rust.",
      ], 16);

      // Page 2: Content page
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

      println!("=== PDF Generation Complete ===");
      println!("PDF size: {} bytes", pdf_bytes.len());
      println!("Pages: {}", doc.pages.len());
      println!();

      // Show the structure
      let pdf_str = String::from_utf8_lossy(&pdf_bytes);
      println!("=== PDF Structure (first 500 chars) ===");
      let preview: String = pdf_str.chars().take(500).collect();
      println!("{}", preview);
      println!("...");
      println!();

      // Verify it starts and ends correctly
      println!("=== Validation ===");
      println!("Starts with %PDF-1.4: {}",
          pdf_str.starts_with("%PDF-1.4"));
      println!("Ends with %%EOF: {}",
          pdf_str.trim().ends_with("%%EOF"));
      println!("Contains xref table: {}",
          pdf_str.contains("xref"));
      println!("Contains trailer: {}",
          pdf_str.contains("trailer"));
  }
expected_output: |
  === PDF Generation Complete ===
  PDF size: ~1200 bytes
  Pages: 2

  === PDF Structure (first 500 chars) ===
  %PDF-1.4
  1 0 obj
  << /Length ... >>
  stream
  BT
  /F1 16 Tf
  ...

  === Validation ===
  Starts with %PDF-1.4: true
  Ends with %%EOF: true
  Contains xref table: true
  Contains trailer: true
---

## Introduction

Generating PDFs on the client side with Rust/Wasm eliminates the need to send sensitive data to a server for document creation. Invoices, reports, certificates, and receipts can all be generated directly in the browser. This lesson builds a minimal but valid PDF generator from scratch and explains the PDF file format internals.

## Why client-side PDF generation?

| Approach | Latency | Privacy | Offline | Server cost |
|----------|---------|---------|---------|-------------|
| Server-side (wkhtmltopdf) | 200-500ms + network | Data sent to server | No | High |
| Server-side (Puppeteer) | 500-2000ms + network | Data sent to server | No | Very high |
| Client JS (jsPDF) | 50-200ms | Data stays local | Yes | None |
| Client Wasm (Rust) | 10-50ms | Data stays local | Yes | None |

Wasm is 3-5x faster than JavaScript for PDF generation because it avoids GC pressure when building large byte arrays and performs efficient string formatting.

## PDF file format anatomy

A PDF file has four main sections:

```
┌──────────────────────────────┐
│  Header                      │  %PDF-1.4
├──────────────────────────────┤
│  Body                        │  Objects (pages, fonts,
│  (numbered objects)          │   images, content streams)
├──────────────────────────────┤
│  Cross-Reference Table       │  Byte offsets of each object
│  (xref)                      │  for random access
├──────────────────────────────┤
│  Trailer                     │  Points to root object
│                              │  and xref table
└──────────────────────────────┘
```

### Object hierarchy

Every PDF has this tree structure:

```
Catalog (root)
└── Pages (collection)
    ├── Page 1
    │   ├── MediaBox [0 0 612 792]  (US Letter size in points)
    │   ├── Resources
    │   │   └── Font /F1 → Helvetica
    │   └── Contents → Stream object
    │       └── "BT /F1 12 Tf (Hello) Tj ET"
    ├── Page 2
    │   └── ...
    └── Page N
```

### PDF objects

Objects are numbered (1 0 obj, 2 0 obj, etc.) and can be dictionaries, arrays, streams, or primitive values:

```
% Dictionary object
1 0 obj
<< /Type /Catalog /Pages 2 0 R >>
endobj

% Stream object (contains drawing commands)
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

The `R` in `2 0 R` means "reference to object 2, generation 0" -- this is how objects link to each other.

## Content stream commands

PDF content streams use a stack-based drawing language similar to PostScript:

| Command | Meaning | Example |
|---------|---------|---------|
| `BT` | Begin text block | `BT` |
| `ET` | End text block | `ET` |
| `Tf` | Set font and size | `/F1 12 Tf` |
| `Tm` | Set text matrix (position) | `1 0 0 1 50 750 Tm` |
| `Tj` | Show text string | `(Hello) Tj` |
| `Td` | Move text position | `0 -14 Td` |
| `m` | Move to point | `50 700 m` |
| `l` | Line to point | `550 700 l` |
| `S` | Stroke path | `S` |
| `re` | Rectangle | `50 50 500 700 re` |
| `rg` | Set fill color (RGB) | `1 0 0 rg` (red) |

The coordinate system starts at the bottom-left corner. US Letter is 612 x 792 points (1 point = 1/72 inch).

## Built-in fonts

PDF defines 14 standard fonts that every PDF reader must support -- no embedding required:

```
Helvetica              Helvetica-Bold
Helvetica-Oblique      Helvetica-BoldOblique
Times-Roman            Times-Bold
Times-Italic           Times-BoldItalic
Courier                Courier-Bold
Courier-Oblique        Courier-BoldOblique
Symbol                 ZapfDingbats
```

Using these fonts keeps the PDF small. Custom fonts require embedding the font file, which can add hundreds of kilobytes.

## The cross-reference table

The xref table enables random access to objects without reading the entire file. Each entry records the byte offset of an object from the start of the file:

```
xref
0 5
0000000000 65535 f     ← free object (always first)
0000000009 00000 n     ← object 1 at byte 9
0000000058 00000 n     ← object 2 at byte 58
0000000115 00000 n     ← object 3 at byte 115
0000000258 00000 n     ← object 4 at byte 258
```

This is critical for large PDFs -- a reader can jump directly to page 500 without parsing pages 1-499.

## Adding images to PDFs

Images in PDF are stored as stream objects with specific filters:

```
% Image object (conceptual)
5 0 obj
<< /Type /XObject
   /Subtype /Image
   /Width 200
   /Height 150
   /ColorSpace /DeviceRGB
   /BitsPerComponent 8
   /Length 90000
   /Filter /DCTDecode    ← This means JPEG data
>>
stream
[raw JPEG bytes here]
endstream
endobj
```

For Wasm, you can pass image bytes from JavaScript to Rust and embed them directly as JPEG (DCTDecode) or PNG (FlateDecode) streams.

## Table generation

Tables in PDF are drawn manually using line and text commands:

```
Content stream for a simple table:
% Draw grid lines
0.5 w                    % line width
50 700 m 550 700 l S     % top border
50 680 m 550 680 l S     % header separator
50 660 m 550 660 l S     % row 1
50 700 m 50 660 l S      % left border
300 700 m 300 660 l S    % column separator
550 700 m 550 660 l S    % right border

% Header text
BT /F1 10 Tf
1 0 0 1 55 685 Tm (Name) Tj
1 0 0 1 305 685 Tm (Value) Tj
ET
```

This is why PDF libraries are valuable -- calculating cell positions, handling text wrapping, and managing page breaks is tedious to do manually.

## Production crates: printpdf and genpdf

For real applications, these Rust crates compile to Wasm:

**printpdf** -- Low-level PDF generation:
```rust
// Requires printpdf crate
use printpdf::*;

let (doc, page1, layer1) = PdfDocument::new(
    "My Document", Mm(210.0), Mm(297.0), "Layer 1"
);
let font = doc.add_builtin_font(BuiltinFont::Helvetica).unwrap();
let layer = doc.get_page(page1).get_layer(layer1);
layer.use_text("Hello from Rust!", 14.0, Mm(10.0), Mm(280.0), &font);
```

**genpdf** -- Higher-level with layout engine:
```rust
// Requires genpdf crate
let font = genpdf::fonts::from_files("./fonts", "Helvetica", None).unwrap();
let mut doc = genpdf::Document::new(font);
doc.push(genpdf::elements::Paragraph::new("Hello from Rust!"));
doc.push(genpdf::elements::Break::new(1));
doc.push(genpdf::elements::Paragraph::new("Second paragraph."));
```

## Try it

Extend the PDF generator to:
- Add **drawing commands** for horizontal rules (lines across the page)
- Support **bold text** by switching to `/F2` (Helvetica-Bold) mid-stream
- Add **page numbers** at the bottom of each page
- Generate a **table** with grid lines and cell text
- Add **color** to text using the `rg` (fill color) command
