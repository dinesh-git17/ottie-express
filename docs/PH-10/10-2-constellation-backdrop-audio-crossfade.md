# E10.2: Build Constellation Backdrop and Audio Crossfade

**Phase:** 10 - Ending Screen
**Class:** Feature
**Design Doc Reference:** §12 Chapter 6 (constellation shape data), §13 Ending Screen (backdrop and audio), §15 Haptics and Sound Design (Audio Management, crossfade rules)
**Dependencies:**

- Phase 2: Navigation Shell and Shared UI (exit criteria met)
- Phase 9: Chapter 6 - The Constellations (exit criteria met)
- E9.1: Build star field and constellation data model (ConstellationData with star positions and connection sequences for all 6 constellations)
- E10.1: Build ending screen layout and Ottie animation (EndingScreenView.swift and EndingConstants.swift available)
- Asset: `bgm-ending` (available in asset catalog)
- Service: AudioService crossfade API (public interface available from Phase 1)

---

## Goal

Render all 6 completed constellations as a faint static backdrop at 0.15 opacity and wire the BGM crossfade from constellation music to ending music via AudioService on screen appearance.

---

## Scope

### File Inventory

| File                                                    | Action | Responsibility                                                                                            |
| ------------------------------------------------------- | ------ | --------------------------------------------------------------------------------------------------------- |
| `OttieExpress/EndingScreen/ConstellationBackdrop.swift` | Create | Renders all 6 constellation shapes (dots and connecting lines) at 0.15 opacity as background layer        |
| `OttieExpress/EndingScreen/EndingScreenView.swift`      | Modify | Add ConstellationBackdrop as ZStack background layer and trigger AudioService BGM crossfade on appearance |

Note: The PHASES.md entry lists only `ConstellationBackdrop.swift` for this epic. The `EndingScreenView.swift` modification is a detected integration requirement. The stated acceptance criteria ("all 6 constellation shapes rendered against night-base" and "bgm-constellation fades out as bgm-ending fades in") cannot be delivered without compositing the backdrop into EndingScreenView and triggering the crossfade from its lifecycle.

### Integration Points

**ConstellationBackdrop.swift**

- Imports from: `OttieExpress/Chapters/Chapter6/ConstellationData.swift` (star positions, connection sequences), `EndingConstants.swift` (constellation opacity value)
- Imported by: `EndingScreenView.swift`
- State reads: None (reads ConstellationData as a static data source, not AppState)
- State writes: None

**EndingScreenView.swift** (Modify)

- Imports from: `EndingConstants.swift` (existing), `ConstellationBackdrop.swift` (added), `OttieExpress/Services/AudioService.swift` (added for crossfade call)
- Imported by: `OttieExpress/App/ChapterRouter.swift` (unchanged)
- State reads: None (unchanged)
- State writes: None (unchanged)
- Modification scope: add `ConstellationBackdrop()` as the first layer in the ZStack behind text and Ottie; add `.task {}` modifier to invoke AudioService BGM crossfade from `bgm-constellation` to `bgm-ending`

---

## Out of Scope

- Constellation drawing interaction, tap targets, or gesture recognizers on the backdrop (constellations on the ending screen are static decorative elements, not interactive per §13)
- Memory card display, constellation name labels, or typewriter name reveals (Chapter 6 interaction elements from §12, not carried to the ending screen)
- AudioService implementation or API modification (this epic consumes the existing crossfade interface from Phase 1; AudioService internals are not touched)
- Audio interruption handling during ending BGM playback (covered by E11.1: AudioService session lifecycle hardening)
- Ending screen text layout, Ottie wave animation, or navigation element removal (delivered by E10.1)
- Constellation opacity calibration or crossfade duration tuning on physical hardware (deferred to E11.1/E11.2 cross-cutting hardening on iPhone 17 Pro)

---

## Definition of Done

- [ ] ConstellationBackdrop renders all 6 constellation shapes using star positions and connection sequences from ConstellationData
- [ ] Constellation dots and connecting lines display at 0.15 opacity against the `night-base` background
- [ ] ConstellationBackdrop composites as a ZStack layer behind the text stack and Ottie sprite in EndingScreenView
- [ ] `bgm-constellation` fades out over 0.8s via AudioService crossfade when the ending screen appears
- [ ] `bgm-ending` fades in over 0.8s simultaneously with the `bgm-constellation` fade-out
- [ ] `bgm-ending` continues playback without abrupt cutoff after crossfade completion
- [ ] ConstellationBackdrop is non-interactive with `allowsHitTesting(false)` to prevent gesture interference with the terminal screen

---

## Implementation Notes

- §13: "Background: `night-base` with all 6 constellations still faintly visible, very low opacity (0.15)." ConstellationBackdrop reads the same `ConstellationData` model created in E9.1. It iterates all 6 constellations and renders each one's dot positions and connection lines as static completed shapes. No animation, no pulsing, no name labels.
- §15 Audio Management: "Only one BGM track playing at any time. Crossfade on BGM transitions (0.8s)." The crossfade from `bgm-constellation` to `bgm-ending` uses the existing AudioService crossfade method with a 0.8s duration. Trigger via `.task {}` on EndingScreenView to ensure the crossfade fires exactly once on initial appearance and cancels cleanly if the view disappears.
- §5 Asset Inventory: `bgm-ending` is described as "Quiet warm ambient, lets her sit in the moment" (M4A format). Configure AudioService to loop this track via `numberOfLoops = -1` or equivalent, preventing abrupt silence on the terminal screen.
- ConstellationData (E9.1) defines star positions and connection sequences for all 6 constellations. ConstellationBackdrop uses these positions to draw static constellation shapes. Each constellation renders as dots at their defined positions connected by line segments matching the connection order. Apply `.opacity(0.15)` to the entire backdrop view or to individual shape layers.
- §5 Asset Inventory: `bg-night-sky` ("Deep navy star-filled sky, rich textured background") is listed for "Chapter 6, Ending." If Chapter 6 uses this asset as the visual sky background, ConstellationBackdrop should render the constellation shapes on top of a matching visual layer. Coordinate with E10.1's `night-base` flat color to determine whether the `bg-night-sky` image replaces the flat token as the background surface in EndingScreenView.
- The `EndingScreenView.swift` modification adds two elements: (1) `ConstellationBackdrop()` as the lowest ZStack layer, rendering behind text and Ottie; (2) a `.task {}` modifier calling the AudioService crossfade. Both additions are minimal and do not restructure the existing layout from E10.1.
