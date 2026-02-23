---
name: sfspeechrecognizer-mic-permission
description: Enforce production-grade SFSpeechRecognizer streaming recognition, dual-permission authorization flows, AVAudioEngine input lifecycle, and audio session coordination for iOS voice-input systems. Use when writing or reviewing SFSpeechRecognizer setup, SFSpeechAudioBufferRecognitionRequest streaming, microphone permission handling, speech authorization state machines, AVAudioEngine input tap management, recognition teardown sequences, or audio session category transitions between recording and playback. Triggers on speech recognition code generation, voice-input architecture, permission flow implementation, recognition result handling, or any task producing SFSpeechRecognizer/AVAudioEngine input code.
---

# SFSpeechRecognizer & Microphone Permission Flow

Governing standard for all speech recognition and microphone permission code in this repository. Every recognition pipeline must handle authorization deterministically, stream audio without leaks, tear down cleanly, and yield the audio session to subsequent playback systems without silent failures.

---

## 1. Speech Recognition Architecture Philosophy

Speech recognition is a transient input system. It acquires shared hardware, opens a server or on-device ML connection, and must release everything deterministically when finished.

**Rules:**

- Recognition sessions are short-lived. Start when needed, stop immediately after phrase detection or timeout. No persistent listeners.
- Both permissions (speech + microphone) are verified before any audio infrastructure is created. No optimistic setup.
- Every recognition session has a bounded lifetime. Implement a hard timeout to prevent runaway sessions.
- AVAudioEngine input taps are installed exactly once per session and removed on every exit path. No orphaned taps.
- Audio session category transitions between recognition (`.playAndRecord`) and playback (`.playback`) follow an explicit deactivate-reconfigure-reactivate sequence. No implicit state carryover.
- Silent failures are the worst class of recognition bug. Every code path must either produce a transcription result or produce a logged error.

---

## 2. Info.plist Requirements (BLOCKING)

Both keys are mandatory. Missing either causes a runtime crash or silent permission failure:

| Key                                   | Purpose                                       | Crash Behavior                                                      |
| ------------------------------------- | --------------------------------------------- | ------------------------------------------------------------------- |
| `NSSpeechRecognitionUsageDescription` | Explains speech recognition usage to the user | App crashes on `SFSpeechRecognizer.requestAuthorization` if missing |
| `NSMicrophoneUsageDescription`        | Explains microphone access to the user        | App crashes on `AVAudioSession.requestRecordPermission` if missing  |

Usage strings must be specific to the app context. Generic strings ("This app needs speech recognition") result in App Store rejection.

---

## 3. Authorization Flow

See [references/authorization-lifecycle.md](references/authorization-lifecycle.md) for full state machine and handler implementation.

### 3.1 Permission Request Order (BLOCKING)

Request speech authorization first, then microphone permission. Both must be granted before any audio infrastructure is created.

```
1. SFSpeechRecognizer.requestAuthorization { speechStatus in }
2. AVAudioSession.sharedInstance().requestRecordPermission { micGranted in }
3. Only if BOTH granted → proceed to recognition setup
```

**Rationale:** Speech recognition is the less familiar permission. Presenting it first lets the usage string explain the feature before the microphone prompt appears. Requesting microphone first causes users to deny the unexpected speech prompt.

### 3.2 Authorization State Matrix (BLOCKING)

| Speech Status    | Mic Status | Action                                                           |
| ---------------- | ---------- | ---------------------------------------------------------------- |
| `.authorized`    | `true`     | Proceed to recognition                                           |
| `.authorized`    | `false`    | Show mic-denied recovery UI                                      |
| `.denied`        | any        | Show speech-denied recovery UI                                   |
| `.restricted`    | any        | Show restriction explanation. No Settings redirect.              |
| `.notDetermined` | any        | Request authorization (should not reach this state post-request) |

### 3.3 Threading Constraint (BLOCKING)

Both authorization callbacks execute on **arbitrary threads**. All UI updates and state mutations must dispatch to the main queue explicitly:

```swift
SFSpeechRecognizer.requestAuthorization { status in
    DispatchQueue.main.async {
        // UI updates here
    }
}
```

### 3.4 Pre-Flight Check Pattern

Check authorization status before entering any view or chapter that requires voice input. Do not wait until the user taps a record button:

```swift
let speechStatus = SFSpeechRecognizer.authorizationStatus()
let micStatus = AVAudioSession.sharedInstance().recordPermission

guard speechStatus == .authorized, micStatus == .granted else {
    // Route to permission request or recovery flow
    return
}
```

Re-check on `sceneDidBecomeActive` — the user may have changed permissions in Settings and returned.

### 3.5 Prohibited Authorization Patterns

