# E1.3: Build AudioService

**Phase:** 1 - Core Systems and Design Foundation
**Class:** Infrastructure
**Design Doc Reference:** SS15 Haptics and Sound Design (Audio Management, Audio Interruption Handling), SS4 Architecture Overview (Dependencies, Technology Stack)
**Dependencies:**

- Phase 0: Project Scaffold (exit criteria met)
- E0.1: Scaffold Xcode project and directory structure (`Services/` directory exists)

---

## Goal

Implement AVAudioPlayer-based audio management with BGM playback, SFX layering, crossfade transitions, volume control, interruption handling, and silent mode compliance.

---

## Scope

### File Inventory

| File                                       | Action | Responsibility                                                                                                                                            |
| ------------------------------------------ | ------ | --------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `OttieExpress/Services/AudioService.swift` | Create | Define `@MainActor` AudioService with BGM playback, SFX layering, 0.8s crossfade, volume control, AVAudioSession configuration, and interruption observer |

### Integration Points

**AudioService.swift**

- Imports from: None within Phase 1 (uses AVFoundation framework only)
- Imported by: None within Phase 1 (consumed by chapter views in Phase 3+ and navigation shell in Phase 2 for BGM lifecycle)
- State reads: None
- State writes: None

---

## Out of Scope

- Audio asset files (M4A BGM tracks, SFX files) bundled in the app (assets are added per-chapter in Phases 3-9 as each chapter's feature epic requires them)
- Voice note playback with exclusive audio session priority and BGM underlay at 0.3 volume (Chapter 3 feature epic E6.1+ handles the cassette playback interaction; this epic provides the underlying `AVAudioPlayer` capability it consumes)
- `SFSpeechRecognizer` or `AVAudioEngine` input tap coordination with audio session (covered by the SFSpeechRecognizer skill and Chapter 3 voice recognition epic)
- Audio ducking for system notifications or phone calls beyond the interruption handler (standard `AVAudioSession` interruption handling covers the required behavior)
- Background audio playback when the app is suspended (SS15 specifies `.playback` category for screen-lock playback, not background mode entitlement)

---

## Definition of Done

- [ ] `AVAudioSession.sharedInstance()` is configured with category `.playback` on service initialization
- [ ] `playBGM(_:)` loads an M4A file from the app bundle by name and starts `AVAudioPlayer` playback
- [ ] Only one BGM track plays at any time; calling `playBGM` while a track is active crossfades from the current track to the new track over 0.8 seconds
- [ ] Crossfade ramps the outgoing player volume from current to 0.0 and the incoming player volume from 0.0 to target over 0.8 seconds concurrently
- [ ] `playSFX(_:)` loads and plays a sound effect file without interrupting or pausing the active BGM track
- [ ] `setBGMVolume(_:)` adjusts the active BGM player's volume to the specified value (range 0.0 to 1.0)
- [ ] `stopBGM()` stops the active BGM player and releases the reference
- [ ] AudioService registers for `AVAudioSession.interruptionNotification` on initialization
- [ ] On interruption began notification, all active players pause
- [ ] On interruption ended notification with `shouldResume` option, all paused players resume from their paused position
- [ ] Project compiles with zero errors after adding AudioService.swift

---

## Implementation Notes

SS15 Audio Management defines the single-BGM constraint: "Only one BGM track playing at any time. Crossfade on BGM transitions (0.8s)." Implement crossfade using two `AVAudioPlayer` instances: a primary (active BGM) and a secondary (incoming BGM). Drive volume ramp via a `Timer` or `withAnimation` block ticking at ~60Hz over 0.8 seconds. After crossfade completes, stop and release the outgoing player, promote the incoming player to primary. Reference: SS15 Audio Management.

SFX playback uses a separate `AVAudioPlayer` instance per effect. SFX players are fire-and-forget: create, play, and release on completion via the `AVAudioPlayerDelegate.audioPlayerDidFinishPlaying` callback. SFX does not interact with BGM volume or crossfade state. Reference: SS15 Audio Management, "SFX can layer on top of BGM."

The secondary player volume control (`setBGMVolume`) supports the Chapter 3 requirement where BGM plays as an underlay at 0.3 volume during voice note playback. This epic exposes the interface; Chapter 3's epic consumes it. Reference: SS15 Audio Management, voice note priority rule.

Audio session category `.playback` enables playback with the screen locked. This does not require the `audio` background mode entitlement; it allows playback to continue when the screen turns off while the app is in the foreground. Reference: SS15 Audio Management, SS4 Technology Stack.

Interruption handling follows SS15 Audio Interruption Handling: register for `AVAudioSession.interruptionNotification`, inspect the `AVAudioSession.InterruptionType`, and pause/resume accordingly. The 5-second resume indicator behavior described in SS15 is a UI concern handled by the consuming view, not this service. This epic provides the pause/resume mechanics. Reference: SS15 Audio Interruption Handling.

Annotate AudioService as `@MainActor` since it manages `AVAudioPlayer` instances that must be created and controlled from the main thread. Reference: SS4 Architecture Overview, concurrency model.
