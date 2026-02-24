# E3.2: Build Vault Animation and Password Screen

**Phase:** 3 - Chapter 0: The Handshake
**Class:** Feature
**Design Doc Reference:** §6 (Vault Animation, Password Screen), §2 (Color Palette, Typography), §5 (Audio Assets: sfx-vault-open), §15 (Haptic Map)
**Dependencies:**

- Phase 2: Navigation Shell and Shared UI (exit criteria met)
- E1.1: Define design system tokens (color, typography, spacing, and motion constants available)
- E1.3: Build AudioService (SFX playback for `sfx-vault-open` available)
- E1.4: Build HapticService (medium impact, success notification, and single sharp impact patterns callable)
- E3.1: Build fingerprint scan screen (`Chapter0Constants.swift` created with shared named constants)
- Asset: `vault-door-top` (available in asset catalog)
- Asset: `vault-door-bottom` (available in asset catalog)
- Asset: `ottie-waving` (available in asset catalog)
- Asset: `sfx-vault-open` (available in audio bundle)

---

## Goal

Build the vault split animation with spring-timed door separation and SFX, and the password entry screen with secure input, shake-on-error haptic feedback, and glow-on-success confirmation.

---

## Scope

### File Inventory

| File | Action | Responsibility |
| --- | --- | --- |
| `OttieExpress/Chapters/Chapter0/VaultAnimation.swift` | Create | Vault door split animation with top/bottom image separation using spring timing, AudioService SFX trigger, and completion callback |
| `OttieExpress/Chapters/Chapter0/PasswordScreen.swift` | Create | Password entry layout with secure text field, validation against hardcoded password, horizontal shake-on-error animation with haptic, hint pulse, and success glow with haptic |

### Integration Points

**VaultAnimation.swift**

- Imports from: `Chapter0Constants.swift` (E3.1), `DesignConstants.swift` (E1.1), `AudioService.swift` (E1.3)
- Imported by: `Chapter0View.swift` (E3.3)
- State reads: None
- State writes: None
- Public interface: `VaultAnimation(onAnimationComplete: @escaping () -> Void)`

**PasswordScreen.swift**

- Imports from: `Chapter0Constants.swift` (E3.1), `Color+DesignTokens.swift` (E1.1), `Font+DesignTokens.swift` (E1.1), `DesignConstants.swift` (E1.1), `HapticService.swift` (E1.4)
- Imported by: `Chapter0View.swift` (E3.3)
- State reads: None
- State writes: None
- Public interface: `PasswordScreen(onPasswordValid: @escaping () -> Void)`

---

## Out of Scope

- Fingerprint scan interaction, screen layout, and fill animation (covered by E3.1)
- Chapter 0 flow coordination wiring FingerprintScreen, VaultAnimation, and PasswordScreen into sequence (covered by E3.3)
- AppState chapter completion marking after successful password entry (covered by E3.3)
- Transition animation from Chapter 0 to Chapter 1 (handled by ChapterRouter in E2.1; this epic validates the password, E3.3 triggers the state mutation)
- Custom keyboard styling or input filtering beyond password string comparison (design doc §6 specifies system default keyboard with no custom keyboard)

---

## Definition of Done

- [ ] `vault-door-top` image renders covering the top half of the screen and slides upward off-screen with a spring animation (stiffness 280, damping 22, scaled to 0.6s settle) when VaultAnimation appears
- [ ] `vault-door-bottom` image renders covering the bottom half and slides downward off-screen with identical spring parameters and timing
- [ ] AudioService plays `sfx-vault-open` at the moment the vault split animation begins
- [ ] Password screen content fades in from zero opacity beneath the separating vault doors as they slide apart
- [ ] `onAnimationComplete` callback fires after the vault door springs settle, signaling the parent that password screen interaction is ready
- [ ] `ottie-waving` asset renders centered in the upper half of the password screen at approximately 200pt tall
- [ ] Secure text field renders centered below Ottie with `midnight-navy` (#111827) fill, `soft-white` (#F9F9F9) text, rounded rect style, and 48pt height
- [ ] "Only you can open this" renders below the text field in New York Regular 15pt, `muted-gray` (#8A8A9A), centered
- [ ] "Psst... Ask Dinesh for the treasure map" renders at the bottom of the screen in SF Pro Text 13pt, `muted-gray` (#8A8A9A), centered, at 0.6 opacity
- [ ] Submitting an incorrect password triggers a horizontal shake animation on the text field (0.3 seconds), fires a medium impact haptic via HapticService, and pulses the treasure map hint opacity from 0.6 to 1.0 and back once
- [ ] Submitting `CarolinaLGTM` applies a `success-glow` (#4ADE80) border or shadow on the text field, fires a success notification haptic via HapticService, and invokes the `onPasswordValid` callback

---

## Implementation Notes

The vault door layout positions two `Image` views to tile the full screen vertically. `vault-door-top` fills from the top edge to the vertical center. `vault-door-bottom` fills from the vertical center to the bottom edge. On appearance, toggle a `@State private var isOpen = false` flag inside a `withAnimation(.spring(duration: Chapter0Constants.vaultSpringDuration))` block. Bind `vault-door-top` offset to `isOpen ? -screenHeight : 0` and `vault-door-bottom` offset to `isOpen ? screenHeight : 0`.

Use `withAnimation(_:completionCriteria:body:completion:)` (available iOS 17+) with `.logicallyComplete` criteria to detect spring settlement and fire `onAnimationComplete`. AudioService SFX call fires in the same `onAppear` block, before the animation toggle, so audio begins as doors start moving.

The password field uses `SecureField` with an `onSubmit` modifier that compares the bound text against `Chapter0Constants.password`. The comparison is case-sensitive per the design doc's explicit password value.

The shake animation uses a horizontal offset keyframe sequence: 0pt, +10pt, -10pt, +6pt, -6pt, 0pt over `Chapter0Constants.shakeAnimationDuration` (0.3s). Implement via `PhaseAnimator` or a `@State` offset driven by a `withAnimation` spring with high stiffness for a snappy shake. The hint pulse animates opacity from `Chapter0Constants.treasureMapHintOpacity` (0.6) to 1.0 and back, timed to coincide with the shake.

The success glow applies `Color.successGlow` as a border (2pt stroke) or shadow on the text field. Hold the glow for 0.3 seconds, then invoke `onPasswordValid`. The success haptic fires from HapticService using `UINotificationFeedbackGenerator` success type per §15 haptic map.
