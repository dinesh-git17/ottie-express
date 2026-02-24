# E11.2: Calibrate HapticService pattern library

**Phase:** 11 - Cross-Cutting Hardening
**Class:** Integration
**Design Doc Reference:** SS15 Haptics and Sound Design (Haptic Map)
**Dependencies:**

- Phase 1: Core Systems and Design Foundation (exit criteria met)
- E1.4: Build HapticService (HapticService created with all SS15 patterns and CoreHaptics engine lifecycle)
- Phase 3: Chapter 0 (exit criteria met, fingerprint scan haptics integrated)
- Phase 4: Chapter 4 (exit criteria met, file slam haptics integrated)
- Phase 5: Chapter 1 (exit criteria met, heart collected/obstacle hit/game complete haptics integrated)
- Phase 6: Chapter 2 (exit criteria met, correct word haptics integrated)
- Phase 7: Chapter 3 (exit criteria met, voice recognition and cassette click haptics integrated)
- Phase 8: Chapter 5 (exit criteria met, wall bump haptics integrated)
- Phase 9: Chapter 6 (exit criteria met, constellation lock and sky illuminate haptics integrated)
- Phase 10: Ending Screen (exit criteria met)
- Asset: iPhone 17 Pro physical device (required for hardware haptic validation)

---

## Goal

Validate every haptic pattern in the SS15 haptic map against iPhone 17 Pro hardware and tune CoreHaptics timing parameters for perceptual quality on target device.

---

## Scope

### File Inventory

| File                                        | Action | Responsibility                                                                                                                                     |
| ------------------------------------------- | ------ | -------------------------------------------------------------------------------------------------------------------------------------------------- |
| `OttieExpress/Services/HapticService.swift` | Modify | Tune CoreHaptics event parameters (intensity curves, timing intervals, duration) based on device testing; verify engine reset and fallback behavior |

### Integration Points

**HapticService.swift**

- Imports from: None (uses CoreHaptics and UIKit frameworks only)
- Imported by: Chapter 0 (`FingerprintView`), Chapter 1 (`RunnerScene`), Chapter 2 (`PoemView`), Chapter 3 (`VoiceUnlockView`, `Chapter3View`), Chapter 4 (`DossierView`), Chapter 5 (`MazeOttieSprite`), Chapter 6 (`ConstellationDrawingView`, `SkyIlluminateView`), all chapter completion flows
- State reads: None
- State writes: None

---

## Out of Scope

