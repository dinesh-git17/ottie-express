---
name: swiftui-motion-animation
description: Enforce production-grade SwiftUI motion systems with physically believable springs, context-aware animation selection, and 60fps-safe transitions. Use when writing, reviewing, or modifying any SwiftUI animation code, transition, or interactive motion. Triggers on SwiftUI animation implementation, spring tuning, matchedGeometryEffect usage, PhaseAnimator/KeyframeAnimator patterns, or motion performance optimization.
---

# SwiftUI Motion & Animation

Governing standard for all SwiftUI animation code in this repository. Every animation must be physically believable, performance-safe, and context-appropriate. Generic `.easeInOut(duration: 0.3)` is prohibited.

---

## 1. Motion Philosophy

Motion communicates hierarchy, causality, and state. It is not decoration.

**Rules:**

- Every animation must have a reason: confirm an action, reveal structure, or maintain spatial continuity.
- Motion duration scales with travel distance. Small movements are fast; large movements are slower.
- Interactive elements use springs. Non-interactive transitions may use timing curves only when spring behavior is inappropriate (e.g., progress indicators, loading bars).
- Animations must never block user interaction. If a transition takes >0.5s, the UI must remain interactive during it.

---

## 2. Spring Physics (Default Animation Type)

Springs are the default. Timing curves require explicit justification.

### 2.1 Mandatory Presets (iOS 17+)

Use named presets as starting points. Raw `.spring(response:dampingFraction:)` only when presets cannot achieve the required feel.

| Context                       | Preset               | Tuning                                     |
| ----------------------------- | -------------------- | ------------------------------------------ |
| Standard UI transitions       | `.smooth`            | `duration: 0.35–0.5`                       |
| Button feedback, toggles      | `.snappy`            | `duration: 0.25–0.35`                      |
| Deliberate playful affordance | `.bouncy`            | `duration: 0.4–0.6, extraBounce: 0.0–0.15` |
| Gesture-driven (drag, scroll) | `.interactiveSpring` | `response: 0.15, dampingFraction: 0.86`    |

### 2.2 Parameter Boundaries (BLOCKING)

These ranges produce physically believable motion. Values outside these ranges require a code comment justifying the deviation.

| Parameter           | Minimum | Maximum | Notes                                        |
| ------------------- | ------- | ------- | -------------------------------------------- |
| `response`          | 0.15    | 0.8     | <0.15 feels instantaneous; >0.8 feels floaty |
| `dampingFraction`   | 0.5     | 1.0     | <0.5 is visually rubbery                     |
| `duration` (iOS 17) | 0.2     | 0.8     | Perceptual duration bound                    |
| `bounce` (iOS 17)   | -0.3    | 0.4     | >0.4 is cartoon physics                      |
| `extraBounce`       | 0.0     | 0.2     | Additive on preset baseline                  |

### 2.3 Prohibited Patterns

- `.linear` for UI element motion (acceptable only for opacity fades on non-interactive overlays).
- `.easeInOut(duration:)` as a default. Every timing curve must be a deliberate choice with a comment.
- `dampingFraction: 0.0` (undamped oscillation — never settles).
- `response > 1.0` or `duration > 1.0` for any interactive element.

---

## 3. Animation Selection Decision Tree

Follow this sequence. Stop at the first match.

```
1. Gesture-driven (drag, scroll, resize)?
   → .interactiveSpring(response: 0.15, dampingFraction: 0.86)

2. Shared element transition (view morphs between contexts)?
   → matchedGeometryEffect + .smooth(duration: 0.35)

3. Multi-phase sequential animation (3+ distinct states)?
   → PhaseAnimator with per-phase spring selection

4. Complex choreographed motion (independent property timelines)?
   → KeyframeAnimator with per-track keyframe types

5. Single property change (opacity, scale, offset)?
   → withAnimation(.snappy(duration: 0.3)) { state = newValue }

6. Continuous/looping effect (pulse, shimmer, breathing)?
   → PhaseAnimator (continuous) or KeyframeAnimator (repeating)

7. Default fallback
   → withAnimation(.smooth) { state = newValue }
```

---

## 4. Animation Scoping Rules

### 4.1 Explicit Over Implicit

Prefer `withAnimation` (explicit) for state-driven transitions. Use `.animation(_:value:)` (implicit) only for self-contained, view-local property animations.

### 4.2 Scoping Priority

