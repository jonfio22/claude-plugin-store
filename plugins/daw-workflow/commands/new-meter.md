---
name: new-meter
description: Scaffold a new audio meter/analyzer component (LUFS, spectrum, stereo field, etc.)
---

# Create New Audio Meter: $1

You are creating a new audio metering/visualization component for SoundSage.

## 1. Analyze Existing Meters

Read existing implementations in `src/components/meters/`:
- `LUFSMeter.tsx` - Loudness metering with canvas
- `SpectrumAnalyzer.tsx` - FFT-based frequency display
- `StereoField.tsx` - Correlation and balance visualization
- `Spectrogram.tsx` - Time-frequency waterfall

Also check:
- `src/types/audio.ts` - Data types (MeterData, SpectrumData, LUFSData)
- `src/stores/monitoringStore.ts` - Monitoring state

## 2. Define Data Type

If needed, add a new data interface to `src/types/audio.ts`:
```typescript
export interface ${1}Data {
  // Define the data shape for this meter
}
```

## 3. Create the Component

Create `src/components/meters/${1}.tsx`:

```typescript
'use client';

import { useEffect, useRef } from 'react';
import type { ${1}Data } from '@/types';
import { cn } from '@/utils/cn';

interface ${1}Props {
  data: ${1}Data;
  className?: string;
}

export function ${1}({ data, className = '' }: ${1}Props) {
  const canvasRef = useRef<HTMLCanvasElement>(null);
  const rafRef = useRef<number>(0);
  const dataRef = useRef(data);

  // Update data ref without restarting RAF
  useEffect(() => {
    dataRef.current = data;
  }, [data]);

  useEffect(() => {
    const canvas = canvasRef.current;
    if (!canvas) return;

    const ctx = canvas.getContext('2d', { alpha: false });
    if (!ctx) return;

    const updateCanvasSize = () => {
      const rect = canvas.getBoundingClientRect();
      const dpr = window.devicePixelRatio || 1;
      canvas.width = rect.width * dpr;
      canvas.height = rect.height * dpr;
      ctx.scale(dpr, dpr);
    };

    updateCanvasSize();
    window.addEventListener('resize', updateCanvasSize);

    const render = () => {
      const width = canvas.width / (window.devicePixelRatio || 1);
      const height = canvas.height / (window.devicePixelRatio || 1);

      // Clear
      ctx.fillStyle = '#09090b';
      ctx.fillRect(0, 0, width, height);

      // Draw meter using dataRef.current
      // ... visualization code

      rafRef.current = requestAnimationFrame(render);
    };

    rafRef.current = requestAnimationFrame(render);

    return () => {
      window.removeEventListener('resize', updateCanvasSize);
      cancelAnimationFrame(rafRef.current);
    };
  }, []);

  return (
    <div className={cn('relative', className)}>
      <canvas ref={canvasRef} style={{ width: '100%', height: '100%' }} />
    </div>
  );
}
```

## 4. Connect to Data Source

Either:
- Add to monitoringStore if it's a new analysis type
- Connect to existing Elementary meter/scope/fft nodes
- Create new analysis in the audio engine

## 5. Add to Monitoring Panel

Update the monitoring UI to include the new meter.

## 6. Export from Index

Add to `src/components/meters/index.ts`.

## Canvas Best Practices

- Use `alpha: false` for opaque canvases (better performance)
- Handle devicePixelRatio for sharp rendering on Retina displays
- Use refs for data to avoid restarting RAF on every update
- Pre-allocate typed arrays for analysis data
- Use monospace fonts for numerical displays
- Follow existing color scheme (zinc backgrounds, gold accents)

## Performance Guidelines

- Meters run at 60fps - keep render function lightweight
- Avoid creating objects in render loop
- Use typed arrays (Float32Array) for analysis data
- Consider smoothing for visual stability
- Debounce resize handler if needed
