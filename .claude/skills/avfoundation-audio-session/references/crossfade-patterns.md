# Crossfade Patterns Reference

Implementation patterns for volume control, crossfading, and perceptual gain management.

## Equal-Power Crossfade

Linear crossfades produce a perceived -6dB dip at the midpoint (both tracks at 0.5 linear = half power each, total perceived loudness drops). Equal-power maintains constant perceived loudness.

### Curve Functions

```swift
func equalPowerGains(progress t: Float) -> (outGain: Float, inGain: Float) {
    let angle = t * Float.pi / 2.0
    return (outGain: cos(angle), inGain: sin(angle))
}
```

At `t = 0.0`: outGain = 1.0, inGain = 0.0
At `t = 0.5`: outGain = 0.707, inGain = 0.707 (combined power = 1.0)
At `t = 1.0`: outGain = 0.0, inGain = 1.0

### Timer-Driven Implementation

`setVolume(_:fadeDuration:)` uses linear interpolation internally. For true equal-power, drive the curve explicitly:

```swift
func performEqualPowerCrossfade(
    from outgoing: AVAudioPlayer,
    to incoming: AVAudioPlayer,
    duration: TimeInterval
) {
    let steps = AudioMixConstants.crossfadeSteps
    let interval = duration / Double(steps)

    incoming.volume = 0.0
    incoming.prepareToPlay()
    incoming.play()

    for i in 0...steps {
        let t = Float(i) / Float(steps)
        let (outGain, inGain) = equalPowerGains(progress: t)

        DispatchQueue.main.asyncAfter(deadline: .now() + interval * Double(i)) {
            outgoing.volume = outGain * outgoing.baseVolume
            incoming.volume = inGain * incoming.baseVolume
        }
    }

    DispatchQueue.main.asyncAfter(deadline: .now() + duration) {
        outgoing.stop()
    }
}
```

`baseVolume` is the layer's target volume from `AudioMixConstants`. The crossfade scales relative to it.

### CADisplayLink Implementation (Smoother)

For frame-synchronized fading without timer scheduling overhead:

```swift
final class CrossfadeController {
    private var displayLink: CADisplayLink?
    private var startTime: CFTimeInterval = 0
    private let duration: TimeInterval
    private let outgoing: AVAudioPlayer
    private let incoming: AVAudioPlayer
    private let outgoingBaseVolume: Float
    private let incomingBaseVolume: Float
    private var completion: (() -> Void)?

    init(from outgoing: AVAudioPlayer, to incoming: AVAudioPlayer,
         duration: TimeInterval, completion: (() -> Void)? = nil) {
        self.outgoing = outgoing
        self.incoming = incoming
        self.duration = duration
        self.outgoingBaseVolume = outgoing.volume
        self.incomingBaseVolume = /* target volume from constants */
        self.completion = completion
    }

    func start() {
        incoming.volume = 0.0
        incoming.prepareToPlay()
        incoming.play()

        displayLink = CADisplayLink(target: self, selector: #selector(tick))
        displayLink?.add(to: .main, forMode: .common)
        startTime = CACurrentMediaTime()
    }

    @objc private func tick() {
        let elapsed = CACurrentMediaTime() - startTime
        let progress = Float(min(elapsed / duration, 1.0))
        let angle = progress * Float.pi / 2.0

        outgoing.volume = cos(angle) * outgoingBaseVolume
        incoming.volume = sin(angle) * incomingBaseVolume

        if progress >= 1.0 {
            displayLink?.invalidate()
            displayLink = nil
            outgoing.stop()
            completion?()
        }
    }
}
```

## Simple Fade In / Fade Out

For single-track fading, the built-in method is acceptable:

