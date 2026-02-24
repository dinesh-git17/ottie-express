# Implementation Roadmap — Ottie Express

**Source Document:** `docs/OTTIE_EXPRESS_DESIGN_DOC.md` (v1.2)
**Generated:** 2026-02-23
**Total Phases:** 13
**Total Epics:** 47

---

## Phase Summary

| Phase | Class          | Name                               | Epics | Depends On                    | Entry Gate                                                       | Exit Gate                                                                                    |
| ----- | -------------- | ---------------------------------- | ----- | ----------------------------- | ---------------------------------------------------------------- | -------------------------------------------------------------------------------------------- |
| 0     | Infrastructure | Project Scaffold                   | 1     | None                          | Xcode 16+ and iOS 18 SDK available                               | Project builds for iOS 18.0+, empty app launches on simulator                                |
| 1     | Infrastructure | Core Systems and Design Foundation | 5     | Phase 0                       | Phase 0 exit criteria met                                        | Tokens render, state persists, services compile with verified interfaces                     |
| 2     | Infrastructure | Navigation Shell and Shared UI     | 4     | Phase 1                       | Phase 1 exit criteria met                                        | Router displays chapters by state, shared components render and animate                      |
| 3     | Feature        | Chapter 0: The Handshake           | 3     | Phase 2                       | Phase 2 exit criteria met, Chapter 0 assets available            | Fingerprint scan, vault split, password validation, and chapter completion all functional    |
| 4     | Feature        | Chapter 4: The Dossier             | 3     | Phase 2                       | Phase 2 exit criteria met, Chapter 4 assets available            | Scene renders, all 10 evidence files display, closing line typewriters, chapter completes    |
| 5     | Feature        | Chapter 1: The Runner              | 5     | Phase 2                       | Phase 2 exit criteria met, Chapter 1 assets available            | Game plays end-to-end, 25-heart win condition triggers letter, chapter completes             |
| 6     | Feature        | Chapter 2: The Poems               | 4     | Phase 2                       | Phase 2 exit criteria met, Anthropic API key in environment      | Poems swipe, API returns replies, typewriter displays responses, chapter completes           |
| 7     | Feature        | Chapter 3: The Cassette            | 5     | Phase 2                       | Phase 2 exit criteria met, Chapter 3 assets and voice note ready | Permissions handled, voice unlock works, tape plays with synced phrases, chapter completes   |
| 8     | Feature        | Chapter 5: The Maze                | 5     | Phase 2                       | Phase 2 exit criteria met, Chapter 5 assets available            | Three maze sections playable, BGM crossfades, reunion cutscene plays, chapter completes      |
| 9     | Feature        | Chapter 6: The Constellations      | 4     | Phase 2                       | Phase 2 exit criteria met, Chapter 6 assets available            | All 6 constellations drawable, sky illuminate fires, final message displays                  |
| 10    | Feature        | Ending Screen                      | 2     | Phase 2, 9                    | Phase 2 and Phase 9 exit criteria met, ending assets available   | Screen renders with constellation backdrop, BGM crossfade, no navigation elements            |
| 11    | Integration    | Cross-Cutting Hardening            | 4     | Phase 3, 4, 5, 6, 7, 8, 9, 10 | All feature phase exit criteria met                              | Interruptions handled, haptics calibrated, API fallback verified, state persistence hardened |
| 12    | Integration    | Final Validation                   | 2     | Phase 11                      | Phase 11 exit criteria met, iPhone 17 Pro available              | Integration tests pass, 60fps confirmed, bundle under 150MB, full playthrough crash-free     |

## Dependency Graph

```
Phase 0 ──► Phase 1 ──► Phase 2 ──┬──► Phase 3 ───────────────┐
                                   ├──► Phase 4 ───────────────┤
                                   ├──► Phase 5 ───────────────┤
                                   ├──► Phase 6 ───────────────┤
                                   ├──► Phase 7 ───────────────┼──► Phase 11 ──► Phase 12
                                   ├──► Phase 8 ───────────────┤
                                   ├──► Phase 9 ──┬────────────┤
                                   │              │            │
                                   └──────────────┼► Phase 10 ─┘
                                                  │
                                        Phase 9 ──┘
```

---

## Phase 0: Project Scaffold

**Class:** Infrastructure
**Depends on:** None
**Design doc sections:** §1 Product Overview, §4 Architecture Overview, §5 Asset Inventory

### Entry Criteria

- Xcode 16+ with iOS 18.0 SDK installed
- Swift 6 language mode available
- Target device profile: iPhone 17 Pro

### Exit Criteria

- Xcode project builds for iOS 18.0+ deployment target with zero errors and zero warnings
- Swift 6 strict concurrency enabled in build settings
- App launches on iPhone 17 Pro simulator and displays an empty root view
- Directory structure matches §4 project structure specification
- Asset catalog initialized with subdirectory structure for Sprites, Backgrounds, Audio
- `NSMicrophoneUsageDescription` and `NSSpeechRecognitionUsageDescription` present in target configuration

### Epics

#### E0.1: Scaffold Xcode project and directory structure

**Scope:** Create the Xcode project, configure build settings for iOS 18.0+ with Swift 6 strict concurrency, establish the directory hierarchy defined in §4, and initialize asset catalogs.
**Deliverable:** Runnable component: empty app that builds and launches on simulator.
**Files touched:**

- `OttieExpress/App/OttieExpressApp.swift` (create)
- `OttieExpress/Assets.xcassets/Contents.json` (create)

**Acceptance criteria:**

- Project compiles with zero errors under Swift 6 strict concurrency
- App launches on iPhone 17 Pro simulator
- Directory tree matches `App/`, `Chapters/Chapter0-6/`, `EndingScreen/`, `SharedUI/`, `Assets/Sprites/`, `Assets/Backgrounds/`, `Assets/Audio/`, `Services/`, `Extensions/`

---

## Phase 1: Core Systems and Design Foundation

**Class:** Infrastructure
**Depends on:** Phase 0
**Design doc sections:** §2 Design System, §4 Architecture Overview (State Management, Technology Stack), §15 Haptics and Sound Design, §16 API Integration

### Entry Criteria

- Phase 0 exit criteria met
- All infrastructure files from Phase 0 verified compiling

### Exit Criteria

- All design tokens (11 colors, 6 typography roles, 4 spacing constants, motion constants) compile and render correctly in SwiftUI previews
- `AppState` persists chapter completion set to `UserDefaults` and restores on initialization
- `AudioService` plays an M4A file, stops playback, and crossfades between two tracks
- `HapticService` fires all haptic types from §15 haptic map (UIImpactFeedbackGenerator light/medium/heavy, UINotificationFeedbackGenerator success, CoreHaptics transient pattern, CoreHaptics sustained pattern)
- `AnthropicService` compiles with API key loaded from environment variable, request/response DTOs decode correctly
- Code compiles with zero errors

### Epics

#### E1.1: Define design system tokens

**Scope:** Translate the §2 color palette, typography scale, spacing constants, and motion parameters into Swift constants and SwiftUI extensions consumable by all views.
**Deliverable:** Complete data layer: design system tokens ready for consumption.
**Files touched:**

- `OttieExpress/Extensions/Color+DesignTokens.swift` (create)
- `OttieExpress/Extensions/Font+DesignTokens.swift` (create)
- `OttieExpress/Extensions/DesignConstants.swift` (create)

**Acceptance criteria:**

- All 11 color tokens from §2 accessible as static `Color` properties
- All 6 typography roles from §2 accessible as `Font` modifiers using SF Pro Display, SF Pro Text, and New York
- Spacing constants (24pt standard, 16pt compact, 44pt min tap target, 20pt card radius, 28pt modal radius) defined as named values
- Baseline spring parameters (`stiffness: 280, damping: 22`) and transition durations (0.4s screen, 0.6s chapter complete) defined as named constants

#### E1.2: Implement AppState and persistence

**Scope:** Build the `@Observable` AppState class with chapter progress tracking, `UserDefaults` persistence, and restore-on-init semantics per §4 state management specification.
**Deliverable:** Working service: AppState with verified persistence round-trip.
**Files touched:**

