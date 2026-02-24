# E5.4: Build HUD and Game State Machine

**Phase:** 5 - Chapter 1: The Runner
**Class:** Feature
**Design Doc Reference:** §7 (Lives, HUD, Game Over, Game Complete), §2 (Typography, Color Palette), §15 (Haptic Map: game complete)
**Dependencies:**

- Phase 2: Navigation Shell and Shared UI (exit criteria met)
- E1.1: Define design system tokens (color, typography, and spacing constants available)
- E1.3: Build AudioService (`playSFX` interface available for `sfx-game-over` and `sfx-game-complete`)
- E1.4: Build HapticService (success notification haptic callable for game complete)
- E5.1: Scaffold SpriteKit scene and parallax world (`RunnerScene` delegate protocol for game events, `Chapter1Constants` with HUD sizing and game text)
- E5.2: Build Ottie sprite and jump controls (`OttieSprite.applyHit()` for obstacle response)
- E5.3: Build collectibles, obstacles, and spawning (heart collection and obstacle hit events reported via RunnerScene delegate)
- Asset: `ottie-hit` (available in asset catalog, reused from E5.2)
- Asset: `ottie-celebrate` (available in asset catalog)
- Asset: `sfx-game-over` (available in bundle)
- Asset: `sfx-game-complete` (available in bundle)

---

## Goal

Implement the in-game HUD with life hearts and heart counter, the game state machine tracking lives and collected hearts, the game over overlay with retry, and the game complete celebration screen.

---

## Scope

### File Inventory

| File                                                    | Action | Responsibility                                                                                                                                                                     |
| ------------------------------------------------------- | ------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `OttieExpress/Chapters/Chapter1/GameHUD.swift`          | Create | SwiftUI overlay rendering life heart icons in the top left and heart counter in the top center, positioned above the SpriteView game scene                                         |
| `OttieExpress/Chapters/Chapter1/GameState.swift`        | Create | Observable game state model tracking remaining lives, collected heart count, and current game phase (playing, gameOver, gameComplete) as the single source of truth for game logic |
| `OttieExpress/Chapters/Chapter1/GameOverView.swift`     | Create | SwiftUI overlay displaying dark backdrop, "Game Over" title, `ottie-hit` sprite, and "Try Again" button that resets the game                                                       |
| `OttieExpress/Chapters/Chapter1/GameCompleteView.swift` | Create | SwiftUI overlay displaying `ottie-celebrate` with bounce animation, "You made it." fade-in title, and game complete celebration sequence                                           |

### Integration Points

**GameHUD.swift**

- Imports from: `GameState.swift`, `Chapter1Constants.swift` (E5.1), `Color+DesignTokens.swift` (E1.1), `Font+DesignTokens.swift` (E1.1)
- Imported by: `Chapter1View.swift` (E5.5)
- State reads: `GameState.lives`, `GameState.heartsCollected`
- State writes: None

**GameState.swift**

- Imports from: `Chapter1Constants.swift` (E5.1)
- Imported by: `GameHUD.swift`, `GameOverView.swift`, `GameCompleteView.swift`, `Chapter1View.swift` (E5.5)
- State reads: None (it is the source of truth)
- State writes: Self-contained; mutated by the SwiftUI host in response to RunnerScene delegate events

**GameOverView.swift**

- Imports from: `GameState.swift`, `Chapter1Constants.swift` (E5.1), `Color+DesignTokens.swift` (E1.1), `Font+DesignTokens.swift` (E1.1)
- Imported by: `Chapter1View.swift` (E5.5)
- State reads: `GameState.phase`
- State writes: `GameState` (reset via `GameState.restart()`)

**GameCompleteView.swift**

- Imports from: `GameState.swift`, `Chapter1Constants.swift` (E5.1), `Color+DesignTokens.swift` (E1.1), `Font+DesignTokens.swift` (E1.1)
- Imported by: `Chapter1View.swift` (E5.5)
- State reads: `GameState.phase`
- State writes: None (reports completion via callback to E5.5 flow coordinator)

---

## Out of Scope

