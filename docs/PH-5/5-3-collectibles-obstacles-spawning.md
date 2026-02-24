# E5.3: Build Collectibles, Obstacles, and Spawning

**Phase:** 5 - Chapter 1: The Runner
**Class:** Feature
**Design Doc Reference:** §7 (Hearts, Obstacles, Difficulty Tier Table), §15 (Haptic Map: heart collected, obstacle hit)
**Dependencies:**

- Phase 2: Navigation Shell and Shared UI (exit criteria met)
- E1.1: Define design system tokens (color constants available for collectible tinting)
- E1.3: Build AudioService (`playSFX` interface available for `sfx-heart-collect` and `sfx-obstacle-hit`)
- E1.4: Build HapticService (light impact for heart collection, medium impact for obstacle hit)
- E5.1: Scaffold SpriteKit scene and parallax world (`RunnerScene` with update loop, `Chapter1Constants` with difficulty tiers and ground height, physics category bitmasks)
- E5.2: Build Ottie sprite and jump controls (`OttieSprite` with physics body and category bitmask for collision detection)
- Asset: `heart-collectible` (available in asset catalog)
- Asset: `obstacle-stormcloud` (available in asset catalog)
- Asset: `obstacle-thorn` (available in asset catalog)
- Asset: `sfx-heart-collect` (available in bundle)
- Asset: `sfx-obstacle-hit` (available in bundle)

---

## Goal

Implement heart collectible nodes, storm cloud and thorn obstacle nodes, SpriteKit physics collision detection, and the progressive spawn manager that scales difficulty across five tiers based on hearts collected.

---

## Scope

### File Inventory

| File                                                    | Action | Responsibility                                                                                                                                                                                |
| ------------------------------------------------------- | ------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `OttieExpress/Chapters/Chapter1/HeartCollectible.swift` | Create | SKSpriteNode subclass for heart pickup nodes with physics body, category bitmask, and self-removal on collection                                                                              |
| `OttieExpress/Chapters/Chapter1/ObstacleNode.swift`     | Create | SKSpriteNode subclass for storm cloud and thorn obstacle nodes with physics body, category bitmask, and type-based spawn positioning (varying height for clouds, ground-only for thorns)      |
| `OttieExpress/Chapters/Chapter1/SpawnManager.swift`     | Create | Spawn timer controller that creates hearts and obstacles at intervals scaled by the current difficulty tier, adjusts world speed multiplier on tier transitions, and removes off-screen nodes |

### Integration Points

**HeartCollectible.swift**

- Imports from: `Chapter1Constants.swift` (E5.1)
- Imported by: `SpawnManager.swift`, `RunnerScene.swift` (E5.1 contact delegate identifies heart nodes)
- State reads: None
- State writes: None
- Public interface: `HeartCollectible()` initializer; physics body with `heart` category bitmask

**ObstacleNode.swift**

- Imports from: `Chapter1Constants.swift` (E5.1)
- Imported by: `SpawnManager.swift`, `RunnerScene.swift` (E5.1 contact delegate identifies obstacle nodes)
- State reads: None
- State writes: None
- Public interface: `ObstacleNode(type: ObstacleType)` where `ObstacleType` is `.stormcloud` or `.thorn`; physics body with `obstacle` category bitmask

**SpawnManager.swift**

- Imports from: `HeartCollectible.swift`, `ObstacleNode.swift`, `Chapter1Constants.swift` (E5.1), `ParallaxLayer.swift` (E5.1 to adjust speed multiplier)
- Imported by: `RunnerScene.swift` (E5.1 owns SpawnManager as a child coordinator)
- State reads: Current heart count (received via update call from scene)
- State writes: `ParallaxLayer.currentSpeedMultiplier` (adjusts on tier transitions)

---

## Out of Scope

