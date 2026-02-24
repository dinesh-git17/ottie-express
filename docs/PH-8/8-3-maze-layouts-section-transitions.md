# E8.3: Build Maze Layouts and Section Transitions

**Phase:** 8 - Chapter 5: The Maze
**Class:** Feature
**Design Doc Reference:** SS11 (Maze Design, Maze Constraints, Section Structure: Sections 1-3, Section Transitions), SS15 (Audio Management: BGM crossfade)
**Dependencies:**

- Phase 2: Navigation Shell and Shared UI (exit criteria met)
- E8.1: Scaffold SpriteKit maze scene and tileset rendering (MazeScene, MazeTileset, Chapter5Constants available)
- E8.2: Build Ottie top-down sprite and swipe controls (MazeOttieSprite grid movement and SwipeController wall grid injection available)
- E1.3: Build AudioService (BGM crossfade at 0.8s and SFX playback available)
- Asset: `sfx-maze-transition` M4A (available in bundle)
- Asset: `bgm-maze-gray` M4A (available in bundle)
- Asset: `bgm-maze-amber` M4A (available in bundle)
- Asset: `bgm-maze-gold` M4A (available in bundle)

---

## Goal

Define the three maze section layouts with wall/path labeling, and implement section transitions with camera fade-to-black, tileset swap, and BGM crossfade.

---

## Scope

### File Inventory

| File                                                     | Action | Responsibility                                                                                                                                                       |
| -------------------------------------------------------- | ------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `OttieExpress/Chapters/Chapter5/MazeLayout.swift`        | Create | Grid-based maze representation with wall/path cell types, label text, and start/exit positions per section                                                           |
| `OttieExpress/Chapters/Chapter5/MazeSectionData.swift`   | Create | Static data definitions for all three section layouts including grid dimensions, wall positions, path positions, labels, dead-end count, and tileset/BGM identifiers |
| `OttieExpress/Chapters/Chapter5/SectionTransition.swift` | Create | Camera fade-to-black animation, tileset swap orchestration, BGM crossfade trigger, sprite repositioning, and wall grid reload for section changes                    |

### Integration Points

**MazeLayout.swift**

- Imports from: `Chapter5Constants.swift` (E8.1)
- Imported by: `MazeSectionData.swift`, `SwipeController.swift` (E8.2, consumes the wall grid), `MazeScene.swift` (E8.1, renders labeled nodes)
- State reads: None
- State writes: None

**MazeSectionData.swift**

- Imports from: `MazeLayout.swift`, `Chapter5Constants.swift` (E8.1)
- Imported by: `SectionTransition.swift`, `Chapter5View.swift` (E8.5, accesses section sequence)
- State reads: None
- State writes: None

**SectionTransition.swift**

- Imports from: `MazeSectionData.swift`, `MazeLayout.swift`, `Chapter5Constants.swift` (E8.1), `MazeTileset.swift` (E8.1), `MazeOttieSprite.swift` (E8.2), `SwipeController.swift` (E8.2)
- Imported by: `MazeScene.swift` (E8.1, called when Ottie reaches a section exit cell)
- State reads: None
- State writes: None

---

## Out of Scope

- Maze scene creation, camera system, and tileset rendering engine (covered by E8.1)
- Ottie sprite movement, walk animation, and swipe input handling (covered by E8.2)
- Reunion cutscene sequence and Carolina otter node (covered by E8.4)
- Chapter 5 intro card and flow coordinator wiring (covered by E8.5)
- Procedural maze generation algorithms (SS11 states "Exact maze topology is an implementation detail"; layouts are hand-authored static data)
- AudioService or HapticService interface modifications (consumed as-is from E1.3 and E1.4)
- BGM interruption handling and resume-after-interruption logic (deferred to Phase 11 cross-cutting hardening)
- Wall and path label localization or dynamic text (single locale, English only per design doc non-goals)

---

## Definition of Done

- [ ] Section 1 layout defines an 8x12 grid with exactly 2-3 dead ends and a solvable path from start to exit
- [ ] Section 1 wall labels render at designated wall positions: "Distance Lag", "WiFi From 2007", "Overthinking Loop"
- [ ] Section 1 path labels render at designated path positions: "Good Morning Calls", "Soft Reassurance Zone", "Inside Joke Territory"
- [ ] Section 2 layout defines a 10x14 grid with exactly 3-4 dead ends and a solvable path from start to exit
- [ ] Section 2 wall labels render: "Sleep Mode Activated", "Schedule Boss Fight", "Crash Out Detour"
- [ ] Section 2 path labels render: "Future Plans Lane", "Walk and Talk Moments", "Laughing For No Reason Path"
- [ ] Section 3 layout defines a 6x10 grid with exactly 0-1 dead ends and a solvable path from start to exit
- [ ] Section 3 wall labels render: "Miss You Wave", "Soft Hours Vulnerability Zone"
- [ ] Section 3 path labels render: "Princess Treatment Route", "Always Us Corridor"
- [ ] Section 3 places `carolina-otter-wait` node visible from the section entry position per SS11
- [ ] Section 3 final stretch contains an open path with no wall obstacles leading to Carolina's position
- [ ] Transition from Section 1 to Section 2 executes a 0.5s camera fade-to-black via `SKAction`
- [ ] During fade-to-black, tileset swaps from gray to amber and `SwipeController` receives the new wall grid
- [ ] `AudioService.crossfade(to: "bgm-maze-amber", duration: 0.8)` fires at section 1-to-2 boundary
- [ ] `sfx-maze-transition` plays at section 1-to-2 boundary
- [ ] Transition from Section 2 to Section 3 executes the same fade/swap/crossfade sequence with gold tileset and `bgm-maze-gold`
- [ ] Ottie sprite repositions to the new section's start cell after tileset swap completes
- [ ] Swipe input disables during transition and re-enables after Ottie repositions

---

## Implementation Notes

SS11 specifies exact grid dimensions and dead-end counts per section. Layouts are static, hand-authored 2D grids. Each cell is one of: wall, path, start, exit, or labeled (wall-labeled, path-labeled). Use an enum for cell types.

Wall and path labels render as `SKLabelNode` children of the respective tile nodes. Use SF Pro Text Regular at a size that fits within a single cell width (approximately 8pt given the 40pt cell size from E8.1 constants). Label color follows the section's emotional palette: `muted-gray` for Section 1 labels, `amber-warm` for Section 2, `warm-gold` for Section 3, per SS2 color tokens.

Section transition sequence (orchestrated by `SectionTransition`):

1. Overlay black `SKSpriteNode` at camera position, alpha 0 to 1 over 0.25s
2. At full black: swap tileset via `MazeTileset`, load new `MazeLayout`, reposition `MazeOttieSprite` to new start cell, inject new wall grid into `SwipeController`
3. `AudioService.crossfade(to:duration:)` with 0.8s per SS15
4. `AudioService.playSFX("sfx-maze-transition")`
5. Fade overlay alpha 1 to 0 over 0.25s (total black duration 0.5s including both fades)
6. `SwipeController.enableInput()`

SS11 states Carolina's otter is "visible in the distance from section entry" in Section 3. Place the `carolina-otter-wait` `SKSpriteNode` at the exit area of the Section 3 grid. Camera scale in Section 3 must allow the exit area to be partially visible from the start cell. The otter node is rendered by this epic as a static sprite; the reunion behavior is wired by E8.4.

Section solve times (45-60s, 60-90s, 30-45s) are design targets, not enforced constraints. Layout complexity and path length should approximate these ranges given one-swipe-per-step input at a natural pace.
