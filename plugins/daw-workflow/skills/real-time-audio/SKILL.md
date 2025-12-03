---
name: real-time-audio
description: Real-time audio programming constraints, Web Audio API architecture, latency management, plugin delay compensation (PDC), sample-accurate scheduling, buffer management, and audio thread safety. Use when debugging audio glitches, implementing playback, optimizing performance, or dealing with timing issues.
---

# Real-Time Audio Programming

You are an expert in real-time audio programming with deep knowledge of Web Audio API, latency compensation, and audio thread constraints.

## Fundamental Constraints

### The Audio Thread

The audio callback runs on a dedicated real-time thread with strict timing requirements:

```typescript
// Block size and timing
const sampleRate = 48000;
const blockSize = 128;  // Web Audio default render quantum

// Time per block
const blockTimeMs = (blockSize / sampleRate) * 1000;  // ~2.67ms at 48kHz

// This means you have ~2.67ms to process each block
// Miss this deadline = audio glitch (click, pop, dropout)
```

### What You CANNOT Do in Audio Thread

```typescript
// NEVER in audio callback:
// - Allocate memory (new, malloc, push to arrays)
// - Lock mutexes (can cause priority inversion)
// - I/O operations (file, network, console.log)
// - Garbage collection triggers
// - DOM manipulation
// - setTimeout/setInterval callbacks

// Elementary Audio handles this by running DSP in AudioWorklet
// Your JavaScript describes the graph; Elementary executes it
```

## Web Audio API Architecture

### Audio Context Lifecycle

```typescript
// Always start suspended and resume on user interaction
const audioContext = new AudioContext();

// Browser requires user gesture to start audio
button.addEventListener('click', async () => {
  if (audioContext.state === 'suspended') {
    await audioContext.resume();
  }
});

// Handle visibility changes (tab switching)
document.addEventListener('visibilitychange', () => {
  if (document.hidden) {
    // Consider suspending to save CPU
    audioContext.suspend();
  } else {
    audioContext.resume();
  }
});
```

### Latency Properties

```typescript
// Base latency: processing latency (DAC buffer)
const baseLatency = audioContext.baseLatency;  // seconds

// Output latency: time to reach speakers
const outputLatency = audioContext.outputLatency;  // seconds

// Total latency
const totalLatency = baseLatency + outputLatency;

// Request low latency (may not be honored)
const lowLatencyContext = new AudioContext({
  latencyHint: 'interactive',  // or 'playback', 'balanced', or seconds
});

// Typical values:
// 'interactive': ~0.01-0.02s (10-20ms)
// 'balanced': ~0.04-0.08s
// 'playback': ~0.1-0.2s
```

### Sample Rate Handling

```typescript
// Get actual sample rate (may differ from requested)
const actualSampleRate = audioContext.sampleRate;

// Common rates: 44100, 48000, 96000
// Always use actual rate for timing calculations
const samplesPerSecond = actualSampleRate;
const secondsPerSample = 1 / actualSampleRate;

// Convert time to samples
const timeToSamples = (seconds: number) => Math.floor(seconds * actualSampleRate);
const samplesToTime = (samples: number) => samples / actualSampleRate;
```

## Latency Compensation

### Recording Latency Compensation

```typescript
// When recording, compensate for round-trip latency
const inputLatency = audioContext.baseLatency;
const outputLatency = audioContext.outputLatency;
const roundTripLatency = inputLatency + outputLatency;

// Shift recorded audio backwards by round-trip latency
function alignRecordedClip(recordedBuffer: AudioBuffer, startTime: number): Clip {
  return {
    startTime: startTime - roundTripLatency,
    buffer: recordedBuffer,
  };
}
```

### Plugin Delay Compensation (PDC)

Some effects introduce latency (e.g., lookahead limiters, linear-phase EQ):

```typescript
interface ProcessorWithLatency {
  process(input: Float32Array): Float32Array;
  latencySamples: number;  // Samples of delay introduced
}

// Track total latency through effects chain
function calculateChainLatency(processors: ProcessorWithLatency[]): number {
  return processors.reduce((total, p) => total + p.latencySamples, 0);
}

// Compensate by delaying other tracks
function applyPDC(tracks: Track[]): void {
  // Find maximum latency
  const maxLatency = Math.max(...tracks.map(t => t.chainLatency));

  // Delay each track by (maxLatency - itsLatency)
  tracks.forEach(track => {
    track.pdcDelaySamples = maxLatency - track.chainLatency;
  });
}
```

