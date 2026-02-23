# State Architecture Patterns — Reference Implementation

Concrete code patterns for Ottie Express state architecture. All patterns enforce the rules defined in `SKILL.md`.

---

## 1. ChapterID Definition

```swift
enum ChapterID: String, Codable, CaseIterable, Hashable {
    case chapterOne = "chapter_one"
    case chapterTwo = "chapter_two"
    case chapterThree = "chapter_three"
    case chapterFour = "chapter_four"
    case chapterFive = "chapter_five"
    case chapterSix = "chapter_six"

    var displayIndex: Int {
        switch self {
        case .chapterOne: 0
        case .chapterTwo: 1
        case .chapterThree: 2
        case .chapterFour: 3
        case .chapterFive: 4
        case .chapterSix: 5
        }
    }

    var next: ChapterID? {
        let all = ChapterID.allCases
        let idx = displayIndex
        return idx + 1 < all.count ? all[idx + 1] : nil
    }
}
```

---

## 2. AppState with Persistence

```swift
import Foundation
import Observation

@Observable
final class AppState {
    // MARK: - Persisted State

    var completedChapters: Set<ChapterID> = [] {
        didSet {
            guard completedChapters != oldValue else { return }
            persistCompletedChapters()
        }
    }

    var currentChapterID: ChapterID = .chapterOne {
        didSet {
            guard currentChapterID != oldValue else { return }
            UserDefaults.standard.set(
                currentChapterID.rawValue,
                forKey: PersistenceKey.currentChapterIndex
            )
        }
    }

    // MARK: - Derived State

    var isAllComplete: Bool {
        completedChapters.count == ChapterID.allCases.count
    }

    func isUnlocked(_ chapter: ChapterID) -> Bool {
        chapter == .chapterOne || completedChapters.contains(
            ChapterID.allCases[chapter.displayIndex - 1]
        )
    }

    // MARK: - Actions

    func markCompleted(_ chapter: ChapterID) {
        completedChapters.insert(chapter)
        if let next = chapter.next {
            currentChapterID = next
        }
    }

    // MARK: - Initialization (Restore from Disk)

    init() {
        self.completedChapters = Self.loadCompletedChapters()
        self.currentChapterID = Self.loadCurrentChapter()
    }

    // MARK: - Persistence (Private)

    private func persistCompletedChapters() {
        guard let data = try? JSONEncoder().encode(completedChapters) else { return }
        UserDefaults.standard.set(data, forKey: PersistenceKey.completedChapters)
    }

    private static func loadCompletedChapters() -> Set<ChapterID> {
        guard let data = UserDefaults.standard.data(
            forKey: PersistenceKey.completedChapters
        ),
            let decoded = try? JSONDecoder().decode(
                Set<ChapterID>.self, from: data
            )
        else { return [] }
        return decoded
    }

    private static func loadCurrentChapter() -> ChapterID {
        guard let raw = UserDefaults.standard.string(
            forKey: PersistenceKey.currentChapterIndex
        ),
            let chapter = ChapterID(rawValue: raw)
        else { return .chapterOne }
        return chapter
    }
}
```

---

## 3. Persistence Key Constants

```swift
enum PersistenceKey {
    static let completedChapters = "ottie.progress.completedChapters"
    static let currentChapterIndex = "ottie.progress.currentChapterIndex"

    // Per-chapter substep keys use a prefix + chapter raw value
    static func chapterSubstep(_ chapter: ChapterID) -> String {
        "ottie.chapter.\(chapter.rawValue).substep"
    }
}
```

---

## 4. Chapter State Container (Example: Chapter One)

```swift
import Observation

@Observable
final class ChapterOneState {
    var currentSubstep: Int = 0
    var userChoices: [String: String] = [:]

    @ObservationIgnored
    private let chapterID: ChapterID = .chapterOne

    var isComplete: Bool {
        currentSubstep >= totalSubsteps
    }

    private let totalSubsteps = 5

    // Optional: persist substep progress for long chapters
    var persistedSubstep: Int {
        get { currentSubstep }
        set {
            currentSubstep = newValue
            UserDefaults.standard.set(
                newValue,
                forKey: PersistenceKey.chapterSubstep(chapterID)
            )
        }
    }

    init() {
        self.currentSubstep = UserDefaults.standard.integer(
            forKey: PersistenceKey.chapterSubstep(.chapterOne)
        )
    }

    deinit {
        #if DEBUG
        print("[Lifecycle] ChapterOneState deallocated")
        #endif
    }
}
```

---

## 5. Chapter View with Isolated State

