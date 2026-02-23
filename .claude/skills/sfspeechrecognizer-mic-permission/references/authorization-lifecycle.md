# Authorization Lifecycle Reference

Complete dual-permission authorization flow for SFSpeechRecognizer + microphone access.

---

## State Machine

```
                    ┌─────────────────┐
                    │  NOT_DETERMINED  │
                    └────────┬────────┘
                             │
                    requestAuthorization()
                             │
               ┌─────────────┼─────────────┐
               ▼             ▼              ▼
         ┌──────────┐ ┌──────────┐  ┌────────────┐
         │AUTHORIZED│ │  DENIED  │  │ RESTRICTED │
         └────┬─────┘ └────┬─────┘  └─────┬──────┘
              │            │               │
     requestRecordPerm     │          Show explanation
              │        Show Settings   (no redirect)
         ┌────┴────┐    redirect
         ▼         ▼       │
     ┌───────┐ ┌───────┐   │
     │GRANTED│ │DENIED │   │
     └───┬───┘ └───┬───┘   │
         │         │        │
    PROCEED   Show Settings │
    to setup    redirect    │
                            │
         All denied/restricted paths
         offer non-voice fallback
```

## Dual-Permission Sequencing

### Phase 1: Speech Authorization

```swift
func requestSpeechAuthorization(completion: @escaping (SFSpeechRecognizerAuthorizationStatus) -> Void) {
    let currentStatus = SFSpeechRecognizer.authorizationStatus()

    switch currentStatus {
    case .authorized:
        completion(.authorized)
    case .notDetermined:
        SFSpeechRecognizer.requestAuthorization { status in
            DispatchQueue.main.async {
                completion(status)
            }
        }
    case .denied, .restricted:
        DispatchQueue.main.async {
            completion(currentStatus)
        }
    @unknown default:
        DispatchQueue.main.async {
            completion(.denied)
        }
    }
}
```

### Phase 2: Microphone Permission (only after speech is authorized)

```swift
func requestMicrophonePermission(completion: @escaping (Bool) -> Void) {
    let session = AVAudioSession.sharedInstance()

    switch session.recordPermission {
    case .granted:
        completion(true)
    case .undetermined:
        session.requestRecordPermission { granted in
            DispatchQueue.main.async {
                completion(granted)
            }
        }
    case .denied:
        DispatchQueue.main.async {
            completion(false)
        }
    @unknown default:
        DispatchQueue.main.async {
            completion(false)
        }
    }
}
```

### Phase 3: Combined Check

```swift
func ensureVoicePermissions(completion: @escaping (VoicePermissionResult) -> Void) {
    requestSpeechAuthorization { speechStatus in
        switch speechStatus {
        case .authorized:
            self.requestMicrophonePermission { micGranted in
                if micGranted {
                    completion(.ready)
                } else {
                    completion(.microphoneDenied)
                }
            }
        case .denied:
            completion(.speechDenied)
        case .restricted:
            completion(.speechRestricted)
        case .notDetermined:
            completion(.speechDenied) // Should not reach here post-request
        @unknown default:
            completion(.speechDenied)
        }
    }
}
```

## Result Enum

```swift
enum VoicePermissionResult {
    case ready
    case speechDenied
    case speechRestricted
    case microphoneDenied
}
```

## Recovery UI Decision Table

| Result              | Alert Title                      | Alert Message                                | Primary Action                            | Secondary Action         |
| ------------------- | -------------------------------- | -------------------------------------------- | ----------------------------------------- | ------------------------ |
| `.speechDenied`     | "Speech Recognition Needed"      | Context-specific reason                      | "Open Settings" → `openSettingsURLString` | "Continue Without Voice" |
| `.speechRestricted` | "Speech Recognition Unavailable" | "This feature is restricted on this device." | "Continue Without Voice"                  | —                        |
| `.microphoneDenied` | "Microphone Access Needed"       | Context-specific reason                      | "Open Settings" → `openSettingsURLString` | "Continue Without Voice" |

## Re-Check on Return from Settings

```swift
func sceneDidBecomeActive(_ scene: UIScene) {
    guard isWaitingForPermissionChange else { return }

    let speechStatus = SFSpeechRecognizer.authorizationStatus()
    let micStatus = AVAudioSession.sharedInstance().recordPermission

    if speechStatus == .authorized && micStatus == .granted {
        isWaitingForPermissionChange = false
        resumeVoiceFlow()
    }
}
```

## Prohibited Patterns

- Requesting `requestAuthorization` when status is `.denied` — it returns `.denied` without showing a prompt. Wasted callback.
- Caching authorization status across app launches — it can change externally in Settings.
- Requesting microphone permission before speech authorization — confuses users with unexpected double-prompt ordering.
- Calling `requestRecordPermission` on a background thread and touching UI in the callback without dispatching to main.
- Showing a generic "permissions needed" alert that does not distinguish between speech and microphone denials.
