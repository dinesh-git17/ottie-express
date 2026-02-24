# E5.1: Scaffold SpriteKit Scene and Parallax World

**Phase:** 5 - Chapter 1: The Runner
**Class:** Feature
**Design Doc Reference:** §7 (Game Engine, World), §2 (Motion Principles), §4 (Architecture Overview, Technology Stack)
**Dependencies:**

- Phase 2: Navigation Shell and Shared UI (exit criteria met)
- E1.1: Define design system tokens (color and motion constants available)
- Asset: `bg-forest-sky` (available in asset catalog)
- Asset: `bg-forest-mid` (available in asset catalog)
- Asset: `bg-forest-floor` (available in asset catalog)

---

## Goal

Build the SpriteKit scene embedded via SpriteView with a three-layer parallax scroll system, seamless ground tiling, and the game world coordinate system with ground height at the bottom 20% of the screen.

---

## Scope

### File Inventory

| File                                                     | Action | Responsibility                                                                                                                                                                      |
| -------------------------------------------------------- | ------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `OttieExpress/Chapters/Chapter1/RunnerScene.swift`       | Create | SKScene subclass embedded via SpriteView, owns the update loop, manages parallax layer children, defines the game world coordinate system and ground plane                          |
| `OttieExpress/Chapters/Chapter1/ParallaxLayer.swift`     | Create | Reusable parallax tiling node that clones and repositions texture sprites to produce seamless infinite horizontal scrolling at a configurable speed                                 |
| `OttieExpress/Chapters/Chapter1/Chapter1Constants.swift` | Create | Named constants for all Chapter 1 screens: scroll speeds, ground height ratio, sprite dimensions, difficulty tiers, timing values, HUD parameters, letter content, and text strings |

### Integration Points

**RunnerScene.swift**

- Imports from: `ParallaxLayer.swift`, `Chapter1Constants.swift`
- Imported by: `Chapter1View.swift` (E5.5)
- State reads: None (scene receives game commands via delegate or public methods)
- State writes: None (scene reports events via delegate callbacks)
- Public interface: `RunnerScene` SKScene subclass; a delegate protocol for reporting game events (heart collected, obstacle hit, game over, game complete) to the SwiftUI host

**ParallaxLayer.swift**

- Imports from: `Chapter1Constants.swift`
- Imported by: `RunnerScene.swift`
- State reads: None
- State writes: None
- Public interface: `ParallaxLayer(textureName: String, speed: CGFloat, zPosition: CGFloat)` with `update(deltaTime:)` method called from the scene's update loop

**Chapter1Constants.swift**

- Imports from: None
- Imported by: `RunnerScene.swift`, `ParallaxLayer.swift`, `OttieSprite.swift` (E5.2), `InputHandler.swift` (E5.2), `HeartCollectible.swift` (E5.3), `ObstacleNode.swift` (E5.3), `SpawnManager.swift` (E5.3), `GameHUD.swift` (E5.4), `GameState.swift` (E5.4), `GameOverView.swift` (E5.4), `GameCompleteView.swift` (E5.4), `LetterView.swift` (E5.5), `Chapter1View.swift` (E5.5), `Chapter1IntroCard.swift` (E5.5)
- State reads: None
- State writes: None

---

## Out of Scope

- Ottie player sprite, run cycle animation, and jump mechanics (covered by E5.2)
- Heart collectible nodes, obstacle nodes, and spawn management (covered by E5.3)
- HUD rendering, game state tracking, game over, and game complete screens (covered by E5.4)
- Intro card, letter screen, BGM playback, and chapter completion flow (covered by E5.5)
- Collision detection physics configuration and category bitmasks (covered by E5.3; this epic creates the scene but does not configure physics contact delegate)
- Performance profiling or frame rate optimization beyond basic scene setup (deferred to Phase 12 final validation)
- AudioService or HapticService integration within the scene (consumed by E5.3 and E5.4 via delegate callbacks to the SwiftUI host)

---

## Definition of Done

