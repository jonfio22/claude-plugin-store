---
name: audio-metering
description: Professional audio metering implementation including LUFS loudness (ITU-R BS.1770), true peak detection, spectrum analysis, stereo field correlation, and VU/PPM metering. Use when implementing meters, loudness normalization, broadcast compliance, or audio analysis visualization.
---

# Professional Audio Metering

You are an expert in broadcast-standard audio metering and loudness measurement.

## LUFS Loudness Metering (ITU-R BS.1770)

LUFS (Loudness Units Full Scale) is the industry standard for loudness measurement.

### Algorithm Overview

1. **K-Weighting Filter** (2-stage pre-filter)
2. **Mean Square Calculation** per channel
3. **Channel Weighting** (surround channels boosted)
4. **Gated Integration** (removes silence/quiet passages)

### K-Weighting Implementation

Two cascaded biquad filters:

**Stage 1: High Shelf (Head modeling)**
```typescript
// Pre-filter coefficients at 48kHz
// These model the acoustic effect of the head as a rigid sphere
const preFilterCoeffs = {
  b0: 1.53512485958697,
  b1: -2.69169618940638,
  b2: 1.19839281085285,
  a1: -1.69065929318241,
  a2: 0.73248077421585,
};
```

**Stage 2: High Pass (RLB weighting)**
```typescript
// Revised Low-frequency B-weighting at 48kHz
const rlbFilterCoeffs = {
  b0: 1.0,
  b1: -2.0,
  b2: 1.0,
  a1: -1.99004745483398,
  a2: 0.99007225036621,
};
```

### Measurement Windows

```typescript
interface LUFSData {
  momentary: number;   // 400ms window, updated every 100ms
  shortTerm: number;   // 3s window, updated every 100ms
  integrated: number;  // Full program length, gated
  truePeak: number;    // Inter-sample peak in dBTP
}

// Momentary loudness calculation
const blockSize = Math.floor(0.4 * sampleRate);  // 400ms
const hopSize = Math.floor(0.1 * sampleRate);    // 100ms overlap

// Short-term loudness
const shortTermBlocks = 30;  // 3s / 100ms = 30 blocks
```

### Gating Algorithm

```typescript
// ITU-R BS.1770-4 two-stage gating
const ABSOLUTE_GATE = -70;  // LUFS - removes silence
const RELATIVE_GATE = -10;  // dB below ungated LKFS

function calculateIntegratedLoudness(blocks: number[]): number {
  // Stage 1: Absolute gate
  const aboveAbsolute = blocks.filter(b => b > ABSOLUTE_GATE);

  // Calculate ungated loudness
  const ungatedLoudness = -0.691 + 10 * Math.log10(
    aboveAbsolute.reduce((sum, b) => sum + Math.pow(10, b / 10), 0) / aboveAbsolute.length
  );

  // Stage 2: Relative gate
  const relativeThreshold = ungatedLoudness + RELATIVE_GATE;
  const aboveRelative = aboveAbsolute.filter(b => b > relativeThreshold);

  // Final integrated loudness
  return -0.691 + 10 * Math.log10(
    aboveRelative.reduce((sum, b) => sum + Math.pow(10, b / 10), 0) / aboveRelative.length
  );
}
```

### Channel Weighting (Surround)

```typescript
// ITU-R BS.1770-4 channel weights
const channelWeights: Record<string, number> = {
  L: 1.0,
  R: 1.0,
  C: 1.0,
  LFE: 0.0,  // LFE excluded from loudness
  Ls: 1.41, // +1.5 dB for surround
  Rs: 1.41,
};
```

## True Peak Detection

True peak detects inter-sample peaks that exceed 0dBFS when reconstructed.

```typescript
// Oversample by 4x and find peak
function calculateTruePeak(samples: Float32Array): number {
  // Use polyphase filter bank or FFT-based upsampling
  const oversampled = upsample4x(samples);

  let peak = 0;
  for (let i = 0; i < oversampled.length; i++) {
    peak = Math.max(peak, Math.abs(oversampled[i]));
  }

  return 20 * Math.log10(peak);  // dBTP
}

// Warning thresholds
const TRUE_PEAK_LIMIT_STREAMING = -1.0;  // dBTP (Spotify, YouTube)
const TRUE_PEAK_LIMIT_BROADCAST = -2.0;  // dBTP (EBU R128)
```

