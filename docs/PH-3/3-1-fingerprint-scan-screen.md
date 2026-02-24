# E3.1: Build Fingerprint Scan Screen

**Phase:** 3 - Chapter 0: The Handshake
**Class:** Feature
**Design Doc Reference:** §6 (Fingerprint Screen), §2 (Color Palette, Typography, Motion Principles), §15 (Haptic Map)
**Dependencies:**

- Phase 2: Navigation Shell and Shared UI (exit criteria met)
- E1.1: Define design system tokens (color, typography, spacing, and motion constants available)
- E1.4: Build HapticService (CoreHaptics progressive scan pattern and single sharp impact pattern callable)
- Asset: `fingerprint-svg` (available in asset catalog)

---

## Goal

Build the fingerprint scan screen with welcome text, SVG display, blinking hint label, hold-to-scan CoreHaptics interaction, bottom-to-top fill mask animation, and circular progress ring.

---

## Scope

### File Inventory

| File                                                     | Action | Responsibility                                                                                                                                                                                  |
| -------------------------------------------------------- | ------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `OttieExpress/Chapters/Chapter0/FingerprintScreen.swift` | Create | Fingerprint scan screen layout with long-press gesture, bottom-to-top fill mask animation, circular progress ring, and CoreHaptics integration                                                  |
| `OttieExpress/Chapters/Chapter0/Chapter0Constants.swift` | Create | Named constants for all Chapter 0 screens: scan duration, fingerprint dimensions, blink interval, password value, vault spring timing, password field height, shake duration, and layout values |

### Integration Points

**FingerprintScreen.swift**

- Imports from: `Chapter0Constants.swift`, `Color+DesignTokens.swift` (E1.1), `Font+DesignTokens.swift` (E1.1), `DesignConstants.swift` (E1.1), `HapticService.swift` (E1.4)
- Imported by: `Chapter0View.swift` (E3.3)
- State reads: None
- State writes: None
- Public interface: `FingerprintScreen(onScanComplete: @escaping () -> Void)`

**Chapter0Constants.swift**

- Imports from: None
- Imported by: `FingerprintScreen.swift`, `VaultAnimation.swift` (E3.2), `PasswordScreen.swift` (E3.2), `Chapter0View.swift` (E3.3)
- State reads: None
- State writes: None

---

## Out of Scope

- Audio playback of `sfx-fingerprint-scan` during scan (asset exists per §5 but §6 interaction specifies CoreHaptics only; audio layering deferred to a polish pass)
- Vault split animation triggered after scan completion (covered by E3.2)
- Password entry screen and validation logic (covered by E3.2)
- Chapter 0 flow coordination and AppState chapter completion (covered by E3.3)
- Reduced motion accessibility fallback for fill animation (§2 motion principles reference `prefers-reduced-motion`; Phase 3 targets functional completion first)

---

## Definition of Done

- [ ] "Welcome Carolina" renders in SF Pro Display Bold 32pt, `soft-white` (#F9F9F9), centered horizontally in the top third of the screen on a `night-base` (#0D0D1A) background
- [ ] Fingerprint SVG renders centered in the vertical middle of the screen in `rose-blush` (#E8708A) at approximately 180pt width
- [ ] "Hold to enter" label renders in SF Pro Text Regular 15pt, `muted-gray` (#8A8A9A), with a repeating opacity blink animation at a 1-second interval
- [ ] Long-press on the fingerprint area triggers a CoreHaptics repeating transient pattern via HapticService with intensity increasing from low to high over 2 seconds
- [ ] Fingerprint SVG fills from bottom to top with a `rose-blush` glow via a vertical mask animation over the 2-second hold duration
- [ ] Circular progress ring in `warm-gold` (#F5C842) appears around the fingerprint and fills proportionally to the hold duration
- [ ] Holding for 2 continuous seconds fires a single sharp heavy impact haptic via HapticService, fades the "Hold to enter" label to zero opacity, and invokes the `onScanComplete` callback
- [ ] Releasing the hold before 2 seconds completes resets the fill mask, progress ring, and haptic pattern to their initial resting state without displaying an error

---

## Implementation Notes

The fingerprint fill uses a vertical rectangle clip mask whose height is driven by a progress value: elapsed hold time divided by `Chapter0Constants.scanDuration` (2.0s). Anchor the mask to the bottom edge so fill progresses upward. The progress ring uses `Circle().trim(from: 0, to: progress)` with a `warm-gold` stroke, rotated -90 degrees so the ring starts at the 12 o'clock position.

Use `DragGesture(minimumDistance: 0)` for continuous hold detection. `LongPressGesture` is inappropriate because it fires a single recognition event rather than providing continuous progress. Track press state in `onChanged` (begin) and `onEnded` (release). On press begin, start a repeating timer or `Task` that increments progress linearly toward 1.0 over `scanDuration`. On release before completion, animate progress back to 0 and cancel the timer.

CoreHaptics patterns are requested from HapticService using the progressive scan pattern (E1.4 acceptance criteria: "CoreHaptics transient repeating pattern with increasing intensity over 2 seconds"). Fire the pattern on press begin. Stop the pattern on early release. On scan completion, fire the single sharp impact pattern (E1.4: "Single sharp impact pattern") and stop the progressive pattern.

`Chapter0Constants` defines named constants consumed by all Chapter 0 epics per §3.1. The complete constant set:

| Constant                 | Value          | Consumer |
| ------------------------ | -------------- | -------- |
| `scanDuration`           | 2.0 (seconds)  | E3.1     |
| `fingerprintWidth`       | 180.0 (points) | E3.1     |
| `hintBlinkInterval`      | 1.0 (seconds)  | E3.1     |
| `vaultSpringDuration`    | 0.6 (seconds)  | E3.2     |
| `ottieWavingHeight`      | 200.0 (points) | E3.2     |
| `passwordFieldHeight`    | 48.0 (points)  | E3.2     |
| `shakeAnimationDuration` | 0.3 (seconds)  | E3.2     |
| `treasureMapHintOpacity` | 0.6            | E3.2     |
| `password`               | "CarolinaLGTM" | E3.2     |

The `onScanComplete` closure is the sole output interface. Chapter0View (E3.3) provides this closure and uses it to advance the internal flow state to the vault phase.