Implicit `.animation()` overrides explicit `withAnimation` for the views it covers.

- Place `.animation()` immediately before the specific animatable modifier it targets.
- Use iOS 17 closure-based `.animation(.spring) { $0.opacity(value) }` for surgical scoping.
- Block unwanted propagation with `.transaction { $0.animation = nil }`.

### 4.3 Conditional View Gotcha

Animation modifiers inside conditional branches are removed with the view. Place `.animation()` **outside** the conditional:

```swift
// CORRECT
VStack {
    if showDetail { DetailView().transition(.slide) }
}
.animation(.smooth, value: showDetail)

// WRONG — animation modifier destroyed with view
if showDetail {
    DetailView()
        .animation(.smooth, value: showDetail)
}
```

### 4.4 Preventing Unintended Animation

- Separate independent state mutations into distinct `withAnimation` calls.
- Use `Transaction(animation: nil)` with `disablesAnimations = true` when a state change must not animate.
- Never use the deprecated `.animation(_:)` (no value parameter).

---

## 5. matchedGeometryEffect

### 5.1 Modifier Order (BLOCKING)

Apply `matchedGeometryEffect` **before** frame modifiers.

```swift
// CORRECT — geometry effect receives flexible size proposals
Color.blue
    .matchedGeometryEffect(id: item.id, in: namespace)
    .frame(width: 100, height: 100)

// BROKEN — fixed frame overrides internal geometry
Color.blue
    .frame(width: 100, height: 100)
    .matchedGeometryEffect(id: item.id, in: namespace)
```

### 5.2 Source/Consumer Discipline

Exactly one view per `(id, namespace)` pair is the source:

```swift
.matchedGeometryEffect(id: item.id, in: ns, isSource: !isExpanded)
.matchedGeometryEffect(id: item.id, in: ns, isSource: isExpanded)
```

Multiple sources or zero sources per pair produce undefined behavior.

### 5.3 Namespace Scoping

- Declare `@Namespace` at the lowest common ancestor of matched views.
- Separate namespaces for independent match groups.
- Pass `Namespace.ID` to children — never create namespaces in child views.
- Namespaces do not cross `NavigationStack` or sheet boundaries.

### 5.4 Required Companion: `geometryGroup()` (iOS 17+)

Apply `geometryGroup()` to parent containers when child views are conditionally created during geometry animations. Without it, newly created children jump to final position.

### 5.5 Non-Animatable Properties

`matchedGeometryEffect` only animates position and size. Color, corner radius, and other visual properties crossfade. Animate these explicitly with separate `withAnimation` calls for smooth morphing.

---

## 6. PhaseAnimator (iOS 17+)

### 6.1 When to Use

Multi-step sequential animations where properties change together per phase.

### 6.2 Rules

- Define phases as a `CaseIterable` enum. No raw arrays of anonymous values.
- The animation closure receives the **destination** phase. First phase has no animation.
- No duplicate consecutive phase values — SwiftUI detects no change and skips instantly.
- For pauses, apply `.delay()` on the return transition animation.
- Prefer triggered mode (with `trigger:`) over continuous mode unless the animation genuinely loops.

### 6.3 View Form vs Modifier Form

Use the view form `PhaseAnimator(phases) { phase in ... }` when the content closure needs full view-building flexibility. Use the modifier form `.phaseAnimator(phases) { content, phase in ... }` for simple transformations on an existing view.

---

## 7. KeyframeAnimator (iOS 17+)

### 7.1 When to Use

Choreographed motion where independent properties animate on different timelines.

### 7.2 Rules

- Define an `AnimationValues` struct holding all animated properties with default (resting) values.
- One `KeyframeTrack` per animated property.
- **Performance-critical**: Content closure executes every frame. No layout recalculation, no string interpolation, no heavy computation inside it.
- Prefer `SpringKeyframe` and `CubicKeyframe`. Use `LinearKeyframe` for constant-speed segments only. Use `MoveKeyframe` for instantaneous jumps only.
- Total animation duration equals the longest track.

### 7.3 Keyframe Type Selection

| Type             | Interpolation   | Use Case                                     |
| ---------------- | --------------- | -------------------------------------------- |
| `SpringKeyframe` | Damped harmonic | Playful overshoots, natural settling         |
| `CubicKeyframe`  | Bezier curve    | Smooth controlled arcs with velocity control |
| `LinearKeyframe` | Constant speed  | Mechanical movement, progress bars           |
| `MoveKeyframe`   | Instantaneous   | State resets, discrete jumps                 |

