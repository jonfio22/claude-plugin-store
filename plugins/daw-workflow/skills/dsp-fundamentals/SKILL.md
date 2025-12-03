---
name: dsp-fundamentals
description: Deep knowledge of Digital Signal Processing mathematics for audio applications. Use when implementing filters (biquad, IIR, FIR), dynamics processors (compressors, limiters, gates), saturation/distortion, or any audio effect requiring mathematical foundations. Covers filter coefficient calculations, envelope followers, transfer functions, and the Audio EQ Cookbook formulas.
---

# DSP Fundamentals for Audio Processing

You are an expert in digital signal processing mathematics for audio applications. Apply these fundamentals when implementing audio effects.

## Biquad Filter Mathematics (Audio EQ Cookbook)

The biquad is the fundamental building block for EQ, filters, and many effects. Transfer function:

```
H(z) = (b₀ + b₁z⁻¹ + b₂z⁻²) / (a₀ + a₁z⁻¹ + a₂z⁻²)
```

### Intermediate Variables (Calculate First)

```typescript
// For all filters
const omega0 = 2 * Math.PI * (frequency / sampleRate);
const cosW0 = Math.cos(omega0);
const sinW0 = Math.sin(omega0);

// Alpha via Q
const alpha = sinW0 / (2 * Q);

// Alpha via bandwidth (octaves)
const alpha = sinW0 * Math.sinh((Math.LN2 / 2) * bandwidth * omega0 / sinW0);

// For peaking/shelving filters - amplitude from dB gain
const A = Math.pow(10, dBGain / 40);  // For shelves
const A = Math.pow(10, dBGain / 20);  // For peaking EQ
```

### Filter Coefficient Formulas

**Low Pass Filter (LPF)**
```typescript
b0 = (1 - cosW0) / 2;
b1 = 1 - cosW0;
b2 = (1 - cosW0) / 2;
a0 = 1 + alpha;
a1 = -2 * cosW0;
a2 = 1 - alpha;
```

**High Pass Filter (HPF)**
```typescript
b0 = (1 + cosW0) / 2;
b1 = -(1 + cosW0);
b2 = (1 + cosW0) / 2;
a0 = 1 + alpha;
a1 = -2 * cosW0;
a2 = 1 - alpha;
```

**Band Pass Filter (constant 0dB peak)**
```typescript
b0 = alpha;
b1 = 0;
b2 = -alpha;
a0 = 1 + alpha;
a1 = -2 * cosW0;
a2 = 1 - alpha;
```

**Notch Filter**
```typescript
b0 = 1;
b1 = -2 * cosW0;
b2 = 1;
a0 = 1 + alpha;
a1 = -2 * cosW0;
a2 = 1 - alpha;
```

**Peaking EQ**
```typescript
b0 = 1 + alpha * A;
b1 = -2 * cosW0;
b2 = 1 - alpha * A;
a0 = 1 + alpha / A;
a1 = -2 * cosW0;
a2 = 1 - alpha / A;
```

**Low Shelf**
```typescript
const sqrtA = Math.sqrt(A);
b0 = A * ((A + 1) - (A - 1) * cosW0 + 2 * sqrtA * alpha);
b1 = 2 * A * ((A - 1) - (A + 1) * cosW0);
b2 = A * ((A + 1) - (A - 1) * cosW0 - 2 * sqrtA * alpha);
a0 = (A + 1) + (A - 1) * cosW0 + 2 * sqrtA * alpha;
a1 = -2 * ((A - 1) + (A + 1) * cosW0);
a2 = (A + 1) + (A - 1) * cosW0 - 2 * sqrtA * alpha;
```

**High Shelf**
```typescript
const sqrtA = Math.sqrt(A);
b0 = A * ((A + 1) + (A - 1) * cosW0 + 2 * sqrtA * alpha);
b1 = -2 * A * ((A - 1) + (A + 1) * cosW0);
b2 = A * ((A + 1) + (A - 1) * cosW0 - 2 * sqrtA * alpha);
a0 = (A + 1) - (A - 1) * cosW0 + 2 * sqrtA * alpha;
a1 = 2 * ((A - 1) - (A + 1) * cosW0);
a2 = (A + 1) - (A - 1) * cosW0 - 2 * sqrtA * alpha;
```

### Coefficient Normalization

Always normalize by a0:
```typescript
b0 /= a0; b1 /= a0; b2 /= a0;
a1 /= a0; a2 /= a0; a0 = 1;
```

## Dynamics Processing Mathematics

### Envelope Follower (Attack/Release)

The core of compressors, limiters, and gates. Uses a leaky integrator (first-order IIR):

