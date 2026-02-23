---
name: ios-design-language
description: Enforce production-grade iOS visual language, typographic hierarchy, symbol discipline, and design-token-driven consistency for the Ottie Express application. Use when writing or reviewing any SwiftUI view code, Color definitions, font usage, SF Symbol selection, surface styling, layout spacing, dark mode handling, or visual hierarchy decisions. Triggers on SwiftUI view implementation, color usage, font selection, symbol rendering mode choice, card or surface construction, spacing and padding decisions, or any task producing visible UI output.
---

# iOS Design Language & Visual Restraint

Governing standard for all visual output in this repository. Every color, font, spacing value, symbol, and surface must derive from the design token registry. Ad-hoc visual decisions are prohibited. Decorative noise is prohibited. Structure communicates hierarchy — not color, not gradients, not shadows.

---

## 1. Visual Philosophy

The Ottie Express aesthetic is dark, warm, tactile, and premium. It is not a system-default iOS app. It is not Material Design. It is a deliberate visual language that communicates intimacy through restraint.

**Governing principles:**

- **Content is the interface.** UI chrome defers to content at all times.
- **Structure over decoration.** Hierarchy through typography weight, spacing, and opacity. Never through borders, gradients, or color variation.
- **Token authority is absolute.** Every visual value must trace to the token registry. If a value is not in the registry, it does not belong in the code.
- **Warmth through palette, not effects.** The dark/gold/rose palette creates warmth. Glow effects, gradients, and drop shadows do not.

---

## 2. Color Token Registry (BLOCKING)

All color values MUST reference a named token. Hex literals, `Color.red`, `Color(red:green:blue:)`, and unnamed `Color(hex:)` calls outside the token definition file are prohibited.

| Token                | Hex       | Usage Context                               |
| -------------------- | --------- | ------------------------------------------- |
| `night-base`         | `#0D0D1A` | Primary backgrounds, deep screens           |
| `midnight-navy`      | `#111827` | Card backgrounds, elevated surfaces         |
| `constellation-blue` | `#1E3A5F` | Accent surfaces, chapter cards              |
| `warm-gold`          | `#F5C842` | Primary accent, hearts, stars, highlights   |
| `rose-blush`         | `#E8708A` | Secondary accent, love elements, UI glow    |
| `parchment`          | `#FAF3E0` | Light text on dark, letter backgrounds      |
| `soft-white`         | `#F9F9F9` | Body text, input text                       |
| `muted-gray`         | `#8A8A9A` | Hint text, secondary labels, disabled       |
| `forest-green`       | `#2D5A27` | Chapter 1 game environment mid tones        |
| `amber-warm`         | `#C97D2E` | Chapter 5 section 2 transition              |
| `success-glow`       | `#4ADE80` | Correct word confirmation, chapter complete |

### 2.1 Color Enforcement Rules

- Every `Color` reference in SwiftUI view code must use a named token constant.
- No inline hex values. No `Color.white`, `Color.black`, `Color.gray`, or any system color outside the token set.
- `Color.primary` and `Color.secondary` are prohibited. Use `soft-white` and `muted-gray`.
- `Color.accentColor` is prohibited. Use `warm-gold` or `rose-blush` by context.
- Opacity modifiers on tokens are permitted only for the hierarchy levels defined in §5.
- New colors require explicit addition to the token registry with a documented usage context.

### 2.2 Prohibited Color Patterns

```swift
// PROHIBITED — inline hex
Color(hex: "#FF5733")

// PROHIBITED — system named colors
Color.blue
Color.orange
.foregroundStyle(.secondary)

// CORRECT — token reference
Color.warmGold
Color.nightBase
.foregroundStyle(Color.softWhite)
```

---

## 3. Typography System (BLOCKING)

### 3.1 Font Role Assignment

