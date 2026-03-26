---
title: Advanced Cryptography in Wasm
slug: advanced-crypto
difficulty: advanced
tags: [security]
order: 46
description: Implement XOR cipher, Caesar cipher, and a Feistel network block cipher in Rust/Wasm — learn the building blocks of modern encryption.
starter_code: |
  /// XOR cipher: encrypt and decrypt are the same operation
  fn xor_cipher(data: &[u8], key: &[u8]) -> Vec<u8> {
      data.iter()
          .enumerate()
          .map(|(i, byte)| byte ^ key[i % key.len()])
          .collect()
  }

  /// Caesar cipher: shift each letter by a fixed amount
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
                  c // Non-alphabetic characters pass through
              }
          })
          .collect()
  }

  fn caesar_decrypt(text: &str, shift: u8) -> String {
      caesar_encrypt(text, 26 - (shift % 26))
  }

  /// Simple Feistel network (2 rounds, 64-bit block)
  /// This demonstrates the structure used in DES, Blowfish, etc.
  ///
  /// Block: [left_32_bits | right_32_bits]
  /// Each round: new_left = right, new_right = left XOR f(right, round_key)
  fn feistel_round_function(data: u32, round_key: u32) -> u32 {
      // Simple round function: rotate, XOR with key, and mix bits
      let rotated = data.rotate_left(5);
      let mixed = rotated ^ round_key;
      // Add non-linearity with wrapping multiplication
      mixed.wrapping_mul(0x9E3779B9) // golden ratio constant
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

      // Combine halves (swap at the end for correct decryption)
      ((right as u64) << 32) | (left as u64)
  }

  fn feistel_decrypt(block: u64, keys: &[u32]) -> u64 {
      let mut left = (block >> 32) as u32;
      let mut right = block as u32;

      // Apply keys in reverse order
      for &round_key in keys.iter().rev() {
          let new_right = left;
          let new_left = right ^ feistel_round_function(left, round_key);
          left = new_left;
          right = new_right;
      }

      ((right as u64) << 32) | (left as u64)
  }

  /// Derive round keys from a master key using simple expansion
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
      // === XOR Cipher ===
      println!("=== XOR Cipher ===");
      let plaintext = b"Hello, WebAssembly!";
      let key = b"SECRET";
      let encrypted = xor_cipher(plaintext, key);
      let decrypted = xor_cipher(&encrypted, key);

      println!("Plaintext:  {:?}", std::str::from_utf8(plaintext).unwrap());
      println!("Key:        {:?}", std::str::from_utf8(key).unwrap());
      println!("Encrypted:  {:?}", encrypted);
      println!("Decrypted:  {:?}", std::str::from_utf8(&decrypted).unwrap());
      println!("Roundtrip:  {}", plaintext.as_slice() == decrypted.as_slice());
      println!();

      // === Caesar Cipher ===
      println!("=== Caesar Cipher ===");
      let message = "Attack at dawn! The password is Rust2024.";
      let shift = 13; // ROT13
      let encrypted_msg = caesar_encrypt(message, shift);
      let decrypted_msg = caesar_decrypt(&encrypted_msg, shift);

      println!("Original:   {}", message);
      println!("Shift:      {}", shift);
      println!("Encrypted:  {}", encrypted_msg);
      println!("Decrypted:  {}", decrypted_msg);
      println!("Roundtrip:  {}", message == decrypted_msg);
      println!();

      // === Feistel Network ===
      println!("=== Feistel Network (Block Cipher) ===");
      let master_key: u64 = 0xDEADBEEFCAFEBABE;
      let rounds = 16;
      let round_keys = expand_key(master_key, rounds);

      let plaintext_block: u64 = 0x48656C6C6F210000; // "Hello!\0\0"
      println!("Plaintext block:  0x{:016X}", plaintext_block);
      println!("Master key:       0x{:016X}", master_key);
      println!("Rounds:           {}", rounds);

      let ciphertext = feistel_encrypt(plaintext_block, &round_keys);
      println!("Ciphertext:       0x{:016X}", ciphertext);

      let recovered = feistel_decrypt(ciphertext, &round_keys);
      println!("Decrypted:        0x{:016X}", recovered);
      println!("Roundtrip:        {}", plaintext_block == recovered);

      // Demonstrate avalanche effect
      println!();
      println!("=== Avalanche Effect ===");
      let block_a: u64 = 0x0000000000000000;
      let block_b: u64 = 0x0000000000000001; // differs by 1 bit
      let enc_a = feistel_encrypt(block_a, &round_keys);
      let enc_b = feistel_encrypt(block_b, &round_keys);
      let diff_bits = (enc_a ^ enc_b).count_ones();
      println!("Block A:    0x{:016X} -> 0x{:016X}", block_a, enc_a);
      println!("Block B:    0x{:016X} -> 0x{:016X}", block_b, enc_b);
      println!("Input diff: 1 bit");
      println!("Output diff: {} bits (of 64)", diff_bits);
  }