- `OttieExpress/App/AppState.swift` (create)

**Acceptance criteria:**

- `AppState` annotated `@MainActor` and `@Observable`
- `completeChapter(_:)` inserts into `completedChapters` set, advances `currentChapter`, and persists to `UserDefaults`
- On init, completed chapters restore from `UserDefaults`
- `UserDefaults` corruption (missing or invalid data) resets to fresh state (Chapter 0)
- Smoke test: complete chapters 0-2, re-initialize AppState, verify `currentChapter == 3` and `completedChapters == {0, 1, 2}`

#### E1.3: Build AudioService

**Scope:** Implement AVAudioPlayer-based audio management with BGM playback, SFX layering, volume control, and crossfade per §15 audio management rules.
**Deliverable:** Working service: AudioService with BGM and SFX playback.
**Files touched:**

- `OttieExpress/Services/AudioService.swift` (create)

**Acceptance criteria:**

- AVAudioSession configured with `.playback` category
- Plays M4A files from app bundle via `AVAudioPlayer`
- Only one BGM track active at any time; crossfade transitions in 0.8s
- SFX layers on top of active BGM without interrupting it
- Exposes volume control for secondary player (needed for Ch3 BGM underlay at 0.3)
- Respects device silent mode for audio; haptics remain independent

#### E1.4: Build HapticService

**Scope:** Implement the complete haptic pattern library from §15 haptic map, including UIFeedbackGenerator wrappers and CoreHaptics engine lifecycle with all declared patterns.
**Deliverable:** Working service: HapticService with all §15 patterns callable.
**Files touched:**

- `OttieExpress/Services/HapticService.swift` (create)

**Acceptance criteria:**

- UIImpactFeedbackGenerator fires at light, medium, and heavy intensities
- UINotificationFeedbackGenerator fires success type
- CoreHaptics engine initializes, handles engine reset, and tears down cleanly
- CoreHaptics transient repeating pattern with increasing intensity over 2 seconds available (fingerprint scan)
- CoreHaptics sustained pattern at medium intensity for 0.8 seconds available (voice recognition)
- Single sharp impact pattern available (scan complete, cassette click, file slam, constellation lock)
- All patterns callable by name through a typed enum interface

#### E1.5: Build AnthropicService

**Scope:** Implement the Anthropic API client for Chapter 2 poem exchange per §16, with URLSession async/await, Sendable DTOs, and environment-based API key storage. Note: §16 specifies hardcoded API key storage; CLAUDE.md §10.4 supersedes, requiring environment-based storage.
**Deliverable:** Working service: AnthropicService with compilable interface and decodable DTOs.
**Files touched:**

- `OttieExpress/Services/AnthropicService.swift` (create)
- `OttieExpress/Services/AnthropicModels.swift` (create)

**Acceptance criteria:**

- `AnthropicService` struct is `Sendable` (immutable stored properties)
- API key loaded from environment variable, not hardcoded in source
- `generateReply(to:)` sends POST to `https://api.anthropic.com/v1/messages` with claude-haiku-4-5-20251001 model and §8 Stage 1 system prompt, returns decoded response text
- `generateContinuation(of:)` sends POST with §8 Stage 2 system prompt, returns decoded response text
- Request/response DTOs conform to `Codable` and `Sendable`
- 10-second timeout configured on URLRequest

---

## Phase 2: Navigation Shell and Shared UI

**Class:** Infrastructure
**Depends on:** Phase 1
**Design doc sections:** §4 Architecture Overview (SharedUI), §14 Navigation and Progression

### Entry Criteria

- Phase 1 exit criteria met
- Design tokens, AppState, and all services verified working

### Exit Criteria

- `ChapterRouter` displays the correct chapter view based on `AppState.currentChapter`
- Chapter transitions animate with crossfade and subtle upward drift (0.4s)
- `TypewriterText` animates character-by-character with 40ms default interval, 200ms punctuation pause, 400ms line break pause, 800ms paragraph break pause
- `ChapterCard` renders with design tokens (midnight-navy background, rose-blush label, display title, muted-gray body)
- `HapticButton` fires configured haptic on tap
- Code compiles with zero errors

### Epics

#### E2.1: Build navigation shell

**Scope:** Implement the chapter routing system that reads `AppState.currentChapter`, displays the corresponding chapter root view, and handles chapter-complete transitions per §14.
**Deliverable:** Runnable component: navigation shell that routes between placeholder chapter views.
**Files touched:**

- `OttieExpress/App/ChapterRouter.swift` (create)
- `OttieExpress/App/ChapterTransition.swift` (create)
- `OttieExpress/App/OttieExpressApp.swift` (modify)

**Acceptance criteria:**

- Root view owns `@State private var appState = AppState()` and injects via `.environment(appState)`
- `ChapterRouter` reads `appState.currentChapter` and displays the correct chapter view
- Chapter transitions use crossfade with upward drift animation (0.4s, spring baseline)
- Chapter complete transitions use 0.6s with scale pulse
- Chapters are gated: a chapter view only appears when all prior chapters are in `completedChapters`

#### E2.2: Build TypewriterText component

**Scope:** Implement the reusable progressive text reveal component used across multiple chapters (Ch1 letter, Ch2 API replies, Ch3 final line, Ch6 final message) per §4 SharedUI.
**Deliverable:** Runnable component: TypewriterText with configurable timing.
**Files touched:**

- `OttieExpress/SharedUI/TypewriterText.swift` (create)

**Acceptance criteria:**

- Characters appear at configurable interval (default 40ms per §7 letter specification)
- Punctuation (`.`, `,`, `!`, `?`, `:`, `;`) adds configurable additional pause (default 200ms)
- Line breaks add configurable pause (default 400ms)
- Paragraph breaks (double newline) add configurable pause (default 800ms)
- Tap-to-complete reveals all remaining text immediately
- Uses New York Regular font by default per §2 typography
- Supports optional `sfx-typewriter-tick` audio callback per character
- Cancellation-safe: stops cleanly when view disappears

#### E2.3: Build ChapterCard component

**Scope:** Implement the reusable intro card component used by every chapter (Ch0-Ch6) with chapter label, title, body text, asset image, and action button.
**Deliverable:** Runnable component: ChapterCard rendering with design tokens.
**Files touched:**

- `OttieExpress/SharedUI/ChapterCard.swift` (create)

**Acceptance criteria:**

- Renders with `midnight-navy` card background, 28pt top corner radius
- Chapter label in SF Pro Text Semibold 13pt, `rose-blush`
- Title in Display Bold 32pt, `soft-white`
- Body in SF Pro Text 16pt, `muted-gray`
- Accepts optional image asset and action button configuration
- Action button renders with `rose-blush` fill, `soft-white` label, 56pt height, 20pt corner radius

#### E2.4: Build HapticButton component

**Scope:** Implement a reusable button wrapper that triggers a configured haptic pattern from HapticService on tap, used across all interactive elements.
**Deliverable:** Runnable component: HapticButton with configurable haptic type.
**Files touched:**

- `OttieExpress/SharedUI/HapticButton.swift` (create)

**Acceptance criteria:**

- Accepts a haptic type parameter (maps to HapticService pattern enum)
- Fires the configured haptic on tap
- Supports custom label content via `@ViewBuilder`
- Applies standard 44pt minimum tap target

---

## Phase 3: Chapter 0 — The Handshake

**Class:** Feature
**Depends on:** Phase 2
**Design doc sections:** §6 Chapter 0

### Entry Criteria

- Phase 2 exit criteria met
- Chapter 0 assets available: `fingerprint-svg`, `vault-door-top`, `vault-door-bottom`, `ottie-waving`, `sfx-fingerprint-scan`, `sfx-vault-open`

### Exit Criteria

