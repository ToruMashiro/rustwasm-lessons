---
title: オーディオ処理
slug: audio-processing
difficulty: advanced
tags: [api]
order: 23
description: Rust/WasmとWeb Audio APIでリアルタイムにオーディオを処理・合成する — フィルター、エフェクト、波形生成。
starter_code: |
  use wasm_bindgen::prelude::*;

  /// サイン波バッファを生成
  #[wasm_bindgen]
  pub fn generate_sine(sample_rate: f32, frequency: f32, duration: f32) -> Vec<f32> {
      let num_samples = (sample_rate * duration) as usize;
      let mut buffer = Vec::with_capacity(num_samples);

      for i in 0..num_samples {
          let t = i as f32 / sample_rate;
          let sample = (2.0 * std::f32::consts::PI * frequency * t).sin();
          buffer.push(sample);
      }

      buffer
  }

  /// ローパスフィルターを適用
  #[wasm_bindgen]
  pub fn low_pass_filter(samples: &mut [f32], cutoff: f32, sample_rate: f32) {
      let rc = 1.0 / (2.0 * std::f32::consts::PI * cutoff);
      let dt = 1.0 / sample_rate;
      let alpha = dt / (rc + dt);

      let mut prev = samples[0];
      for i in 1..samples.len() {
          samples[i] = prev + alpha * (samples[i] - prev);
          prev = samples[i];
      }
  }

  /// バッファのRMS音量を計算
  #[wasm_bindgen]
  pub fn rms_volume(samples: &[f32]) -> f32 {
      let sum: f32 = samples.iter().map(|s| s * s).sum();
      (sum / samples.len() as f32).sqrt()
  }
expected_output: |
  Generated 44100 samples at 440Hz (1 second)
  RMS volume: 0.707
  After low-pass filter at 200Hz: RMS = 0.412
---

## はじめに

オーディオ処理はWasmに最適です — 低レイテンシでリアルタイムの計算が必要です。Rust/WasmはJavaScriptよりも高速にオーディオバッファを処理し、Web Audio APIが再生を担当します。

## アーキテクチャ

```
┌──────────────────┐         ┌──────────────────┐
│  Web Audio API   │ buffer  │    Rust/Wasm     │
│  (AudioContext)  │────────▶│  (オーディオ処理)  │
│                  │◀────────│                   │
│  スピーカー       │ buffer  │  DSP、フィルター、 │
│                  │         │  合成              │
└──────────────────┘         └──────────────────┘
```

## セットアップ

```toml
[dependencies]
wasm-bindgen = "0.2"

[dependencies.web-sys]
version = "0.3"
features = [
    "AudioContext",
    "AudioBuffer",
    "AudioBufferSourceNode",
    "GainNode",
    "AudioDestinationNode",
]
```

## 生成したオーディオの再生

```js
import init, { generate_sine } from './pkg/audio_wasm.js';

await init();

const ctx = new AudioContext();
const samples = generate_sine(ctx.sampleRate, 440, 1.0); // A4音、1秒

// Wasmで生成したデータからAudioBufferを作成
const buffer = ctx.createBuffer(1, samples.length, ctx.sampleRate);
buffer.copyToChannel(samples, 0);

// 再生
const source = ctx.createBufferSource();
source.buffer = buffer;
source.connect(ctx.destination);
source.start();
```

## AudioWorkletによるリアルタイム処理

低レイテンシのオーディオ処理にはAudioWorkletを使います:

```js
// audio-processor.js（オーディオスレッドで実行）
class WasmProcessor extends AudioWorkletProcessor {
    constructor() {
        super();
        this.wasmReady = false;
        this.port.onmessage = async (e) => {
            if (e.data.type === 'init') {
                // オーディオスレッドでWasmモジュールを読み込む
                const { default: init, low_pass_filter } = await import('./pkg/audio_wasm.js');
                await init();
                this.filter = low_pass_filter;
                this.wasmReady = true;
            }
        };
    }

    process(inputs, outputs) {
        if (!this.wasmReady || !inputs[0].length) return true;

        const input = inputs[0][0];
        const output = outputs[0][0];
        output.set(input);

        // Rust/Wasmで処理
        this.filter(output, 1000, sampleRate);

        return true;
    }
}
registerProcessor('wasm-processor', WasmProcessor);
```

## よく使うオーディオDSP関数

### ゲイン（音量）

```rust
#[wasm_bindgen]
pub fn apply_gain(samples: &mut [f32], gain: f32) {
    for sample in samples.iter_mut() {
        *sample *= gain;
        *sample = sample.clamp(-1.0, 1.0);
    }
}
```

### ディストーション

```rust
#[wasm_bindgen]
pub fn distortion(samples: &mut [f32], amount: f32) {
    for sample in samples.iter_mut() {
        let x = *sample * amount;
        *sample = x / (1.0 + x.abs());
    }
}
```

### リバーブ（シンプルなディレイ）

```rust
#[wasm_bindgen]
pub fn delay(samples: &mut [f32], delay_samples: usize, feedback: f32) {
    let len = samples.len();
    for i in delay_samples..len {
        samples[i] += samples[i - delay_samples] * feedback;
        samples[i] = samples[i].clamp(-1.0, 1.0);
    }
}
```

## パフォーマンス: なぜWasmでオーディオ処理？

| 処理 | JavaScript | Rust/Wasm |
|-----------|-----------|----------|
| FFT（1024サンプル） | ~0.5ms | ~0.08ms |
| ローパスフィルター（44100サンプル） | ~2ms | ~0.3ms |
| 波形生成（1秒） | ~5ms | ~0.8ms |

オーディオ処理はバッファあたり約5ms以内に完了する必要があります（44.1kHz、256サンプルブロック時）。JavaScriptはこの期限に間に合わないことがあり、オーディオのグリッチが発生します。Rust/Wasmなら安定して間に合います。

## 試してみよう

スターターコードはサイン波を生成し、ローパスフィルターを適用し、音量を測定します。実際のプロジェクトでは、これらの関数はマイクや音声ファイルからのライブオーディオバッファを処理します。
