---
name: new-store-action
description: Add a new action to a Zustand store following SoundSage patterns (immer, selectors, subscriptions)
---

# Add Store Action: $ARGUMENTS

You are adding a new action to a Zustand store in SoundSage.

## 1. Identify the Store

Common stores:
- `src/stores/mixerStore.ts` - Track/master mix state, FX
- `src/stores/transportStore.ts` - Playback, tempo, loop
- `src/stores/projectStore.ts` - Project, tracks, clips
- `src/stores/automationStore.ts` - Automation lanes, points
- `src/stores/editingStore.ts` - Selection, editing mode
- `src/stores/uiStore.ts` - UI state

## 2. Define the Action Type

Add to the store's state interface:
```typescript
interface MyStoreState {
  // Existing state...

  // New action
  myNewAction: (param1: string, param2: number) => void;
}
```

## 3. Implement with Immer

```typescript
export const useMyStore = create<MyStoreState>()(
  subscribeWithSelector(
    immer((set, get) => ({
      // Existing state and actions...

      // New action using Immer draft
      myNewAction: (param1, param2) =>
        set((state) => {
          // Mutate state directly (Immer handles immutability)
          state.someValue = param1;
          state.someNumber = Math.max(0, Math.min(100, param2));  // Clamp values

          // Access nested state safely
          const item = state.items[param1];
          if (item) {
            item.property = param2;
          }
        }),

      // For actions that need current state
      complexAction: () => {
        const { someValue, otherValue } = get();
        // Use current values...
        set((state) => {
          state.computed = someValue + otherValue;
        });
      },
    }))
  )
);
```

## 4. Patterns to Follow

### Clamping Values
```typescript
// Always clamp user-controllable values
setVolume: (trackId, volume) =>
  set((state) => {
    const track = state.tracks[trackId];
    if (track) {
      track.volume = Math.max(-60, Math.min(12, volume));  // dB range
    }
  }),
```

### Safe Nested Access
```typescript
updateFx: (trackId, fx, params) =>
  set((state) => {
    const track = state.tracks[trackId];
    if (track?.fx[fx]) {
      Object.assign(track.fx[fx], params);
    }
  }),
```

### Returning Values
```typescript
// Actions can return values via get()
addItem: (item) => {
  const id = generateId();
  set((state) => {
    state.items[id] = item;
  });
  return id;  // Return the generated ID
},
```

### Batch Updates
```typescript
// Multiple state changes in one set() call
resetTrack: (trackId) =>
  set((state) => {
    const track = state.tracks[trackId];
    if (track) {
      track.volume = 0;
      track.pan = 0;
      track.mute = false;
      track.solo = false;
    }
  }),
```

## 5. Usage in Components

```typescript
// Get action from store
const myNewAction = useMyStore((state) => state.myNewAction);

// Or destructure multiple
const { myNewAction, anotherAction } = useMyStore();

// Call action
myNewAction('value', 42);
```

## 6. Testing

Add tests in the corresponding `.test.ts` file:
```typescript
describe('myNewAction', () => {
  it('should update state correctly', () => {
    const { myNewAction } = useMyStore.getState();

    myNewAction('test', 50);

    const state = useMyStore.getState();
    expect(state.someValue).toBe('test');
    expect(state.someNumber).toBe(50);
  });

  it('should clamp values', () => {
    const { myNewAction } = useMyStore.getState();

    myNewAction('test', 150);  // Over max

    expect(useMyStore.getState().someNumber).toBe(100);  // Clamped
  });
});
```

## 7. Verify

```bash
npm run typecheck  # Ensure types are correct
npm run lint       # Check for issues
npm run test       # Run tests
```
