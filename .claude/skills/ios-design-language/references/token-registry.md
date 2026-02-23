# Token Registry — Implementation Reference

Complete implementation patterns for the Ottie Express design token system. The SKILL.md defines enforcement rules; this file provides SwiftUI code patterns and detailed usage context.

---

## Color Token Implementation

All color tokens must be defined in a single source-of-truth file. Tokens are accessed as static properties on a `Color` extension.

```swift
import SwiftUI

extension Color {
    // MARK: - Backgrounds
    static let nightBase = Color(hex: 0x0D0D1A)
    static let midnightNavy = Color(hex: 0x111827)
    static let constellationBlue = Color(hex: 0x1E3A5F)

    // MARK: - Accents
    static let warmGold = Color(hex: 0xF5C842)
    static let roseBlush = Color(hex: 0xE8708A)

    // MARK: - Text
    static let parchment = Color(hex: 0xFAF3E0)
    static let softWhite = Color(hex: 0xF9F9F9)
    static let mutedGray = Color(hex: 0x8A8A9A)

    // MARK: - Contextual
    static let forestGreen = Color(hex: 0x2D5A27)
    static let amberWarm = Color(hex: 0xC97D2E)
    static let successGlow = Color(hex: 0x4ADE80)
}

extension Color {
    init(hex: UInt, opacity: Double = 1.0) {
        self.init(
            .sRGB,
            red: Double((hex >> 16) & 0xFF) / 255,
            green: Double((hex >> 8) & 0xFF) / 255,
            blue: Double(hex & 0xFF) / 255,
            opacity: opacity
        )
    }
}
```

**Naming convention:** Token names in the design document use kebab-case (`night-base`). Swift identifiers use camelCase (`nightBase`). The mapping is 1:1.

---

## Color Usage Context Map

| Token               | Primary Context                        | Secondary Context          | Never Use For                |
| ------------------- | -------------------------------------- | -------------------------- | ---------------------------- |
| `nightBase`         | Screen `background`                    | Deepest container fill     | Text color, accents          |
| `midnightNavy`      | Card `background`                      | Elevated container fill    | Screen background, text      |
| `constellationBlue` | Chapter card accent surface            | Featured content highlight | Body text, small elements    |
| `warmGold`          | Primary interactive accent             | Stars, hearts, highlights  | Background fills, body text  |
| `roseBlush`         | Secondary interactive accent           | Love-themed elements       | Background fills, body text  |
| `parchment`         | Literary text on dark backgrounds      | Letter/poem surfaces       | UI labels, button text       |
| `softWhite`         | Primary body text, input text          | High-contrast labels       | Background fills             |
| `mutedGray`         | Hint text, secondary labels            | Disabled state text        | Primary labels, headings     |
| `forestGreen`       | Chapter 1 environment mid tones        | —                          | Any screen outside Chapter 1 |
| `amberWarm`         | Chapter 5 section 2 transition palette | —                          | Any screen outside Chapter 5 |
| `successGlow`       | Word confirmation, chapter complete    | Achievement indicators     | Persistent UI elements       |

**Chapter-scoped tokens:** `forestGreen`, `amberWarm`, and `successGlow` have restricted usage contexts. They are not general-purpose accent colors.

---

## Opacity Scale Implementation

Text hierarchy uses opacity-modified tokens. Define these as reusable view modifiers or computed properties.

```swift
enum TextHierarchy {
    case primary    // 1.0 opacity
    case secondary  // 0.6 opacity
    case tertiary   // 0.3 opacity
    case disabled   // 0.18 opacity

    var opacity: Double {
        switch self {
        case .primary: return 1.0
        case .secondary: return 0.6
        case .tertiary: return 0.3
        case .disabled: return 0.18
        }
    }
}
```

**Usage pattern:**

```swift
// Primary text — full opacity soft-white
Text("Chapter Title")
    .foregroundStyle(Color.softWhite)

// Secondary text — muted-gray (inherently lower contrast)
Text("Subtitle description")
    .foregroundStyle(Color.mutedGray)

// Tertiary text — muted-gray at reduced opacity
Text("Last updated: 2 hours ago")
    .foregroundStyle(Color.mutedGray.opacity(0.5))

// Disabled text — muted-gray at minimum visibility
Text("Locked")
    .foregroundStyle(Color.mutedGray.opacity(0.3))
```

