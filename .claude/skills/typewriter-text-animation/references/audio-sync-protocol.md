# Audio Synchronization Protocol

## 1. Timing Source Protocol

```swift
protocol RevealTimingSource: Sendable {
    /// Returns the delay before the next character should be revealed.
    /// Implementations may block (via async sleep) or return immediately.
    func waitForNext(character: Character, at index: Int, total: Int) async throws
}
```

Two conformances:

### ClockTimingSource (Default — No Audio)

```swift
struct ClockTimingSource: RevealTimingSource {
    let configuration: RevealTimingConfiguration
    private let clock = ContinuousClock()
    private var baseline: ContinuousClock.Instant?
    private var accumulatedDelay: Duration = .zero

    mutating func waitForNext(character: Character, at index: Int, total: Int) async throws {
        if baseline == nil { baseline = clock.now }

        let delay = configuration.delay(for: character, next: nil)
        accumulatedDelay += delay

        try await clock.sleep(until: baseline!.advanced(by: accumulatedDelay))
    }
}
```

Deadline-based. Each sleep target is `baseline + cumulativeDelay`. Thread scheduling latency does not accumulate.

### AudioTimingSource (Audio-Synced)

```swift
struct AudioTimingSource: RevealTimingSource {
    let manifest: [RevealTimestamp]
    let timeProvider: @Sendable () -> TimeInterval  // e.g., { player.currentTime }
    let fallback: ClockTimingSource
    private let clock = ContinuousClock()

    func waitForNext(character: Character, at index: Int, total: Int) async throws {
        // Find the next sync anchor at or after this index
        guard let nextAnchor = manifest.first(where: { $0.characterIndex > index }) else {
            // Past the last anchor — fall back to clock timing
            try await fallback.waitForNext(character: character, at: index, total: total)
            return
        }

        // Poll until audio reaches the anchor's timestamp
        while true {
            try Task.checkCancellation()
            let currentTime = timeProvider()
            if currentTime >= nextAnchor.audioTime { return }

            // Poll at display refresh rate
            try await clock.sleep(for: .milliseconds(16))
        }
    }
}
```

## 2. Timestamp Manifest Format

```swift
struct RevealTimestamp: Sendable {
    /// Character index in the source text (0-based).
    let characterIndex: Int

    /// Seconds into the audio track when this character should be visible.
    let audioTime: TimeInterval
}
```

### Manifest Construction Rules

- Sorted by `audioTime` ascending. Binary search requires sorted order.
- Anchor density: one per word boundary or one per clause. Not one per character — that defeats the purpose of natural timing.
- First anchor: `characterIndex: 0, audioTime: 0.0`. Reveal starts when audio starts.
- Last anchor: `characterIndex: text.count - 1, audioTime: <near end of audio>`. Ensures audio and text finish together.
- Between anchors: the timing source interpolates using the `ClockTimingSource` delay table. Anchors provide course correction; the delay table provides intra-word rhythm.

### Example Manifest

For the text `"Once upon a time, in a land far away..."` with a 4-second voiceover:

```swift
let manifest: [RevealTimestamp] = [
    RevealTimestamp(characterIndex: 0,  audioTime: 0.0),    // "O"
    RevealTimestamp(characterIndex: 5,  audioTime: 0.4),    // "u" (after "Once ")
    RevealTimestamp(characterIndex: 10, audioTime: 0.9),    // "a" (after "upon ")
    RevealTimestamp(characterIndex: 12, audioTime: 1.2),    // "t" (after "a ")
    RevealTimestamp(characterIndex: 17, audioTime: 1.6),    // "," (after "time")
    RevealTimestamp(characterIndex: 22, audioTime: 2.1),    // "a" (after "in ")
    RevealTimestamp(characterIndex: 27, audioTime: 2.6),    // "f" (after "land ")
    RevealTimestamp(characterIndex: 31, audioTime: 3.0),    // "a" (after "far ")
    RevealTimestamp(characterIndex: 39, audioTime: 3.8),    // "." (end)
]
```

## 3. Hybrid Timing: Anchors + Interpolation

Between consecutive anchors, the reveal engine uses clock-based timing from the delay table. At each anchor, it course-corrects:

