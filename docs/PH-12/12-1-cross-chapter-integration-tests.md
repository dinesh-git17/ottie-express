# E12.1: Write cross-chapter integration tests

**Phase:** 12 - Final Validation
**Class:** Integration
**Design Doc Reference:** SS4 Architecture Overview (State Machine and Resume Semantics, Concurrency Model, Technology Stack), SS16 API Integration (Error Handling), SS1 Product Overview (linear chapter progression)
**Dependencies:**

- Phase 11: Cross-Cutting Hardening (exit criteria met)
- E1.2: Implement AppState and Persistence (AppState created with UserDefaults persistence and corruption recovery)
- E1.3: Build AudioService (AudioService created with interruption handling)
- E1.5: Build AnthropicService (AnthropicService created with typed errors and timeout)
- E2.1: Navigation shell (ChapterRouter created with routing logic for all chapter values)
- E11.1: Harden AudioService session lifecycle (observable interruption state exposed)
- E11.3: Harden AnthropicService resilience (fallback constant defined, error paths validated)
- E11.4: Harden AppState persistence and resume semantics (corruption recovery hardened)

---

## Goal

Implement integration tests covering cross-chapter state transitions, AppState persistence round-trips, AnthropicService fallback behavior, AudioService interruption state, and navigation routing correctness.

---

## Scope

### File Inventory

| File                                       | Action | Responsibility                                                                                                                                                                                         |
| ------------------------------------------ | ------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `OttieExpressTests/IntegrationTests.swift` | Create | Define Swift Testing suite with tests for chapter state transitions, persistence round-trips, AnthropicService error-to-fallback paths, AudioService interruption signaling, and ChapterRouter routing |

### Integration Points

**IntegrationTests.swift**

- Imports from: `OttieExpress` module (`AppState`, `AnthropicService`, `AudioService`, ChapterRouter routing types)
- Imported by: None (test target only)
- State reads: Creates test instances of `AppState`, reads `currentChapter` and `completedChapters` after operations
- State writes: Calls `completeChapter(_:)` on test `AppState` instances, writes to `UserDefaults` in test scope

---

## Out of Scope

- Chapter-internal unit tests for individual chapter views, game logic, or animation correctness (each chapter's feature epic owns its own tests; this epic tests cross-chapter integration only)
- XCUITest UI automation for end-to-end user flows (UI-driven testing is covered by the manual playthrough in E12.2, not by programmatic tests)
- Performance measurement or profiling tests (covered by E12.2 with Instruments and XCTest measure blocks)
- HapticService integration tests (haptic output is perceptual and cannot be verified programmatically; validated on device in E11.2)
- AudioService audio output verification (audio playback requires hardware; this epic tests the interruption state machine, not the audio output)

---

## Definition of Done

- [ ] Test suite compiles and all tests pass via `swift test` or `xcodebuild test` with zero failures
- [ ] A test verifies that calling `completeChapter(0)` on a fresh `AppState` sets `currentChapter` to 1 and `completedChapters` contains 0
- [ ] A test verifies that completing chapters 0 through 6 sequentially on a single `AppState` instance produces `currentChapter == 7` and `completedChapters == Set(0...6)`
- [ ] A test verifies that after persisting `AppState` to `UserDefaults`, initializing a new `AppState()` instance produces identical `currentChapter` and `completedChapters` values
- [ ] A test verifies that `AppState` initialized after clearing all `UserDefaults` keys produces `currentChapter == 0` and empty `completedChapters`
- [ ] A test verifies that `AnthropicService.fallbackMessage` equals the exact string defined in SS16: "bebe, your words were so beautiful I'm still processing them. 🥰"
- [ ] A test verifies that `AnthropicService` initialized without the `ANTHROPIC_API_KEY` environment variable throws `AnthropicService.Error.missingAPIKey` when `generateReply(to:)` is called
- [ ] A test verifies that posting `AVAudioSession.interruptionNotification` with type `.began` transitions `AudioService.isInterrupted` to `true`
- [ ] A test verifies that the ChapterRouter routing logic maps each `currentChapter` value (0 through 6) to its corresponding chapter view and maps value 7 to the ending screen
- [ ] No test makes a real HTTP request to the Anthropic API or any external endpoint

---

## Implementation Notes

SS4 Technology Stack specifies Swift Testing (`@Test`, `#expect`, `@Suite`) for unit tests and `XCTest` for UI automation. This integration test suite uses Swift Testing syntax since the tests are programmatic assertions, not UI-driven. Reference: SS4 Technology Stack.

**Test isolation:** Each test that interacts with `UserDefaults` must use a unique suite prefix or clear the relevant keys in a setup/teardown block to prevent inter-test contamination. `AppState` reads from `UserDefaults.standard`; tests must reset the persisted state before each test that verifies init-from-persistence behavior. Use `UserDefaults.standard.removeObject(forKey:)` for the `completedChapters` key. Reference: E1.2 Implementation Notes, UserDefaults key management.

**AppState test strategy:** Instantiate `AppState` directly. Call `completeChapter(_:)` and assert `currentChapter` and `completedChapters`. For persistence round-trip tests: call `completeChapter`, create a new `AppState()`, and verify values match. For corruption tests: write invalid data to the UserDefaults key, create `AppState()`, verify fresh state. All assertions use `#expect` (Swift Testing). Reference: E1.2 DoD, E11.4 DoD.

**AnthropicService test strategy:** The service loads its API key from `ProcessInfo.processInfo.environment["ANTHROPIC_API_KEY"]`. In the test environment, this variable is unset by default. Calling `generateReply(to:)` without the key throws `Error.missingAPIKey`. Assert this error case directly. For the fallback constant, assert `AnthropicService.fallbackMessage` string equality. Testing the actual timeout-to-fallback network path without real HTTP calls requires a `URLProtocol` subclass registered via `URLProtocol.registerClass(_:)` that intercepts requests to the Anthropic endpoint and simulates a timeout error. Reference: E1.5 DoD, E11.3 Implementation Notes.

**AudioService interruption test strategy:** `AVAudioSession.interruptionNotification` is a standard `NotificationCenter` notification. Post it manually in the test with the correct `userInfo` dictionary containing `AVAudioSession.InterruptionType.began.rawValue` under the `AVAudioSessionInterruptionTypeKey`. Assert `AudioService.isInterrupted` transitions to `true`. Repeat with `.ended` type and `AVAudioSession.InterruptionOptions.shouldResume` to verify the reverse transition. AudioService is `@MainActor`; use `@MainActor` on the test function or `await MainActor.run {}`. Reference: E1.3 DoD, E11.1 DoD.

**ChapterRouter test strategy:** If ChapterRouter is a SwiftUI view with internal routing logic, extract the routing decision into a testable function or verify the body's structure using the `@ViewBuilder` output type. Alternatively, if routing is a switch on `currentChapter` that returns different view types, create an `AppState` with each `currentChapter` value (0-7) and verify the routing function returns the expected identifier or view type. The exact approach depends on ChapterRouter's implementation from E2.1. Reference: E2.1, SS4 State Machine.

**Test execution:** The PHASES.md acceptance criteria specifies `swift test`. For an Xcode-managed iOS target, `xcodebuild test -scheme OttieExpress -destination 'platform=iOS Simulator,name=iPhone 17 Pro'` is the equivalent invocation. The DoD accepts either. Reference: SS4 Technology Stack, Build Tooling.
