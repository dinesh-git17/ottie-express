# E2.1: Build Navigation Shell and Chapter Routing

**Phase:** 2 - Navigation Shell and Shared UI
**Class:** Infrastructure
**Design Doc Reference:** SS4 Architecture Overview (State Management, Injection Pattern, State Machine and Resume Semantics), SS14 Navigation and Progression, SS2 Motion Principles
**Dependencies:**

- Phase 1: Core Systems and Design Foundation (exit criteria met)
- E1.1: Define design system tokens (spring parameters and transition durations available as named constants)
- E1.2: Implement AppState and persistence (`currentChapter`, `completedChapters`, and `completeChapter(_:)` available)

---

## Goal

Implement the chapter routing system that reads `AppState.currentChapter`, displays the corresponding chapter root view, gates locked chapters, and animates transitions with crossfade-drift and scale-pulse effects.

---

## Scope

### File Inventory

| File                                       | Action | Responsibility                                                                                                       |
| ------------------------------------------ | ------ | -------------------------------------------------------------------------------------------------------------------- |
| `OttieExpress/App/ChapterRouter.swift`     | Create | Route to the correct chapter view based on `AppState.currentChapter` with chapter-gating enforcement                 |
| `OttieExpress/App/ChapterTransition.swift` | Create | Define crossfade + upward drift transition and chapter-complete scale pulse transition using design token values     |
| `OttieExpress/App/OttieExpressApp.swift`   | Modify | Add `@State private var appState = AppState()` ownership and `.environment(appState)` injection, embed ChapterRouter |

### Integration Points

**ChapterRouter.swift**

- Imports from: `AppState.swift`, `ChapterTransition.swift`
- Imported by: `OttieExpressApp.swift`
- State reads: `appState.currentChapter`, `appState.completedChapters`
- State writes: None

**ChapterTransition.swift**

- Imports from: `DesignConstants.swift` (`Motion.springStiffness`, `Motion.springDamping`, `Motion.screenTransitionDuration`, `Motion.chapterCompleteDuration`)
- Imported by: `ChapterRouter.swift`
- State reads: None
- State writes: None

**OttieExpressApp.swift**

- Imports from: `AppState.swift`, `ChapterRouter.swift`
- Imported by: None (app entry point)
- State reads: None
- State writes: None (owns `@State private var appState` instance)

---

## Out of Scope

- Individual chapter view implementations beyond minimal placeholders for routing verification (covered by Phase 3-10 feature epics; ChapterRouter renders placeholder views until each chapter is built)
- Within-chapter navigation stacks or sub-screen routing (SS14 specifies each chapter manages its own internal navigation independently)
- Chapter select screen, back navigation between chapters, or chapter skipping (SS14 explicitly prohibits all three)
- `prefers-reduced-motion` accessibility adaptation for transition animations (deferred to Phase 11 cross-cutting hardening)
- Ending screen implementation beyond a placeholder view for routing verification (covered by Phase 10 ending screen epic)
- Transition animation tuning beyond the SS2 baseline spring and duration values (polish deferred to Phase 11)

---

## Definition of Done

- [ ] `OttieExpressApp` body contains `@State private var appState = AppState()` and injects it into the view hierarchy via `.environment(appState)`
- [ ] `ChapterRouter` reads `appState.currentChapter` via `@Environment(AppState.self)` and displays the corresponding chapter root view
- [ ] `ChapterRouter` displays the ending screen placeholder when `appState.currentChapter` equals 7 (one past the final chapter index)
- [ ] Standard chapter transitions animate with crossfade and upward drift over 0.4s using `Spring(stiffness: 280, damping: 22)`
- [ ] Chapter-complete transitions animate over 0.6s with a 1.0 to 1.02 to 1.0 scale pulse combined with crossfade
- [ ] A chapter view renders only when all prior chapter indices (0 through N-1) exist in `appState.completedChapters`
- [ ] When `appState.currentChapter` exceeds `completedChapters` (state corruption), ChapterRouter falls back to the earliest incomplete chapter
- [ ] `ChapterRouter` re-renders the correct chapter view within the same app session after `appState.completeChapter(_:)` advances `currentChapter`
- [ ] Project compiles with zero errors after modifications to `OttieExpressApp.swift` and addition of `ChapterRouter.swift` and `ChapterTransition.swift`

---

## Implementation Notes

SS14 specifies no global `NavigationStack` between chapters. `ChapterRouter` is a pure view-switching container that conditionally renders one chapter root view at a time based on `appState.currentChapter`. It is not a navigation controller. Each chapter view manages its own internal navigation. Reference: SS14 Within-Chapter Navigation.

The injection pattern follows SS4 exactly: `OttieExpressApp` owns `@State private var appState = AppState()` and applies `.environment(appState)`. `ChapterRouter` consumes via `@Environment(AppState.self) private var appState`. No `@Bindable` is needed in the router since it only reads state. Reference: SS4 State Management, Injection Pattern.

The crossfade + upward drift transition combines `.opacity` and `.offset(y:)` modifiers animated with `Spring(stiffness: DesignConstants.Motion.springStiffness, damping: DesignConstants.Motion.springDamping)`. The upward drift offset starts at approximately 12pt below final position and animates to zero. Use `.transition(.asymmetric(insertion: .opacity.combined(with: .offset(y: 12)), removal: .opacity))` or equivalent. Reference: SS2 Motion Principles ("crossfade with subtle upward drift").

The chapter-complete variant extends the standard transition with `.scaleEffect`. The pulse animates from 1.0 to 1.02 at the midpoint, then back to 1.0. Use `DesignConstants.Motion.chapterCompleteDuration` (0.6) as the total animation duration. Reference: SS2 Motion Principles ("0.6s with light scale pulse").

Chapter gating reads `appState.completedChapters` directly in `ChapterRouter`. A chapter at index N is accessible when `completedChapters.isSuperset(of: Set(0..<N))`. If `currentChapter` is ahead of what `completedChapters` permits (corruption), derive the correct chapter as the first index absent from `completedChapters` starting from 0. Reference: SS14 Chapter Progression, SS4 State Machine and Resume Semantics corruption recovery.

Until feature epics build real chapter views, `ChapterRouter` renders minimal placeholder views per chapter index. Each placeholder displays the chapter number and a "Complete" button calling `appState.completeChapter(N)` to enable end-to-end routing verification. These placeholders are replaced one-to-one as Phase 3-10 epics deliver chapter implementations.
