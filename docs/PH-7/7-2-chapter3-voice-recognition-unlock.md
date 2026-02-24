# E7.2: Implement Voice Recognition and Unlock

**Phase:** 7 - Chapter 3: The Cassette
**Class:** Feature
**Design Doc Reference:** SS9 (Voice Unlock interaction, ambient display, recognition target)
**Dependencies:**

- Phase 2: Navigation Shell and Shared UI (exit criteria met)
- E7.1: Build hint screen and permission flow (permissions granted before this epic activates)
- E1.1: Define design system tokens (color and typography tokens available)
- E1.3: Build AudioService (SFX playback for `sfx-tape-static`)
- E1.4: Build HapticService (CoreHaptics sustained pattern for recognition success)
- Asset: `cassette-tape` (available in asset catalog)
- Asset: `cassette-reel-l` (available in asset catalog)
- Asset: `cassette-reel-r` (available in asset catalog)
- Asset: `sfx-tape-static` (available in asset catalog)
- Service: PermissionHandler interface (public API available from E7.1)

---

## Goal

Implement `SFSpeechRecognizer` streaming recognition with on-device preference, phrase matching for "I love you baby", ambient cassette display with waveform visualizer, and the recognition success sequence with haptic and reel spin-up.

---

## Scope

### File Inventory

| File                                                   | Action | Responsibility                                                                                  |
| ------------------------------------------------------ | ------ | ----------------------------------------------------------------------------------------------- |
| `OttieExpress/Chapters/Chapter3/VoiceRecognizer.swift` | Create | `SFSpeechRecognizer` streaming recognition engine with phrase matching and lifecycle management |
| `OttieExpress/Chapters/Chapter3/VoiceUnlockView.swift` | Create | Ambient cassette display, waveform visualizer, and recognition success animation                |

### Integration Points

**VoiceRecognizer.swift**

- Imports from: `Speech`, `AVFoundation`, `Chapter3Constants.swift`
- Imported by: `VoiceUnlockView.swift`
- State reads: None
- State writes: None (exposes recognition state via `@Observable` properties)

**VoiceUnlockView.swift**

- Imports from: `VoiceRecognizer.swift`, `Chapter3Constants.swift`, `Color+DesignTokens.swift`, `Font+DesignTokens.swift`, `DesignConstants.swift`
- Imported by: `Chapter3View.swift` (E7.5)
- State reads: None
- State writes: None (reports recognition completion via callback)

---

## Out of Scope

- Microphone and speech recognition permission requests (handled by E7.1 `PermissionHandler`)
- Cassette engagement transition animation after recognition (covered by E7.5)
- Tape playback screen and VU meter rendering (covered by E7.3)
- Voice note audio playback (covered by E7.4)
- Audio interruption handling during recognition (deferred to Phase 11 cross-cutting hardening)
- Modifying AudioService session management API or adding new audio session category transitions (AudioService interface owned by E1.3)
- Waveform visualizer frequency-band decomposition or FFT analysis (simple audio level metering is sufficient per SS9)

---

## Definition of Done

- [ ] `VoiceRecognizer` configures `SFSpeechRecognizer` for English locale with `supportsOnDeviceRecognition` preference enabled
- [ ] `SFSpeechAudioBufferRecognitionRequest` streams audio buffers from `AVAudioEngine` input node tap
- [ ] Recognition result text is compared case-insensitively; a match occurs when the transcript contains "i love you"
- [ ] Ambient state renders cassette tape asset centered on screen with left and right reels animating at slow rotation speed
- [ ] `sfx-tape-static` plays softly via AudioService during ambient listening state
- [ ] Waveform visualizer renders beneath the cassette and reacts to live microphone audio level (RMS power from `AVAudioEngine` input tap)
- [ ] On phrase match: waveform visualization surges to peak amplitude animation
- [ ] On phrase match: HapticService fires CoreHaptics sustained pattern (0.8s, medium intensity)
- [ ] On phrase match: cassette reels accelerate to full rotation speed over 0.5s
- [ ] `sfx-tape-static` stops on phrase match
- [ ] `VoiceRecognizer` tears down `AVAudioEngine` input tap, recognition request, and recognition task on completion without resource leaks
- [ ] `VoiceRecognizer` tears down all resources when the view disappears before recognition completes

---

## Implementation Notes

Per design doc SS9, the recognition target is "I love you baby" but partial match "i love you" is sufficient. Perform case-insensitive substring containment check on `SFSpeechRecognitionResult.bestTranscription.formattedString`.

`VoiceRecognizer` must be `@MainActor` and `@Observable`. It owns the `AVAudioEngine` instance, installs an input tap on the engine's `inputNode` at a sample rate matching the device hardware, and appends buffers to the `SFSpeechAudioBufferRecognitionRequest`. The same input tap provides RMS power values for the waveform visualizer by computing average power across the buffer's float channel data.

Set `SFSpeechAudioBufferRecognitionRequest.shouldReportPartialResults = true` to enable real-time matching without waiting for silence-triggered finalization.

The `AVAudioEngine` must configure its audio session for `.record` category before starting. On recognition completion or teardown, restore the audio session category to `.playback` per SS15 audio management rules, enabling subsequent tape playback in E7.4.

Reel rotation uses a continuous `Animation.linear(duration:).repeatForever(autoreverses: false)` at slow speed during ambient state. On match, replace with a faster duration to achieve the spin-up effect. Per SS2, spring animations are preferred for transitions, but continuous rotation is an exception where linear is correct.

Per the SFSpeechRecognizer Mic Permission skill (`.claude/skills/sfspeechrecognizer-mic-permission/`), ensure the recognition teardown sequence follows: cancel recognition task, end audio engine, remove input tap, invalidate request. This ordering prevents audio graph crashes.