```typescript
// Time constant to coefficient conversion
const attackCoef = Math.exp(-1 / (attackTimeSeconds * sampleRate));
const releaseCoef = Math.exp(-1 / (releaseTimeSeconds * sampleRate));

// Per-sample envelope following
function updateEnvelope(input: number, envelope: number): number {
  const rectified = Math.abs(input);
  if (rectified > envelope) {
    // Attack: envelope rises toward input
    return attackCoef * envelope + (1 - attackCoef) * rectified;
  } else {
    // Release: envelope falls toward input
    return releaseCoef * envelope + (1 - releaseCoef) * rectified;
  }
}
```

### Compressor Gain Calculation

```typescript
// Input level in dB
const inputDb = 20 * Math.log10(Math.max(envelope, 1e-6));

// Gain reduction calculation
let gainReductionDb = 0;
if (inputDb > threshold) {
  // Above threshold: apply ratio
  const overDb = inputDb - threshold;
  gainReductionDb = overDb - (overDb / ratio);
}

// Soft knee (optional)
if (knee > 0 && inputDb > threshold - knee/2 && inputDb < threshold + knee/2) {
  const kneeRange = inputDb - threshold + knee/2;
  gainReductionDb = (kneeRange * kneeRange) / (2 * knee) * (1 - 1/ratio);
}

// Apply gain reduction
const gainLinear = Math.pow(10, -gainReductionDb / 20);
```

### Limiter (Brick-Wall)

A limiter is a compressor with:
- Very fast attack (< 1ms, typically 0.1ms)
- High ratio (100:1 or higher = brick wall)
- Threshold near 0dBFS (-0.3 to -1 dB typical)

```typescript
// Lookahead limiter for true peak limiting
const lookaheadSamples = Math.floor(0.001 * sampleRate); // 1ms lookahead
// Delay input by lookahead, apply gain from envelope detected ahead
```

## Saturation/Distortion Mathematics

### Soft Clipping (tanh)

Adds odd harmonics, musical distortion:

```typescript
// Basic tanh saturation
const driven = input * driveAmount;  // driveAmount typically 1-10
const saturated = Math.tanh(driven);

// Normalize output (tanh reduces level at high drive)
const output = saturated / Math.tanh(driveAmount);
```

### Asymmetric Saturation (Tape Character)

Adds even harmonics (tape warmth):

```typescript
// Add DC offset before waveshaping for asymmetry
const offset = 0.1;  // Experiment with 0.05-0.2
const driven = input * driveAmount + offset;
const saturated = Math.tanh(driven);

// High-frequency rolloff (tape characteristic)
// Apply lowpass/highshelf at 8-12kHz
```

### Polynomial Waveshaping

```typescript
// Soft saturation: f(x) = x - (x³/3) for |x| < 1
function softSaturate(x: number): number {
  if (Math.abs(x) < 1) {
    return x - (x * x * x) / 3;
  }
  return Math.sign(x) * (2/3);  // Clamp to ±2/3
}
```

## Conversion Formulas

### dB Conversions
```typescript
const dB = 20 * Math.log10(linear);      // Linear to dB
const linear = Math.pow(10, dB / 20);    // dB to linear

// For power (squared quantities like energy)
const dB = 10 * Math.log10(power);
const power = Math.pow(10, dB / 10);
```

### Frequency/Musical Conversions
```typescript
// MIDI note to frequency
const freq = 440 * Math.pow(2, (midiNote - 69) / 12);

// Frequency to MIDI note
const midiNote = 69 + 12 * Math.log2(freq / 440);

// Semitones to frequency ratio
const ratio = Math.pow(2, semitones / 12);

// Cents to frequency ratio
const ratio = Math.pow(2, cents / 1200);
```

### Pan Law (Equal Power)
```typescript
// pan: -1 (left) to +1 (right)
const leftGain = Math.cos((pan + 1) * Math.PI / 4);
const rightGain = Math.sin((pan + 1) * Math.PI / 4);
```

## Q Factor and Bandwidth

```typescript
// Q to bandwidth (octaves)
const bandwidth = 2 * Math.asinh(1 / (2 * Q)) / Math.LN2;

// Bandwidth (octaves) to Q
const Q = 1 / (2 * Math.sinh(Math.LN2 / 2 * bandwidth));

// Common Q values:
// Q = 0.707 (Butterworth) - maximally flat passband
// Q = 1.0 - moderate resonance
// Q = 2.0 - narrow, resonant
// Q = 10.0 - very narrow notch/peak
```

## Stability Considerations

For biquad filters, poles must be inside the unit circle:
- Always verify |a2| < 1 and |a1| < 1 + a2
- High Q at low frequencies near Nyquist can cause instability
- Use double precision for coefficient calculation
- Consider cascading lower-order sections for high-order filters

## References

- [Audio EQ Cookbook](https://webaudio.github.io/Audio-EQ-Cookbook/audio-eq-cookbook.html)
- [DAFX: Digital Audio Effects](https://www.dafx.de/)
- [JAES Digital Dynamic Range Compressor Tutorial](https://www.eecs.qmul.ac.uk/~josh/documents/2012/GiannoulisMassbergReiss-dynamicrangecompression-JAES2012.pdf)