- SpriteKit scene rendering, parallax scrolling, and world setup (covered by E5.1)
- Ottie sprite animation, jump mechanics, and hit knockback (covered by E5.2)
- Heart and obstacle spawning, collision detection, and difficulty scaling (covered by E5.3)
- Intro card, letter screen, BGM playback, and chapter completion wiring (covered by E5.5)
- AudioService or HapticService method additions beyond existing interfaces (services owned by E1.3 and E1.4; this epic consumes existing APIs)
- HUD animation polish such as heart icon bounce on collection or counter pulse (deferred to Phase 11 cross-cutting hardening)

---

## Definition of Done

- [ ] Top-left HUD displays 3 heart icons representing remaining lives, with filled hearts for active lives and empty/dimmed hearts for lost lives
- [ ] Top-center HUD displays heart counter in "X / 25" format where X is the current `heartsCollected` value
- [ ] HUD text renders in SF Pro Text Semibold 15pt, `soft-white` (#F9F9F9), with a slight drop shadow for legibility over the scrolling background
- [ ] `GameState.collectHeart()` increments `heartsCollected` by 1 and transitions to `.gameComplete` phase when count reaches 25
- [ ] `GameState.loseLife()` decrements `lives` by 1 and transitions to `.gameOver` phase when lives reach 0
- [ ] Game over overlay renders with a dark semi-transparent backdrop over the frozen game scene, "Game Over" in SF Pro Display Bold 28pt `soft-white`, `ottie-hit` sprite centered, and a full-width "Try Again" button in `rose-blush`
- [ ] `sfx-game-over` plays via AudioService when the game over overlay appears
- [ ] "Try Again" tap calls `GameState.restart()` which resets `heartsCollected` to 0, `lives` to 3, and `phase` to `.playing`, and signals the RunnerScene to reset the game world
- [ ] Game complete state stops the world scrolling, renders `ottie-celebrate` bouncing in the center of the screen, plays `sfx-game-complete` via AudioService, and fades in "You made it." in SF Pro Display Bold 32pt `soft-white`
- [ ] Success notification haptic fires via HapticService when the game complete state triggers
- [ ] `GameState` exposes a `phase` property typed as an enum with cases `.playing`, `.gameOver`, `.gameComplete`, used by the SwiftUI host to conditionally render overlays

---

## Implementation Notes

`GameState` is an `@Observable` class annotated `@MainActor`:

```swift
@MainActor
@Observable
final class GameState {
    private(set) var lives: Int = Chapter1Constants.startingLives
    private(set) var heartsCollected: Int = 0
    private(set) var phase: GamePhase = .playing

    enum GamePhase {
        case playing
        case gameOver
        case gameComplete
    }

    func collectHeart() { ... }
    func loseLife() { ... }
    func restart() { ... }
}
```

The SwiftUI host (E5.5's `Chapter1View`) owns `@State private var gameState = GameState()`. The RunnerScene delegate callbacks invoke `gameState.collectHeart()` on heart events and `gameState.loseLife()` on hit events. The host observes `gameState.phase` to conditionally overlay `GameOverView` or `GameCompleteView` above the `SpriteView`.

`GameHUD` is a SwiftUI view overlaid on the SpriteView using a `ZStack`. It uses `.allowsHitTesting(false)` so taps pass through to the SpriteView beneath for jump input. Life hearts render as SF Symbol `heart.fill` (active) and `heart` (lost) in `warm-gold`. The counter uses a monospaced digit font variant to prevent layout jitter when the number changes.

`GameOverView` and `GameCompleteView` are conditional overlays controlled by `gameState.phase`. Both capture user input (blocking pass-through to the scene). `GameOverView` renders the "Try Again" button; on tap, it calls `gameState.restart()` and signals the RunnerScene (via a binding or callback) to reset all spawned nodes and restart the parallax. `GameCompleteView` fires `sfx-game-complete` and the success haptic on appearance, displays the `ottie-celebrate` asset with a repeating vertical bounce animation (`offset` oscillation), and fades in the text after a brief delay. It exposes an `onComplete` callback that the flow coordinator (E5.5) uses to transition to the letter screen after the `Chapter1Constants.gameCompleteTransitionDelay` (1.5s).
