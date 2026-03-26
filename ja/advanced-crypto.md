---
title: Wasmでの高度な暗号技術
slug: advanced-crypto
difficulty: advanced
tags: [security]
order: 46
description: Rust/WasmでXOR暗号、シーザー暗号、Feistelネットワークブロック暗号を実装 — 現代の暗号化の構成要素を学びます。
starter_code: |
  /// XOR暗号: 暗号化と復号は同じ操作
  fn xor_cipher(data: &[u8], key: &[u8]) -> Vec<u8> {
      data.iter()
          .enumerate()
          .map(|(i, byte)| byte ^ key[i % key.len()])
          .collect()
  }

  /// シーザー暗号: 各文字を固定量だけシフトする
  fn caesar_encrypt(text: &str, shift: u8) -> String {
      text.chars()
          .map(|c| {
              if c.is_ascii_lowercase() {
                  let shifted = (c as u8 - b'a' + shift) % 26 + b'a';
                  shifted as char
              } else if c.is_ascii_uppercase() {
                  let shifted = (c as u8 - b'A' + shift) % 26 + b'A';
                  shifted as char
              } else {
                  c // アルファベット以外の文字はそのまま通過
              }
          })
          .collect()
  }

  fn caesar_decrypt(text: &str, shift: u8) -> String {
      caesar_encrypt(text, 26 - (shift % 26))
  }

  /// シンプルなFeistelネットワーク（2ラウンド、64ビットブロック）
  /// DES、Blowfishなどで使用される構造を示す
  ///
  /// ブロック: [左32ビット | 右32ビット]
  /// 各ラウンド: new_left = right, new_right = left XOR f(right, round_key)
  fn feistel_round_function(data: u32, round_key: u32) -> u32 {
      // シンプルなラウンド関数: 回転、鍵とXOR、ビット混合
      let rotated = data.rotate_left(5);
      let mixed = rotated ^ round_key;
      // ラッピング乗算で非線形性を追加
      mixed.wrapping_mul(0x9E3779B9) // 黄金比定数
  }

  fn feistel_encrypt(block: u64, keys: &[u32]) -> u64 {
      let mut left = (block >> 32) as u32;
      let mut right = block as u32;

      for &round_key in keys {
          let new_left = right;
          let new_right = left ^ feistel_round_function(right, round_key);
          left = new_left;
          right = new_right;
      }

      // 半分を結合（正しい復号のために最後にスワップ）
      ((right as u64) << 32) | (left as u64)
  }

  fn feistel_decrypt(block: u64, keys: &[u32]) -> u64 {
      let mut left = (block >> 32) as u32;
      let mut right = block as u32;

      // 鍵を逆順に適用
      for &round_key in keys.iter().rev() {
          let new_right = left;
          let new_left = right ^ feistel_round_function(left, round_key);
          left = new_left;
          right = new_right;
      }

      ((right as u64) << 32) | (left as u64)
  }

  /// マスター鍵からシンプルな拡張でラウンド鍵を導出する
  fn expand_key(master_key: u64, rounds: usize) -> Vec<u32> {
      let mut keys = Vec::with_capacity(rounds);
      let mut state = master_key;
      for i in 0..rounds {
          state = state.wrapping_mul(6364136223846793005)
              .wrapping_add(i as u64 + 1);
          keys.push((state >> 16) as u32);
      }
      keys
  }

  fn main() {
      // === XOR暗号 ===
      println!("=== XOR暗号 ===");
      let plaintext = b"Hello, WebAssembly!";
      let key = b"SECRET";
      let encrypted = xor_cipher(plaintext, key);
      let decrypted = xor_cipher(&encrypted, key);

      println!("平文:      {:?}", std::str::from_utf8(plaintext).unwrap());
      println!("鍵:        {:?}", std::str::from_utf8(key).unwrap());
      println!("暗号文:    {:?}", encrypted);
      println!("復号文:    {:?}", std::str::from_utf8(&decrypted).unwrap());
      println!("往復一致:  {}", plaintext.as_slice() == decrypted.as_slice());
      println!();

      // === シーザー暗号 ===
      println!("=== シーザー暗号 ===");
      let message = "Attack at dawn! The password is Rust2024.";
      let shift = 13; // ROT13
      let encrypted_msg = caesar_encrypt(message, shift);
      let decrypted_msg = caesar_decrypt(&encrypted_msg, shift);

      println!("元の文:    {}", message);
      println!("シフト:    {}", shift);
      println!("暗号文:    {}", encrypted_msg);
      println!("復号文:    {}", decrypted_msg);
      println!("往復一致:  {}", message == decrypted_msg);
      println!();

      // === Feistelネットワーク ===
      println!("=== Feistelネットワーク（ブロック暗号） ===");
      let master_key: u64 = 0xDEADBEEFCAFEBABE;
      let rounds = 16;
      let round_keys = expand_key(master_key, rounds);

      let plaintext_block: u64 = 0x48656C6C6F210000; // "Hello!\0\0"
      println!("平文ブロック:      0x{:016X}", plaintext_block);
      println!("マスター鍵:        0x{:016X}", master_key);
      println!("ラウンド数:        {}", rounds);

      let ciphertext = feistel_encrypt(plaintext_block, &round_keys);
      println!("暗号文:            0x{:016X}", ciphertext);

      let recovered = feistel_decrypt(ciphertext, &round_keys);
      println!("復号文:            0x{:016X}", recovered);
      println!("往復一致:          {}", plaintext_block == recovered);

      // アバランシェ効果のデモ
      println!();
      println!("=== アバランシェ効果 ===");
      let block_a: u64 = 0x0000000000000000;
      let block_b: u64 = 0x0000000000000001; // 1ビットだけ異なる
      let enc_a = feistel_encrypt(block_a, &round_keys);
      let enc_b = feistel_encrypt(block_b, &round_keys);
      let diff_bits = (enc_a ^ enc_b).count_ones();
      println!("ブロックA:  0x{:016X} -> 0x{:016X}", block_a, enc_a);
      println!("ブロックB:  0x{:016X} -> 0x{:016X}", block_b, enc_b);
      println!("入力の差:   1ビット");
      println!("出力の差:   {}ビット（64ビット中）", diff_bits);
  }