### Lookahead Processing

```typescript
// Lookahead limiter: analyze future samples before they play
const lookaheadMs = 5;  // 5ms lookahead
const lookaheadSamples = Math.ceil(lookaheadMs * sampleRate / 1000);

class LookaheadLimiter {
  private buffer: Float32Array;
  private writeIndex = 0;

  constructor(lookaheadSamples: number) {
    this.buffer = new Float32Array(lookaheadSamples);
  }

  process(input: number): number {
    // Read delayed sample
    const delayed = this.buffer[this.writeIndex];

    // Look ahead to find peak in buffer
    const futurePeak = this.findPeakInBuffer();

    // Calculate gain reduction based on future peak
    const gainReduction = this.calculateGain(futurePeak);

    // Write input to buffer
    this.buffer[this.writeIndex] = input;
    this.writeIndex = (this.writeIndex + 1) % this.buffer.length;

    // Return delayed + gain-reduced sample
    return delayed * gainReduction;
  }
}
```

## Sample-Accurate Scheduling

### Web Audio Scheduling

```typescript
// Schedule events with audioContext.currentTime
const now = audioContext.currentTime;

// Start a sound at precise future time
source.start(now + 0.5);  // Start in 500ms

// Schedule parameter changes
gainNode.gain.setValueAtTime(0, now);
gainNode.gain.linearRampToValueAtTime(1, now + 0.1);  // 100ms fade in

// NEVER use setTimeout for audio timing!
// setTimeout(() => source.start(), 500);  // BAD - jittery, inaccurate
```

### Transport Synchronization

```typescript
class Transport {
  private startTime: number = 0;
  private pauseTime: number = 0;

  play() {
    // Record when playback started (audio time, not Date.now())
    this.startTime = audioContext.currentTime - this.pauseTime;
  }

  pause() {
    this.pauseTime = this.getPosition();
  }

  getPosition(): number {
    return audioContext.currentTime - this.startTime;
  }

  // Convert transport position to audio context time
  positionToAudioTime(position: number): number {
    return this.startTime + position;
  }
}
```

### Clip Scheduling

```typescript
// Schedule clip playback sample-accurately
function scheduleClip(clip: Clip, transport: Transport) {
  const source = audioContext.createBufferSource();
  source.buffer = clip.buffer;

  // Calculate when clip should start in audio context time
  const clipStartAudioTime = transport.positionToAudioTime(clip.startTime);

  // If clip start is in the past, offset into the buffer
  const now = audioContext.currentTime;
  if (clipStartAudioTime < now) {
    const offsetSeconds = now - clipStartAudioTime;
    source.start(now, offsetSeconds + clip.offset);
  } else {
    source.start(clipStartAudioTime, clip.offset);
  }

  // Schedule stop
  const clipEndAudioTime = clipStartAudioTime + clip.duration;
  source.stop(clipEndAudioTime);
}
```

## Buffer Management

### Pre-buffering Strategy

```typescript
// Schedule audio blocks ahead of time
const SCHEDULE_AHEAD_TIME = 0.1;  // 100ms lookahead
const SCHEDULE_INTERVAL = 0.025;  // Check every 25ms

class Scheduler {
  private nextBlockTime: number = 0;

  scheduleAhead() {
    while (this.nextBlockTime < audioContext.currentTime + SCHEDULE_AHEAD_TIME) {
      this.scheduleBlock(this.nextBlockTime);
      this.nextBlockTime += BLOCK_DURATION;
    }
  }

  start() {
    this.nextBlockTime = audioContext.currentTime;
    setInterval(() => this.scheduleAhead(), SCHEDULE_INTERVAL * 1000);
  }
}
```

### Memory-Efficient Buffer Pools

```typescript
// Reuse buffers to avoid allocation in hot path
class BufferPool {
  private pool: Float32Array[] = [];
  private size: number;

  constructor(bufferSize: number, poolSize: number = 10) {
    this.size = bufferSize;
    for (let i = 0; i < poolSize; i++) {
      this.pool.push(new Float32Array(bufferSize));
    }
  }

  acquire(): Float32Array {
    return this.pool.pop() ?? new Float32Array(this.size);
  }

  release(buffer: Float32Array) {
    buffer.fill(0);  // Clear
    this.pool.push(buffer);
  }
}
```