## Target Loudness Levels

| Platform | Target LUFS | True Peak Max |
|----------|-------------|---------------|
| Spotify | -14 LUFS | -1 dBTP |
| Apple Music | -16 LUFS (normalize off) | -1 dBTP |
| YouTube | -14 LUFS | -1 dBTP |
| Broadcast (EBU R128) | -23 LUFS | -1 dBTP |
| Podcast (Apple) | -16 LUFS | -1 dBTP |
| CD/Vinyl | -9 to -12 LUFS | 0 dBFS |

## VU and Peak Metering

### VU Meter (Volume Unit)

Traditional analog meter with ~300ms integration time:

```typescript
// VU meter ballistics
const VU_INTEGRATION_TIME = 0.3;  // 300ms to reach 99%
const vuCoef = 1 - Math.exp(-2.2 / (VU_INTEGRATION_TIME * sampleRate));

function updateVU(input: number, currentVU: number): number {
  const rectified = Math.abs(input);
  return currentVU + vuCoef * (rectified - currentVU);
}

// VU reference: 0 VU = +4 dBu = -18 dBFS (broadcast)
const VU_REFERENCE_DBFS = -18;
```

### PPM (Peak Programme Meter)

Fast attack, slow release:

```typescript
// PPM Type I (BBC/EBU)
const PPM_ATTACK_TIME = 0.005;   // 5ms to -2dB of steady tone
const PPM_RELEASE_TIME = 1.7;   // -24dB in 2.8s

const attackCoef = 1 - Math.exp(-2.2 / (PPM_ATTACK_TIME * sampleRate));
const releaseCoef = 1 - Math.exp(-2.2 / (PPM_RELEASE_TIME * sampleRate));

function updatePPM(input: number, currentPPM: number): number {
  const rectified = Math.abs(input);
  if (rectified > currentPPM) {
    return currentPPM + attackCoef * (rectified - currentPPM);
  } else {
    return currentPPM + releaseCoef * (rectified - currentPPM);
  }
}
```

### Peak Hold

```typescript
// Hold peak for visual reference
const PEAK_HOLD_TIME = 2.0;  // seconds
const peakHoldSamples = Math.floor(PEAK_HOLD_TIME * sampleRate);

class PeakHold {
  private peak = 0;
  private holdCounter = 0;

  update(input: number): number {
    const rectified = Math.abs(input);
    if (rectified >= this.peak) {
      this.peak = rectified;
      this.holdCounter = peakHoldSamples;
    } else if (this.holdCounter > 0) {
      this.holdCounter--;
    } else {
      this.peak *= 0.9999;  // Slow decay after hold
    }
    return this.peak;
  }
}
```

## Spectrum Analysis

### FFT-Based Analyzer

```typescript
interface SpectrumData {
  frequencies: Float32Array;  // Center frequencies per bin
  magnitudes: Float32Array;   // Magnitude in dB
}

function analyzeSpectrum(samples: Float32Array, fftSize: number = 2048): SpectrumData {
  // Apply window function (Blackman-Harris for low sidelobes)
  const windowed = applyBlackmanHarris(samples);

  // FFT
  const fft = performFFT(windowed);

  // Convert to dB
  const magnitudes = new Float32Array(fftSize / 2);
  for (let i = 0; i < fftSize / 2; i++) {
    const real = fft.real[i];
    const imag = fft.imag[i];
    const magnitude = Math.sqrt(real * real + imag * imag) / fftSize;
    magnitudes[i] = 20 * Math.log10(Math.max(magnitude, 1e-10));
  }

  return { frequencies, magnitudes };
}

// Smoothing for visual display
function smoothSpectrum(current: Float32Array, previous: Float32Array, smoothing: number = 0.8): Float32Array {
  const result = new Float32Array(current.length);
  for (let i = 0; i < current.length; i++) {
    result[i] = smoothing * previous[i] + (1 - smoothing) * current[i];
  }
  return result;
}
```

### Frequency Scale Options