```swift
// Fade in
func fadeIn(_ player: AVAudioPlayer, to targetVolume: Float, duration: TimeInterval) {
    player.volume = 0.0
    player.play()
    DispatchQueue.main.async {
        player.setVolume(targetVolume, fadeDuration: duration)
    }
}

// Fade out and stop
func fadeOutAndStop(_ player: AVAudioPlayer, duration: TimeInterval) {
    DispatchQueue.main.async {
        player.setVolume(0.0, fadeDuration: duration)
    }
    DispatchQueue.main.asyncAfter(deadline: .now() + duration) {
        player.stop()
    }
}
```

The `DispatchQueue.main.async` wrapper prevents the known iOS bug where `setVolume(_:fadeDuration:)` executes as an instant jump when called synchronously in certain contexts.

## AVAudioEngine Volume Control

### Via AVAudioUnitEQ (Recommended)

```swift
// Convert linear volume to dB
func linearToDecibels(_ linear: Float) -> Float {
    guard linear > 0 else { return -96.0 }
    return 20.0 * log10(linear)
}

// Smooth gain ramp via display link
func rampGain(eq: AVAudioUnitEQ, from startDB: Float, to endDB: Float,
              duration: TimeInterval) {
    let steps = 60
    let interval = duration / Double(steps)

    for i in 0...steps {
        let t = Float(i) / Float(steps)
        let currentDB = startDB + (endDB - startDB) * t

        DispatchQueue.main.asyncAfter(deadline: .now() + interval * Double(i)) {
            eq.globalGain = currentDB
        }
    }
}
```

### Via sendParameters (AVAudioEngine Real-Time)

For runtime-driven volume modulation synced to animation or gesture:

```swift
// Requires AVAudioEngine with running player
func modulateVolume(player: CHHapticAdvancedPatternPlayer,
                    intensity: Float) {
    // This pattern applies to haptic players, but the principle is identical
    // for AVAudioEngine: use sendParameters for real-time modulation.
}
```

## Perceptual Volume Mapping

### UI Slider to Linear Volume

Human hearing is logarithmic. A linear 0–1 slider feels uneven. Map slider position to perceptual volume:

```swift
// Quadratic approximation (simple, sufficient for most UI)
func perceptualVolume(sliderPosition: Float) -> Float {
    return pow(sliderPosition, 2.0)
}

// Logarithmic mapping (more accurate)
func logarithmicVolume(sliderPosition: Float, minDB: Float = -60.0) -> Float {
    guard sliderPosition > 0 else { return 0.0 }
    let db = minDB + (0.0 - minDB) * sliderPosition
    return pow(10.0, db / 20.0)
}
```

| Slider Position | Quadratic Volume | Logarithmic Volume (-60dB floor) |
| --------------- | ---------------- | -------------------------------- |
| 0.0             | 0.0              | 0.0                              |
| 0.25            | 0.0625           | 0.018                            |
| 0.5             | 0.25             | 0.032                            |
| 0.75            | 0.5625           | 0.178                            |
| 1.0             | 1.0              | 1.0                              |

## Crossfade Coordination with Layer Architecture

When crossfading one layer while others continue playing, scale the crossfade relative to each layer's base volume:

```swift
func crossfadeInstrumental(to newTrackURL: URL) throws {
    let newPlayer = try AVAudioPlayer(contentsOf: newTrackURL)
    newPlayer.numberOfLoops = -1

    let crossfader = CrossfadeController(
        from: currentInstrumental,
        to: newPlayer,
        duration: AudioMixConstants.crossfadeDuration
    ) { [weak self] in
        self?.currentInstrumental = newPlayer
    }
    crossfader.start()

    // Voice note and tape hiss continue unaffected
}
```

## Prohibited Patterns

- Using `setVolume(_:fadeDuration:)` for crossfades between tracks (linear only, creates loudness dip).
- Setting `.volume` property directly during an active `setVolume(_:fadeDuration:)` call (cancels the fade).
- Hardcoded volume floats anywhere in crossfade logic.
- Timer intervals faster than display refresh rate (wastes CPU, no perceptual benefit).
- Crossfading without calling `prepareToPlay()` on the incoming player first (risks delayed start).
