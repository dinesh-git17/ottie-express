# E4.3: Build Chapter 4 Flow Coordinator

**Phase:** 4 - Chapter 4: The Dossier
**Class:** Feature
**Design Doc Reference:** SS10 (Chapter 4 Flow, Intro Card, Final Closing Line), SS4 (State Management, State Machine and Resume Semantics), SS14 (Navigation and Progression, Chapter Completion), SS2 (Typography, Color Palette)
**Dependencies:**

- Phase 2: Navigation Shell and Shared UI (exit criteria met)
- E1.1: Define design system tokens (color, typography, spacing, and motion constants available)
- E1.2: Implement AppState and persistence (`completeChapter(_:)` method and `completedChapters` set available)
- E1.4: Build HapticService (light impact pattern callable for final line haptic)
- E2.1: Build navigation shell (ChapterRouter routes to Chapter4View when `currentChapter == 4`)
- E2.2: Build TypewriterText component (typewriter animation for final closing line)
- E2.3: Build ChapterCard component (intro card rendering with chapter label, title, body, asset, button)
- E4.1: Build dossier scene layout and evidence data (`DossierSceneView`, `EvidenceFile.allFiles`, `Chapter4Constants`)
- E4.2: Build speech bubble and briefcase animations (`BriefcaseAnimation`, `SpeechBubbleView`)
- Asset: `ottie-suit-seated` (available in asset catalog)

---

## Goal

Wire the intro card, scene entry, briefcase animation, evidence file progression, final closing line, and chapter completion into a sequential Chapter 4 flow that marks chapter completion in AppState and integrates with ChapterRouter for forward navigation.

---

## Scope

### File Inventory

| File | Action | Responsibility |
| --- | --- | --- |
| `OttieExpress/Chapters/Chapter4/Chapter4View.swift` | Create | Flow coordinator managing sequential phase transitions between intro card, scene entry, briefcase animation, evidence file progression, closing line display, and AppState chapter completion |

### Integration Points

**Chapter4View.swift**

- Imports from: `DossierSceneView.swift` (E4.1), `EvidenceFile.swift` (E4.1), `Chapter4Constants.swift` (E4.1), `BriefcaseAnimation.swift` (E4.2), `SpeechBubbleView.swift` (E4.2), `ChapterCard.swift` (E2.3), `TypewriterText.swift` (E2.2), `HapticService.swift` (E1.4), `Color+DesignTokens.swift` (E1.1), `Font+DesignTokens.swift` (E1.1)
- Imported by: `ChapterRouter.swift` (E2.1)
- State reads: `AppState.completedChapters` (to verify chapter gating on relaunch)
- State writes: `AppState.completedChapters`, `AppState.currentChapter` (via `appState.completeChapter(4)`)

---

## Out of Scope

- Scene layout compositing, background rendering, and Ottie sprite positioning (covered by E4.1)
- Evidence file data model and text content (covered by E4.1)
- Briefcase shake animation, morph sequence, and related SFX (covered by E4.2)
- Speech bubble rendering and tap-to-advance file progression (covered by E4.2)
- Chapter 5 content, layout, or initialization (covered by Phase 8)
- Cross-chapter transition animation from Chapter 4 to Chapter 5 (handled by ChapterRouter in E2.1; this epic triggers the AppState mutation, the router handles the visual transition)
- Within-chapter sub-state persistence across app relaunch (SS4 state machine specifies incomplete chapters reset to the intro card on relaunch)

---

## Definition of Done

