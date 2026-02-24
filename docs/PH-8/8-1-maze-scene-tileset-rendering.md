# E8.1: Scaffold SpriteKit Maze Scene and Tileset Rendering

**Phase:** 8 - Chapter 5: The Maze
**Class:** Feature
**Design Doc Reference:** SS11 (Maze Design, Section Structure), SS4 (Architecture Overview, SpriteKit via SpriteView), SS2 (Color Palette)
**Dependencies:**

- Phase 2: Navigation Shell and Shared UI (exit criteria met)
- E1.1: Define design system tokens (color tokens `bg-maze-gray`, `bg-maze-amber`, `bg-maze-gold` referenced)
- Asset: `bg-maze-gray` tileset PNG (available in asset catalog)
- Asset: `bg-maze-amber` tileset PNG (available in asset catalog)
- Asset: `bg-maze-gold` tileset PNG (available in asset catalog)

---

## Goal

Build the SpriteKit scene embedded via SpriteView that renders top-down maze tilesets across three visual themes with a player-tracking camera system.

---

## Scope

### File Inventory

| File                                                     | Action | Responsibility                                                                                        |
| -------------------------------------------------------- | ------ | ----------------------------------------------------------------------------------------------------- |
| `OttieExpress/Chapters/Chapter5/MazeScene.swift`         | Create | SKScene subclass hosting the maze world node, camera, and SpriteView integration                      |
| `OttieExpress/Chapters/Chapter5/MazeTileset.swift`       | Create | Tileset loading and tile node rendering for gray, amber, and gold visual themes                       |
| `OttieExpress/Chapters/Chapter5/Chapter5Constants.swift` | Create | Named constants for grid cell size, maze dimensions per section, camera parameters, and timing values |

### Integration Points

**MazeScene.swift**

- Imports from: `Chapter5Constants.swift`, `MazeTileset.swift`
- Imported by: `Chapter5View.swift` (E8.5, via SpriteView embedding)
- State reads: None (scene receives configuration via initializer, not AppState)
- State writes: None

**MazeTileset.swift**

- Imports from: `Chapter5Constants.swift`
- Imported by: `MazeScene.swift`
- State reads: None
- State writes: None

**Chapter5Constants.swift**

- Imports from: None
- Imported by: `MazeScene.swift`, `MazeTileset.swift`, `MazeOttieSprite.swift` (E8.2), `MazeLayout.swift` (E8.3), `SwipeController.swift` (E8.2), `ReunionCutscene.swift` (E8.4), `Chapter5View.swift` (E8.5), `Chapter5IntroCard.swift` (E8.5)
- State reads: None
- State writes: None

---

## Out of Scope

- Ottie sprite rendering, animation, or movement logic (covered by E8.2)
- Maze layout data, wall/path definitions, and dead-end generation (covered by E8.3)
- Section transition logic including fade-to-black and BGM crossfade (covered by E8.3)
- Reunion cutscene, particle effects, and camera pullback animation (covered by E8.4)
- SpriteView SwiftUI host, intro card, and flow coordinator wiring (covered by E8.5)
- AudioService modifications or BGM playback integration (consumed from E1.3 interface, wired in E8.3)
- HapticService calls for wall bumps or footstep feedback (consumed from E1.4 interface, wired in E8.2)
- Performance profiling and 60fps validation on device (deferred to Phase 12 final validation)
- `prefers-reduced-motion` adaptations for camera follow animation (deferred to Phase 11 hardening)

---

## Definition of Done

- [ ] `MazeScene` subclasses `SKScene` and renders inside a `SpriteView` in a SwiftUI preview
- [ ] Gray tileset loads from `bg-maze-gray` asset and renders a grid of tile nodes filling an 8x12 cell area
- [ ] Amber tileset loads from `bg-maze-amber` asset and renders a grid of tile nodes filling a 10x14 cell area
- [ ] Gold tileset loads from `bg-maze-gold` asset and renders a grid of tile nodes filling a 6x10 cell area
- [ ] `MazeScene` contains an `SKCameraNode` that tracks a configurable target position with smooth follow using `SKAction` interpolation
- [ ] Camera follow updates position each frame via `update(_:)` without snapping
- [ ] Grid cell size, all three maze dimension pairs (8x12, 10x14, 6x10), camera follow speed, and camera scale are defined as named constants in `Chapter5Constants.swift`
- [ ] `MazeTileset` exposes a public method that accepts a theme enum (gray, amber, gold) and grid dimensions, returning an array of positioned `SKSpriteNode` tiles
- [ ] Tileset swap replaces all tile nodes in the scene world node when called with a different theme
- [ ] Scene `size` and `scaleMode` configure correctly for SpriteView embedding without letterboxing or stretching

---

## Implementation Notes

The design doc SS4 specifies SpriteKit embedded in SwiftUI via `SpriteView` for Chapter 5. Use `SpriteView(scene:)` with a configured `MazeScene` instance. The scene owns a world node (`SKNode`) as the root container for all maze content; the camera tracks the world, not individual nodes.

Camera follow uses linear interpolation between current camera position and the target position each frame in `update(_:)`. The interpolation factor is a named constant in `Chapter5Constants`. SS11 does not specify camera follow speed; use 0.15 as the lerp factor (smooth but responsive, standard for tile-based games).

Grid cell size is not specified in the design doc. Derive from the rendering constraint: the maze must be navigable on iPhone 17 Pro (393pt logical width). For an 8x12 grid (smallest section), cells at 40pt give 320pt width with 73pt total horizontal margin, fitting comfortably. Define `cellSize` as 40pt in constants. Camera scale adjusts per section to keep 5-7 cells visible vertically.

Tileset enum cases map directly to the three asset names: `bg-maze-gray`, `bg-maze-amber`, `bg-maze-gold`. The design doc SS11 specifies distinct color language per section: muted blues/cool grays for gray, warm ambers/orange tones for amber, full warm gold for gold. The tilesets encode this visually; no programmatic color transformation is needed.

`scaleMode` should be `.resizeFill` to match the SpriteView container size without bars. Scene `anchorPoint` at (0.5, 0.5) centers the coordinate system on the camera.