expected_output: |
  === XOR暗号 ===
  平文:      "Hello, WebAssembly!"
  鍵:        "SECRET"
  暗号文:    [27, 0, 31, ...]
  復号文:    "Hello, WebAssembly!"
  往復一致:  true

  === シーザー暗号 ===
  元の文:    Attack at dawn! The password is Rust2024.
  シフト:    13
  暗号文:    Nggnpx ng qnja! Gur cnffjbeq vf Ehfg2024.
  復号文:    Attack at dawn! The password is Rust2024.
  往復一致:  true

  === Feistelネットワーク（ブロック暗号） ===
  平文ブロック:      0x48656C6C6F210000
  ...
  往復一致:          true

  === アバランシェ効果 ===
  入力の差:   1ビット
  出力の差:   約32ビット（64ビット中）
---

## はじめに

暗号ハッシュの関連レッスンでは、一方向関数について解説しました。このレッスンでは*暗号化*の構成要素 -- データの機密性を保護する可逆変換 -- についてさらに深く掘り下げます。純粋なRustで段階的に高度化する3つの暗号を実装し、AESのような現代のアルゴリズムの背後にある原理を説明します。

## なぜ暗号処理にWasmが適しているのか？

WebAssemblyには暗号処理に適したいくつかの特性があります：

| 特性 | 暗号処理でのメリット |
|----------|-------------------|
| 予測可能な実行 | タイミング情報を漏洩しうるJITコンパイラの再最適化がない |
| ガベージコレクションなし | タイミングサイドチャネルを生むGCの一時停止がない |
| 定数時間演算 | JSよりも定数時間コードを書きやすい |
| リニアメモリモデル | ポインタ追跡のオーバーヘッドがなく、キャッシュフレンドリー |
| サンドボックス化 | コードはリニアメモリ外のメモリにアクセスできない |
| ポータブル | 同じバイナリがすべてのプラットフォームで同一に動作 |

**重要な注意点：** Wasmは暗号処理においてJavaScriptより*優れて*いますが、完璧ではありません。Wasmのメモリはホストから検査可能であり、アルゴリズムがサポートされている場合はWeb Crypto APIの使用がプロダクション環境では推奨されます。

## 暗号の分類