---

## 8. Performance Engineering (BLOCKING)

### 8.1 Frame Budget

| Refresh Rate       | Frame Duration | Usable Budget |
| ------------------ | -------------- | ------------- |
| 60 Hz              | 16.67ms        | ~10ms         |
| 120 Hz (ProMotion) | 8.33ms         | ~5ms          |

Animations that exceed budget must be simplified or refactored.

### 8.2 Transform-Only Animation Rule

Animate transforms (GPU-composited, zero layout cost):

| Safe to Animate    | Triggers Layout Recalculation      |
| ------------------ | ---------------------------------- |
| `scaleEffect()`    | `frame()` size changes             |
| `offset()`         | `padding()` changes                |
| `rotationEffect()` | Size-dependent `GeometryReader`    |
| `opacity()`        | Conditional view insertion/removal |

**Rule**: If an animation modifies `frame`, `padding`, or triggers `GeometryReader` recalculation per frame, refactor to use transforms.

### 8.3 View Identity Stability

- Prefer inert modifiers (`.opacity(condition ? 1 : 0)`) over conditional branches (`if condition { View() }`) when you want property animation rather than transition.
- Use stable `Identifiable` values in `ForEach`. Never array indices.
- Never change `.id()` during an active animation.

### 8.4 Compositing Modifiers

| Modifier             | Use When                                                                      |
| -------------------- | ----------------------------------------------------------------------------- |
| `drawingGroup()`     | Many overlapping layers with gradients/blends. Never on interactive controls. |
| `compositingGroup()` | Overlapping translucent siblings need group-level opacity.                    |
| `geometryGroup()`    | Parent animates while children are conditionally created.                     |

### 8.5 Prohibited Performance Patterns

- Animating `.frame(width:height:)` directly — use `scaleEffect`.
- String interpolation inside `KeyframeAnimator` content closures.
- `GeometryReader` inside an animated subtree without caching the output.
- Creating new views (conditional branches) during frame-by-frame animation.

---

## 9. Transition Defaults

| Context                       | Transition                                      | Animation                 |
| ----------------------------- | ----------------------------------------------- | ------------------------- |
| Navigation push/pop           | `.move(edge:)`                                  | `.smooth`                 |
| Modal presentation            | `.move(edge: .bottom).combined(with: .opacity)` | `.smooth(duration: 0.4)`  |
| Content swap (same container) | `.opacity`                                      | `.snappy(duration: 0.2)`  |
| List item insertion           | `.slide.combined(with: .opacity)`               | `.smooth(duration: 0.3)`  |
| List item removal             | `.opacity`                                      | `.snappy(duration: 0.15)` |

Use `Transition` protocol (iOS 17+) or `AnyTransition.modifier(active:identity:)` for custom transitions. Active = offscreen state. Identity = resting state.

---

## 10. Completion Handling (iOS 17+)

Use `withAnimation(.spring) { ... } completion: { ... }` for sequencing post-animation logic. Do not use `DispatchQueue.main.asyncAfter` with estimated durations — spring durations are non-deterministic.

---

## 11. Code Review Checklist

Reject animation code if ANY of these are true:

- [ ] Uses generic `.easeInOut(duration: 0.3)` without justification comment.
- [ ] Spring parameters outside bounds in §2.2 without justification comment.
- [ ] `matchedGeometryEffect` applied after `frame()` modifier.
- [ ] Missing `geometryGroup()` on parent of conditionally-created children during geometry animation.
- [ ] Animates layout properties (`frame`, `padding`) per-frame instead of transforms.
- [ ] `KeyframeAnimator` content closure contains layout recalculation or string interpolation.
- [ ] Uses deprecated `.animation(_:)` without value parameter.
- [ ] Animation duration exceeds 1.0s for interactive elements.
- [ ] `.id()` modifier changes during active animation.
- [ ] Continuous `PhaseAnimator` where triggered mode is appropriate.
- [ ] `DispatchQueue.main.asyncAfter` used for animation sequencing on iOS 17+.

---

## References

- `references/spring-tuning.md` — Spring parameter mapping, preset internals, tuning recipes.
- `references/api-patterns.md` — Production code patterns for all animation APIs.
