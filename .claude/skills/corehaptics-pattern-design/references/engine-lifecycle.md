# CHHapticEngine Lifecycle Reference

## Initialization

```swift
guard CHHapticEngine.capabilitiesForHardware().supportsHaptics else { return }

let engine = try CHHapticEngine()
engine.playsHapticsOnly = true
engine.isAutoShutdownEnabled = true
```

With shared audio session (required for audio-haptic patterns):

```swift
let session = AVAudioSession.sharedInstance()
let engine = try CHHapticEngine(audioSession: session)
```

## Starting

Asynchronous (preferred — does not block main thread):

```swift
engine.start { error in
    if let error {
        // Transition to UIKit fallback
    }
}
```

Synchronous (acceptable on background queues only):

```swift
try engine.start()
```

## stoppedHandler

Fires on an internal non-main queue. Reasons:

| Reason                   | Raw | Recovery                                                   |
| ------------------------ | --- | ---------------------------------------------------------- |
| `.audioSessionInterrupt` | 1   | Restart when interruption ends                             |
| `.applicationSuspended`  | 2   | Restart on `willEnterForeground`                           |
| `.idleTimeout`           | 3   | No action — engine restarts transparently on next playback |
| `.notifyWhenFinished`    | 4   | Normal completion — no action needed                       |
| `.systemError`           | -1  | May require engine recreation                              |

```swift
engine.stoppedHandler = { [weak self] reason in
    guard let self else { return }
    switch reason {
    case .audioSessionInterrupt:
        self.engineNeedsRestart = true
    case .applicationSuspended:
        self.engineNeedsRestart = true
    case .idleTimeout:
        break // Auto-shutdown handles restart transparently
    case .notifyWhenFinished:
        break
    case .systemError:
        self.engineNeedsRecreation = true
    @unknown default:
        break
    }
}
```

## resetHandler

Fires when the haptic server crashes. All pattern players are invalidated.

```swift
engine.resetHandler = { [weak self] in
    guard let self else { return }
    do {
        try self.engine.start()
        self.recreateAllPlayers()
    } catch {
        self.transitionToFallback()
    }
}
```

The `recreateAllPlayers()` method must rebuild players from stored `CHHapticPattern` instances. Patterns survive resets; players do not.

## notifyWhenPlayersFinished

Controls engine behavior after all active players complete:

```swift
engine.notifyWhenPlayersFinished { error in
    return .leaveEngineRunning // or .stopEngine
}
```

Use `.stopEngine` only in batch-playback scenarios where you know no further haptics are needed. Use `.leaveEngineRunning` for interactive contexts.

## Engine Clock

`engine.currentTime` returns the engine's absolute clock in seconds. Use this for scheduling:

```swift
let scheduledTime = engine.currentTime + desiredDelay
try player.start(atTime: scheduledTime)
```

The engine clock is independent of `CACurrentMediaTime()`. Compute the offset once if synchronizing with Core Animation:

```swift
let clockOffset = CACurrentMediaTime() - engine.currentTime
```

## Resource Management

| Strategy                                 | When                                            |
| ---------------------------------------- | ----------------------------------------------- |
| `isAutoShutdownEnabled = true`           | Intermittent haptics (forms, auth flows)        |
| `isAutoShutdownEnabled = false`          | Continuous interaction (games, drawing)         |
| Explicit `stop()`                        | View disappearance in manual-lifecycle contexts |
| `notifyWhenPlayersFinished(.stopEngine)` | Known-finite playback sequences                 |

## Muting

```swift
engine.isMutedForHaptics = false
engine.isMutedForAudio = false
```

These allow selectively disabling one channel while keeping the other active. Useful for user preferences or debug modes.
