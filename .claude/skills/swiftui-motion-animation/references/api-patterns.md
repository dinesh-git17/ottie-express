# Animation API Production Patterns

## 1. PhaseAnimator: Success State Feedback

```swift
enum SuccessPhase: CaseIterable {
    case idle
    case scaled
    case settled
}

PhaseAnimator(SuccessPhase.allCases, trigger: didSucceed) { phase in
    Image(systemName: "checkmark.circle.fill")
        .scaleEffect(phase == .scaled ? 1.3 : 1.0)
        .opacity(phase == .idle ? 0.0 : 1.0)
        .rotationEffect(phase == .scaled ? .degrees(-10) : .zero)
} animation: { phase in
    switch phase {
    case .idle: .snappy(duration: 0.15)
    case .scaled: .bouncy(duration: 0.4, extraBounce: 0.1)
    case .settled: .smooth(duration: 0.3)
    }
}
```

Key: phases are an enum, animations are springs, trigger-based (not continuous).

## 2. PhaseAnimator: Loading Pulse (Continuous)

```swift
enum PulsePhase: CaseIterable {
    case dim
    case bright
}

Circle()
    .fill(.secondary)
    .phaseAnimator(PulsePhase.allCases) { content, phase in
        content.opacity(phase == .dim ? 0.3 : 0.8)
    } animation: { phase in
        switch phase {
        case .dim: .easeInOut(duration: 1.2)
        case .bright: .easeInOut(duration: 1.2)
        }
    }
```

Timing curves are acceptable here: this is a non-interactive ambient effect. No trigger = continuous.

## 3. KeyframeAnimator: Celebration Bounce

```swift
struct CelebrationValues {
    var scale: Double = 1.0
    var yOffset: Double = 0.0
    var rotation: Angle = .zero
}

.keyframeAnimator(initialValue: CelebrationValues(), trigger: celebrate) { content, value in
    content
        .scaleEffect(value.scale)
        .offset(y: value.yOffset)
        .rotationEffect(value.rotation)
} keyframes: { _ in
    KeyframeTrack(\.scale) {
        SpringKeyframe(1.4, duration: 0.2, spring: .bouncy)
        CubicKeyframe(0.9, duration: 0.15)
        SpringKeyframe(1.0, duration: 0.3, spring: .smooth)
    }
    KeyframeTrack(\.yOffset) {
        SpringKeyframe(-30, duration: 0.25, spring: .snappy)
        CubicKeyframe(0, duration: 0.35)
    }
    KeyframeTrack(\.rotation) {
        CubicKeyframe(.degrees(8), duration: 0.15)
        CubicKeyframe(.degrees(-5), duration: 0.15)
        CubicKeyframe(.zero, duration: 0.2)
    }
}
```

Note: content closure uses only transforms. No layout properties.

## 4. matchedGeometryEffect: Card-to-Detail Hero

```swift
struct CardListView: View {
    @Namespace private var heroNamespace
    @State private var expandedID: Item.ID?

    var body: some View {
        ZStack {
            ScrollView {
                LazyVStack {
                    ForEach(items) { item in
                        if expandedID != item.id {
                            CardView(item: item)
                                .matchedGeometryEffect(
                                    id: item.id,
                                    in: heroNamespace,
                                    isSource: expandedID != item.id
                                )
                                .onTapGesture {
                                    withAnimation(.smooth(duration: 0.4)) {
                                        expandedID = item.id
                                    }
                                }
                        }
                    }
                }
            }

            if let id = expandedID, let item = items.first(where: { $0.id == id }) {
                DetailView(item: item)
                    .matchedGeometryEffect(
                        id: item.id,
                        in: heroNamespace,
                        isSource: expandedID == item.id
                    )
                    .onTapGesture {
                        withAnimation(.smooth(duration: 0.35)) {
                            expandedID = nil
                        }
                    }
            }
        }
        .geometryGroup()
    }
}
```

Key patterns:

- Single `@Namespace` at lowest common ancestor (ZStack).
- `isSource` toggles with state.
- `geometryGroup()` on parent ZStack — children conditionally created during animation.
- `.smooth` animation — hero transitions must not bounce.

## 5. Scoped Animation: Independent Properties

```swift
struct ToggleCard: View {
    @State private var isExpanded = false
    @State private var isHighlighted = false

    var body: some View {
        RoundedRectangle(cornerRadius: 16)
            .fill(isHighlighted ? Color.blue : Color.gray)
            .animation(.snappy(duration: 0.2), value: isHighlighted)
            .frame(height: isExpanded ? 300 : 100)
            .animation(.smooth(duration: 0.4), value: isExpanded)
    }
}
```

Two independent animations with different springs. The `.animation()` modifiers are placed immediately before their respective animatable properties.

## 6. iOS 17 Closure-Based Scoping

```swift
RoundedRectangle(cornerRadius: 16)
    .animation(.snappy(duration: 0.2)) {
        $0.foregroundStyle(isActive ? Color.blue : Color.secondary)
    }
    .animation(.smooth(duration: 0.4)) {
        $0.scaleEffect(isExpanded ? 1.1 : 1.0)
    }
```

Surgical: each closure scopes animation to exactly one modifier.

## 7. Gesture-to-Spring Handoff

```swift
@State private var offset: CGSize = .zero
@State private var isDragging = false

var body: some View {
    CardView()
        .offset(offset)
        .gesture(
            DragGesture()
                .onChanged { value in
                    isDragging = true
                    offset = value.translation
                }
                .onEnded { _ in
                    isDragging = false
                    withAnimation(.interactiveSpring(response: 0.3, dampingFraction: 0.8)) {
                        offset = .zero
                    }
                }
        )
        .animation(isDragging ? .interactiveSpring(response: 0.15, dampingFraction: 0.86) : .smooth, value: offset)
}
```

During drag: fast-tracking interactive spring. On release: slightly slower spring for snap-back.

## 8. Transition Composition

```swift
extension AnyTransition {
    static var slideUp: AnyTransition {
        .modifier(
            active: SlideModifier(offset: 40, opacity: 0),
            identity: SlideModifier(offset: 0, opacity: 1)
        )
    }
}

struct SlideModifier: ViewModifier {
    let offset: CGFloat
    let opacity: Double

    func body(content: Content) -> some View {
        content
            .offset(y: offset)
            .opacity(opacity)
    }
}

// Usage
if showContent {
    ContentView()
        .transition(.slideUp)
}
```

Custom transition uses offset (transform) + opacity. No layout-triggering properties.

## 9. Completion-Based Sequencing (iOS 17+)

```swift
func expandAndReveal() {
    withAnimation(.smooth(duration: 0.35)) {
        isExpanded = true
    } completion: {
        withAnimation(.snappy(duration: 0.25)) {
            showActions = true
        }
    }
}
```

Chained animations using completion handlers. No `asyncAfter`.

## 10. Disabling Animation for Specific State Changes

```swift
func resetWithoutAnimation() {
    var transaction = Transaction(animation: nil)
    transaction.disablesAnimations = true
    withTransaction(transaction) {
        offset = .zero
        scale = 1.0
        rotation = .zero
    }
}
```

`disablesAnimations = true` prevents implicit `.animation()` modifiers from creating animations for this state change cycle.
