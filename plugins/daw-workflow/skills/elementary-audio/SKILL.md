---
name: elementary-audio
description: Expert knowledge of Elementary Audio library for real-time DSP in JavaScript/TypeScript. Use when building audio graphs, implementing effects with el.* nodes, understanding the declarative rendering model, optimizing Elementary performance, or debugging Elementary Audio issues. Covers WebRenderer, graph composition, virtual file system, and all built-in DSP nodes.
---

# Elementary Audio Expert Guide

You are an expert in Elementary Audio, a JavaScript/TypeScript library for declarative real-time audio DSP.

## Core Philosophy

Elementary uses **functional reactive programming** for audio:
1. Describe audio as pure functions of application state
2. Elementary efficiently diffs and updates the underlying audio graph
3. Similar to React's virtual DOM, but for audio processing

```typescript
// The fundamental pattern: state → audio graph
function buildAudioGraph(state: AppState): { left: NodeRepr_t, right: NodeRepr_t } {
  // Return Elementary nodes representing your audio processing
}

// When state changes, re-render the graph
renderer.render(left, right);
```

## SoundSage Architecture Patterns

Based on the SoundSage codebase, follow these patterns:

### ElementaryRenderer Wrapper

```typescript
import WebRenderer from '@elemaudio/web-renderer';
import type { NodeRepr_t } from '@elemaudio/core';

export class ElementaryRenderer {
  private renderer: WebRenderer;
  private audioNode: AudioWorkletNode | null = null;

  async initialize(audioContext: AudioContext): Promise<AudioWorkletNode> {
    // CRITICAL: Use 2 separate inputs for true stereo
    this.audioNode = await this.renderer.initialize(audioContext, {
      numberOfInputs: 2,    // Two input buses (L, R)
      numberOfOutputs: 1,
      outputChannelCount: [2],  // Stereo output
    });
    return this.audioNode;
  }

  render(left: NodeRepr_t, right: NodeRepr_t): void {
    // Elementary renders L/R to output channels 0 and 1
    this.renderer.render(left, right);
  }
}
```

### Effects Chain Pattern

```typescript
import { el } from '@elemaudio/core';
import type { NodeRepr_t } from '@elemaudio/core';

class EffectsChain {
  apply(
    left: NodeRepr_t,
    right: NodeRepr_t,
    fx: FxState,
    trackId: string
  ): { left: NodeRepr_t; right: NodeRepr_t } {
    let processedL = left;
    let processedR = right;

    // Chain effects in series: EQ → Compressor → Saturation → Limiter
    if (fx.eq.enabled) {
      processedL = this.buildEQ(processedL, fx.eq, `${trackId}:eq:l`);
      processedR = this.buildEQ(processedR, fx.eq, `${trackId}:eq:r`);
    }
    // ... continue chain

    return { left: processedL, right: processedR };
  }
}
```

## Essential Elementary Nodes

### Signal Sources

```typescript
// Constant value (use for parameters that change via state)
el.const({ key: 'myParam', value: 0.5 })

// Oscillators
el.cycle(frequency)                    // Sine wave
el.saw(frequency)                      // Sawtooth
el.square(frequency)                   // Square wave
el.triangle(frequency)                 // Triangle

// Anti-aliased oscillators (band-limited)
el.blepsaw(frequency)
el.blepsquare(frequency)
el.bleptriangle(frequency)

// Sample playback from virtual file system
el.sample({ path: 'buffer:0' }, trigger, rate)
el.table({ path: 'buffer:0' }, position)  // Direct lookup
```

### Filters (use for EQ)

```typescript
// Basic filters
el.lowpass(frequency, Q, signal)
el.highpass(frequency, Q, signal)
el.bandpass(frequency, Q, signal)
el.notch(frequency, Q, signal)
el.allpass(frequency, Q, signal)

// Shelving EQ
el.lowshelf(frequency, Q, gainLinear, signal)
el.highshelf(frequency, Q, gainLinear, signal)

// Peaking EQ (parametric band)
el.peak(frequency, Q, gainLinear, signal)

// State Variable Filter (multi-output)
el.svf(frequency, Q, signal)  // Returns [lp, bp, hp]
```

### Dynamics Processing

