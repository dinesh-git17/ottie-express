---
name: corehaptics-pattern-design
description: Enforce production-grade CoreHaptics pattern architecture for iOS haptic feedback systems. Use when writing or reviewing CHHapticEngine lifecycle code, CHHapticPattern construction, progressive haptic sequences, audio-haptic synchronization, animation-haptic coordination, or UIFeedbackGenerator fallback strategies. Triggers on CoreHaptics code generation, haptic pattern design, engine lifecycle management, sensory feedback implementation, or any task producing CHHapticEvent/CHHapticPattern output.
---

# CoreHaptics Pattern Design

Enforce deterministic, production-grade haptic systems using Apple CoreHaptics. Every haptic moment must be architecturally correct, perceptually intentional, and gracefully degradable.

## Sensory Design Philosophy

Three principles govern all haptic output (Apple HIG, WWDC19-223):

1. **Causality.** Every haptic event must have an unambiguous trigger. Users must instantly understand what caused the feedback.
2. **Harmony.** Haptic, audio, and visual channels must be temporally synchronized. Misalignment by even 20ms destroys believability.
3. **Utility.** Restraint over saturation. Add haptic feedback only when it delivers informational or experiential value. Overuse causes sensory fatigue and user opt-out.

## 1. CHHapticEngine Lifecycle

See [references/engine-lifecycle.md](references/engine-lifecycle.md) for full API detail.

### Mandatory Rules

- **Capability gate first.** Always check `CHHapticEngine.capabilitiesForHardware().supportsHaptics` before engine creation.
- **Single retained instance.** One `CHHapticEngine` per logical haptic context (e.g., one per view controller or coordinator managing haptic output). Never create an engine per event.
- **Set `playsHapticsOnly = true`** when patterns contain no audio events. This eliminates audio hardware initialization latency entirely.
- **Set `isAutoShutdownEnabled = true`** for intermittent-use contexts (forms, authentication flows). The engine auto-stops after ~2 minutes of inactivity and transparently restarts on next playback.
- **Disable auto-shutdown** for continuous-interaction contexts (games, drawing canvases). Manage lifecycle explicitly via scene/view appearance.
- **Start asynchronously.** Use the completion-handler variant of `start()`. Never block the main thread on engine startup.

### Interruption Handling

- **`stoppedHandler`**: Fires on a non-main queue. Switch on `CHHapticEngine.StoppedReason`. Set an internal `engineNeedsRestart` flag. Do not attempt immediate restart from within the handler for `.applicationSuspended` — restart when the app returns to foreground.
- **`resetHandler`**: Fires when the haptic server crashes. **All existing pattern players are invalidated.** The handler must: (1) restart the engine, (2) recreate all retained players from stored `CHHapticPattern` instances. Store patterns, not players.

### Prohibited

- Creating a new `CHHapticEngine` per haptic event.
- Calling `start()` and `stop()` in rapid succession (engine thrashing).
- Ignoring `resetHandler` — this guarantees silent failure after server recovery.
- Using the synchronous `startAndReturnError()` on the main thread.

## 2. Event Modeling

### Transient vs Continuous

| Type       | API                 | Use Case                               | Duration                  |
| ---------- | ------------------- | -------------------------------------- | ------------------------- |
| Transient  | `.hapticTransient`  | Button taps, collisions, confirmations | System-controlled (~30ms) |
| Continuous | `.hapticContinuous` | Sustained feedback, textures, progress | Explicit, max 30s         |

### Parameter Space

| Parameter          | Range   | Perceptual Effect                                          |
| ------------------ | ------- | ---------------------------------------------------------- |
| `.hapticIntensity` | 0.0–1.0 | Vibration amplitude (how strong)                           |
| `.hapticSharpness` | 0.0–1.0 | Tactile character: 0 = round/organic, 1 = crisp/mechanical |
| `.attackTime`      | 0.0–1.0 | Ramp-up speed at event start                               |
| `.decayTime`       | 0.0–1.0 | Intensity drop-off after peak                              |
| `.releaseTime`     | 0.0–1.0 | Fade-out at event end                                      |

### Mandatory Rules

- **Minimum inter-event spacing: 40ms.** Below this, transient events merge perceptually into a single buzz. For distinct rhythmic beats, use 80–150ms spacing.
- **Ghost haptic priming.** The first event in a rapid transient sequence acts as a subconscious primer — four closely-spaced taps feel like three perceived beats. Account for this in rhythmic patterns.
- **No flat sustained continuous events.** Constant intensity causes rapid sensory adaptation. Always apply a parameter curve (ramp, decay, or modulation) to continuous events.
- **Max practical continuous duration: 5 seconds.** Beyond this, perception degrades sharply regardless of intensity. Use intermittent transient bursts for longer feedback sequences.
- **Vary both dimensions.** Modulate sharpness alongside intensity to maintain perceptual novelty. Monotonous stimuli adapt fastest.

