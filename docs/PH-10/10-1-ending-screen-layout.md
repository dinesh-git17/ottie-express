# E10.1: Build Ending Screen Layout and Ottie Animation

**Phase:** 10 - Ending Screen
**Class:** Feature
**Design Doc Reference:** §2 Design System (Color Palette, Typography, Motion Principles), §5 Asset Inventory (ottie-ending), §13 Ending Screen
**Dependencies:**

- Phase 2: Navigation Shell and Shared UI (exit criteria met)
- Phase 9: Chapter 6 - The Constellations (exit criteria met)
- Asset: `ottie-ending` (available in asset catalog, two wave frames for 2-frame loop)
- Service: ChapterRouter routes to EndingScreenView when currentChapter exceeds Chapter 6

---

## Goal

Implement the terminal ending screen with centered Valentine's message, signature line, and PhaseAnimator-driven waving Ottie sprite against the night-base background.

---

## Scope

### File Inventory

| File                                               | Action | Responsibility                                                                                    |
| -------------------------------------------------- | ------ | ------------------------------------------------------------------------------------------------- |
| `OttieExpress/EndingScreen/EndingScreenView.swift` | Create | Ending screen layout with text stack, Ottie wave animation, and night-base background             |
| `OttieExpress/EndingScreen/EndingConstants.swift`  | Create | Named constants for typography sizes, spacing, sprite dimensions, animation timing, opacity value |

### Integration Points

**EndingScreenView.swift**

- Imports from: `EndingConstants.swift`
- Imported by: `OttieExpress/App/ChapterRouter.swift` (routes to this view after Chapter 6 completion)
- State reads: None
- State writes: None

**EndingConstants.swift**

- Imports from: None
- Imported by: `EndingScreenView.swift`, `ConstellationBackdrop.swift` (E10.2)
- State reads: None
- State writes: None

---

## Out of Scope

- Constellation backdrop rendering at 0.15 opacity (delivered by E10.2: ConstellationBackdrop)
- BGM crossfade from `bgm-constellation` to `bgm-ending` (delivered by E10.2: audio crossfade wiring)
- ChapterRouter routing case for the ending screen after Chapter 6 completion (covered by Phase 2 ChapterRouter, which routes based on `AppState.currentChapter`)
- Haptic feedback on the ending screen (§13 defines zero interactive elements; no haptic triggers exist)
- `bg-night-sky` textured background layering (E10.2 determines whether the textured asset or flat color token is the final background surface)

---

## Definition of Done

- [ ] EndingScreenView renders `night-base` (#0D0D1A) as the full-screen background on appearance
- [ ] "Happy Valentine's Day, Carolina." displays in New York Regular 28pt with `parchment` (#FAF3E0) color, horizontally centered
- [ ] "With love, Dinesh" displays in New York Italic 20pt with `muted-gray` (#8A8A9A) color, positioned below the primary message with vertical spacing
- [ ] `ottie-ending` sprite renders at approximately 80pt in the bottom-right corner of the screen
- [ ] Ottie 2-frame wave animation loops continuously via PhaseAnimator with gentle timing
- [ ] Screen contains zero buttons, zero continue prompts, zero tap targets, and zero navigation elements
- [ ] EndingConstants defines named constants for font sizes (28pt, 20pt), color token references, sprite dimension (80pt), animation timing, and layout spacing
- [ ] Ottie animation respects `AccessibilityReduceMotion`: static display when reduced motion is enabled

---

## Implementation Notes

- §13: "No buttons. No continue. This is the end. She can stay here as long as she wants." EndingScreenView is a terminal view. It reads no AppState, writes no AppState, and provides no navigation affordances. The screen exists solely as a resting place.
- §4 Architecture: "PhaseAnimator handles looping state-driven animations (fingerprint pulse, star dot idle, waving Ottie)." Use `PhaseAnimator` with two phases alternating between wave frame states. The animation timing should feel gentle and unhurried, matching the emotional register of the ending.
- §2 Typography: New York Regular 28pt for the primary message, New York Italic 20pt for the signature. Use `Font.system(size:design:)` with `.serif` design to access the New York font family. Verify the italic variant resolves correctly on iOS 18 via `.italic()` modifier or explicit font name registration.
- §5 Asset Inventory: `ottie-ending` is listed as a single asset entry ("Ottie small, waving gently"). The 2-frame wave loop per §13 requires two frame images in the asset catalog. Other multi-frame sprites in the inventory use numbered suffixes (e.g., `ottie-run-01` through `ottie-run-04`). If only one image is provided, implement the wave via alternating rotation or offset transform on the single sprite to produce the 2-frame visual.
- §2 Motion Principles: "Respect `prefers-reduced-motion` accessibility setting throughout." Query `@Environment(\.accessibilityReduceMotion)` and render Ottie as a static image when the preference is active.
- §5 Asset Inventory: `bg-night-sky` ("Deep navy star-filled sky") is listed for "Chapter 6, Ending" usage. §13 specifies "Background: `night-base`" which is the color token (#0D0D1A). E10.1 uses the flat color token as the background. E10.2 determines whether the textured `bg-night-sky` asset overlays or replaces the flat color.
- §3 Global Rules: No em dashes in any text surface. Both message strings defined in §13 are em-dash-free. Maintain this constraint in EndingConstants string definitions.