- Fingerprint screen renders with correct layout (welcome text, fingerprint SVG, blinking hint label)
- Hold-to-scan fills fingerprint bottom-to-top with `rose-blush` glow over 2 seconds with CoreHaptics progressive pattern
- Early release resets scan smoothly
- Vault split animation plays with spring timing (0.6s) and `sfx-vault-open`
- Password screen renders with `ottie-waving`, input field, and hint text
- Incorrect password shakes input field with error haptic
- Correct password (`CarolinaLGTM`) transitions to Chapter 1
- Chapter completion persists to AppState
- Code compiles with zero errors

### Epics

#### E3.1: Build fingerprint screen

**Scope:** Implement the fingerprint screen with welcome text, SVG display, blinking hint label, hold-to-scan interaction with CoreHaptics progressive pattern, and bottom-to-top fill animation with progress ring.
**Deliverable:** Runnable component: fingerprint screen with full scan interaction.
**Files touched:**

- `OttieExpress/Chapters/Chapter0/FingerprintScreen.swift` (create)
- `OttieExpress/Chapters/Chapter0/Chapter0Constants.swift` (create)

**Acceptance criteria:**

- "Welcome Carolina" in Display font, centered top third, `soft-white`
- Fingerprint SVG centered, `rose-blush`, approximately 180pt wide
- "Hold to enter" blinks at 1-second interval in SF Pro Text Regular 15pt, `muted-gray`
- On press: CoreHaptics transient pattern fires with increasing intensity over 2 seconds
- Fingerprint fills bottom-to-top via mask animation over 2-second hold
- Circular progress ring in `warm-gold` appears around fingerprint during hold
- After 2 seconds: single sharp haptic, hint label fades
- Early release: scan resets smoothly, no error state

#### E3.2: Build vault animation and password screen

**Scope:** Implement the vault split animation (top/bottom door halves sliding off-screen) and the password entry screen with validation, shake-on-error, and glow-on-success.
**Deliverable:** Runnable component: vault animation and password screen with full validation.
**Files touched:**

- `OttieExpress/Chapters/Chapter0/VaultAnimation.swift` (create)
- `OttieExpress/Chapters/Chapter0/PasswordScreen.swift` (create)

**Acceptance criteria:**

- `vault-door-top` slides upward off-screen, `vault-door-bottom` slides downward (spring, 0.6s each)
- `sfx-vault-open` plays on split begin
- Password screen fades in beneath separating vault doors
- `ottie-waving` centered upper half, approximately 200pt tall
- Secure text field with `midnight-navy` fill, `soft-white` text, 48pt height
- "Only you can open this" in New York Regular 15pt, `muted-gray`
- Treasure map hint at bottom in SF Pro Text 13pt, `muted-gray`, 0.6 opacity
- Wrong password: horizontal shake (0.3s), medium haptic, hint pulses once
- Correct password (`CarolinaLGTM`): input glows `success-glow`, success haptic

#### E3.3: Build Chapter 0 flow coordinator

**Scope:** Wire the fingerprint → vault → password screen sequence, connect correct password to AppState chapter completion, and integrate with ChapterRouter navigation.
**Deliverable:** Integrated interaction: complete Chapter 0 flow from launch to Chapter 1 transition.
**Files touched:**

- `OttieExpress/Chapters/Chapter0/Chapter0View.swift` (create)

**Acceptance criteria:**

- Screen sequence: FingerprintScreen → VaultAnimation → PasswordScreen
- Correct password calls `appState.completeChapter(0)`
- ChapterRouter transitions to Chapter 1 after completion
- Chapter 0 is skipped on subsequent launches when already completed

---

## Phase 4: Chapter 4 — The Dossier

**Class:** Feature
**Depends on:** Phase 2
**Design doc sections:** §10 Chapter 4

### Entry Criteria

- Phase 2 exit criteria met
- Chapter 4 assets available: `bg-office-room`, `ottie-suit-seated`, `ottie-suit-pointing`, `ottie-suit-soft`, `briefcase-idle`, `briefcase-shake`, `dossier-file`, `speech-bubble`, `sfx-briefcase-shake`, `sfx-file-slam`, `sfx-speech-bubble-pop`

### Exit Criteria

- Intro card renders with correct copy, `ottie-suit-seated`, and "View Files" button
- Scene entry composites office room, Ottie, and briefcase correctly
- Briefcase shake animation plays with SFX, morphs into dossier file
- All 10 evidence files display in sequence via tap-to-advance
- Speech bubble renders evidence file text in correct format
- Ottie expression changes on file 10 (pointing → soft)
- Final closing line typewriters in New York Italic 20pt
- Chapter completion persists to AppState
- Code compiles with zero errors

### Epics

#### E4.1: Build dossier scene layout and evidence data

**Scope:** Implement the composited scene view (office room, Ottie, briefcase, table) and define all 10 evidence file data entries with their classification format per §10.
**Deliverable:** Runnable component: static scene rendering with evidence file data model.
**Files touched:**

- `OttieExpress/Chapters/Chapter4/DossierSceneView.swift` (create)
- `OttieExpress/Chapters/Chapter4/EvidenceFile.swift` (create)
- `OttieExpress/Chapters/Chapter4/Chapter4Constants.swift` (create)

**Acceptance criteria:**

- `bg-office-room` fills screen, `ottie-suit-seated` centered behind table, `briefcase-idle` on table right of Ottie
- All 10 evidence files stored as typed data with file number, classification level, and body text matching §10 verbatim
- Evidence file format renders as: `EVIDENCE FILE [number]` / `CLASSIFICATION: LEVEL HEART` / body text

#### E4.2: Build speech bubble and briefcase animations

**Scope:** Implement the briefcase shake → dossier morph sequence, speech bubble appearance and text population, tap-to-advance through evidence files, and Ottie expression transitions.
**Deliverable:** Runnable component: animated evidence file presentation with full progression.
**Files touched:**

- `OttieExpress/Chapters/Chapter4/SpeechBubbleView.swift` (create)
- `OttieExpress/Chapters/Chapter4/BriefcaseAnimation.swift` (create)

**Acceptance criteria:**

- After 1 second: briefcase oscillates 3 times with `sfx-briefcase-shake`
- Briefcase crossfades + scales into `dossier-file` with `sfx-file-slam`
- Speech bubble appears above Ottie with `sfx-speech-bubble-pop`
- Tap anywhere advances to next evidence file
- Ottie displays `ottie-suit-pointing` during files 1-9
- Ottie transitions to `ottie-suit-soft` on file 10

#### E4.3: Build Chapter 4 flow coordinator

**Scope:** Wire the intro card → scene entry → evidence file sequence → final closing line → chapter completion flow.
**Deliverable:** Integrated interaction: complete Chapter 4 from intro to chapter transition.
**Files touched:**

- `OttieExpress/Chapters/Chapter4/Chapter4View.swift` (create)

**Acceptance criteria:**

- Intro card displays with "The Dossier" title and §10 body copy
- After file 10: 1.5 second pause, dossier closes, speech bubble fades
- Final line typewriters centered in New York Italic 20pt, `parchment`: both lines from §10
- Light haptic on final line appearance
- 3 second hold, then "Continue" prompt in `muted-gray`
- Continue tap calls `appState.completeChapter(4)`

---

## Phase 5: Chapter 1 — The Runner

**Class:** Feature
**Depends on:** Phase 2
**Design doc sections:** §7 Chapter 1

### Entry Criteria

- Phase 2 exit criteria met
- Chapter 1 assets available: `ottie-run-01` through `ottie-run-04`, `ottie-jump`, `ottie-hit`, `ottie-celebrate`, `ottie-game-intro`, `bg-forest-sky`, `bg-forest-mid`, `bg-forest-floor`, `heart-collectible`, `obstacle-stormcloud`, `obstacle-thorn`, `bgm-game-runner`, `sfx-heart-collect`, `sfx-obstacle-hit`, `sfx-game-over`, `sfx-game-complete`, `sfx-chapter-transition`

### Exit Criteria

