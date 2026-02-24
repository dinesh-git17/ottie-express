# E7.5: Build Final Line and Chapter Flow Coordinator

**Phase:** 7 - Chapter 3: The Cassette
**Class:** Feature
**Design Doc Reference:** SS9 (Vault Transition / Cassette Engagement, Final Line screen, full chapter flow)
**Dependencies:**

- Phase 2: Navigation Shell and Shared UI (exit criteria met)
- E2.1: Build navigation shell (ChapterRouter routes to Chapter3View)
- E2.2: Build TypewriterText component (typewriter animation at configurable interval)
- E1.1: Define design system tokens (color and typography tokens available)
- E1.3: Build AudioService (SFX playback for `sfx-cassette-click`)
- E1.4: Build HapticService (heavy impact for cassette click)
- E1.2: Implement AppState and persistence (`completeChapter(3)` available)
- E7.1: Build hint screen and permission flow (HintScreen and PermissionHandler available)
- E7.2: Implement voice recognition and unlock (VoiceUnlockView available)
- E7.3: Build tape playback UI (CassetteView, VUMeterView available)
- E7.4: Implement audio playback and floating phrase sync (TapePlayerView available)
- Asset: `sfx-cassette-click` (available in asset catalog)

---

## Goal

Implement the cassette engagement transition, the post-playback final line typewriter sequence, and the full Chapter 3 flow coordinator that orchestrates all screens from intro card through chapter completion.

---

## Scope

### File Inventory

| File                                                 | Action | Responsibility                                                                                                      |
| ---------------------------------------------------- | ------ | ------------------------------------------------------------------------------------------------------------------- |
| `OttieExpress/Chapters/Chapter3/FinalLineView.swift` | Create | Post-playback silence, final line typewriter at 60ms/character, and "Continue" prompt                               |
| `OttieExpress/Chapters/Chapter3/Chapter3View.swift`  | Create | Flow coordinator: intro card, hint, voice unlock, cassette engagement, tape playing, final line, chapter completion |

### Integration Points

**FinalLineView.swift**

- Imports from: `TypewriterText` (SharedUI), `Chapter3Constants.swift`, `Color+DesignTokens.swift`, `Font+DesignTokens.swift`
- Imported by: `Chapter3View.swift`
- State reads: None
- State writes: None (reports completion via callback)

**Chapter3View.swift**

- Imports from: `HintScreen.swift` (E7.1), `PermissionHandler.swift` (E7.1), `VoiceUnlockView.swift` (E7.2), `TapePlayerView.swift` (E7.4), `FinalLineView.swift`, `Chapter3Constants.swift`, `Color+DesignTokens.swift`, `DesignConstants.swift`
- Imported by: `ChapterRouter.swift` (E2.1)
- State reads: `AppState.currentChapter`, `AppState.completedChapters`
- State writes: `AppState.completeChapter(3)` on chapter completion

---

## Out of Scope

- VU meter and reel animation implementation (covered by E7.3)
- Voice note audio playback and floating phrase timing (covered by E7.4)
- Voice recognition engine and waveform visualizer (covered by E7.2)
- Permission request UI and denial redirect (covered by E7.1)
- Chapter 3 to Chapter 4 navigation transition animation (owned by ChapterRouter in E2.1)
- Audio interruption recovery during any Chapter 3 screen (deferred to Phase 11 cross-cutting hardening)
- Modifying ChapterRouter to support intra-chapter back navigation or deep linking (ChapterRouter owned by E2.1)
- Reduced motion accessibility variants for cassette engagement transition (deferred to Phase 11 hardening)

---

## Definition of Done

- [ ] `Chapter3View` presents screens in sequence: intro card, hint screen, voice unlock, cassette engagement, tape playing, final line
- [ ] Cassette engagement transition animates the cassette sliding downward over 0.8s using spring animation
- [ ] Screen background transitions from `night-base` to `constellation-blue` with amber undertones during cassette engagement
- [ ] `sfx-cassette-click` plays via AudioService at the moment the cassette engagement completes
- [ ] HapticService fires heavy impact haptic on cassette engagement completion
- [ ] After voice note playback ends: VU meter needles settle to zero, cassette reels stop spinning
- [ ] A 1.5-second silence holds after playback ends before the final line appears
- [ ] Final line typewriters at 60ms per character in New York Italic 21pt, `parchment`, horizontally centered: "If I had to choose again, in every lifetime, I would still find my way to you."
- [ ] Final line remains on screen after typewriter completes (no fade)
- [ ] "Continue" prompt appears 3 seconds after the final line completes, rendered in `muted-gray`
- [ ] Tapping "Continue" calls `appState.completeChapter(3)`, persisting completion to UserDefaults
- [ ] `Chapter3View` reads `AppState` via `@Environment(AppState.self)` and does not own state initialization
- [ ] Flow advances only forward; back navigation within the chapter sequence is not supported
- [ ] Audio session category transitions from `.record` to `.playback` before tape playback begins

---

## Implementation Notes

Per design doc SS9, the full Chapter 3 flow is: intro card, hint screen, voice unlock, cassette engagement, tape playing, final line. `Chapter3View` manages this as a linear state machine with an enum representing each phase. Transitions are unidirectional. Each child view reports completion via a callback closure; the coordinator advances the state enum to the next value.

The cassette engagement transition per SS9: the cassette animates sliding downward "as if inserting into a player." Implement with `.offset(y:)` animation from the cassette's current position to a lower position (define the offset magnitude as a named constant). The background color transition from `night-base` to `constellation-blue` with amber undertones uses `.animation(.easeInOut(duration: 0.8))` on a background color state change. "Amber undertones" means blending a warm tint; define the target color as a named constant in `Chapter3Constants` (a Color interpolation between `constellation-blue` and `amber-warm`).

The final line uses `TypewriterText` from E2.2 with `interval: 0.060` (60ms per character), overriding the default 40ms. Per SS9, the font is New York Italic 21pt, which differs from the default New York Regular 18pt. Pass the font as a parameter to `TypewriterText` if it accepts one, or apply `.font()` on the wrapping view.

The 1.5-second silence between playback end and final line appearance uses `Task.sleep(nanoseconds:)` inside the coordinator's state transition logic. The 3-second delay before the "Continue" prompt uses the same pattern.

Per SS9, the `sfx-cassette-click` plays "on insert" at the completion of the slide-down animation. Use `AudioService.playSFX()` triggered by a completion callback or `.onAnimationCompleted` equivalent. Per SS15 haptic map, cassette click fires `UIImpactFeedbackGenerator` heavy.

`Chapter3View` must handle the audio session category transition. Before the tape playback screen appears, switch `AVAudioSession.sharedInstance()` from `.record` (used during voice recognition in E7.2) to `.playback` category. This ensures `AVAudioPlayer` playback functions correctly.
