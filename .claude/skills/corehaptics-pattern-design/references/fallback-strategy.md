# Fallback Strategy Reference

## Capability Detection

```swift
let capabilities = CHHapticEngine.capabilitiesForHardware()
let supportsHaptics = capabilities.supportsHaptics  // iPhone 8+ with Taptic Engine
let supportsAudio = capabilities.supportsAudio      // Audio event support
```

CoreHaptics requires iPhone 8 or later. Older devices and iPads without Taptic Engine return `false`.

## Protocol-Based Branching

Define a protocol for all haptic operations. Instantiate the correct implementation once at initialization.

```swift
protocol HapticFeedbackProviding {
    func playInitiation()
    func playProcessing(progress: Float)
    func playConfirmation()
    func playWarning()
    func playError()
    func playSelection()
    func prepare()
}
```

### Rich Implementation (CoreHaptics)

```swift
final class CoreHapticsProvider: HapticFeedbackProviding {
    private let engine: CHHapticEngine
    private var confirmationPlayer: CHHapticPatternPlayer?
    private let confirmationPattern: CHHapticPattern
    // ... stored patterns for each semantic event

    init(engine: CHHapticEngine) {
        self.engine = engine
        self.confirmationPattern = try Self.buildConfirmationPattern()
        self.confirmationPlayer = try engine.makePlayer(with: confirmationPattern)
        // Build and store all patterns
    }

    func playConfirmation() {
        try? confirmationPlayer?.start(atTime: CHHapticTimeImmediate)
    }

    func playProcessing(progress: Float) {
        let param = CHHapticDynamicParameter(
            parameterID: .hapticIntensityControl,
            value: progress,
            relativeTime: 0
        )
        try? processingPlayer?.sendParameters([param], atTime: CHHapticTimeImmediate)
    }

    // ... other implementations with full pattern richness
}
```

### Fallback Implementation (UIKit Generators)

```swift
final class UIKitHapticProvider: HapticFeedbackProviding {
    private let impactLight = UIImpactFeedbackGenerator(style: .light)
    private let impactMedium = UIImpactFeedbackGenerator(style: .medium)
    private let impactHeavy = UIImpactFeedbackGenerator(style: .heavy)
    private let notification = UINotificationFeedbackGenerator()
    private let selection = UISelectionFeedbackGenerator()

    func playInitiation() {
        impactLight.impactOccurred()
    }

    func playProcessing(progress: Float) {
        // Map continuous progress to discrete taps at thresholds
        // Fire impact at 25%, 50%, 75% progress milestones only
    }

    func playConfirmation() {
        notification.notificationOccurred(.success)
    }

    func playWarning() {
        notification.notificationOccurred(.warning)
    }

    func playError() {
        notification.notificationOccurred(.error)
    }

    func playSelection() {
        selection.selectionChanged()
    }

    func prepare() {
        impactLight.prepare()
        impactMedium.prepare()
        impactHeavy.prepare()
        notification.prepare()
        selection.prepare()
    }
}
```

### Factory

```swift
enum HapticProviderFactory {
    static func makeProvider() -> HapticFeedbackProviding {
        guard CHHapticEngine.capabilitiesForHardware().supportsHaptics else {
            return UIKitHapticProvider()
        }
        do {
            let engine = try CHHapticEngine()
            engine.playsHapticsOnly = true
            engine.isAutoShutdownEnabled = true
            return CoreHapticsProvider(engine: engine)
        } catch {
            return UIKitHapticProvider()
        }
    }
}
```

Engine creation failure falls through to UIKit — no silent failure path.

## UIKit Generator Details

### UIImpactFeedbackGenerator Styles

| Style     | Intensity | Sharpness | Semantic                   |
| --------- | --------- | --------- | -------------------------- |
| `.light`  | ~0.3      | ~0.5      | Subtle acknowledgment      |
| `.medium` | ~0.6      | ~0.5      | Standard interaction       |
| `.heavy`  | ~1.0      | ~0.3      | Pronounced physical impact |
| `.rigid`  | ~0.8      | ~0.9      | Crisp mechanical feedback  |
| `.soft`   | ~0.4      | ~0.1      | Gentle organic feedback    |

iOS 17+ supports `impactOccurred(intensity:)` for variable-strength impacts without style switching.

### UINotificationFeedbackGenerator Types

| Type       | Pattern              | Semantic            |
| ---------- | -------------------- | ------------------- |
| `.success` | Two ascending taps   | Positive completion |
| `.warning` | Two level taps       | Caution required    |
| `.error`   | Three declining taps | Failure occurred    |

### prepare() Timing

Call `prepare()` 1–2 seconds before expected use. The warmed state expires after a few seconds. Calling `prepare()` repeatedly is safe but wastes minor CPU.

For predictable interaction flows (e.g., a button becoming tappable after form validation), call `prepare()` when the UI state transitions to "ready."

## Progressive Feedback Fallback

CoreHaptics progressive sequences (continuous ramp + transient bursts) cannot be replicated with UIKit generators. The fallback strategy preserves rhythm and semantic milestones:

```swift
func playProgressiveScan(totalDuration: TimeInterval) {
    let milestoneCount = 4
    let interval = totalDuration / TimeInterval(milestoneCount)

    for i in 0..<milestoneCount {
        let delay = interval * TimeInterval(i)
        let style: UIImpactFeedbackGenerator.FeedbackStyle = {
            switch i {
            case 0: return .light
            case 1: return .medium
            case 2: return .medium
            default: return .heavy
            }
        }()

        DispatchQueue.main.asyncAfter(deadline: .now() + delay) {
            UIImpactFeedbackGenerator(style: style).impactOccurred()
        }
    }

    // Final confirmation at end
    DispatchQueue.main.asyncAfter(deadline: .now() + totalDuration) {
        UINotificationFeedbackGenerator().notificationOccurred(.success)
    }
}
```

This preserves: (1) escalating intensity, (2) rhythmic pacing, (3) distinct confirmation at completion. It loses: continuous ramp texture, sharpness modulation, smooth parameter curves.

## SwiftUI sensoryFeedback (iOS 17+)

For SwiftUI views, `.sensoryFeedback` provides declarative haptics without UIKit or CoreHaptics imports:

```swift
.sensoryFeedback(.success, trigger: isComplete)
.sensoryFeedback(.impact(weight: .heavy, intensity: 0.9), trigger: collisionCount)
```

This is a convenience API backed by the system haptic engine. It does not provide CoreHaptics-level control but is appropriate for standard UI feedback. Do not mix `.sensoryFeedback` with manual `UIFeedbackGenerator` calls for the same interaction.