```
Timeline:
  Audio: ──────────[anchor A]────────────[anchor B]────────────
  Reveal: ───clock delays───→ snap to A ───clock delays───→ snap to B

If reveal is AHEAD of anchor: hold (do not advance) until audio catches up.
If reveal is BEHIND anchor: advance revealCount to anchor index instantly.
```

This hybrid approach produces natural intra-word rhythm (from the delay table) with macro-level synchronization (from anchors).

## 4. Drift Correction

### Case 1: Audio Stall

`currentTime` stops advancing (buffer underrun, Bluetooth audio reconnect).

**Response:** Hold current `revealCount`. Do not advance on clock time alone. The polling loop in `AudioTimingSource.waitForNext` naturally pauses because `currentTime` never reaches the next anchor.

### Case 2: Audio Seek Forward

`currentTime` jumps ahead of the next anchor (user scrubs forward in debug mode).

**Response:** Advance `revealCount` to the highest anchor index where `audioTime <= currentTime`. This may skip multiple characters. The reveal "catches up" instantly.

### Case 3: Audio Seek Backward

`currentTime` jumps behind the current `revealCount` position.

**Response:** Do NOT rewind `revealCount`. Text never un-reveals. Continue from the current position. The audio will eventually catch up to the revealed text, at which point normal sync resumes.

### Case 4: Audio Ends Before Text

Audio playback completes but characters remain unrevealed.

**Response:** Switch to `ClockTimingSource` for remaining characters. The reveal engine detects audio completion when `currentTime >= duration` and falls through to clock-based timing at the `standard` rate.

### Case 5: Text Ends Before Audio

All characters are revealed but audio is still playing.

**Response:** The reveal is complete. Fire `onRevealComplete`. Audio continues playing independently. The cursor hides. No interaction between the two systems after reveal completes.

## 5. Time Provider Abstraction

The audio timing source does not reference `AVAudioPlayer` directly. It accepts a closure:

```swift
let timeProvider: @Sendable () -> TimeInterval
```

This decouples the reveal engine from the audio framework. Valid providers:

| Provider                       | Implementation                         |
| ------------------------------ | -------------------------------------- |
| `AVAudioPlayer`                | `{ [weak player] in player?.currentTime ?? 0 }` |
| `AVAudioEngine` + `PlayerNode` | Compute from `lastRenderTime` + `playerTime(forNodeTime:)` |
| Manual clock (testing)         | `{ Date().timeIntervalSince(startDate) }` |
| Fixed sequence (testing)       | Return pre-determined values from an array |

The reveal engine never imports `AVFoundation`. The audio layer provides the closure at initialization.

## 6. Integration Pattern

```swift
// Without audio — pure clock-driven
let timingSource = ClockTimingSource(configuration: .standard)
TypewriterText(
    sourceText: passage,
    timingSource: timingSource,
    onRevealComplete: { advanceChapter() }
)

// With audio — synced to voiceover
let player = try AVAudioPlayer(contentsOf: audioURL)
let timingSource = AudioTimingSource(
    manifest: chapterManifest,
    timeProvider: { [weak player] in player?.currentTime ?? 0 },
    fallback: ClockTimingSource(configuration: .standard)
)
player.play()
TypewriterText(
    sourceText: passage,
    timingSource: timingSource,
    onRevealComplete: { advanceChapter() }
)
```

The `TypewriterText` view is identical in both cases. The timing source injection is the only difference. No conditional branches inside the reveal loop.

## 7. Testing Audio Sync

### Deterministic Test Provider

```swift
struct MockTimeProvider {
    private var times: [TimeInterval]
    private var callIndex: Int = 0

    mutating func next() -> TimeInterval {
        guard callIndex < times.count else { return times.last ?? 0 }
        let time = times[callIndex]
        callIndex += 1
        return time
    }
}
```

Feed a sequence of `currentTime` values. Assert that `revealCount` matches expected values at each step.

### Assertions

- After time reaches anchor N: `revealCount >= manifest[N].characterIndex`.
- After audio ends: all characters revealed within 2 seconds (fallback clock rate).
- After audio seeks backward: `revealCount` never decreases.
- After cancellation: `revealCount == indices.count` and `isRevealing == false`.
