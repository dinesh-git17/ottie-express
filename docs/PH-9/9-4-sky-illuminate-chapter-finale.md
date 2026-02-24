# E9.4: Build Sky Illuminate and Chapter Finale

**Phase:** 9 - Chapter 6: The Constellations
**Class:** Feature
**Design Doc Reference:** §12 Chapter 6 (Full Sky Illuminate, Final Message, Chapter 6 Intro Card), §14 Navigation and Progression (chapter completion semantics), §15 Haptics and Sound Design (sky illuminate haptic), §2 Design System (color palette, typography, motion principles)
**Dependencies:**

- Phase 2: Navigation Shell and Shared UI (exit criteria met)
- E2.2: Build TypewriterText component (word-by-word typewriter rendering available)
- E2.3: Build ChapterCard component (intro card rendering available)
- E9.1: Build star field and constellation data model (ConstellationData positions, StarFieldView, Chapter6Constants available)
- E9.2: Implement constellation drawing interaction (ConstellationDrawingView rendering available for sky illuminate display)
- E9.3: Build memory cards and sequential constellation unlock (`onAllComplete` signal from ConstellationSequencer available)
- E1.1: Define design system tokens (color tokens `parchment`, `muted-gray`, `warm-gold`; typography New York Regular 20pt; motion constants)
- E1.2: Implement AppState and persistence (`completeChapter(6)` interface available)
- E1.3: Build AudioService (BGM playback for `bgm-constellation`, SFX playback for `sfx-sky-illuminate`, crossfade interface)
- E1.4: Build HapticService (success notification haptic available per §15)

---

## Goal

Implement the sky illuminate sequence, the word-by-word final message typewriter, the Chapter 6 intro card, and wire the complete Chapter 6 flow from intro through constellation interactions to chapter completion.

---

## Scope

### File Inventory

| File                                                     | Action | Responsibility                                                                                                                                                    |
| -------------------------------------------------------- | ------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `OttieExpress/Chapters/Chapter6/SkyIlluminateView.swift` | Create | Render the full sky illuminate sequence: simultaneous 6-constellation flash, camera zoom-out revealing spatial layout, screen dim, and steady warm glow state     |
| `OttieExpress/Chapters/Chapter6/FinalMessageView.swift`  | Create | Render three-line final message with word-by-word typewriter at 80ms interval, `parchment` text with glow, 4-second hold, and `muted-gray` "Continue" fade-in     |
| `OttieExpress/Chapters/Chapter6/Chapter6View.swift`      | Create | Orchestrate the complete Chapter 6 flow: intro card, star field entry, constellation sequence, sky illuminate, final message, and chapter completion via AppState |

### Integration Points

**SkyIlluminateView.swift**

- Imports from: `Chapter6Constants.swift` (E9.1), `ConstellationData.swift` (E9.1), `OttieExpress/Extensions/Color+DesignTokens.swift`
- Imported by: `Chapter6View.swift`
- State reads: All 6 constellation definitions from ConstellationData (positions and connection data for rendering)
- State writes: None (exposes `onComplete` callback when illuminate animation finishes)

**FinalMessageView.swift**

- Imports from: `Chapter6Constants.swift` (E9.1), `OttieExpress/Extensions/Color+DesignTokens.swift`, `OttieExpress/Extensions/Font+DesignTokens.swift`
- Imported by: `Chapter6View.swift`
- State reads: None
- State writes: None (exposes `onContinue` callback when user taps Continue)

**Chapter6View.swift**

- Imports from: `StarFieldView.swift` (E9.1), `Chapter6Constants.swift` (E9.1), `ConstellationSequencer.swift` (E9.3), `SkyIlluminateView.swift`, `FinalMessageView.swift`, `OttieExpress/SharedUI/ChapterCard.swift` (E2.3), `OttieExpress/Services/AudioService.swift` (E1.3), `OttieExpress/Services/HapticService.swift` (E1.4)
- Imported by: `OttieExpress/App/ChapterRouter.swift` (E2.1)
- State reads: `AppState.currentChapter`, `AppState.completedChapters`
- State writes: `AppState` via `completeChapter(6)`

---

## Out of Scope

- Constellation data model, star positions, and memory card text definitions (covered by E9.1)
- Drag-to-connect drawing mechanic and per-connection line rendering (covered by E9.2)
- Memory card slide-up/dismiss behavior and sequential unlock state machine (covered by E9.3)
- Ending screen layout, waving Ottie animation, and ending text (covered by Phase 10, E10.1)
- BGM crossfade from `bgm-constellation` to `bgm-ending` (covered by E10.2)
- Audio interruption handling and resume indicator logic (covered by Phase 11, E11.1)
- 60fps animation performance profiling on physical device (deferred to integration testing in Phase 12)