- Intro card renders with game description and "Play" button
- Three-layer parallax scrolls at differentiated speeds
- Ottie 4-frame run cycle animates at 12fps
- Tap triggers jump arc with `ottie-jump` frame; double jump blocked
- Hearts spawn, scroll left, and register collection with SFX and haptic
- Obstacles spawn with progressive difficulty per §7 tier table
- Obstacle hit triggers `ottie-hit`, knockback, and life loss
- HUD displays 3 life hearts and heart counter
- 0 lives triggers game over screen with retry
- 25 hearts triggers game complete screen with celebration
- Letter typewriters with §7 timing (40ms chars, 200ms punctuation, 400ms line break, 800ms paragraph break)
- Final line fades in (not typewriter) in New York Italic 20pt after 1.5 second pause
- Chapter completion persists to AppState
- Game sustains 60fps on simulator

### Epics

#### E5.1: Scaffold SpriteKit scene and parallax world

**Scope:** Create the SKScene subclass embedded via SpriteView, configure the three-layer parallax scroll system (sky, midground, ground floor), and set up the game world coordinate system with ground height at bottom 20%.
**Deliverable:** Runnable component: scrolling parallax forest scene.
**Files touched:**

- `OttieExpress/Chapters/Chapter1/RunnerScene.swift` (create)
- `OttieExpress/Chapters/Chapter1/ParallaxLayer.swift` (create)
- `OttieExpress/Chapters/Chapter1/Chapter1Constants.swift` (create)

**Acceptance criteria:**

- SKScene renders inside SpriteView in a SwiftUI host
- `bg-forest-sky` scrolls slowest, `bg-forest-mid` at medium speed, `bg-forest-floor` at fastest speed
- Ground floor tiles seamlessly with no visible seam during continuous scroll
- Ground height fixed at bottom 20% of screen
- Named constants for scroll speeds, ground height ratio, spawn regions

#### E5.2: Build Ottie sprite and jump controls

**Scope:** Implement the Ottie player sprite with 4-frame run cycle at 12fps, tap-to-jump mechanic with physics arc, hit state with knockback animation, and input handling that blocks double jump.
**Deliverable:** Runnable component: controllable Ottie sprite running and jumping in the scene.
**Files touched:**

- `OttieExpress/Chapters/Chapter1/OttieSprite.swift` (create)
- `OttieExpress/Chapters/Chapter1/InputHandler.swift` (create)

**Acceptance criteria:**

- Run cycle animates `ottie-run-01` through `ottie-run-04` at 12fps
- Tap anywhere triggers jump with physics arc; sprite displays `ottie-jump` during airtime
- Second tap during airtime is ignored (no double jump)
- On obstacle contact: sprite swaps to `ottie-hit`, brief knockback plays
- Ottie positioned on left third of screen, runs in place while world scrolls

#### E5.3: Build collectibles, obstacles, and spawning

**Scope:** Implement heart collectible nodes, storm cloud and thorn obstacle nodes, collision detection, and the progressive spawn manager that scales difficulty per §7 tier table.
**Deliverable:** Runnable component: hearts and obstacles spawning with collision and progressive difficulty.
**Files touched:**

- `OttieExpress/Chapters/Chapter1/HeartCollectible.swift` (create)
- `OttieExpress/Chapters/Chapter1/ObstacleNode.swift` (create)
- `OttieExpress/Chapters/Chapter1/SpawnManager.swift` (create)

**Acceptance criteria:**

- `heart-collectible` spawns at varying heights, scrolls left, collected on Ottie overlap
- `sfx-heart-collect` and light haptic on collection
- `obstacle-stormcloud` spawns at varying heights including mid-air
- `obstacle-thorn` spawns from ground only with varying widths
- Difficulty tiers: 1.0x/Low (0-5 hearts), 1.2x/Low-Medium (6-10), 1.4x/Medium (11-15), 1.6x/Medium-High (16-20), 1.8x/High (21-25)
- `sfx-obstacle-hit` and medium haptic on hit

#### E5.4: Build HUD and game state machine

**Scope:** Implement the in-game HUD (life hearts, heart counter), game state tracking (lives, collected hearts, game over, game complete), game over overlay with retry, and game complete celebration.
**Deliverable:** Runnable component: complete game loop with win/lose conditions.
**Files touched:**

- `OttieExpress/Chapters/Chapter1/GameHUD.swift` (create)
- `OttieExpress/Chapters/Chapter1/GameState.swift` (create)
- `OttieExpress/Chapters/Chapter1/GameOverView.swift` (create)
- `OttieExpress/Chapters/Chapter1/GameCompleteView.swift` (create)

**Acceptance criteria:**

- Top left: 3 life heart icons; top center: "X / 25" heart counter
- HUD text in SF Pro Text Semibold 15pt, `soft-white`, slight drop shadow
- Life lost on obstacle hit; 0 lives triggers game over
- Game over: dark overlay, "Game Over" in Display Bold 28pt, `ottie-hit` centered, "Try Again" button resets hearts and lives
- 25 hearts collected: world stops, `ottie-celebrate` bounces in center, `sfx-game-complete`, "You made it." fades in
- `sfx-game-over` plays on game over; success haptic on game complete

#### E5.5: Build letter screen and chapter flow

**Scope:** Implement the Chapter 1 intro card, the post-game letter screen with typewriter text using §7 letter content and timing, the fade-in final line, and chapter completion wiring.
**Deliverable:** Integrated interaction: complete Chapter 1 from intro card through letter to chapter transition.
**Files touched:**

- `OttieExpress/Chapters/Chapter1/LetterView.swift` (create)
- `OttieExpress/Chapters/Chapter1/Chapter1View.swift` (create)
- `OttieExpress/Chapters/Chapter1/Chapter1IntroCard.swift` (create)

**Acceptance criteria:**

- Intro card: "Chapter 1", "The Runner", §7 body text, `ottie-game-intro`, "Play" button
- After game complete: 1.5 second transition to letter screen
- Letter on `parchment` background, New York Regular 18pt, `night-base`, line height 1.6
- Full letter text from §7 typewriters with 40ms/char, 200ms punctuation, 400ms line break, 800ms paragraph break
- Final line after 1.5s pause fades in (no typewriter) in New York Italic 20pt, centered
- "Continue" prompt appears at bottom in `muted-gray`
- Continue tap calls `appState.completeChapter(1)`

---

## Phase 6: Chapter 2 — The Poems

**Class:** Feature
**Depends on:** Phase 2
**Design doc sections:** §8 Chapter 2, §16 API Integration

### Entry Criteria

- Phase 2 exit criteria met
- Anthropic API key configured as environment variable
- AnthropicService from Phase 1 verified compiling

### Exit Criteria

- Intro card renders with correct copy
- All 3 fill-in poems render with swipe-to-select blanks
- Swipe up/down cycles words in looping list; correct word locks with `success-glow`, haptic, and scale pulse
- "Next" button activates only when all blanks correct
- Poem collection reveal animates all 3 poems into scrollable stack
- Her-turn input captures text and sends to AnthropicService
- Stage 1 reply typewriters in with per-character tick SFX
- Stage 2 continuation typewriters below her poem
- API failure triggers fallback message within 10 seconds
- Per-poem completion state persists (solved poems not replayed on resume)
- Chapter completion persists to AppState
- Code compiles with zero errors

### Epics

#### E6.1: Build fill-in poem screens

**Scope:** Implement the poem view with swipe-to-select blank widgets, word list cycling, correct-word detection with lock animation, and per-poem activation of the "Next" button. Includes all poem text and word lists from §8.
**Deliverable:** Runnable component: all 3 fill-in poems with swipe interaction and validation.
**Files touched:**

- `OttieExpress/Chapters/Chapter2/PoemView.swift` (create)
- `OttieExpress/Chapters/Chapter2/SwipeBlankView.swift` (create)
- `OttieExpress/Chapters/Chapter2/PoemData.swift` (create)
- `OttieExpress/Chapters/Chapter2/Chapter2Constants.swift` (create)

**Acceptance criteria:**

- Poem text renders in New York Regular 18pt, `parchment`, on `night-base`
- Blanks display as pill-shaped widgets with `warm-gold` border, `soft-white` text
- Swipe up cycles to next word, swipe down to previous; list loops infinitely
- Correct word: pill glows `success-glow`, medium haptic, small scale pulse, word locks
- All 3 poems contain exactly the word lists and correct answers from §8
- "Next" button inactive (`muted-gray`) until all blanks correct, then active (`rose-blush`)

