---
name: state-persistence
description: DAW state management patterns including Zustand stores with Immer, undo/redo history, automation lanes, project serialization, and real-time state synchronization. Use when implementing track state, mixer state, transport state, project save/load, or debugging state issues in SoundSage.
---

# DAW State Management Patterns

You are an expert in state management for DAW applications, with deep knowledge of Zustand, Immer, and real-time audio state synchronization.

## SoundSage Store Architecture

The codebase uses Zustand with Immer middleware for immutable updates:

```typescript
import { create } from 'zustand';
import { immer } from 'zustand/middleware/immer';
import { subscribeWithSelector } from 'zustand/middleware';

// Standard store pattern
export const useMyStore = create<MyStoreState>()(
  subscribeWithSelector(
    immer((set, get) => ({
      // State
      data: {},

      // Actions using Immer draft
      updateData: (id, updates) =>
        set((state) => {
          const item = state.data[id];
          if (item) {
            Object.assign(item, updates);
          }
        }),
    }))
  )
);
```

## Core Stores

### Transport Store

```typescript
interface TransportState {
  isPlaying: boolean;
  position: number;      // Current playhead in seconds
  tempo: number;         // BPM
  timeSignature: { numerator: number; denominator: number };
  loopEnabled: boolean;
  loopStart: number;
  loopEnd: number;
}

// Actions
play: () => set({ isPlaying: true }),
pause: () => set({ isPlaying: false }),
stop: () => set({ isPlaying: false, position: 0 }),
setPosition: (position) => set({ position }),
setTempo: (tempo) => set((state) => { state.tempo = Math.max(20, Math.min(300, tempo)); }),
```

### Mixer Store

```typescript
interface MixerState {
  tracks: Record<string, TrackMixState>;
  master: MasterMixState;
  sends: {
    reverb: ReverbSendState;
    delay: DelaySendState;
  };
}

interface TrackMixState {
  volume: number;      // dB (-60 to +12)
  pan: number;         // -1 (L) to +1 (R)
  mute: boolean;
  solo: boolean;
  fx: TrackFxState;
  sends: { reverb: number; delay: number };  // 0-1
  outputId: string;    // 'master' or bus ID
}

// Clamped setters
setTrackVolume: (trackId, volume) =>
  set((state) => {
    const track = state.tracks[trackId];
    if (track) {
      track.volume = Math.max(-60, Math.min(12, volume));
    }
  }),
```

### Automation Store

```typescript
interface AutomationLane {
  id: string;
  trackId: string;
  parameter: AutomationParameterType;  // 'volume' | 'pan' | 'eq:band0:gain' etc.
  points: AutomationPoint[];
  visible: boolean;
  height: number;
}

interface AutomationPoint {
  id: string;
  time: number;        // Seconds
  value: number;       // Normalized 0-1
  curve: 'linear' | 'exponential' | 'scurve';
}

// Interpolation function
function interpolate(leftValue: number, rightValue: number, t: number, curve: string): number {
  switch (curve) {
    case 'linear':
      return leftValue + (rightValue - leftValue) * t;
    case 'exponential':
      return leftValue + (rightValue - leftValue) * (1 - Math.exp(-4 * t));
    case 'scurve':
      const smoothT = t * t * (3 - 2 * t);  // Hermite smoothstep
      return leftValue + (rightValue - leftValue) * smoothT;
  }
}

// Get value at playhead position
getInterpolatedValue: (laneId, time) => {
  const lane = get().lanes.find(l => l.id === laneId);
  if (!lane || lane.points.length === 0) return null;

  // Find surrounding points
  let leftPoint = null, rightPoint = null;
  for (const point of lane.points) {
    if (point.time <= time) leftPoint = point;
    if (point.time >= time && !rightPoint) rightPoint = point;
  }

  // Edge cases
  if (!leftPoint && rightPoint) return rightPoint.value;
  if (leftPoint && !rightPoint) return leftPoint.value;

  // Interpolate
  const t = (time - leftPoint.time) / (rightPoint.time - leftPoint.time);
  return interpolate(leftPoint.value, rightPoint.value, t, leftPoint.curve);
}
```

### Project Store

