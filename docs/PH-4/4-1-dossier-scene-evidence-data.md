# E4.1: Build Dossier Scene Layout and Evidence Data

**Phase:** 4 - Chapter 4: The Dossier
**Class:** Feature
**Design Doc Reference:** SS10 (Scene Entry, Evidence Files), SS2 (Color Palette, Typography), SS5 (Chapter 4 Assets)
**Dependencies:**

- Phase 2: Navigation Shell and Shared UI (exit criteria met)
- E1.1: Define design system tokens (color, typography, spacing, and motion constants available)
- Asset: `bg-office-room` (available in asset catalog)
- Asset: `ottie-suit-seated` (available in asset catalog)
- Asset: `briefcase-idle` (available in asset catalog)

---

## Goal

Build the composited dossier scene layout with office room background, Ottie, and briefcase positioning, define the typed evidence file data model with all 10 file entries, and declare all Chapter 4 named constants.

---

## Scope

### File Inventory

| File | Action | Responsibility |
| --- | --- | --- |
| `OttieExpress/Chapters/Chapter4/DossierSceneView.swift` | Create | Composited scene layout with `bg-office-room` background, `ottie-suit-seated` centered behind table, and `briefcase-idle` on table to the right of Ottie |
| `OttieExpress/Chapters/Chapter4/EvidenceFile.swift` | Create | Typed data model for evidence files with file number, classification level, and body text for all 10 entries from SS10, plus formatted display string computation |
| `OttieExpress/Chapters/Chapter4/Chapter4Constants.swift` | Create | Named constants for all Chapter 4 screens: animation timings, layout dimensions, text content, and classification format values |

### Integration Points

**DossierSceneView.swift**

- Imports from: `Chapter4Constants.swift`, `Color+DesignTokens.swift` (E1.1), `Font+DesignTokens.swift` (E1.1), `DesignConstants.swift` (E1.1)
- Imported by: `Chapter4View.swift` (E4.3)
- State reads: None
- State writes: None
- Public interface: `DossierSceneView(ottieAsset: String, showBriefcase: Bool)` where `ottieAsset` controls which Ottie expression renders and `showBriefcase` controls briefcase visibility before morph

**EvidenceFile.swift**

- Imports from: None
- Imported by: `SpeechBubbleView.swift` (E4.2), `Chapter4View.swift` (E4.3)
- State reads: None
- State writes: None
- Public interface: `EvidenceFile` struct with static `allFiles` array returning all 10 entries in order

**Chapter4Constants.swift**

- Imports from: None
- Imported by: `DossierSceneView.swift`, `SpeechBubbleView.swift` (E4.2), `BriefcaseAnimation.swift` (E4.2), `Chapter4View.swift` (E4.3)
- State reads: None
- State writes: None

---

## Out of Scope

- Briefcase shake animation, scale oscillation, and morph into dossier file (covered by E4.2)
- Speech bubble rendering and evidence file text population within the bubble (covered by E4.2)
- Tap-to-advance interaction through evidence files (covered by E4.2)
- Ottie expression transitions between `ottie-suit-pointing` and `ottie-suit-soft` during file progression (covered by E4.2)
- Audio playback of `sfx-briefcase-shake`, `sfx-file-slam`, or `sfx-speech-bubble-pop` (covered by E4.2 AudioService integration)
- Intro card rendering, final closing line typewriter, and chapter completion flow (covered by E4.3)
- Design token additions or modifications to `Color+DesignTokens.swift` or `Font+DesignTokens.swift` (owned by E1.1; this epic consumes existing tokens without modification)
- Reduced motion accessibility fallback for scene entry animations (deferred to Phase 11 cross-cutting hardening)

---

## Definition of Done

- [ ] `bg-office-room` renders as a full-screen background image scaled to fill the screen bounds on a `night-base` (#0D0D1A) base
- [ ] `ottie-suit-seated` renders centered horizontally behind the table area in the lower-center portion of the scene
- [ ] `briefcase-idle` renders on the table to the right of Ottie when `showBriefcase` is true and is hidden when `showBriefcase` is false
- [ ] `EvidenceFile` struct exposes `fileNumber` (Int), `classification` (String), and `bodyText` (String) properties with a computed `formattedText` that produces `"EVIDENCE FILE [number]\nCLASSIFICATION: LEVEL HEART\n[bodyText]"`
- [ ] `EvidenceFile.allFiles` returns exactly 10 entries with body text matching SS10 verbatim, ordered by file number 01 through 10
- [ ] `DossierSceneView` accepts an `ottieAsset` parameter and renders the corresponding sprite image, allowing E4.2 to swap between `ottie-suit-pointing` and `ottie-suit-soft`
- [ ] All layout dimensions, animation timings, and text content values in the scene reference named constants from `Chapter4Constants` with zero magic numbers

---

## Implementation Notes

`DossierSceneView` composites layers using a `ZStack`. The layering order from back to front: `bg-office-room` background image, `ottie-suit-seated` (or alternate Ottie expression) sprite, table surface (implied by background art), `briefcase-idle` sprite. The briefcase position is offset to the right of center. Use `Image(ottieAsset)` to allow the parent to swap Ottie expressions without modifying this file.

The `showBriefcase` boolean controls initial briefcase visibility. E4.2's `BriefcaseAnimation` replaces the static briefcase with the animated sequence, so the parent (E4.3) hides the static briefcase once the animation phase begins.

`EvidenceFile` is a value type conforming to `Sendable`:

```swift
struct EvidenceFile: Sendable, Identifiable {
    let id: Int
    let bodyText: String

    var formattedText: String {
        "EVIDENCE FILE \(String(format: "%02d", id))\nCLASSIFICATION: LEVEL HEART\n\(bodyText)"
    }
}
```

The `allFiles` static property returns an array literal containing all 10 entries. Each entry's `bodyText` is copied verbatim from SS10. No truncation, no rewording.

`Chapter4Constants` defines named constants consumed by all Chapter 4 epics:

| Constant | Value | Consumer |
| --- | --- | --- |
| `briefcaseAnimationDelay` | 1.0 (seconds) | E4.2 |
| `briefcaseShakeCount` | 3 | E4.2 |
| `postFile10Pause` | 1.5 (seconds) | E4.3 |
| `finalLineHoldDuration` | 3.0 (seconds) | E4.3 |
| `evidenceFileCount` | 10 | E4.2, E4.3 |
| `ottieExpressionChangeFile` | 10 | E4.2 |
| `introCardTitle` | "The Dossier" | E4.3 |
| `introCardBody` | "Classified intelligence has been gathered. You've been observed." | E4.3 |
| `finalLineTextLine1` | "After reviewing all available evidence, the conclusion is simple." | E4.3 |
| `finalLineTextLine2` | "I'm completely in love with you." | E4.3 |
