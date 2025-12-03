---
name: audio-check
description: Run full audio system verification (typecheck, lint, and audio-specific validations)
---

# Audio System Verification

Running comprehensive checks on the SoundSage audio system.

## 1. Type Checking

Run TypeScript compiler to catch type errors:
```bash
npm run typecheck
```

Look especially for errors in:
- `src/audio/` - Audio engine files
- `src/stores/` - State management
- `src/types/` - Type definitions

## 2. Linting

Run ESLint to catch code quality issues:
```bash
npm run lint
```

## 3. Audio-Specific Validations

Check these critical patterns:

### Elementary Audio Keying
Search for unkeyed constants (potential performance issue):
```bash
# Look for el.const without key
grep -r "el\.const({" src/audio/ | grep -v "key:"
```

### Stereo Processing
Verify L/R channels are processed separately:
```bash
# Should see paired L/R processing
grep -r ":l\`\|:r\`" src/audio/
```

### dB Conversions
Verify correct dB ↔ linear conversions:
```bash
# Should use Math.pow(10, dB/20) for amplitude
grep -r "Math.pow(10" src/audio/
```

### Channel Configuration
Check for proper stereo setup:
```bash
grep -r "channelCount" src/audio/
```

## 4. Test Audio

If tests exist, run them:
```bash
npm run test:run
```

## 5. Build Check

Verify production build succeeds:
```bash
npm run build
```

## Report Issues

After running checks, report:
1. **TypeScript Errors**: File, line, error message
2. **Lint Warnings**: Potential issues
3. **Pattern Violations**: Any audio anti-patterns found
4. **Recommendations**: Suggested improvements

## Quick Fix Common Issues

### Missing Keys on el.const
```typescript
// Before (bad)
el.const({ value: x })

// After (good)
el.const({ key: `${prefix}:paramName`, value: x })
```

### Incorrect dB Conversion
```typescript
// Before (wrong - power formula)
Math.pow(10, dB / 10)

// After (correct - amplitude formula)
Math.pow(10, dB / 20)
```

### Unthrottled Parameter Updates
```typescript
// Before (causes excessive renders)
onChange={(e) => setVolume(trackId, e.target.value)}

// After (throttled)
const throttledSetVolume = useThrottledCallback(
  (v) => setVolume(trackId, v),
  16
);
onChange={(e) => throttledSetVolume(e.target.value)}
```
