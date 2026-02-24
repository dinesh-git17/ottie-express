# E1.4: Build HapticService

**Phase:** 1 - Core Systems and Design Foundation
**Class:** Infrastructure
**Design Doc Reference:** SS15 Haptics and Sound Design (Haptic Map), SS4 Architecture Overview (Dependencies, Technology Stack)
**Dependencies:**

- Phase 0: Project Scaffold (exit criteria met)
- E0.1: Scaffold Xcode project and directory structure (`Services/` directory exists)

---

## Goal

Implement the complete haptic pattern library from SS15 with UIFeedbackGenerator wrappers, CoreHaptics engine lifecycle management, and all declared patterns callable through a typed enum interface.

---

## Scope

### File Inventory

| File                                        | Action | Responsibility                                                                                                                                                                                                                                                              |
| ------------------------------------------- | ------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `OttieExpress/Services/HapticService.swift` | Create | Define `@MainActor` HapticService with typed haptic pattern enum, UIFeedbackGenerator wrappers for light/medium/heavy impact and success notification, CoreHaptics engine lifecycle, and three CoreHaptics patterns (transient progressive, sustained medium, sharp impact) |

### Integration Points

**HapticService.swift**

- Imports from: None within Phase 1 (uses CoreHaptics and UIKit frameworks only)
- Imported by: None within Phase 1 (consumed by `HapticButton` in E2.4 and chapter views in Phase 3+)
- State reads: None
- State writes: None

---

## Out of Scope

- Haptic pattern timing synchronization with animations or audio playback (coordination logic lives in each chapter's feature epic that triggers haptics alongside other effects)
- `UIFeedbackGenerator.prepare()` pre-warming strategy based on view lifecycle (consuming views call `prepare()` as needed; this epic provides the fire methods)
- Accessibility `prefers-reduced-motion` gating of haptic intensity (haptics and motion are independent accessibility dimensions; haptic gating, if required, is a cross-cutting hardening concern in Phase 11)
- Custom AHAP file loading or file-based haptic pattern definitions (SS15 defines patterns programmatically, not via AHAP files)
- Haptic pattern design iteration or tuning beyond the SS15 specification (pattern parameters match SS15 exactly; tuning is a polish activity in later phases)

---

## Definition of Done

- [ ] `HapticService` exposes a `HapticPattern` enum with cases for every SS15 haptic map entry: `fingerprintScan`, `fingerprintComplete`, `wrongPassword`, `correctWord`, `heartCollected`, `obstacleHit`, `gameComplete`, `voicePhraseRecognized`, `cassetteClick`, `fileSlam`, `constellationLock`, `skyIlluminate`, `chapterComplete`
- [ ] `fire(.heartCollected)` triggers `UIImpactFeedbackGenerator` at `.light` intensity
- [ ] `fire(.wrongPassword)`, `fire(.correctWord)`, and `fire(.obstacleHit)` trigger `UIImpactFeedbackGenerator` at `.medium` intensity
- [ ] `fire(.fingerprintComplete)`, `fire(.cassetteClick)`, `fire(.fileSlam)`, and `fire(.constellationLock)` trigger `UIImpactFeedbackGenerator` at `.heavy` intensity
- [ ] `fire(.gameComplete)`, `fire(.skyIlluminate)`, and `fire(.chapterComplete)` trigger `UINotificationFeedbackGenerator` with `.success` type
- [ ] `fire(.fingerprintScan)` plays a CoreHaptics transient repeating pattern with intensity increasing from low to high over 2 seconds
- [ ] `fire(.voicePhraseRecognized)` plays a CoreHaptics sustained pattern at medium intensity for 0.8 seconds
- [ ] `CHHapticEngine` initializes on first CoreHaptics pattern request and handles the engine reset callback by re-creating the engine
- [ ] `CHHapticEngine` stops when no CoreHaptics patterns are active, releasing system resources
- [ ] If CoreHaptics is unavailable on the device, CoreHaptics patterns degrade to the closest `UIImpactFeedbackGenerator` equivalent without crashing

---

## Implementation Notes

The SS15 Haptic Map defines 13 interaction-to-haptic mappings across three haptic technologies: `UIImpactFeedbackGenerator` (light, medium, heavy), `UINotificationFeedbackGenerator` (success), and `CoreHaptics` (transient progressive, sustained, sharp impact). Reference: SS15 Haptics and Sound Design, Haptic Map table.

The typed enum `HapticPattern` maps each SS15 interaction to its implementation. The `fire(_:)` method is the single public API. Internally, it switches on the pattern to dispatch to the correct generator or CoreHaptics player. This eliminates stringly-typed haptic calls per CLAUDE.md SS3.3. Reference: SS15 Haptic Map, CLAUDE.md SS3 Forbidden Code Artifacts.

CoreHaptics engine lifecycle: Create `CHHapticEngine` lazily on first CoreHaptics pattern invocation. Register the `resetHandler` to re-create the engine on system reset. Register the `stoppedHandler` to nil out the engine reference so it re-initializes on next use. Call `engine.start()` before each pattern player creation. Call `engine.stop(completionHandler:)` after the pattern player finishes. Reference: SS4 Dependencies, CoreHaptics fallback rule.

The fingerprint scan pattern (`fingerprintScan`) constructs a `CHHapticPattern` with a sequence of `CHHapticEvent` entries of type `.hapticTransient`, spaced at intervals over 2 seconds with `CHHapticEventParameter` intensity values ramping from 0.2 to 1.0. The exact event count and spacing should produce a perceptible progressive escalation over the 2-second duration. Reference: SS15 Haptic Map, "CoreHaptics transient pattern, repeating, increasing intensity, Low to High over 2s."

The voice phrase recognized pattern (`voicePhraseRecognized`) constructs a single `CHHapticEvent` of type `.hapticContinuous` with duration 0.8 seconds and intensity parameter at 0.5 (medium). Reference: SS15 Haptic Map, "CoreHaptics sustained, 0.8s Medium."

CoreHaptics fallback per SS4 Dependencies: "Degrade to `UIImpactFeedbackGenerator`." If `CHHapticEngine.capabilitiesForHardware().supportsHaptics` returns false, the three CoreHaptics patterns fall back to `UIImpactFeedbackGenerator` at `.heavy` (fingerprintScan, fingerprintComplete) or `.medium` (voicePhraseRecognized). Reference: SS4 Dependencies table, CoreHaptics row.

Annotate HapticService as `@MainActor` since UIKit feedback generators must be created and triggered on the main thread. Reference: SS4 Architecture Overview, concurrency model.
