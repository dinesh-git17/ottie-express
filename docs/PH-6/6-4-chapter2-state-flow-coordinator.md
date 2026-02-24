# E6.4: Wire Chapter 2 State and Flow Coordinator

**Phase:** 6 - Chapter 2: The Poems
**Class:** Feature
**Design Doc Reference:** §8 (Chapter 2 Flow, Chapter 2 Intro Card), §4 (State Machine and Resume Semantics, Chapter 2 persistence exception)
**Dependencies:**

- Phase 2: Navigation Shell and Shared UI (exit criteria met)
- E1.1: Define design system tokens (color tokens available)
- E1.2: Implement AppState and persistence (`completeChapter(_:)` interface and `UserDefaults` persistence available)
- E2.1: Build navigation shell (ChapterRouter routes to Chapter2View based on `appState.currentChapter`)
- E2.3: Build ChapterCard component (reusable intro card available)
- E6.1: Build fill-in poem screens (PoemView, PoemData, Chapter2Constants available)
- E6.2: Build poem collection reveal and her-turn input (PoemCollectionView, HerTurnView available)
- E6.3: Build API display screens (HaikuReplyView, PoemContinuationView available)

---

## Goal

Implement Chapter 2 per-poem completion persistence, the complete screen flow from intro card through API exchange, and chapter completion that writes to AppState.

---

## Scope

### File Inventory

| File                                                 | Action | Responsibility                                                                                                                                                                        |
| ---------------------------------------------------- | ------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `OttieExpress/Chapters/Chapter2/Chapter2State.swift` | Create | `@Observable` `@MainActor` state container tracking per-poem solved status with `UserDefaults` persistence, current flow step, and captured her-poem text                             |
| `OttieExpress/Chapters/Chapter2/Chapter2View.swift`  | Create | Root view for Chapter 2 that owns Chapter2State, sequences all subviews (intro card, poems, collection, her-turn, API stages), and calls `appState.completeChapter(2)` on chapter end |

### Integration Points

**Chapter2State.swift**

- Imports from: `PoemData.swift` (E6.1)
- Imported by: `Chapter2View.swift`
- State reads: `UserDefaults` for per-poem completion flags on initialization
- State writes: `UserDefaults` per-poem completion flags on each poem solve; does not write to AppState directly (Chapter2View handles chapter completion)

**Chapter2View.swift**

- Imports from: `Chapter2State.swift`, `Chapter2Constants.swift` (E6.1), `PoemView.swift` (E6.1), `PoemCollectionView.swift` (E6.2), `HerTurnView.swift` (E6.2), `HaikuReplyView.swift` (E6.3), `PoemContinuationView.swift` (E6.3), `ChapterCard` (E2.3)
- Imported by: `ChapterRouter.swift` (E2.1)
- State reads: `appState.currentChapter` (via environment), `Chapter2State.currentStep`, `Chapter2State.solvedPoems`
- State writes: `appState.completeChapter(2)` on chapter completion

---

## Out of Scope

- Fill-in poem UI, swipe interaction, and word validation logic (covered by E6.1)
- Poem collection reveal animation and her-turn text input UI (covered by E6.2)
- API display screens, typewriter animation, and fallback handling (covered by E6.3)
- AnthropicService network layer and DTO definitions (covered by E1.5)
- AppState structure or `completeChapter` method implementation (covered by E1.2)
- ChapterRouter routing logic or chapter transition animation (covered by E2.1)
- Screen transition animations between flow steps beyond standard SwiftUI transitions (deferred to Phase 12 polish)
- Analytics or telemetry for chapter completion events (not specified in design doc)

---

## Definition of Done

- [ ] Chapter 2 intro card renders with "Chapter 2" label, "The Poems" title, and "Some words are missing. You know what belongs there." body text using ChapterCard component
- [ ] Chapter 2 flow progresses in exact sequence: intro card, poem 1, poem 2, poem 3, collection reveal, her-turn, Stage 1 reply, Stage 2 continuation
- [ ] Chapter2State persists per-poem completion to `UserDefaults` immediately when each poem is solved
- [ ] On mid-chapter resume (app relaunch during Chapter 2), the flow restarts at the first unsolved poem, skipping previously solved poems
- [ ] Solved poems are not replayed on resume; the first unsolved poem is the entry point
- [ ] Her poem text captured from HerTurnView is stored in Chapter2State and passed to both HaikuReplyView and PoemContinuationView
- [ ] Chapter completion calls `appState.completeChapter(2)` after Stage 2 continuation finishes and the user taps "Continue"
- [ ] Chapter 2 completion persists to AppState and the user advances to Chapter 3 via ChapterRouter
- [ ] `UserDefaults` corruption or missing per-poem keys resets Chapter 2 to the first poem (no crash, no undefined state)
- [ ] Chapter2State is annotated `@Observable` and `@MainActor` per §4 concurrency requirements
- [ ] Chapter2View reads `appState` from the SwiftUI environment and does not create its own AppState instance

---

## Implementation Notes

Per §4 State Machine and Resume Semantics, Chapter 2 is the explicit exception to the rule that within-chapter sub-state is not persisted. The design doc states: "Exception: Chapter 2 persists per-poem completion state so solved poems are not replayed." This means `UserDefaults` stores a set of solved poem indices (e.g., `Set<Int>` keyed to `"ch2_solved_poems"`). On resume, Chapter2State reads this set and advances the flow to the first index not in the set.

Per §4, the persistence format is `UserDefaults` with `Set<Int>` for completed chapters and per-poem state for Chapter 2. The per-poem key must not collide with the global chapter completion key managed by AppState (E1.2). Use a distinct key namespace (e.g., `"ch2_solved_poems"`) separate from the `"completedChapters"` key.

The flow coordinator in Chapter2View uses Chapter2State.currentStep (an enum representing each flow stage) to determine which subview to display. Each subview reports completion via a callback closure. The coordinator advances the step on each callback. This is a linear, non-branching flow with no backward navigation.

The intro card uses the ChapterCard shared component from E2.3. The card configuration matches §8 Chapter 2 Intro Card: chapter label "Chapter 2", title "The Poems", body text "Some words are missing. You know what belongs there.", and Ottie idle asset.

Per §8 Flow, the complete sequence is: `Chapter 2 Intro Card -> Poem 1 -> Poem 2 -> Poem 3 -> Poem Collection Reveal -> Her Turn Screen -> Haiku Reply (Stage 1) -> Poem Continuation (Stage 2) -> Chapter 3`. The transition to Chapter 3 occurs when `appState.completeChapter(2)` is called, which advances `currentChapter` and triggers ChapterRouter to display the next chapter.
