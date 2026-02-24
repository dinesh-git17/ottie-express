# E8.4: Build Reunion Cutscene and Particle Effects

**Phase:** 8 - Chapter 5: The Maze
**Class:** Feature
**Design Doc Reference:** SS11 (Reunion Cutscene), SS2 (Typography: New York Italic 20pt, Color: warm-gold), SS15 (Audio Management)
**Dependencies:**

- Phase 2: Navigation Shell and Shared UI (exit criteria met)
- E8.1: Scaffold SpriteKit maze scene and tileset rendering (MazeScene world node and camera available)
- E8.2: Build Ottie top-down sprite and swipe controls (`disableInput()` and `autoWalk(to:completion:)` available on sprite/controller)
- E8.3: Build maze layouts and section transitions (Section 3 layout with Carolina otter position defined)
- E1.3: Build AudioService (BGM fade and SFX playback available)
- Asset: `ottie-hug` PNG (available in asset catalog)
- Asset: `carolina-otter-hug` PNG (available in asset catalog)
- Asset: `carolina-otter-wait` PNG (available in asset catalog)
- Asset: `sfx-reunion-fanfare` M4A (available in bundle)

---

## Goal

Implement the reunion cutscene triggered on reaching Carolina's otter: auto-walk, sprite swap to hug frames, heart burst particle effect, camera pullback revealing the full maze, and the closing line.

---

## Scope

### File Inventory

| File                                                        | Action | Responsibility                                                                                                                                |
| ----------------------------------------------------------- | ------ | --------------------------------------------------------------------------------------------------------------------------------------------- |
| `OttieExpress/Chapters/Chapter5/ReunionCutscene.swift`      | Create | Cutscene sequence coordinator: disable controls, auto-walk, sprite swap, particle trigger, camera pullback, text fade-in, and continue prompt |
| `OttieExpress/Chapters/Chapter5/HeartParticleEmitter.swift` | Create | SKEmitterNode configuration for 8-12 heart particles bursting outward in an arc from center                                                   |
| `OttieExpress/Chapters/Chapter5/CarolinaOtterNode.swift`    | Create | SKSpriteNode wrapper for Carolina otter with idle (`carolina-otter-wait`) and hug (`carolina-otter-hug`) texture states                       |

### Integration Points

**ReunionCutscene.swift**

- Imports from: `Chapter5Constants.swift` (E8.1), `MazeOttieSprite.swift` (E8.2), `SwipeController.swift` (E8.2), `HeartParticleEmitter.swift`, `CarolinaOtterNode.swift`
- Imported by: `MazeScene.swift` (E8.1, triggered when Ottie reaches Carolina's grid cell in Section 3)
- State reads: None
- State writes: None (chapter completion is handled by E8.5 flow coordinator via a completion callback)

**HeartParticleEmitter.swift**

- Imports from: `Chapter5Constants.swift` (E8.1)
- Imported by: `ReunionCutscene.swift`
- State reads: None
- State writes: None

**CarolinaOtterNode.swift**

- Imports from: `Chapter5Constants.swift` (E8.1)
- Imported by: `MazeScene.swift` (E8.1, added to world node in Section 3 via E8.3 layout), `ReunionCutscene.swift`
- State reads: None
- State writes: None

---

## Out of Scope

- Maze scene setup, tileset rendering, and camera follow system (covered by E8.1)
- Ottie movement, swipe input, and walk animation (covered by E8.2; this epic calls `disableInput()` and `autoWalk(to:completion:)`)
- Section 3 layout definition and Carolina otter initial placement (covered by E8.3; this epic consumes the position)
- Chapter completion persistence to AppState (covered by E8.5 flow coordinator)
- Chapter 5 intro card and end-to-end flow orchestration (covered by E8.5)
- Heart particle color tuning, size variants, or physics-based drift beyond the arc specification (SS11 specifies 8-12 hearts arcing outward; no additional particle physics described)
- BGM track selection or crossfade timing (this epic fades BGM to quiet; track management is E8.3)
- `prefers-reduced-motion` adaptations for camera pullback and particle burst (deferred to Phase 11)

---

## Definition of Done

- [ ] Cutscene triggers when Ottie occupies the same grid cell as Carolina's otter in Section 3
- [ ] `SwipeController.disableInput()` fires immediately on cutscene trigger, preventing all swipe processing
- [ ] Ottie auto-walks the remaining steps to Carolina's exact position using `autoWalk(to:completion:)`
- [ ] `MazeOttieSprite` texture swaps to `ottie-hug` after auto-walk completes
- [ ] `CarolinaOtterNode` texture swaps from `carolina-otter-wait` to `carolina-otter-hug` simultaneously with Ottie's swap
- [ ] `HeartParticleEmitter` emits 8-12 heart-shaped particles bursting outward in an arc from the center point between the two sprites
- [ ] Camera executes a scale transform on the world node over 1.5s, pulling back to reveal the full maze
- [ ] `AudioService` fades current BGM to volume 0 during the camera pullback
- [ ] `sfx-reunion-fanfare` plays via AudioService at the start of the camera pullback
- [ ] "Worth every detour." renders as an `SKLabelNode` in New York Italic 20pt, `warm-gold` (#F5C842), centered on screen, fading in from alpha 0 to 1 after camera pullback completes
- [ ] Text holds for 3 seconds after fade-in completes
- [ ] "Continue" prompt appears after the 3-second hold, centered below the closing line
- [ ] Tapping "Continue" fires a completion callback provided by the parent scene (consumed by E8.5 to trigger chapter completion)

---

## Implementation Notes

SS11 defines the reunion sequence as a strict linear chain: disable controls, auto-walk, sprite swap, heart burst, camera pullback, BGM fade, text fade-in, hold, continue prompt. Implement this as an `SKAction.sequence` on the scene, with grouped sub-actions where parallelism is specified.

Heart particle emitter: use `SKEmitterNode` configured programmatically (not from an .sks file, to keep the file inventory bounded). Set `particleBirthRate` to emit 8-12 particles in a single burst (high birth rate for 0.1s, then 0 birth rate). `emissionAngle` spans a 180-degree arc (upward semicircle). Use `particleTexture` from a heart shape rendered via `UIGraphicsImageRenderer` at runtime (a small filled heart path at 16x16pt in `rose-blush` #E8708A). `particleLifetime` of 1.5s with `particleAlpha` decaying to 0. `particleSpeed` of 80-120 (randomized range) for the outward burst.

Camera pullback: animate the `MazeScene`'s camera `xScale` and `yScale` from the current section scale to a value that fits the entire Section 3 grid (plus surrounding context) in the viewport. The target scale is computed from the full maze world bounds divided by the screen size. Use `SKAction.scale(to:duration:)` with 1.5s timing curve set to `.easeInEaseOut`.

The closing line uses `SKLabelNode` with `fontName` set to "NewYorkItalic-Regular" (the system serif italic available on iOS 17+). If the exact font name fails to resolve, fall back to "NewYork-RegularItalic". Set `fontSize` to 20, `fontColor` to `warm-gold` (#F5C842). Position at the camera's center point (screen center in scene coordinates).

The completion callback pattern: `ReunionCutscene` accepts an `onComplete: @Sendable () -> Void` closure at initialization. The "Continue" prompt is an `SKLabelNode` with an associated tap handler (detected via `touchesEnded` on the scene, hit-testing against the label node). On tap, the closure fires.