- Requesting permissions inside `viewDidLoad` without checking current status first.
- Proceeding with recognition setup before both permissions are confirmed.
- Calling `requestAuthorization` when status is already `.denied` (no-op, wastes a callback cycle).
- Showing a Settings redirect for `.restricted` status (user cannot change it).
- Storing authorization status in a persisted cache (it can change externally at any time).

---

## 4. Permission Recovery UX

### 4.1 Settings Redirect (BLOCKING)

When either permission is `.denied`, present an alert explaining why the permission is needed, with a button that opens Settings:

```swift
guard let settingsURL = URL(string: UIApplication.openSettingsURLString) else { return }
UIApplication.shared.open(settingsURL)
```

`UIApplication.openSettingsURLString` opens the app's specific settings page, not the general Settings app.

### 4.2 Recovery Flow Rules

- Always provide an alternative path (tap-to-continue, skip) if voice input is denied. No dead-end states.
- After returning from Settings, re-check permissions in `sceneDidBecomeActive`. Do not assume the user granted access.
- For `.restricted` status: show an explanation that the feature is unavailable due to device restrictions. No Settings redirect (the user cannot change MDM/Screen Time restrictions).
- Track whether the permission alert has been shown this session to avoid repeated prompts on every screen transition.

### 4.3 Narrative App Integration

For apps where voice input is part of a story flow (e.g., Ottie Express Chapter 3):

1. Pre-flight permissions before the chapter begins, not mid-scene.
2. If denied, offer a non-voice fallback that preserves narrative continuity.
3. If the user grants permission via Settings mid-chapter, detect on return and resume the voice flow.
4. Never block the narrative permanently on a denied permission.

---

## 5. SFSpeechRecognizer Configuration

### 5.1 Recognizer Initialization

```swift
guard let recognizer = SFSpeechRecognizer(locale: Locale(identifier: "en-US")) else {
    // Locale not supported — handle gracefully
    return
}
recognizer.delegate = self
```

- `SFSpeechRecognizer(locale:)` is failable. Always use `guard let`.
- Set the delegate to monitor `availabilityDidChange`. The recognizer can become unavailable at runtime due to network loss, Siri configuration changes, or system resource pressure.
- Store the recognizer as a strong property. Do not recreate per session.

### 5.2 Availability Monitoring (BLOCKING)

```swift
func speechRecognizer(_ speechRecognizer: SFSpeechRecognizer, availabilityDidChange available: Bool) {
    // Update UI to reflect availability
    // If recognition is active and available == false, cancel gracefully
}
```

### 5.3 On-Device vs Server Recognition Decision Matrix

| Factor           | On-Device                          | Server                              |
| ---------------- | ---------------------------------- | ----------------------------------- |
| Duration limit   | None                               | ~60 seconds per task                |
| Rate limiting    | None                               | Per-device and per-app daily limits |
| Network required | No                                 | Yes                                 |
| Accuracy         | Lower (improving each iOS version) | Higher (larger models)              |
| Latency          | Lower (local inference)            | Higher (network round-trip)         |
| Battery          | Higher CPU, no radio               | Lower CPU, higher radio             |
| Privacy          | Audio stays on device              | Audio transmitted to Apple          |

**For phrase detection (Ottie Express):** Use on-device recognition. Short phrases, no duration concern, deterministic latency, no network dependency.

```swift
request.requiresOnDeviceRecognition = true
```

**Guard:** Always check `supportsOnDeviceRecognition` before setting `requiresOnDeviceRecognition = true`. The request fails with an error if the device does not support it — there is no silent fallback.

### 5.4 Request Configuration

```swift
let request = SFSpeechAudioBufferRecognitionRequest()
request.shouldReportPartialResults = true
request.requiresOnDeviceRecognition = recognizer.supportsOnDeviceRecognition
request.taskHint = .confirmation   // Short phrase detection
request.contextualStrings = ["open sesame", "magic word"]  // Boost target phrases
```

- `shouldReportPartialResults = true` is mandatory for live phrase detection.
- `taskHint = .confirmation` optimizes for short utterances.
- `contextualStrings` accepts domain-specific phrases (keep under 100 entries) to boost recognition accuracy.
- `addsPunctuation` (iOS 16+) is unnecessary for phrase detection — leave at default.

---

## 6. Streaming Recognition Lifecycle

See [references/streaming-recognition-patterns.md](references/streaming-recognition-patterns.md) for full implementation patterns.

### 6.1 AVAudioEngine Setup (BLOCKING)

```
1. Configure audio session for recording
2. Create AVAudioEngine (store as strong property)
3. Get inputNode reference
4. Query outputFormat(forBus: 0) from inputNode
5. Install tap on inputNode bus 0
6. Prepare engine
7. Start engine
```

**Critical:** Query `inputNode.outputFormat(forBus: 0)` immediately before `installTap`. The format reflects hardware state and can change after route changes.

