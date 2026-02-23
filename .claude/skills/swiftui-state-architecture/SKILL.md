---
name: swiftui-state-architecture
description: Enforce production-grade SwiftUI state architecture for linear narrative iOS applications with chapter-based progression. Use when writing or reviewing @Observable model classes, AppState design, chapter state containers, progress persistence via UserDefaults, completion callback patterns, environment action injection, or navigation state separation. Triggers on SwiftUI state management, @Observable architecture, chapter progression logic, state persistence implementation, or any task producing state-owning Swift code in a multi-chapter narrative context.
---

# SwiftUI State Architecture for Linear Narratives

Governing standard for all SwiftUI state management code in Ottie Express. Every state model must have a single owner, every chapter must be isolated, every persistence write must be deterministic, and every navigation transition must be driven by data — not imperative view logic.

---

## Architecture Philosophy

A linear narrative app is a state machine. Chapters are nodes. Progression is a directed edge. The app's job is to track which node is active, what local state that node holds, and which edges have been traversed — surviving every app lifecycle event iOS can produce.

**Three invariants govern all state code:**

1. **Single ownership.** Every piece of mutable state has exactly one owner. State flows downward; events flow upward.
2. **Chapter isolation.** No chapter reads or writes another chapter's state. Cross-chapter communication flows exclusively through AppState or environment actions.
3. **Persistence determinism.** If state must survive app termination, the write path is explicit, immediate, and testable. No implicit syncing. No deferred flushes.

---

## 1. Observation Model (BLOCKING)

All model classes targeting iOS 17+ use the `@Observable` macro (Observation framework). The legacy `ObservableObject` / `@Published` protocol is prohibited in new code.

### Property Wrapper Selection

| Need                                    | Wrapper                   | Context                                 |
| --------------------------------------- | ------------------------- | --------------------------------------- |
| View owns the model's lifecycle         | `@State`                  | Root view creating the model            |
| View reads a model from environment     | `@Environment(Type.self)` | Global state consumption                |
| View needs bindings to a received model | `@Bindable`               | Forms, controls binding to passed model |
| View reads a model passed via init      | Plain `let`/`var`         | Read-only access, no wrapper            |

### Mandatory Rules

- All model classes use `@Observable`. No `ObservableObject`, `@Published`, `@StateObject`, `@ObservedObject`, or `@EnvironmentObject` in new code.
- `@Observable` class initializers must be trivial. No I/O, no subscriptions, no network calls, no expensive computation in `init()`.
- Properties that should not trigger view updates (caches, internal counters, bookkeeping) must use `@ObservationIgnored`.
- Nested observable hierarchies require `@Observable` at every level. Mixing observable and plain classes breaks observation silently.
- Computed properties depending on external sources (not stored properties) require manual `access()`/`withMutation()` calls.

### Prohibited

- `@StateObject`, `@ObservedObject`, `@EnvironmentObject`, `.environmentObject()` in any iOS 17+ code.
- Expensive work in `@Observable` initializers — `@State` re-invokes `init()` on view hierarchy rebuilds.
- Storing `@Observable` models in global singletons. Lifecycle must be tied to the view hierarchy.

---

## 2. Global AppState (BLOCKING)

A single `AppState` model holds cross-cutting concerns. It is created once in the `App` struct and injected via `.environment()`.

### AppState Scope

AppState contains **only**:

- Chapter progression tracking (which chapters are completed, current chapter index).
- App-level flags (onboarding complete, feature gates).
- Sync status indicators.

AppState **never** contains:

- Chapter-specific view models or local state.
- Navigation paths (owned by the Router).
- Ephemeral UI state (scroll position, form input, animation flags).

### Injection Pattern

```swift
@main
struct OttieExpressApp: App {
    @State private var appState = AppState()
    @State private var router = Router()

    var body: some Scene {
        WindowGroup {
            RootView()
                .environment(appState)
                .environment(router)
        }
    }
}
```

### Mandatory Rules

- AppState is `@Observable` and created with `@State` in the `App` struct. No other creation site.
- Views consume AppState via `@Environment(AppState.self)`. Never pass it through init parameters.
- AppState mutations occur through explicit methods, not direct property writes from views.
- AppState must not hold references to chapter-local state containers.