#### E6.2: Build poem collection reveal and her-turn input

**Scope:** Implement the post-poem-3 collection reveal animation (three poems slide into a vertical stack) and the her-turn text input screen with prompt and send button.
**Deliverable:** Runnable component: poem collection view and text input screen.
**Files touched:**

- `OttieExpress/Chapters/Chapter2/PoemCollectionView.swift` (create)
- `OttieExpress/Chapters/Chapter2/HerTurnView.swift` (create)

**Acceptance criteria:**

- Three poem cards slide in from different directions, settle into vertical scroll stack
- `sfx-poem-reveal` plays on animation start
- "Your poems" label in SF Pro Semibold 15pt, `rose-blush`
- "Continue" prompt appears after 2 seconds
- Her-turn screen: Ottie idle (small), prompt in New York Regular 20pt, multiline input in `midnight-navy` with New York Regular 17pt, "Send" button in `rose-blush`
- Input text captured on send

#### E6.3: Build API display screens

**Scope:** Implement the Stage 1 (reply) and Stage 2 (continuation) display screens with typewriter animation, loading state (blinking cursor), and fallback message on API failure.
**Deliverable:** Runnable component: API response display with typewriter and fallback.
**Files touched:**

- `OttieExpress/Chapters/Chapter2/HaikuReplyView.swift` (create)
- `OttieExpress/Chapters/Chapter2/PoemContinuationView.swift` (create)

**Acceptance criteria:**

- Stage 1: her poem displayed top half in New York Regular 17pt, `muted-gray`, slight opacity; reply typewriters below in New York Regular 18pt, `parchment`, with `sfx-typewriter-tick` per character
- Loading state: blinking cursor on empty screen for up to 10 seconds
- API failure: fallback message typewriters: "bebe, your words were so beautiful I'm still processing them. 🥰"
- Stage 2: her poem dimmed at top; continuation typewriters below with spacing (no line divider)
- After Stage 2 completion: both halves visible, "Continue" prompt after 3 seconds

#### E6.4: Wire Chapter 2 state and flow coordinator

**Scope:** Implement per-poem completion persistence (Chapter 2 exception from §4 resume semantics), the intro card → poems → collection → her-turn → API display flow, and chapter completion.
**Deliverable:** Integrated interaction: complete Chapter 2 from intro through API exchange to chapter transition.
**Files touched:**

- `OttieExpress/Chapters/Chapter2/Chapter2State.swift` (create)
- `OttieExpress/Chapters/Chapter2/Chapter2View.swift` (create)

**Acceptance criteria:**

- Per-poem completion persisted to `UserDefaults`; solved poems skipped on resume
- Flow: intro card → poem 1 → poem 2 → poem 3 → collection → her turn → stage 1 → stage 2
- Intro card: "Chapter 2", "The Poems", §8 body text
- Chapter completion calls `appState.completeChapter(2)`
- Mid-chapter resume restarts at the first unsolved poem

---

## Phase 7: Chapter 3 — The Cassette

**Class:** Feature
**Depends on:** Phase 2
**Design doc sections:** §9 Chapter 3

### Entry Criteria

