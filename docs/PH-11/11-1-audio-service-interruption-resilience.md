# E11.1: Harden AudioService session lifecycle

**Phase:** 11 - Cross-Cutting Hardening
**Class:** Integration
**Design Doc Reference:** SS15 Haptics and Sound Design (Audio Management, Audio Interruption Handling), SS4 Architecture Overview (State Machine and Resume Semantics)
**Dependencies:**

- Phase 1: Core Systems and Design Foundation (exit criteria met)
- E1.3: Build AudioService (AudioService created with base interruption observer and pause/resume mechanics)
- Phase 3: Chapter 0 (exit criteria met)
- Phase 4: Chapter 4 (exit criteria met)
- Phase 5: Chapter 1 (exit criteria met)
- Phase 6: Chapter 2 (exit criteria met)
- Phase 7: Chapter 3 (exit criteria met, VU meter/reel/floating phrase views consuming AudioService)
- Phase 8: Chapter 5 (exit criteria met, BGM crossfade on maze section transitions)
- Phase 9: Chapter 6 (exit criteria met)
- Phase 10: Ending Screen (exit criteria met, BGM crossfade from constellation to ending)

---

## Goal

Extend AudioService with observable interruption state, pause duration tracking, and resume signaling to enable verified interruption resilience across all chapters.

---

## Scope

### File Inventory

| File                                       | Action | Responsibility                                                                                                                   |
| ------------------------------------------ | ------ | -------------------------------------------------------------------------------------------------------------------------------- |
| `OttieExpress/Services/AudioService.swift` | Modify | Add observable interruption state property, pause duration tracking with 5-second threshold, and resume indicator signaling logic |

### Integration Points

**AudioService.swift**

- Imports from: None (uses AVFoundation framework only)
- Imported by: Chapter 1 (`Chapter1View`), Chapter 3 (`TapePlayerView`, `FloatingPhraseView`, `VUMeterView`, `ReelAnimation`), Chapter 5 (`MazeScene`, `SectionTransition`), Chapter 6 (`Chapter6View`), Ending Screen (`ConstellationBackdrop`)
- State reads: None
- State writes: None (AudioService manages its own playback state internally; does not read or write AppState)

---

## Out of Scope

- Chapter 3 VU meter and reel animation freeze/unfreeze view logic (consuming views observe AudioService's `isInterrupted` property; view-side freeze implementation is owned by Phase 7 chapter views)
- Chapter 3 floating phrase timer UI freeze and resync calculations (FloatingPhraseView consumes AudioService's interruption state and current playback position to compute timer offsets; timer math is view-owned)
- Resume indicator UI rendering, positioning, and dismiss behavior (consuming chapter views read AudioService's `showResumeIndicator` signal and present their own styled indicator)
- Audio route change handling for headphone connect/disconnect events (separate from `AVAudioSession.interruptionNotification`; not specified in SS15)
- Audio session category or mode reconfiguration (`.playback` category configured in E1.3; this epic does not change session configuration)

---

## Definition of Done

- [ ] AudioService exposes an observable `isInterrupted` boolean property that transitions to `true` on interruption began and `false` on interruption ended
- [ ] All active `AVAudioPlayer` instances pause within the same run loop cycle as the interruption began notification
- [ ] All paused `AVAudioPlayer` instances resume from their exact paused position when interruption ended fires with `shouldResume` option
- [ ] AudioService records a timestamp on interruption began and computes elapsed pause duration on interruption ended
- [ ] AudioService exposes an observable `showResumeIndicator` boolean that transitions to `true` when computed pause duration exceeds 5 seconds
- [ ] `showResumeIndicator` resets to `false` after playback resumes
- [ ] An interruption that fires mid-crossfade during a BGM transition suspends both players and the volume ramp, and the crossfade resumes cleanly on interruption end without audible artifacts

---

## Implementation Notes

E1.3 established the base `AVAudioSession.interruptionNotification` observer and pause/resume mechanics for all active players. This epic extends that foundation with three capabilities: observable interruption state, pause duration tracking, and crossfade-under-interruption safety. Reference: E1.3 Implementation Notes, SS15 Audio Interruption Handling.

**Observable interruption state:** Add an `@Observable`-tracked `isInterrupted` property (Bool, default `false`). Set to `true` in the interruption began handler, `false` in the interruption ended handler. Chapter 3 views observe this property: `VUMeterView` freezes needle animation, `ReelAnimation` freezes reel rotation, and `FloatingPhraseView` freezes phrase drift timers. These views resume their animations when `isInterrupted` returns to `false`. The AudioService does not own view animation state; it provides the signal. Reference: SS15 Audio Interruption Handling, "freeze VU meter and reel animations (Chapter 3), freeze floating phrase timers."

**Pause duration tracking:** Store `Date()` when interruption begins. On interruption end, compute `Date().timeIntervalSince(interruptionStartDate)`. If the interval exceeds 5.0 seconds, set `showResumeIndicator` to `true` before calling resume on the players. Consuming views display a themed resume indicator (not a spinner, per SS3 Global Rules) and auto-dismiss after a brief hold. The AudioService sets `showResumeIndicator` back to `false` after resume completes. Reference: SS15 Audio Interruption Handling, "If paused for longer than 5 seconds: show a subtle Resume indicator before auto-resuming."

**Crossfade interruption safety:** If an interruption fires while a BGM crossfade is in progress (two active `AVAudioPlayer` instances with a volume ramp timer), the handler must: (1) pause both the outgoing and incoming players, (2) capture the current volume ramp progress, and (3) suspend the ramp timer. On resume, restart the ramp timer from the captured progress point and continue the crossfade. Validate this scenario against Chapter 5 section transitions where `sfx-maze-transition` and BGM crossfade trigger at maze boundaries, and against the ending screen where `bgm-constellation` crossfades to `bgm-ending`. Reference: SS15 Audio Management, "Crossfade on BGM transitions (0.8s)."
