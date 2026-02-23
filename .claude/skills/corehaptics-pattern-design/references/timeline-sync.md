# Timeline Synchronization Reference

## Clock Domains

CoreHaptics uses its own clock (`engine.currentTime`), independent of:

- `CACurrentMediaTime()` (Core Animation / QuartzCore)
- `AVAudioTime` (AVFoundation)
- `CFAbsoluteTimeGetCurrent()` (system wall clock)

Compute the offset once at sync start:

```swift
let coreAnimationOffset = CACurrentMediaTime() - engine.currentTime
```

## Immediate vs Scheduled Playback

**Immediate** (`CHHapticTimeImmediate`): Minimum latency (~1–5ms with running engine). Use for direct user actions.

```swift
try player.start(atTime: CHHapticTimeImmediate)
```

**Scheduled** (absolute engine time): Precise future scheduling. Use for animation/audio-synced haptics.

```swift
let targetTime = engine.currentTime + 0.25
try player.start(atTime: targetTime)
```

## CADisplayLink-Driven Haptics

For real-time haptic modulation tied to animation frames:

```swift
private var displayLink: CADisplayLink?
private var continuousPlayer: CHHapticPatternPlayer?

func startAnimationSyncedHaptics() {
    displayLink = CADisplayLink(target: self, selector: #selector(displayLinkFired))
    displayLink?.add(to: .main, forMode: .common)
}

@objc private func displayLinkFired(_ link: CADisplayLink) {
    let progress = computeAnimationProgress(link.timestamp)

    // 10–20ms lookahead compensates for actuator latency
    let lookaheadCompensation: TimeInterval = 0.015
    let hapticTime = engine.currentTime + lookaheadCompensation

    let intensityParam = CHHapticDynamicParameter(
        parameterID: .hapticIntensityControl,
        value: Float(progress),
        relativeTime: 0
    )
    try? continuousPlayer?.sendParameters([intensityParam], atTime: hapticTime)
}
```

## Audio-Reactive Haptics

Tap the audio mix bus, compute amplitude, drive haptic intensity:

```swift
audioEngine.mainMixerNode.installTap(
    onBus: 0,
    bufferSize: 1024,
    format: audioEngine.mainMixerNode.outputFormat(forBus: 0)
) { [weak self] buffer, time in
    guard let self else { return }
    let rms = self.computeRMS(buffer: buffer)
    let mappedIntensity = min(max(rms * self.amplificationFactor, 0), 1)

    let param = CHHapticDynamicParameter(
        parameterID: .hapticIntensityControl,
        value: Float(mappedIntensity),
        relativeTime: 0
    )
    try? self.hapticPlayer?.sendParameters([param], atTime: CHHapticTimeImmediate)
}
```

For sample-accurate audio-haptic sync, embed both `.audioContinuous`/`.audioCustom` and `.hapticTransient`/`.hapticContinuous` events in the same `CHHapticPattern`. The engine guarantees synchronization within a single pattern player.

## Embedded Audio Events

```swift
let hapticTap = CHHapticEvent(
    eventType: .hapticTransient,
    parameters: [
        CHHapticEventParameter(parameterID: .hapticIntensity, value: 0.8),
        CHHapticEventParameter(parameterID: .hapticSharpness, value: 0.6)
    ],
    relativeTime: 0
)

let audioChime = CHHapticEvent(
    eventType: .audioCustom,
    parameters: [
        CHHapticEventParameter(parameterID: .audioVolume, value: 0.5)
    ],
    relativeTime: 0,
    duration: 0 // Plays full file
)

// Register audio resource before pattern creation
let resourceID = try engine.registerAudioResource(
    Bundle.main.url(forResource: "chime", withExtension: "wav")!
)
```

When using embedded audio, do NOT set `playsHapticsOnly = true`. The engine needs audio hardware initialized.

## Animation Phase Mapping

Map UIKit/SwiftUI animation phases to haptic events:

```swift
// UIKit: Align haptic phases to animation keyframes
UIView.animateKeyframes(withDuration: 2.0, delay: 0) {
    UIView.addKeyframe(withRelativeStartTime: 0, relativeDuration: 0.3) {
        // Scale up — initiation haptic at relativeTime 0
    }
    UIView.addKeyframe(withRelativeStartTime: 0.3, relativeDuration: 0.5) {
        // Pulse — processing haptic at relativeTime 0.6
    }
    UIView.addKeyframe(withRelativeStartTime: 0.8, relativeDuration: 0.2) {
        // Final state — confirmation haptic at relativeTime 1.6
    }
} completion: { _ in
    // Do NOT schedule confirmation haptic here — use pattern timing instead
    // Completion blocks have unpredictable delay
}
```

```swift
// SwiftUI (iOS 17+): Bind sensoryFeedback to the same state driving animation
.sensoryFeedback(.success, trigger: authenticationState) { oldValue, newValue in
    newValue == .authenticated ? .success : nil
}
```

## Advanced Player Seek Limitation

`seek(toOffset:)` cannot resume a continuous event mid-stream. The player starts output from the next event boundary after the seek position.

Workaround: Split long continuous events into 250–500ms segments:

```swift
// Instead of one 3-second continuous event:
// let longEvent = CHHapticEvent(eventType: .hapticContinuous, ..., duration: 3.0)

// Use a sequence of shorter events:
let segmentDuration: TimeInterval = 0.3
let segmentCount = 10
var events: [CHHapticEvent] = []
for i in 0..<segmentCount {
    let event = CHHapticEvent(
        eventType: .hapticContinuous,
        parameters: baseParameters,
        relativeTime: Double(i) * segmentDuration,
        duration: segmentDuration
    )
    events.append(event)
}
```

This ensures seek lands near a segment boundary, minimizing dead time.
