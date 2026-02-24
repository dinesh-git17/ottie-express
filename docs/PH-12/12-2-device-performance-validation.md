# E12.2: Validate performance and bundle on device

**Phase:** 12 - Final Validation
**Class:** Integration
**Design Doc Reference:** SS1 Product Overview (FAANG-level polish), SS4 Architecture Overview (Technology Stack, Build Tooling), SS5 Asset Inventory (bundle size estimate), Rollout Plan (developer testing checklist)
**Dependencies:**

- Phase 11: Cross-Cutting Hardening (exit criteria met)
- E12.1: Write cross-chapter integration tests (integration test suite passing)
- Phase 3: Chapter 0 (exit criteria met)
- Phase 4: Chapter 4 (exit criteria met)
- Phase 5: Chapter 1 (exit criteria met, SpriteKit runner game implemented)
- Phase 6: Chapter 2 (exit criteria met)
- Phase 7: Chapter 3 (exit criteria met)
- Phase 8: Chapter 5 (exit criteria met, SpriteKit maze implemented)
- Phase 9: Chapter 6 (exit criteria met, constellation drawing implemented)
- Phase 10: Ending Screen (exit criteria met)
- Asset: iPhone 17 Pro running iOS 18 (required for device profiling)
- Asset: All audio and visual assets in final form
- Asset: Voice note with calibrated floating phrase timings integrated

---

## Goal

Confirm 60fps rendering across all SpriteKit and SwiftUI scenes, verify memory stability during chapter transitions, validate bundle size compliance, and execute a crash-free full playthrough on iPhone 17 Pro.

---

## Scope

### File Inventory

| File                                       | Action | Responsibility                                                                                                                        |
| ------------------------------------------ | ------ | ------------------------------------------------------------------------------------------------------------------------------------- |
| `OttieExpressTests/PerformanceTests.swift` | Create | Define XCTest performance measurement tests for chapter transition timing, state persistence latency, and memory allocation baselines |

### Integration Points

**PerformanceTests.swift**

- Imports from: XCTest, `OttieExpress` module (`AppState`)
- Imported by: None (test target only)
- State reads: Creates test instances of `AppState` for performance measurement
- State writes: Calls `completeChapter(_:)` within `measure {}` blocks

---

## Out of Scope

- Fixing performance defects discovered during profiling (frame drops, memory leaks, or excessive allocations are filed as defects and resolved in dedicated fix branches before Phase 12 exit criteria can be met)
- Automated CI/CD performance regression testing infrastructure (single-device private app; no continuous performance monitoring pipeline)
- Network latency profiling for AnthropicService API calls (network performance is external and variable; API timeout hardening is covered by E11.3)
- Battery consumption or thermal throttling analysis (not specified in design doc exit criteria; single-session app usage pattern makes this low risk)
- Accessibility audit or VoiceOver compatibility testing (not specified in Phase 12 scope; single-user app with known accessibility preferences)

---

## Definition of Done

- [ ] Instruments Time Profiler trace on iPhone 17 Pro records zero sustained frame drops below 60fps during a complete Chapter 1 runner game session (start to 25-heart win condition)
- [ ] Instruments Time Profiler trace on iPhone 17 Pro records zero sustained frame drops below 60fps during Chapter 5 maze navigation across all three sections including section transition crossfades
- [ ] Instruments Time Profiler trace on iPhone 17 Pro records zero sustained frame drops below 60fps during Chapter 6 constellation drawing across all six constellations including the sky illuminate sequence
- [ ] Instruments Allocations trace records no leaked allocations (`Leaked Blocks` column reads 0) across a full sequential chapter transition from Chapter 0 completion through Ending Screen
- [ ] Archived app bundle (.ipa exported from Xcode) measures under 150MB total
- [ ] Full sequential playthrough on iPhone 17 Pro (Chapter 0, Chapter 1, Chapter 2, Chapter 3, Chapter 4, Chapter 5, Chapter 6, Ending Screen) completes without crash, hang, or unrecoverable state
- [ ] `protocol-zero.sh` exits 0 on the branch containing the final validated build
- [ ] `PerformanceTests.swift` compiles and all `measure {}` blocks execute on iPhone 17 Pro via Xcode Test without failure