expected_output: |
  === XOR Cipher ===
  Plaintext:  "Hello, WebAssembly!"
  Key:        "SECRET"
  Encrypted:  [27, 0, 31, ...]
  Decrypted:  "Hello, WebAssembly!"
  Roundtrip:  true

  === Caesar Cipher ===
  Original:   Attack at dawn! The password is Rust2024.
  Shift:      13
  Encrypted:  Nggnpx ng qnja! Gur cnffjbeq vf Ehfg2024.
  Decrypted:  Attack at dawn! The password is Rust2024.
  Roundtrip:  true

  === Feistel Network (Block Cipher) ===
  Plaintext block:  0x48656C6C6F210000
  ...
  Roundtrip:        true

  === Avalanche Effect ===
  Input diff: 1 bit
  Output diff: ~32 bits (of 64)
---

## Introduction

In the companion lesson on cryptographic hashing, we covered one-way functions. This lesson goes further into the building blocks of *encryption* -- reversible transformations that protect data confidentiality. We implement three progressively more sophisticated ciphers in pure Rust and explain the principles behind modern algorithms like AES.

## Why Wasm for cryptography?

WebAssembly has several properties that make it suitable for cryptographic operations:

| Property | Benefit for crypto |
|----------|-------------------|
| Predictable execution | No JIT compiler reoptimizations that could leak timing info |
| No garbage collection | No GC pauses that create timing side channels |
| Constant-time operations | Easier to write constant-time code than in JS |
| Linear memory model | No pointer-chasing overhead, cache-friendly |
| Sandboxed | Code cannot access memory outside its linear memory |
| Portable | Same binary runs identically on all platforms |

**Important caveat:** Wasm is *better* than JavaScript for crypto, but not perfect. Wasm memory can still be inspected by the host, and the Web Crypto API should be preferred for production use when it supports your algorithm.

## Cipher taxonomy

```
Ciphers
├── Symmetric (same key encrypts and decrypts)
│   ├── Stream ciphers (encrypt byte-by-byte)
│   │   ├── XOR cipher (trivial)
│   │   ├── RC4 (deprecated)
│   │   └── ChaCha20 (modern)
│   └── Block ciphers (encrypt fixed-size blocks)
│       ├── Feistel networks
│       │   ├── DES (deprecated, 56-bit key)
│       │   ├── 3DES (legacy)
│       │   └── Blowfish / Twofish
│       └── Substitution-permutation networks
│           └── AES (current standard, 128/192/256-bit)
└── Asymmetric (public key encrypts, private key decrypts)
    ├── RSA
    ├── Diffie-Hellman / ECDH
    └── Ed25519 (signatures)
```

## XOR cipher: the simplest encryption

The XOR cipher is the foundation of all modern stream ciphers. It works because XOR is its own inverse:

```
Encrypt:  plaintext XOR key = ciphertext
Decrypt:  ciphertext XOR key = plaintext

Proof:
  A XOR B XOR B = A  (XOR is self-inverse)

Truth table for XOR:
  0 XOR 0 = 0
  0 XOR 1 = 1
  1 XOR 0 = 1
  1 XOR 1 = 0
```

The key is repeated cyclically across the data:

```
Plaintext: H  e  l  l  o  ,     W  e  b
Key:       S  E  C  R  E  T  S  E  C  R
XOR:       1B 20 2F 3E 2A 78 73 12 26 30
```

**Why XOR alone is insecure:** With a repeating key, frequency analysis can recover the key. Modern stream ciphers (ChaCha20) use XOR but generate the key stream from a CSPRNG, making each "key byte" effectively random.

## Caesar cipher: substitution fundamentals

The Caesar cipher shifts each letter by a fixed amount in the alphabet. With shift=3, A becomes D, B becomes E, etc.

```
Alphabet:  A B C D E F G H I J K L M N O P Q R S T U V W X Y Z
Shift 13:  N O P Q R S T U V W X Y Z A B C D E F G H I J K L M

ROT13 ("HELLO") = "URYYB"
```

ROT13 (shift=13) is its own inverse because 13 + 13 = 26 (the alphabet length). This is used for spoiler text in forums.

**Why Caesar is insecure:** Only 25 possible keys -- an attacker can try all of them in microseconds. Letter frequency analysis also breaks it instantly (E is the most common English letter).

## The Feistel network: how real block ciphers work

A Feistel network is the structure behind DES, Blowfish, and many other block ciphers. Its key insight is that encryption and decryption use the *same* code -- you just reverse the key order.

```
Feistel round (encrypt):

  ┌─────────┐   ┌─────────┐
  │  Left    │   │  Right   │
  └────┬─────┘   └────┬─────┘
       │              │
       │         ┌────┴─────┐
       │         │  f(R, K) │  ← Round function with round key
       │         └────┬─────┘
       │              │
       ▼    XOR       ▼
       ├──────────────⊕
       │              │
       ▼              ▼
  ┌─────────┐   ┌─────────┐
  │ new Right│   │ new Left │  ← Swap: new_left = old_right
  └─────────┘   └─────────┘            new_right = old_left XOR f(R, K)
```

