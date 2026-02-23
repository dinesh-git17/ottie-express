# Epic Granularity

Sizing rules, file-disjointness enforcement, and deliverable definitions for epics within each phase.

## Epic Sizing Rules

### Duration Bounds

| Bound   | Limit                 | Enforcement                             |
| ------- | --------------------- | --------------------------------------- |
| Minimum | 0.5 days (4 hours)    | Below this, merge into an adjacent epic |
| Target  | 1-2 days (8-16 hours) | Preferred epic size                     |
| Maximum | 3 days (24 hours)     | Above this, split into smaller epics    |

An epic exceeding 3 days indicates mixed concerns. Decompose by:

1. Separating data layer work from view layer work
2. Separating static UI from interactive behavior
3. Separating animation/motion from layout
4. Separating service integration from core logic

### File Scope Bounds

| Bound   | Limit     | Enforcement                      |
| ------- | --------- | -------------------------------- |
| Minimum | 1 file    | Valid for single-component epics |
| Target  | 3-8 files | Preferred file count per epic    |
| Maximum | 12 files  | Above this, split into sub-epics |

### Deliverable Requirement

Every epic MUST produce exactly one of:

| Deliverable Type       | Definition                                            | Example                                          |
| ---------------------- | ----------------------------------------------------- | ------------------------------------------------ |
| Runnable component     | A view or scene that renders and responds to input    | Chapter 0 fingerprint screen                     |
| Working service        | A service with public interface and verified behavior | HapticService with UIImpactFeedbackGenerator     |
| Passing test suite     | Tests that validate a unit of behavior                | AppState persistence round-trip tests            |
| Complete data layer    | Models, constants, or tokens ready for consumption    | Design system color and typography tokens        |
| Integrated interaction | End-to-end flow through multiple connected systems    | Chapter 2 poem submission through API to display |

An epic without a named deliverable is not an epic. It is undefined work.

## File-Disjointness Enforcement

### Within-Phase Rule

Two epics in the same phase MUST NOT modify the same file.

**Verification procedure:**

1. List all files each epic will create or modify
2. Compute the intersection of file sets between all epic pairs in the phase
3. If any intersection is non-empty, restructure:
   - Extract the shared file's changes into a separate preparatory epic
   - OR reassign one of the conflicting epics to a different phase

### Cross-Phase Rule

A later phase MAY modify a file created in an earlier phase, but:

- The modification must be in its own dedicated epic (not embedded in a feature epic)
- The modification must be additive (extending an interface, adding a method) not destructive (rewriting existing logic)

### Common Violation Patterns

| Violation                | Description                                                | Resolution                                                              |
| ------------------------ | ---------------------------------------------------------- | ----------------------------------------------------------------------- |
| Shared service extension | Two feature epics both need to add methods to AudioService | Create a service-extension epic in a prior phase that adds both methods |
| Shared view modification | Two epics both modify a shared UI component                | Extract shared component changes into an infrastructure epic            |
| Asset catalog conflict   | Two epics both add assets to the same asset catalog        | Dedicate one epic to all asset imports for that phase, run it first     |

## Epic Decomposition by Chapter Type

### Simple Chapter (Chapter 0, Chapter 4, Ending Screen)

Pure SwiftUI, no game engine, no external APIs, no speech recognition.

**Standard decomposition:**

| Epic | Scope                     | Deliverable                                                  |
| ---- | ------------------------- | ------------------------------------------------------------ |
| 1    | Static layout and assets  | All screens render with correct layout, typography, colors   |
| 2    | Interaction and animation | All interactions work (taps, holds, transitions, typewriter) |
| 3    | State integration         | Chapter completion wired to AppState, navigation works       |

### Game Chapter (Chapter 1, Chapter 5)

SpriteKit via SpriteView, physics, sprites, HUD overlay.

**Standard decomposition:**

| Epic | Scope                      | Deliverable                                                |
| ---- | -------------------------- | ---------------------------------------------------------- |
| 1    | Scene scaffold and world   | SKScene subclass, parallax layers or tileset, camera       |
| 2    | Player sprite and controls | Sprite animation, input handling (tap/swipe), physics body |
| 3    | Game objects and spawning  | Collectibles, obstacles, spawn logic, collision detection  |
| 4    | HUD and game state         | Score display, life tracking, game over, game complete     |
| 5    | Post-game screens          | Letter/cutscene, state integration, chapter completion     |

### API Chapter (Chapter 2)

Network calls, loading states, fallback behavior, multi-screen flow.

**Standard decomposition:**

| Epic | Scope                       | Deliverable                                                |
| ---- | --------------------------- | ---------------------------------------------------------- |
| 1    | Fill-in poem screens        | Poem text, swipe mechanic, word validation, per-poem state |
| 2    | Poem collection and input   | Collection reveal animation, her-turn text input screen    |
| 3    | API integration and display | AnthropicService calls, typewriter reply display, fallback |
| 4    | State and flow integration  | Per-poem persistence, chapter completion, navigation       |

### Media Chapter (Chapter 3)

Speech recognition, audio playback, VU meter, synchronized floating text.

**Standard decomposition:**

| Epic | Scope                       | Deliverable                                                      |
| ---- | --------------------------- | ---------------------------------------------------------------- |
| 1    | Hint screen and permissions | Permission request flow, settings redirect, UI                   |
| 2    | Voice recognition           | SFSpeechRecognizer setup, phrase matching, unlock transition     |
| 3    | Tape playback UI            | VU meter animation, reel animation, cassette visual              |
| 4    | Audio playback and sync     | Voice note player, BGM underlay, floating phrases timed to audio |
| 5    | Final line and completion   | Post-playback typewriter, state integration                      |

### Constellation Chapter (Chapter 6)

Star field rendering, drag-to-connect interaction, sequential reveals, cinematic finale.

**Standard decomposition:**

| Epic | Scope                             | Deliverable                                                        |
| ---- | --------------------------------- | ------------------------------------------------------------------ |
| 1    | Star field and constellation data | Star positions, constellation definitions, rendering               |
| 2    | Drawing interaction               | Drag-to-connect mechanic, line rendering, completion detection     |
| 3    | Memory cards and sequencing       | Card content, slide-up animation, sequential constellation unlock  |
| 4    | Sky illuminate and finale         | Full-sky animation, final message typewriter, transition to ending |

## Naming Convention

Epic names follow this format:

```
E{phase}.{sequence}: {imperative verb} {scope}
```

Examples:

- `E0.1: Scaffold Xcode project structure`
- `E0.2: Define design system tokens`
- `E3.1: Build Chapter 0 fingerprint screen layout`
- `E3.2: Implement Chapter 0 interactions and animations`