---

## 3. Chapter State Isolation (BLOCKING)

Each chapter owns an independent state container. Chapter state is created when the chapter view appears and deallocated when it disappears.

### Ownership Model

```
AppState (global, App-lifetime)
├── ChapterOneState (local, created by ChapterOneView)
├── ChapterTwoState (local, created by ChapterTwoView)
└── ...
```

Chapter state containers are `@Observable` classes held via `@State` in the chapter's root view:

```swift
struct ChapterOneView: View {
    @State private var chapterState = ChapterOneState()
    @Environment(AppState.self) private var appState

    var body: some View { /* ... */ }
}
```

### Mandatory Rules

- Each chapter's state is a distinct `@Observable` class. No shared base class with mutable stored properties.
- Chapter state is created via `@State` in the chapter's root view. Never in AppState, never in a parent coordinator.
- Chapter state must deallocate when navigation pops the chapter. Verify via `deinit` logging during development.
- No chapter state container may hold a reference to another chapter's state container.
- Shared read-only data (user profile, app config) flows through AppState or environment, never through cross-chapter references.

### Prohibited

- A single "ChapterState" class parameterized by chapter ID. Each chapter gets its own typed container.
- Storing chapter state in AppState or any long-lived collection.
- Chapter state containers referencing the Router or navigation path.

---

## 4. Persistence Strategy (BLOCKING)

Chapter progress must survive app termination — including force-quit. UserDefaults is the persistence layer for lightweight progress data.

See [references/state-architecture-patterns.md](references/state-architecture-patterns.md) for full code patterns.

### What to Persist

| Data                                 | Persist?                           | Mechanism                               |
| ------------------------------------ | ---------------------------------- | --------------------------------------- |
| Completed chapter set                | Yes                                | UserDefaults via Codable                |
| Current chapter index                | Yes                                | UserDefaults                            |
| Chapter-internal progress (substeps) | Yes, if non-trivial to reconstruct | UserDefaults via Codable                |
| Ephemeral animation/UI state         | No                                 | Recompute on launch                     |
| Navigation path                      | Optional                           | UserDefaults via Codable NavigationPath |

### Persistence Keys

All UserDefaults keys are named constants in a single enum namespace:

```swift
enum PersistenceKey {
    static let completedChapters = "ottie.progress.completedChapters"
    static let currentChapterIndex = "ottie.progress.currentChapterIndex"
    static let chapterSubstepPrefix = "ottie.chapter."
}
```

### Write Timing

Persist immediately on state change via `didSet`:

```swift
@Observable
final class AppState {
    var completedChapters: Set<ChapterID> = [] {
        didSet { persistCompletedChapters() }
    }
}
```

### Mandatory Rules

- All UserDefaults keys are defined as static constants. No string literals at call sites.
- Complex types (sets, structs, arrays) use `Codable` + `JSONEncoder`/`JSONDecoder`. No `RawRepresentable` hacks for `@AppStorage`.
- `@AppStorage` is permitted only for scalar preferences (Bool, Int, String). Progress data uses manual UserDefaults with `didSet` persistence.
- Persistence writes are synchronous and immediate. No debouncing, no batching, no deferred writes for critical progress data.
- `@SceneStorage` must not be the sole persistence mechanism for any data that must survive force-quit.
- Restoration logic runs in AppState's initializer, loading from UserDefaults with safe defaults for missing keys.

### Prohibited

- Persisting derived/computed state. Persist source-of-truth only; recompute derived values on load.
- Storing large blobs (images, audio data) in UserDefaults. Use file-system persistence for binary data.
- Redundant writes — guard `didSet` against no-op changes with an equality check.

---

## 5. Chapter Coordination (BLOCKING)

Chapters signal completion upward. The parent (navigation layer) decides what happens next. Chapters never navigate themselves.

### Completion Signaling via Environment Actions

Use defunctionalized environment actions — value-type structs with `callAsFunction`. Raw closures in the environment cause unnecessary view invalidations because SwiftUI cannot determine equality for reference types.