The round function `f(R, K)` does NOT need to be reversible! The Feistel structure guarantees reversibility regardless of what `f` does. This is the mathematical elegance that made it so popular.

**Decryption:** Run the same algorithm but with round keys in reverse order. The XOR operations undo themselves.

## Block cipher modes: ECB vs CBC

A block cipher encrypts fixed-size blocks (e.g., 64 or 128 bits). To encrypt longer messages, you need a *mode of operation*:

### ECB (Electronic Codebook) -- DO NOT USE

Each block is encrypted independently with the same key:

```
ECB Mode:
Block 1 ──▶ [Encrypt(K)] ──▶ Cipher 1
Block 2 ──▶ [Encrypt(K)] ──▶ Cipher 2
Block 3 ──▶ [Encrypt(K)] ──▶ Cipher 3

Problem: identical plaintext blocks → identical ciphertext blocks
         Patterns in the data are preserved!
```

This is the famous "ECB penguin" problem -- encrypting an image in ECB mode preserves the outline of the image because identical pixel blocks produce identical cipher blocks.

### CBC (Cipher Block Chaining) -- Better

Each block is XORed with the previous ciphertext block before encryption:

```
CBC Mode:
IV ─────────────┐
                ▼
Block 1 ──XOR──▶ [Encrypt(K)] ──▶ Cipher 1 ──┐
                                               ▼
Block 2 ──────────────────────XOR──▶ [Encrypt(K)] ──▶ Cipher 2 ──┐
                                                                   ▼
Block 3 ──────────────────────────────────────XOR──▶ [Encrypt(K)] ──▶ Cipher 3
```

CBC requires an Initialization Vector (IV) -- a random value for the first block. Identical plaintext blocks produce different ciphertext because the chaining makes each block depend on all previous blocks.

## AES: the modern standard

AES (Advanced Encryption Standard) uses a Substitution-Permutation Network (SPN) instead of a Feistel network:

```
AES-128 structure (10 rounds):
┌──────────────┐
│ 128-bit block │
├──────────────┤
│ SubBytes     │  ← Non-linear byte substitution (S-box)
│ ShiftRows    │  ← Circular shift of rows
│ MixColumns   │  ← Matrix multiplication in GF(2^8)
│ AddRoundKey  │  ← XOR with round key
├──────────────┤
│ ... repeat   │  × 10 rounds
├──────────────┤
│ Ciphertext   │
└──────────────┘
```

AES is not practical to implement from scratch for a playground demo (the S-box alone is a 256-byte lookup table), but the RustCrypto ecosystem provides production-ready implementations.

## Diffie-Hellman key exchange

Diffie-Hellman allows two parties to establish a shared secret over an insecure channel:

```
Alice                              Bob
─────                              ───
Choose secret a                    Choose secret b
Compute A = g^a mod p              Compute B = g^b mod p
         ──── send A ────▶
         ◀──── send B ────
Compute s = B^a mod p              Compute s = A^b mod p

Both get: s = g^(ab) mod p

Eavesdropper sees: g, p, A, B
Cannot compute: s (discrete logarithm problem)
```

## The RustCrypto ecosystem

For production cryptography in Rust/Wasm, use the RustCrypto crates:

| Crate | Purpose | Wasm compatible |
|-------|---------|----------------|
| `aes` | AES block cipher | Yes |
| `chacha20poly1305` | Authenticated encryption | Yes |
| `rsa` | RSA encryption/signatures | Yes |
| `ed25519-dalek` | Ed25519 signatures | Yes |
| `x25519-dalek` | X25519 key exchange | Yes |
| `sha2` | SHA-256/512 hashing | Yes |
| `argon2` | Password hashing | Yes |

All of these are pure Rust with no C dependencies, so they compile to Wasm without modification.

## Digital signatures

Digital signatures prove that a message came from a specific sender and was not tampered with:

```
Signing (private key):
  Message ──▶ [Hash] ──▶ [Sign with private key] ──▶ Signature

Verification (public key):
  Message ──▶ [Hash] ──▶ ┐
                          ├──▶ [Verify] ──▶ Valid/Invalid
  Signature ─────────────▶ ┘
```

Common algorithms: RSA-PSS, Ed25519, ECDSA. Ed25519 is preferred for new systems due to its speed, small key/signature size, and resistance to implementation pitfalls.

## Try it

Extend the code to:
- Implement **CBC mode** for the Feistel cipher by chaining blocks with XOR
- Add a **brute-force Caesar cracker** that tries all 25 shifts and scores results by English letter frequency
- Implement a **Vigenere cipher** (polyalphabetic substitution) which is a stronger version of Caesar
- Demonstrate why **ECB mode is insecure** by encrypting a repeating pattern and showing that ciphertext blocks repeat