---

## Definition of Done

- [ ] Intro card renders using ChapterCard component with "Chapter 6" label, "Written in the Stars" title, "Some things were always going to happen." body, star dot asset against dark background, and "Look up" button
- [ ] Tapping "Look up" transitions from intro card to star field entry sequence
- [ ] `bgm-constellation` begins playing via AudioService when the star field appears
- [ ] After ConstellationSequencer fires `onAllComplete`, all 6 constellations flash simultaneously in SkyIlluminateView
- [ ] `sfx-sky-illuminate` plays via AudioService during the simultaneous flash
- [ ] Success notification haptic fires via HapticService (`UINotificationFeedbackGenerator` success) at the moment of sky illuminate per §15 haptic map
- [ ] Camera zoom-out animation scales from 1.0 to approximately 0.7 over 2-3 seconds using spring animation, revealing all 6 constellations in spatial relation
- [ ] Screen dims slightly (background opacity reduces) and all constellations stabilize at steady warm glow intensity after zoom-out completes
- [ ] Final message renders word-by-word at 80ms per word interval across three lines: "If someone mapped every step that brought us together," / "it would look less like chance and more like a constellation." / "Turns out the universe ships us."
- [ ] Final message text renders in New York Regular 20pt, `parchment` (#FAF3E0), centered, with a subtle outer glow effect
- [ ] 4-second hold occurs after the last word ("us.") renders, with no interactive elements visible during the hold
- [ ] "Continue" prompt fades in using `muted-gray` (#8A8A9A) color after the 4-second hold completes
- [ ] Tapping "Continue" calls `appState.completeChapter(6)`, persisting chapter 6 completion to UserDefaults
- [ ] Chapter6View orchestrates the complete phase sequence: `.intro` -> `.starField` -> `.constellations` -> `.skyIlluminate` -> `.finalMessage` -> `.complete`, with each transition driven by child view completion callbacks

---

## Implementation Notes

Chapter6View owns a phase state machine with six states: `.intro`, `.starField`, `.constellations`, `.skyIlluminate`, `.finalMessage`, `.complete`. Each state maps to a single child view. Transitions are driven by completion callbacks, not timers. The view body switches on the current phase and renders the corresponding child.

Intro card per §12 Screen: Chapter 6 Intro Card. Use the shared ChapterCard component (E2.3) with configuration: label "Chapter 6", title "Written in the Stars", body "Some things were always going to happen.", button label "Look up". The star dot asset renders against the dark card background.

BGM: AudioService.playBGM("bgm-constellation") fires on transition from `.intro` to `.starField`. The crossfade from `bgm-constellation` to `bgm-ending` is not this epic's responsibility; it is handled by E10.2 when the ending screen mounts.

Sky illuminate per §12 Full Sky Illuminate: "All 6 constellations flash simultaneously" followed by "Camera slowly zooms out, revealing all constellations in relation to each other." SkyIlluminateView renders all 6 constellations using their position data from ConstellationData. The flash is a 0.3s opacity pulse (0 -> 1 -> steady). The zoom-out applies `.scaleEffect` transitioning from 1.0 to 0.7 with a slow spring (stiffness 80, damping 18) over 2-3 seconds. Screen dim applies a `.night-base` overlay at 0.3 opacity. Steady glow: all constellation lines render at `warm-gold` with `.shadow(color: .warmGold.opacity(0.4), radius: 6)`.

Sky illuminate haptic: §15 haptic map "Sky illuminate" row specifies `UINotificationFeedbackGenerator` success type. Fire at the exact frame the flash begins.

Final message per §12: "word by word, slower than any previous typewriter sequence (80ms per word)." FinalMessageView splits the three-line message into individual words and reveals them sequentially at 80ms intervals using a `Task`-based timer loop. This is word-level reveal, distinct from TypewriterText's character-level reveal (E2.2). The three lines from §12:

```
If someone mapped every step that brought us together,
it would look less like chance and more like a constellation.
Turns out the universe ships us.
```

The glow effect uses `.shadow(color: .parchment.opacity(0.3), radius: 8)` on the text view. The 4-second hold is a `Task.sleep(for: .seconds(4))` after the last word renders. The "Continue" prompt fades in with a 0.6s opacity transition using `muted-gray` (#8A8A9A).

Chapter completion per §14: "A chapter is marked complete when the user explicitly taps a Continue prompt on the chapter's final screen." The Continue tap calls `appState.completeChapter(6)`, which inserts 6 into `completedChapters`, advances `currentChapter` to 7, and persists to UserDefaults per the §4 AppState contract.
