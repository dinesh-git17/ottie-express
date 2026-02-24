# E5.5: Build Letter Screen and Chapter Flow

**Phase:** 5 - Chapter 1: The Runner
**Class:** Feature
**Design Doc Reference:** §7 (Intro Card, The Letter, Final Line, Flow), §4 (State Management, Resume Semantics), §14 (Navigation and Progression, Chapter Completion), §2 (Typography, Color Palette), §15 (Audio Management)
**Dependencies:**

- Phase 2: Navigation Shell and Shared UI (exit criteria met)
- E1.1: Define design system tokens (color, typography, spacing, and motion constants available)
- E1.2: Implement AppState and persistence (`completeChapter(_:)` method and `completedChapters` set available)
- E1.3: Build AudioService (BGM playback and crossfade interface available for `bgm-game-runner`)
- E1.4: Build HapticService (success notification haptic callable for chapter complete)
- E2.1: Build navigation shell (ChapterRouter routes to Chapter1View when `currentChapter == 1`)
- E2.2: Build TypewriterText component (typewriter animation for letter text with configurable timing)
- E2.3: Build ChapterCard component (intro card rendering with chapter label, title, body, asset, button)
- E5.1: Scaffold SpriteKit scene and parallax world (`RunnerScene`, `Chapter1Constants`)
- E5.2: Build Ottie sprite and jump controls (OttieSprite integrated in RunnerScene)
- E5.3: Build collectibles, obstacles, and spawning (SpawnManager and collision handling integrated in RunnerScene)
- E5.4: Build HUD and game state machine (`GameState`, `GameHUD`, `GameOverView`, `GameCompleteView`)
- Asset: `ottie-game-intro` (available in asset catalog)
- Asset: `bgm-game-runner` (available in bundle)
- Asset: `sfx-chapter-transition` (available in bundle)

---

## Goal

Wire the intro card, SpriteKit game, game state overlays, post-game letter screen with typewriter text, fade-in final line, and chapter completion into a sequential Chapter 1 flow that marks chapter completion in AppState and integrates with ChapterRouter for forward navigation.

---

## Scope

### File Inventory

| File | Action | Responsibility |
| --- | --- | --- |
| `OttieExpress/Chapters/Chapter1/LetterView.swift` | Create | Post-game letter screen with `parchment` background, full §7 letter text via TypewriterText, fade-in final line in New York Italic 20pt, and "Continue" prompt |
| `OttieExpress/Chapters/Chapter1/Chapter1View.swift` | Create | Flow coordinator managing sequential phase transitions between intro card, game, letter, and chapter completion, with BGM lifecycle and RunnerScene delegate handling |
| `OttieExpress/Chapters/Chapter1/Chapter1IntroCard.swift` | Create | Chapter 1 intro card using ChapterCard component with "The Runner" title, §7 body text, `ottie-game-intro` asset, and "Play" button |

### Integration Points

**LetterView.swift**

- Imports from: `Chapter1Constants.swift` (E5.1), `TypewriterText.swift` (E2.2), `Color+DesignTokens.swift` (E1.1), `Font+DesignTokens.swift` (E1.1)
- Imported by: `Chapter1View.swift`
- State reads: None
- State writes: None
- Public interface: `LetterView(onComplete: @escaping () -> Void)` where the callback fires when the user taps "Continue"

**Chapter1View.swift**

- Imports from: `Chapter1IntroCard.swift`, `RunnerScene.swift` (E5.1), `GameState.swift` (E5.4), `GameHUD.swift` (E5.4), `GameOverView.swift` (E5.4), `GameCompleteView.swift` (E5.4), `LetterView.swift`, `Chapter1Constants.swift` (E5.1), `AudioService.swift` (E1.3), `HapticService.swift` (E1.4)
- Imported by: `ChapterRouter.swift` (E2.1)
- State reads: `AppState.completedChapters` (to verify chapter gating on relaunch)
- State writes: `AppState.completedChapters`, `AppState.currentChapter` (via `appState.completeChapter(1)`)

**Chapter1IntroCard.swift**

- Imports from: `ChapterCard.swift` (E2.3), `Chapter1Constants.swift` (E5.1)
- Imported by: `Chapter1View.swift`
- State reads: None
- State writes: None
- Public interface: `Chapter1IntroCard(onPlay: @escaping () -> Void)`

---

## Out of Scope

- SpriteKit scene rendering, parallax scrolling, and world setup (covered by E5.1)
- Ottie sprite animation, run cycle, jump mechanics, and hit knockback (covered by E5.2)
- Heart and obstacle spawning, collision detection, and difficulty scaling (covered by E5.3)
- HUD rendering, game state tracking, game over overlay, and game complete overlay (covered by E5.4)
- Cross-chapter transition animation from Chapter 1 to Chapter 2 (handled by ChapterRouter in E2.1; this epic triggers the AppState mutation, the router handles the visual transition)
- AudioService interface additions or BGM crossfade duration tuning beyond the existing 0.8s default (AudioService owned by E1.3)
- Within-chapter sub-state persistence across app relaunch (§4 resume semantics specify incomplete chapters reset to the intro card on relaunch; game progress is not saved mid-run)

