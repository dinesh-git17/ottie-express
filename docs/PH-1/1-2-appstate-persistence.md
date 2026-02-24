# E1.2: Implement AppState and Persistence

**Phase:** 1 - Core Systems and Design Foundation
**Class:** Infrastructure
**Design Doc Reference:** SS4 Architecture Overview (State Management, State Machine and Resume Semantics, Injection Pattern)
**Dependencies:**

- Phase 0: Project Scaffold (exit criteria met)
- E0.1: Scaffold Xcode project and directory structure (`App/` directory exists)

---

## Goal

Build the `@Observable` AppState class with chapter progress tracking, `UserDefaults` persistence, and restore-on-init semantics that survives app termination and handles data corruption gracefully.

---

## Scope

### File Inventory

| File                              | Action | Responsibility                                                                                                                                                                    |
| --------------------------------- | ------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `OttieExpress/App/AppState.swift` | Create | Define `@Observable` `@MainActor` AppState with `currentChapter`, `completedChapters`, `completeChapter(_:)`, `UserDefaults` persistence, and corruption-resilient initialization |

### Integration Points

**AppState.swift**

- Imports from: None (uses Foundation and Observation framework only)
- Imported by: None within Phase 1 (consumed by `OttieExpressApp.swift` in E2.1 via `@State private var appState = AppState()` and `.environment(appState)`)
- State reads: `UserDefaults.standard` on initialization (restores `completedChapters` and derives `currentChapter`)
- State writes: `UserDefaults.standard` on every `completeChapter(_:)` call (persists `completedChapters` set)

---

## Out of Scope

- Per-chapter sub-state persistence such as Chapter 2 per-poem completion flags (each chapter manages its own internal state persistence in its respective feature epic)
- `@Environment` injection into the SwiftUI view hierarchy (covered by E2.1 navigation shell epic which owns `@State private var appState`)
- Chapter gating logic that prevents navigation to locked chapters (covered by E2.1 `ChapterRouter` which reads `completedChapters`)
- Reset/debug functionality to clear `UserDefaults` state for testing (ad hoc developer tooling, not a shipped feature)
- `Codable` conformance or JSON-based serialization of AppState (SS4 specifies `UserDefaults` with `Set<Int>` directly, not a serialized model)

---

## Definition of Done

- [ ] `AppState` class compiles with `@Observable` macro and `@MainActor` annotation
- [ ] `AppState` is declared `final class` with `internal` access (not `public`, not `open`)
- [ ] `currentChapter` property reflects the next incomplete chapter (one past the highest completed)
- [ ] `completedChapters` property is a `Set<Int>` tracking all finished chapter indices
- [ ] `completeChapter(0)` inserts `0` into `completedChapters`, advances `currentChapter` to `1`, and writes to `UserDefaults`
- [ ] Calling `completeChapter` for chapters 0 through 2 sequentially, then initializing a new `AppState()`, produces `currentChapter == 3` and `completedChapters == Set([0, 1, 2])`
- [ ] Initializing `AppState()` when `UserDefaults` contains no persisted data sets `currentChapter` to `0` and `completedChapters` to empty set
- [ ] Initializing `AppState()` when `UserDefaults` contains corrupted or non-decodable data for the completion key resets to fresh state (`currentChapter == 0`, empty `completedChapters`)
- [ ] `completeChapter` called with an already-completed chapter index does not duplicate the entry or change `currentChapter`

---

## Implementation Notes

SS4 State Management provides the canonical class signature:

```swift
@MainActor
@Observable
final class AppState {
    var currentChapter: Int = 0
    var completedChapters: Set<Int> = []

    func completeChapter(_ chapter: Int) {
        completedChapters.insert(chapter)
        currentChapter = chapter + 1
        persist()
    }
}
```

The `currentChapter` derivation on init must compute from the persisted `completedChapters` set, not from a separately persisted integer. This avoids state divergence where `currentChapter` and `completedChapters` disagree. Derive `currentChapter` as: starting from 0, find the first index not in `completedChapters`. Reference: SS4 State Machine and Resume Semantics.

Persistence uses `UserDefaults.standard`. Store `completedChapters` as an `Array<Int>` (since `Set` is not directly `UserDefaults`-compatible) under a named constant key. On init, read the array back, convert to `Set<Int>`, and derive `currentChapter`. If the stored value is `nil` or fails to cast to `[Int]`, treat as fresh state. Reference: SS4 State Machine and Resume Semantics, corruption recovery rule.

The `persist()` method is `private` and called from `completeChapter(_:)`. It writes `Array(completedChapters)` to `UserDefaults`. No external caller should trigger persistence directly. Reference: SS4 State Management.

The total chapter count (7 chapters, indices 0-6) is defined in the design doc across SS6-SS12. `currentChapter` is clamped to this range. If all chapters are complete, `currentChapter` remains at 7 (one past the last index), which the navigation layer interprets as "show ending screen." Reference: SS4 State Machine, SS13 Ending Screen.