**Note:** `softWhite` at 1.0 serves as primary. `mutedGray` at 1.0 serves as secondary (it is already visually reduced against dark backgrounds). For tertiary and disabled, apply opacity modifiers to `mutedGray`. Do not apply opacity modifiers to `softWhite` — use `mutedGray` instead.

---

## Background Hierarchy Usage Map

Three-level depth system. No exceptions.

```
┌─────────────────────────────────────────────┐
│  Screen (night-base #0D0D1A)                │
│                                              │
│  ┌───────────────────────────────────────┐  │
│  │  Card (midnight-navy #111827)         │  │
│  │                                       │  │
│  │  ┌─────────────────────────────────┐  │  │
│  │  │  Accent (constellation-blue     │  │  │
│  │  │          #1E3A5F)               │  │  │
│  │  └─────────────────────────────────┘  │  │
│  └───────────────────────────────────────┘  │
└─────────────────────────────────────────────┘
```

**Rules:**

- Screens always start with `nightBase`.
- Cards and elevated content use `midnightNavy`.
- Accent surfaces (chapter cards, featured sections) use `constellationBlue`.
- Nesting beyond three levels is prohibited. If a design requires it, restructure the layout.
- Modals use `midnightNavy` background with 28pt corner radius.

---

## Spacing Constants Implementation

```swift
enum Spacing {
    static let standard: CGFloat = 24
    static let compact: CGFloat = 16
    static let grid: CGFloat = 8
    static let fine: CGFloat = 4
}

enum CornerRadius {
    static let card: CGFloat = 20
    static let modal: CGFloat = 28
}

enum TapTarget {
    static let minimum: CGFloat = 44
}
```

**Permitted spacing values (all multiples of 4):**

4, 8, 12, 16, 20, 24, 28, 32, 36, 40, 44, 48, 52, 56, 60, 64

Any spacing value not in this set or not divisible by 4 is a violation.

---

## Contrast Verification

Minimum contrast ratios for text legibility on Ottie backgrounds:

| Text Token  | Background Token    | Contrast Ratio | Passes WCAG AA | Passes WCAG AAA |
| ----------- | ------------------- | -------------- | -------------- | --------------- |
| `softWhite` | `nightBase`         | ~16.5:1        | Yes            | Yes             |
| `softWhite` | `midnightNavy`      | ~14.8:1        | Yes            | Yes             |
| `softWhite` | `constellationBlue` | ~8.2:1         | Yes            | Yes             |
| `mutedGray` | `nightBase`         | ~5.1:1         | Yes            | No              |
| `mutedGray` | `midnightNavy`      | ~4.6:1         | Yes            | No              |
| `parchment` | `nightBase`         | ~14.9:1        | Yes            | Yes             |
| `parchment` | `midnightNavy`      | ~13.3:1        | Yes            | Yes             |
| `warmGold`  | `nightBase`         | ~9.8:1         | Yes            | Yes             |
| `roseBlush` | `nightBase`         | ~5.4:1         | Yes            | No              |

**All primary text combinations exceed WCAG AA (4.5:1).** `mutedGray` on `midnightNavy` is the tightest pairing at 4.6:1 — acceptable for secondary text but not for primary content.

---

## Surface Construction Patterns

### Card Pattern (Permitted)

```swift
VStack(alignment: .leading, spacing: Spacing.grid) {
    Text("Card Title")
        .font(.system(size: 24, weight: .semibold))
        .foregroundStyle(Color.softWhite)
    Text("Card description text")
        .font(.system(size: 16, weight: .regular))
        .foregroundStyle(Color.mutedGray)
}
.padding(Spacing.standard)
.background(Color.midnightNavy)
.clipShape(RoundedRectangle(cornerRadius: CornerRadius.card))
```

### Card Pattern (Prohibited)

```swift
// VIOLATION: gradient, border, shadow, arbitrary radius
VStack {
    Text("Card Title")
}
.padding(20) // hardcoded — use Spacing.standard
.background(
    LinearGradient(colors: [.blue, .purple], ...) // §6.2 violation
)
.cornerRadius(12) // §7.2 violation — must be 20 or 28
.shadow(radius: 10) // §6.2 violation
.overlay(
    RoundedRectangle(cornerRadius: 12)
        .stroke(Color.white.opacity(0.2), lineWidth: 1) // §6.2 violation
)
```
