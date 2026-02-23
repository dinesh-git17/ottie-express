# Streaming Recognition Patterns Reference

Complete AVAudioEngine + SFSpeechRecognizer integration patterns for live phrase detection.

---

## Full Recognition Session Lifecycle

```
┌─────────────────────────────────────────────────────────────┐
│                    RECOGNITION SESSION                       │
│                                                              │
│  SETUP                                                       │
│  ├─ Configure audio session (.playAndRecord, .measurement)   │
│  ├─ Activate audio session                                   │
│  ├─ Create SFSpeechAudioBufferRecognitionRequest             │
│  ├─ Configure request (partial results, on-device, hints)    │
│  ├─ Query inputNode.outputFormat(forBus: 0)                  │
│  ├─ Install tap on inputNode bus 0                           │
│  ├─ Prepare audio engine                                     │
│  ├─ Start audio engine                                       │
│  ├─ Start recognition task                                   │
│  └─ Start timeout timer                                      │
│                                                              │
│  ACTIVE                                                      │
│  ├─ Tap delivers buffers → request.append(buffer)            │
│  ├─ Result handler fires with partial results                │
│  ├─ Check partial text for target phrase                     │
│  └─ On match OR isFinal OR error OR timeout → TEARDOWN       │
│                                                              │
│  TEARDOWN                                                    │
│  ├─ Invalidate timeout timer                                 │
│  ├─ Stop audio engine                                        │
│  ├─ Remove input tap (bus 0)                                 │
│  ├─ Call endAudio() on request                               │
│  ├─ Cancel recognition task                                  │
│  ├─ Nil request reference                                    │
│  ├─ Nil task reference                                       │
│  └─ Transition audio session (if playback follows)           │
└─────────────────────────────────────────────────────────────┘
```

## Setup Implementation

```swift
private var audioEngine = AVAudioEngine()
private var recognitionRequest: SFSpeechAudioBufferRecognitionRequest?
private var recognitionTask: SFSpeechRecognitionTask?
private var recognitionTimer: Timer?

private let recognitionTimeoutSeconds: TimeInterval = 30

func startRecognition(
    recognizer: SFSpeechRecognizer,
    targetPhrases: [String],
    onPhraseDetected: @escaping (String) -> Void,
    onTimeout: @escaping () -> Void,
    onError: @escaping (Error) -> Void
) throws {
    // Teardown any prior session (idempotent)
    stopRecognition()

    // Audio session
    let session = AVAudioSession.sharedInstance()
    try session.setCategory(.playAndRecord, mode: .measurement, options: [.defaultToSpeaker])
    try session.setActive(true, options: .notifyOthersOnDeactivation)

    // Request
    let request = SFSpeechAudioBufferRecognitionRequest()
    request.shouldReportPartialResults = true
    request.requiresOnDeviceRecognition = recognizer.supportsOnDeviceRecognition
    request.taskHint = .confirmation
    request.contextualStrings = targetPhrases
    self.recognitionRequest = request

    // Tap
    let inputNode = audioEngine.inputNode
    let recordingFormat = inputNode.outputFormat(forBus: 0)

    guard recordingFormat.sampleRate > 0, recordingFormat.channelCount > 0 else {
        let msg = "Invalid audio input format: \(recordingFormat)"
        throw RecognitionError.invalidInputFormat(msg)
    }

    inputNode.installTap(onBus: 0, bufferSize: 1024, format: recordingFormat) { [weak request] buffer, _ in
        request?.append(buffer)
    }

    // Engine
    audioEngine.prepare()
    try audioEngine.start()

    // Task
    let lowercasedPhrases = targetPhrases.map { $0.lowercased() }

    recognitionTask = recognizer.recognitionTask(with: request) { [weak self] result, error in
        guard let self else { return }

        var shouldStop = false

        if let result {
            let text = result.bestTranscription.formattedString.lowercased()

            if let matched = lowercasedPhrases.first(where: { text.contains($0) }) {
                shouldStop = true
                DispatchQueue.main.async { onPhraseDetected(matched) }
            }

            if result.isFinal {
                shouldStop = true
            }
        }

        if let error {
            shouldStop = true
            let nsError = error as NSError
            // Error 1110 = no speech detected. Not fatal for interactive flows.
            if nsError.domain == "kAFAssistantErrorDomain" && nsError.code == 1110 {
                // Silence — handle as timeout, not error
            } else {
                DispatchQueue.main.async { onError(error) }
            }
        }

        if shouldStop {
            self.stopRecognition()
        }
    }

    // Timeout
    recognitionTimer = Timer.scheduledTimer(withTimeInterval: recognitionTimeoutSeconds, repeats: false) { [weak self] _ in
        self?.stopRecognition()
        onTimeout()
    }
}
```

## Teardown Implementation

```swift
func stopRecognition() {
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

## Format Validation

The `inputNode.outputFormat(forBus: 0)` call can return a zero-sample-rate format on some devices intermittently. Always validate:

```swift
let format = inputNode.outputFormat(forBus: 0)
guard format.sampleRate > 0 else {
    // Retry after a short delay or report error
    return
}
guard format.channelCount > 0 else {
    // Invalid format — microphone not available
    return
}
```

## Partial Result Behavior

- `bestTranscription.formattedString` is **cumulative** — the entire recognized text so far.
- Each callback replaces the previous text, not appends to it.
- `transcriptions` array contains alternatives ordered by confidence.
- Individual segments available via `bestTranscription.segments`, each with `confidence`, `timestamp`, `duration`.

## Error Code Reference

| Domain                    | Code | Meaning                                   | Action                                            |
| ------------------------- | ---- | ----------------------------------------- | ------------------------------------------------- |
| `kAFAssistantErrorDomain` | 203  | Rate limit / resource limit               | Switch to on-device or backoff                    |
| `kAFAssistantErrorDomain` | 209  | General recognition failure               | Retry once, then surface error                    |
| `kAFAssistantErrorDomain` | 1101 | XPC communication failure (service crash) | Full teardown and retry                           |
| `kAFAssistantErrorDomain` | 1107 | XPC failure variant (common on simulator) | Ignore on simulator, retry on device              |
| `kAFAssistantErrorDomain` | 1110 | No speech detected                        | Expected in interactive flows. Handle as timeout. |

## Interruption During Recognition

If `AVAudioSession.interruptionNotification` fires during active recognition:

1. On `.began`: Call `stopRecognition()`. Save the recognition state (was it mid-phrase?).
2. On `.ended` with `.shouldResume`: Restart recognition from scratch. Do not attempt to resume a cancelled task.
3. On `.ended` without `.shouldResume`: Stay stopped. Show "tap to try again" UI.

Recognition tasks cannot be paused and resumed. After any interruption, a new session must be created.

## Prohibited Patterns

- Reusing a `SFSpeechAudioBufferRecognitionRequest` across multiple sessions. Create a new one each time.
- Calling `recognitionTask.finish()` when you intend to discard results — use `cancel()`.
- Installing the tap after starting the engine (misses initial buffers, may crash on some devices).
- Using `DispatchQueue.main.asyncAfter` for timeout instead of `Timer` (not cancellable without stored work item).
- Storing `audioEngine.inputNode` as a separate strong property (it can change after engine configuration changes).
- Ignoring the `error` parameter in the result handler — both `result` and `error` can be non-nil in the same callback.