- Adding haptic patterns not present in the SS15 haptic map (new patterns are feature scope, not hardening)
- Modifying chapter view code to change when or where haptics fire (interaction trigger points are owned by each chapter's feature epic)
- Haptic-audio sub-frame synchronization tuning (confirming both fire at the same trigger point is sufficient; perceptual latency below 16ms is beyond design doc specification)
- Accessibility reduced-haptics gating or intensity scaling preferences (not specified in the design doc; if added, a separate accessibility epic)
- AHAP file migration or file-based pattern loading (SS15 defines all patterns programmatically per E1.4)

---

## Definition of Done

- [ ] `fire(.fingerprintScan)` plays a CoreHaptics transient repeating pattern with intensity ramping from 0.2 to 1.0 over exactly 2 seconds on iPhone 17 Pro
- [ ] `fire(.fingerprintComplete)` fires `UIImpactFeedbackGenerator` at `.heavy` intensity on iPhone 17 Pro
- [ ] `fire(.wrongPassword)` fires `UIImpactFeedbackGenerator` at `.medium` intensity on iPhone 17 Pro
- [ ] `fire(.correctWord)` fires `UIImpactFeedbackGenerator` at `.medium` intensity on iPhone 17 Pro
- [ ] `fire(.heartCollected)` fires `UIImpactFeedbackGenerator` at `.light` intensity on iPhone 17 Pro
- [ ] `fire(.obstacleHit)` fires `UIImpactFeedbackGenerator` at `.medium` intensity on iPhone 17 Pro
- [ ] `fire(.gameComplete)` fires `UINotificationFeedbackGenerator` with `.success` type on iPhone 17 Pro
- [ ] `fire(.voicePhraseRecognized)` plays a CoreHaptics continuous event at intensity 0.5 for exactly 0.8 seconds on iPhone 17 Pro
- [ ] `fire(.cassetteClick)` fires `UIImpactFeedbackGenerator` at `.heavy` intensity on iPhone 17 Pro
- [ ] `fire(.fileSlam)` fires `UIImpactFeedbackGenerator` at `.heavy` intensity on iPhone 17 Pro
- [ ] `fire(.constellationLock)` fires `UIImpactFeedbackGenerator` at `.heavy` intensity on iPhone 17 Pro
- [ ] `fire(.skyIlluminate)` fires `UINotificationFeedbackGenerator` with `.success` type on iPhone 17 Pro
- [ ] `fire(.chapterComplete)` fires `UINotificationFeedbackGenerator` with `.success` type on iPhone 17 Pro
- [ ] All 13 patterns fire when the device ring/silent switch is set to silent
- [ ] `CHHapticEngine` restarts without crash or missed haptic events after the app returns from background to foreground

---

## Implementation Notes

E1.4 implemented all 13 SS15 haptic map entries with a typed `HapticPattern` enum and a single `fire(_:)` dispatch method. This epic validates each pattern on target hardware and tunes parameters where simulator behavior diverges from device perception. Reference: E1.4 Implementation Notes, SS15 Haptics and Sound Design Haptic Map table.

**Calibration methodology:** Run each chapter on iPhone 17 Pro and trigger every haptic interaction in context. For each interaction, verify: (1) the haptic type matches SS15 (impact vs. notification vs. CoreHaptics), (2) the perceived intensity matches the specified level (light, medium, heavy, or success), and (3) CoreHaptics patterns produce the intended temporal profile. Reference: SS15 Haptic Map, all 13 rows.

**Fingerprint scan tuning:** The transient repeating pattern must produce a perceptible progressive escalation from light taps to heavy taps over 2 seconds. E1.4 constructed this as a sequence of `CHHapticEvent` entries of type `.hapticTransient` with intensity ramping from 0.2 to 1.0. If the event count or inter-event spacing does not produce smooth escalation on iPhone 17 Pro hardware, adjust the event count (target 10-20 events over 2 seconds) and the intensity ramp curve. A linear ramp may feel front-loaded on device; an ease-in curve (quadratic or cubic) produces better perceived progression. Reference: SS15, "CoreHaptics transient pattern, repeating, increasing intensity, Low to High over 2s."

**Voice recognition sustained tuning:** The continuous event at 0.8 seconds must feel like a sustained hold, not a series of discrete pulses. Verify `CHHapticEvent` type is `.hapticContinuous` with intensity parameter 0.5 and duration 0.8. If the sustained feel is weak on device, increase sharpness parameter slightly (0.3 to 0.5 range). Reference: SS15, "CoreHaptics sustained, 0.8s Medium."

**Engine reset validation:** Background the app during a CoreHaptics pattern (fingerprint scan in Chapter 0 is the easiest trigger). Foreground the app. Verify the engine's `resetHandler` fires, the engine re-creates via the lazy initialization path from E1.4, and subsequent `fire(_:)` calls for CoreHaptics patterns succeed without crash or silent failure. Reference: E1.4 Implementation Notes, CoreHaptics engine lifecycle.

**Silent mode verification:** `UIFeedbackGenerator` and `CHHapticEngine` both fire regardless of the ring/silent switch position. This is standard iOS behavior but must be confirmed on iPhone 17 Pro. Toggle the switch to silent, trigger each pattern, verify tactile output. Reference: SS15, "If device is on silent, haptics remain active."
