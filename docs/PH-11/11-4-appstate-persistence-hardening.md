# E11.4: Harden AppState persistence and resume semantics

**Phase:** 11 - Cross-Cutting Hardening
**Class:** Integration
**Design Doc Reference:** SS4 Architecture Overview (State Machine and Resume Semantics)
**Dependencies:**

- Phase 1: Core Systems and Design Foundation (exit criteria met)
- E1.2: Implement AppState and Persistence (AppState created with UserDefaults persistence and corruption-resilient init)
- Phase 3: Chapter 0 (exit criteria met)
- Phase 4: Chapter 4 (exit criteria met)
- Phase 5: Chapter 1 (exit criteria met, game state reset on resume validated)
- Phase 6: Chapter 2 (exit criteria met, per-poem persistence implemented in Chapter2State)
- Phase 7: Chapter 3 (exit criteria met)
- Phase 8: Chapter 5 (exit criteria met)
- Phase 9: Chapter 6 (exit criteria met)
- Phase 10: Ending Screen (exit criteria met)

---

## Goal

Validate all SS4 resume semantics across every chapter, including mid-chapter restart, Chapter 2 per-poem persistence, Chapter 1 game state reset, and UserDefaults corruption recovery.

---

## Scope

### File Inventory

| File                              | Action | Responsibility                                                                                                                                                                          |
| --------------------------------- | ------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `OttieExpress/App/AppState.swift` | Modify | Add defensive guards for all UserDefaults reads covering type mismatch and partial corruption; validate resume edge cases for Chapter 1 game reset and Chapter 2 per-poem key coexistence |

### Integration Points

**AppState.swift**

- Imports from: None (uses Foundation and Observation framework only)
- Imported by: `OttieExpressApp.swift` (ownership via `@State`), `ChapterRouter` (reads `currentChapter` and `completedChapters` for routing), all chapter views (call `completeChapter(_:)` on chapter end), `EndingScreenView` (displayed when `currentChapter == 7`)
- State reads: `UserDefaults.standard` on initialization (restores `completedChapters` set, derives `currentChapter`)
- State writes: `UserDefaults.standard` on every `completeChapter(_:)` call (persists `completedChapters` as `Array<Int>`)

---

## Out of Scope

- Chapter 2 per-poem persistence implementation in `Chapter2State.swift` (implemented in E6.4; this epic validates AppState coexists with those keys without collision or corruption)
- Chapter 1 game state reset logic in Chapter 1 views (Chapter 1 views own hearts/lives state and reset on init; this epic validates AppState does not persist or interfere with that transient state)
- ChapterRouter navigation routing logic changes (ChapterRouter reads `currentChapter` and routes; this epic validates AppState provides correct values under all resume conditions)
- iCloud sync, Keychain storage, or cross-device state transfer (not specified in design doc; single-device deployment)
- UserDefaults schema migration between app versions (single version with no prior schema; no migration path required)
- Debug or developer-only state reset UI (ad hoc tooling, not a shipped feature)

---

## Definition of Done

- [ ] Force-quit during Chapter 0 (before completing fingerprint scan) and relaunch places the user at the beginning of Chapter 0 with `currentChapter == 0`
- [ ] Force-quit during Chapter 3 mid-flow (after completing Chapters 0-2) and relaunch places the user at the beginning of Chapter 3 with `completedChapters` containing 0, 1, 2
- [ ] Force-quit during Chapter 6 and relaunch places the user at the beginning of Chapter 6 with `completedChapters` containing 0 through 5
- [ ] Chapter 2 per-poem completion flags (managed by `Chapter2State` in separate UserDefaults keys) persist across force-quit; solved poems display as completed and are not replayed on resume
- [ ] Chapter 1 game state (hearts and lives) resets to initial values when Chapter 1 is re-entered after a force-quit mid-game
- [ ] Deleting all UserDefaults keys for the app's bundle identifier domain causes AppState to initialize with `currentChapter == 0` and empty `completedChapters` without crash
- [ ] Writing a `String` value to the `completedChapters` UserDefaults key (type mismatch) causes AppState to initialize with `currentChapter == 0` and empty `completedChapters` without crash
- [ ] Completing all chapters (0 through 6) and reinstalling via Xcode (simulating TestFlight update) preserves `completedChapters` and routes to the ending screen on relaunch

---

## Implementation Notes

E1.2 established the core AppState with `@Observable`, `@MainActor`, UserDefaults persistence of `completedChapters` as `Array<Int>`, and corruption-resilient initialization that falls back to fresh state on `nil` or non-decodable data. This epic hardens those semantics under real multi-chapter integration and additional corruption vectors. Reference: E1.2 Implementation Notes, SS4 State Machine and Resume Semantics.

**Resume semantics per SS4:**

- **Default rule:** Within-chapter sub-state is not persisted. On relaunch mid-chapter, the user resumes at the beginning of the current incomplete chapter. `currentChapter` is derived from `completedChapters` by finding the first index not in the set (starting from 0), so it always reflects the correct resume point.
- **Chapter 2 exception:** Per-poem completion persists to UserDefaults under keys managed by `Chapter2State` (e.g., `ch2_poem0_solved`, `ch2_poem1_solved`). AppState does not own or read these keys. Validate that `Chapter2State` keys and AppState's `completedChapters` key coexist in UserDefaults without namespace collision or mutual corruption.
- **Chapter 1 exception:** Game state (hearts and lives) resets on resume. This is enforced by Chapter 1 view initialization, not AppState. Validate that AppState does not persist any Chapter 1 transient game state. Hearts and lives are view-local and reset to initial values when Chapter 1's view initializes.

Reference: SS4 State Machine and Resume Semantics, all three rules.

**Corruption recovery hardening:** E1.2 handles `nil` and failed `[Int]` cast by resetting to fresh state. This epic validates additional corruption vectors:

- **Type mismatch:** A `String` or `Bool` stored under the `completedChapters` key. The `UserDefaults.array(forKey:) as? [Int]` cast returns `nil`, triggering fresh state. Verify no crash.
- **Partial corruption:** An array containing mixed types (e.g., `[0, "invalid", 2]`). The cast to `[Int]` fails, triggering fresh state. Verify no crash.
- **Domain deletion:** All keys for the app's bundle identifier removed via `UserDefaults.standard.removePersistentDomain(forName:)`. Init reads `nil`, produces fresh state. Verify no crash.

Reference: SS4, "UserDefaults corruption or reset: app treats state as fresh, user starts from Chapter 0."

**TestFlight update survival:** UserDefaults persists across app updates delivered via TestFlight or Xcode re-deployment. Simulate by running the app, completing chapters 0 through 6, then re-deploying from Xcode (Product > Run) without resetting the simulator or device. Verify `completedChapters` survives and the app routes to the ending screen. Reference: SS4, "TestFlight app update: state is preserved (UserDefaults survives app updates)."