| Role            | Font           | Weight   | Size    | Usage                            |
| --------------- | -------------- | -------- | ------- | -------------------------------- |
| Display / Title | SF Pro Display | Bold     | 32–40pt | Screen titles, chapter names     |
| Chapter heading | SF Pro Display | Semibold | 24pt    | Section headers within chapters  |
| Body            | SF Pro Text    | Regular  | 16–18pt | Standard UI text, descriptions   |
| Poem / Letter   | New York       | Regular  | 17–20pt | Literary content, poems, letters |
| Hint / Caption  | SF Pro Text    | Regular  | 13pt    | Secondary labels, hint text      |
| Typewriter text | New York       | Regular  | 18pt    | Typewriter-animated sequences    |

### 3.2 Font Selection Rules

- **SF Pro** is mandatory for all UI chrome: navigation, buttons, labels, controls, HUD elements, input fields.
- **New York** is permitted ONLY for literary content: poems, letters, voice note transcripts, typewriter-animated text.
- Mixing SF Pro and New York in the same visual unit (card, row, overlay) is prohibited unless literary content is the primary content of that unit.
- Custom fonts are prohibited. No `Font.custom()` calls.

### 3.3 Weight Discipline

- Permitted weights: Regular, Medium, Semibold, Bold.
- Ultralight, Thin, and Light are prohibited. They compromise legibility on dark backgrounds.
- Heavy and Black are prohibited. They create visual clutter.
- Maximum two weight levels per visual unit. Bold for headings, Regular for body.

### 3.4 Size Discipline

- All text sizes must match a role in §3.1. Arbitrary point sizes are prohibited.
- The system handles SF Pro Text / Display switching at the 20pt threshold automatically. Never manually specify `SF Pro Text` or `SF Pro Display` — use `.system(size:weight:design:)`.
- Dynamic Type must be supported. All text styles must scale with user preference.

### 3.5 Line Height

- System-provided leading for each text style by default.
- Custom line height permitted only for New York literary content: 1.5–1.7x multiplier.
- Typewriter text uses 1.6x line height.

---

## 4. SF Symbol Discipline

### 4.1 Rendering Mode Selection

| Mode         | When to Use                                                | When NOT to Use                      |
| ------------ | ---------------------------------------------------------- | ------------------------------------ |
| Monochrome   | UI controls, navigation, standard interface elements       | When depth emphasis needed           |
| Hierarchical | Single accent color with layer-based depth differentiation | When multiple distinct colors needed |
| Palette      | Brand/theme colors applied to distinct symbol layers       | General UI (too visually heavy)      |
| Multicolor   | Real-world objects with inherent color                     | Abstract UI controls (never)         |

**Default: Monochrome.** Hierarchical permitted when depth serves comprehension. Palette and Multicolor require explicit justification per use.

### 4.2 Weight and Scale Matching

- Symbol weight MUST match adjacent text weight. Semibold label requires semibold symbol.
- Size symbols via `Font`-based sizing: `.font(.system(size: 17, weight: .semibold))`. Never `.frame(width:height:)`.
- Default symbol scale: `.medium`. Adjust only when optical alignment requires it.

### 4.3 Prohibited Symbol Patterns

- Decorative symbols with no functional purpose.
- Symbols as visual filler (scattered stars, ambient icons). Use asset art.
- `Image(systemName:)` with hardcoded `.frame(width:height:)` instead of font sizing.
- Multicolor rendering on abstract interface icons.

---

## 5. Visual Hierarchy Through Structure

Hierarchy is communicated through typography weight, opacity, spacing, and position. Not through color variation, borders, or shadows.

### 5.1 Opacity-Based Text Hierarchy

| Level     | Opacity | Token        | Usage                                |
| --------- | ------- | ------------ | ------------------------------------ |
| Primary   | 1.0     | `soft-white` | Headings, primary labels, key values |
| Secondary | 0.6     | `muted-gray` | Descriptions, supporting text        |
| Tertiary  | 0.3     | `muted-gray` | Timestamps, metadata, hint text      |
| Disabled  | 0.18    | `muted-gray` | Disabled labels, inactive states     |