```typescript
// Compressor: (attackMs, releaseMs, thresholdDb, ratio, sidechain, signal)
el.compress(
  el.const({ key: 'attack', value: 10 }),    // ms
  el.const({ key: 'release', value: 100 }),   // ms
  el.const({ key: 'threshold', value: -20 }), // dB
  el.const({ key: 'ratio', value: 4 }),
  signal,  // Sidechain (use signal itself for standard compression)
  signal   // Input to compress
)

// Simplified compressor
el.skcompress(attack, release, threshold, ratio, signal)

// Envelope generators
el.adsr(attack, decay, sustain, release, gate)
el.env(attack, release, signal)
```

### Math & Utilities

```typescript
// Basic math
el.add(a, b, c, ...)      // Sum signals
el.mul(a, b, c, ...)      // Multiply signals
el.sub(a, b)              // Subtract
el.div(a, b)              // Divide

// Comparison
el.min(a, b)
el.max(a, b)
el.le(a, b)               // a <= b ? 1 : 0
el.ge(a, b)               // a >= b ? 1 : 0

// Clipping
el.tanh(signal)           // Soft saturation (-1 to 1)
el.clip(min, max, signal)

// Gain
el.db2gain(dbSignal)      // dB to linear
el.gain2db(linearSignal)  // Linear to dB
```

### Delay & Time-Based

```typescript
// Simple delay
el.delay({ size: maxSamples }, delaySamples, signal)

// Tap delay (for feedback networks)
el.tapIn({ name: 'delayLine' }, signal)
el.tapOut({ name: 'delayLine' }, delaySamples)

// DC blocking
el.dcblock(signal)
```

### Analysis & Metering

```typescript
// Meter (for VU/peak metering)
el.meter({ name: 'trackMeter' }, signal)

// Scope (waveform visualization)
el.scope({ name: 'waveform', size: 512 }, signal)

// FFT (frequency analysis)
el.fft({ name: 'spectrum', size: 2048 }, signal)

// Capture (record to buffer)
el.capture({ name: 'recording', size: 44100 }, signal)
```

## Key Patterns

### Keying for Efficient Updates

Elementary uses keys to identify nodes for diffing. Always key stateful nodes:

```typescript
// GOOD: Keyed constants update efficiently
el.const({ key: `${trackId}:volume`, value: volumeDb })

// BAD: Unkeyed nodes may cause graph rebuilds
el.const({ value: volumeDb })

// Key pattern for multi-instance effects
const keyPrefix = `${trackId}:eq:band${bandIndex}`;
el.peak(
  el.const({ key: `${keyPrefix}:freq`, value: band.frequency }),
  el.const({ key: `${keyPrefix}:q`, value: band.q }),
  el.const({ key: `${keyPrefix}:gain`, value: Math.pow(10, band.gain / 20) }),
  signal
)
```

### Virtual File System

Elementary uses a virtual file system for audio buffers:

```typescript
// Register buffers
await renderer.updateVirtualFileSystem({
  'track1:0': leftChannelFloat32Array,
  'track1:1': rightChannelFloat32Array,
});

// Access in graph
el.table({ path: 'track1:0' }, positionSignal)  // Left channel
el.table({ path: 'track1:1' }, positionSignal)  // Right channel

// On-demand loading via load event
renderer.on('load', async (event) => {
  const buffer = await loadBuffer(event.source);
  return { buffer };
});
```

### Stereo Processing

```typescript
// ALWAYS process L/R separately for true stereo
function processStereo(
  left: NodeRepr_t,
  right: NodeRepr_t,
  params: Params,
  keyPrefix: string
): { left: NodeRepr_t; right: NodeRepr_t } {
  return {
    left: processChannel(left, params, `${keyPrefix}:l`),
    right: processChannel(right, params, `${keyPrefix}:r`),
  };
}

// Pan law (equal power)
const leftGain = Math.cos((pan + 1) * Math.PI / 4);
const rightGain = Math.sin((pan + 1) * Math.PI / 4);
const pannedL = el.mul(signal, el.const({ key: `${id}:panL`, value: leftGain }));
const pannedR = el.mul(signal, el.const({ key: `${id}:panR`, value: rightGain }));
```

### Mixing Multiple Sources

```typescript
// Sum multiple track signals
const trackSignals = tracks.map(track => buildTrackSignal(track));

if (trackSignals.length === 0) {
  return { left: el.const({ key: 'silence', value: 0 }), right: ... };
}

if (trackSignals.length === 1) {
  return trackSignals[0];
}

// el.add accepts variadic arguments
const left = el.add(...trackSignals.map(s => s.left));
const right = el.add(...trackSignals.map(s => s.right));
```

