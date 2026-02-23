---
name: typewriter-text-animation
description: Enforce production-grade progressive text reveal systems with variable timing semantics, deterministic cursor behavior, scroll stability, safe cancellation, and optional audio synchronization for narrative iOS applications. Use when writing or reviewing typewriter-style text animation, character-by-character reveal loops, async-driven text timing, cursor blink implementations, ScrollViewReader auto-scroll, skip-to-end tap handling, or audio-synced text pacing. Triggers on progressive text reveal, typewriter animation, narrative text timing, character reveal loop, text presentation pacing, or any task producing incremental text display code in SwiftUI.
---

# Typewriter Text Animation

Governing standard for all progressive text reveal code in this repository. Every reveal system must produce natural cadence through variable timing, maintain deterministic cursor state, keep revealed text visible without layout jumps, cancel cleanly on user interaction, and optionally synchronize with audio playback. Fixed-interval `Timer` loops with uniform character spacing are prohibited.

---

## 1. Reveal Philosophy

Progressive text reveal is a narrative pacing tool. It controls reading speed, creates tension, and punctuates emotional beats. Mechanical uniform timing destroys these properties.

**Rules:**

- Every reveal system must vary timing by character class. Punctuation pauses longer than letters. Paragraph breaks breathe.
- The reveal loop is the single source of truth for which characters are visible. No secondary state tracks "revealed text" separately.
- Reveal state is an integer index into a pre-computed index array. The displayed text is always `sourceText.prefix(upTo: indices[revealCount])`.
- The user can skip to full reveal at any point. Skip is instantaneous — no fast-forward animation.
- Reveal must not block the main thread. All timing is async. All state mutations execute on `@MainActor`.

---

## 2. Reveal Engine Architecture

### 2.1 Timing Driver Selection (BLOCKING)

Use `Task.sleep(until:)` with `ContinuousClock` inside a `.task` modifier. This is the only approved timing driver.

**Prohibited timing drivers:**

| Driver                     | Reason for Prohibition                                                   |
| -------------------------- | ------------------------------------------------------------------------ |
| `Timer.publish`            | Combine dependency; RunLoop mode sensitivity; no structured cancellation |
| `Timer.scheduledTimer`     | UIKit holdover; requires manual invalidation; RunLoop-bound              |
| `Task.sleep(for:)`         | Accumulates drift across iterations; each sleep adds scheduling latency  |
| `DispatchQueue.asyncAfter` | No cancellation; no structured concurrency integration                   |
| `CADisplayLink`            | Frame-rate granularity (8–16ms) is overkill for character-level timing   |
| `TimelineView`             | Per-frame body evaluation; unnecessary for 20–50ms character intervals   |

### 2.2 Deadline-Based Loop Pattern (BLOCKING)

Prevent timing drift by computing each deadline from a fixed baseline:

```
var deadline = clock.now
for i in 0..<indices.count {
    guard !Task.isCancelled else { break }
    revealCount = i + 1
    deadline += delay(for: character[i])
    try await clock.sleep(until: deadline)
}
```

Each deadline is `baseline + cumulative_delay`, not `previousWakeTime + interval`. This eliminates drift from thread pool scheduling latency.

### 2.3 Pre-Computed Index Array (BLOCKING)

Swift `String.Index` advancement is O(n) due to variable-width Unicode encoding. Pre-compute all indices once:

```
let indices = Array(sourceText.indices)
let characters = Array(sourceText)
```

Access `indices[revealCount]` for O(1) lookup. Never call `sourceText.index(sourceText.startIndex, offsetBy: n)` inside the reveal loop.

### 2.4 View Integration

The reveal loop lives in a `.task(id:)` modifier on the text container view. The `id` parameter enables cancellation and restart:

- When the source text changes, the `id` changes, cancelling the old reveal and starting a new one.
- When the view disappears, SwiftUI cancels the task automatically.
- No manual `Task` storage or `onDisappear` cleanup required.

### 2.5 Prohibited Engine Patterns

- `AsyncStream` wrapping a timer loop. Adds indirection without benefit for a single-consumer reveal.
- `withTaskGroup` for reveal timing. Character reveal is sequential by definition.
- `@Published` property wrappers on the reveal count. Use `@State` or `@Observable` with direct mutation.
- Storing the `Task` handle manually when `.task(id:)` provides lifecycle management.

---

## 3. Timing Semantics

### 3.1 Character Class Delay Table (BLOCKING)

Every reveal implementation must define named constants for these delay classes. No magic numbers.