```swift
struct ChapterOneView: View {
    @State private var chapterState = ChapterOneState()
    @Environment(AppState.self) private var appState
    @Environment(\.completeChapter) private var completeChapter

    var body: some View {
        ChapterOneContent(state: chapterState) {
            completeChapter?(.chapterOne)
        }
    }
}

private struct ChapterOneContent: View {
    @Bindable var state: ChapterOneState
    let onFinish: () -> Void

    var body: some View {
        VStack {
            Text("Step \(state.currentSubstep + 1)")

            Button("Next") {
                state.persistedSubstep += 1
            }
            .disabled(state.isComplete)

            if state.isComplete {
                Button("Complete Chapter", action: onFinish)
            }
        }
    }
}
```

---

## 6. Environment Action — Complete Chapter

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

---

## 7. Router with Route Enum

```swift
import SwiftUI
import Observation

enum AppRoute: Hashable, Codable {
    case chapter(ChapterID)
    case chapterSelection
    case settings
}

@Observable
final class Router {
    var path = NavigationPath()

    func navigate(to route: AppRoute) {
        path.append(route)
    }

    func pop() {
        guard !path.isEmpty else { return }
        path.removeLast()
    }

    func popToRoot() {
        path = NavigationPath()
    }

    func advanceToNextChapter(from current: ChapterID) {
        if let next = current.next {
            popToRoot()
            path.append(AppRoute.chapter(next))
        } else {
            popToRoot()
        }
    }
}
```

---

## 8. App Entry Point — Wiring It Together

```swift
@main
struct OttieExpressApp: App {
    @State private var appState = AppState()
    @State private var router = Router()

    var body: some Scene {
        WindowGroup {
            NavigationStack(path: $router.path) {
                ChapterSelectionView()
                    .navigationDestination(for: AppRoute.self) { route in
                        destinationView(for: route)
                    }
            }
            .environment(appState)
            .environment(router)
            .environment(
                \.completeChapter,
                CompleteChapterAction { [weak appState, weak router] id in
                    appState?.markCompleted(id)
                    router?.advanceToNextChapter(from: id)
                }
            )
        }
    }

    @ViewBuilder
    private func destinationView(for route: AppRoute) -> some View {
        switch route {
        case .chapter(let id):
            chapterView(for: id)
        case .chapterSelection:
            ChapterSelectionView()
        case .settings:
            SettingsView()
        }
    }

    @ViewBuilder
    private func chapterView(for id: ChapterID) -> some View {
        switch id {
        case .chapterOne: ChapterOneView()
        case .chapterTwo: ChapterTwoView()
        case .chapterThree: ChapterThreeView()
        case .chapterFour: ChapterFourView()
        case .chapterFive: ChapterFiveView()
        case .chapterSix: ChapterSixView()
        }
    }
}
```

---

## 9. Navigation Path Persistence (Optional)

```swift
extension Router {
    private static let navPathKey = "ottie.navigation.path"

    func persistPath() {
        guard let representation = path.codable else { return }
        guard let data = try? JSONEncoder().encode(representation) else { return }
        UserDefaults.standard.set(data, forKey: Self.navPathKey)
    }

    func restorePath() {
        guard let data = UserDefaults.standard.data(forKey: Self.navPathKey),
              let representation = try? JSONDecoder().decode(
                  NavigationPath.CodableRepresentation.self, from: data
              )
        else { return }
        path = NavigationPath(representation)
    }
}
```

**Caution:** Route enum changes across app versions invalidate persisted paths. Wrap restoration in a `do`/`catch` and fall back to root on failure.

---

## 10. Anti-Patterns — What Not To Do

### God AppState

```swift
// WRONG: AppState owns chapter view models
@Observable
final class AppState {
    var chapterOneState = ChapterOneState()  // Violates isolation
    var chapterTwoState = ChapterTwoState()  // Never deallocates
}
```

### Chapter Navigating Itself

```swift
// WRONG: Chapter view controls navigation
struct ChapterOneView: View {
    @Environment(Router.self) private var router  // Prohibited

    var body: some View {
        Button("Done") {
            router.navigate(to: .chapter(.chapterTwo))  // Prohibited
        }
    }
}
```

### Raw Closure in Environment

```swift
// WRONG: Closure in environment causes unnecessary invalidations
extension EnvironmentValues {
    var onComplete: ((ChapterID) -> Void)? {  // Reference type, no equality
        get { self[OnCompleteKey.self] }
        set { self[OnCompleteKey.self] = newValue }
    }
}
```

### Shared Mutable Chapter State

```swift
// WRONG: Single class for all chapters
@Observable
final class ChapterState {
    let chapterID: ChapterID  // Parameterized — violates typed isolation
    var substep: Int = 0
}
```
