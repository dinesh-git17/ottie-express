# Spring Tuning Reference

## iOS 17 Preset Internals

| Preset    | Baseline Bounce | Effective dampingFraction Equivalent |
| --------- | --------------- | ------------------------------------ |
| `.smooth` | 0.0             | ~1.0 (critically damped)             |
| `.snappy` | ~0.15           | ~0.78 (slight underdamping)          |
| `.bouncy` | ~0.3            | ~0.6 (visible underdamping)          |

All presets accept `duration:` (default 0.5s) and `extraBounce:` (default 0.0).

## Parameter Mapping: Old API → iOS 17 API

| Old API           | iOS 17 API      | Relationship                                                                                 |
| ----------------- | --------------- | -------------------------------------------------------------------------------------------- |
| `response`        | `duration`      | Approximate 1:1 mapping. `duration` is perceptual; `response` is settling half-life.         |
| `dampingFraction` | `bounce`        | Inverse relationship. `bounce = 0` ≈ `dampingFraction = 1.0`. Higher bounce = lower damping. |
| `blendDuration`   | `blendDuration` | Identical semantics.                                                                         |

## Tuning Recipes

### Button Press Feedback

```swift
.snappy(duration: 0.25)
```

Short, brisk. No visible overshoot. Scale down on press: `scaleEffect(isPressed ? 0.95 : 1.0)`.

### Card Expansion (Hero Transition)

```swift
.smooth(duration: 0.4)
```

Critically damped. Paired with `matchedGeometryEffect`. No bounce — expansion must feel controlled.

### Bottom Sheet Snap

```swift
.interactiveSpring(response: 0.3, dampingFraction: 0.8)
```

Slightly underdamped for a subtle settle. Response tuned for distance: sheets travel further than buttons.

### Tab Indicator Slide

```swift
.snappy(duration: 0.3)
```

Brisk lateral motion. Slight built-in bounce from `.snappy` preset provides liveliness.

### Floating Action Button Entrance

```swift
.spring(duration: 0.5, bounce: 0.2)
```

Visible but controlled overshoot. Scale from 0.0 to 1.0 with `scaleEffect` + `opacity`.

### Notification Banner Slide-In

```swift
.smooth(duration: 0.35)
```

Offset from top. No bounce — banners should not feel playful. Dismissal: `.snappy(duration: 0.2)` (faster out than in).

### Drag Release Snap-Back

```swift
.interactiveSpring(response: 0.15, dampingFraction: 0.86, blendDuration: 0.25)
```

Default interactive spring. Fast response for finger-tracking fidelity. `blendDuration` smooths retargeting when the user changes drag direction.

### Pull-to-Refresh Indicator

```swift
.interactiveSpring(response: 0.3, dampingFraction: 0.7)
```

Slightly bouncier than default interactive spring. The indicator should feel tethered to the pull gesture with a small settle on release.

### Modal Dismiss (Swipe Down)

```swift
.interactiveSpring(response: 0.35, dampingFraction: 0.86)
```

Slightly slower response than default for the larger travel distance of a modal dismissal.

## Velocity Preservation

When transitioning from a gesture to a spring animation, pass the gesture's velocity to ensure seamless handoff:

```swift
.onEnded { value in
    let velocity = value.predictedEndTranslation.height / value.translation.height
    withAnimation(.spring(response: 0.3, dampingFraction: 0.8, blendDuration: 0.1)) {
        // Apply final state; the spring blends with the gesture's momentum
    }
}
```

The `blendDuration` parameter controls how aggressively the spring merges with the outgoing gesture velocity. Higher values produce smoother handoffs at the cost of delayed settling.

## Dangerous Ranges (Why Boundaries Exist)

| Parameter               | Dangerous Value       | Visual Result                                |
| ----------------------- | --------------------- | -------------------------------------------- |
| `dampingFraction < 0.3` | Heavy oscillation     | UI feels broken, elements bounce repeatedly  |
| `dampingFraction = 0.0` | Perpetual oscillation | Element never stops moving                   |
| `response > 1.0`        | Extreme sluggishness  | UI feels unresponsive, user re-taps          |
| `response < 0.1`        | Near-instantaneous    | No perceptible animation, wasted computation |
| `bounce > 0.6`          | Cartoon physics       | Unprofessional, undermines trust             |
| `duration > 1.0`        | Extended animation    | Blocks perceived responsiveness              |