| Character Class          | Delay Range       | Named Constant Pattern    |
| ------------------------ | ----------------- | ------------------------- |
| Letter (a–z, A–Z)        | 30–50ms           | `baseLetterDelay`         |
| Digit (0–9)              | 30–50ms           | `baseDigitDelay`          |
| Space                    | 20–35ms           | `spaceDelay`              |
| Comma (`,`)              | 120–180ms         | `commaDelay`              |
| Period (`.`)             | 200–350ms         | `periodDelay`             |
| Exclamation (`!`)        | 200–350ms         | `exclamationDelay`        |
| Question mark (`?`)      | 200–350ms         | `questionDelay`           |
| Semicolon (`;`)          | 150–200ms         | `semicolonDelay`          |
| Colon (`:`)              | 120–180ms         | `colonDelay`              |
| Ellipsis (`…` or `...`)  | 300–500ms         | `ellipsisDelay`           |
| Em dash (`—`)            | 150–250ms         | `emDashDelay`             |
| Newline (`\n`)           | 250–450ms         | `newlineDelay`            |
| Paragraph break (`\n\n`) | 400–700ms         | `paragraphBreakDelay`     |
| Opening quote (`"`, `'`) | 60–100ms          | `openQuoteDelay`          |
| Closing quote (`"`, `'`) | 80–120ms          | `closeQuoteDelay`         |
| All other characters     | `baseLetterDelay` | Fallback to letter timing |

### 3.2 Delay Resolution Function (BLOCKING)

A single pure function maps a character (and optional context of the next character) to a `Duration`. This function must:

- Accept the current `Character` and optionally the next `Character` for lookahead.
- Return a `Duration` value (not `TimeInterval`).
- Handle the paragraph break case: when current character is `\n` and next character is also `\n`, return `paragraphBreakDelay` and advance the index by 2.
- Handle the three-dot ellipsis case: when current and next two characters are `.`, return `ellipsisDelay` and advance the index by 3.
- Be deterministic. No randomization unless explicitly configured as a "jitter" parameter.

### 3.3 Optional Jitter

For more organic feel, a small random variance may be applied:

- Maximum jitter: ±15% of the base delay for that character class.
- Jitter source: `Double.random(in:)` seeded per-passage for reproducibility during testing.
- Jitter is opt-in. Default configuration produces deterministic timing.

### 3.4 Timing Configuration (BLOCKING)

All timing values must be encapsulated in a single configuration type:

```
struct RevealTimingConfiguration {
    let baseLetterDelay: Duration
    let commaDelay: Duration
    let periodDelay: Duration
    // ... all character classes
    let jitterEnabled: Bool
    let jitterRange: ClosedRange<Double>  // e.g., 0.85...1.15

    static let standard: Self  // Default narrative pacing
    static let fast: Self      // Rapid reveal for short UI text
    static let dramatic: Self  // Slow reveal for emotional beats
}
```

No timing values hardcoded outside this type.

### 3.5 Prohibited Timing Patterns

- Uniform delay for all characters. Produces robotic cadence.
- Delay values defined as literals inside the reveal loop.
- `usleep()` or `Thread.sleep()` for timing. Blocks the thread.
- Randomizing delay without a bounded range. Unbounded randomness produces jarring rhythm.

---

## 4. Cursor Behavior

### 4.1 Implementation (BLOCKING — iOS 17+)

Use `PhaseAnimator` with a two-phase boolean for cursor blink:

```
PhaseAnimator([true, false]) { phase in
    cursorView.opacity(phase ? 1 : 0)
} animation: { _ in
    .easeInOut(duration: 0.53)
}
```

The cursor is a separate `View` in the hierarchy. It does not share state or animation transactions with the text content.

### 4.2 Cursor Lifecycle States

| Reveal State | Cursor Behavior         | Implementation                                                 |
| ------------ | ----------------------- | -------------------------------------------------------------- |
| Not started  | Hidden                  | `opacity(0)` — no PhaseAnimator active                         |
| Revealing    | Blinking at end of text | PhaseAnimator active, positioned after last revealed character |
| Completed    | Hidden after delay      | Fade out with `.smooth(duration: 0.3)` after 1.0s hold         |
| Skipped      | Hidden immediately      | Set `opacity(0)` in the skip handler                           |

### 4.3 Cursor Positioning

The cursor tracks the trailing edge of revealed text. Two approaches, in order of preference:

1. **Inline cursor**: Append a cursor character (`|` or `▏`) to the displayed text string. Remove it when reveal completes. Zero layout complexity.
2. **Overlay cursor**: Position a `Rectangle` view using text measurement. Requires `GeometryReader` or manual width calculation. Use only when the cursor must have independent styling (color, thickness) that inline text cannot express.

### 4.4 Transaction Isolation (BLOCKING)

Cursor blink animation must not affect text content updates. Enforce isolation:

- The cursor view exists in a sibling branch, not a parent branch, of the text view.
- Text state mutations use `Transaction(animation: nil)` with `disablesAnimations = true` to prevent the cursor's repeating animation from leaking into text updates.
- Never wrap `revealCount` mutations in `withAnimation`. The text grows without animation; only the cursor animates.

### 4.5 Prohibited Cursor Patterns

- `withAnimation(.repeatForever)` — cannot be stopped cleanly; leaks transactions to siblings.
- Toggling `@State var cursorVisible` on a `Timer` — manual lifecycle; orphan risk.
- Multiple cursor views for the same text block.
- Cursor blink rate faster than 400ms full cycle (0.4s). Causes visual flutter.

---

## 5. Scroll Coordination

### 5.1 Strategy Selection

| iOS Target | Strategy                       | Implementation                                                     |
| ---------- | ------------------------------ | ------------------------------------------------------------------ |
| iOS 17+    | `defaultScrollAnchor(.bottom)` | Set on `ScrollView`. Anchors to bottom as content grows.           |
| iOS 16     | `ScrollViewReader` + sentinel  | Place invisible anchor view after text. Call `scrollTo` on update. |
| iOS 18+    | `ScrollPosition` + edge scroll | Bind `ScrollPosition`, call `scrollTo(edge: .bottom)` on update.   |

### 5.2 Sentinel View Pattern (iOS 16 Fallback)

```
ScrollViewReader { proxy in
    ScrollView {
        Text(revealedText)
        Color.clear
            .frame(height: 1)
            .id("revealAnchor")
    }
    .onChange(of: revealCount) {
        withAnimation(.easeOut(duration: 0.15)) {
            proxy.scrollTo("revealAnchor", anchor: .bottom)
        }
    }
}
```

### 5.3 Scroll Behavior Rules (BLOCKING)

- Auto-scroll fires on every character reveal that increases visible content height. Not on every character — only when the text wraps to a new line.
- Auto-scroll uses a short ease-out animation (100–200ms). Springs cause bouncing during rapid character updates.
- If the user manually scrolls up during reveal, auto-scroll pauses. Resume auto-scroll only when the user scrolls back to within one screen-height of the bottom.
- `scrollTo` calls must not occur during the first frame of view appearance. Guard with a `didAppear` flag or `.task` to avoid layout thrashing on initial render.

### 5.4 User Scroll Detection

Track whether the user has scrolled away from the bottom. On iOS 18+, use `onScrollPhaseChange` to detect `.tracking` phase. On iOS 16–17, compare `scrollPosition` or content offset against content height minus viewport height.

When user-scrolled-away is detected, suppress auto-scroll until the user returns to the bottom region. This prevents the reveal from fighting the user's reading position.

### 5.5 Layout Stability (BLOCKING)

- Constrain the `Text` view width with `.frame(maxWidth:)` or parent container width. Unconstrained text causes horizontal re-measurement on every content change.
- Use `fixedSize(horizontal: false, vertical: true)` if the text must grow vertically without constraint.
- Never use `.id(revealCount)` or `.id(revealedText)` on the `Text` view. Changing identity destroys and recreates the view, resetting all state.
- Padding applied to the container, not to individual text views inside a `VStack`.

### 5.6 Prohibited Scroll Patterns

- `scrollTo` without animation — causes instantaneous jumps visible as layout flicker.
- `scrollTo` on every single character regardless of line wrapping — burns CPU on no-op scrolls.
- Using `LazyVStack` for a single growing `Text` view. `LazyVStack` is for multiple discrete children.
- Wrapping the entire `ScrollView` in a `GeometryReader` to manually calculate scroll position.

---

## 6. Cancellation & Skip Handling

### 6.1 Skip-to-End Protocol (BLOCKING)

When the user taps during reveal:

```
1. Set revealCount = indices.count (full text visible instantly)
2. Set isRevealing = false
3. Hide cursor
4. Cancel the reveal task (via .task(id:) parameter change)
5. Fire the onRevealComplete callback
```

All five steps execute in a single synchronous state mutation. No partial states.

### 6.2 Task Cancellation Contract

