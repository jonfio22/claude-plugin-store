---
name: new-effect
description: Scaffold a new audio effect following SoundSage patterns (type, state, DSP, UI panel)
---

# Create New Audio Effect: $1

You are scaffolding a new audio effect for SoundSage. Follow these steps:

## 1. Analyze Existing Patterns

First, read the existing effect implementations:
- `src/audio/EffectsChain.ts` - How effects are built with Elementary
- `src/stores/mixerStore.ts` - Effect state shape and defaults
- `src/types/index.ts` - Type definitions for effects
- `src/components/fx/` - UI panel components (e.g., CompressorPanel.tsx)

## 2. Define the Effect State Type

Add to `src/types/index.ts` or the appropriate types file:
```typescript
export interface ${1}State {
  enabled: boolean;
  // Add effect-specific parameters
  // Use sensible ranges and defaults
}
```

## 3. Add to TrackFxState

Update the `TrackFxState` interface to include the new effect.

## 4. Create Default State

In `mixerStore.ts`, add default values for the new effect in `defaultTrackMixState.fx`.

## 5. Implement DSP in EffectsChain

Add a `build${1}` method to `EffectsChain` class:
```typescript
private build${1}(
  signal: NodeRepr_t,
  params: ${1}State,
  keyPrefix: string
): NodeRepr_t {
  // Use el.* nodes with proper keying
  // Return processed signal
}
```

Add the effect to the `apply()` method chain in the correct position.

## 6. Create UI Panel Component

Create `src/components/fx/${1}Panel.tsx`:
- Follow the pattern of existing panels (CompressorPanel, EQPanel)
- Use Knob component for rotary controls
- Include enable/bypass toggle
- Connect to mixerStore with proper selectors

## 7. Register in FxRack

Update `src/components/fx/FxRack.tsx` to include the new effect panel.

## 8. Verify

- Run `npm run typecheck` to ensure types are correct
- Run `npm run lint` to check for issues
- Test the effect with audio to verify it sounds correct

## DSP Guidelines

- Always key el.const nodes: `el.const({ key: \`\${keyPrefix}:param\`, value: x })`
- Process L/R channels separately for true stereo
- Use proper unit conversions (dB ↔ linear, seconds ↔ ms)
- Consider CPU efficiency - avoid unnecessary processing
- Handle edge cases (bypass, extreme parameters)
