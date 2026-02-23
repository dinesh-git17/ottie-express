# Interruption Lifecycle Reference

Complete implementation patterns for AVAudioSession interruption handling, route changes, and media server recovery.

## Interruption Handler

```swift
@objc private func handleInterruption(_ notification: Notification) {
    guard let userInfo = notification.userInfo,
          let typeValue = userInfo[AVAudioSessionInterruptionTypeKey] as? UInt,
          let type = AVAudioSession.InterruptionType(rawValue: typeValue) else {
        return
    }

    switch type {
    case .began:
        handleInterruptionBegan()

    case .ended:
        guard let optionsValue = userInfo[AVAudioSessionInterruptionOptionKey] as? UInt else {
            return
        }
        let options = AVAudioSession.InterruptionOptions(rawValue: optionsValue)
        let reason = extractInterruptionReason(from: userInfo)

        if reason == .appWasSuspended {
            // User navigated away. Do NOT auto-resume.
            return
        }

        if options.contains(.shouldResume) {
            handleInterruptionEndedWithResume()
        } else {
            handleInterruptionEndedWithoutResume()
        }

    @unknown default:
        break
    }
}

private func extractInterruptionReason(from userInfo: [AnyHashable: Any]) -> AVAudioSession.InterruptionReason? {
    guard let reasonValue = userInfo[AVAudioSessionInterruptionReasonKey] as? UInt else {
        return nil
    }
    return AVAudioSession.InterruptionReason(rawValue: reasonValue)
}
```

## Interruption Response Actions

### On `.began`

```swift
private func handleInterruptionBegan() {
    // System has already stopped audio. Save state.
    for (index, player) in allLayers.enumerated() {
        savedPositions[index] = player.currentTime
    }
    isInterrupted = true
    // Update UI to paused state (dispatch to main if needed)
}
```

### On `.ended` with Resume

```swift
private func handleInterruptionEndedWithResume() {
    isInterrupted = false
    do {
        let session = AVAudioSession.sharedInstance()
        try session.setCategory(.playback, mode: .default)
        try session.setActive(true)

        for (index, player) in allLayers.enumerated() {
            if let savedTime = savedPositions[index] {
                player.currentTime = savedTime
            }
            guard player.prepareToPlay() else {
                rebuildPlayer(at: index)
                continue
            }
            player.play()
        }
        savedPositions.removeAll()
    } catch {
        // Log structured error. Do not silently swallow.
    }
}
```

### On `.ended` without Resume

```swift
private func handleInterruptionEndedWithoutResume() {
    isInterrupted = false
    // Remain paused. UI already shows paused state.
    // User must explicitly press play.
}
```

## No `.ended` Notification Received

Phone call accepted or app backgrounded during interruption. Detect on foreground return:

```swift
func sceneDidBecomeActive(_ scene: UIScene) {
    if isInterrupted {
        // Interruption ended without notification.
        // Present play button. Do not auto-resume.
        isInterrupted = false
    }
}
```

## AVAudioEngine Configuration Change Coordination

When using AVAudioEngine, the configuration change notification can fire after interruption `.ended`. A restart at `.ended` time may fail if the audio hardware reconfigured.

```swift
private var isInterrupted = false
private var configurationChangePending = false

@objc private func handleInterruption(_ notification: Notification) {
    // ... extract type ...
    switch type {
    case .began:
        isInterrupted = true
        engine.pause()
    case .ended:
        isInterrupted = false
        if configurationChangePending {
            rebuildAudioGraph()
            configurationChangePending = false
        }
        attemptEngineRestart()
    @unknown default:
        break
    }
}

@objc private func handleEngineConfigChange(_ notification: Notification) {
    if isInterrupted {
        // Defer rebuild until interruption ends
        configurationChangePending = true
    } else {
        rebuildAudioGraph()
        attemptEngineRestart()
    }
}
```

## Media Server Reset Handler

The media server crash is rare but catastrophic. All audio objects become orphaned.

```swift
@objc private func handleMediaServicesReset(_ notification: Notification) {
    // 1. Tear down everything. Ignore errors.
    allLayers.forEach { $0.delegate = nil }
    allLayers.removeAll()
    engine?.stop()
    engine = nil

    // 2. Reset internal state
    savedPositions.removeAll()
    isInterrupted = false

    // 3. Reconfigure session
    let session = AVAudioSession.sharedInstance()
    try? session.setCategory(.playback, mode: .default)

    // 4. Recreate audio objects
    setupAudioPlayers()

    // 5. Do NOT auto-activate or auto-play.
    // Wait for user action.
}
```

## Media Services Lost Handler

Fires when the media server is in the process of crashing. Audio objects are not yet fully invalidated but will be.

```swift
@objc private func handleMediaServicesLost(_ notification: Notification) {
    // Stop all playback immediately. Do not attempt recovery here.
    // Recovery happens in the mediaServicesWereReset handler.
    allLayers.forEach { $0.stop() }
    isInterrupted = true
}
```

## Secondary Audio Hint Handler

Monitors whether your secondary audio should yield to another non-mixable session:

```swift
@objc private func handleSecondaryAudioHint(_ notification: Notification) {
    guard let userInfo = notification.userInfo,
          let hintValue = userInfo[AVAudioSessionSilenceSecondaryAudioHintTypeKey] as? UInt,
          let hint = AVAudioSession.SilenceSecondaryAudioHintType(rawValue: hintValue) else {
        return
    }

    switch hint {
    case .begin:
        // Another non-mixable session is active. Duck or pause secondary audio.
        break
    case .end:
        // Other session ended. Resume secondary audio if appropriate.
        break
    @unknown default:
        break
    }
}
```

Only relevant when using `.mixWithOthers` and your audio is secondary to the user's primary content.

## Route Change Handler

```swift
@objc private func handleRouteChange(_ notification: Notification) {
    guard let userInfo = notification.userInfo,
          let reasonValue = userInfo[AVAudioSessionRouteChangeReasonKey] as? UInt,
          let reason = AVAudioSession.RouteChangeReason(rawValue: reasonValue) else {
        return
    }

    switch reason {
    case .oldDeviceUnavailable:
        guard let previousRoute = userInfo[AVAudioSessionRouteChangePreviousRouteKey]
                as? AVAudioSessionRouteDescription else { return }

        let headphoneTypes: Set<AVAudioSession.Port> = [
            .headphones, .bluetoothA2DP, .bluetoothLE, .bluetoothHFP
        ]
        let hadHeadphones = previousRoute.outputs.contains {
            headphoneTypes.contains($0.portType)
        }
        if hadHeadphones {
            pauseAllLayers()
        }

    case .newDeviceAvailable:
        // Do NOT auto-resume. User decides.
        break

    case .routeConfigurationChange:
        // Hardware properties may have changed.
        let session = AVAudioSession.sharedInstance()
        let newSampleRate = session.sampleRate
        let newChannels = session.outputNumberOfChannels
        // Reconfigure audio processing if these changed.
        _ = (newSampleRate, newChannels)

    default:
        break
    }
}
```
