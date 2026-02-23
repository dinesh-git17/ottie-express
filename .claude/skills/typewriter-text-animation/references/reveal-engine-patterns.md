# Reveal Engine Production Patterns

## 1. Core Reveal Loop

```swift
struct TypewriterText: View {
    let sourceText: String
    let timing: RevealTimingConfiguration
    let onRevealComplete: () -> Void

    @State private var revealCount: Int = 0
    @State private var isRevealing: Bool = false
    @State private var revealID: UUID = UUID()

    private var indices: [String.Index] { Array(sourceText.indices) }
    private var characters: [Character] { Array(sourceText) }

    var body: some View {
        ScrollViewReader { proxy in
            ScrollView {
                VStack(alignment: .leading, spacing: 0) {
                    Text(sourceText.prefix(upTo: revealIndex))
                        .fixedSize(horizontal: false, vertical: true)

                    Color.clear
                        .frame(height: 1)
                        .id("revealAnchor")
                }
            }
            .defaultScrollAnchor(.bottom)
            .onChange(of: revealCount) {
                // Scroll only when content height changes materially
                withAnimation(.easeOut(duration: 0.15)) {
                    proxy.scrollTo("revealAnchor", anchor: .bottom)
                }
            }
        }
        .overlay(alignment: .bottomTrailing) {
            CursorView(isActive: isRevealing)
        }
        .onTapGesture { skipReveal() }
        .task(id: revealID) { await runRevealLoop() }
    }

    private var revealIndex: String.Index {
        guard revealCount > 0, revealCount <= indices.count else {
            return sourceText.startIndex
        }
        return indices[revealCount - 1]
    }

    private func runRevealLoop() async {
        isRevealing = true
        let clock = ContinuousClock()
        var deadline = clock.now
        var i = 0

        while i < characters.count {
            guard !Task.isCancelled else {
                revealCount = indices.count
                isRevealing = false
                return
            }

            let delay = timing.delay(for: characters[i], next: i + 1 < characters.count ? characters[i + 1] : nil)
            let advance = timing.advanceCount(for: characters[i], next: i + 1 < characters.count ? characters[i + 1] : nil)

            i += advance
            revealCount = min(i, indices.count)
            deadline += delay

            do {
                try await clock.sleep(until: deadline)
            } catch {
                // CancellationError — exit cleanly
                revealCount = indices.count
                isRevealing = false
                return
            }
        }

        isRevealing = false
        onRevealComplete()
    }

    private func skipReveal() {
        guard isRevealing else { return }
        revealCount = indices.count
        isRevealing = false
        revealID = UUID()  // Cancels the .task(id:) reveal loop
        onRevealComplete()
    }
}
```

Key structural decisions:

- `revealID` as `.task(id:)` parameter enables cancellation by identity change.
- `skipReveal` is idempotent — `guard isRevealing` prevents double-fire.
- `revealIndex` computed property eliminates stored derived state.
- `onRevealComplete` fires from exactly two code paths: natural loop end and skip. The cancellation path (view disappear) does not fire the callback.

## 2. Cursor View (PhaseAnimator)

```swift
struct CursorView: View {
    let isActive: Bool

    var body: some View {
        Rectangle()
            .fill(Color.primary)
            .frame(width: 2, height: 20)
            .phaseAnimator([true, false]) { content, phase in
                content.opacity(isActive ? (phase ? 1 : 0) : 0)
            } animation: { _ in
                .easeInOut(duration: 0.53)
            }
    }
}
```

The cursor is a sibling view, not embedded in the text. `PhaseAnimator` manages the blink lifecycle. When `isActive` becomes `false`, the cursor fades to zero opacity on the next phase transition without residual animation artifacts.

## 3. Timing Configuration