## Performance Optimization

### Minimize Graph Changes

```typescript
// Use el.const with keys for parameters that change frequently
// The graph structure stays the same, only values update
el.mul(
  signal,
  el.const({ key: 'volume', value: state.volume })  // Updates in-place
)

// Avoid creating new nodes for every render
// BAD:
return el.mul(signal, 0.5);  // Creates new node each render

// GOOD:
return el.mul(signal, el.const({ key: 'gain', value: 0.5 }));
```

### Conditional Processing

```typescript
// Skip disabled effects entirely (don't include in graph)
if (fx.eq.enabled) {
  processedL = this.buildEQ(processedL, fx.eq, `${trackId}:eq:l`);
}

// Or use el.select for smooth transitions
const processed = el.select(
  el.const({ key: 'bypass', value: fx.enabled ? 1 : 0 }),
  wetSignal,
  drySignal
)
```

### Buffer Efficiency

```typescript
// Pre-allocate buffers at known sizes
const maxBufferSamples = 44100 * 60;  // 1 minute at 44.1kHz

// Use typed arrays
const buffer = new Float32Array(samples);

// Batch VFS updates when possible
await renderer.updateVirtualFileSystem({
  'track1:0': buffer1L,
  'track1:1': buffer1R,
  'track2:0': buffer2L,
  'track2:1': buffer2R,
});
```

## Common Patterns in SoundSage

### Building a 7-Band Parametric EQ

```typescript
private buildEQ(signal: NodeRepr_t, eq: EQState, keyPrefix: string): NodeRepr_t {
  let processed = signal;

  for (let i = 0; i < eq.bands.length; i++) {
    const band = eq.bands[i];
    if (band.gain === 0 && band.type === 'peaking') continue;  // Skip unity bands

    const key = `${keyPrefix}:band${i}`;
    const gainLinear = Math.pow(10, band.gain / 40);  // dB to linear

    switch (band.type) {
      case 'lowshelf':
        processed = el.lowshelf(
          el.const({ key: `${key}:freq`, value: band.frequency }),
          el.const({ key: `${key}:q`, value: band.q }),
          el.const({ key: `${key}:gain`, value: gainLinear }),
          processed
        );
        break;
      case 'peaking':
        processed = el.peak(
          el.const({ key: `${key}:freq`, value: band.frequency }),
          el.const({ key: `${key}:q`, value: band.q }),
          el.const({ key: `${key}:gain`, value: Math.pow(10, band.gain / 20) }),
          processed
        );
        break;
      // ... other types
    }
  }
  return processed;
}
```

### Saturation with Dry/Wet Mix

```typescript
private buildSaturation(signal: NodeRepr_t, sat: SaturationState, key: string): NodeRepr_t {
  const driveAmount = 1 + sat.drive * 9;  // 0-1 UI → 1-10 DSP range

  // Waveshaping
  const driven = el.mul(signal, el.const({ key: `${key}:drive`, value: driveAmount }));
  const saturated = el.tanh(driven);

  // Normalize
  const normalized = el.mul(saturated, el.const({ key: `${key}:norm`, value: 1 / driveAmount }));

  // Dry/wet mix
  const dry = el.mul(signal, el.const({ key: `${key}:dry`, value: 1 - sat.mix }));
  const wet = el.mul(normalized, el.const({ key: `${key}:wet`, value: sat.mix }));

  return el.add(dry, wet);
}
```

## Debugging

```typescript
// Log graph structure
console.log(JSON.stringify(audioGraph, null, 2));

// Meter for level monitoring
const metered = el.meter({ name: 'debug' }, signal);
renderer.on('meter', (event) => {
  console.log(`${event.name}: ${event.max}`);
});

// Verify renderer state
console.log({
  numberOfInputs: audioNode.numberOfInputs,
  numberOfOutputs: audioNode.numberOfOutputs,
  channelCount: audioNode.channelCount,
});
```

## References

- [Elementary Audio Docs](https://docs.elementary.audio/)
- [Elementary GitHub](https://github.com/elemaudio/elementary)
- [Functional Audio Applications](https://www.nickwritesablog.com/functional-declarative-audio-applications/)
