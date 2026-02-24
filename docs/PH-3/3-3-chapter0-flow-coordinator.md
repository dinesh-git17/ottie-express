# E3.3: Build Chapter 0 Flow Coordinator

**Phase:** 3 - Chapter 0: The Handshake
**Class:** Feature
**Design Doc Reference:** §6 (Chapter 0 Flow), §4 (State Management, State Machine and Resume Semantics), §14 (Navigation and Progression)
**Dependencies:**

- Phase 2: Navigation Shell and Shared UI (exit criteria met)
- E1.2: Implement AppState and persistence (`completeChapter(_:)` method and `completedChapters` set available)
- E2.1: Build navigation shell (ChapterRouter routes to Chapter0View when `currentChapter == 0`)
- E3.1: Build fingerprint scan screen (`FingerprintScreen` view with `onScanComplete` callback)
- E3.2: Build vault animation and password screen (`VaultAnimation` with `onAnimationComplete`, `PasswordScreen` with `onPasswordValid`)

---

## Goal

Wire the fingerprint, vault, and password screens into a sequential Chapter 0 flow that marks chapter completion in AppState and integrates with ChapterRouter for forward navigation.

---

## Scope

### File Inventory

| File | Action | Responsibility |
| --- | --- | --- |
| `OttieExpress/Chapters/Chapter0/Chapter0View.swift` | Create | Flow coordinator managing sequential screen transitions between FingerprintScreen, VaultAnimation, and PasswordScreen, with AppState completion on successful password entry |

### Integration Points

**Chapter0View.swift**

- Imports from: `FingerprintScreen.swift` (E3.1), `VaultAnimation.swift` (E3.2), `PasswordScreen.swift` (E3.2), `Chapter0Constants.swift` (E3.1)
- Imported by: `ChapterRouter.swift` (E2.1)
- State reads: `AppState.completedChapters` (to verify chapter gating on relaunch)
- State writes: `AppState.completedChapters`, `AppState.currentChapter` (via `appState.completeChapter(0)`)

---

## Out of Scope

- Fingerprint scan screen layout, fill animation, progress ring, and haptic interaction (covered by E3.1)
- Vault animation rendering, spring timing, door images, and SFX playback (covered by E3.2)
- Password screen layout, secure text field, validation logic, shake/glow feedback (covered by E3.2)
- Chapter 1 content, layout, or initialization (covered by Phase 4+)
- Cross-chapter transition animation from Chapter 0 to Chapter 1 (handled by ChapterRouter in E2.1; this epic triggers the AppState mutation, the router handles the visual transition)
- Within-chapter sub-state persistence across app relaunch (§4 state machine specifies Chapter 0 resets to the fingerprint screen on relaunch if incomplete)

---

## Definition of Done

- [ ] Chapter0View displays FingerprintScreen as its initial screen when the chapter flow begins
- [ ] Successful fingerprint scan advances the view to VaultAnimation without additional user input
- [ ] Vault animation completion advances the view to PasswordScreen without additional user input
- [ ] Correct password entry calls `appState.completeChapter(0)`, inserting `0` into `completedChapters` and advancing `currentChapter` to `1`
- [ ] ChapterRouter observes the `currentChapter` change and navigates away from Chapter0View to the Chapter 1 view
- [ ] Relaunching the app after completing Chapter 0 routes directly to the current incomplete chapter, bypassing Chapter0View entirely

---

## Implementation Notes

Chapter0View manages a private `@State` enum representing the active phase of the flow:

```swift
private enum Chapter0Phase {
    case fingerprint
    case vault
    case password
}
```

The view body switches on this enum. Each case renders the corresponding child view with a callback that advances the phase:

- `.fingerprint`: renders `FingerprintScreen(onScanComplete: { phase = .vault })`
- `.vault`: renders `VaultAnimation(onAnimationComplete: { phase = .password })`
- `.password`: renders `PasswordScreen(onPasswordValid: { appState.completeChapter(0) })`

Access `AppState` via `@Environment(AppState.self) private var appState` per the injection pattern in §4. The `completeChapter(0)` call mutates `currentChapter` to `1` and persists to `UserDefaults`. ChapterRouter (E2.1) observes this property and performs the chapter transition using the 0.6-second scale pulse animation defined in E2.1 acceptance criteria.

The skip-on-relaunch behavior requires no code in this view. ChapterRouter reads `appState.currentChapter` on launch. If `currentChapter > 0`, ChapterRouter never instantiates Chapter0View. The §4 resume semantics confirm that within-chapter sub-state is not persisted: if the app terminates mid-chapter (after fingerprint but before password), the next launch starts Chapter 0 from the fingerprint screen. This is the default behavior because the `Chapter0Phase` state is local `@State`, which resets on view re-creation.
