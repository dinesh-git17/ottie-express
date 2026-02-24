# E9.3: Build Memory Cards and Sequential Constellation Unlock

**Phase:** 9 - Chapter 6: The Constellations
**Class:** Feature
**Design Doc Reference:** §12 Chapter 6 (Constellation Interaction memory cards, sequential unlock, 6th constellation special behavior), §2 Design System (typography New York Regular 16pt, motion principles)
**Dependencies:**

- Phase 2: Navigation Shell and Shared UI (exit criteria met)
- E9.1: Build star field and constellation data model (ConstellationData with memory card text and constellation definitions available)
- E9.2: Implement constellation drawing interaction (`onComplete` callback from ConstellationDrawingView available)
- E1.1: Define design system tokens (typography New York Regular, color tokens, motion constants)

---

## Goal

Implement the memory card slide-up presentation with per-constellation text, tap-to-dismiss behavior, sequential constellation unlock progression, and constellation 6 special handling including the "Carolina." typewriter reveal.

---

## Scope

### File Inventory

| File                                                          | Action | Responsibility                                                                                                                                                            |
| ------------------------------------------------------------- | ------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `OttieExpress/Chapters/Chapter6/MemoryCardView.swift`         | Create | Render frosted memory card with `memory-card-bg` styling, constellation-specific text in New York Regular 16pt, slide-up/slide-down animation, and tap-to-dismiss gesture |
| `OttieExpress/Chapters/Chapter6/ConstellationSequencer.swift` | Create | Manage sequential constellation progression from 1 through 6, present memory cards after lock, advance unlock state on dismissal, and handle constellation 6 special flow |

### Integration Points

**MemoryCardView.swift**

- Imports from: `Chapter6Constants.swift` (E9.1), `OttieExpress/Extensions/Color+DesignTokens.swift`, `OttieExpress/Extensions/Font+DesignTokens.swift`
- Imported by: `ConstellationSequencer.swift`
- State reads: Memory card text string (passed as parameter)
- State writes: None (exposes `onDismiss` callback)

**ConstellationSequencer.swift**

- Imports from: `ConstellationData.swift` (E9.1), `Chapter6Constants.swift` (E9.1), `MemoryCardView.swift`, `ConstellationDrawingView.swift` (E9.2)
- Imported by: `Chapter6View.swift` (E9.4)
- State reads: Constellation completion signals from ConstellationDrawingView `onComplete` callbacks
- State writes: Active constellation index (drives which constellation dots are visible and pulsing), constellation 6 completion flag (consumed by E9.4 to trigger sky illuminate)

---

## Out of Scope

- Star field background rendering and star fade-in animation (covered by E9.1)
- Constellation data model and star position definitions (covered by E9.1)
- Drag-to-connect drawing mechanic, line rendering, and per-connection SFX (covered by E9.2)
- Sky illuminate sequence, camera zoom-out, and full constellation flash (covered by E9.4)
- Final message typewriter, continue prompt, and chapter completion wiring (covered by E9.4)
- BGM playback and SFX coordination beyond constellation interactions (wired by E9.4)
- Audio crossfade to ending screen (covered by E10.2)
- Memory card slide-up/slide-down spring tuning and gesture velocity matching (deferred to Phase 12 integration testing on physical device)

---

## Definition of Done

- [ ] Memory card slides up from the bottom of the screen with `memory-card-bg` styling after a constellation locks, using spring animation (stiffness 280, damping 22)
- [ ] Memory card text renders in New York Regular 16pt, matching §12 verbatim for each of the 5 named constellations (3 lines per card, no em dashes)
- [ ] Tapping anywhere on the memory card dismisses it with a slide-down animation
- [ ] After card dismissal for constellations 1 through 4, the next constellation's dots fade in and begin pulsing within 0.5 seconds
- [ ] Constellations 1 through 5 unlock strictly in sequence; constellation N+1 dots are invisible until constellation N memory card is dismissed
- [ ] After constellation 5 memory card dismissal, constellation 6 dots appear brighter (higher base opacity) and with a slight tremble animation (1-2pt displacement per dot)
- [ ] Constellation 6 dots provide no name hint before, during, or after drawing begins
- [ ] On constellation 6 lock, "Carolina." typewriters character-by-character above the otter shape in the constellation name position
- [ ] The "Carolina." text holds visible for exactly 2 seconds after the typewriter completes before the sequencer fires its completion signal
- [ ] ConstellationSequencer exposes an `onAllComplete` closure that fires exactly once after the constellation 6 hold completes, signaling E9.4 to begin sky illuminate

---

## Implementation Notes

Design doc §12: "After lock: memory card slides up from the bottom of the screen, `memory-card-bg` style, text in New York Regular 16pt. She taps to dismiss the card and the next constellation dots appear." MemoryCardView uses a vertical `offset` transition from screen bottom (offset.y = screen height) to final position, animated with the baseline spring from §2 (stiffness 280, damping 22). Dismissal reverses the offset.

ConstellationSequencer maintains `currentConstellationIndex` (0-based, mapping to constellations 1-6). Its state machine:

1. `showingCard` (memory card visible for constellations 0-4, waiting for tap dismiss)
2. `advancingToNext` (card dismissed, next constellation dots fading in)
3. `constellation6Active` (constellation 6 drawing in progress, no card after lock)
4. `constellation6Hold` ("Carolina." visible, 2 second timer running)
5. `complete` (fires `onAllComplete`)

Constellation 6 special behavior per §12: "a new cluster of brighter, slightly trembling star dots appears. No name hint given." The tremble effect applies a repeating `offset` modifier with randomized x/y displacement of 1-2pt per dot, using `Animation.easeInOut(duration: 0.15).repeatForever(autoreverses: true)` with per-dot phase offsets to avoid synchronization.

The "Carolina." typewriter after constellation 6 lock is a localized character reveal within this component, not the shared TypewriterText (E2.2). Implement as a simple timer that appends one character per 40ms (matching the default typewriter interval from §12/§2.2 TypewriterText spec) to a displayed substring. After all characters are revealed, start a 2-second `Task.sleep` before firing `onAllComplete`.

Memory card text is sourced directly from `ConstellationData.constellations[index].memoryCardText` (E9.1). No transformation or formatting required.