```typescript
interface Project {
  id: string;
  name: string;
  tracks: Track[];
  tempo: number;
  timeSignature: { numerator: number; denominator: number };
  createdAt: string;
  updatedAt: string;
}

interface Track {
  id: string;
  name: string;
  color: string;
  clips: Clip[];
  sourceId: string;    // Reference to audio buffer
  muted: boolean;
  solo: boolean;
}

interface Clip {
  id: string;
  startTime: number;   // Position on timeline (seconds)
  duration: number;    // Clip length (seconds)
  offset: number;      // Start offset within source (seconds)
  gain: number;        // Linear gain multiplier
  fadeIn: number;      // Fade in duration (seconds)
  fadeOut: number;     // Fade out duration (seconds)
}
```

## Undo/Redo Pattern

```typescript
interface HistoryState<T> {
  past: T[];
  present: T;
  future: T[];
}

// History store wrapper
export const useHistoryStore = create<HistoryStoreState>()(
  immer((set, get) => ({
    past: [],
    future: [],

    pushState: (snapshot) =>
      set((state) => {
        state.past.push(snapshot);
        state.future = [];  // Clear redo stack on new action
        // Limit history size
        if (state.past.length > 100) {
          state.past.shift();
        }
      }),

    undo: () => {
      const { past, future } = get();
      if (past.length === 0) return null;

      const previous = past[past.length - 1];
      set((state) => {
        state.past.pop();
        state.future.push(getCurrentSnapshot());
      });
      return previous;
    },

    redo: () => {
      const { future } = get();
      if (future.length === 0) return null;

      const next = future[future.length - 1];
      set((state) => {
        state.future.pop();
        state.past.push(getCurrentSnapshot());
      });
      return next;
    },
  }))
);

// Hook for components
function useUndoRedo() {
  const { undo, redo, past, future } = useHistoryStore();
  return {
    undo,
    redo,
    canUndo: past.length > 0,
    canRedo: future.length > 0,
  };
}
```

## Subscription Patterns

### Selective Subscriptions

```typescript
// Subscribe to specific state changes (avoids unnecessary re-renders)
const volume = useMixerStore((state) => state.tracks[trackId]?.volume);

// Subscribe with equality check
const trackIds = useMixerStore(
  (state) => Object.keys(state.tracks),
  (a, b) => a.length === b.length && a.every((id, i) => id === b[i])
);

// External subscription for audio engine
useEffect(() => {
  const unsubscribe = useMixerStore.subscribe(
    (state) => state.tracks,
    (tracks) => {
      // Rebuild audio graph when tracks change
      rebuildAudioGraph(tracks);
    },
    { equalityFn: shallow }
  );
  return unsubscribe;
}, []);
```

### Derived State

```typescript
// Compute derived values efficiently
const soloedTracks = useMemo(() => {
  return Object.entries(tracks)
    .filter(([_, track]) => track.solo)
    .map(([id]) => id);
}, [tracks]);

// Or use selector with computation
const hasSoloedTracks = useMixerStore(
  (state) => Object.values(state.tracks).some(t => t.solo)
);
```

## Project Serialization

### Save Format

```typescript
interface ProjectFile {
  version: string;
  project: Project;
  mixer: {
    tracks: Record<string, TrackMixState>;
    master: MasterMixState;
    sends: SendsState;
  };
  automation: AutomationLane[];
  transport: {
    tempo: number;
    timeSignature: TimeSignature;
    loopEnabled: boolean;
    loopStart: number;
    loopEnd: number;
  };
  // Audio files stored separately (referenced by sourceId)
}

// Serialize
function serializeProject(): ProjectFile {
  return {
    version: '1.0.0',
    project: useProjectStore.getState().currentProject,
    mixer: {
      tracks: useMixerStore.getState().tracks,
      master: useMixerStore.getState().master,
      sends: useMixerStore.getState().sends,
    },
    automation: useAutomationStore.getState().lanes,
    transport: {
      tempo: useTransportStore.getState().tempo,
      // ...
    },
  };
}

// Deserialize
function loadProject(file: ProjectFile) {
  // Version migration if needed
  const migrated = migrateProjectFile(file);

  // Restore stores
  useProjectStore.setState({ currentProject: migrated.project });
  useMixerStore.setState({
    tracks: migrated.mixer.tracks,
    master: migrated.mixer.master,
    sends: migrated.mixer.sends,
  });
  useAutomationStore.setState({ lanes: migrated.automation });
  // ...

  // Clear history
  useHistoryStore.getState().clear();
}
```

### Supabase Persistence