## Elementary Audio Specifics

### Graph Rendering Timing

```typescript
// Elementary re-renders graph on each render() call
// This is lightweight IF the graph structure hasn't changed

// Efficient: only values change (same graph structure)
el.const({ key: 'volume', value: newVolume })

// Inefficient: different graph structure each render
// (adding/removing nodes causes full reconciliation)
if (effectEnabled) {
  return el.compress(...);  // Different structure than...
} else {
  return signal;  // ...this path
}

// Better: always include node, control with parameter
return el.select(
  el.const({ key: 'bypass', value: effectEnabled ? 0 : 1 }),
  signal,  // Bypassed signal
  el.compress(...)  // Processed signal
);
```

### Virtual File System Timing

```typescript
// VFS updates are async - don't block on them
await renderer.updateVirtualFileSystem({
  [bufferId]: audioData
});

// Elementary will fire 'load' event if buffer is needed but not ready
renderer.on('load', async (event) => {
  // Load buffer on demand
  const buffer = await loadAudioFile(event.source);
  return { buffer };
});
```

## Performance Optimization

### Profiling Audio

```typescript
// Measure DSP cost
const startTime = performance.now();
// ... DSP operations
const endTime = performance.now();
const processingTimeMs = endTime - startTime;

// Compare to deadline
const blockDeadlineMs = (128 / sampleRate) * 1000;
const cpuUsage = processingTimeMs / blockDeadlineMs;
console.log(`CPU: ${(cpuUsage * 100).toFixed(1)}%`);

// Warning threshold: > 80% CPU means risk of dropouts
```

### SIMD and Optimization

```typescript
// Process in blocks for cache efficiency
function processBlock(input: Float32Array, output: Float32Array) {
  const len = input.length;
  for (let i = 0; i < len; i++) {
    output[i] = processSample(input[i]);
  }
}

// Unroll loops for SIMD-like behavior (JIT will optimize)
function processBlockUnrolled(input: Float32Array, output: Float32Array) {
  const len = input.length;
  for (let i = 0; i < len; i += 4) {
    output[i] = processSample(input[i]);
    output[i + 1] = processSample(input[i + 1]);
    output[i + 2] = processSample(input[i + 2]);
    output[i + 3] = processSample(input[i + 3]);
  }
}
```

### Avoiding Garbage Collection

```typescript
// Pre-allocate working buffers
const workBuffer = new Float32Array(BLOCK_SIZE);

// Reuse objects instead of creating new ones
const reusableResult = { left: 0, right: 0 };

function process(): { left: number; right: number } {
  reusableResult.left = computeLeft();
  reusableResult.right = computeRight();
  return reusableResult;  // Same object each time
}
```

## Debugging Audio Issues

### Common Problems

```typescript
// Clicks/pops: discontinuities in signal
// - Check for proper fade-ins/outs at buffer boundaries
// - Ensure smooth parameter changes (use ramps, not direct sets)

// Silence: signal not reaching output
// - Verify audio context is running: audioContext.state === 'running'
// - Check node connections
// - Verify buffers have data (not all zeros)

// Mono instead of stereo:
// - Check channelCount and channelCountMode on all nodes
// - Ensure stereo buffers are properly loaded
// - Verify L/R processing paths are separate

// High latency:
// - Check latencyHint setting
// - Profile DSP to ensure it's fast enough
// - Reduce buffer size if stable
```

### Debugging Tools

```typescript
// Visualize audio context graph
console.log(audioContext.destination);

// Monitor for errors
audioContext.addEventListener('statechange', () => {
  console.log('AudioContext state:', audioContext.state);
});

// Analyser node for visualization
const analyser = audioContext.createAnalyser();
analyser.fftSize = 2048;
const dataArray = new Uint8Array(analyser.frequencyBinCount);
analyser.getByteFrequencyData(dataArray);
```

## References

- [Web Audio API MDN](https://developer.mozilla.org/en-US/docs/Web/API/Web_Audio_API)
- [Web Audio Timing](https://www.html5rocks.com/en/tutorials/audio/scheduling/)
- [W3C Workshop: Audio Latency in Browser DAWs](https://www.w3.org/2021/03/media-production-workshop/talks/ulf-hammarqvist-audio-latency.html)
- [Paul Adenot's Web Audio Perf Notes](https://padenot.github.io/web-audio-perf/)