- Phase 2 exit criteria met
- Chapter 3 assets available: `ottie-thinking`, `cassette-tape`, `cassette-reel-l`, `cassette-reel-r`, `vu-meter`, `sfx-tape-static`, `sfx-cassette-click`, `bgm-voice-underlay`
- Voice note recorded and delivered as M4A with final duration known
- Floating phrase timings calibrated to voice note content (Open Question #1 resolved)

### Exit Criteria

- Intro card renders with cassette tape asset and correct copy
- Hint screen displays with Ottie thinking and guidance text
- "I'm ready" triggers permission requests; denial shows Settings redirect; grant resumes flow
- `SFSpeechRecognizer` detects "I love you baby" (partial match "I love you" sufficient)
- On recognition: waveform surge, 0.8s sustained haptic, reels spin up
- Cassette engagement animation slides tape down with `sfx-cassette-click`
- Tape playing screen: VU meter needles react to audio level, reels animate (left depletes, right fills)
- Voice note plays via `AVAudioPlayer`; BGM underlay fades in at 30-second mark at volume 0.3
- Floating phrases appear at calibrated times and drift upward
- Final line typewriters at 60ms/character in New York Italic 21pt after 1.5 second silence
- Chapter completion persists to AppState
- Code compiles with zero errors

### Epics

#### E7.1: Build hint screen and permission flow

**Scope:** Implement the Chapter 3 intro card, the hint screen with Ottie thinking and guidance text, the "I'm ready" button that triggers dual permission requests (microphone + speech recognition), and the denial-to-Settings redirect flow.
**Deliverable:** Runnable component: hint screen with full permission handling.
**Files touched:**

- `OttieExpress/Chapters/Chapter3/HintScreen.swift` (create)
- `OttieExpress/Chapters/Chapter3/PermissionHandler.swift` (create)
- `OttieExpress/Chapters/Chapter3/Chapter3Constants.swift` (create)

**Acceptance criteria:**

- Hint screen: `ottie-thinking` centered, guidance text in New York Regular 17pt, `parchment`
- "I'm ready" button at bottom in `midnight-navy` fill, `muted-gray` text
- On tap: requests microphone and speech recognition permissions
- On denial: centered screen with "To continue..." text and "Open Settings" button (`rose-blush`) calling `UIApplication.openSettings()`
- On return from Settings with permission granted: resumes voice unlock flow

#### E7.2: Implement voice recognition and unlock

**Scope:** Implement `SFSpeechRecognizer` streaming recognition with on-device preference, phrase matching for "I love you baby" (case insensitive, partial match), waveform visualization reacting to microphone input, and the recognition success sequence.
**Deliverable:** Runnable component: voice unlock with phrase detection and visual feedback.
**Files touched:**

- `OttieExpress/Chapters/Chapter3/VoiceRecognizer.swift` (create)
- `OttieExpress/Chapters/Chapter3/VoiceUnlockView.swift` (create)

**Acceptance criteria:**

- `SFSpeechRecognizer` configured for English, on-device when available
- `SFSpeechAudioBufferRecognitionRequest` streams from `AVAudioEngine` input
- Recognition target: "i love you" as minimum match (case insensitive)
- Ambient state: cassette centered with slow reel animation, `sfx-tape-static` playing softly
- Waveform visualizer beneath cassette reacts to live audio level
- On match: waveform surges, 0.8s CoreHaptics sustained haptic, reels spin to full speed
- Recognition teardown cleans up engine and request properly

#### E7.3: Build tape playback UI

**Scope:** Implement the tape playing screen layout with VU meter panel (needles animated by audio level), cassette with dual reel animation (left depletes, right fills), and tape-style progress bar.
**Deliverable:** Runnable component: tape playback UI with animated VU meter and reels.
**Files touched:**

- `OttieExpress/Chapters/Chapter3/CassetteView.swift` (create)
- `OttieExpress/Chapters/Chapter3/VUMeterView.swift` (create)
- `OttieExpress/Chapters/Chapter3/ReelAnimation.swift` (create)

**Acceptance criteria:**

- `vu-meter` panel top center, two needle gauges animated by audio output level
- Cassette below VU meter, `cassette-reel-l` starts full and depletes, `cassette-reel-r` starts empty and fills
- Reel animation driven by playback progress (0.0 to 1.0)
- Progress bar styled as tape transfer beneath cassette, no timestamps
- VU needles settle to zero when audio stops

#### E7.4: Implement audio playback and floating phrase sync

**Scope:** Implement voice note playback via `AVAudioPlayer`, BGM underlay fade-in at 30-second mark, and the floating key phrase system timed to audio playback position per §9.
**Deliverable:** Integrated interaction: synchronized audio playback with floating phrases.
**Files touched:**

- `OttieExpress/Chapters/Chapter3/TapePlayerView.swift` (create)
- `OttieExpress/Chapters/Chapter3/FloatingPhraseView.swift` (create)
- `OttieExpress/Chapters/Chapter3/PhraseTimingData.swift` (create)

**Acceptance criteria:**

- Voice note plays from app bundle via `AVAudioPlayer`
- `bgm-voice-underlay` fades in at 30-second mark at volume 0.3 via AudioService secondary player
- Floating phrases ("my person" ~38s, "my peace" ~42s, "a love that feels like home" ~46s, "the future I want" ~52s) fade in and drift upward at calibrated times
- Phrases render in New York Italic 19pt, `warm-gold`, centered
- Phrase timing data stored as typed constants, adjustable per final audio duration

#### E7.5: Build final line and chapter flow coordinator

**Scope:** Implement the cassette engagement transition (tape slides down, screen warms), the post-playback final line typewriter at 60ms/character, and the full Chapter 3 flow coordinator with chapter completion.
**Deliverable:** Integrated interaction: complete Chapter 3 from intro to chapter transition.
**Files touched:**

- `OttieExpress/Chapters/Chapter3/FinalLineView.swift` (create)
- `OttieExpress/Chapters/Chapter3/Chapter3View.swift` (create)

**Acceptance criteria:**

- Cassette engagement: tape slides downward (0.8s), screen transitions from `night-base` to `constellation-blue` with amber undertones, `sfx-cassette-click` plays
- After voice note ends: VU meters settle, reels stop, 1.5 second silence
- Final line typewriters at 60ms/character in New York Italic 21pt, `parchment`, centered: "If I had to choose again, in every lifetime, I would still find my way to you."
- Line remains on screen (no fade)
- "Continue" prompt after 3 seconds in `muted-gray`
- Continue calls `appState.completeChapter(3)`
- Flow: intro card → hint → voice unlock → cassette engagement → tape playing → final line

---

## Phase 8: Chapter 5 — The Maze

**Class:** Feature
**Depends on:** Phase 2
**Design doc sections:** §11 Chapter 5

### Entry Criteria

- Phase 2 exit criteria met
- Chapter 5 assets available: `ottie-topdown`, all `ottie-topdown-walk-*` directional frames, `carolina-otter-wait`, `carolina-otter-hug`, `ottie-hug`, `bg-maze-gray`, `bg-maze-amber`, `bg-maze-gold`, `bgm-maze-gray`, `bgm-maze-amber`, `bgm-maze-gold`, `sfx-footstep-soft`, `sfx-wall-bump`, `sfx-maze-transition`, `sfx-reunion-fanfare`

### Exit Criteria

- Intro card renders with correct copy and "Find the way" button
- Section 1 (gray, 8x12, 2-3 dead ends) renders with correct tileset and BGM
- Section 2 (amber, 10x14, 3-4 dead ends) renders with BGM crossfade from Section 1
- Section 3 (gold, 6x10, 0-1 dead ends) renders with BGM crossfade, Carolina otter visible from entry
- Swipe controls move Ottie one step per swipe in 4 directions with directional walk animation
- Wall contact plays `sfx-wall-bump` and gentle haptic; Ottie does not move
- Section transitions use camera fade-to-black (0.5s) with tileset and BGM crossfade
- Wall and path labels render per §11 specifications
- Reunion cutscene: auto-walk, hug sprites, heart burst particles, camera pullback, "Worth every detour" fade-in
- Chapter completion persists to AppState
- Maze renders at 60fps on simulator

### Epics

#### E8.1: Scaffold SpriteKit maze scene and tileset rendering

**Scope:** Create the SpriteKit scene for top-down maze rendering embedded via SpriteView, implement tileset loading and rendering for all three visual themes (gray, amber, gold), and configure the camera system.
**Deliverable:** Runnable component: maze scene rendering a static tileset with camera.
**Files touched:**

- `OttieExpress/Chapters/Chapter5/MazeScene.swift` (create)
- `OttieExpress/Chapters/Chapter5/MazeTileset.swift` (create)
- `OttieExpress/Chapters/Chapter5/Chapter5Constants.swift` (create)

**Acceptance criteria:**

- SKScene renders inside SpriteView in SwiftUI host
- Three tilesets render correctly: `bg-maze-gray`, `bg-maze-amber`, `bg-maze-gold`
- Camera centers on player position with smooth follow
- Named constants for grid cell size, maze dimensions per section, camera parameters

#### E8.2: Build Ottie top-down sprite and swipe controls

**Scope:** Implement the top-down Ottie sprite with 4-directional walking animation, swipe-based movement (one step per swipe), wall collision detection, and movement SFX/haptics.
**Deliverable:** Runnable component: controllable Ottie navigating a maze grid.
**Files touched:**

- `OttieExpress/Chapters/Chapter5/MazeOttieSprite.swift` (create)
- `OttieExpress/Chapters/Chapter5/SwipeController.swift` (create)

**Acceptance criteria:**

- Swipe up/down/left/right moves Ottie one grid cell in that direction
- Directional walk animation uses correct `ottie-topdown-walk-*` frames for each direction
- Wall collision: `sfx-wall-bump`, gentle haptic, Ottie remains in place
- `sfx-footstep-soft` plays on valid movement (alternating variants)

#### E8.3: Build maze layouts and section transitions

**Scope:** Define maze layouts for all three sections meeting §11 grid size and dead-end constraints, implement wall and path labeling, and build the section transition with camera fade-to-black, tileset swap, and BGM crossfade.
**Deliverable:** Runnable component: three navigable maze sections with labeled elements and transitions.
**Files touched:**

- `OttieExpress/Chapters/Chapter5/MazeLayout.swift` (create)
- `OttieExpress/Chapters/Chapter5/MazeSectionData.swift` (create)
- `OttieExpress/Chapters/Chapter5/SectionTransition.swift` (create)

**Acceptance criteria:**

- Section 1: 8x12 grid, 2-3 dead ends, ~45-60 second solve, labeled walls (Distance Lag, WiFi From 2007, Overthinking Loop) and paths (Good Morning Calls, Soft Reassurance Zone, Inside Joke Territory)
- Section 2: 10x14 grid, 3-4 dead ends, ~60-90 second solve, labeled walls/paths per §11
- Section 3: 6x10 grid, 0-1 dead ends, ~30-45 second solve, labeled walls/paths per §11, Carolina otter visible from entry
- Transitions: 0.5s camera fade-to-black, tileset swaps, BGM crossfades via AudioService
- `sfx-maze-transition` plays at section boundaries

#### E8.4: Build reunion cutscene and particle effects

**Scope:** Implement the reunion sequence triggered when Ottie reaches Carolina's otter: auto-walk, sprite swap to hug frames, heart burst particle effect, camera pullback revealing the full maze, and the closing line.
**Deliverable:** Runnable component: reunion cutscene with particles and camera animation.
**Files touched:**

- `OttieExpress/Chapters/Chapter5/ReunionCutscene.swift` (create)
- `OttieExpress/Chapters/Chapter5/HeartParticleEmitter.swift` (create)
- `OttieExpress/Chapters/Chapter5/CarolinaOtterNode.swift` (create)

**Acceptance criteria:**

- On reaching Carolina: movement controls disable
- Ottie auto-walks final steps to Carolina
- Sprites swap to `ottie-hug` and `carolina-otter-hug`
- 8-12 heart particles burst outward from center in an arc
- Camera pulls back over 1.5s revealing full maze (scale transform on world)
- BGM fades to quiet
- `sfx-reunion-fanfare` plays
- "Worth every detour." fades in, New York Italic 20pt, `warm-gold`, centered
- 3 second hold, "Continue" prompt appears

#### E8.5: Wire Chapter 5 flow coordinator

**Scope:** Implement the Chapter 5 intro card and the full flow coordinator connecting intro → Section 1 → Section 2 → Section 3 → reunion → chapter completion.
**Deliverable:** Integrated interaction: complete Chapter 5 from intro to chapter transition.
**Files touched:**

- `OttieExpress/Chapters/Chapter5/Chapter5View.swift` (create)
- `OttieExpress/Chapters/Chapter5/Chapter5IntroCard.swift` (create)

**Acceptance criteria:**

- Intro card: "Chapter 5", "The Way Home", §11 body text, `ottie-topdown`, "Find the way" button
- Flow: intro → section 1 → section 2 → section 3 → reunion cutscene
- Continue calls `appState.completeChapter(5)`
- Ch1 game state resets on resume per §4 (hearts and lives reset)

---

## Phase 9: Chapter 6 — The Constellations

**Class:** Feature
**Depends on:** Phase 2
**Design doc sections:** §12 Chapter 6

### Entry Criteria

- Phase 2 exit criteria met
- Chapter 6 assets available: `star-dot-idle`, `star-dot-active`, `constellation-line`, `memory-card-bg`, `bg-night-sky`, `bgm-constellation`, `sfx-star-connect`, `sfx-constellation-lock`, `sfx-sky-illuminate`
- Constellation dot positions for constellations 1-5 designed (Open Question #2 resolved)
- Constellation 6 otter silhouette designed at small scale

### Exit Criteria

- Intro card renders with correct copy and "Look up" button
- Star field fades in over 2 seconds with intro line
- `bgm-constellation` plays, sparse and quiet
- 5 named constellations unlock sequentially via drag-to-connect
- Drawing: drag from star to star in order, glowing line renders between connected dots
- On lock: constellation flashes, name appears, `sfx-constellation-lock` plays
- Memory cards slide up with correct text per §12, dismiss on tap
- Constellation 6: brighter trembling dots, otter silhouette forms, "Carolina." typewriters on lock
- Sky illuminate: all 6 flash, `sfx-sky-illuminate` (orchestral swell), strong haptic, camera zoom-out
- Final message typewriters word-by-word at 80ms in New York Regular 20pt, `parchment`, with glow
- Chapter completion persists to AppState
- Code compiles with zero errors

### Epics

#### E9.1: Build star field and constellation data model

**Scope:** Implement the star field background view with fade-in animation, define the constellation data model (star positions, connection order, names, memory card text) for all 6 constellations, and build the intro sequence.
**Deliverable:** Complete data layer: constellation data model and star field rendering.
**Files touched:**

- `OttieExpress/Chapters/Chapter6/StarFieldView.swift` (create)
- `OttieExpress/Chapters/Chapter6/ConstellationData.swift` (create)
- `OttieExpress/Chapters/Chapter6/Chapter6Constants.swift` (create)

**Acceptance criteria:**

- `night-base` background with stars fading in one-by-one over 2 seconds
- Intro line drifts in: "Some things are written in the stars." in New York Regular 18pt, `parchment`
- All 6 constellations defined with star positions, connection sequence, names, and memory card text matching §12 verbatim
- Constellation 6 dot positions form recognizable otter silhouette at rendering scale
- Only current active constellation dots pulse; future constellations invisible

#### E9.2: Implement constellation drawing interaction

**Scope:** Implement the drag-to-connect mechanic: user drags from one star dot to the next in sequence, glowing line segments render between connected dots, and constellation completion detection triggers lock.
**Deliverable:** Runnable component: interactive constellation drawing with line rendering.
**Files touched:**

- `OttieExpress/Chapters/Chapter6/ConstellationDrawingView.swift` (create)
- `OttieExpress/Chapters/Chapter6/StarDotView.swift` (create)
- `OttieExpress/Chapters/Chapter6/ConnectionLineView.swift` (create)

**Acceptance criteria:**

- Active star dots pulse with `star-dot-idle`; connected dots display `star-dot-active`
- Drag from one dot to next in sequence renders `constellation-line` between them
- `sfx-star-connect` plays per connection
- Dots must be connected in order; out-of-order connections rejected
- On final connection: constellation flashes brightly, name appears in SF Pro Semibold 13pt, `warm-gold`
- `sfx-constellation-lock` plays on lock

#### E9.3: Build memory cards and sequential constellation unlock

**Scope:** Implement the memory card slide-up presentation (frosted card background with text) shown after each constellation lock, tap-to-dismiss behavior, and the sequential unlock system that reveals the next constellation after card dismissal.
**Deliverable:** Runnable component: memory cards with sequential constellation progression.
**Files touched:**

- `OttieExpress/Chapters/Chapter6/MemoryCardView.swift` (create)
- `OttieExpress/Chapters/Chapter6/ConstellationSequencer.swift` (create)

**Acceptance criteria:**

- Memory card slides up from bottom with `memory-card-bg` styling
- Card text in New York Regular 16pt, matching §12 text for each constellation verbatim
- Tap dismisses card
- After dismissal: next constellation dots appear and begin pulsing
- Constellation 6: brighter, slightly trembling dots, no name hint
- On constellation 6 lock: "Carolina." typewriters above the otter shape, 2 second hold

#### E9.4: Build sky illuminate and chapter finale

**Scope:** Implement the full sky illuminate sequence (all constellations flash, orchestral swell, camera zoom-out), the final message typewriter (word-by-word at 80ms), and chapter completion wiring including the intro card.
**Deliverable:** Integrated interaction: complete Chapter 6 from intro through sky illuminate to transition.
**Files touched:**

- `OttieExpress/Chapters/Chapter6/SkyIlluminateView.swift` (create)
- `OttieExpress/Chapters/Chapter6/FinalMessageView.swift` (create)
- `OttieExpress/Chapters/Chapter6/Chapter6View.swift` (create)

**Acceptance criteria:**

- After "Carolina." hold: all 6 constellations flash simultaneously
- `sfx-sky-illuminate` plays (orchestral swell)
- Strong success haptic
- Camera zooms out revealing all constellations in relation
- Screen dims slightly, constellations glow at steady warm intensity
- Final message typewriters word-by-word at 80ms: three lines from §12 in New York Regular 20pt, `parchment`, centered, slight glow
- 4 second hold after final word, "Continue" prompt fades in
- Intro card: "Chapter 6", "Written in the Stars", §12 body text, "Look up" button
- Continue calls `appState.completeChapter(6)`

---

## Phase 10: Ending Screen

**Class:** Feature
**Depends on:** Phase 2, 9
**Design doc sections:** §13 Ending Screen

### Entry Criteria

- Phase 2 exit criteria met
- Phase 9 exit criteria met (constellation data model and rendering available)
- Ending assets available: `ottie-ending`, `bgm-ending`

### Exit Criteria

- `night-base` background displays with all 6 constellations at 0.15 opacity
- `bgm-ending` fades in as constellation BGM fades out via AudioService crossfade
- "Happy Valentine's Day, Carolina." in New York Regular 28pt, `parchment`
- "With love, Dinesh" in New York Italic 20pt, `muted-gray`
- `ottie-ending` in bottom right corner (~80pt), 2-frame wave loop
- No buttons, no continue prompt, no navigation elements
- Code compiles with zero errors

### Epics

#### E10.1: Build ending screen layout and Ottie animation

**Scope:** Implement the ending screen with centered text stack, waving Ottie sprite in bottom right with 2-frame loop animation, and the `night-base` background.
**Deliverable:** Runnable component: ending screen with correct layout and animation.
**Files touched:**

- `OttieExpress/EndingScreen/EndingScreenView.swift` (create)
- `OttieExpress/EndingScreen/EndingConstants.swift` (create)

**Acceptance criteria:**

- "Happy Valentine's Day, Carolina." in New York Regular 28pt, `parchment`, centered
- "With love, Dinesh" in New York Italic 20pt, `muted-gray`, below with spacing
- `ottie-ending` approximately 80pt in bottom right corner
- Ottie 2-frame wave animation loops gently
- No buttons, no continue, no navigation elements; screen is terminal

#### E10.2: Build constellation backdrop and audio crossfade

**Scope:** Render all 6 constellations from Chapter 6 data at 0.15 opacity as a faint background layer, and wire the BGM crossfade from `bgm-constellation` to `bgm-ending`.
**Deliverable:** Integrated interaction: ending screen with constellation atmosphere and audio.
**Files touched:**

- `OttieExpress/EndingScreen/ConstellationBackdrop.swift` (create)

**Acceptance criteria:**

- All 6 constellation shapes rendered using positions from `ConstellationData` (Phase 9)
- Constellation lines and dots at 0.15 opacity against `night-base`
- `bgm-constellation` fades out as `bgm-ending` fades in via AudioService crossfade (0.8s)
- Ending BGM loops or sustains without abrupt cutoff

---

## Phase 11: Cross-Cutting Hardening

**Class:** Integration
**Depends on:** Phase 3, 4, 5, 6, 7, 8, 9, 10
**Design doc sections:** §3 Global Rules, §4 Architecture (State Machine and Resume Semantics), §15 Haptics and Sound Design (Audio Management), §16 API Integration

### Entry Criteria

- All feature phase (3-10) exit criteria met
- All chapters individually playable end-to-end
- iPhone 17 Pro physical device available for testing

### Exit Criteria

- Audio interruption handling: on interruption began, all playback pauses, VU meter and reel animations freeze (Ch3), floating phrase timers freeze
- Audio interruption ended with `.shouldResume`: playback resumes from paused position, floating phrase timers resync to current playback time
- Pause longer than 5 seconds: subtle "Resume" indicator before auto-resuming
- Full haptic map from §15 verified firing at correct intensities across all chapters
- API timeout fires at exactly 10 seconds; fallback message renders correctly
- AppState resume: force-quit mid-chapter relaunches at chapter beginning (except Ch2 per-poem state)
- `UserDefaults` corruption recovery: missing/invalid data resets to Chapter 0
- Chapter 1 game state resets on resume (hearts and lives return to initial)
- Code compiles with zero errors

### Epics

#### E11.1: Harden AudioService session lifecycle

**Scope:** Add `AVAudioSession.interruptionNotification` observer, implement pause/resume logic for all active players, add floating phrase timer freeze/resync for Chapter 3, and add 5-second resume indicator logic.
**Deliverable:** Working service: AudioService with verified interruption resilience.
**Files touched:**

- `OttieExpress/Services/AudioService.swift` (modify)

**Acceptance criteria:**

- Registers for `AVAudioSession.interruptionNotification`
- On `.began`: pauses all active `AVAudioPlayer` instances
- On `.ended` with `.shouldResume`: resumes from paused position
- Exposes pause/resume callbacks for dependent UI (VU meter, reels, phrase timers)
- Tracks pause duration; if >5 seconds, signals for resume indicator display
- Crossfade validated across Ch5 section transitions under interruption scenarios

#### E11.2: Calibrate HapticService pattern library

**Scope:** Validate all CoreHaptics and UIFeedbackGenerator patterns against the §15 haptic map, fine-tune timing parameters for perceptual quality on iPhone 17 Pro hardware, and verify silent mode behavior.
**Deliverable:** Working service: HapticService with all patterns verified on device.
**Files touched:**

- `OttieExpress/Services/HapticService.swift` (modify)

**Acceptance criteria:**

- Every row in §15 haptic map fires at the specified type and intensity
- CoreHaptics fingerprint scan pattern increases intensity smoothly over 2 seconds
- CoreHaptics voice recognition sustained pattern holds for 0.8 seconds
- Haptics fire when device is on silent mode
- Engine reset recovery verified: engine restarts cleanly after background/foreground cycle

#### E11.3: Harden AnthropicService resilience

**Scope:** Validate timeout behavior at exactly 10 seconds, verify fallback message rendering, test network error paths, and confirm the service is Sendable under Swift 6 strict concurrency.
**Deliverable:** Working service: AnthropicService with verified timeout and fallback.
**Files touched:**

- `OttieExpress/Services/AnthropicService.swift` (modify)

**Acceptance criteria:**

- URLRequest timeout interval set to 10 seconds
- Network timeout triggers fallback: "bebe, your words were so beautiful I'm still processing them. 🥰"
- HTTP error responses (4xx, 5xx) trigger same fallback
- JSON decode failures trigger same fallback
- Sendable conformance verified under strict concurrency

#### E11.4: Harden AppState persistence and resume semantics

**Scope:** Validate all §4 resume behaviors: mid-chapter resume at chapter start, Chapter 2 per-poem persistence, Chapter 1 game reset on resume, UserDefaults corruption recovery, and TestFlight update survival.
**Deliverable:** Working service: AppState with verified resume semantics across all edge cases.
**Files touched:**

- `OttieExpress/App/AppState.swift` (modify)

**Acceptance criteria:**

- Force-quit mid-chapter: relaunches at beginning of incomplete chapter
- Chapter 2 per-poem state survives force-quit; solved poems not replayed
- Chapter 1 game state (hearts, lives) resets to initial on resume
- `UserDefaults` cleared or corrupted: app starts from Chapter 0 with no crash
- State survives simulated TestFlight update (`UserDefaults` persists across app updates)

---

## Phase 12: Final Validation

**Class:** Integration
**Depends on:** Phase 11
**Design doc sections:** Goals, §4 Architecture, Rollout Plan

### Entry Criteria

- Phase 11 exit criteria met
- iPhone 17 Pro running iOS 18 available
- All audio and visual assets in final form
- Voice note with calibrated floating phrase timings integrated

### Exit Criteria

- Integration test suite passes covering cross-chapter state transitions and service interactions
- All SwiftUI animations and SpriteKit scenes sustain 60fps on iPhone 17 Pro (verified via Instruments trace)
- App bundle size under 150MB
- Full playthrough (Chapter 0 through Ending Screen) completes without crash on iPhone 17 Pro
- `protocol-zero.sh` exits 0 on the final branch
- No blocking defects remain

### Epics

#### E12.1: Write cross-chapter integration tests

**Scope:** Implement integration tests covering chapter state transitions, AppState persistence round-trips, AudioService interruption recovery, AnthropicService fallback behavior, and navigation routing correctness.
**Deliverable:** Passing test suite: integration tests for cross-system behavior.
**Files touched:**

- `OttieExpressTests/IntegrationTests.swift` (create)

**Acceptance criteria:**

- Tests cover: complete chapter N → currentChapter advances to N+1
- Tests cover: persist state → re-init AppState → state matches
- Tests cover: AnthropicService timeout → fallback string returned
- Tests cover: ChapterRouter displays correct view for each currentChapter value (0-6 and ending)
- No real network calls in tests; AnthropicService calls mocked
- All tests pass with `swift test`

#### E12.2: Validate performance and bundle on device

**Scope:** Run Instruments profiling on iPhone 17 Pro for all chapters, verify 60fps sustained across SpriteKit scenes and SwiftUI animations, measure bundle size, and execute a full end-to-end playthrough.
**Deliverable:** Passing test suite: performance benchmarks confirmed on target hardware.
**Files touched:**

- `OttieExpressTests/PerformanceTests.swift` (create)

**Acceptance criteria:**

- Instruments Time Profiler trace shows no frame drops below 60fps during Ch1 game, Ch5 maze, and Ch6 constellation drawing
- Instruments Allocations trace shows no memory leaks during chapter transitions
- App bundle size measured and confirmed under 150MB (estimated ~45MB per §5)
- Full playthrough: Chapter 0 → Chapter 1 → Chapter 2 → Chapter 3 → Chapter 4 → Chapter 5 → Chapter 6 → Ending Screen completes without crash
- `protocol-zero.sh` exits 0

---

## Execution Notes

- Phases execute sequentially. Do not begin Phase N+1 until Phase N exit criteria are met.
- Epics within a phase may execute in parallel if their file sets are disjoint.
- If an epic's acceptance criteria cannot be met, stop and reassess before continuing.
- This roadmap is deterministic. Follow it top-to-bottom.
- Open Question dependencies: Phase 7 (Ch3) is blocked until the voice note is recorded and floating phrase timings are calibrated. Phase 9 (Ch6) requires constellation dot positions to be designed before implementation. Audio/BGM/SFX tracks must be sourced before their consuming feature phases begin.
- API key governance: AnthropicService loads the API key from an environment variable per CLAUDE.md §10.4, diverging from design doc §16 which specifies hardcoded storage. CLAUDE.md authority supersedes.
