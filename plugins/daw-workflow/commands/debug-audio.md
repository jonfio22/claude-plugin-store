---
name: debug-audio
description: Diagnose and fix audio issues (clicks, pops, silence, mono, latency, dropouts)
---

# Debug Audio Issue: $ARGUMENTS

You are debugging an audio issue in SoundSage. Follow this systematic approach:

## Common Issues and Solutions

### 1. Clicks/Pops/Glitches

**Causes:**
- Discontinuities in signal (abrupt value changes)
- Buffer underruns (DSP too slow)
- Graph structure changes during playback

**Debug Steps:**
```typescript
// Check for abrupt parameter changes
// BAD: Direct assignment causes clicks
gainNode.gain.value = newValue;

// GOOD: Use ramps for smooth transitions
gainNode.gain.setTargetAtTime(newValue, audioContext.currentTime, 0.01);

// In Elementary, use el.smooth() or el.pole() for parameter smoothing
el.smooth(el.tau2pole(0.01), el.const({ key: 'param', value: x }))
```

**Files to Check:**
- `src/audio/EffectsChain.ts` - Look for unsmoothed parameter changes
- `src/audio/ClipSequencer.ts` - Check fade envelopes at clip boundaries

### 2. No Audio / Silence

**Debug Steps:**
1. Check AudioContext state:
```typescript
console.log(audioContext.state);  // Should be 'running'
```

2. Verify Elementary renderer initialized:
```typescript
console.log(renderer.getNode());  // Should return AudioWorkletNode
```

3. Check signal chain connections in browser DevTools

4. Verify buffers are loaded in VFS:
```typescript
// Check if source buffers exist
console.log('Loaded buffers:', bufferIds);
```

**Files to Check:**
- `src/audio/ElementaryRenderer.ts` - Initialization
- Audio engine initialization code
- Transport store - is playback actually started?

### 3. Mono Instead of Stereo

**Causes:**
- channelCount defaulting to 1
- L/R signals mixed before output
- Incorrect Elementary render() call

**Debug Steps:**
```typescript
// Verify stereo configuration
console.log({
  numberOfInputs: audioNode.numberOfInputs,
  numberOfOutputs: audioNode.numberOfOutputs,
  channelCount: audioNode.channelCount,
  channelCountMode: audioNode.channelCountMode,
});

// Should see:
// numberOfInputs: 2 (for stereo input)
// channelCount: 2
// channelCountMode: 'explicit'
```

**Files to Check:**
- `src/audio/ElementaryRenderer.ts` - Check initialization options
- `src/audio/ConvolverReverb.ts` - Check channelCount settings
- `src/audio/DelayEffect.ts` - Check channelCount settings

### 4. High Latency

**Debug Steps:**
```typescript
console.log({
  baseLatency: audioContext.baseLatency,
  outputLatency: audioContext.outputLatency,
  sampleRate: audioContext.sampleRate,
});

// Try lower latency hint
const ctx = new AudioContext({ latencyHint: 'interactive' });
```

**Causes:**
- Default latencyHint is 'balanced' (higher latency)
- Large buffer sizes
- Heavy DSP processing

### 5. Dropouts / Buffer Underruns

**Debug Steps:**
1. Profile DSP timing:
```typescript
const start = performance.now();
renderer.render(left, right);
const elapsed = performance.now() - start;
console.log(`Render time: ${elapsed.toFixed(2)}ms`);
```

2. Check for expensive operations:
- Large number of active tracks
- Many effects enabled
- Complex automation

**Solutions:**
- Reduce number of active effects
- Increase buffer size (trade latency for stability)
- Optimize DSP graph (combine operations, skip bypassed effects)

## Diagnostic Commands

Run these to gather information:

```bash
# Check for TypeScript errors
npm run typecheck

# Check for obvious issues
npm run lint

# Run in development mode to see console logs
npm run dev
```

## Key Files to Examine

1. **Audio Engine:**
   - `src/audio/ElementaryRenderer.ts`
   - `src/audio/EffectsChain.ts`
   - `src/audio/ClipSequencer.ts`

2. **State:**
   - `src/stores/mixerStore.ts`
   - `src/stores/transportStore.ts`

3. **Types:**
   - `src/types/audio.ts`
   - `src/types/index.ts`

## Report Format

After diagnosing, provide:
1. **Root Cause**: What's causing the issue
2. **Location**: File and line number
3. **Solution**: Specific code changes needed
4. **Prevention**: How to avoid this in future