### 6.2 Audio Session for Recognition (BLOCKING)

```swift
let session = AVAudioSession.sharedInstance()
try session.setCategory(.playAndRecord, mode: .measurement, options: [.defaultToSpeaker])
try session.setActive(true, options: .notifyOthersOnDeactivation)
```

- Category `.playAndRecord` is required for microphone input.
- Mode `.measurement` minimizes system signal processing (AGC, noise suppression) for cleaner recognition input.
- `.defaultToSpeaker` routes output to speaker instead of earpiece.

### 6.3 Input Tap Installation (BLOCKING)

```swift
let inputNode = audioEngine.inputNode
let recordingFormat = inputNode.outputFormat(forBus: 0)

inputNode.installTap(onBus: 0, bufferSize: 1024, format: recordingFormat) { [weak request] buffer, _ in
    request?.append(buffer)
}
```

**Rules:**

- Buffer size 1024 balances CPU overhead and latency for speech recognition.
- Capture `request` weakly in the tap closure. The tap outlives the request if teardown races occur.
- Only one tap per bus. Installing a second tap without removing the first crashes with `required condition is false: nullptr == Tap()`.
- Never pass a custom format. Always use `inputNode.outputFormat(forBus: 0)`. Format mismatch crashes with `required condition is false: format.sampleRate == hwFormat.sampleRate`.

### 6.4 Recognition Task Management

```swift
recognitionTask = recognizer.recognitionTask(with: request) { [weak self] result, error in
    guard let self else { return }

    if let result {
        let text = result.bestTranscription.formattedString
        // Check for target phrase in text (cumulative, not delta)

        if result.isFinal {
            self.stopRecognition()
        }
    }

    if error != nil {
        self.stopRecognition()
    }
}
```

**Rules:**

- Use `[weak self]` in the result handler. The task retains the closure, creating a retain cycle through `self -> task -> closure -> self`.
- `bestTranscription.formattedString` is cumulative — the entire recognized text so far, not a delta.
- Handle both `isFinal == true` and `error != nil` as termination signals.
- Always call the same cleanup method from both paths. No divergent teardown logic.

### 6.5 Phrase Detection Strategy

For detecting specific phrases in partial results:

```swift
let normalized = result.bestTranscription.formattedString.lowercased()
if targetPhrases.contains(where: { normalized.contains($0) }) {
    // Phrase detected — stop recognition and proceed
    stopRecognition()
    onPhraseDetected()
}
```

- Normalize both the transcription and target phrases to lowercase.
- Use `contains` rather than exact match — partial results may include surrounding words.
- Stop recognition immediately on detection. Do not wait for `isFinal`.

### 6.6 Session Timeout (BLOCKING)

Every recognition session must have a hard timeout:

```swift
private let recognitionTimeoutSeconds: TimeInterval = 30

recognitionTimer = Timer.scheduledTimer(withTimeInterval: recognitionTimeoutSeconds, repeats: false) { [weak self] _ in
    self?.stopRecognition()
    self?.onRecognitionTimeout()
}
```

Without a timeout, a session where the user never speaks runs indefinitely, draining battery and holding the microphone.

---

## 7. Teardown Sequence (BLOCKING)

See [references/streaming-recognition-patterns.md](references/streaming-recognition-patterns.md) for edge cases.

### 7.1 Canonical Teardown Order

```
1. Invalidate timeout timer
2. Stop AVAudioEngine
3. Remove input tap from inputNode (bus 0)
4. Call endAudio() on the recognition request
5. Cancel the recognition task (if not already finishing)
6. Nil the recognition request reference
7. Nil the recognition task reference
8. Deactivate audio session (if transitioning to playback)
```

**Order matters.** Stop the engine before removing the tap. Removing a tap while buffers are actively being delivered causes timing issues on some devices.

### 7.2 Idempotent Cleanup

The teardown method must be safe to call multiple times (from timeout, from `isFinal`, from error, from view dismissal):

```swift
private func stopRecognition() {
    recognitionTimer?.invalidate()
    recognitionTimer = nil

    audioEngine.stop()
    audioEngine.inputNode.removeTap(onBus: 0)

    recognitionRequest?.endAudio()
    recognitionRequest = nil

    recognitionTask?.cancel()
    recognitionTask = nil
}
```

- `audioEngine.stop()` on a stopped engine is a no-op. Safe.
- `removeTap(onBus:)` on a bus with no tap is safe on modern iOS.
- `endAudio()` on a nil request is a no-op (optional chaining). Safe.
- `cancel()` on a nil or already-completed task is a no-op. Safe.

### 7.3 Prohibited Teardown Patterns

