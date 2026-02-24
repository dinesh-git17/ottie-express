# E8.5: Wire Chapter 5 Flow Coordinator

**Phase:** 8 - Chapter 5: The Maze
**Class:** Feature
**Design Doc Reference:** SS11 (Flow, Screen: Chapter 5 Intro Card), SS4 (State Management, Resume Semantics), SS14 (Navigation and Progression), SS2 (Typography, Color Palette, Spacing)
**Dependencies:**

- Phase 2: Navigation Shell and Shared UI (exit criteria met; ChapterRouter, ChapterCard, and TypewriterText available)
- E8.1: Scaffold SpriteKit maze scene and tileset rendering (MazeScene available for SpriteView embedding)
- E8.2: Build Ottie top-down sprite and swipe controls (controllable Ottie available in scene)
- E8.3: Build maze layouts and section transitions (all three sections navigable with transitions)
- E8.4: Build reunion cutscene and particle effects (cutscene completion callback available)
- E1.2: Implement AppState and persistence (`completeChapter(_:)` available)
- E1.3: Build AudioService (BGM playback for `bgm-maze-gray` initial track)
- Service: AppState interface (`completeChapter(5)`, `currentChapter`, `completedChapters`)
- Asset: `ottie-topdown` PNG (available in asset catalog, used in intro card)

---

## Goal

Implement the Chapter 5 intro card and the flow coordinator connecting intro, three maze sections, reunion cutscene, and chapter completion.

---

## Scope

### File Inventory

| File                                                     | Action | Responsibility                                                                                                                                      |
| -------------------------------------------------------- | ------ | --------------------------------------------------------------------------------------------------------------------------------------------------- |
| `OttieExpress/Chapters/Chapter5/Chapter5View.swift`      | Create | SwiftUI host view embedding SpriteView, managing chapter flow state machine (intro, section 1-3, reunion, complete), and wiring AppState completion |
| `OttieExpress/Chapters/Chapter5/Chapter5IntroCard.swift` | Create | Intro card screen with chapter label, title, body text, otter sprite, and "Find the way" action button                                              |

### Integration Points

**Chapter5View.swift**

- Imports from: `Chapter5IntroCard.swift`, `MazeScene.swift` (E8.1), `MazeSectionData.swift` (E8.3), `Chapter5Constants.swift` (E8.1)
- Imported by: `ChapterRouter.swift` (Phase 2, routes to this view when `currentChapter == 5`)
- State reads: `AppState.currentChapter`, `AppState.completedChapters`
- State writes: `AppState.completeChapter(5)` via the reunion cutscene completion callback

**Chapter5IntroCard.swift**

- Imports from: `Chapter5Constants.swift` (E8.1)
- Imported by: `Chapter5View.swift`
- State reads: None
- State writes: None

---

## Out of Scope

- MazeScene internals, tileset rendering, and camera system (covered by E8.1)
- Ottie sprite, swipe controls, and movement logic (covered by E8.2)
- Maze layout definitions, section data, and transition sequences (covered by E8.3)
- Reunion cutscene choreography, particle effects, and camera pullback (covered by E8.4)
- ChapterRouter modifications or chapter transition animation (covered by Phase 2 E2.1; Chapter5View is consumed by the existing router)
- AudioService or HapticService interface changes (consumed as-is)
- Chapter 5 state sub-persistence (SS4 resume semantics: within-chapter sub-state is not persisted; maze restarts from Section 1 on relaunch)
- Cross-chapter state validation or corruption recovery (deferred to Phase 11 hardening)
- Chapter 1 game state reset on resume (referenced in PHASES.md E8.5 acceptance criteria as "per SS4"; this is Chapter 1's concern, not Chapter 5's)

---

## Definition of Done

- [ ] `Chapter5IntroCard` renders "Chapter 5" label in SF Pro Display Semibold 24pt
- [ ] `Chapter5IntroCard` renders "The Way Home" as the title in SF Pro Display Bold 32pt
- [ ] `Chapter5IntroCard` renders body text "The path isn't always clear. But it always leads somewhere worth it." in SF Pro Text Regular 16pt
- [ ] `Chapter5IntroCard` displays `ottie-topdown` asset centered below the body text
- [ ] "Find the way" button renders and triggers transition from intro to maze gameplay on tap
- [ ] `Chapter5View` transitions from intro card to SpriteView embedding `MazeScene` configured for Section 1
- [ ] `bgm-maze-gray` begins playing via AudioService when Section 1 loads
- [ ] `MazeScene` progresses through Section 1, Section 2, Section 3 in order via section transitions (E8.3)
- [ ] Reunion cutscene completion callback calls `appState.completeChapter(5)`
- [ ] `AppState.completedChapters` contains 5 after chapter completion
- [ ] `ChapterRouter` advances to Chapter 6 after `completeChapter(5)` fires
- [ ] Relaunching mid-chapter restarts Chapter 5 from the intro card (within-chapter sub-state is not persisted per SS4)

---

## Implementation Notes

SS11 defines the flow as: `Chapter 5 Intro Card -> Section 1 (Gray) -> Section 2 (Amber) -> Section 3 (Gold) -> Reunion Cutscene -> Chapter 6`. The `Chapter5View` manages this as a state machine with an enum: `.intro`, `.playing`, `.complete`. The `.playing` state embeds `SpriteView(scene: mazeScene)` where `mazeScene` handles section progression internally via E8.3's `SectionTransition`. The `.complete` state is transient: the reunion completion callback writes to AppState, and the ChapterRouter handles navigation to Chapter 6.

The intro card follows the `ChapterCard` pattern established in E2.3 (Phase 2). However, Chapter 5's intro card includes a custom `ottie-topdown` sprite image not present in the generic `ChapterCard` component. Build `Chapter5IntroCard` as a standalone view consuming design tokens directly rather than extending `ChapterCard`.

Intro card layout per SS11 and SS2:

- "The Way Home": SF Pro Display Bold 32pt, `parchment` (#FAF3E0)
- Body text: SF Pro Text Regular 16pt, `soft-white` (#F9F9F9)
- `ottie-topdown` image: centered, rendered at asset intrinsic size (small per SS11)
- "Find the way" button: `HapticButton` from Phase 2 SharedUI, `warm-gold` text
- Background: `night-base` (#0D0D1A)
- Standard horizontal padding: 24pt per SS2

SS4 resume semantics: "On relaunch mid-chapter: user resumes at the beginning of the current (incomplete) chapter." Chapter 5 does not persist maze progress. If `currentChapter == 5` and 5 is not in `completedChapters`, show the intro card. The maze scene is created fresh each time.

BGM initialization: when `Chapter5View` transitions from `.intro` to `.playing`, call `AudioService.playBGM("bgm-maze-gray")`. Subsequent BGM changes are handled by E8.3's section transitions.

The `SpriteView` embedding uses `SpriteView(scene:options:)`. Pass `.allowsTransparency` if the scene background should be transparent; otherwise set the scene `backgroundColor` to the section's base color. Per SS11, Section 1 uses "muted blues, cool grays" which is the tileset responsibility (E8.1). Set scene background to `.clear` and let the tileset nodes provide the visual ground.