Follows Apple's semantic label color hierarchy: 100% → 60% → 30% → 18%.

**Rules:**

- `soft-white` at full opacity for primary text. `muted-gray` for secondary and below.
- Never create hierarchy through distinct hues. Hierarchy is structure, not color.
- Arbitrary opacity values outside this scale (100/60/30/18) are prohibited for text.

### 5.2 Spatial Hierarchy

- Important content occupies more vertical space (generous padding above and below).
- Secondary content is spatially compressed.
- Whitespace is deliberate. Every spacing value derives from the spacing scale in §7.

### 5.3 Elevation Hierarchy

- `night-base` (#0D0D1A) — deepest layer, screen backgrounds.
- `midnight-navy` (#111827) — elevated content, cards.
- `constellation-blue` (#1E3A5F) — highest non-accent surface.
- This three-level depth system is the only permitted background hierarchy. No additional background colors.

---

## 6. Surface Construction (BLOCKING)

### 6.1 Permitted Surface Techniques

- **Flat color fills** using tokens from §2.
- **Spacing and padding** to define card boundaries.
- **Corner radius**: 20pt for cards, 28pt for modals. No other values.
- **Opacity layering** using the three background tokens at prescribed depth levels.

### 6.2 Prohibited Surface Techniques

BLOCKED unless the design document explicitly specifies them for a specific screen element:

- **Gradients** — `LinearGradient`, `RadialGradient`, `AngularGradient`, `MeshGradient`. Flat color only.
- **Glassmorphism / blur materials** — `.ultraThinMaterial`, `.regularMaterial`, `VisualEffectView`.
- **Drop shadows** — `.shadow()` modifier. Depth from background color stepping, not shadows.
- **Borders / strokes** — `overlay(RoundedRectangle(...).stroke(...))`. Cards defined by color contrast, not edges.
- **Inner shadows** — No simulated depth through shadow effects.
- **Glow effects** — No `.blur()` halos around elements.

### 6.3 Explicit Override Protocol

If the design document specifies a gradient, border, or shadow for a specific screen element:

1. Cite the exact design document section and element.
2. Apply the effect to that element only.
3. Do not generalize the effect to similar elements elsewhere.

---

## 7. Spacing and Layout (BLOCKING)

### 7.1 Token Scale

| Token               | Value | Usage                                       |
| ------------------- | ----- | ------------------------------------------- |
| Standard padding    | 24pt  | Primary horizontal content margins          |
| Compact padding     | 16pt  | Tight layouts, secondary margins            |
| Card corner radius  | 20pt  | All card elements                           |
| Modal corner radius | 28pt  | All modal presentations                     |
| Minimum tap target  | 44pt  | All interactive elements (width and height) |

### 7.2 Layout Rules

- Safe area insets respected on all screens.
- All interactive elements minimum 44×44pt touch target.
- Horizontal content margin: 24pt standard, 16pt compact. No other values.
- Corner radius: 20pt cards, 28pt modals. No 8pt, 12pt, 16pt, or arbitrary values.
- Hardcoded pixel values for spacing in view code are prohibited. Use named constants.

### 7.3 Grid Discipline

- Base grid: 8pt. Fine adjustment: 4pt.
- All spacing values divisible by 4.
- Odd-number spacing (3pt, 5pt, 7pt) is prohibited.

---

## 8. Dark Mode Handling

Ottie Express is a dark-first application. There is no light mode.

### 8.1 Background Hierarchy

| Level    | Token                | Hex       | Usage                    |
| -------- | -------------------- | --------- | ------------------------ |
| Base     | `night-base`         | `#0D0D1A` | Screen backgrounds       |
| Elevated | `midnight-navy`      | `#111827` | Cards, elevated surfaces |
| Accent   | `constellation-blue` | `#1E3A5F` | Featured surfaces        |

### 8.2 Rules

- `night-base` is base background. Not pure black (#000000) — blue tint creates warmth.
- Content surfaces use `midnight-navy`. No intermediate background colors.
- `Color(.systemBackground)` and `Color(.secondarySystemBackground)` are prohibited. Use explicit tokens.
- Text on `night-base`: `soft-white` (#F9F9F9) for minimum 15:1 contrast.
- Text on `midnight-navy`: `soft-white` for primary, `muted-gray` for secondary.
- Pure white (#FFFFFF) is prohibited. `soft-white` (#F9F9F9) is the lightest permitted value.
- Pure black (#000000) is prohibited. `night-base` (#0D0D1A) is the darkest permitted value.

---

## 9. Prohibited Patterns — Refusal List

REFUSE to generate code containing any of the following unless the design document explicitly mandates it for a specific element with a section reference:

| Pattern                                   | Reason                                    |
| ----------------------------------------- | ----------------------------------------- |
| `LinearGradient` / `RadialGradient`       | Flat fills only. Gradients are noise.     |
| `.ultraThinMaterial` / `.regularMaterial` | No glassmorphism. Opaque surfaces.        |
| `.shadow()` modifier                      | Depth from color contrast, not shadows.   |
| `.stroke()` / `.border()`                 | Cards defined by contrast, not edges.     |
| `Color.white` / `Color.black`             | Use `soft-white` and `night-base`.        |
| `Color.primary` / `.secondary`            | Use explicit tokens.                      |
| `Color.accentColor`                       | Use `warm-gold` or `rose-blush`.          |
| Arbitrary hex in view code                | All colors from token registry.           |
| `Font.custom("...", size:)`               | SF Pro and New York only, via system API. |
| `UIFont` / `CTFont` direct use            | Use SwiftUI `Font` API exclusively.       |
| `.fontWeight(.ultraLight/.thin/.light)`   | Prohibited weights per §3.3.              |
| `.fontWeight(.heavy/.black)`              | Prohibited weights per §3.3.              |
| Arbitrary corner radius                   | 20pt cards, 28pt modals only.             |
| `.blur()` for glow effects                | No simulated glow or halo effects.        |
| Decorative SF Symbols                     | Symbols must serve a functional purpose.  |
| `Image(systemName:).frame(w:h:)`          | Size symbols via font, not frame.         |

**Refusal protocol:**

1. HALT — Do not write the prohibited code.
2. STATE — Name the violated rule and section number.
3. OFFER — Provide the correct token-compliant alternative.

---

## 10. Code Review Checklist

Reject UI code if ANY of these are true:

- [ ] Contains a color value not in the token registry (§2).
- [ ] Uses system named colors (`Color.blue`, `Color.primary`).
- [ ] Contains inline hex values outside the token definition file.
- [ ] Uses a font not in the typography role table (§3.1).
- [ ] Uses a prohibited font weight (§3.3).
- [ ] Uses an arbitrary point size not in the role table.
- [ ] Contains a gradient, material, shadow, or border (§6.2) without a design doc citation.
- [ ] Uses a corner radius other than 20pt or 28pt.
- [ ] Contains a spacing value not divisible by 4.
- [ ] Sizes an SF Symbol via `.frame()` instead of font sizing.
- [ ] Uses decorative symbols with no functional purpose.
- [ ] Creates text hierarchy through distinct hues instead of opacity/weight.
- [ ] Uses `Color(.systemBackground)` or any `UIColor` semantic color.
- [ ] Contains pure white (#FFFFFF) or pure black (#000000).
- [ ] Missing Dynamic Type support on text elements.
- [ ] Interactive element smaller than 44×44pt.

---

## References

- `references/typography-and-symbols.md` — SF Pro / New York selection decision tree, Dynamic Type style table, SF Symbol rendering mode decision matrix, weight matching rules.
- `references/token-registry.md` — SwiftUI token implementation patterns, opacity scale code examples, background hierarchy usage map, spacing constant patterns.
