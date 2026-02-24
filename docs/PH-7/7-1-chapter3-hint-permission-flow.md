# E7.1: Build Hint Screen and Permission Flow

**Phase:** 7 - Chapter 3: The Cassette
**Class:** Feature
**Design Doc Reference:** SS9 (Hint Screen, Voice Unlock permissions)
**Dependencies:**

- Phase 2: Navigation Shell and Shared UI (exit criteria met)
- E2.1: Build navigation shell (ChapterRouter routing available)
- E2.3: Build ChapterCard component (ChapterCard renders intro cards)
- E2.4: Build HapticButton component (HapticButton available)
- E1.1: Define design system tokens (color and typography tokens available)
- E1.4: Build HapticService (haptic patterns callable)
- Asset: `ottie-thinking` (available in asset catalog)
- Asset: `cassette-tape` (available in asset catalog for intro card)

---

## Goal

Implement the Chapter 3 intro card, hint screen with Ottie thinking guidance, and the dual-permission request flow for microphone and speech recognition with Settings redirect on denial.

---

## Scope

### File Inventory

| File                                                     | Action | Responsibility                                                                           |
| -------------------------------------------------------- | ------ | ---------------------------------------------------------------------------------------- |
| `OttieExpress/Chapters/Chapter3/HintScreen.swift`        | Create | Hint screen layout with `ottie-thinking`, guidance text, and "I'm ready" button          |
| `OttieExpress/Chapters/Chapter3/PermissionHandler.swift` | Create | Dual-permission orchestration for microphone and speech recognition with denial redirect |
| `OttieExpress/Chapters/Chapter3/Chapter3Constants.swift` | Create | Named constants for Chapter 3 colors, typography, timing, and copy strings               |

### Integration Points

**HintScreen.swift**

- Imports from: `Chapter3Constants.swift`, `PermissionHandler.swift`, `Color+DesignTokens.swift`, `Font+DesignTokens.swift`
- Imported by: `Chapter3View.swift` (E7.5)
- State reads: None (permission state read via `PermissionHandler`)
- State writes: None

**PermissionHandler.swift**

- Imports from: `Speech`, `AVFoundation`
- Imported by: `HintScreen.swift`, `VoiceRecognizer.swift` (E7.2)
- State reads: None
- State writes: None

**Chapter3Constants.swift**

- Imports from: `Color+DesignTokens.swift`, `Font+DesignTokens.swift`
- Imported by: `HintScreen.swift`, `VoiceUnlockView.swift` (E7.2), `CassetteView.swift` (E7.3), `TapePlayerView.swift` (E7.4), `FinalLineView.swift` (E7.5), `Chapter3View.swift` (E7.5)
- State reads: None
- State writes: None

---

## Out of Scope

- Voice recognition and `SFSpeechRecognizer` streaming setup (covered by E7.2)
- Cassette ambient display and waveform visualizer during listening (covered by E7.2)
- Tape playback, VU meter, reel animation (covered by E7.3 and E7.4)
- Haptic escalation patterns during voice recognition (covered by E7.2, using HapticService from E1.4)
- Audio playback of `sfx-tape-static` during ambient listening state (covered by E7.2)
- Chapter 3 flow coordinator wiring hint screen into the full chapter sequence (covered by E7.5)
- Adding new haptic patterns to HapticService for Chapter 3 interactions (HapticService interface owned by E1.4)
- Animation polish for screen transitions between hint and voice unlock (deferred to Phase 11 hardening)

---

## Definition of Done

- [ ] Chapter 3 intro card renders via `ChapterCard` with "Chapter 3" label, "The Cassette" title, "Some things are better said out loud." body, and static `cassette-tape` asset
- [ ] Hint screen displays `ottie-thinking` asset centered on a full dark (`night-base`) background
- [ ] Guidance text renders in New York Regular 17pt, `parchment`: "Before you press play, there's something you need to say. Go back to Dinesh and ask him about the magic words."
- [ ] "I'm ready" button renders at screen bottom with `midnight-navy` fill and `muted-gray` text label
- [ ] Tapping "I'm ready" triggers sequential permission requests for microphone (`AVAudioSession.requestRecordPermission`) and speech recognition (`SFSpeechRecognizer.requestAuthorization`)
- [ ] When both permissions are granted, `PermissionHandler` reports authorized status and the flow advances to the voice unlock state
- [ ] When either permission is denied, a centered denial screen renders with text in New York Regular 17pt, `parchment`: "To continue, Ottie needs to hear you. Open Settings to allow microphone access."
- [ ] Denial screen displays an "Open Settings" button in `rose-blush` that calls `UIApplication.shared.open(URL(string: UIApplication.openSettingsURLString)!)`
- [ ] On return from Settings with permissions now granted, `PermissionHandler` detects the change and the flow resumes to voice unlock
- [ ] `Chapter3Constants` defines named constants for all copy strings, font configurations, and timing values used in this epic

---

## Implementation Notes

Per design doc SS9, the `NSMicrophoneUsageDescription` must read "Ottie needs to hear you say the magic words to unlock this chapter." and the `NSSpeechRecognitionUsageDescription` must read "Ottie is listening for something special." These are set in the Xcode target configuration (Phase 0 E0.1 established the plist entries).

`PermissionHandler` must be `@MainActor` since it drives UI state transitions. Use `SFSpeechRecognizer.authorizationStatus()` and `AVAudioSession.sharedInstance().recordPermission` to check current status before requesting, avoiding redundant prompts. On return from Settings, monitor permission state in `scenePhase` changes (`.active`) to detect grant without requiring user action.

The "I'm ready" button uses `HapticButton` from E2.4 with a medium impact haptic on tap per SS15 haptic map conventions.

Per SS3 Global Rules, no em dashes in any copy string.