- [ ] `RunnerScene` subclass renders inside a `SpriteView` within a SwiftUI host view on a `night-base` background
- [ ] `bg-forest-sky` scrolls continuously from right to left at the slowest configured speed as the far parallax layer
- [ ] `bg-forest-mid` scrolls continuously from right to left at medium configured speed as the mid parallax layer
- [ ] `bg-forest-floor` scrolls continuously from right to left at the fastest configured speed as the ground parallax layer
- [ ] Ground floor tiles seamlessly with no visible seam or gap during continuous scrolling at any speed tier
- [ ] Ground height is fixed at the bottom 20% of the screen as defined by `Chapter1Constants.groundHeightRatio`
- [ ] `ParallaxLayer` repositions texture clones when they scroll off the leading (left) edge, producing an infinite scroll illusion with no texture pop-in
- [ ] All scroll speeds, ground height ratio, spawn region boundaries, and world dimensions reference named constants from `Chapter1Constants` with zero magic numbers

---

## Implementation Notes

`RunnerScene` sets `scaleMode = .resizeFill` and `backgroundColor` to the `night-base` color. The scene is presented via `SpriteView(scene:)` in the SwiftUI host. The SpriteKit Game Architecture skill (`.claude/skills/spritekit-game-architecture/`) governs all SKScene patterns in this chapter.

The three `ParallaxLayer` instances are added as children of the scene with ascending `zPosition` values: sky at 0, mid at 1, ground at 2. Player sprites (E5.2) render at zPosition 3+, HUD at the highest layer.

`ParallaxLayer` tiles the texture by creating enough `SKSpriteNode` clones to cover the screen width plus one extra tile. On each `update(deltaTime:)` call, all tile nodes shift left by `speed * deltaTime * currentSpeedMultiplier`. When the leading tile's trailing edge crosses the left screen boundary, it wraps to the right end of the tile chain. The `currentSpeedMultiplier` is a public property that E5.3's `SpawnManager` adjusts per difficulty tier.

The `update(_ currentTime:)` method in `RunnerScene` calculates `deltaTime` from the previous frame timestamp and propagates it to all parallax layers. Use `CACurrentMediaTime()` delta tracking, not fixed timestep, to maintain smooth scrolling at variable frame rates.

`Chapter1Constants` defines named constants consumed by all Chapter 1 epics:

| Constant                      | Value                                                                                                | Consumer         |
| ----------------------------- | ---------------------------------------------------------------------------------------------------- | ---------------- |
| `parallaxSpeedSky`            | 30.0 (points/sec)                                                                                    | E5.1             |
| `parallaxSpeedMid`            | 60.0 (points/sec)                                                                                    | E5.1             |
| `parallaxSpeedGround`         | 120.0 (points/sec)                                                                                   | E5.1             |
| `groundHeightRatio`           | 0.2                                                                                                  | E5.1, E5.2, E5.3 |
| `ottieScreenXRatio`           | 0.33 (left third)                                                                                    | E5.2             |
| `ottieRunFrameCount`          | 4                                                                                                    | E5.2             |
| `ottieRunFPS`                 | 12.0                                                                                                 | E5.2             |
| `heartWinCount`               | 25                                                                                                   | E5.3, E5.4       |
| `startingLives`               | 3                                                                                                    | E5.4             |
| `difficultyTiers`             | See tier table below                                                                                 | E5.3             |
| `hudFontSize`                 | 15.0                                                                                                 | E5.4             |
| `gameOverTitleFontSize`       | 28.0                                                                                                 | E5.4             |
| `gameCompleteTitleFontSize`   | 32.0                                                                                                 | E5.4             |
| `gameCompleteText`            | "You made it."                                                                                       | E5.4             |
| `gameOverTitle`               | "Game Over"                                                                                          | E5.4             |
| `gameCompleteTransitionDelay` | 1.5 (seconds)                                                                                        | E5.5             |
| `finalLinePause`              | 1.5 (seconds)                                                                                        | E5.5             |
| `introCardTitle`              | "The Runner"                                                                                         | E5.5             |
| `introCardBody`               | "Run through the magical forest. Collect 25 hearts. Dodge the storms and the thorns. Reach the end." | E5.5             |
| `finalLineText`               | "No checkpoints needed. You already have it forever."                                                | E5.5             |
| `letterText`                  | Full verbatim letter from §7                                                                         | E5.5             |

**Difficulty Tier Table (encoded as an array of structs in `Chapter1Constants`):**

| Hearts Collected | Speed Multiplier | Spawn Rate Label |
| ---------------- | ---------------- | ---------------- |
| 0-5              | 1.0              | Low              |
| 6-10             | 1.2              | Low-Medium       |
| 11-15            | 1.4              | Medium           |
| 16-20            | 1.6              | Medium-High      |
| 21-25            | 1.8              | High             |