```swift
struct RevealTimingConfiguration: Sendable {
    let baseLetterDelay: Duration
    let spaceDelay: Duration
    let commaDelay: Duration
    let periodDelay: Duration
    let exclamationDelay: Duration
    let questionDelay: Duration
    let semicolonDelay: Duration
    let colonDelay: Duration
    let ellipsisDelay: Duration
    let emDashDelay: Duration
    let newlineDelay: Duration
    let paragraphBreakDelay: Duration
    let openQuoteDelay: Duration
    let closeQuoteDelay: Duration
    let jitterEnabled: Bool
    let jitterRange: ClosedRange<Double>

    func delay(for character: Character, next: Character?) -> Duration {
        // Multi-character token detection
        if character == "\n", next == "\n" {
            return applyJitter(paragraphBreakDelay)
        }
        if character == ".", next == "." {
            return applyJitter(ellipsisDelay)
        }

        // Single character classification
        switch character {
        case "a"..."z", "A"..."Z": return applyJitter(baseLetterDelay)
        case "0"..."9":            return applyJitter(baseLetterDelay)
        case " ":                  return applyJitter(spaceDelay)
        case ",":                  return applyJitter(commaDelay)
        case ".":                  return applyJitter(periodDelay)
        case "!":                  return applyJitter(exclamationDelay)
        case "?":                  return applyJitter(questionDelay)
        case ";":                  return applyJitter(semicolonDelay)
        case ":":                  return applyJitter(colonDelay)
        case "\u{2026}":           return applyJitter(ellipsisDelay)  // Unicode ellipsis
        case "\u{2014}":           return applyJitter(emDashDelay)    // Em dash
        case "\n":                 return applyJitter(newlineDelay)
        case "\"", "\u{201C}":     return applyJitter(openQuoteDelay)
        case "\"", "\u{201D}":     return applyJitter(closeQuoteDelay)
        case "'", "\u{2018}":      return applyJitter(openQuoteDelay)
        case "'", "\u{2019}":      return applyJitter(closeQuoteDelay)
        default:                   return applyJitter(baseLetterDelay)
        }
    }

    func advanceCount(for character: Character, next: Character?) -> Int {
        if character == "\n", next == "\n" { return 2 }
        if character == ".", next == "." { return 3 }  // Three-dot ellipsis
        return 1
    }

    private func applyJitter(_ base: Duration) -> Duration {
        guard jitterEnabled else { return base }
        let factor = Double.random(in: jitterRange)
        let ms = Double(base.components.seconds) * 1000.0
            + Double(base.components.attoseconds) / 1_000_000_000_000_000.0
        return .milliseconds(ms * factor)
    }
}

extension RevealTimingConfiguration {
    static let standard = RevealTimingConfiguration(
        baseLetterDelay: .milliseconds(40),
        spaceDelay: .milliseconds(25),
        commaDelay: .milliseconds(150),
        periodDelay: .milliseconds(280),
        exclamationDelay: .milliseconds(280),
        questionDelay: .milliseconds(280),
        semicolonDelay: .milliseconds(170),
        colonDelay: .milliseconds(150),
        ellipsisDelay: .milliseconds(400),
        emDashDelay: .milliseconds(200),
        newlineDelay: .milliseconds(350),
        paragraphBreakDelay: .milliseconds(550),
        openQuoteDelay: .milliseconds(80),
        closeQuoteDelay: .milliseconds(100),
        jitterEnabled: false,
        jitterRange: 0.85...1.15
    )

    static let fast = RevealTimingConfiguration(
        baseLetterDelay: .milliseconds(15),
        spaceDelay: .milliseconds(10),
        commaDelay: .milliseconds(60),
        periodDelay: .milliseconds(100),
        exclamationDelay: .milliseconds(100),
        questionDelay: .milliseconds(100),
        semicolonDelay: .milliseconds(70),
        colonDelay: .milliseconds(60),
        ellipsisDelay: .milliseconds(150),
        emDashDelay: .milliseconds(80),
        newlineDelay: .milliseconds(120),
        paragraphBreakDelay: .milliseconds(200),
        openQuoteDelay: .milliseconds(30),
        closeQuoteDelay: .milliseconds(40),
        jitterEnabled: false,
        jitterRange: 0.85...1.15
    )

    static let dramatic = RevealTimingConfiguration(
        baseLetterDelay: .milliseconds(65),
        spaceDelay: .milliseconds(40),
        commaDelay: .milliseconds(250),
        periodDelay: .milliseconds(450),
        exclamationDelay: .milliseconds(450),
        questionDelay: .milliseconds(450),
        semicolonDelay: .milliseconds(280),
        colonDelay: .milliseconds(250),
        ellipsisDelay: .milliseconds(600),
        emDashDelay: .milliseconds(350),
        newlineDelay: .milliseconds(500),
        paragraphBreakDelay: .milliseconds(750),
        openQuoteDelay: .milliseconds(120),
        closeQuoteDelay: .milliseconds(150),
        jitterEnabled: true,
        jitterRange: 0.85...1.15
    )
}
```

## 4. Preventing Transaction Leakage

When updating `revealCount` inside the reveal loop, suppress all animations:

```swift
var transaction = Transaction(animation: nil)
transaction.disablesAnimations = true
withTransaction(transaction) {
    revealCount = i + 1
}
```

This ensures the cursor's `PhaseAnimator` blink does not propagate to the text content. Without this, the text view may inherit the cursor's easing curve, causing subtle opacity flicker on each character append.

## 5. Text View Identity Stability

The `Text` view must maintain structural identity across all updates:

```swift
// CORRECT — stable identity, content changes trigger diff only
Text(sourceText.prefix(upTo: revealIndex))

// WRONG — identity changes on every update, view destroyed and recreated
Text(sourceText.prefix(upTo: revealIndex))
    .id(revealCount)
```

For custom `Equatable` optimization:

```swift
struct RevealTextView: View, Equatable {
    let sourceText: String
    let revealCount: Int

    static func == (lhs: Self, rhs: Self) -> Bool {
        lhs.revealCount == rhs.revealCount
    }

    var body: some View {
        let indices = Array(sourceText.indices)
        let endIndex = revealCount > 0 ? indices[min(revealCount - 1, indices.count - 1)] : sourceText.startIndex
        Text(sourceText.prefix(upTo: endIndex))
    }
}
```

The `Equatable` conformance compares a single `Int` instead of SwiftUI's default reflection-based comparison of all stored properties.

## 6. Restart Protocol

When the source text changes (e.g., navigating to a new chapter passage):

```swift
.onChange(of: sourceText) {
    revealCount = 0
    isRevealing = false
    revealID = UUID()  // Triggers .task(id:) cancellation and restart
}
```

The `UUID` change cancels the running loop. The new `.task(id:)` invocation starts a fresh reveal with `revealCount = 0`.