```typescript
// Linear frequency scale
const linearFreq = (bin: number, fftSize: number, sampleRate: number) =>
  bin * sampleRate / fftSize;

// Logarithmic frequency scale (musical)
const logFreqBins = (minFreq: number, maxFreq: number, numBins: number): number[] => {
  const bins: number[] = [];
  const logMin = Math.log10(minFreq);
  const logMax = Math.log10(maxFreq);
  for (let i = 0; i < numBins; i++) {
    bins.push(Math.pow(10, logMin + (logMax - logMin) * i / (numBins - 1)));
  }
  return bins;
};

// 1/3 octave bands (ISO standard)
const thirdOctaveBands = [
  20, 25, 31.5, 40, 50, 63, 80, 100, 125, 160, 200, 250, 315, 400, 500,
  630, 800, 1000, 1250, 1600, 2000, 2500, 3150, 4000, 5000, 6300, 8000,
  10000, 12500, 16000, 20000
];
```

## Stereo Field Analysis

### Correlation Meter

```typescript
// Pearson correlation coefficient between L and R
function calculateCorrelation(left: Float32Array, right: Float32Array): number {
  let sumL = 0, sumR = 0, sumLR = 0, sumL2 = 0, sumR2 = 0;
  const n = left.length;

  for (let i = 0; i < n; i++) {
    sumL += left[i];
    sumR += right[i];
    sumLR += left[i] * right[i];
    sumL2 += left[i] * left[i];
    sumR2 += right[i] * right[i];
  }

  const numerator = n * sumLR - sumL * sumR;
  const denominator = Math.sqrt((n * sumL2 - sumL * sumL) * (n * sumR2 - sumR * sumR));

  return denominator === 0 ? 0 : numerator / denominator;
}

// Correlation interpretation:
// +1.0 = Mono (L = R)
// 0.0 = Uncorrelated (independent)
// -1.0 = Out of phase (L = -R) - PROBLEM!
```

### Balance Meter

```typescript
// Stereo balance (-1 = full left, +1 = full right)
function calculateBalance(left: Float32Array, right: Float32Array): number {
  let sumL = 0, sumR = 0;

  for (let i = 0; i < left.length; i++) {
    sumL += left[i] * left[i];
    sumR += right[i] * right[i];
  }

  const rmsL = Math.sqrt(sumL / left.length);
  const rmsR = Math.sqrt(sumR / right.length);
  const total = rmsL + rmsR;

  return total === 0 ? 0 : (rmsR - rmsL) / total;
}
```

### Goniometer / Lissajous Display

```typescript
// Convert stereo to M/S for goniometer
function stereoToMS(left: number, right: number): { mid: number; side: number } {
  return {
    mid: (left + right) / Math.SQRT2,   // Mono sum
    side: (left - right) / Math.SQRT2,  // Stereo difference
  };
}

// Plot as X/Y where X=Side, Y=Mid
// Vertical line = mono, horizontal = out of phase, diagonal = stereo
```

## SoundSage Metering Implementation

Based on the codebase patterns:

```typescript
interface LUFSData {
  momentary: number;   // 400ms
  shortTerm: number;   // 3s
  integrated: number;  // Full session
  truePeak: number;    // dBTP
}

// Canvas-based meter rendering (LUFSMeter.tsx pattern)
function renderLUFSMeter(ctx: CanvasRenderingContext2D, lufs: LUFSData) {
  // LUFS scale: -60 to 0
  const minLUFS = -60;
  const maxLUFS = 0;

  const lufsToY = (value: number): number => {
    const clamped = Math.max(minLUFS, Math.min(maxLUFS, value));
    return 1 - (clamped - minLUFS) / (maxLUFS - minLUFS);
  };

  // Draw target lines
  const streamingTarget = lufsToY(-14);  // Spotify/YouTube
  const podcastTarget = lufsToY(-16);    // Apple Podcasts

  // Warning: True peak > -1 dBTP
  if (lufs.truePeak > -1) {
    // Show warning indicator
  }
}
```

## References

- [ITU-R BS.1770-5](https://www.itu.int/rec/R-REC-BS.1770)
- [EBU R 128](https://tech.ebu.ch/loudness)
- [pyloudnorm](https://github.com/csteinmetz1/pyloudnorm) - Python reference
- [LUFSMeter](https://github.com/klangfreund/LUFSMeter) - JUCE reference
