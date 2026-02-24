# E9.2: Implement Constellation Drawing Interaction

**Phase:** 9 - Chapter 6: The Constellations
**Class:** Feature
**Design Doc Reference:** §12 Chapter 6 (Constellation Interaction drawing mechanic), §2 Design System (typography SF Pro Semibold 13pt, color `warm-gold`, spacing 44pt min tap target), §15 Haptics and Sound Design (constellation lock haptic)
**Dependencies:**

- Phase 2: Navigation Shell and Shared UI (exit criteria met)
- E9.1: Build star field and constellation data model (ConstellationData star positions and Chapter6Constants available)
- E1.1: Define design system tokens (color token `warm-gold`, typography SF Pro Semibold, motion constants)
- E1.3: Build AudioService (`playSFX` interface available for `sfx-star-connect` and `sfx-constellation-lock`)
- E1.4: Build HapticService (heavy impact haptic available for constellation lock per §15 haptic map)

---

## Goal

Implement the drag-to-connect constellation drawing mechanic with sequential star connection validation, glowing line segment rendering between connected dots, and constellation lock detection with name display.

---

## Scope

### File Inventory

| File                                                            | Action | Responsibility                                                                                                                               |
| --------------------------------------------------------------- | ------ | -------------------------------------------------------------------------------------------------------------------------------------------- |
| `OttieExpress/Chapters/Chapter6/ConstellationDrawingView.swift` | Create | Orchestrate drag-to-connect interaction, validate connection order against ConstellationData sequence, and fire completion callback on lock  |
| `OttieExpress/Chapters/Chapter6/StarDotView.swift`              | Create | Render individual star dots in idle (pulsing), active (drag target), and connected states using `star-dot-idle` and `star-dot-active` assets |
| `OttieExpress/Chapters/Chapter6/ConnectionLineView.swift`       | Create | Render glowing `constellation-line` segments between pairs of connected star dot positions                                                   |

### Integration Points

**ConstellationDrawingView.swift**

- Imports from: `ConstellationData.swift` (E9.1), `Chapter6Constants.swift` (E9.1), `StarDotView.swift`, `ConnectionLineView.swift`, `OttieExpress/Services/AudioService.swift` (E1.3), `OttieExpress/Services/HapticService.swift` (E1.4)
- Imported by: `ConstellationSequencer.swift` (E9.3), `Chapter6View.swift` (E9.4)
- State reads: Current constellation definition (passed as parameter from sequencer)
- State writes: None (exposes `onComplete` callback)

**StarDotView.swift**

- Imports from: `Chapter6Constants.swift` (E9.1), `OttieExpress/Extensions/Color+DesignTokens.swift`
- Imported by: `ConstellationDrawingView.swift`
- State reads: Dot display state (idle, active, connected) passed as parameter
- State writes: None

**ConnectionLineView.swift**

- Imports from: `Chapter6Constants.swift` (E9.1), `OttieExpress/Extensions/Color+DesignTokens.swift`
- Imported by: `ConstellationDrawingView.swift`
- State reads: Array of connected point pairs passed as parameter
- State writes: None

---

## Out of Scope

- Memory card presentation after constellation lock (covered by E9.3)
- Sequential constellation unlock logic and progression state machine (covered by E9.3)
- Constellation 6 special behavior: brighter trembling dots, "Carolina." typewriter on lock (covered by E9.3)
- Sky illuminate sequence, camera zoom-out, and final message (covered by E9.4)
- BGM playback coordination (wired by E9.4)
- Star field background rendering and intro line animation (covered by E9.1)
- Animation performance profiling on physical device (deferred to Phase 12 integration testing)

---

## Definition of Done

- [ ] Active star dots render with `star-dot-idle` asset and pulse with a repeating opacity animation when the constellation is the current active constellation
- [ ] Connected star dots transition from `star-dot-idle` to `star-dot-active` asset after successful connection
- [ ] Dragging from one dot to the next dot in the connection sequence renders a `constellation-line` (glowing line segment) between them
- [ ] `sfx-star-connect` plays via AudioService on each successful star-to-star connection
- [ ] Out-of-order connection attempts produce no visual change, no audio, and no haptic feedback
- [ ] On the final connection completing a constellation, the entire constellation (all lines and dots) flashes at full brightness for 0.3 seconds
- [ ] After the flash, the constellation name appears above the constellation shape in SF Pro Semibold 13pt, `warm-gold` (#F5C842)
- [ ] `sfx-constellation-lock` plays via AudioService on constellation lock
- [ ] Heavy impact haptic fires via HapticService on constellation lock per §15 haptic map
- [ ] ConstellationDrawingView exposes an `onComplete` closure that fires exactly once when the constellation is fully connected and the lock animation finishes
- [ ] Drag gesture hit detection uses a 44pt proximity threshold per §2 minimum tap target

---

## Implementation Notes

Design doc §12 Constellation Interaction: "User draws a line by dragging from one star dot to the next in sequence. Dots must be tapped or dragged through in order." ConstellationDrawingView maintains a `connectedCount` integer. On each `DragGesture.onChanged`, compute the distance from the current gesture location to the position of `constellation.starPositions[constellation.connectionSequence[connectedCount]]`. If distance is within 44pt (§2 minimum tap target), increment `connectedCount`, append the new connection to the rendered line array, play `sfx-star-connect`, and update the dot state.

The `constellation-line` glow effect: ConnectionLineView renders each segment as a `Path.addLine` stroke with a shadow modifier. Use `Color.warmGold` (design token) as the line color with `.shadow(color: .warmGold.opacity(0.6), radius: 4)` for the glow.

Constellation flash on lock: animate all dot and line opacities to 1.0 briefly (0.15s up, 0.15s down) using a spring animation. After the flash sequence completes, display the name label and fire the `onComplete` callback.

Haptic on constellation lock: §15 haptic map specifies "Constellation lock" as `UIImpactFeedbackGenerator` at Heavy intensity. Call `HapticService.fire(.constellationLock)` (or the typed enum equivalent defined in E1.4).

Star positions from ConstellationData are in normalized 0...1 coordinate space. ConstellationDrawingView uses `GeometryReader` to obtain the view bounds and multiplies each normalized position by the frame size to produce absolute CGPoint values for rendering and hit testing.
