# Audio Session Transition Reference

Protocol for transitioning the audio session from speech recognition (recording) to playback without silent failures.

---

## The Silent Playback Bug

After speech recognition ends, if the audio session is not explicitly transitioned, subsequent `AVAudioPlayer` or `AVPlayer` playback produces no audible output. The player reports `isPlaying == true` but audio routes through a dead configuration.

**Root cause:** The session remains in `.playAndRecord` category with `.measurement` mode. This mode minimizes signal processing, and on some devices, suppresses speaker output that was routed through the earpiece path during recording.

---

## Transition Protocol

### Step 1: Verify Recognition Teardown is Complete

```swift
assert(!audioEngine.isRunning, "Audio engine must be stopped before session transition")
// recognitionRequest and recognitionTask should be nil
```

### Step 2: Deactivate the Recording Session

```swift
do {
    try AVAudioSession.sharedInstance().setActive(false, options: [.notifyOthersOnDeactivation])
} catch {
    // Error 560030580 = "Deactivating an audio session that has running I/O"
    // If this fires, teardown is incomplete. Do not proceed.
    Logger.audio.error("Session deactivation failed: \(error)")
    return
}
```

**`.notifyOthersOnDeactivation`** tells the system to notify other apps that were interrupted by your recording session. Without this flag, background music and other audio sources remain silenced.

### Step 3: Reconfigure for Playback

```swift
do {
    try AVAudioSession.sharedInstance().setCategory(.playback, mode: .default)
} catch {
    Logger.audio.error("Category reconfiguration failed: \(error)")
    return
}
```

### Step 4: Reactivate for Playback

```swift
do {
    try AVAudioSession.sharedInstance().setActive(true)
} catch {
    Logger.audio.error("Session reactivation failed: \(error)")
    return
}
```

### Step 5: Proceed with Playback Setup

Now `AVAudioPlayer`, `AVPlayer`, or `AVAudioEngine` (in playback mode) will produce audible output.

---

## Complete Transition Function

```swift
func transitionToPlayback() throws {
    guard !audioEngine.isRunning else {
        let msg = "Cannot transition: audio engine is still running"
        throw AudioTransitionError.engineStillRunning(msg)
    }

    let session = AVAudioSession.sharedInstance()

    try session.setActive(false, options: [.notifyOthersOnDeactivation])
    try session.setCategory(.playback, mode: .default)
    try session.setActive(true)
}
```

---

## Timing Considerations

On most devices, the deactivate → reconfigure → reactivate sequence executes synchronously and completes in under 10ms. However:

- **Bluetooth audio routes:** Switching from a Bluetooth microphone input to Bluetooth A2DP output can take 200–500ms as the Bluetooth profile negotiates the change.
- **USB audio interfaces:** Similar delay for profile switching.

For immediate playback after recognition, call `transitionToPlayback()` as soon as recognition stops, then set up players. By the time `prepareToPlay()` and `play()` execute, the transition will have completed.

---

## Integration with AVFoundation Audio Session Skill

This transition protocol feeds directly into the `avfoundation-audio-session` skill's activation model:

1. This skill handles: `.playAndRecord` → deactivation → `.playback` reconfiguration → reactivation.
2. The `avfoundation-audio-session` skill takes over: playback session management, interruption handling, multi-player coordination, route changes.

The boundary is the `setActive(true)` call with `.playback` category. After that point, the audio session is in the `avfoundation-audio-session` skill's domain.

---

## Prohibited Patterns

- Changing category without deactivating first. Some iOS versions silently apply the wrong mode.
- Skipping deactivation entirely and hoping `.playback` category overrides the previous state.
- Calling `setActive(false)` while `audioEngine.isRunning == true`. Always stop first.
- Omitting `.notifyOthersOnDeactivation`. Other apps' audio remains silenced.
- Using `try?` to swallow transition errors. Every failure here causes silent playback — log and surface all errors.
- Attempting to play audio between the `setActive(false)` and `setActive(true)` calls.
