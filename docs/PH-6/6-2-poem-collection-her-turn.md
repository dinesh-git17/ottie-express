# E6.2: Build Poem Collection Reveal and Her-Turn Input

**Phase:** 6 - Chapter 2: The Poems
**Class:** Feature
**Design Doc Reference:** §8 (Poem Collection Reveal, Her Turn)
**Dependencies:**

- Phase 2: Navigation Shell and Shared UI (exit criteria met)
- E1.1: Define design system tokens (color tokens `rose-blush`, `parchment`, `midnight-navy`, `soft-white` and typography roles available)
- E1.3: Build AudioService (`sfx-poem-reveal` playback available)
- E6.1: Build fill-in poem screens (PoemData definitions and Chapter2Constants available)
- Asset: `sfx-poem-reveal` (available in app bundle)
- Asset: Ottie idle sprite (available in asset catalog)

---

## Goal

Implement the post-poem-3 collection reveal animation that assembles solved poems into a scrollable vertical stack, and the her-turn text input screen where the user composes a poem and submits it.

---

## Scope

### File Inventory

| File                                                      | Action | Responsibility                                                                                                                                                         |
| --------------------------------------------------------- | ------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `OttieExpress/Chapters/Chapter2/PoemCollectionView.swift` | Create | Animated reveal of three poem cards sliding in from different directions, settling into a vertical scroll stack with "Your poems" header and delayed "Continue" prompt |
| `OttieExpress/Chapters/Chapter2/HerTurnView.swift`        | Create | Text input screen with Ottie idle sprite, prompt text, multiline text field, and "Send" button that captures input text                                                |

### Integration Points

**PoemCollectionView.swift**

- Imports from: `PoemData.swift` (E6.1), `Chapter2Constants.swift` (E6.1)
- Imported by: `Chapter2View.swift` (E6.4)
- State reads: None (receives solved poem texts as input)
- State writes: None (reports continue action via callback closure)

**HerTurnView.swift**

- Imports from: `Chapter2Constants.swift` (E6.1)
- Imported by: `Chapter2View.swift` (E6.4)
- State reads: None
- State writes: None (reports submitted poem text via callback closure with the captured String)

---

## Out of Scope

- Fill-in poem interaction, swipe mechanic, and word list cycling (covered by E6.1)
- API call execution, typewriter display of reply, and fallback handling (covered by E6.3)
- Flow coordination between poem screens, collection reveal, her-turn, and API display (covered by E6.4)
- Per-poem completion persistence and resume logic (covered by E6.4)
- Haptic feedback on "Send" button tap (consumed from HapticButton in E2.4; this epic builds the view, not the haptic integration)
- Character count enforcement or input validation beyond basic text capture (§8 specifies no character limit)
- Keyboard dismiss behavior or input accessory customization (deferred to Phase 12 polish)

---

## Definition of Done

- [ ] PoemCollectionView animates three poem cards sliding in from different directions on appearance
- [ ] Poem cards settle into a vertically scrollable stack after the entrance animation completes
- [ ] `sfx-poem-reveal` plays via AudioService at the start of the entrance animation
- [ ] "Your poems" label renders at the top of the collection in SF Pro Semibold 15pt with `rose-blush` color
- [ ] All three solved poems are visible as continuous scrollable content within the vertical stack
- [ ] A "Continue" prompt fades in at the bottom of the screen after a 2-second delay from animation completion
- [ ] HerTurnView renders Ottie idle sprite at small size positioned at the top of the screen
- [ ] Prompt text "You felt every word. Now write one of your own." renders centered in New York Regular 20pt with `parchment` color
- [ ] Multiline text input renders with `midnight-navy` background, `soft-white` text in New York Regular 17pt, and "Your words..." placeholder
- [ ] "Send" button renders at the bottom of the screen in `rose-blush`
- [ ] Tapping "Send" captures the current text input and reports it via the completion callback
- [ ] "Send" button is disabled and visually dimmed when the text input is empty

---

## Implementation Notes

Per §8 Poem Collection Reveal, each poem card slides in from a different direction. The design doc does not specify exact directions or timing curves. Use three distinct entrance vectors (e.g., leading, trailing, bottom) with the baseline spring parameters from E1.1 (`stiffness: 280, damping: 22`) staggered by 150-200ms per card.

Per §8 Her Turn, the prompt text is fixed copy. The text input has no enforced character limit per §8 ("Character limit: none enforced"). The placeholder text is "Your words..." per §8. The "Send" button must capture the raw text string; the API call itself is executed by E6.3 and orchestrated by E6.4.

The Ottie idle sprite on the her-turn screen is described as "small, top of screen" in §8. Use the standard Ottie idle asset scaled to approximately 80pt height, centered horizontally, per the visual language established in other chapters.

AudioService is consumed (not modified) by this epic. The `sfx-poem-reveal` playback call uses the existing AudioService.playSFX interface from E1.3.