- [ ] Intro card renders with "Chapter 4" label, "The Dossier" title, SS10 body copy ("Classified intelligence has been gathered. You've been observed."), `ottie-suit-seated` asset, and "View Files" button using the ChapterCard component (E2.3)
- [ ] Tapping "View Files" transitions from the intro card to the dossier scene entry with a crossfade and upward drift animation (0.4s per SS2 screen transition)
- [ ] Scene entry displays `DossierSceneView` with `ottie-suit-seated` and visible briefcase, then triggers `BriefcaseAnimation` after the 1-second delay
- [ ] After briefcase morph completes, speech bubble appears and evidence file 1 populates via `SpeechBubbleView`
- [ ] `currentFileIndex` state increments on each tap-to-advance, driving the speech bubble to display the next evidence file
- [ ] Ottie expression parameter updates from `ottie-suit-pointing` to `ottie-suit-soft` on `DossierSceneView` when `currentFileIndex` reaches file 10
- [ ] After `onAllFilesViewed` fires (post-file-10 1.5-second pause), the dossier file fades out, the speech bubble fades out, and the scene transitions to the closing line display
- [ ] Final closing line renders centered on screen using TypewriterText (E2.2) in New York Italic 20pt, `parchment` (#FAF3E0), displaying both lines from SS10: "After reviewing all available evidence, the conclusion is simple." followed by "I'm completely in love with you."
- [ ] Light impact haptic fires via HapticService when the final closing line begins its typewriter reveal
- [ ] "Continue" prompt renders in SF Pro Text Regular 15pt, `muted-gray` (#8A8A9A), after a 3-second hold following the closing line typewriter completion
- [ ] Tapping "Continue" calls `appState.completeChapter(4)`, inserting `4` into `completedChapters` and advancing `currentChapter`
- [ ] ChapterRouter observes the `currentChapter` change and navigates away from Chapter4View to the next chapter view
- [ ] Relaunching the app after completing Chapter 4 routes directly to the current incomplete chapter, bypassing Chapter4View entirely

---

## Implementation Notes

Chapter4View manages a private `@State` enum representing the active phase of the flow:

```swift
private enum Chapter4Phase {
    case intro
    case scene
    case briefcase
    case evidence
    case closingLine
}
```

The view body switches on this enum:

- `.intro`: Renders `ChapterCard` with `Chapter4Constants.introCardTitle`, `Chapter4Constants.introCardBody`, `ottie-suit-seated` asset, and "View Files" action. Tapping the action advances to `.scene`.
- `.scene`: Renders `DossierSceneView(ottieAsset: "ottie-suit-seated", showBriefcase: true)`. After layout, immediately transitions to `.briefcase`.
- `.briefcase`: Overlays `BriefcaseAnimation(onMorphComplete: { phase = .evidence })` on the scene. Hides the static briefcase on `DossierSceneView` by setting `showBriefcase` to false.
- `.evidence`: Overlays `SpeechBubbleView` on the scene. Manages `@State private var currentFileIndex: Int = 1`. The parent handles the tap gesture, incrementing `currentFileIndex` on each tap for files 1-9. Passes `currentFileIndex` to both `SpeechBubbleView` and uses it to derive the Ottie asset: `currentFileIndex >= Chapter4Constants.ottieExpressionChangeFile ? "ottie-suit-soft" : "ottie-suit-pointing"`. When `onAllFilesViewed` fires, transitions to `.closingLine`.
- `.closingLine`: Fades out the speech bubble and dossier file overlays. Renders the final closing line centered using `TypewriterText` with New York Italic 20pt and `parchment` color. Fires a light impact haptic via HapticService on appearance. After typewriter completion, starts a 3-second timer (`Chapter4Constants.finalLineHoldDuration`). On timer expiry, reveals "Continue" in `muted-gray`. Continue tap calls `appState.completeChapter(4)`.

Access `AppState` via `@Environment(AppState.self) private var appState` per the injection pattern in SS4.

The skip-on-relaunch behavior requires no code in this view. ChapterRouter reads `appState.currentChapter` on launch. If `currentChapter != 4`, ChapterRouter never instantiates Chapter4View. SS4 resume semantics confirm that within-chapter sub-state is not persisted: if the app terminates mid-chapter, the next launch starts Chapter 4 from the intro card. This is the default behavior because the `Chapter4Phase` state is local `@State`, which resets on view re-creation.

The final closing line is two sentences rendered as a single `TypewriterText` block. Both lines come from `Chapter4Constants.finalLineTextLine1` and `Chapter4Constants.finalLineTextLine2`, concatenated with a newline. TypewriterText (E2.2) handles the inter-line pause (400ms for line break per E2.2 acceptance criteria) and the punctuation pause on the period (200ms per E2.2 acceptance criteria).