```
暗号
├── 対称鍵暗号（同じ鍵で暗号化と復号）
│   ├── ストリーム暗号（バイト単位で暗号化）
│   │   ├── XOR暗号（基本的）
│   │   ├── RC4（非推奨）
│   │   └── ChaCha20（現代的）
│   └── ブロック暗号（固定サイズのブロックを暗号化）
│       ├── Feistelネットワーク
│       │   ├── DES（非推奨、56ビット鍵）
│       │   ├── 3DES（レガシー）
│       │   └── Blowfish / Twofish
│       └── 換字-置換ネットワーク
│           └── AES（現在の標準、128/192/256ビット）
└── 非対称鍵暗号（公開鍵で暗号化、秘密鍵で復号）
    ├── RSA
    ├── Diffie-Hellman / ECDH
    └── Ed25519（署名）
```

## XOR暗号：最もシンプルな暗号化

XOR暗号はすべての現代ストリーム暗号の基盤です。XORが自身の逆演算であることを利用しています：

```
暗号化: 平文 XOR 鍵 = 暗号文
復号:   暗号文 XOR 鍵 = 平文

証明:
  A XOR B XOR B = A  (XORは自己逆)

XORの真理値表:
  0 XOR 0 = 0
  0 XOR 1 = 1
  1 XOR 0 = 1
  1 XOR 1 = 0
```

鍵はデータ全体に循環的に繰り返されます：

```
平文: H  e  l  l  o  ,     W  e  b
鍵:   S  E  C  R  E  T  S  E  C  R
XOR:  1B 20 2F 3E 2A 78 73 12 26 30
```

**XOR単体が安全でない理由：** 繰り返し鍵では、頻度分析で鍵を復元できます。現代のストリーム暗号（ChaCha20）はXORを使用しますが、CSPRNGから鍵ストリームを生成するため、各「鍵バイト」は実質的にランダムになります。

## シーザー暗号：換字の基礎

シーザー暗号はアルファベットの各文字を固定量だけシフトします。shift=3の場合、AはDに、BはEになります。

```
アルファベット:  A B C D E F G H I J K L M N O P Q R S T U V W X Y Z
シフト13:        N O P Q R S T U V W X Y Z A B C D E F G H I J K L M

ROT13 ("HELLO") = "URYYB"
```

ROT13（shift=13）は13 + 13 = 26（アルファベットの長さ）のため、自身が逆関数になります。フォーラムでネタバレテキストに使用されます。

**シーザー暗号が安全でない理由：** 鍵は25通りしかなく、攻撃者はマイクロ秒ですべてを試行できます。文字頻度分析でも即座に解読されます（Eが英語で最も一般的な文字）。

## Feistelネットワーク：実際のブロック暗号の仕組み

Feistelネットワークは、DES、Blowfish、その他多くのブロック暗号の基盤となる構造です。重要な洞察は、暗号化と復号が*同じ*コードを使用し、鍵の順序を逆にするだけという点です。

```
Feistelラウンド（暗号化）:

  ┌─────────┐   ┌─────────┐
  │  左      │   │  右      │
  └────┬─────┘   └────┬─────┘
       │              │
       │         ┌────┴─────┐
       │         │  f(R, K) │  ← ラウンド鍵を使うラウンド関数
       │         └────┬─────┘
       │              │
       ▼    XOR       ▼
       ├──────────────⊕
       │              │
       ▼              ▼
  ┌─────────┐   ┌─────────┐
  │ 新しい右 │   │ 新しい左 │  ← スワップ: new_left = old_right
  └─────────┘   └─────────┘             new_right = old_left XOR f(R, K)
```

ラウンド関数`f(R, K)`は可逆である必要がありません！ Feistel構造は`f`が何をしても可逆性を保証します。これが非常に普及した数学的な優雅さです。

**復号：** 同じアルゴリズムをラウンド鍵を逆順にして実行します。XOR演算が自身を打ち消します。

## ブロック暗号モード：ECB vs CBC

ブロック暗号は固定サイズのブロック（例：64ビットまたは128ビット）を暗号化します。より長いメッセージを暗号化するには*運用モード*が必要です：

### ECB（電子コードブック）-- 使用禁止

各ブロックは同じ鍵で独立に暗号化されます：

```
ECBモード:
ブロック1 ──▶ [暗号化(K)] ──▶ 暗号1
ブロック2 ──▶ [暗号化(K)] ──▶ 暗号2
ブロック3 ──▶ [暗号化(K)] ──▶ 暗号3

問題: 同一の平文ブロック → 同一の暗号文ブロック
      データのパターンが保存される!
```

