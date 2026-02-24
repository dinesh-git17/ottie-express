# E2.2: Build TypewriterText Component

**Phase:** 2 - Navigation Shell and Shared UI
**Class:** Infrastructure
**Design Doc Reference:** SS4 Architecture Overview (SharedUI, Technology Stack, Concurrency Model), SS7 Chapter 1 (The Letter screen timing specification), SS2 Design System (Typography)
**Dependencies:**

- Phase 1: Core Systems and Design Foundation (exit criteria met)
- E1.1: Define design system tokens (`Font.typewriterText` and `Font.poemLetter` available for default and override fonts)

---

## Goal

Implement the reusable progressive text reveal component with configurable character timing, punctuation pauses, line break pauses, paragraph break pauses, tap-to-complete, optional per-character callback, and cancellation-safe async animation for use across all narrative chapters.

---

## Scope

### File Inventory

| File                                         | Action | Responsibility                                                                                                                                           |
| -------------------------------------------- | ------ | -------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `OttieExpress/SharedUI/TypewriterText.swift` | Create | Progressive text reveal view with configurable timing intervals, tap-to-complete gesture, optional per-character callback, and async cancellation safety |

### Integration Points

**TypewriterText.swift**

- Imports from: `Font+DesignTokens.swift` (`Font.typewriterText` for New York Regular 18pt default)
- Imported by: None within Phase 2 (consumed by E5.5 Chapter 1 letter screen, E6.x Chapter 2 API reply display, E7.x Chapter 3 final line, E9.x Chapter 6 final message)
- State reads: None (self-contained; receives text content and configuration as parameters)
- State writes: None

---

## Out of Scope

- Audio playback for `sfx-typewriter-tick` (the component exposes an optional `onCharacterReveal` callback closure; the calling chapter wires `AudioService.play(_:)` to it in its own scope)
- `ScrollViewReader` auto-scroll coordination during text reveal (the parent view manages scroll position; TypewriterText exposes its current reveal progress via a binding or callback for the parent to coordinate)
- Cursor blink animation at the insertion point or end of revealed text (not specified in SS2 or SS4 SharedUI definition; deferred unless a future epic adds it)
- Fade-in text variant for "breath" lines like "No checkpoints needed" in SS7 (the calling chapter uses a standard SwiftUI opacity animation on a separate `Text` view for these; TypewriterText handles character-by-character reveal only)
- Haptic feedback synchronized to character reveal cadence (each chapter decides independently whether to wire haptics via the callback closure)
- Rich text or attributed string support within the revealed text (all TypewriterText consumers use plain strings per design doc)

---

## Definition of Done

- [ ] Characters appear one at a time at the configured base interval (default 40ms) when the view appears on screen
- [ ] Punctuation characters (`.`, `,`, `!`, `?`, `:`, `;`) add a configurable additional pause (default 200ms) after the character appears
- [ ] Single line break characters add a configurable pause (default 400ms) before the next character appears
- [ ] Paragraph breaks (double newline `\n\n`) add a configurable pause (default 800ms) before the next character appears
- [ ] Tapping the view reveals all remaining text immediately and stops the animation loop
- [ ] Text renders in `Font.typewriterText` (New York Regular 18pt) by default when no font override is provided
- [ ] Text renders in a caller-specified font when a font override parameter is provided
- [ ] The optional `onCharacterReveal` callback fires once per revealed character, receiving the revealed `Character` value
- [ ] The animation `Task` cancels cleanly when the view disappears mid-reveal, producing no runtime warnings and no orphaned background tasks
- [ ] Passing an empty string renders no visible content and does not start the animation loop

---

## Implementation Notes

The reveal loop runs inside a `.task {}` view modifier attached to the `TypewriterText` view body. This guarantees automatic cancellation when the view leaves the hierarchy (view disappear). Each loop iteration: reveal one character, fire the `onCharacterReveal` callback if provided, determine the pause duration (base interval + punctuation/line break/paragraph break addition), then `try await Task.sleep(for:)`. Check `Task.isCancelled` before each sleep to exit promptly on cancellation. Reference: SS4 Technology Stack, concurrency model ("Async work is scoped to view lifecycle via `.task {}` modifier for automatic cancellation on view disappear").

The timing configuration is a struct with four `Duration` properties:

- `characterInterval`: default `.milliseconds(40)` per SS7 letter timing
- `punctuationPause`: default `.milliseconds(200)` per SS7 letter timing
- `lineBreakPause`: default `.milliseconds(400)` per SS7 letter timing
- `paragraphBreakPause`: default `.milliseconds(800)` per SS7 letter timing

Call sites override individual values for chapter-specific variants (e.g., 60ms character interval for Chapter 5, 80ms per-word pacing for Chapter 6). The struct provides a `static let`default`` computed from these SS7 values. Reference: SS7 The Letter screen timing specification.

Punctuation detection checks the revealed character against the set `[".", ",", "!", "?", ":", ";"]`. Line break detection checks for `\n` not preceded by another `\n`. Paragraph break detection checks for consecutive `\n\n` sequences in the source text. The paragraph break pause replaces (not adds to) the line break pause for double-newline boundaries. Reference: SS7 The Letter timing rules.

Tap-to-complete uses an `@State private var isComplete: Bool` flag. A `.onTapGesture` modifier on the view sets this flag. The animation loop checks the flag on each iteration. When set, the loop assigns the full source text to the displayed string in a single update and exits. This avoids a visual stutter by performing one atomic state mutation rather than fast-forwarding through remaining characters. Reference: SS7 ("tap to complete" interaction pattern).

The default font is `Font.typewriterText` from `Font+DesignTokens.swift` (New York Regular 18pt per SS2 Typography table). The view accepts an optional `font: Font` parameter. When provided, the override replaces the default. This enables italic variants (New York Italic 20pt for final lines per SS7) and size variants (New York Regular 17pt for poems per SS8) without requiring separate components. Reference: SS2 Typography table, typewriter text role.