---

## Definition of Done

- [ ] Intro card renders with "Chapter 1" label, "The Runner" title, body text "Run through the magical forest. Collect 25 hearts. Dodge the storms and the thorns. Reach the end.", `ottie-game-intro` asset, and "Play" button using the ChapterCard component (E2.3)
- [ ] Tapping "Play" transitions from the intro card to the game view with a crossfade and upward drift animation (0.4s per §2 screen transition)
- [ ] `bgm-game-runner` begins playing via AudioService when the game phase starts
- [ ] RunnerScene delegate events are handled by Chapter1View: heart collection calls `gameState.collectHeart()`, obstacle hit calls `gameState.loseLife()`
- [ ] GameHUD overlay displays above the SpriteView during the `.playing` phase
- [ ] GameOverView overlay displays when `gameState.phase` transitions to `.gameOver`, and "Try Again" resets the game
- [ ] GameCompleteView overlay displays when `gameState.phase` transitions to `.gameComplete`, and its `onComplete` callback fires after the 1.5-second celebration delay
- [ ] BGM stops or crossfades out when transitioning from the game to the letter screen
- [ ] Letter screen renders on a `parchment` (#FAF3E0) full-screen background with the complete §7 letter text in New York Regular 18pt, `night-base` (#0D0D1A), line height 1.6, via TypewriterText with 40ms per character, 200ms punctuation pause, 400ms line break pause, and 800ms paragraph break pause
- [ ] After typewriter completion and a 1.5-second pause, the final line fades in (no typewriter) centered in New York Italic 20pt: "No checkpoints needed. You already have it forever."
- [ ] "Continue" prompt renders in SF Pro Text Regular 15pt, `muted-gray` (#8A8A9A), after the final line fully appears
- [ ] Tapping "Continue" calls `appState.completeChapter(1)`, inserting `1` into `completedChapters` and advancing `currentChapter`
- [ ] ChapterRouter observes the `currentChapter` change and navigates away from Chapter1View to the next chapter view
- [ ] Relaunching the app after completing Chapter 1 routes directly to the current incomplete chapter, bypassing Chapter1View entirely

---

## Implementation Notes

Chapter1View manages a private `@State` enum representing the active phase of the flow:

```swift
private enum Chapter1Phase {
    case intro
    case game
    case letter
}
```

The view body switches on this enum:

- `.intro`: Renders `Chapter1IntroCard(onPlay: { phase = .game })`.
- `.game`: Renders the game composition: `SpriteView(scene: runnerScene)` layered in a `ZStack` with `GameHUD` overlay and conditional `GameOverView` / `GameCompleteView` overlays driven by `gameState.phase`. Starts `bgm-game-runner` via AudioService on appearance. The RunnerScene delegate maps events to `GameState` mutations. `GameCompleteView.onComplete` transitions to `.letter` after the 1.5-second delay.
- `.letter`: Stops BGM via AudioService (crossfade to silence over 0.8s). Renders `LetterView(onComplete: { appState.completeChapter(1) })`.

Access `AppState` via `@Environment(AppState.self) private var appState` per the injection pattern in §4.

`Chapter1View` owns `@State private var gameState = GameState()` and `@State private var runnerScene = RunnerScene()`. The `RunnerScene` communicates with the SwiftUI host via a delegate protocol:

```swift
protocol RunnerSceneDelegate: AnyObject {
    func didCollectHeart()
    func didHitObstacle()
}
```

The delegate is set on scene creation. Chapter1View conforms to the delegate (or uses a coordinator class) and routes events to `gameState.collectHeart()` and `gameState.loseLife()` respectively. AudioService and HapticService side effects (SFX, haptics) are triggered from the delegate handler, not inside the SpriteKit scene.

`LetterView` displays the full letter text from `Chapter1Constants.letterText` using TypewriterText (E2.2). The letter text is a multi-paragraph string stored verbatim in the constants file. TypewriterText handles paragraph break detection (double newline) and applies the 800ms pause per E2.2 acceptance criteria. After the typewriter's `onComplete` callback fires, a 1.5-second `Task.sleep` elapses (`Chapter1Constants.finalLinePause`), then the final line fades in via `withAnimation(.easeIn(duration: 0.6))`. The "Continue" prompt appears after the fade-in completes.

`Chapter1IntroCard` is a thin wrapper that passes `Chapter1Constants.introCardTitle`, `Chapter1Constants.introCardBody`, `"ottie-game-intro"`, `"Play"`, and the `onPlay` callback to `ChapterCard` (E2.3).

The skip-on-relaunch behavior requires no code in this view. ChapterRouter reads `appState.currentChapter` on launch. If `currentChapter != 1`, ChapterRouter never instantiates Chapter1View. §4 resume semantics confirm that within-chapter sub-state is not persisted: if the app terminates mid-game, the next launch starts Chapter 1 from the intro card. This is the default behavior because the `Chapter1Phase` state is local `@State`, which resets on view re-creation.
