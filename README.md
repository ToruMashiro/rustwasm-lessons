# RustWasm Lessons

Lesson content for [rustwasm.app](https://rustwasm.app) — a free, bilingual (English/Japanese) learning platform for Rust + WebAssembly.

## Lessons

| # | Lesson | Difficulty |
|---|--------|-----------|
| 1 | What is WebAssembly? | Beginner |
| 2 | Rust Basics for Wasm | Beginner |
| 3 | Hello WebAssembly | Beginner |
| 4 | How wasm-bindgen Works | Beginner |
| 5 | DOM Manipulation | Beginner |
| 6 | Fetch API Calls | Intermediate |
| 7 | Cryptographic Hashing | Intermediate |
| 8 | Particle Simulation | Advanced |
| 9 | Setting Up Your First Project | Beginner |
| 10 | Debugging Wasm | Beginner |
| 11 | Canvas & 2D Graphics | Intermediate |
| 12 | State Management | Intermediate |
| 13 | File Processing | Intermediate |
| 14 | WebSockets | Intermediate |
| 15 | Wasm + Frontend Frameworks | Advanced |
| 16 | Deploying to Production | Advanced |

## Structure

```
en/          English lessons (Markdown)
ja/          Japanese lessons (Markdown)
```

Each lesson is a Markdown file with YAML frontmatter containing:
- `title`, `slug`, `difficulty`, `tags`, `order`
- `description` — short summary
- `starter_code` — Rust code shown in the editor
- `expected_output` — what the code produces

## Contributing

Found a mistake? Want to improve a lesson?

### Report an issue
[Open an issue](../../issues/new) with:
- Which lesson (filename or number)
- What's wrong (incorrect info, typo, unclear explanation, broken code)
- Suggested fix (optional)

### Submit a fix
1. Fork this repo
2. Edit the lesson file(s) in `en/` and/or `ja/`
3. Submit a pull request

### Guidelines
- Keep explanations clear and beginner-friendly
- Code examples must be valid Rust
- If editing English, update the Japanese version too (or note that it needs translation)
- Don't change the frontmatter `slug` or `order` fields

## License

Lesson content is licensed under [CC BY-NC 4.0](https://creativecommons.org/licenses/by-nc/4.0/) — you can share and adapt with attribution, but not for commercial use.

## Links

- **Website**: [rustwasm.app](https://rustwasm.app)
- **English lessons**: [rustwasm.app/en/learn](https://rustwasm.app/en/learn)
- **Japanese lessons**: [rustwasm.app/ja/learn](https://rustwasm.app/ja/learn)
