# Phase Classification

Canonical phase types, ordering constraints, and mapping to the Ottie Express design document.

## Phase Types

### Type I: Infrastructure Phase

Foundational systems with zero feature dependencies. These are DAG roots.

**Characteristics:**

- Produces interfaces, protocols, or base types consumed by all subsequent phases
- Contains no user-visible behavior
- Produces artifacts verifiable in isolation (compiles, tests pass, types resolve)

**Canonical infrastructure work units for Ottie Express:**

| Work Unit              | Design Doc Section | Produces                                            |
| ---------------------- | ------------------ | --------------------------------------------------- |
| Xcode project scaffold | §4 Architecture    | Directory structure, build settings, asset catalogs |
| Design system tokens   | §2 Design System   | Color constants, typography modifiers, spacing      |
| AppState + persistence | §4 State Mgmt      | Observable state, UserDefaults read/write, restore  |
| Navigation shell       | §14 Navigation     | Chapter routing, progression gating, transitions    |
| Shared UI components   | §4 SharedUI        | TypewriterText, ChapterCard, HapticButton           |

**Ordering rule:** All infrastructure work units are peers (no internal ordering) unless one consumes another's output. Design tokens must exist before SharedUI components that reference them.

### Type F: Feature Phase

User-facing functionality scoped to a single chapter or screen.

**Characteristics:**

- Depends on infrastructure outputs (tokens, state, navigation, shared UI)
- Self-contained within a chapter directory
- May depend on services (audio, haptics, API) but does not define them
- Deliverable is a working chapter playable end-to-end

**Canonical feature work units for Ottie Express:**

| Work Unit     | Design Doc Section | Dependencies                                                                              |
| ------------- | ------------------ | ----------------------------------------------------------------------------------------- |
| Chapter 0     | §6                 | Design tokens, AppState, navigation, shared UI, HapticService (CoreHaptics)               |
| Chapter 1     | §7                 | Design tokens, AppState, navigation, shared UI, SpriteKit engine, AudioService            |
| Chapter 2     | §8                 | Design tokens, AppState, navigation, shared UI, AnthropicService, AudioService            |
| Chapter 3     | §9                 | Design tokens, AppState, navigation, shared UI, AudioService (advanced), Speech framework |
| Chapter 4     | §10                | Design tokens, AppState, navigation, shared UI, AudioService, HapticService               |
| Chapter 5     | §11                | Design tokens, AppState, navigation, shared UI, SpriteKit, AudioService (crossfade)       |
| Chapter 6     | §12                | Design tokens, AppState, navigation, shared UI, AudioService, HapticService               |
| Ending Screen | §13                | Design tokens, AppState, AudioService                                                     |

**Ordering rule:** Feature phases are ordered by dependency complexity (ascending) and system overlap:

1. Chapters that use only basic infrastructure ship first (Chapter 0, Chapter 4)
2. Chapters that introduce new system capabilities ship next (Chapter 1 introduces SpriteKit, Chapter 2 introduces API)
3. Chapters that combine multiple complex systems ship last (Chapter 3, Chapter 5, Chapter 6)

This ordering validates simpler patterns before complex chapters compound them.

### Type G: Integration Phase

Cross-cutting system wiring, service hardening, and polish.

**Characteristics:**

- Depends on both infrastructure and feature outputs
- Modifies or extends service interfaces defined in infrastructure
- Produces behavior observable across multiple chapters
- Often addresses non-functional requirements (performance, resilience, interruption handling)

**Canonical integration work units for Ottie Express:**

| Work Unit                      | Design Doc Section | Scope                                      |
| ------------------------------ | ------------------ | ------------------------------------------ |
| AudioService session lifecycle | §15 Audio Mgmt     | Interruption handling, crossfade, resume   |
| HapticService pattern library  | §15 Haptic Map     | Full haptic map wiring across all chapters |
| API resilience + fallback      | §16 API            | Timeout, fallback message, error handling  |
| Full-app state persistence     | §4 State Machine   | Resume semantics, corruption recovery      |
| Performance and bundle audit   | Goals table        | 60fps validation, bundle size verification |

## Dependency Ordering Algorithm

Given classified work units, determine phase assignment:

```
1. Place all Type I units with zero dependencies in Phase 0
2. Place remaining Type I units in Phase 1 (those depending on Phase 0 outputs)
3. For each Type F unit:
   a. Compute its dependency set (which infrastructure/service outputs it needs)
   b. Assign to the earliest phase where ALL dependencies are satisfied
   c. If two Type F units have no dependency relationship, they MAY share a phase
4. For each Type G unit:
   a. Compute its dependency set (which features and infrastructure it wires)
   b. Assign to the earliest phase AFTER all its dependencies are satisfied
5. Validate: no cycles exist. If Phase N depends on Phase M where M > N, restructure.
```

## Phase Numbering Convention

| Phase Range | Class          | Description                     |
| ----------- | -------------- | ------------------------------- |
| 0           | Infrastructure | Zero-dependency foundation      |
| 1           | Infrastructure | Foundation depending on Phase 0 |
| 2-N         | Feature        | Chapters, ordered by complexity |
| N+1         | Integration    | Cross-cutting wiring and polish |
| N+2         | Integration    | Final hardening, performance    |

Phase numbers are sequential with no gaps. Each phase has exactly one class label.
