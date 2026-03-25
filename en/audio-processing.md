---
title: Audio Processing
slug: audio-processing
difficulty: advanced
tags: [api]
order: 23
description: Process and synthesize audio in real-time with Rust/Wasm and the Web Audio API — filters, effects, and waveform generation.
starter_code: |
  use wasm_bindgen::prelude::*;

  /// Generate a sine wave buffer
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

  /// Apply a low-pass filter
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

  /// Calculate the RMS volume of a buffer
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

## Introduction

Audio processing is a perfect fit for Wasm — it requires low-latency, real-time computation. Rust/Wasm processes audio buffers faster than JavaScript while the Web Audio API handles playback.

## Architecture

```
┌──────────────────┐         ┌──────────────────┐
│  Web Audio API   │ buffer  │    Rust/Wasm     │
│  (AudioContext)  │────────▶│  (process audio) │
│                  │◀────────│                   │
│  🔊 Speakers    │ buffer  │  DSP, filters,   │
│                  │         │  synthesis        │
└──────────────────┘         └──────────────────┘
```

## Setup

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

## Playing Generated Audio

```js
import init, { generate_sine } from './pkg/audio_wasm.js';

await init();

const ctx = new AudioContext();
const samples = generate_sine(ctx.sampleRate, 440, 1.0); // A4 note, 1 second

// Create AudioBuffer from Wasm-generated data
const buffer = ctx.createBuffer(1, samples.length, ctx.sampleRate);
buffer.copyToChannel(samples, 0);

// Play it
const source = ctx.createBufferSource();
source.buffer = buffer;
source.connect(ctx.destination);
source.start();
```

## Real-time Processing with AudioWorklet

For low-latency audio processing, use AudioWorklet:

```js
// audio-processor.js (runs in audio thread)
class WasmProcessor extends AudioWorkletProcessor {
    constructor() {
        super();
        this.wasmReady = false;
        this.port.onmessage = async (e) => {
            if (e.data.type === 'init') {
                // Load Wasm module in the audio thread
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

        // Process with Rust/Wasm
        this.filter(output, 1000, sampleRate);

        return true;
    }
}
registerProcessor('wasm-processor', WasmProcessor);
```

## Common Audio DSP Functions

### Gain (volume)

```rust
#[wasm_bindgen]
pub fn apply_gain(samples: &mut [f32], gain: f32) {
    for sample in samples.iter_mut() {
        *sample *= gain;
        *sample = sample.clamp(-1.0, 1.0);
    }
}
```

### Distortion

```rust
#[wasm_bindgen]
pub fn distortion(samples: &mut [f32], amount: f32) {
    for sample in samples.iter_mut() {
        let x = *sample * amount;
        *sample = x / (1.0 + x.abs());
    }
}
```

### Reverb (simple delay)

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

## Performance: Why Wasm for Audio?

| Operation | JavaScript | Rust/Wasm |
|-----------|-----------|----------|
| FFT (1024 samples) | ~0.5ms | ~0.08ms |
| Low-pass filter (44100 samples) | ~2ms | ~0.3ms |
| Waveform generation (1 sec) | ~5ms | ~0.8ms |

Audio processing requires completing within ~5ms per buffer (at 44.1kHz, 256 sample blocks). JavaScript sometimes misses this deadline → audio glitches. Rust/Wasm consistently meets it.

## Try It

The starter code generates a sine wave, applies a low-pass filter, and measures volume. In a real project, these functions would process live audio buffers from a microphone or audio file.
