# E9.1: Build Star Field and Constellation Data Model

**Phase:** 9 - Chapter 6: The Constellations
**Class:** Feature
**Design Doc Reference:** §12 Chapter 6 (Star Field Entry, Constellation Interaction data definitions), §2 Design System (color palette, typography)
**Dependencies:**

- Phase 2: Navigation Shell and Shared UI (exit criteria met)
- E1.1: Define design system tokens (color tokens `night-base`, `parchment`, `warm-gold`; typography New York Regular 18pt; motion constants baseline spring)

---

## Goal

Implement the star field background view with sequential fade-in animation and define the complete constellation data model encoding star positions, connection sequences, names, and memory card text for all 6 constellations.

---

## Scope

### File Inventory

| File                                                     | Action | Responsibility                                                                                                                          |
| -------------------------------------------------------- | ------ | --------------------------------------------------------------------------------------------------------------------------------------- |
| `OttieExpress/Chapters/Chapter6/Chapter6Constants.swift` | Create | Define named constants for Chapter 6 timing, sizing, opacity, and layout values                                                         |
| `OttieExpress/Chapters/Chapter6/ConstellationData.swift` | Create | Define constellation data model with star positions, connection sequences, display names, and memory card text for all 6 constellations |
| `OttieExpress/Chapters/Chapter6/StarFieldView.swift`     | Create | Render `night-base` background with sequential star fade-in over 2 seconds and drifting intro line                                      |

### Integration Points

**Chapter6Constants.swift**

- Imports from: `OttieExpress/Extensions/Color+DesignTokens.swift`, `OttieExpress/Extensions/Font+DesignTokens.swift`, `OttieExpress/Extensions/DesignConstants.swift`
- Imported by: `StarFieldView.swift`, `ConstellationData.swift`, `ConstellationDrawingView.swift` (E9.2), `StarDotView.swift` (E9.2), `ConnectionLineView.swift` (E9.2), `MemoryCardView.swift` (E9.3), `ConstellationSequencer.swift` (E9.3), `SkyIlluminateView.swift` (E9.4), `FinalMessageView.swift` (E9.4), `Chapter6View.swift` (E9.4)
- State reads: None
- State writes: None

**ConstellationData.swift**

- Imports from: `Chapter6Constants.swift`
- Imported by: `ConstellationDrawingView.swift` (E9.2), `StarDotView.swift` (E9.2), `MemoryCardView.swift` (E9.3), `ConstellationSequencer.swift` (E9.3), `SkyIlluminateView.swift` (E9.4), `ConstellationBackdrop.swift` (E10.2)
- State reads: None
- State writes: None

**StarFieldView.swift**

- Imports from: `Chapter6Constants.swift`, `OttieExpress/Extensions/Color+DesignTokens.swift`, `OttieExpress/Extensions/Font+DesignTokens.swift`
- Imported by: `Chapter6View.swift` (E9.4)
- State reads: None
- State writes: None

---

## Out of Scope

- Drag-to-connect drawing mechanic and star connection interaction (covered by E9.2)
- Memory card presentation, tap-to-dismiss behavior, and sequential constellation unlock logic (covered by E9.3)
- Sky illuminate sequence, final message typewriter, intro card, and chapter completion wiring (covered by E9.4)
- Audio playback of `bgm-constellation` and all SFX (wired by E9.4 when Chapter6View orchestrates the full flow)
- Haptic feedback for constellation lock and sky illuminate (consumed by E9.2 and E9.4 via HapticService)
- Constellation dot position design and otter silhouette shape authoring (asset/design prerequisite resolved before Phase 9 entry)
- Star fade-in animation timing polish and per-device frame rate profiling (deferred to Phase 12 integration testing on physical device)

---

## Definition of Done

- [ ] `night-base` (#0D0D1A) background fills the full screen when StarFieldView appears
- [ ] Stars fade in one-by-one over 2 seconds after StarFieldView mounts, using staggered opacity transitions
- [ ] Intro line "Some things are written in the stars." drifts in using New York Regular 18pt in `parchment` (#FAF3E0) color
- [ ] First constellation dots begin pulsing gently 1.5 seconds after star field entry completes
- [ ] ConstellationData defines exactly 6 constellations, each with a `starPositions` array of CGPoint values, a `connectionSequence` array of indices, a `displayName` string, and a `memoryCardText` string
- [ ] Memory card text for constellations 1-5 matches §12 verbatim: three lines per constellation, no em dashes
- [ ] Constellation 6 dot positions form a recognizable otter silhouette at rendering scale, with `displayName` set to "Carolina."
- [ ] Only the current active constellation's dots are visible and pulse; all future constellation dots remain invisible
- [ ] StarFieldView resumes the star fade-in sequence from the current progress point when the app returns to foreground after backgrounding mid-animation
- [ ] Chapter6Constants contains named constants for all timing values (2s star fade-in duration, 1.5s intro line delay, pulse animation period), sizing values (star dot radius, constellation line width), and layout parameters with zero magic numbers

---

## Implementation Notes

Design doc §12 Star Field Entry specifies stars appear "one by one, slow fade-in, over 2 seconds." Implement as staggered `withAnimation` blocks or a timer-driven opacity array where each star receives a delay offset of `2.0 / totalStarCount` seconds.

The intro line uses New York Regular 18pt per §12. Apply `parchment` (#FAF3E0) foreground. The "drift in" effect is a vertical offset transition (from +20pt to 0pt) combined with opacity fade, using the baseline spring (stiffness 280, damping 22) from §2 Motion Principles.

ConstellationData stores star positions as normalized CGPoint values in a 0...1 coordinate space relative to the star field bounds. This allows ConstellationDrawingView (E9.2) to map positions to the actual view frame using `GeometryReader`. The `connectionSequence` is an ordered array of indices into `starPositions`, defining the valid drawing order.

The pulse animation for active constellation dots uses a repeating opacity oscillation between 0.4 and 1.0. Per §2 Motion Principles: "Spring animations preferred over linear." Use `Animation.easeInOut(duration: 1.2).repeatForever(autoreverses: true)` for the breathing effect.

Constellation 6 stores an empty string for `displayName` in the data model. The "Carolina." label is handled by the sequencer (E9.3) as a special-case typewriter effect, not as a standard name display.

Per §3 Global Rules: no em dashes in any text surface. All memory card text is copied verbatim from §12. Verify each string during implementation.