The reveal loop must check `Task.isCancelled` at the top of every iteration, before mutating state:

```
for i in 0..<indices.count {
    guard !Task.isCancelled else {
        // Set final consistent state
        revealCount = indices.count
        isRevealing = false
        break
    }
    revealCount = i + 1
    // ... sleep
}
```

The `guard` placement ensures no state mutation occurs after cancellation.

### 6.3 Post-Cancellation State Guarantee

After cancellation (whether from skip, view disappear, or text change), the following invariants hold:

- `revealCount == indices.count` (full text visible) OR `revealCount == 0` (reset for new text).
- `isRevealing == false`.
- No pending `Task.sleep` awaits.
- The `onRevealComplete` callback has fired exactly once, or not at all if the view was dismissed.

### 6.4 Debounce Protection

If the user taps multiple times rapidly during reveal, the skip handler must be idempotent. Check `isRevealing` before executing skip logic. If already `false`, ignore the tap.

### 6.5 Prohibited Cancellation Patterns

- `Task.cancel()` on a stored `Task` reference when `.task(id:)` is available.
- Fast-forward animation (revealing remaining characters at accelerated speed). Skip is instantaneous.
- Setting `revealCount` without also setting `isRevealing = false`.
- Calling `onRevealComplete` from both the natural loop end and the skip handler without deduplication.

---

## 7. Audio Synchronization

### 7.1 Architecture

Audio synchronization is an overlay on the base reveal engine. When audio is present, the timing source switches from `ContinuousClock` to audio timestamp polling. When audio is absent, the engine falls back to clock-driven timing.

### 7.2 Timestamp Manifest (BLOCKING when audio sync is required)

Define a manifest mapping character indices to audio timestamps:

```
struct RevealTimestamp {
    let characterIndex: Int
    let audioTime: TimeInterval  // seconds into the audio track
}
```

The manifest is an array sorted by `audioTime`. It need not contain an entry for every character — entries define sync anchors. Characters between anchors interpolate using the base timing table.

### 7.3 Sync Loop Pattern

See [references/audio-sync-protocol.md](references/audio-sync-protocol.md) for the full synchronization protocol.

When audio is the timing source:

```
1. On each tick (16ms via TimelineView or polled via Task.sleep):
2. Read AVAudioPlayer.currentTime
3. Binary search the manifest for the highest index where audioTime <= currentTime
4. If targetIndex > revealCount, advance revealCount to targetIndex
5. If targetIndex <= revealCount, hold (do not rewind)
```

### 7.4 Drift Correction

Audio-synced reveal must handle:

- **Audio stall**: If `currentTime` stops advancing (buffer underrun), hold the current reveal position. Do not advance on clock time alone.
- **Audio seek**: If `currentTime` jumps forward (user scrubs), advance `revealCount` to match. If it jumps backward, do not rewind — hold current position.
- **Audio end**: When playback completes, reveal any remaining characters using clock-driven timing at the base rate.

### 7.5 Graceful Fallback (BLOCKING)

The reveal system must function identically whether audio is present or absent. The timing source is injected, not hardcoded:

```
protocol RevealTimingSource {
    func nextDelay(for character: Character, at index: Int) async throws -> Duration
}
```

Two conformances: `ClockTimingSource` (uses delay table) and `AudioTimingSource` (uses manifest + polling). The reveal loop does not know which source it uses.

### 7.6 Prohibited Audio Sync Patterns

- Polling `currentTime` faster than display refresh rate. 60Hz (16ms) is sufficient.
- Rewinding `revealCount` when audio seeks backward. Text should never "un-reveal."
- Using `AVAudioPlayerDelegate.audioPlayerDidFinishPlaying` as the sole signal for reveal completion — the delegate fires after a delay; use a polling check.
- Hardcoding `AVAudioPlayer` dependency. The timing source protocol must accept any time-position provider.

---

## 8. Performance Engineering

### 8.1 Text Rendering Budget

At 30–50ms per character (20–33 characters/second), each update has a budget of 30ms minus render time. SwiftUI `Text` body evaluation, string diffing, and layout for passages up to 5,000 characters completes in <1ms on A15+ silicon.

### 8.2 String Operations (BLOCKING)

- Use `sourceText.prefix(upTo: indices[revealCount])` to produce a `Substring`. This shares storage with the original string — zero allocation per update.
- Never construct a new `String` on each update via concatenation or interpolation.
- For styled text: pre-parse the full `AttributedString` once at initialization. Slice the pre-parsed result per update. Never re-parse AttributedString on each character.

