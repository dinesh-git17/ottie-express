# E8.2: Build Ottie Top-Down Sprite and Swipe Controls

**Phase:** 8 - Chapter 5: The Maze
**Class:** Feature
**Design Doc Reference:** SS11 (Controls, Ottie Sprite), SS15 (Haptic Map: wall bump, footstep)
**Dependencies:**

- Phase 2: Navigation Shell and Shared UI (exit criteria met)
- E8.1: Scaffold SpriteKit maze scene and tileset rendering (MazeScene and Chapter5Constants available)
- E1.4: Build HapticService (gentle haptic for wall bump available via typed enum interface)
- E1.3: Build AudioService (SFX playback for `sfx-wall-bump` and `sfx-footstep-soft` available)
- Asset: `ottie-topdown` PNG (available in asset catalog)
- Asset: `ottie-topdown-walk-u` 2-frame PNG (available in asset catalog)
- Asset: `ottie-topdown-walk-d` 2-frame PNG (available in asset catalog)
- Asset: `ottie-topdown-walk-l` 2-frame PNG (available in asset catalog)
- Asset: `ottie-topdown-walk-r` 2-frame PNG (available in asset catalog)
- Asset: `sfx-wall-bump` M4A (available in bundle)
- Asset: `sfx-footstep-soft` M4A, 2 variants (available in bundle)

---

## Goal

Implement the controllable top-down Ottie sprite with 4-directional walk animation, swipe-based grid movement, wall collision response, and movement audio/haptic feedback.

---

## Scope

### File Inventory

| File                                                   | Action | Responsibility                                                                                                                         |
| ------------------------------------------------------ | ------ | -------------------------------------------------------------------------------------------------------------------------------------- |
| `OttieExpress/Chapters/Chapter5/MazeOttieSprite.swift` | Create | SKSpriteNode subclass with 4-directional walk animation, grid-snapped positioning, and idle/walk state                                 |
| `OttieExpress/Chapters/Chapter5/SwipeController.swift` | Create | UISwipeGestureRecognizer coordination for 4-direction input, one-step-per-swipe enforcement, and movement validation against wall data |

### Integration Points

**MazeOttieSprite.swift**

- Imports from: `Chapter5Constants.swift` (E8.1)
- Imported by: `MazeScene.swift` (E8.1, sprite added to world node), `ReunionCutscene.swift` (E8.4, disables controls and triggers auto-walk)
- State reads: None
- State writes: None (position updates are internal to the sprite node)

**SwipeController.swift**

- Imports from: `Chapter5Constants.swift` (E8.1), `MazeOttieSprite.swift`
- Imported by: `MazeScene.swift` (E8.1, gesture recognizers attached to the SKView)
- State reads: None (reads maze layout grid passed via initializer to validate wall collisions)
- State writes: None

---

## Out of Scope

- Maze layout data structures and wall position definitions (covered by E8.3)
- Section transition triggers when Ottie reaches an exit cell (covered by E8.3)
- Reunion cutscene auto-walk and control disable sequence (covered by E8.4; E8.2 provides the `disableInput()` and `autoWalk(to:)` methods on the sprite)
- BGM playback, crossfade, or section-boundary audio (covered by E8.3)
- Carolina otter sprite rendering or proximity detection (covered by E8.4)
- Camera follow implementation (covered by E8.1; E8.2 updates sprite position, camera reads it)
- Maze grid rendering and tileset swapping (covered by E8.1)
- Haptic pattern calibration and interruption resilience (deferred to Phase 11 hardening)
- `sfx-footstep-soft` variant selection logic beyond simple alternation (no additional complexity specified in design doc)

---

## Definition of Done

- [ ] `MazeOttieSprite` renders the `ottie-topdown` idle texture at the grid cell size defined in `Chapter5Constants`
- [ ] Swipe up triggers upward movement of exactly one grid cell and plays `ottie-topdown-walk-u` 2-frame animation
- [ ] Swipe down triggers downward movement of exactly one grid cell and plays `ottie-topdown-walk-d` 2-frame animation
- [ ] Swipe left triggers leftward movement of exactly one grid cell and plays `ottie-topdown-walk-l` 2-frame animation
- [ ] Swipe right triggers rightward movement of exactly one grid cell and plays `ottie-topdown-walk-r` 2-frame animation
- [ ] Sprite returns to directional idle frame after walk animation completes
- [ ] Swipe toward a wall cell plays `sfx-wall-bump` via AudioService, fires gentle haptic via HapticService, and Ottie remains in the current cell
- [ ] `sfx-footstep-soft` plays on each valid movement step, alternating between the two variants
- [ ] Swipe input is ignored while a movement animation is in progress (one step per swipe, no queuing)
- [ ] `SwipeController` validates the target cell against a wall grid (2D boolean array) passed at initialization
- [ ] `disableInput()` method on `SwipeController` prevents all swipe processing (called by reunion cutscene)
- [ ] `autoWalk(to:completion:)` method on `MazeOttieSprite` moves the sprite along a sequence of grid positions with walk animation, calling the completion handler on arrival

---

## Implementation Notes

SS11 specifies swipe-based navigation with one step per swipe in four directions. Attach four `UISwipeGestureRecognizer` instances (one per direction) to the `SKView` hosting the scene. SpriteKit's `SKView` is a `UIView` subclass and supports standard gesture recognizers.

Movement animation uses `SKAction.move(to:duration:)` for the position change and `SKAction.animate(with:timePerFrame:)` for the walk texture cycle, grouped via `SKAction.group`. Movement duration is a named constant; use 0.15s per cell step (responsive for swipe-based input). Walk animation `timePerFrame` matches half the movement duration so both frames display during the step.

Wall collision check: `SwipeController` holds a reference to the current section's wall grid (a 2D array of booleans where `true` indicates a wall). Before initiating movement, compute the target grid coordinate and check the wall grid. If the target is a wall or out of bounds, trigger the bump response instead.

The `autoWalk(to:completion:)` method is public API consumed by E8.4's reunion cutscene. It accepts an array of grid positions and sequentially animates through them using `SKAction.sequence` with the same walk animation per step. The completion closure fires after the final step.

SS15 haptic map specifies no explicit maze-footstep haptic; `sfx-footstep-soft` is audio-only feedback. Wall bump uses `UIImpactFeedbackGenerator` at light intensity (design doc says "gentle haptic"). Use `HapticService.fire(.light)` for wall contact.
