# E2.4: Build HapticButton Component

**Phase:** 2 - Navigation Shell and Shared UI
**Class:** Infrastructure
**Design Doc Reference:** SS4 Architecture Overview (SharedUI, Technology Stack), SS2 Design System (Spacing and Layout), SS15 Haptics and Sound Design
**Dependencies:**

- Phase 1: Core Systems and Design Foundation (exit criteria met)
- E1.1: Define design system tokens (`DesignConstants.Spacing.minTapTarget` available as named constant)
- E1.4: Build HapticService (haptic pattern enum and callable `fire(_:)` interface available)

---

## Goal

Implement the reusable button wrapper that fires a configured haptic pattern from HapticService on tap with `@ViewBuilder` label support and 44pt minimum tap target enforcement.

---

## Scope

### File Inventory

| File                                       | Action | Responsibility                                                                                                           |
| ------------------------------------------ | ------ | ------------------------------------------------------------------------------------------------------------------------ |
| `OttieExpress/SharedUI/HapticButton.swift` | Create | Generic button view that triggers a HapticService pattern on tap, enforces minimum tap target, and renders custom labels |

### Integration Points

**HapticButton.swift**

- Imports from: `HapticService.swift` (HapticPattern enum type, HapticService callable interface), `DesignConstants.swift` (`Spacing.minTapTarget`)
- Imported by: None within Phase 2 (consumed by interactive elements across Chapter 0-6 and ending screen in Phase 3-10 feature epics)
- State reads: None
- State writes: None (triggers HapticService side effect on tap; no persistent state mutation)

---

## Out of Scope

- Custom haptic pattern definitions or CoreHaptics engine configuration (HapticService from E1.4 owns all pattern definitions; HapticButton references the existing pattern enum only)
- Visual button styling beyond minimum tap target enforcement (each call site provides its own visual appearance via the `@ViewBuilder` label closure)
- Long-press haptic variants, gesture recognizer customization, or hold-to-trigger patterns (long-press interactions like Ch0 fingerprint scan are built as custom gesture views, not as HapticButton instances)
- Disabled state haptic suppression logic (the calling view controls enabled/disabled state externally; a disabled HapticButton simply does not fire)
- Simultaneous gesture conflict resolution for HapticButtons embedded in ScrollViews or gesture-heavy contexts (resolved per-use in feature epics where the conflict arises)
- Button press animation or scale-on-tap feedback (visual press feedback is applied by call sites via the label closure or SwiftUI `.buttonStyle`, not by HapticButton itself)

---

## Definition of Done

- [ ] HapticButton accepts a `HapticPattern` enum value from E1.4 and fires the corresponding haptic via HapticService when tapped
- [ ] The button enforces a minimum 44pt x 44pt interactive area via frame constraint or content shape, regardless of the label content size
- [ ] Custom label content provided via `@ViewBuilder` closure renders as the button's visible content
- [ ] Tapping HapticButton executes both the haptic trigger and the caller-provided action closure in sequence
- [ ] HapticButton compiles against the HapticService interface from E1.4 without requiring additional service initialization or configuration at the call site

---

## Implementation Notes

`HapticButton` is a generic SwiftUI view parameterized by its label content type (`HapticButton<Label: View>`). It wraps a standard `Button` whose action closure: (1) calls `HapticService` to fire the configured pattern, then (2) executes the caller's action closure. The two operations execute sequentially in the action closure; haptic firing is synchronous (UIKit feedback generators fire immediately). Reference: SS15 Haptics and Sound Design, E1.4 acceptance criteria ("All patterns callable by name through a typed enum interface").

The `HapticPattern` type is the enum defined in E1.4 `HapticService.swift`. `HapticButton` does not define its own pattern type or subset. It accepts any value from the full pattern enum. The implementing engineer must verify the E1.4 deliverable exposes the enum as `internal` or wider access. Reference: E1.4 scope.

The 44pt minimum tap target uses `.frame(minWidth: DesignConstants.Spacing.minTapTarget, minHeight: DesignConstants.Spacing.minTapTarget)` combined with `.contentShape(Rectangle())` to ensure the interactive hit region meets the SS2 minimum regardless of label size. If the label content is larger than 44pt in either dimension, the frame constraint does not shrink it. Reference: SS2 Spacing and Layout ("Minimum tap target: 44x44pt").

`HapticButton` does not own, initialize, or configure a `HapticService` instance. It accesses the service through whatever pattern E1.4 establishes. If E1.4 implements a singleton (`HapticService.shared`), `HapticButton` calls `HapticService.shared.fire(pattern)`. If E1.4 uses environment injection, `HapticButton` reads it via `@Environment`. The access pattern is determined by the E1.4 deliverable, not by this epic. Reference: SS4 Architecture Overview, E1.4 HapticService interface.

The component initializer signature:

```swift
init(
    haptic: HapticPattern,
    action: @escaping () -> Void,
    @ViewBuilder label: () -> Label
)
```

This mirrors the standard `Button` initializer pattern with the addition of the `haptic` parameter. Reference: SS4 SharedUI ("Reusable components: TypewriterText, ChapterCard, HapticButton").