```typescript
// Save to Supabase
async function saveProjectToCloud(projectId: string) {
  const projectData = serializeProject();

  const { error } = await supabase
    .from('projects')
    .upsert({
      id: projectId,
      data: projectData,
      updated_at: new Date().toISOString(),
    });

  if (error) throw error;
}

// Auto-save with debounce
const debouncedSave = debounce(saveProjectToCloud, 5000);

// Subscribe to changes that trigger auto-save
useEffect(() => {
  const unsubscribes = [
    useProjectStore.subscribe(() => debouncedSave(projectId)),
    useMixerStore.subscribe(() => debouncedSave(projectId)),
    useAutomationStore.subscribe(() => debouncedSave(projectId)),
  ];
  return () => unsubscribes.forEach(u => u());
}, [projectId]);
```

## Real-Time Sync with Audio Thread

### Throttled Updates

```typescript
// Throttle high-frequency UI updates (faders, knobs)
function useThrottledValue<T>(value: T, intervalMs: number = 16): T {
  const [throttled, setThrottled] = useState(value);
  const lastUpdate = useRef(0);

  useEffect(() => {
    const now = Date.now();
    if (now - lastUpdate.current >= intervalMs) {
      setThrottled(value);
      lastUpdate.current = now;
    } else {
      const timeout = setTimeout(() => {
        setThrottled(value);
        lastUpdate.current = Date.now();
      }, intervalMs - (now - lastUpdate.current));
      return () => clearTimeout(timeout);
    }
  }, [value, intervalMs]);

  return throttled;
}

// Usage in fader component
const throttledVolume = useThrottledValue(volume, 16);  // ~60fps
useEffect(() => {
  setTrackVolume(trackId, throttledVolume);
}, [throttledVolume]);
```

### Batch State Updates

```typescript
// Batch multiple updates into single render
function batchedUpdate(updates: Array<() => void>) {
  unstable_batchedUpdates(() => {
    updates.forEach(update => update());
  });
}

// Or use Zustand's set with multiple properties
set((state) => {
  state.tracks[trackId].volume = newVolume;
  state.tracks[trackId].pan = newPan;
  state.tracks[trackId].mute = newMute;
});
```

## Default Values Pattern

```typescript
// Centralized defaults for consistency
function createDefaultEQBands(): EQBand[] {
  return [
    { frequency: 80, gain: 0, q: 0.7, type: 'lowshelf' },
    { frequency: 200, gain: 0, q: 1.4, type: 'peaking' },
    { frequency: 500, gain: 0, q: 1.4, type: 'peaking' },
    { frequency: 1000, gain: 0, q: 1.4, type: 'peaking' },
    { frequency: 2500, gain: 0, q: 1.4, type: 'peaking' },
    { frequency: 6000, gain: 0, q: 1.4, type: 'peaking' },
    { frequency: 12000, gain: 0, q: 0.7, type: 'highshelf' },
  ];
}

const defaultTrackMixState: TrackMixState = {
  volume: 0,  // 0 dB (unity gain)
  pan: 0,     // Center
  fx: {
    eq: { enabled: true, bands: createDefaultEQBands() },
    compressor: {
      enabled: false,
      threshold: -20,
      ratio: 4,
      attack: 0.01,
      release: 0.1,
      knee: 6,
      makeupGain: 0,
    },
    saturation: { enabled: false, drive: 0.3, mix: 0.5, type: 'harmonics' },
    limiter: { enabled: false, threshold: -1, release: 0.1 },
  },
  sends: { reverb: 0, delay: 0 },
  outputId: 'master',
};

// Initialize new track with deep clone
initializeTrack: (trackId) =>
  set((state) => {
    state.tracks[trackId] = JSON.parse(JSON.stringify(defaultTrackMixState));
  }),
```

## Debugging State

```typescript
// DevTools integration
import { devtools } from 'zustand/middleware';

export const useMyStore = create<MyState>()(
  devtools(
    immer((set) => ({
      // ...
    })),
    { name: 'MyStore' }
  )
);

// Logging middleware
const logMiddleware = (config) => (set, get, api) =>
  config(
    (...args) => {
      console.log('  previous state:', get());
      set(...args);
      console.log('  next state:', get());
    },
    get,
    api
  );

// State snapshot for debugging
function dumpState() {
  console.log({
    project: useProjectStore.getState(),
    mixer: useMixerStore.getState(),
    automation: useAutomationStore.getState(),
    transport: useTransportStore.getState(),
  });
}
```