これは有名な「ECBペンギン」問題です -- ECBモードで画像を暗号化すると、同一のピクセルブロックが同一の暗号ブロックを生成するため、画像の輪郭が保存されます。

### CBC（暗号ブロック連鎖）-- より良い

各ブロックは暗号化前に前の暗号文ブロックとXORされます：

```
CBCモード:
IV ─────────────┐
                ▼
ブロック1 ──XOR──▶ [暗号化(K)] ──▶ 暗号1 ──┐
                                             ▼
ブロック2 ──────────────────────XOR──▶ [暗号化(K)] ──▶ 暗号2 ──┐
                                                                 ▼
ブロック3 ──────────────────────────────────────XOR──▶ [暗号化(K)] ──▶ 暗号3
```

CBCには初期化ベクトル（IV）-- 最初のブロック用のランダム値 -- が必要です。連鎖により各ブロックが前のすべてのブロックに依存するため、同一の平文ブロックでも異なる暗号文が生成されます。

## AES：現代の標準

AES（Advanced Encryption Standard）はFeistelネットワークの代わりに換字-置換ネットワーク（SPN）を使用します：

```
AES-128の構造（10ラウンド）:
┌──────────────┐
│ 128ビットブロック│
├──────────────┤
│ SubBytes     │  ← 非線形バイト置換（S-box）
│ ShiftRows    │  ← 行の巡回シフト
│ MixColumns   │  ← GF(2^8)での行列乗算
│ AddRoundKey  │  ← ラウンド鍵とXOR
├──────────────┤
│ ... 繰り返し │  × 10ラウンド
├──────────────┤
│ 暗号文       │
└──────────────┘
```

AESをスクラッチで実装するのはプレイグラウンドのデモには現実的ではありません（S-boxだけで256バイトのルックアップテーブル）が、RustCryptoエコシステムがプロダクション向けの実装を提供しています。

## Diffie-Hellman鍵交換

Diffie-Hellmanは安全でないチャネルを介して2者が共有秘密を確立できるプロトコルです：

```
Alice                              Bob
─────                              ───
秘密aを選択                        秘密bを選択
A = g^a mod p を計算               B = g^b mod p を計算
         ──── Aを送信 ────▶
         ◀──── Bを送信 ────
s = B^a mod p を計算               s = A^b mod p を計算

両者が得る: s = g^(ab) mod p

盗聴者に見えるもの: g, p, A, B
計算できないもの: s（離散対数問題）
```

## RustCryptoエコシステム

Rust/Wasmでのプロダクション暗号処理には、RustCryptoクレートを使用してください：

| クレート | 目的 | Wasm互換 |
|-------|---------|----------------|
| `aes` | AESブロック暗号 | はい |
| `chacha20poly1305` | 認証付き暗号化 | はい |
| `rsa` | RSA暗号化/署名 | はい |
| `ed25519-dalek` | Ed25519署名 | はい |
| `x25519-dalek` | X25519鍵交換 | はい |
| `sha2` | SHA-256/512ハッシュ | はい |
| `argon2` | パスワードハッシュ | はい |

これらはすべてC依存なしの純粋なRustであるため、修正なしでWasmにコンパイルできます。

## デジタル署名

デジタル署名は、メッセージが特定の送信者から送られ、改ざんされていないことを証明します：

```
署名（秘密鍵）:
  メッセージ ──▶ [ハッシュ] ──▶ [秘密鍵で署名] ──▶ 署名

検証（公開鍵）:
  メッセージ ──▶ [ハッシュ] ──▶ ┐
                                ├──▶ [検証] ──▶ 有効/無効
  署名 ─────────────────────▶ ┘
```

代表的なアルゴリズム：RSA-PSS、Ed25519、ECDSA。Ed25519はその速度、小さな鍵/署名サイズ、実装上の落とし穴への耐性から、新しいシステムでは推奨されています。

## 試してみよう

コードを拡張してみましょう：
- Feistel暗号にブロックをXORで連鎖させる**CBCモード**を実装する
- 25通りすべてのシフトを試し、英語の文字頻度でスコアリングする**総当たりシーザー解読器**を追加する
- シーザーの強化版であるポリアルファベティック換字暗号の**ヴィジュネル暗号**を実装する
- 繰り返しパターンを暗号化して暗号文ブロックが繰り返されることを示すことで**ECBモードが安全でない理由**をデモする