- Ottie sprite rendering, run cycle, jump mechanics, and knockback animation (covered by E5.2; this epic triggers `OttieSprite.applyHit()` via collision but does not own the animation)
- HUD heart counter display and life tracking (covered by E5.4; this epic reports collection and hit events via the RunnerScene delegate)
- Game over and game complete state transitions (covered by E5.4; this epic reports events, E5.4 owns the state machine)
- BGM playback during gameplay (covered by E5.5 flow coordinator)
- AudioService or HapticService interface additions beyond existing `playSFX` and impact patterns (services owned by E1.3 and E1.4; this epic consumes existing interfaces)
- Spawn rate tuning, balance adjustments, or difficulty curve modifications beyond the §7 tier table values (deferred to Phase 12 playtesting)

---

## Definition of Done

- [ ] `heart-collectible` nodes spawn on the right edge of the screen at varying heights between the ground plane and 80% of screen height
- [ ] Heart nodes scroll left at the current world speed and are removed when they pass the left screen edge
- [ ] Ottie overlap with a heart node removes the heart from the scene and reports a collection event to the RunnerScene delegate
- [ ] `sfx-heart-collect` plays via AudioService on each heart collection
- [ ] Light impact haptic fires via HapticService on each heart collection
- [ ] `obstacle-stormcloud` nodes spawn at varying heights including mid-air positions above the ground plane
- [ ] `obstacle-thorn` nodes spawn from ground level only, anchored to the ground plane
- [ ] Ottie contact with an obstacle node triggers `OttieSprite.applyHit()` and reports a hit event to the RunnerScene delegate
- [ ] `sfx-obstacle-hit` plays via AudioService and medium impact haptic fires via HapticService on each obstacle hit
- [ ] SpawnManager scales speed multiplier and spawn interval across 5 difficulty tiers matching §7: 1.0x/Low at 0-5 hearts, 1.2x/Low-Medium at 6-10, 1.4x/Medium at 11-15, 1.6x/Medium-High at 16-20, 1.8x/High at 21-25
- [ ] Collision detection uses SpriteKit `SKPhysicsContactDelegate` with distinct category bitmasks for ottie, heart, and obstacle, configured on `RunnerScene`
- [ ] Off-screen nodes (hearts and obstacles that scroll past the left edge) are removed from the scene to prevent unbounded node accumulation

---

## Implementation Notes

`RunnerScene` adopts `SKPhysicsContactDelegate` and implements `didBegin(_ contact:)`. The contact handler identifies the collision pair by checking category bitmasks:

- `ottie & heart`: Remove the heart node. Report `.heartCollected` via the scene delegate to the SwiftUI host.
- `ottie & obstacle`: Call `ottieSprite.applyHit()`. Report `.obstacleHit` via the scene delegate. Apply a brief invincibility window (0.5s) to prevent multiple hits from the same obstacle cluster.

AudioService and HapticService calls happen in the SwiftUI host's delegate callback handler, not inside the SpriteKit scene. The scene reports events; the host triggers side effects. This maintains the separation between the SpriteKit render loop and UIKit/SwiftUI service singletons.

`SpawnManager` runs two independent spawn timers: one for hearts, one for obstacles. Each timer uses `SKAction.repeatForever(SKAction.sequence([.wait(forDuration: interval), .run { spawn() }]))`. The interval is derived from the base interval divided by the current tier's speed multiplier. When the heart count crosses a tier boundary, `SpawnManager.updateDifficulty(heartsCollected:)` recalculates intervals and restarts the timer actions.

The speed multiplier from the difficulty tier applies to both spawn scroll speed and `ParallaxLayer.currentSpeedMultiplier`. When `SpawnManager` transitions tiers, it updates all three parallax layers' speed multipliers simultaneously to maintain visual coherence between world scroll and object scroll.

`ObstacleNode(type:)` uses the type enum to determine spawn position:

- `.thorn`: Y fixed at `groundHeight` (sits on the ground)

Both types scroll left via `SKAction.moveTo(x: -offScreenBuffer, duration:)` calculated from current speed. On completion, the node removes itself from the scene.

Heart spawn heights use a random Y between `groundHeight + heartMinHeight` and `screenHeight * 0.7`, ensuring hearts are reachable via a single jump from the ground.