### 8.3 View Hierarchy Isolation

Isolate the text display view so that `revealCount` changes invalidate only the text subtree:

- The text view receives `revealCount` and `sourceText` as parameters. No other state.
- Cursor, scroll controls, and chapter navigation exist as siblings, not ancestors.
- Apply `.equatable()` on the text view with a custom `Equatable` conformance that compares only `revealCount` — an `Int` comparison is faster than SwiftUI's default reflection-based diffing.

### 8.4 Large Passage Threshold

For passages exceeding 10,000 characters, split into paragraph-level `Text` views inside a `LazyVStack`. Each paragraph reveals independently. Only the active paragraph runs a reveal loop; completed paragraphs display static text.

### 8.5 Prohibited Performance Patterns

- Animating the `Text` content change with `withAnimation`. Text growth must not trigger SwiftUI's animation interpolation on the text value.
- Using `.id(revealCount)` on the `Text` view. Forces view destruction and recreation on every character.
- Concatenating `Text` views with `+` operator for styled reveal. Creates separate text runs with independent measurement passes.
- `GeometryReader` inside the text container. Causes layout recalculation on every content change.

---

## 9. State Architecture

### 9.1 Minimum Required State

```
@State private var revealCount: Int = 0
@State private var isRevealing: Bool = false
```

All other state derives from these two values:

- `revealedText` = `sourceText.prefix(upTo: indices[revealCount])`
- `isComplete` = `revealCount == indices.count`
- `cursorVisible` = `isRevealing`

### 9.2 State Mutation Rules (BLOCKING)

- `revealCount` increments by 1 (or more for multi-character tokens like `\n\n` or `...`) inside the reveal loop.
- `revealCount` jumps to `indices.count` on skip.
- `isRevealing` is set to `true` when the loop starts and `false` when it ends (naturally or via cancellation).
- No other code path mutates `revealCount` or `isRevealing`.

### 9.3 Callback Contract

The reveal engine exposes two callbacks:

| Callback              | Fires When                     | Guarantee                                   |
| --------------------- | ------------------------------ | ------------------------------------------- |
| `onRevealComplete`    | All characters visible         | Fires exactly once per reveal pass          |
| `onCharacterRevealed` | Each character becomes visible | Fires `indices.count` times or 0 if skipped |

`onRevealComplete` fires whether the reveal ended naturally or was skipped. The caller does not distinguish between the two.

---

## 10. Code Review Checklist

Reject typewriter animation code if ANY are true:

- [ ] Uses `Timer.publish`, `Timer.scheduledTimer`, or `DispatchQueue.asyncAfter` as the timing driver.
- [ ] Uses `Task.sleep(for:)` instead of `Task.sleep(until:)` with deadline tracking.
- [ ] Uniform delay for all character classes (no punctuation pauses).
- [ ] Timing delay values are literal numbers inside the reveal loop, not named constants.
- [ ] `revealCount` mutated with `withAnimation` — text growth must not animate.
- [ ] Cursor implemented with `withAnimation(.repeatForever)` instead of `PhaseAnimator`.
- [ ] Cursor and text share the same animation transaction scope.
- [ ] `.id(revealCount)` or `.id(revealedText)` applied to the `Text` view.
- [ ] `String.Index` computed via `index(_:offsetBy:)` inside the loop instead of pre-computed array.
- [ ] No cancellation check (`Task.isCancelled`) at the top of the reveal loop.
- [ ] Skip handler does not set both `revealCount` and `isRevealing` atomically.
- [ ] `onRevealComplete` can fire more than once per reveal pass.
- [ ] Auto-scroll uses spring animation (causes bounce during rapid updates).
- [ ] Auto-scroll fires on every character instead of only on line-wrap events.
- [ ] No `RevealTimingConfiguration` type — timing values scattered across the codebase.
- [ ] `AttributedString` re-parsed on every character update instead of pre-parsed and sliced.
- [ ] Audio sync polls `currentTime` without fallback for absent audio.
- [ ] Audio sync rewinds `revealCount` when playback seeks backward.
- [ ] Manual `Task` storage when `.task(id:)` modifier is available.
- [ ] `GeometryReader` inside the text container view hierarchy.

---

## References

- `references/reveal-engine-patterns.md` — Production code patterns for the reveal loop, timing configuration, and skip handling.
- `references/timing-tables.md` — Character class classification, delay value derivation, jitter implementation.
- `references/audio-sync-protocol.md` — Audio timestamp manifest format, polling loop, drift correction, graceful fallback protocol.
