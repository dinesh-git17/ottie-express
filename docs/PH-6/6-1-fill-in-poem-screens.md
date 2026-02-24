# E6.1: Build Fill-in Poem Screens

**Phase:** 6 - Chapter 2: The Poems
**Class:** Feature
**Design Doc Reference:** §8 (Fill-in Poem layout, Swipe Mechanic, Word Lists per Blank, Full Poem Texts)
**Dependencies:**

- Phase 2: Navigation Shell and Shared UI (exit criteria met)
- E1.1: Define design system tokens (color tokens `parchment`, `night-base`, `warm-gold`, `soft-white`, `success-glow`, `muted-gray`, `rose-blush` and typography roles available)
- E1.4: Build HapticService (UIImpactFeedbackGenerator medium pattern available)
- E2.4: Build HapticButton (reusable button with haptic feedback available)

---

## Goal

Implement three fill-in poem screens with swipe-to-select blank widgets, looping word list cycling, correct-word lock animation, and per-poem "Next" button activation gated on all blanks solved.

---

## Scope

### File Inventory

| File                                                     | Action | Responsibility                                                                                                                                            |
| -------------------------------------------------------- | ------ | --------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `OttieExpress/Chapters/Chapter2/PoemView.swift`          | Create | Full-screen poem layout with embedded SwipeBlankView instances, scroll view for overflow content, and "Next" button with inactive/active state toggle     |
| `OttieExpress/Chapters/Chapter2/SwipeBlankView.swift`    | Create | Pill-shaped swipe widget that cycles through a word list on vertical swipe, detects correct selection, triggers lock animation and haptic feedback        |
| `OttieExpress/Chapters/Chapter2/PoemData.swift`          | Create | Static data definitions for all three poems: full text with blank placeholders, word lists per blank with correct answer indices, and poem count constant |
| `OttieExpress/Chapters/Chapter2/Chapter2Constants.swift` | Create | Named constants for Chapter 2: typography sizes, animation durations, swipe thresholds, pill dimensions, glow parameters, and color token references      |

### Integration Points

**PoemView.swift**

- Imports from: `SwipeBlankView.swift`, `PoemData.swift`, `Chapter2Constants.swift`
- Imported by: `Chapter2View.swift` (E6.4)
- State reads: None (receives poem data as input parameter)
- State writes: None (reports poem completion via callback closure)

**SwipeBlankView.swift**

- Imports from: `PoemData.swift`, `Chapter2Constants.swift`
- Imported by: `PoemView.swift`
- State reads: None
- State writes: None (reports correct selection via binding)

**PoemData.swift**

- Imports from: None
- Imported by: `PoemView.swift`, `SwipeBlankView.swift`, `Chapter2State.swift` (E6.4)
- State reads: None
- State writes: None

**Chapter2Constants.swift**

- Imports from: None
- Imported by: `PoemView.swift`, `SwipeBlankView.swift`, `PoemCollectionView.swift` (E6.2), `HerTurnView.swift` (E6.2), `HaikuReplyView.swift` (E6.3), `PoemContinuationView.swift` (E6.3), `Chapter2State.swift` (E6.4), `Chapter2View.swift` (E6.4)
- State reads: None
- State writes: None

---

## Out of Scope

- Poem collection reveal animation after all three poems are solved (covered by E6.2)
- Her-turn text input screen and "Send" button (covered by E6.2)
- API response display, typewriter animation, and fallback handling (covered by E6.3)
- Chapter 2 flow coordination, intro card, per-poem persistence, and chapter completion (covered by E6.4)
- Audio SFX during poem interaction such as `sfx-poem-reveal` (no SFX specified in §8 for the fill-in mechanic itself; poem reveal SFX belongs to E6.2)
- Transition animations between poems (orchestrated by E6.4 flow coordinator)
- Dark mode token variants or dynamic type scaling beyond the specified typography (deferred to Phase 12 polish)

---

## Definition of Done

- [ ] PoemView renders poem text in New York Regular 18pt with `parchment` color on `night-base` background for each of the three poems
- [ ] Each blank in the poem renders as a pill-shaped widget with `warm-gold` border and `soft-white` text displaying the current word
- [ ] Swiping up on a blank cycles to the next word in that blank's curated word list
- [ ] Swiping down on a blank cycles to the previous word in that blank's curated word list
- [ ] Word list cycling loops infinitely in both directions (swiping past the last word returns to the first, and vice versa)
- [ ] Selecting the correct word triggers a `success-glow` on the pill, a UIImpactFeedbackGenerator medium haptic, and a small scale pulse animation
- [ ] A correctly selected word locks in place and the blank no longer responds to swipe gestures
- [ ] "Next" button renders in `muted-gray` (inactive) when any blank remains unsolved
- [ ] "Next" button renders in `rose-blush` (active) and becomes tappable only when all blanks in the current poem are correct
- [ ] Poem 1 contains exactly 4 blanks ("there", "home", "heart", "you") with the word lists defined in §8
- [ ] Poem 2 contains exactly 4 blanks ("think", "hand", "alone", "yours") with the word lists defined in §8
- [ ] Poem 3 contains exactly 4 blanks ("name", "hands", "steps", "you") with the word lists defined in §8
- [ ] Full poem text for all three poems matches §8 verbatim, with blank positions marked by bracket notation in the source data
- [ ] PoemView scrolls vertically when poem content exceeds the viewport height

---

## Implementation Notes

Per §8 Swipe Mechanic, each blank operates independently. The word list for each blank is defined in §8 Word Lists per Blank with the correct answer bolded in the design doc. The correct answer is the last entry in each list as presented in §8.

The pill widget must distinguish three visual states: default (cycling, `warm-gold` border), correct (locked, `success-glow` glow), and the transition between them (scale pulse). The scale pulse is a momentary animation on lock, not a persistent visual state.

All three poems use the same PoemView component parameterized by a `PoemData` value. The poem index (0, 1, 2) is passed to the view to select the correct data set.

Per §8, the "Next" button position is bottom of screen. The inactive/active state is purely visual and functional: inactive state prevents tap recognition, active state fires the completion callback.

All magic numbers (font sizes, animation durations, pill dimensions, glow radius) belong in `Chapter2Constants.swift`. Reference §2 design system for color token values and §8 for typography specifications.