- Stopping the engine inside the result handler closure without also removing the tap.
- Calling `finish()` instead of `cancel()` when discarding results (finish processes remaining audio; cancel aborts).
- Forgetting to nil `recognitionRequest` and `recognitionTask` (retains the closure chain, leaks memory).
- Leaving the audio session in `.playAndRecord` after recognition ends.

---

## 8. Audio Session Transition to Playback

See [references/audio-session-transition.md](references/audio-session-transition.md) for the full transition protocol.

### 8.1 Transition Sequence (BLOCKING)

After recognition teardown, before any AVAudioPlayer/AVPlayer playback:

```
1. Verify audioEngine is stopped and tap is removed
2. Deactivate: setActive(false, options: [.notifyOthersOnDeactivation])
3. Reconfigure: setCategory(.playback, mode: .default)
4. Reactivate: setActive(true)
5. Proceed with playback setup
```

**Without this sequence:** AVAudioPlayer/AVPlayer plays with no audible output. The audio session remains configured for `.playAndRecord` with `.measurement` mode, which suppresses normal playback output on some devices.

### 8.2 Deactivation Precondition

`setActive(false)` throws error code 560030580 if any I/O is still running. Verify:

- `audioEngine.isRunning == false`
- No active `AVAudioPlayer` instances are playing
- No active `AVPlayer` instances are playing

### 8.3 Timing

Insert a brief delay (~100ms) between deactivation and reactivation if the transition is immediate. Some devices need time to release hardware resources:

```swift
try session.setActive(false, options: [.notifyOthersOnDeactivation])
try session.setCategory(.playback, mode: .default)
try session.setActive(true)
```

### 8.4 Prohibited Transition Patterns

- Changing category without deactivating first.
- Skipping the deactivation step entirely (leaves session in recording mode).
- Calling `setActive(false)` while the audio engine is still running.
- Omitting `.notifyOthersOnDeactivation` (leaves other apps' audio silenced).

---

## 9. Known Platform Issues

| Issue                                                 | iOS Versions          | Workaround                                                                           |
| ----------------------------------------------------- | --------------------- | ------------------------------------------------------------------------------------ |
| `isFinal` never set to `true`                         | 13.0–13.2             | Fixed in 13.3. Use `endAudio()` + silence timer as fallback.                         |
| On-device recognition clears text after pause         | 18.0                  | Set `requiresOnDeviceRecognition = false` or handle text clearing in result handler. |
| `inputNode.outputFormat` returns zero sample rate     | Intermittent, various | Check format validity before tap installation. Retry after short delay.              |
| Tap crash on second recognition attempt               | All versions          | Always `removeTap(onBus: 0)` before `installTap`.                                    |
| Error 203 (rate limit)                                | Server recognition    | Switch to on-device or implement exponential backoff.                                |
| Error 1110 (no speech detected)                       | All versions          | Expected when user is silent. Handle gracefully, do not treat as fatal.              |
| `AVAudioEngineConfigurationChange` after interruption | All versions          | Track both interruption and config change. Do not restart until both resolved.       |

---

## 10. Code Review Checklist

Reject speech recognition code if ANY are true:

- [ ] `NSSpeechRecognitionUsageDescription` missing from Info.plist
- [ ] `NSMicrophoneUsageDescription` missing from Info.plist
- [ ] Recognition setup proceeds before both permissions are confirmed `.authorized`/`.granted`
- [ ] Authorization callback updates UI without dispatching to main queue
- [ ] `SFSpeechRecognizer(locale:)` used without `guard let` (failable initializer)
- [ ] `availabilityDidChange` delegate not implemented
- [ ] `supportsOnDeviceRecognition` not checked before setting `requiresOnDeviceRecognition = true`
- [ ] Input tap format is not `inputNode.outputFormat(forBus: 0)`
- [ ] Tap installed without removing previous tap first
- [ ] Result handler closure captures `self` strongly (retain cycle)
- [ ] `recognitionRequest` and `recognitionTask` not nilled on completion
- [ ] No hard timeout on recognition session
- [ ] Audio engine not stopped in teardown
- [ ] Input tap not removed in teardown
- [ ] `endAudio()` not called on the request in teardown
- [ ] Audio session left in `.playAndRecord` after recognition ends
- [ ] `setActive(false)` called while audio engine is still running
- [ ] `.notifyOthersOnDeactivation` omitted from session deactivation
- [ ] No alternative path when permissions are denied (dead-end state)
- [ ] Settings redirect shown for `.restricted` authorization status

---

## References

- `references/authorization-lifecycle.md` — Full authorization state machine, dual-permission sequencing, recovery flows.
- `references/streaming-recognition-patterns.md` — AVAudioEngine + SFSpeechRecognizer integration, tap management, result handling.
- `references/audio-session-transition.md` — Category transitions between recognition and playback, deactivation protocol, silent failure prevention.
