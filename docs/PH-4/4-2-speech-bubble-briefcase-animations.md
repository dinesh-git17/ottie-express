# E4.2: Build Speech Bubble and Briefcase Animations

**Phase:** 4 - Chapter 4: The Dossier
**Class:** Feature
**Design Doc Reference:** SS10 (Scene Entry, Evidence Files), SS2 (Motion Principles), SS15 (Haptic Map, Audio Management), SS5 (Chapter 4 Assets)
**Dependencies:**

- Phase 2: Navigation Shell and Shared UI (exit criteria met)
- E1.1: Define design system tokens (color, typography, spacing, and motion constants available)
- E1.3: Build AudioService (SFX playback interface available for `sfx-briefcase-shake`, `sfx-file-slam`, `sfx-speech-bubble-pop`)
- E1.4: Build HapticService (heavy impact pattern callable for file slam)
- E4.1: Build dossier scene layout and evidence data (`EvidenceFile.allFiles` data available, `Chapter4Constants` defined)
- Asset: `ottie-suit-pointing` (available in asset catalog)
- Asset: `ottie-suit-soft` (available in asset catalog)
- Asset: `briefcase-idle` (available in asset catalog)
- Asset: `briefcase-shake` (available in asset catalog)
- Asset: `dossier-file` (available in asset catalog)
- Asset: `speech-bubble` (available in asset catalog)
- Asset: `sfx-briefcase-shake` (available in bundle)
- Asset: `sfx-file-slam` (available in bundle)
- Asset: `sfx-speech-bubble-pop` (available in bundle)

---

## Goal

Implement the briefcase shake-to-dossier morph animation sequence, the speech bubble evidence file display with tap-to-advance progression through all 10 files, and Ottie expression transitions synchronized to file number.

---

## Scope

### File Inventory

| File | Action | Responsibility |
| --- | --- | --- |
| `OttieExpress/Chapters/Chapter4/BriefcaseAnimation.swift` | Create | Briefcase shake oscillation, crossfade and scale morph into dossier file, coordinated SFX playback for shake and file slam |
| `OttieExpress/Chapters/Chapter4/SpeechBubbleView.swift` | Create | Speech bubble container rendering above Ottie, evidence file text display in classification format, tap-to-advance through files 1-10 |

### Integration Points

**BriefcaseAnimation.swift**

- Imports from: `Chapter4Constants.swift` (E4.1), `AudioService.swift` (E1.3), `HapticService.swift` (E1.4)
- Imported by: `Chapter4View.swift` (E4.3)
- State reads: None
- State writes: None
- Public interface: `BriefcaseAnimation(onMorphComplete: @escaping () -> Void)` where the callback fires after the dossier file lands on the table

**SpeechBubbleView.swift**

- Imports from: `EvidenceFile.swift` (E4.1), `Chapter4Constants.swift` (E4.1), `Color+DesignTokens.swift` (E1.1), `Font+DesignTokens.swift` (E1.1), `AudioService.swift` (E1.3)
- Imported by: `Chapter4View.swift` (E4.3)
- State reads: None
- State writes: None
- Public interface: `SpeechBubbleView(currentFileIndex: Int, onFileAdvanced: @escaping (Int) -> Void, onAllFilesViewed: @escaping () -> Void)` where `currentFileIndex` drives which evidence file displays, `onFileAdvanced` reports each advance, and `onAllFilesViewed` fires after file 10 is displayed and the post-file-10 pause completes

---

## Out of Scope

- Static scene layout and background compositing (covered by E4.1)
- Evidence file data model and text content (covered by E4.1; this epic consumes `EvidenceFile.allFiles`)
- Intro card with "The Dossier" title and "View Files" button (covered by E4.3)
- Final closing line typewriter animation after evidence files complete (covered by E4.3)
- Chapter completion wiring to AppState (covered by E4.3)
- BGM playback or background audio for Chapter 4 (SS10 does not specify a BGM track for this chapter)
- AudioService interface changes or new playback methods beyond the existing SFX API (AudioService interface owned by E1.3; this epic consumes the existing `playSFX` interface)
- Reduced motion accessibility fallbacks for briefcase shake and morph animations (deferred to Phase 11 cross-cutting hardening)

---

## Definition of Done

- [ ] After a 1-second delay from scene entry, `briefcase-idle` begins a scale oscillation animation that repeats 3 times with `sfx-briefcase-shake` playing on oscillation start
- [ ] Briefcase oscillation completes and the briefcase crossfades and scales into `dossier-file` with `sfx-file-slam` playing on the morph landing
- [ ] Heavy impact haptic fires via HapticService simultaneously with `sfx-file-slam` on dossier file landing
- [ ] Speech bubble appears above Ottie with a scale-in entrance animation and `sfx-speech-bubble-pop` playing on appearance
- [ ] Speech bubble displays evidence file text in the `"EVIDENCE FILE [number]\nCLASSIFICATION: LEVEL HEART\n[body]"` format using `EvidenceFile.formattedText`
- [ ] Tapping anywhere on the screen advances to the next evidence file for files 1 through 9
- [ ] Ottie displays `ottie-suit-pointing` expression during evidence files 1 through 9
- [ ] Ottie transitions to `ottie-suit-soft` expression when evidence file 10 displays, with a crossfade transition
- [ ] File 10 does not require a tap to advance; after display, a 1.5-second pause elapses and `onAllFilesViewed` fires
- [ ] `onMorphComplete` callback fires exactly once after the dossier file morph completes and before the speech bubble appears

---

## Implementation Notes

The briefcase animation sequence is a three-phase state machine:

1. **Idle**: 1-second delay timer (`Chapter4Constants.briefcaseAnimationDelay`). On expiry, transition to Shake.
2. **Shake**: Scale oscillation between 1.0 and 1.1 repeated `Chapter4Constants.briefcaseShakeCount` times using `PhaseAnimator` or explicit `withAnimation` sequencing. Play `sfx-briefcase-shake` via AudioService on phase entry. On completion, transition to Morph.
3. **Morph**: Simultaneously animate opacity from 1 to 0 on `briefcase-shake` and opacity from 0 to 1 on `dossier-file` with a scale transform (1.2 to 1.0 on the dossier file for a "landing" feel). Play `sfx-file-slam` and fire heavy impact haptic via HapticService at the midpoint. On completion, invoke `onMorphComplete`.

Use the spring baseline from SS2 (`stiffness: 280, damping: 22`) for the morph landing spring. The shake uses a faster, tighter spring or linear timing to feel mechanical.

`SpeechBubbleView` renders as an overlay positioned above the Ottie sprite. The speech bubble uses the `speech-bubble` asset as a background image with evidence file text overlaid. Text renders in SF Pro Text Regular 14pt, `night-base` (#0D0D1A) on the white speech bubble surface. The classification header ("EVIDENCE FILE" and "CLASSIFICATION: LEVEL HEART") renders in SF Pro Text Bold 12pt for visual hierarchy within the bubble.

Tap detection for file advancement uses a full-screen `.onTapGesture` modifier on the parent container. The parent (E4.3) manages the `currentFileIndex` state and passes it down. When `currentFileIndex` equals `Chapter4Constants.ottieExpressionChangeFile` (10), the parent swaps the Ottie asset parameter from `"ottie-suit-pointing"` to `"ottie-suit-soft"` on `DossierSceneView`. After file 10 renders, an internal `Task.sleep` of `Chapter4Constants.postFile10Pause` (1.5 seconds) elapses before `onAllFilesViewed` fires. No tap input is processed during file 10 display.