### Layering

Events at overlapping `relativeTime` values mix additively. Use layering to build complex tactile textures:

```swift
// Base rumble + sharp accent overlay
let base = CHHapticEvent(eventType: .hapticContinuous, parameters: [
    CHHapticEventParameter(parameterID: .hapticIntensity, value: 0.3),
    CHHapticEventParameter(parameterID: .hapticSharpness, value: 0.1)
], relativeTime: 0, duration: 1.5)

let accent = CHHapticEvent(eventType: .hapticTransient, parameters: [
    CHHapticEventParameter(parameterID: .hapticIntensity, value: 1.0),
    CHHapticEventParameter(parameterID: .hapticSharpness, value: 0.9)
], relativeTime: 0.5)
```

## 3. Pattern Construction

See [references/pattern-architecture.md](references/pattern-architecture.md) for full construction reference.

### Mandatory Rules

- **Named constants for all tuning values.** No magic numbers for intensity, sharpness, timing, or duration. Define constants at the call site or in a dedicated haptic constants namespace.
- **Parameter curves over dynamic parameters for pre-authored sequences.** `CHHapticParameterCurve` provides smooth interpolation. `CHHapticDynamicParameter` provides abrupt step-function changes. Use curves for designed patterns; use dynamic parameters for runtime-driven feedback.
- **Control parameter IDs are multipliers.** `.hapticIntensityControl` and `.hapticSharpnessControl` multiply the base event values. A control value of 0.5 halves the event's intensity. Design base events at full range, then shape with curves.
- **16 control points per curve segment.** Points beyond 16 in a single `CHHapticParameterCurve` are silently ignored. Chain multiple curve entries with sequential time offsets for longer envelopes.
- **Store `CHHapticPattern` instances, not players.** Players are invalidated on engine reset. Patterns are stable data that survive lifecycle events.

### Progressive Sequence Architecture

For progressive feedback (e.g., fingerprint scan, voice recognition):

```
Phase 1: Initiation    — Low intensity transient. Signal start.
Phase 2: Processing    — Continuous with rising intensity curve. Build anticipation.
Phase 3: Confirmation  — Sharp transient burst at peak. Signal completion.
Phase 4: Decay         — Continuous fade-out. Graceful resolution.
```

Each phase must be a discrete event or event group with explicit timing. Do not rely on a single continuous event with complex curves — split into phase-aligned events for maintainability and seek compatibility.

### AHAP vs Programmatic

| Use AHAP Files                                      | Use Programmatic Construction                        |
| --------------------------------------------------- | ---------------------------------------------------- |
| Static, pre-designed patterns                       | Patterns driven by runtime state                     |
| Designer-authored, iterable without recompilation   | Calculated parameters (distance, velocity, progress) |
| Pattern library catalogs                            | Conditional event inclusion                          |
| Sharing with design tools (Haptrix, Haptics Studio) | Algorithmically generated sequences                  |

## 4. Timeline Synchronization

See [references/timeline-sync.md](references/timeline-sync.md) for implementation patterns.

### Mandatory Rules

- **Use `CHHapticTimeImmediate` for user-triggered events.** Taps, presses, and direct interactions demand minimum latency.
- **Use scheduled timestamps for animation-synced events.** Compute target time as `engine.currentTime + desiredOffset`. The engine clock is independent of `CACurrentMediaTime()` — compute the delta once and reuse.
- **10–20ms lookahead for display-link-driven haptics.** When driving haptic parameters from `CADisplayLink`, schedule the parameter update 10–20ms ahead of `engine.currentTime` to compensate for processing latency between the callback and actuator response.
- **`sendParameters(_:atTime:)` for real-time sync.** Send `CHHapticDynamicParameter` arrays to a running player to modulate intensity/sharpness in response to animation progress, audio amplitude, or gesture state. Thread-safe — callable from any queue.
- **Split long continuous events for seek compatibility.** `CHHapticAdvancedPatternPlayer.seek(toOffset:)` cannot resume mid-continuous-event. Break long sustains into a series of shorter continuous events (250–500ms each) so seek lands on the nearest event boundary.

### Audio-Haptic Coordination

When pairing haptics with audio:

- Initialize the engine with a shared `AVAudioSession`: `CHHapticEngine(audioSession:)`. This ensures the same clock domain.
- For audio-reactive haptics, install a tap on `AVAudioEngine.mainMixerNode`, compute RMS or peak amplitude, and feed the result as a `CHHapticDynamicParameter` to the haptic player.
- Audio events (`.audioContinuous`, `.audioCustom`) can be embedded directly in `CHHapticPattern` for sample-accurate synchronization within a single pattern player.