```swift
struct CompleteChapterAction {
    let perform: (ChapterID) -> Void
    func callAsFunction(_ id: ChapterID) { perform(id) }
}

private struct CompleteChapterActionKey: EnvironmentKey {
    static let defaultValue: CompleteChapterAction? = nil
}

extension EnvironmentValues {
    var completeChapter: CompleteChapterAction? {
        get { self[CompleteChapterActionKey.self] }
        set { self[CompleteChapterActionKey.self] = newValue }
    }
}
```

### Injection and Consumption

```swift
// Parent injects
ChapterOneView()
    .environment(\.completeChapter, CompleteChapterAction { id in
        appState.markCompleted(id)
        router.advanceToNextChapter()
    })

// Chapter consumes
@Environment(\.completeChapter) private var completeChapter
Button("Finish") { completeChapter?(.chapterOne) }
```

### Mandatory Rules

- Callbacks crossing more than one view level use environment actions, not closure init parameters.
- Environment action types are structs with `callAsFunction`. No raw closures stored in `EnvironmentValues`.
- Environment actions have a safe default (`nil` or no-op). Views must handle the absent case.
- Closures stored on `@Observable` classes use `[weak self]` capture lists.
- `Task` blocks inside `@Observable` models use `[weak self]` capture lists.
- Chapters call the completion action. Chapters never mutate AppState directly for progression.

### Prohibited

- Chapters importing or referencing the Router type.
- Chapters calling `NavigationPath.append()` or any navigation API.
- Direct AppState mutation from within chapter views for progression logic.
- Passing completion closures through more than one intermediate view via init parameters.

---

## 6. Navigation Integrity (BLOCKING)

Navigation state is separate from feature state. A Router owns the path. Chapter views own their content state.

### Route Definition

```swift
enum AppRoute: Hashable, Codable {
    case chapter(ChapterID)
    case chapterSelection
    case settings
}
```

### Router

```swift
@Observable
final class Router {
    var path = NavigationPath()

    func navigate(to route: AppRoute) { path.append(route) }
    func pop() { guard !path.isEmpty else { return }; path.removeLast() }
    func popToRoot() { path = NavigationPath() }

    func advanceToNextChapter() { /* deterministic progression logic */ }
}
```

### Mandatory Rules

- Routes are `Hashable`, `Codable` enum cases. No stringly-typed navigation.
- The Router is `@Observable`, created with `@State` in the `App` struct, injected via `.environment()`.
- Feature state (chapter view models) is created at destination view initialization, never eagerly by the Router.
- Deep links are handled via route state mutation on the Router, not imperative view construction.
- All types appended to `NavigationPath` conform to `Codable` if navigation persistence is required.

### Prohibited

- Navigation logic inside chapter views. Chapters signal completion; the Router decides the destination.
- Sharing a single `NavigationPath` across multiple `NavigationStack` instances.
- Storing feature view models in the Router.
- Using `NavigationLink` with a destination closure for programmatic navigation. Use value-based `NavigationLink` with `.navigationDestination(for:)`.

---

## 7. View Body Discipline

### Mandatory Rules

- View `body` is synchronous and free of side effects. No mutations, no I/O, no logging in `body`.
- Async work uses `.task` modifier, never `.onAppear { Task { } }`.
- No formatter instantiation, date computation, or string processing in `body`. Precompute or cache.
- `@State` in views holds only lightweight, view-specific data (toggle flags, text field bindings). Business state lives in `@Observable` models.

---

## 8. Verification Checklist

Before declaring state architecture work complete:

- [ ] All model classes use `@Observable`. No legacy observation types.
- [ ] AppState created in `App` struct, injected via `.environment()`.
- [ ] Each chapter has its own typed state container, owned via `@State` in its root view.
- [ ] No chapter references another chapter's state.
- [ ] Progress persists to UserDefaults via `didSet` with `Codable` encoding.
- [ ] All persistence keys are named constants.
- [ ] Completion signaling uses environment actions (struct with `callAsFunction`).
- [ ] No closures on `@Observable` classes without `[weak self]`.
- [ ] Routes are `Hashable`, `Codable` enums.
- [ ] Router is separate from feature state.
- [ ] No navigation logic in chapter views.
- [ ] `deinit` verified on chapter state containers during development.
