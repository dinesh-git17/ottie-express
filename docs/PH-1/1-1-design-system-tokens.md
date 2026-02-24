# E1.1: Define Design System Tokens

**Phase:** 1 - Core Systems and Design Foundation
**Class:** Infrastructure
**Design Doc Reference:** SS2 Design System (Color Palette, Typography, Spacing and Layout, Motion Principles)
**Dependencies:**

- Phase 0: Project Scaffold (exit criteria met)
- E0.1: Scaffold Xcode project and directory structure (directory hierarchy and asset catalog exist)

---

## Goal

Translate the SS2 color palette, typography scale, spacing constants, and motion parameters into typed Swift constants and SwiftUI extensions consumable by all views and services.

---

## Scope

### File Inventory

| File                                               | Action | Responsibility                                                                                                                       |
| -------------------------------------------------- | ------ | ------------------------------------------------------------------------------------------------------------------------------------ |
| `OttieExpress/Extensions/Color+DesignTokens.swift` | Create | Define all 11 named color tokens from SS2 as static `Color` properties using hex initializer                                         |
| `OttieExpress/Extensions/Font+DesignTokens.swift`  | Create | Define all 6 typography roles from SS2 as static `Font` properties mapping to SF Pro Display, SF Pro Text, and New York              |
| `OttieExpress/Extensions/DesignConstants.swift`    | Create | Define spacing constants, corner radii, tap target minimums, spring parameters, and transition durations as namespaced static values |

### Integration Points

**Color+DesignTokens.swift**

- Imports from: None (extends SwiftUI.Color only)
- Imported by: None within Phase 1 (consumed by all view files in Phase 2+ via `import SwiftUI`)
- State reads: None
- State writes: None

**Font+DesignTokens.swift**

- Imports from: None (extends SwiftUI.Font only)
- Imported by: None within Phase 1 (consumed by all view files in Phase 2+ via `import SwiftUI`)
- State reads: None
- State writes: None

**DesignConstants.swift**

- Imports from: None (standalone constants namespace)
- Imported by: None within Phase 1 (consumed by view and animation code in Phase 2+)
- State reads: None
- State writes: None

---

## Out of Scope

- Dark mode alternate color values or dynamic color resolution (SS2 defines a single palette; the app uses a fixed dark aesthetic per SS1 product definition)
- SwiftUI preview scaffolding or token gallery views for visual verification (verification is via SwiftUI previews written by the implementing engineer ad hoc, not a committed artifact)
- `Color(uiColor:)` bridging or UIKit color compatibility extensions (all rendering is SwiftUI-native per SS4 Technology Stack)
- Accessibility overrides for Dynamic Type scaling or bold text (addressed per-view in feature epics, not at the token definition layer)
- Animation helper functions, view modifiers, or transition definitions that consume these constants (covered by Phase 2 navigation shell and shared UI epics)

---

## Definition of Done

- [ ] `Color.nightBase` resolves to `#0D0D1A` when inspected in a SwiftUI preview
- [ ] All 11 color tokens from SS2 (`nightBase`, `midnightNavy`, `constellationBlue`, `warmGold`, `roseBlush`, `parchment`, `softWhite`, `mutedGray`, `forestGreen`, `amberWarm`, `successGlow`) are accessible as static `Color` properties
- [ ] `Font.displayTitle` returns SF Pro Display Bold at 32pt when applied to a SwiftUI `Text` view
- [ ] All 6 typography roles from SS2 (`displayTitle`, `chapterHeading`, `body`, `poemLetter`, `hintCaption`, `typewriterText`) are accessible as static `Font` properties
- [ ] `DesignConstants.Spacing.standard` equals `24` and `DesignConstants.Spacing.compact` equals `16`
- [ ] `DesignConstants.Spacing.minTapTarget` equals `44`, `DesignConstants.CornerRadius.card` equals `20`, and `DesignConstants.CornerRadius.modal` equals `28`
- [ ] `DesignConstants.Motion.springStiffness` equals `280` and `DesignConstants.Motion.springDamping` equals `22`
- [ ] `DesignConstants.Motion.screenTransitionDuration` equals `0.4` and `DesignConstants.Motion.chapterCompleteDuration` equals `0.6`
- [ ] Project compiles with zero errors after adding all three files

---

## Implementation Notes

Color tokens use hex values from SS2 Color Palette table. Implement a private `Color.init(hex:)` initializer in `Color+DesignTokens.swift` that parses a `UInt32` hex literal into RGB components. Each token is a `static let` on a `Color` extension. Token names use lowerCamelCase Swift convention derived from the SS2 kebab-case names (e.g., `night-base` becomes `nightBase`). Reference: SS2 Design System, Color Palette table.

Typography roles map to system fonts via `Font.custom(_:size:)` for New York and `Font.system(size:weight:design:)` for SF Pro variants. SF Pro Display maps to `.design(.default)` with explicit weight. New York maps to `.design(.serif)`. The SS2 Typography table defines six roles with specific font/weight/size combinations. Where a size range is specified (e.g., 32-40pt for Display/Title), use the lower bound as the default and expose the role as a function accepting an optional size override. Reference: SS2 Design System, Typography table.

Spacing, corner radius, and motion constants live in a `DesignConstants` enum (caseless, used purely as a namespace) with nested enums for `Spacing`, `CornerRadius`, and `Motion`. All values are `static let` of type `CGFloat` or `Double` as appropriate. The spring baseline values (`stiffness: 280, damping: 22`) are the default parameters referenced by SS2 Motion Principles. Reference: SS2 Design System, Spacing and Layout section, Motion Principles section.