### Animation-Haptic Coordination

- Map animation phase boundaries (start, midpoint, completion) to haptic event `relativeTime` values within the same pattern.
- For SwiftUI `.sensoryFeedback` (iOS 17+), bind the trigger to the same state driving the animation. This guarantees frame-level synchronization via SwiftUI's transaction system.
- For UIKit animations with explicit completion handlers, schedule the confirmation haptic from the completion block, not from estimated duration.

## 5. Fallback Strategy

See [references/fallback-strategy.md](references/fallback-strategy.md) for mapping tables.

### Mandatory Rules

- **Branch once at initialization, not per event.** Check `CHHapticEngine.capabilitiesForHardware().supportsHaptics` once. Instantiate a protocol-conforming player (rich or fallback). All call sites use the protocol interface.
- **Map by semantic intent, not waveform shape.** A "success confirmation" maps to `UINotificationFeedbackGenerator.notificationOccurred(.success)`, regardless of how complex the CoreHaptics version is.
- **Call `prepare()` on UIKit generators 1–2 seconds before expected use.** The prepared state warms the Taptic Engine and expires after a few seconds.
- **Never fire both CoreHaptics and UIKit haptics for the same event.** Mutual exclusion is enforced by the protocol branching, not by runtime checks.
- **No silent failures.** If CoreHaptics is available but engine creation fails, fall back to UIKit generators rather than producing no output.

### Fallback Mapping

| Semantic Intent      | CoreHaptics                                | UIKit Fallback                                    |
| -------------------- | ------------------------------------------ | ------------------------------------------------- |
| Success confirmation | Multi-event ascending pattern              | `.notificationOccurred(.success)`                 |
| Warning              | Dual transient with moderate sharpness     | `.notificationOccurred(.warning)`                 |
| Error                | Triple transient with descending intensity | `.notificationOccurred(.error)`                   |
| Light tap            | Transient, intensity ~0.3                  | `UIImpactFeedbackGenerator(style: .light)`        |
| Medium impact        | Transient, intensity ~0.6                  | `UIImpactFeedbackGenerator(style: .medium)`       |
| Heavy impact         | Transient, intensity ~1.0                  | `UIImpactFeedbackGenerator(style: .heavy)`        |
| Crisp selection      | Transient, intensity ~0.15                 | `UISelectionFeedbackGenerator.selectionChanged()` |
| Progressive scan     | Multi-phase continuous + transient         | Series of timed `.impactOccurred()` at intervals  |

## 6. Performance Constraints

- **Reuse pattern players** for patterns that repeat (taps, collisions). Create once, call `start(atTime:)` repeatedly.
- **Limit concurrent active players to fewer than 10.** Each active player consumes haptic server resources.
- **`engine.playPattern(from:)` is a convenience method that creates and discards a player.** Acceptable for one-off patterns. Inefficient for repeated use.
- **Engine creation and player creation can occur on any thread** but prefer the main thread or a dedicated serial queue.
- **`stoppedHandler` and `resetHandler` fire on an internal non-main queue.** Dispatch to `DispatchQueue.main` for any UI state updates.
- **Continuous events at full intensity for extended periods measurably impact battery.** Keep continuous events short and use lower intensity where perceptually sufficient.

## 7. Prohibited Patterns

- `UIImpactFeedbackGenerator.impactOccurred()` as the sole haptic implementation for complex feedback moments. This is a sensory design failure.
- `print()` or `NSLog()` for haptic debugging in committed code. Use structured logging gated behind a debug flag.
- Force-unwrapping `try!` on engine or player creation. CoreHaptics operations are fallible by design.
- Hardcoded `Float` literals for intensity/sharpness/timing without named constants.
- Single continuous event spanning an entire interaction (>5s) without parameter modulation.
- Playing haptics when the app is backgrounded (the system stops the engine; attempting playback is a no-op that wastes cycles).

## Verification Checklist

Before declaring haptic implementation complete:

- [ ] Capability check gates all CoreHaptics code paths
- [ ] Engine is a single retained instance with `stoppedHandler` and `resetHandler` configured
- [ ] `playsHapticsOnly = true` is set when no audio events are used
- [ ] All intensity/sharpness/timing values use named constants
- [ ] Continuous events have parameter curves — no flat sustained output
- [ ] Progressive sequences use multi-phase architecture with explicit phase boundaries
- [ ] Fallback to UIKit generators preserves semantic intent
- [ ] No duplicate haptic signals (CoreHaptics and UIKit never fire for the same event)
- [ ] Pattern players are recreated in `resetHandler`
- [ ] Real-device testing confirmed (Simulator produces no haptic output)
