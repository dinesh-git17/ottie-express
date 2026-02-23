# Ottie Express — Product Design Document

**Version:** 1.2
**Author:** Dinesh
**Reviewers:** Claude (automated design review)
**Platform:** iOS 18.0+ (SwiftUI) — iPhone 17 Pro
**Orientation:** Portrait only
**Audience:** Engineering
**Status:** Draft
**Created:** 2026-02-23
**Last Updated:** 2026-02-23

---

## Table of Contents

1. [Product Overview](#1-product-overview)
2. [Design System](#2-design-system)
3. [Global Rules](#3-global-rules)
4. [Goals](#goals)
5. [Non-Goals](#non-goals)
6. [Architecture Overview](#4-architecture-overview)
7. [Alternatives Considered](#alternatives-considered)
8. [Asset Inventory](#5-asset-inventory)
9. [Chapter 0 — The Handshake](#6-chapter-0--the-handshake)
10. [Chapter 1 — The Runner](#7-chapter-1--the-runner)
11. [Chapter 2 — The Poems](#8-chapter-2--the-poems)
12. [Chapter 3 — The Cassette](#9-chapter-3--the-cassette)
13. [Chapter 4 — The Dossier](#10-chapter-4--the-dossier)
14. [Chapter 5 — The Maze](#11-chapter-5--the-maze)
15. [Chapter 6 — The Constellations](#12-chapter-6--the-constellations)
16. [Ending Screen](#13-ending-screen)
17. [Navigation and Progression](#14-navigation-and-progression)
18. [Haptics and Sound Design](#15-haptics-and-sound-design)
19. [API Integration and Security](#16-api-integration)
20. [Rollout Plan](#rollout-plan)
21. [Open Questions](#open-questions)

---

## 1. Product Overview

Ottie Express is a private, single-user iOS application built as a Valentine's Day gift. It is not distributed via the App Store. Distribution is via TestFlight. The sole user is Carolina.

The app is structured as a linear narrative experience across six chapters plus an ending screen. Each chapter is a self-contained interactive experience — mini-games, puzzles, animated scenes, voice notes — all building toward a final emotional payoff. The tone moves from playful and mysterious in the early chapters to deeply personal and romantic by the end.

The mascot and primary character throughout the app is Ottie, a Pixar-style animated male otter who appears in various roles across chapters. All character assets are AI-generated, background-removed, and cropped manually before import.

The experience is designed to feel like a premium native iOS product. FAANG-level polish. No placeholder UI, no generic components, no AI-generated aesthetic cliches. Every screen should feel intentional.

---

## 2. Design System

### Color Palette

| Token                | Hex       | Usage                                       |
| -------------------- | --------- | ------------------------------------------- |
| `night-base`         | `#0D0D1A` | Primary backgrounds, deep screens           |
| `midnight-navy`      | `#111827` | Card backgrounds, elevated surfaces         |
| `constellation-blue` | `#1E3A5F` | Accent surfaces, chapter cards              |
| `warm-gold`          | `#F5C842` | Primary accent, hearts, stars, highlights   |
| `rose-blush`         | `#E8708A` | Secondary accent, love elements, UI glow    |
| `parchment`          | `#FAF3E0` | Light text on dark, letter backgrounds      |
| `soft-white`         | `#F9F9F9` | Body text, input text                       |
| `muted-gray`         | `#8A8A9A` | Hint text, secondary labels                 |
| `forest-green`       | `#2D5A27` | Chapter 1 game environment mid tones        |
| `amber-warm`         | `#C97D2E` | Chapter 5 section 2 transition              |
| `success-glow`       | `#4ADE80` | Correct word confirmation, chapter complete |

### Typography

| Role            | Font           | Weight   | Size    |
| --------------- | -------------- | -------- | ------- |
| Display / Title | SF Pro Display | Bold     | 32–40pt |
| Chapter heading | SF Pro Display | Semibold | 24pt    |
| Body            | SF Pro Text    | Regular  | 16–18pt |
| Poem / Letter   | New York       | Regular  | 17–20pt |
| Hint / Caption  | SF Pro Text    | Regular  | 13pt    |
| Typewriter text | New York       | Regular  | 18pt    |

All typewriter-animated text uses New York for a literary feel. All UI chrome uses SF Pro for a native feel.

### Spacing and Layout

- Safe area insets respected on all screens
- Minimum tap target: 44x44pt
- Card corner radius: 20pt
- Modal corner radius: 28pt
- Standard horizontal padding: 24pt
- Compact horizontal padding: 16pt

### Motion Principles

- Spring animations preferred over linear: `stiffness 280, damping 22` as baseline
- Screen transitions: crossfade with subtle upward drift (0.4s)
- Chapter complete transitions: 0.6s with light scale pulse
- Nothing snaps. Everything breathes.
- Respect `prefers-reduced-motion` accessibility setting throughout

---

## 3. Global Rules

These rules apply to every screen and every line of copy in the app without exception.

**No em dashes.** Do not use em dashes (--) anywhere in the app. In copy, UI text, poems, letters, dossier files, constellation cards, or any other text surface. Use periods, line breaks, or restructure the sentence.

**No AI-generated aesthetic.** No gradient mesh blobs, no purple-to-teal gradients, no glassmorphism overuse. The app is dark, warm, and tactile.

**No generic loading states.** Any async operation that requires waiting should have a themed animation, not a spinner.

**Single user, no auth.** The app has one hardcoded password. No accounts, no backend user system, no analytics.

**Linear chapter progression.** Chapters unlock sequentially. A chapter cannot be accessed until the previous chapter is marked complete. There is no chapter select screen.

**Haptics are non-negotiable.** Every meaningful interaction has a corresponding haptic response. See Section 15.

---

## Goals

| Priority | Goal                             | Success Criterion                                                                                |
| -------- | -------------------------------- | ------------------------------------------------------------------------------------------------ |
| P0       | Crash-free full playthrough      | App completes all chapters and ending screen without crash on iPhone 17 Pro / iOS 18             |
| P0       | Chapter state persistence        | Completed chapter state survives app termination, relaunch, and device restart                   |
| P0       | API integration resilience       | Anthropic API returns valid response or graceful fallback triggers within 10 seconds             |
| P0       | Voice recognition responsiveness | `SFSpeechRecognizer` detects trigger phrase within 5 seconds of utterance in a quiet environment |
| P1       | Animation performance            | All SwiftUI animations and SpriteKit scenes sustain 60fps on iPhone 17 Pro                       |
| P1       | Bundle size                      | Total app bundle under 150MB                                                                     |
| P1       | Audio session correctness        | Voice note and BGM playback resume correctly after system audio interruption                     |

---

## Non-Goals

| Non-Goal                                 | Rationale                                                        |
| ---------------------------------------- | ---------------------------------------------------------------- |
| App Store distribution                   | Private gift app. TestFlight-only distribution to a single user. |
| User accounts or authentication system   | Single hardcoded password. No backend user system required.      |
| Analytics, telemetry, or crash reporting | Single user. No data collection. Bugs reported directly.         |
| VoiceOver or Dynamic Type support        | Single known user with no accessibility requirements.            |
| Localization                             | English only. Single user, single locale.                        |
| Multi-device sync                        | Single device (iPhone 17 Pro). No iCloud or cross-device state.  |

---

## 4. Architecture Overview

### Project Structure

```
OttieExpress/
  App/
    OttieExpressApp.swift
    AppState.swift              // Global chapter progress, persisted via UserDefaults
  Chapters/
    Chapter0/                   // Handshake + vault + password
    Chapter1/                   // Infinite runner game
    Chapter2/                   // Fill-in poems + Haiku exchange
    Chapter3/                   // Cassette tape voice note
    Chapter4/                   // Dossier roast
    Chapter5/                   // Maze
    Chapter6/                   // Constellation map
  EndingScreen/
  SharedUI/                     // Reusable components: TypewriterText, ChapterCard, HapticButton
  Assets/
    Sprites/
    Backgrounds/
    Audio/
  Services/
    AnthropicService.swift      // Haiku API wrapper (Chapter 2)
    AudioService.swift          // AVAudioPlayer management
    HapticService.swift         // UIImpactFeedbackGenerator management
  Extensions/
```

### State Management

Chapter progress is stored in `AppState` using the `@Observable` macro (Observation framework, iOS 17+) and injected into the SwiftUI view hierarchy via `.environment()`. Views consume it with `@Environment(AppState.self)`. Only the root view that owns the instance applies `@State`. Completed chapters are written to `UserDefaults` so progress persists across app launches. Carolina should never lose her place.

```swift
// AppState.swift
import Observation

@MainActor
@Observable
final class AppState {
    var currentChapter: Int = 0
    var completedChapters: Set<Int> = []

    func completeChapter(_ chapter: Int) {
        completedChapters.insert(chapter)
        currentChapter = chapter + 1
        persist()
    }
}
```

**Injection pattern:**

- Root: `@State private var appState = AppState()` → `.environment(appState)`
- Child views: `@Environment(AppState.self) private var appState`
- Two-way bindings: `@Bindable var appState` to derive `$appState.property`

This replaces the legacy `ObservableObject` + `@Published` + `@EnvironmentObject` pattern. The Observation framework provides granular property-level tracking: views only re-render when properties they actually read in their body change, eliminating the over-invalidation behavior of `ObservableObject`.

### State Machine and Resume Semantics

```
Launch → Restore State → Route to Current Chapter → Within-Chapter Flow → Chapter Complete → Next Chapter
```

**Resume behavior on app termination:**

- Chapter completion state is persisted to `UserDefaults`. Within-chapter sub-state is not persisted.
- On relaunch mid-chapter: user resumes at the beginning of the current (incomplete) chapter.
- Exception: Chapter 2 persists per-poem completion state so solved poems are not replayed.
- Chapter 1 game resets on resume (hearts and lives reset to initial values).
- `UserDefaults` corruption or reset: app treats state as fresh, user starts from Chapter 0.
- TestFlight app update: state is preserved (`UserDefaults` survives app updates).

### Dependencies

| Dependency      | Purpose                                            | Fallback on Failure                                                  |
| --------------- | -------------------------------------------------- | -------------------------------------------------------------------- |
| Observation     | `@Observable` macro for state management (iOS 17+) | N/A. Required framework. iOS 18 target guarantees availability.      |
| SpriteKit       | Chapter 1 and Chapter 5 game engine                | Risk accepted. Failure on iPhone 17 Pro not expected.                |
| AVFoundation    | Voice note playback (Ch3), BGM, SFX audio engine   | Display voice note transcript as static text if playback fails.      |
| Speech          | Speech recognition for voice unlock (Ch3)          | Require permission via Settings redirect. See Chapter 3 permissions. |
| CoreHaptics     | Advanced haptic patterns (Ch0 fingerprint scan)    | Degrade to `UIImpactFeedbackGenerator`.                              |
| UIKit (bridged) | UIImpactFeedbackGenerator for standard haptics     | Silent no-op if unavailable.                                         |

No third-party package manager dependencies. Native Apple frameworks only. Zero external packages.

### Technology Stack

| Layer                | Technology                                                                                        |
| -------------------- | ------------------------------------------------------------------------------------------------- |
| UI Framework         | SwiftUI (primary), SpriteKit via `SpriteView` (Ch1, Ch5), UIKit bridge (haptics, settings URL)    |
| State Management     | `@Observable` macro (Observation framework), `@Environment` injection, `@State` ownership         |
| Persistence          | `UserDefaults` (`Set<Int>`, booleans)                                                             |
| Networking           | `URLSession` with `async/await`, `JSONEncoder`/`JSONDecoder`                                      |
| Concurrency          | Swift 6 strict concurrency, `@MainActor` on state types, `Sendable` DTOs, structured `Task`       |
| Audio                | `AVAudioPlayer`, `AVAudioSession` (`.playback` category)                                          |
| Haptics              | `CoreHaptics` (advanced patterns), `UIImpactFeedbackGenerator`, `UINotificationFeedbackGenerator` |
| Speech               | `SFSpeechRecognizer` (on-device, Speech framework)                                                |
| Animation            | SwiftUI `withAnimation`, `KeyframeAnimator`, `PhaseAnimator`, `TimelineView`                      |
| Audio Format         | AAC/M4A (hardware-decoded on Apple silicon, superior compression to MP3)                          |
| Testing              | Swift Testing (`@Test`, `#expect`, `@Suite`) for unit tests, `XCTest` for UI automation           |
| Build Tooling        | Xcode 16+, Swift 6 language mode (strict concurrency enabled), Instruments                        |
| Distribution         | TestFlight                                                                                        |
| Third-Party Packages | None                                                                                              |

**Concurrency model:** All `@Observable` state types are annotated `@MainActor`. API response DTOs conform to `Sendable`. Async work is scoped to view lifecycle via `.task {}` modifier for automatic cancellation on view disappear. The `AnthropicService` struct is `Sendable` by construction (immutable stored properties only).

**Animation system:** All animations use native SwiftUI APIs. `KeyframeAnimator` (iOS 17+) handles complex multi-property animations (vault split, cassette insert, sky illuminate). `PhaseAnimator` handles looping state-driven animations (fingerprint pulse, star dot idle, waving Ottie). `TimelineView` handles frame-accurate animations synced to audio playback (VU meter needles, reel rotation, floating phrases). No Lottie or third-party animation libraries.

---

## Alternatives Considered

### Persistence: UserDefaults vs. SwiftData

**Chosen:** UserDefaults with `Set<Int>` for completed chapters and per-poem state for Chapter 2.

**Alternative: SwiftData.** Advantages: schema migration support, complex queries, type-safe model layer. Rejected: total persisted state is a single set of integers and a handful of booleans. SwiftData introduces model definition overhead, migration complexity, and a Core Data dependency stack for a data model that fits in a single dictionary. UserDefaults is sufficient and simpler.

### Poetry Interaction: Live Anthropic API vs. Pre-Written Response Pool

**Chosen:** Live API call to Claude Haiku for personalized reply and poem continuation.

**Alternative: Pre-written response pool.** Advantages: zero network dependency, zero API key exposure, deterministic behavior, no loading state. Rejected: the emotional impact of receiving a reply that responds to her actual words is qualitatively different from a generic pre-written response. The magic of this chapter depends on the reply feeling real and personal. Accepted trade-off: network dependency with 10-second timeout and graceful fallback message.

### State Management: @Observable vs. ObservableObject

**Chosen:** `@Observable` macro (Observation framework, iOS 17+) with `@Environment` injection.

**Alternative: `ObservableObject` + `@Published` + `@EnvironmentObject`.** This was the original v1.1 design. Rejected: `ObservableObject` uses Combine-backed whole-object observation, causing views to re-render when any `@Published` property changes regardless of which properties the view reads. `@Observable` provides granular property-level tracking, reducing unnecessary re-renders. `@EnvironmentObject` risks runtime crashes on missing injection; `@Environment` with typed lookup is compile-time safe. `@Observable` is Apple's current default for iOS 17+ and the direction of the platform.

### Animation: Native SwiftUI vs. Lottie

**Chosen:** Native SwiftUI animation APIs (`KeyframeAnimator`, `PhaseAnimator`, `TimelineView`, `withAnimation`).

**Alternative: Lottie (third-party).** v1.1 included an `Assets/Lottie/` directory in the project structure, but no specific Lottie animations were defined in any chapter. All declared animations (typewriter effects, vault split, fingerprint fill, cassette insert, constellation drawing, sky illuminate) are implementable with native SwiftUI APIs available on iOS 17+. Lottie would introduce a third-party dependency for zero capability gain. Removed.

### Audio Format: AAC/M4A vs. MP3

**Chosen:** AAC/M4A for all audio assets.

**Alternative: MP3.** v1.1 specified MP3 for all audio. Rejected: Apple silicon hardware-decodes AAC, eliminating CPU overhead during playback. MP3 requires software decode. AAC achieves better compression at equivalent quality (15-25% smaller files on BGM tracks). `AVAudioPlayer` handles M4A transparently; no code changes required. Power efficiency improvement is material during Chapter 3 (continuous voice note + BGM underlay) and Chapter 5 (continuous maze BGM).

### Game Engine: SwiftUI + SpriteKit vs. Pure SpriteKit

**Chosen:** SpriteKit embedded in SwiftUI via `SpriteView` for Chapter 1 and Chapter 5.

**Alternative: Pure SpriteKit for all game scenes.** Advantages: full control over rendering, no SwiftUI-SpriteKit bridge overhead, simpler game loop. Rejected: non-game screens (poems, letters, dossier, constellations) are standard UI and are faster to build in SwiftUI. Embedding SpriteKit via `SpriteView` for the two game chapters keeps the majority of the app in SwiftUI while using SpriteKit where it provides value. Bridge overhead is negligible for these use cases.

---

## 5. Asset Inventory

All assets are AI-generated (GPT-image-1 or Grok Aurora, whichever produces superior output per asset). Background removal and cropping performed manually before import. Assets are exported as PNG with transparent backgrounds unless otherwise noted.

### Ottie Character Assets

| Asset ID               | Description                                                        | Format | Used In                               |
| ---------------------- | ------------------------------------------------------------------ | ------ | ------------------------------------- |
| `ottie-idle`           | Ottie standing, neutral expression, slight smile                   | PNG    | Chapter 0, general                    |
| `ottie-waving`         | Ottie waving hello with one hand raised                            | PNG    | Chapter 0 password screen             |
| `ottie-thinking`       | Ottie with one hand on chin, looking upward                        | PNG    | Chapter 0 password screen (secondary) |
| `ottie-run-01`         | Running frame 1, left foot forward                                 | PNG    | Chapter 1 sprite                      |
| `ottie-run-02`         | Running frame 2, mid stride                                        | PNG    | Chapter 1 sprite                      |
| `ottie-run-03`         | Running frame 3, right foot forward                                | PNG    | Chapter 1 sprite                      |
| `ottie-run-04`         | Running frame 4, recovery step                                     | PNG    | Chapter 1 sprite                      |
| `ottie-jump`           | Ottie mid-jump, arms up, joyful                                    | PNG    | Chapter 1 sprite                      |
| `ottie-hit`            | Ottie stumbling, arms flailing, surprised                          | PNG    | Chapter 1 sprite                      |
| `ottie-celebrate`      | Ottie arms raised, big smile, celebratory                          | PNG    | Chapter 1 game complete               |
| `ottie-game-intro`     | Ottie waving at camera, ready to run                               | PNG    | Chapter 1 intro card                  |
| `ottie-suit-seated`    | Ottie in business suit seated at table, neutral serious expression | PNG    | Chapter 4                             |
| `ottie-suit-pointing`  | Ottie in suit with laser pointer extended, eyebrow raised          | PNG    | Chapter 4                             |
| `ottie-suit-spin`      | Ottie mid-spin in suit, coat flaring                               | PNG    | Chapter 4                             |
| `ottie-suit-soft`      | Ottie in suit, tie loosened, warm gentle expression                | PNG    | Chapter 4 final moment                |
| `ottie-topdown`        | Ottie top-down view, small, walking sprite                         | PNG    | Chapter 5 maze                        |
| `ottie-topdown-walk-u` | Top-down Ottie walking up, 2 frames                                | PNG    | Chapter 5                             |
| `ottie-topdown-walk-d` | Top-down Ottie walking down, 2 frames                              | PNG    | Chapter 5                             |
| `ottie-topdown-walk-l` | Top-down Ottie walking left, 2 frames                              | PNG    | Chapter 5                             |
| `ottie-topdown-walk-r` | Top-down Ottie walking right, 2 frames                             | PNG    | Chapter 5                             |
| `ottie-hug`            | Ottie mid-hug animation frame, arms wrapped around                 | PNG    | Chapter 5 reunion                     |
| `ottie-ending`         | Ottie small, waving gently                                         | PNG    | Ending screen                         |

### Carolina Otter Assets

| Asset ID              | Description                                                            | Format | Used In            |
| --------------------- | ---------------------------------------------------------------------- | ------ | ------------------ |
| `carolina-otter-wait` | Small female-coded otter seated, small heart above head, top-down view | PNG    | Chapter 5 maze end |
| `carolina-otter-hug`  | Carolina otter hugging back, matching ottie-hug frame                  | PNG    | Chapter 5 reunion  |

### Environment and Background Assets

| Asset ID          | Description                                                     | Format | Used In           |
| ----------------- | --------------------------------------------------------------- | ------ | ----------------- |
| `bg-forest-sky`   | Magical forest sky layer, parallax far, soft purples and blues  | PNG    | Chapter 1         |
| `bg-forest-mid`   | Forest midground layer, stylized trees, parallax mid            | PNG    | Chapter 1         |
| `bg-forest-floor` | Seamless scrolling floor tile, grassy forest ground             | PNG    | Chapter 1         |
| `bg-office-room`  | Overhead view of small cozy office, desk, warm light, bookshelf | PNG    | Chapter 4         |
| `bg-maze-gray`    | Top-down tileset for maze section 1, muted blues and grays      | PNG    | Chapter 5         |
| `bg-maze-amber`   | Top-down tileset for maze section 2, warm ambers and oranges    | PNG    | Chapter 5         |
| `bg-maze-gold`    | Top-down tileset for maze section 3, full warm gold tones       | PNG    | Chapter 5         |
| `bg-night-sky`    | Deep navy star-filled sky, rich textured background             | PNG    | Chapter 6, Ending |

### Game Object Assets

| Asset ID              | Description                                                | Format | Used In   |
| --------------------- | ---------------------------------------------------------- | ------ | --------- |
| `heart-collectible`   | Glowing heart, warm gold and rose, small, clearly readable | PNG    | Chapter 1 |
| `obstacle-stormcloud` | Small dark animated storm cloud with lightning detail      | PNG    | Chapter 1 |
| `obstacle-thorn`      | Cluster of thorny vines protruding from ground             | PNG    | Chapter 1 |

### UI Object Assets

| Asset ID             | Description                                                   | Format     | Used In   |
| -------------------- | ------------------------------------------------------------- | ---------- | --------- |
| `fingerprint-svg`    | Clean fingerprint line art SVG, centered                      | SVG        | Chapter 0 |
| `vault-door-top`     | Top half of vault door, thick steel aesthetic                 | PNG        | Chapter 0 |
| `vault-door-bottom`  | Bottom half of vault door, matching bottom steel aesthetic    | PNG        | Chapter 0 |
| `briefcase-idle`     | Small stylized leather briefcase on table                     | PNG        | Chapter 4 |
| `briefcase-shake`    | Same briefcase, slight blur/distortion for shake frame        | PNG        | Chapter 4 |
| `dossier-file`       | Manila folder / classified dossier file, stamped look         | PNG        | Chapter 4 |
| `speech-bubble`      | Clean rounded speech bubble, white, no tail initially         | PNG        | Chapter 4 |
| `cassette-tape`      | Detailed vintage cassette tape, label reads "For Carolina"    | PNG        | Chapter 3 |
| `cassette-reel-l`    | Left reel isolated, for independent rotation animation        | PNG        | Chapter 3 |
| `cassette-reel-r`    | Right reel isolated, for independent rotation animation       | PNG        | Chapter 3 |
| `vu-meter`           | Vintage VU meter panel, two needle gauges                     | PNG        | Chapter 3 |
| `star-dot-idle`      | Small glowing star point, pulsing idle state                  | PNG        | Chapter 6 |
| `star-dot-active`    | Star point after connection, brighter                         | PNG        | Chapter 6 |
| `constellation-line` | Glowing line segment connecting two star dots                 | PNG/Vector | Chapter 6 |
| `memory-card-bg`     | Soft frosted card background for constellation memory reveals | PNG        | Chapter 6 |

### Audio Assets

All audio assets use AAC/M4A format. AAC is hardware-decoded on Apple silicon, reducing CPU overhead and improving power efficiency compared to MP3 (which requires software decode). AAC also achieves better compression at equivalent quality, reducing bundle size.

| Asset ID                 | Description                                                   | Format |
| ------------------------ | ------------------------------------------------------------- | ------ |
| `sfx-fingerprint-scan`   | Subtle pulsing biometric tone, builds slightly over 2 seconds | M4A    |
| `sfx-vault-open`         | Mechanical thunk and hydraulic hiss of vault splitting open   | M4A    |
| `sfx-word-correct`       | Soft chime with gentle resonance, correct word landed         | M4A    |
| `sfx-poem-reveal`        | Gentle whoosh as poems animate into collection                | M4A    |
| `sfx-tape-static`        | Low ambient tape hiss, loopable                               | M4A    |
| `sfx-cassette-click`     | Mechanical cassette engagement click                          | M4A    |
| `sfx-briefcase-shake`    | Rattling briefcase sound, 1 second                            | M4A    |
| `sfx-file-slam`          | File folder slapping onto table                               | M4A    |
| `sfx-speech-bubble-pop`  | Light pop as speech bubble appears                            | M4A    |
| `sfx-footstep-soft`      | Soft padded footstep, 2 variants for alternating              | M4A    |
| `sfx-wall-bump`          | Soft gentle thud, no punishment feel                          | M4A    |
| `sfx-maze-transition`    | Soft chime at section boundary                                | M4A    |
| `sfx-reunion-fanfare`    | Small sweet fanfare, 2 seconds                                | M4A    |
| `sfx-star-connect`       | Soft chime per star dot connection                            | M4A    |
| `sfx-constellation-lock` | Full chord, satisfying resolve                                | M4A    |
| `sfx-sky-illuminate`     | Orchestral swell, biggest sound moment in the app             | M4A    |
| `sfx-heart-collect`      | Light cheerful chime, Chapter 1                               | M4A    |
| `sfx-obstacle-hit`       | Soft thud, non-punishing, Chapter 1                           | M4A    |
| `sfx-game-over`          | Short descending tone, gentle                                 | M4A    |
| `sfx-game-complete`      | Ascending fanfare, 3 seconds                                  | M4A    |
| `sfx-chapter-transition` | Global chapter complete chime                                 | M4A    |
| `sfx-typewriter-tick`    | Single key click, looped for typewriter effect                | M4A    |
| `bgm-game-runner`        | Infinite runner BGM, upbeat, magical forest feel, loopable    | M4A    |
| `bgm-voice-underlay`     | Soft instrumental, fades in halfway through voice note        | M4A    |
| `bgm-maze-gray`          | Ambient, slightly melancholic, sparse                         | M4A    |
| `bgm-maze-amber`         | Warmer ambient, hope building                                 | M4A    |
| `bgm-maze-gold`          | Full warm instrumental, arrival feeling                       | M4A    |
| `bgm-constellation`      | Sparse cinematic, builds to swell at illuminate moment        | M4A    |
| `bgm-ending`             | Quiet warm ambient, lets her sit in the moment                | M4A    |

### Bundle Size Estimate

| Asset Category           | Count | Estimated Size |
| ------------------------ | ----- | -------------- |
| Character PNGs (@3x)     | ~23   | ~1.2 MB        |
| Background PNGs (@3x)    | ~8    | ~4 MB          |
| UI Object PNGs           | ~11   | ~1 MB          |
| SFX (M4A)                | ~22   | ~1.5 MB        |
| BGM (M4A, ~2-3 min each) | ~8    | ~18 MB         |
| Voice note (M4A)         | 1     | ~4 MB          |
| App binary + frameworks  | —     | ~15 MB         |
| **Total estimate**       | —     | **~45 MB**     |

Well under the 200MB TestFlight cellular download limit. AAC encoding yields ~20-25% smaller files than MP3 at equivalent quality.

---

## 6. Chapter 0 — The Handshake

### Purpose

Chapter 0 is the entry gate. It establishes the app's personality immediately: mysterious, playful, intimate. It should feel like a secret only she can open.

### Flow

```
Fingerprint Screen -> [successful scan] -> Vault Split Animation -> Password Screen -> [correct password] -> Chapter 1
```

### Screen: Fingerprint

**Layout:**

- Full dark background (`night-base`)
- "Welcome Carolina" in Display font, centered, top third of screen, `soft-white`
- `fingerprint-svg` centered in the middle of the screen, `rose-blush` color, approximately 180pt wide
- Blinking label at the bottom third: "Hold to enter" in SF Pro Text Regular 15pt, `muted-gray`, blinking animation at 1 second interval

**Interaction:**

- User presses and holds the fingerprint SVG
- On press begin: CoreHaptics pattern fires. A repeating transient pattern with increasing intensity over 2 seconds, simulating a progressive scan
- The fingerprint SVG fills from bottom to top with a `rose-blush` glow over the 2-second hold duration, using a mask animation
- Simultaneously, a circular progress ring appears around the fingerprint in `warm-gold`
- After 2 seconds of continuous hold: scan completes. Single sharp haptic. "Hold to enter" label fades. Vault animation begins
- If user releases early: scan resets smoothly, fingerprint unfills, no error state shown

**Vault Animation:**

- The entire screen splits horizontally at the center
- `vault-door-top` image slides upward off screen (spring animation, 0.6s)
- `vault-door-bottom` image slides downward off screen (spring animation, 0.6s, same timing)
- Underneath: password screen fades in as the vault opens
- `sfx-vault-open` plays on split begin

### Screen: Password

**Layout:**

- Background remains `night-base`
- `ottie-waving` asset, centered, upper half of screen, approximately 200pt tall
- Below Ottie: password input field, centered, rounded rect style, `midnight-navy` fill, `soft-white` text, 48pt height
- Beneath input field: "Only you can open this" in New York Regular 15pt, `muted-gray`, centered
- Bottom of screen: "Psst... Ask Dinesh for the treasure map" in SF Pro Text 13pt, `muted-gray`, centered, subtle opacity (0.6)

**Interaction:**

- Standard iOS secure text entry
- Hardcoded password: `CarolinaLGTM`
- On incorrect entry: input field shakes (horizontal shake animation, 0.3s), light error haptic. No error message shown. The hint at the bottom gently pulses once
- On correct entry: input field glows `success-glow` briefly, satisfying haptic, then transition to Chapter 1 intro

**Keyboard:** System default keyboard. No custom keyboard.

---

## 7. Chapter 1 — The Runner

### Purpose

Chapter 1 is the first real challenge. A playful infinite runner set in a magical forest. The emotional payoff is the letter delivered after she wins. Tone: joyful, slightly tense, triumphant.

### Flow

```
Chapter 1 Intro Card -> Game -> [collect 25 hearts] -> Game Complete Screen -> Letter (typewriter) -> Chapter 2
```

### Screen: Chapter 1 Intro Card

**Layout:**

- `midnight-navy` card, full screen, rounded corners at top only (28pt)
- Chapter label: "Chapter 1" in SF Pro Text Semibold 13pt, `rose-blush`, top left
- Title: "The Runner" in Display Bold 32pt, `soft-white`
- Body: brief description of the game in SF Pro Text 16pt, `muted-gray`
  - "Run through the magical forest. Collect 25 hearts. Dodge the storms and the thorns. Reach the end."
- `ottie-game-intro` asset, right-aligned, overlapping the card edge slightly
- "Play" button, full-width, `rose-blush` fill, `soft-white` label, 56pt height, 20pt corner radius

### Game Engine

Built with SpriteKit embedded in a SwiftUI view via `SpriteView`.

**World:**

- Three-layer parallax scroll
  - Far layer: `bg-forest-sky`, slowest scroll speed
  - Mid layer: `bg-forest-mid`, medium scroll speed
  - Ground layer: `bg-forest-floor`, fastest scroll speed, seamless tile loop
- Ottie runs in place on the left third of the screen. The world moves right to left
- Ground height: fixed at bottom 20% of screen

**Ottie Sprite:**

- 4-frame run cycle: `ottie-run-01` through `ottie-run-04`, cycling at 12fps
- Jump: tap anywhere on screen triggers jump arc. Ottie swaps to `ottie-jump` frame during airtime
- Double jump: not allowed
- On obstacle hit: swaps to `ottie-hit` frame, brief knockback animation, life lost

**Lives:** 3 lives displayed as small hearts in top left HUD. No lives remaining triggers game over.

**Hearts:**

- `heart-collectible` sprites spawn at varying heights in the right side and scroll left
- Ottie collects on overlap
- Counter displayed top center HUD: "hearts collected / 25"
- `sfx-heart-collect` on each collection
- Light haptic on each collection

**Obstacles:**

- `obstacle-stormcloud`: spawns at varying heights, including mid-air
- `obstacle-thorn`: spawns from the ground only, varying widths
- Spawn rate and scroll speed increase progressively in tiers based on hearts collected

| Hearts Collected | Speed Multiplier | Spawn Rate  |
| ---------------- | ---------------- | ----------- |
| 0-5              | 1.0x             | Low         |
| 6-10             | 1.2x             | Low-Medium  |
| 11-15            | 1.4x             | Medium      |
| 16-20            | 1.6x             | Medium-High |
| 21-25            | 1.8x             | High        |

**HUD:**

- Top left: life hearts (3 icons)
- Top center: heart counter
- HUD uses SF Pro Text Semibold 15pt, `soft-white`, slight drop shadow for legibility over scrolling background

### Screen: Game Over

- Dark overlay fades in over frozen game state
- "Game Over" in Display Bold 28pt, `soft-white`
- Ottie hit frame centered
- "Try Again" button, `rose-blush`, full-width
- Retry resets heart count to 0 and resets lives to 3

### Screen: Game Complete

- World stops scrolling
- `ottie-celebrate` asset bounces in center screen
- `sfx-game-complete` plays
- "You made it." in Display Bold 32pt, `soft-white`, fades in
- After 1.5 seconds, transition to letter screen

### Screen: The Letter

**Layout:**

- Background: `parchment` full screen
- Letter text in New York Regular 18pt, `night-base`, generous line height (1.6)
- Text appears via typewriter effect: each character appears at 40ms intervals
- Punctuation adds an additional 200ms pause
- Line breaks add 400ms pause
- Paragraph breaks add 800ms pause

**Letter Text:**

```
When I was building this little game, I kept thinking about how funny it is
that I tried to turn love into something you could win. Collect the hearts,
dodge the obstacles, keep running long enough and eventually you reach the
reward. It felt neat in theory.

But the truth is, with you, I never had to win anything.

You walked into my life and somehow made it warmer, steadier, more meaningful
without even trying. The days feel lighter with you in them. The future feels
less like a question and more like something I actually look forward to. You
didn't take my heart in some dramatic moment. You just kept being you, and one
day I realized it was already yours.

That's my favorite part of us. There's no performance, no pretending, no keeping
score. Just you being yourself and me being quietly grateful that I get to love you.

So yeah, maybe you beat the game. Maybe you collected every heart and made it
to the end.

But the real truth is this:

You won my heart a long time ago.
And unlike this game, that's one level you never have to replay.
```

**Final Line:**
After a 1.5 second pause following the last paragraph, the final line appears in New York Italic 20pt, slightly larger, centered:

```
No checkpoints needed. You already have it forever.
```

This line fades in rather than typewriters. It should feel like a breath, not a keystroke.

After the letter is fully displayed, a subtle "Continue" prompt appears at the bottom in `muted-gray`.

---

## 8. Chapter 2 — The Poems

### Purpose

Chapter 2 is collaborative and playful. She fills in poems, then writes her own, and receives a reply that sounds like Dinesh. Tone: creative, intimate, warm, a little funny.

### Flow

```
Chapter 2 Intro Card -> Poem 1 -> Poem 2 -> Poem 3 -> Poem Collection Reveal -> Her Turn Screen -> Haiku Reply (Stage 1) -> Poem Continuation (Stage 2) -> Chapter 3
```

### Screen: Chapter 2 Intro Card

- "Chapter 2" label
- Title: "The Poems"
- Body: "Some words are missing. You know what belongs there."
- Ottie idle asset

### Screen: Fill-in Poem

**Layout:**

- Full screen scroll view if content exceeds viewport
- Poem text in New York Regular 18pt, `parchment`, on `night-base` background
- Blank words replaced with a pill-shaped swipe widget: `warm-gold` border, current word displayed inside, `soft-white` text
- Multiple blanks per poem, each independently swipeable

**Swipe Mechanic:**

- Swipe up on a blank: cycles to next word in the curated list
- Swipe down on a blank: cycles to previous word
- Words are arranged in a loop so infinite swiping is possible
- When the correct word is selected: pill glows `success-glow`, gentle haptic fires (UIImpactFeedbackGenerator medium), word locks in with a small scale pulse animation
- All blanks must be correct before the "Next" button activates
- "Next" button appears bottom of screen, inactive state `muted-gray`, active state `rose-blush`

**Word Lists per Blank:**

_Poem 1:_

Blank: "there"
Options (in order): refrigerator, raccoon convention, Mars, dentist waiting room, trampoline park, **there**

Blank: "home"
Options: parking garage, IKEA showroom, airport bathroom, submarine, haunted corn maze, **home**

Blank: "heart"
Options: sandwich, left sock, phone charger, pizza crust, tax return, **heart**

Blank: "you"
Options: the mailman, a confused squirrel, my WiFi router, the barista from 2017, a sentient houseplant, **you**

_Poem 2:_

Blank: "think"
Options: sneeze, blink aggressively, microwave something, check my bank app, fight a pigeon, **think**

Blank: "hand"
Options: elbow, WiFi password, grocery list, shoelace, spoon, **hand**

Blank: "alone"
Options: upside down, in a canoe with raccoons, wearing socks with sandals, on hold with customer service, inside a cereal box, **alone**

Blank: "yours"
Options: the neighbor's cat, my gym membership, a random QR code, the office printer, an expired coupon, **yours**

_Poem 3:_

Blank: "name"
Options: tax form, pizza topping, WiFi network, group chat, delivery tracking number, **name**

Blank: "hands"
Options: oven mitts, calculator, TV remote, shopping cart, sandwich press, **hands**

Blank: "steps"
Options: elevator music, grocery receipts, IKEA arrows, loading screen, treadmill, **steps**

Blank: "you"
Options: my dentist, a lost penguin, the customer support bot, a suspiciously friendly pigeon, the self checkout machine, **you**

**Full Poem Texts:**

_Poem 1:_

```
I didn't notice the moment it happened,
no fireworks, no sudden sound,
just the quiet way my world felt softer
whenever you were [there].

You laugh and something in me settles,
like a storm deciding to rest,
and somehow your voice feels warmer
than any place I've ever called [home].

I don't think you realize how often
you cross my mind in the middle of nothing,
how your name feels like a small secret
I carry close to my [heart].

Maybe love isn't loud or dramatic.
Maybe it's just this simple truth
that wherever life takes me,
I hope it keeps taking me back to [you].
```

_Poem 2:_

```
Somewhere between our ordinary days
and the quiet talks that last too long,
you became the person I reach for
without even having to [think].

You don't try to change the world,
you just change the way mine feels,
like suddenly the future isn't heavy
because I imagine it with your [hand] in mine.

There's a calm that follows you,
not loud, not dramatic, just steady,
the kind that makes me believe
I don't have to face life [alone].

And if love is just choosing someone
again and again without hesitation,
then I think my heart made its choice
the moment it recognized [yours].
```

_Poem 3:_

```
Loving you never felt like falling,
it felt like arriving somewhere quiet,
like my life finally exhaled
the moment it found your [name].

You didn't ask for my heart,
you just held it carefully
until it realized on its own
it was safest in your [hands].

I don't know what the future looks like,
but I know how I want to walk into it,
a little braver, a little softer,
with your laughter beside my [steps].

And if there's one truth I carry with me
through every version of tomorrow,
it's that whatever path I follow,
I want it to keep leading me back to [you].
```

### Screen: Poem Collection Reveal

After Poem 3 is completed, all three poems animate into a single scroll view.

- Transition: each poem card slides in from different directions and settles into a vertical stack
- Light `sfx-poem-reveal` plays
- "Your poems" label at the top in SF Pro Semibold 15pt, `rose-blush`
- All three poems visible as a continuous scroll
- After 2 seconds, a "Continue" prompt appears at the bottom

### Screen: Her Turn

- Dark background
- Ottie idle, small, top of screen
- Prompt in New York Regular 20pt, `parchment`, centered:
  "You felt every word. Now write one of your own."
- Multiline text input below, `midnight-navy` background, `soft-white` text, New York Regular 17pt
- Character limit: none enforced, but placeholder text: "Your words..."
- "Send" button, `rose-blush`, bottom of screen
- On send: input is captured and passed to Anthropic API

### API: Haiku Stage 1 (Reply)

**Endpoint:** `POST https://api.anthropic.com/v1/messages`

**Model:** `claude-haiku-4-5-20251001`

**System Prompt:**

```
You are replying as Dinesh, a loving and soft-spoken boyfriend who is a nerdy engineer.
You are deeply in love and your tone is warm, gentle, and sincere.
You call her baby, bebe, mi amor, and my love.
You use soft and loving language throughout.
You do not use em dashes anywhere.
You may use these emojis naturally: 🥰 ♥️ 🤓
Write a short, heartfelt reply to her poem.
3 to 5 lines.
Sound like a real person responding to someone they love, not a formal poem critique.
Match her emotional register and respond with warmth and recognition.
```

**User message:** Her poem text verbatim.

**Max tokens:** 300

### Screen: Haiku Stage 1 Display

- Full screen, warm dark background
- Her poem displayed top half in New York Regular 17pt, `muted-gray`, slight opacity (text she wrote, now displayed back to her as context)
- Reply typewriters in below in New York Regular 18pt, `parchment`
- `sfx-typewriter-tick` plays per character
- After reply is fully displayed, 2 second pause, then transition to Stage 2

### API: Haiku Stage 2 (Continuation)

**System Prompt:**

```
You are Dinesh, a loving and soft-spoken boyfriend who is a nerdy engineer.
You are continuing her poem as if it is one continuous poem written by both of you together.
Pick up from her last line and extend it.
Match her rhythm, her imagery, and her tone exactly.
Write 3 to 5 additional lines that feel like a natural extension of what she wrote.
It should read as one poem, not two separate pieces.
Do not use em dashes anywhere.
Use soft, loving, warm language.
You may use these emojis naturally where appropriate: 🥰 ♥️ 🤓
```

**User message:** Her poem text verbatim.

**Max tokens:** 300

### Screen: Haiku Stage 2 Display

- Her poem fades softly at the top, slightly dimmed
- Below it, a subtle dividing breath (no line, just spacing)
- Haiku continuation typewriters in below her stanza
- Effect: one continuous poem on screen, two halves
- After completion: both halves visible together on screen
- "Continue" prompt fades in after 3 seconds

---

## 9. Chapter 3 — The Cassette

### Purpose

Chapter 3 is the most intimate chapter. Her voice to unlock it. His voice as the reward. Tone: tender, vulnerable, quietly devastating.

### Flow

```
Chapter 3 Intro Card -> Hint Screen -> [she says phrase] -> Voice Unlock Transition -> Tape Playing Screen -> Final Line Display -> Chapter 4
```

### Screen: Chapter 3 Intro Card

- "Chapter 3" label
- Title: "The Cassette"
- Body: "Some things are better said out loud."
- Cassette tape asset displayed below title, static

### Screen: Hint

- Full dark screen
- `ottie-thinking` asset centered
- Text in New York Regular 17pt, `parchment`:
  "Before you press play, there's something you need to say. Go back to Dinesh and ask him about the magic words."
- A "I'm ready" button at the bottom, `midnight-navy` fill, `muted-gray` text, activates microphone

### Interaction: Voice Unlock

- On "I'm ready" tap: microphone and speech recognition permissions requested if not yet granted
  - `NSMicrophoneUsageDescription`: "Ottie needs to hear you say the magic words to unlock this chapter."
  - `NSSpeechRecognitionUsageDescription`: "Ottie is listening for something special."
- On permission denied: display centered screen with text in New York Regular 17pt, `parchment`: "To continue, Ottie needs to hear you. Open Settings to allow microphone access." Below: "Open Settings" button (`rose-blush`) calling `UIApplication.openSettings()`. On return from Settings with permission granted: resume voice unlock flow
- On permission granted: `SFSpeechRecognizer` begins listening
- Ambient display: the cassette tape appears centered, reels animating slowly, `sfx-tape-static` playing softly
- Waveform visualizer beneath the cassette reacts to microphone input (live audio level)
- Recognition target phrase: "I love you baby" (case insensitive, partial match acceptable: "i love you" is sufficient)
- On recognition: waveform surges animation, sustained haptic rumble (CoreHaptics, 0.8s), cassette reels spin up to full speed

### Vault Transition (Cassette Engagement)

- Cassette animates sliding downward as if inserting into a player
- Screen warms from `night-base` to deep `constellation-blue` with amber undertones
- Transition duration: 0.8s
- `sfx-cassette-click` plays on insert

### Screen: Tape Playing

**Layout:**

- VU meter panel (`vu-meter`) top center, needles animated via audio level monitoring
- Cassette with dual reel animation below VU meter
  - Left reel: starts full, depletes as audio plays (scale/mask animation)
  - Right reel: starts empty, fills as audio plays
- Progress bar styled as tape transfer: positioned below cassette
- Timestamps not shown. Progress is visual only

**Voice Note Playback:**

- Audio file: bundled in app bundle (not streamed)
- `AVAudioPlayer` manages playback
- Instrumental underlay (`bgm-voice-underlay`) fades in at the 30-second mark at volume 0.3, does not compete with voice

**Floating Key Phrases:**
Timed to audio playback position. Each phrase fades in and drifts upward slowly before fading out. Timing must be calibrated once voice note is recorded and exact duration is known.

| Phrase                        | Approximate Cue |
| ----------------------------- | --------------- |
| "my person"                   | ~38s            |
| "my peace"                    | ~42s            |
| "a love that feels like home" | ~46s            |
| "the future I want"           | ~52s            |

Phrases rendered in New York Italic 19pt, `warm-gold`, centered.

### Screen: Final Line

After voice note ends:

- VU meters settle to zero (animated)
- Cassette reels stop
- 1.5 second silence
- Final line typewriters in, slower than other typewriter sequences (60ms per character), New York Italic 21pt, `parchment`, centered:

```
If I had to choose again, in every lifetime, I would still find my way to you.
```

- Line remains on screen. Does not fade.
- After 3 seconds, "Continue" prompt appears in `muted-gray` at the bottom

---

## 10. Chapter 4 — The Dossier

### Purpose

Chapter 4 is the comedic peak. After the emotional intensity of Chapter 3, this chapter makes her laugh. Ottie presents a classified surveillance dossier about Carolina. Tone: absurdist, loving, deeply specific.

### Flow

```
Chapter 4 Intro Card -> Scene Entry -> File 01 -> ... -> File 10 -> Final Closing Line -> Chapter 5
```

### Screen: Chapter 4 Intro Card

- "Chapter 4" label
- Title: "The Dossier"
- Body: "Classified intelligence has been gathered. You've been observed."
- `ottie-suit-seated` asset
- "View Files" button

### Screen: Scene Entry

Full scene composited in SwiftUI:

- `bg-office-room` fills the screen
- Wooden table visible in foreground, top-down perspective
- `ottie-suit-seated` sits behind the table, centered
- `briefcase-idle` on the table to the right of Ottie

After 1 second:

- `briefcase-shake` animation plays (scale oscillation 3 times, `sfx-briefcase-shake`)
- Briefcase morphs into `dossier-file` via crossfade + scale transform
- `sfx-file-slam` plays on landing
- File sits on table. Speech bubble appears above Ottie (`sfx-speech-bubble-pop`)
- First evidence file text populates in speech bubble

### Evidence Files

Each file follows this format in the speech bubble:

```
EVIDENCE FILE [number]
CLASSIFICATION: LEVEL HEART
[Subject observation in dry official language]
```

Files are displayed one at a time. Tap anywhere on screen to advance.

| File | Content                                                                                                                                                                                                                                                                            |
| ---- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 01   | Subject displays the ability to sound unusually endearing while expressing frustration. Agent reports difficulty taking grievances seriously due to excessive cuteness. Statements often begin with "I'm actually annoyed" but tone softens mid-sentence. Threat level: disarming. |
| 02   | Subject maintains composed, logical demeanor. Upon receiving minimal affection from Agent, de-escalation of tension is immediate. Response time under 4 seconds. Mechanism: unknown. Effectiveness: 100%.                                                                          |
| 03   | Subject attempts to assert dominance in emotional negotiations while visibly enjoying Agent's attention. Simultaneous eye-rolling and satisfaction detected. Phrase on record: "You're so dramatic," delivered with concealed amusement.                                           |
| 04   | Subject rarely requests reassurance directly. However, specific behavioral signals have been identified and catalogued with high accuracy. Noted indicators include: slower responses, quieter tone, sudden inquiry into Agent's current activities. Detection rate: reliable.     |
| 05   | Subject attempts to suppress laughter when exposed to Agent humor. Resistance duration averages under two seconds before full composure failure. Standard response sequence: "Stop." followed immediately by audible laughter.                                                     |
| 06   | Subject demonstrates increased message processing time when Agent communications carry emotional weight. Longer latency strongly correlates with thoughtful or affectionate reply content. Pattern is consistent and exploitable.                                                  |
| 07   | Subject verbally minimizes impact of Agent's romantic gestures. Behavioral evidence contradicts verbal record. Dismissive statements are frequently followed by renewed references to the same gesture at a later date.                                                            |
| 08   | Subject self-identifies as independent. Simultaneously initiates frequent low-context check-ins with Agent. Primary communication pattern: unsolicited "What are you doing?" messages. Classified as indirect demonstrations of attachment.                                        |
| 09   | Subject exhibits a distinct vocal tone when concealing emotional vulnerability. This tone has been identified as a reliable indicator of deeper sentiment. Initial brief response typically followed by unexpectedly honest or soft admission.                                     |
| 10   | Subject self-identifies as the rational party in the relationship. Analysis indicates predictable deviation from logical framing when emotionally invested. Statements begin analytical. They conclude sentimental. Every time.                                                    |

On final file (10), Ottie's expression shifts from `ottie-suit-pointing` to `ottie-suit-soft`. No tap to advance is needed. After a 1.5 second pause, the dossier file closes, the speech bubble fades, and a final line appears centered on screen in New York Italic 20pt, `parchment`:

```
After reviewing all available evidence, the conclusion is simple.
I'm completely in love with you.
```

Light haptic. 3 second hold. Then "Continue" prompt.

---

## 11. Chapter 5 — The Maze

### Purpose

Chapter 5 is a top-down maze journey through the metaphors of a long-distance relationship. Visual progression from gray and cold to warm gold. Tone: meaningful, quietly emotional, triumphant at the end.

### Flow

```
Chapter 5 Intro Card -> Section 1 (Gray) -> Section 2 (Amber) -> Section 3 (Gold) -> Reunion Cutscene -> Chapter 6
```

### Screen: Chapter 5 Intro Card

- "Chapter 5" label
- Title: "The Way Home"
- Body: "The path isn't always clear. But it always leads somewhere worth it."
- `ottie-topdown` asset, small, centered
- "Find the way" button

### Maze Design

The maze is rendered in a top-down 2D view. The world is illustrated, not clinical. Walls are not gray lines but themed objects that tell a story. The walkable paths are visually distinct and labeled.

**Maze Constraints:**

| Section                   | Grid Size | Dead Ends | Target Solve Time |
| ------------------------- | --------- | --------- | ----------------- |
| Section 1: The Distance   | 8×12      | 2–3       | 45–60 seconds     |
| Section 2: The In-Between | 10×14     | 3–4       | 60–90 seconds     |
| Section 3: The Arrival    | 6×10      | 0–1       | 30–45 seconds     |

Exact maze topology is an implementation detail. Layouts must satisfy the constraints above. Section transitions use a camera fade-to-black (0.5s) with tileset and BGM crossfade.

**Controls:** Swipe-based navigation. Swipe up, down, left, or right to move Ottie one step in that direction. On wall contact: `sfx-wall-bump` plays, gentle haptic, Ottie does not move.

**Ottie Sprite:** 4-directional walking animation using `ottie-topdown-walk-*` frames.

**Section Structure:**

Each section has its own tileset and emotional color language.

_Section 1: The Distance (Gray)_

- Tileset: `bg-maze-gray`
- Muted blues, cool grays, slight fog overlay
- BGM: `bgm-maze-gray`
- Wall obstacles (labeled):
  - Distance Lag
  - WiFi From 2007
  - Overthinking Loop
- Open path labels:
  - Good Morning Calls
  - Soft Reassurance Zone
  - Inside Joke Territory

_Section 2: The In-Between (Amber)_

- Tileset: `bg-maze-amber`
- Warm ambers, orange tones, fog lifts
- BGM crossfades to `bgm-maze-amber`
- `sfx-maze-transition` plays at section boundary
- Wall obstacles (labeled):
  - Sleep Mode Activated
  - Schedule Boss Fight
  - Crash Out Detour
- Open path labels:
  - Future Plans Lane
  - Walk and Talk Moments
  - Laughing For No Reason Path

_Section 3: The Arrival (Gold)_

- Tileset: `bg-maze-gold`
- Full warm gold, no fog, bright open
- BGM crossfades to `bgm-maze-gold`
- Wall obstacles (labeled):
  - Miss You Wave
  - Soft Hours Vulnerability Zone
- Open path labels:
  - Princess Treatment Route
  - Always Us Corridor
- Carolina's otter visible in the distance from section entry, small, with a tiny heart above her head

Final stretch of Section 3: path opens wide, no obstacles. `carolina-otter-wait` centered at the end.

### Reunion Cutscene

When Ottie reaches Carolina's otter:

1. Movement controls disable
2. Ottie auto-walks the final few steps
3. `ottie-hug` and `carolina-otter-hug` assets swap in
4. Heart burst particle effect fires from center (8-12 hearts, arc outward)
5. Camera pulls back slowly to reveal the full maze behind them (scale transform on the world view, 1.5s)
6. BGM fades to quiet
7. One line fades in, New York Italic 20pt, `warm-gold`, centered:

```
Worth every detour.
```

Hold for 3 seconds. "Continue" prompt appears.

---

## 12. Chapter 6 — The Constellations

### Purpose

Chapter 6 is the finale. A dark sky. She draws constellations that trace the story of the relationship. The 6th constellation is the shape of an otter. The sky illuminates. The final message appears. Tone: magical, cinematic, emotional.

### Flow

```
Chapter 6 Intro Card -> Star Field Entry -> Constellation 1 -> 2 -> 3 -> 4 -> 5 -> 6 (Otter reveal) -> Full Sky Illuminate -> Final Message -> Ending Screen
```

### Screen: Chapter 6 Intro Card

- "Chapter 6" label
- Title: "Written in the Stars"
- Body: "Some things were always going to happen."
- Star dot asset against dark background
- "Look up" button

### Screen: Star Field Entry

- Full `night-base` screen
- Stars begin appearing one by one, slow fade-in, over 2 seconds
- `bgm-constellation` begins, sparse and quiet
- Intro line drifts in, New York Regular 18pt, `parchment`:
  "Some things are written in the stars."
- After 1.5 seconds, first constellation dots begin pulsing gently

### Constellation Interaction

**Layout:** Full screen star field. Current active constellation dots pulse. Inactive future constellations invisible until prior one completes.

**Drawing:** User draws a line by dragging from one star dot to the next in sequence. Dots must be tapped or dragged through in order. A `constellation-line` (glowing line segment) renders between connected dots. On final star connection: `sfx-constellation-lock` plays, constellation flashes brightly, name appears above it in SF Pro Semibold 13pt, `warm-gold`.

After lock: memory card slides up from the bottom of the screen, `memory-card-bg` style, text in New York Regular 16pt. She taps to dismiss the card and the next constellation dots appear.

**The 5 Constellations:**

_Constellation 1: The Spark That Found Us_
Memory card:

```
It looked small at the time. Just another message in a sea of them.
But that tiny moment quietly shifted the direction of my life.
Everything we are now traces back to that first hello.
```

_Constellation 2: The Line That Became a Bridge_
Memory card:

```
That was when it stopped feeling like a coincidence and started feeling real.
Our conversations stopped being temporary and began feeling like something steady.
It was the moment I realized this might actually become us.
```

_Constellation 3: The Little Guardian Star_
Memory card:

```
Somehow a tiny otter became part of our world and made it softer.
It turned something playful into something that felt like ours.
A small symbol that quietly held a lot of warmth.
```

_Constellation 4: The Orbit of Laughter_
Memory card:

```
Those jokes became our way of feeling close without trying too hard.
They turned ordinary moments into shared language only we understand.
The kind of laughter that makes a relationship feel like home.
```

_Constellation 5: The Star We Built Together_
Memory card:

```
Claudie isn't just a project or a name.
It's proof of how naturally our worlds started blending together.
Something we created that feels like it belongs to both of us.
```

**The 6th Constellation: The Otter**

After the 5th memory card is dismissed, a new cluster of brighter, slightly trembling star dots appears. No name hint given. She begins connecting them. The shape that forms as she draws is recognizable as an otter silhouette.

On final star connection: constellation flashes, then the name typewriters in above it:

```
Carolina.
```

`sfx-constellation-lock` plays. 2 second hold on that word. Then the sky event begins.

### Full Sky Illuminate

- All 6 constellations flash simultaneously
- `sfx-sky-illuminate` plays (orchestral swell, biggest sound moment in the app)
- Strong satisfying haptic
- Camera slowly zooms out, revealing all constellations in relation to each other
- Screen dims slightly, all constellations glowing at steady warm intensity

### Final Message

Final message typewriters in across the sky, word by word, slower than any previous typewriter sequence (80ms per word), New York Regular 20pt, `parchment`, centered, with slight glow:

```
If someone mapped every step that brought us together,
it would look less like chance and more like a constellation.
Turns out the universe ships us.
```

Hold for 4 seconds after final word. "Continue" prompt fades in, `muted-gray`.

---

## 13. Ending Screen

### Purpose

The landing pad. Warm, quiet. Lets her sit in the moment. Ottie waves. That's enough.

### Layout

- Background: `night-base` with all 6 constellations still faintly visible, very low opacity (0.15)
- `bgm-ending` fades in as constellation BGM fades out
- Centered content stack:
  - "Happy Valentine's Day, Carolina." in New York Regular 28pt, `parchment`
  - Spacing
  - "With love, Dinesh" in New York Italic 20pt, `muted-gray`
- `ottie-ending` asset in bottom right corner, small (80pt), waving gently. Simple 2-frame wave loop

No buttons. No continue. This is the end. She can stay here as long as she wants.

---

## 14. Navigation and Progression

### Chapter Progression

Chapters are sequential and locked. A chapter cannot be started until the previous chapter is marked complete. There is no chapter select, no back navigation between chapters, and no way to skip.

`AppState.completedChapters` persists to `UserDefaults`. Progress survives app termination and relaunch.

### Within-Chapter Navigation

Each chapter manages its own navigation stack internally. No global NavigationStack between chapters. Chapter transitions use a full-screen crossfade with subtle upward drift.

### Chapter Completion

A chapter is marked complete when the user explicitly taps a "Continue" prompt on the chapter's final screen. This prevents accidental chapter completion mid-experience.

---

## 15. Haptics and Sound Design

### Haptic Map

| Interaction                    | Haptic Type                                                    | Intensity           |
| ------------------------------ | -------------------------------------------------------------- | ------------------- |
| Fingerprint scan (progressive) | CoreHaptics transient pattern, repeating, increasing intensity | Low to High over 2s |
| Fingerprint scan complete      | Single sharp impact                                            | Heavy               |
| Wrong password                 | UIImpactFeedbackGenerator                                      | Medium              |
| Correct word in poem           | UIImpactFeedbackGenerator                                      | Medium              |
| Heart collected in game        | UIImpactFeedbackGenerator                                      | Light               |
| Obstacle hit in game           | UIImpactFeedbackGenerator                                      | Medium              |
| Game complete                  | UINotificationFeedbackGenerator                                | Success             |
| Voice phrase recognized        | CoreHaptics sustained                                          | 0.8s Medium         |
| Cassette click                 | UIImpactFeedbackGenerator                                      | Heavy               |
| File slam on table             | UIImpactFeedbackGenerator                                      | Heavy               |
| Constellation lock             | UIImpactFeedbackGenerator                                      | Heavy               |
| Sky illuminate                 | UINotificationFeedbackGenerator                                | Success             |
| Chapter complete               | UINotificationFeedbackGenerator                                | Success             |

### Audio Management

`AudioService` manages all audio playback. Rules:

- Only one BGM track playing at any time. Crossfade on BGM transitions (0.8s)
- SFX can layer on top of BGM
- Voice note (`AVAudioPlayer`) is given exclusive audio session priority during Chapter 3 playback. BGM instrumental underlay plays via a secondary `AVAudioPlayer` at volume 0.3
- All audio respects device silent mode. If device is on silent, haptics remain active
- Audio session category: `.playback` to allow audio to play with screen lock

**Audio Interruption Handling:**

- Register for `AVAudioSession.interruptionNotification`
- On interruption began: pause all playback, freeze VU meter and reel animations (Chapter 3), freeze floating phrase timers
- On interruption ended with `.shouldResume`: resume playback from paused position, resync floating phrase timers to current playback time
- If paused for longer than 5 seconds: show a subtle "Resume" indicator before auto-resuming

---

## 16. API Integration

### Anthropic API

Used exclusively in Chapter 2 for the Haiku poetry exchange.

**Service class:** `AnthropicService.swift`

```swift
// Services/AnthropicService.swift
// URLSession async/await wrapper for Anthropic /v1/messages endpoint

struct AnthropicService: Sendable {
    private let apiKey: String = "[ANTHROPIC_API_KEY]"
    private let endpoint = "https://api.anthropic.com/v1/messages"
    private let model = "claude-haiku-4-5-20251001"

    func generateReply(to poem: String) async throws -> String
    func generateContinuation(of poem: String) async throws -> String
}
```

`AnthropicService` is `Sendable` by construction (all stored properties are immutable `let` constants). Network calls use `URLSession.shared.data(for:)` with `async/await`. Response DTOs are `Sendable` structs decoded via `JSONDecoder`.

**API Key:** Hardcoded in `AnthropicService.swift`. This is acceptable given the app is private, single-user, and not App Store distributed. Do not commit the key to any public repository.

**Error handling:** If either API call fails, display a fallback message in the typewriter format:

```
bebe, your words were so beautiful I'm still processing them. 🥰
```

This maintains tone without breaking the experience.

**Loading state during API calls:** The typewriter cursor blinks on an empty screen for up to 10 seconds before fallback triggers. No spinner. No loading text.

### Accepted Security Risks

| Risk                                          | Status   | Mitigation                                                                                                                                                           |
| --------------------------------------------- | -------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| API key hardcoded in `AnthropicService.swift` | Accepted | $5/month spend cap on Anthropic account. Usage monitored via Anthropic dashboard. Key is extractable from binary but exposure is limited to a single low-cost model. |
| Password visible in this design document      | Accepted | Document is private. Do not commit to any public repository. Single-implementer context.                                                                             |
| User poem text transmitted to Anthropic API   | Accepted | Data sent over HTTPS. No PII beyond creative text. Anthropic data retention policy applies.                                                                          |
| No certificate pinning on API calls           | Accepted | Standard `URLSession` TLS validation. MITM risk negligible for a private gift app on a personal device.                                                              |

---

## Rollout Plan

### Phase 1: Developer Testing

- Full playthrough on physical iPhone 17 Pro
- Verify all chapters end-to-end: fingerprint scan, vault animation, game completion, poem interaction, voice recognition, cassette playback, dossier sequence, maze navigation, constellation drawing, sky illumination, ending screen
- Verify Chapter 2 API integration (both reply and continuation calls)
- Verify Chapter 3 voice recognition with target phrase in quiet environment
- Verify audio interruption handling (simulate incoming call during Chapter 3 voice note)
- Verify state persistence: force-quit mid-chapter, relaunch, confirm resume at chapter start

### Phase 2: TestFlight Internal Build

- Build and upload to TestFlight
- Install on target iPhone 17 Pro
- Full regression pass: all chapters, all interactions, all haptics, all audio
- Verify bundle size matches estimate (~53 MB)
- Verify TestFlight expiration date covers delivery window

### Phase 3: Delivery

- Send TestFlight invite to Carolina
- Confirm successful installation and app launch

### Rollback

- If critical bug reported: push hotfix TestFlight build within 24 hours
- TestFlight supports multiple builds; new build replaces previous automatically on next launch

---

## Open Questions

| #   | Question                                                                                                                                                                                                                 | Owner       | Target Date            |
| --- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ----------- | ---------------------- |
| 1   | Voice note for Chapter 3 is not yet recorded. Floating phrase timings depend on final audio duration and must be calibrated post-recording.                                                                              | Dinesh      | Before Phase 1 testing |
| 2   | Constellation dot positions for constellations 1-5 are unspecified. Shapes to be designed during implementation. Constellation 6 (otter silhouette) requires recognizable outline at small scale.                        | Implementer | During implementation  |
| 3   | Verify `SFSpeechRecognizer` on-device recognition availability for English on iPhone 17 Pro / iOS 18. If on-device is unavailable, speech recognition requires network access.                                           | Implementer | Phase 1 testing        |
| 4   | Background music and SFX tracks need to be sourced or produced. No audio files currently exist. Confirm source: royalty-free library, commissioned, or self-produced. Final files must be delivered in M4A (AAC) format. | Dinesh      | Before Phase 1 testing |

---

_Ottie Express — Design Document v1.2_
_Built with love, for Carolina._
