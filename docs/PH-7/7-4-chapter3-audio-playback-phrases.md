# E7.4: Implement Audio Playback and Floating Phrase Sync

**Phase:** 7 - Chapter 3: The Cassette
**Class:** Feature
**Design Doc Reference:** SS9 (Voice Note Playback, BGM underlay, Floating Key Phrases)
**Dependencies:**

- Phase 2: Navigation Shell and Shared UI (exit criteria met)
- E1.1: Define design system tokens (color and typography tokens available)
- E1.3: Build AudioService (BGM secondary player at volume 0.3, M4A playback)
- E7.1: Build hint screen and permission flow (Chapter3Constants available)
- E7.3: Build tape playback UI (CassetteView, VUMeterView accept progress and audioLevel parameters)
- Asset: voice note M4A file (bundled in app)
- Asset: `bgm-voice-underlay` (available in asset catalog)

---

## Goal

Implement voice note playback via `AVAudioPlayer`, BGM underlay fade-in at the 30-second mark, and the floating key phrase system timed to audio playback position with drift animation.

---

## Scope

### File Inventory

| File                                                      | Action | Responsibility                                                                                                |
| --------------------------------------------------------- | ------ | ------------------------------------------------------------------------------------------------------------- |
| `OttieExpress/Chapters/Chapter3/TapePlayerView.swift`     | Create | Compose CassetteView and VUMeterView with audio playback driving progress and level, overlay floating phrases |
| `OttieExpress/Chapters/Chapter3/FloatingPhraseView.swift` | Create | Single floating phrase with fade-in, upward drift, and fade-out animation                                     |
| `OttieExpress/Chapters/Chapter3/PhraseTimingData.swift`   | Create | Typed constant array of phrase text and cue timestamps                                                        |

### Integration Points

**TapePlayerView.swift**

- Imports from: `CassetteView.swift` (E7.3), `VUMeterView.swift` (E7.3), `FloatingPhraseView.swift`, `PhraseTimingData.swift`, `Chapter3Constants.swift`, `Color+DesignTokens.swift`, `Font+DesignTokens.swift`
- Imported by: `Chapter3View.swift` (E7.5)
- State reads: None
- State writes: None (reports playback completion via callback)

**FloatingPhraseView.swift**

- Imports from: `Chapter3Constants.swift`, `Color+DesignTokens.swift`, `Font+DesignTokens.swift`
- Imported by: `TapePlayerView.swift`
- State reads: None
- State writes: None

**PhraseTimingData.swift**

- Imports from: None (pure data)
- Imported by: `TapePlayerView.swift`
- State reads: None
- State writes: None

---

## Out of Scope

- VU meter needle rendering and reel animation implementation (covered by E7.3; this epic provides audio level and progress values)
- Voice recognition and microphone input (covered by E7.2; this epic handles post-recognition playback)
- Cassette engagement transition (covered by E7.5; this epic assumes the tape playing screen is already visible)
- Audio interruption handling, pause/resume on system interruption (deferred to Phase 11 cross-cutting hardening)
- Final line typewriter after playback ends (covered by E7.5)
- Extending AudioService secondary player API with new volume fade curve options (AudioService interface owned by E1.3)
- Fine-tuning phrase cue timestamps to final voice note edit (requires final audio asset; timing constants are adjustable)

---

## Definition of Done

- [ ] Voice note M4A file loads from app bundle and plays via `AVAudioPlayer`
- [ ] `TapePlayerView` drives `CassetteView.progress` with normalized playback position (`currentTime / duration`, range 0.0 to 1.0)
- [ ] `TapePlayerView` drives `VUMeterView.audioLevel` with `AVAudioPlayer` average power metering (normalized 0.0 to 1.0)
- [ ] `AVAudioPlayer.isMeteringEnabled` is set to `true` and `updateMeters()` is called on a display-link-frequency timer
- [ ] `bgm-voice-underlay` begins playback via AudioService secondary player at volume 0.0 and fades to volume 0.3 at the 30-second playback mark
- [ ] BGM underlay does not compete with the voice note (voice note retains primary audio priority)
- [ ] Floating phrase "my person" fades in and drifts upward at approximately 38 seconds
- [ ] Floating phrase "my peace" fades in and drifts upward at approximately 42 seconds
- [ ] Floating phrase "a love that feels like home" fades in and drifts upward at approximately 46 seconds
- [ ] Floating phrase "the future I want" fades in and drifts upward at approximately 52 seconds
- [ ] Each floating phrase renders in New York Italic 19pt, `warm-gold`, horizontally centered
- [ ] Each floating phrase fades in over 0.8s, drifts upward by a defined vertical offset, then fades out over 0.8s
- [ ] `PhraseTimingData` stores all phrase strings and cue timestamps as typed constants, adjustable without code changes to view logic
- [ ] `TapePlayerView` reports playback completion via callback when `AVAudioPlayer` finishes

---

## Implementation Notes

Per design doc SS9, the voice note file is bundled in the app, not streamed. Use `AVAudioPlayer(contentsOf:)` with the bundle URL. Enable metering with `isMeteringEnabled = true` to support the VU meter. Use a `CADisplayLink` or `TimelineView(.animation)` to poll `updateMeters()` and `currentTime` at display refresh rate, feeding normalized values to `CassetteView` and `VUMeterView`.

Audio level normalization: `AVAudioPlayer.averagePower(forChannel:)` returns dBFS values (typically -160 to 0). Map this to 0.0-1.0 using a clamped linear interpolation from a floor constant (e.g., -50 dBFS = 0.0) to a ceiling constant (e.g., -5 dBFS = 1.0). Define floor and ceiling as named constants in `Chapter3Constants`.

For the BGM underlay at the 30-second mark: use `AudioService` secondary player API (E1.3 exposes volume control for secondary player). Start `bgm-voice-underlay` playback at volume 0.0 when the voice note begins, then animate volume to 0.3 when `currentTime >= 30.0`. The fade duration is a named constant.

`PhraseTimingData` defines a struct with `text: String` and `cueTime: TimeInterval` fields. The four phrases and their approximate cue times are stored as a static array. When the final voice note duration is known, these values are adjusted in this single file without touching any view code.

Floating phrase drift uses an implicit SwiftUI animation. Each phrase view starts with `opacity(0)` and `offset(y: driftStart)`. On cue, transition to `opacity(1)` and `offset(y: driftEnd)` with a spring animation, then after a hold duration, transition to `opacity(0)`. Define `driftStart`, `driftEnd`, hold duration, and fade duration as named constants.

Per SS15 audio management, the `AVAudioSession` category must be `.playback` during voice note playback. The transition from `.record` (used during E7.2 voice recognition) to `.playback` is handled by the E7.5 flow coordinator before this view appears.