---

## Implementation Notes

Phase 12 is the final gate before the Rollout Plan executes. The Rollout Plan (Developer Testing phase) specifies a full playthrough on physical iPhone 17 Pro verifying all chapters end-to-end, API integration, voice recognition, audio interruption handling, and state persistence. This epic formalizes those verification steps as binary pass/fail criteria. Reference: Rollout Plan Phase 1 (Developer Testing).

**Instruments profiling methodology:**

Run three separate Instruments traces on iPhone 17 Pro, one per high-rendering-load chapter:

1. **Chapter 1 (SpriteKit runner):** Profile with Time Profiler from game start through 25-heart collection. Monitor the Core Animation FPS instrument. A sustained drop (3+ consecutive frames below 60fps) is a failure. Isolated single-frame drops during scene transitions are acceptable.
2. **Chapter 5 (SpriteKit maze):** Profile from Section 1 entry through Section 3 reunion cutscene. The section transition (0.5s fade-to-black with tileset swap and BGM crossfade) is the highest-risk moment for frame drops. Monitor through all three transitions.
3. **Chapter 6 (SwiftUI constellation drawing):** Profile from first constellation dot appearance through the sky illuminate sequence. The simultaneous flash of all six constellations with camera zoom-out and particle effects is the peak GPU load.

Reference: SS4 Technology Stack (Instruments), SS1 Product Overview (FAANG-level polish implies 60fps target).

**Memory leak detection:** Run the full app lifecycle in Instruments with the Allocations instrument. Navigate Chapter 0 through Ending Screen. After reaching the ending screen, check the Allocations summary for `Leaked Blocks`. Any non-zero count indicates a retain cycle or resource leak introduced during chapter transitions. Common leak sources: unremoved `NotificationCenter` observers, uncancelled `Task` instances, and strong reference cycles in closures captured by animation timers. Reference: SS4 Concurrency Model (`.task {}` for automatic cancellation), E1.3 (AudioService observer lifecycle).

**Bundle size verification:** Archive the app via Xcode (Product > Archive) and export the .ipa for Ad Hoc or Development distribution. Inspect the .ipa file size. SS5 Asset Inventory estimates ~45MB total. The hard constraint from Phase 12 exit criteria is under 150MB. The 150MB threshold also satisfies the TestFlight 200MB cellular download limit with margin. Reference: SS5 Asset Inventory, Rollout Plan Phase 2 (TestFlight Internal Build, "Verify bundle size matches estimate (~53 MB)").

**Full playthrough protocol:** Execute a sequential playthrough on iPhone 17 Pro with no debugger attached (Release build via TestFlight or Archive > Install). Play every chapter from start to completion in order: Chapter 0 (fingerprint scan, password), Chapter 1 (runner game to 25 hearts, letter), Chapter 2 (three poems, her turn, API exchange), Chapter 3 (voice unlock, cassette playback), Chapter 4 (dossier files), Chapter 5 (three maze sections, reunion), Chapter 6 (six constellations, sky illuminate), Ending Screen. A crash, hang (unresponsive for >5 seconds), or unrecoverable state (stuck on a screen with no way to proceed) at any point is a failure. Reference: Rollout Plan Phase 1 (Developer Testing).

**Performance test file:** `PerformanceTests.swift` uses `XCTestCase` with `measure {}` blocks (Swift Testing does not support performance measurement). Include baseline measurements for `AppState.completeChapter` round-trip latency and `AppState` initialization-from-UserDefaults latency. These baselines establish reference values; the primary performance validation is the Instruments traces. Reference: SS4 Technology Stack ("XCTest for UI automation"), XCTest performance measurement API.

**Protocol Zero final gate:** `protocol-zero.sh` must exit 0 on the branch containing the final build. This is the last enforcement point before the Rollout Plan executes. Any attribution artifacts introduced during the full development lifecycle are caught here. Reference: CLAUDE.md SS2 Protocol Zero, Phase 12 exit criteria.
