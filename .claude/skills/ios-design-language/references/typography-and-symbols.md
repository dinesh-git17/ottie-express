# Typography & Symbol Reference

Deep reference for typographic rules and SF Symbol usage patterns. The SKILL.md defines enforcement rules; this file provides the technical foundation.

---

## SF Pro vs New York Decision Tree

```
Is this content literary (poem, letter, note, voice transcript)?
├── YES → New York, Regular, 17–20pt
│   └── Is it typewriter-animated?
│       ├── YES → New York, Regular, 18pt, 1.6x line height, 40ms/char
│       └── NO  → New York, Regular, size per content type
└── NO → SF Pro
    └── What is the text role?
        ├── Screen title / display   → Bold, 32–40pt
        ├── Section heading          → Semibold, 24pt
        ├── Body / description       → Regular, 16–18pt
        ├── HUD / game overlay       → Semibold, 15pt
        └── Caption / hint           → Regular, 13pt
```

---

## Dynamic Type Text Styles (Default "Large" Size)

Apple's system text styles and their default metrics. Use these as reference when mapping Ottie typography roles to Dynamic Type.

| Text Style  | Weight   | Size | Leading | Ottie Role Mapping      |
| ----------- | -------- | ---- | ------- | ----------------------- |
| Large Title | Bold     | 34pt | 41pt    | Display / Title         |
| Title 1     | Regular  | 28pt | 34pt    | —                       |
| Title 2     | Regular  | 22pt | 28pt    | Chapter heading (≈24pt) |
| Title 3     | Regular  | 20pt | 25pt    | —                       |
| Headline    | Semibold | 17pt | 22pt    | Body emphasis           |
| Body        | Regular  | 17pt | 22pt    | Body                    |
| Callout     | Regular  | 16pt | 21pt    | Body (lower bound)      |
| Subheadline | Regular  | 15pt | 20pt    | HUD text                |
| Footnote    | Regular  | 13pt | 18pt    | Hint / Caption          |
| Caption 1   | Regular  | 12pt | 16pt    | Fine print              |
| Caption 2   | Regular  | 11pt | 13pt    | Minimum text (floor)    |

**Leading variants:**

- Tight leading: system value minus 2pt.
- Loose leading: system value plus 2pt.

**Ottie-specific note:** Custom sizes (32pt, 40pt, 24pt) do not map 1:1 to Dynamic Type styles. Implement Dynamic Type scaling by using `@ScaledMetric` with the base size and a reference text style:

```swift
@ScaledMetric(relativeTo: .largeTitle) private var displaySize: CGFloat = 36
@ScaledMetric(relativeTo: .title2) private var chapterHeadingSize: CGFloat = 24
```

---

## SF Pro Text vs Display Auto-Switching

The system font automatically selects the optical variant:

- **SF Pro Text** (≤19pt): Wider letter spacing, generous apertures for small-size legibility.
- **SF Pro Display** (≥20pt): Tighter spacing, refined letterforms for large-size elegance.

The switch is automatic when using `.system(size:weight:design:)`. Manual specification of `SF Pro Text` or `SF Pro Display` by name is prohibited — it bypasses optical optimization.

**Width variants** (standard, compressed, condensed, expanded) exist but are not used in Ottie Express. Width variation is prohibited unless a future design document revision explicitly adds it.

---

## New York Usage Rules

New York is Apple's serif companion to SF Pro. In Ottie Express, it creates a literary, intimate contrast against the SF Pro UI chrome.

**Permitted contexts:**

- Poem text in Chapter 2 (18pt, Regular, generous line height 1.6x)
- Letter/note content where the design document specifies "literary feel"
- Typewriter-animated text sequences (18pt, Regular, 40ms per character, 200ms on punctuation)
- Floating phrases in Chapter 3 (19pt, Italic)
- Final emotional lines that the design document calls out specifically (e.g., Chapter 4 closing: 20pt, Italic, centered)

**Prohibited contexts:**

- Navigation bars, tab bars, toolbars
- Buttons, toggles, switches, input labels
- HUD overlays, score displays, game UI
- Error messages, alerts, confirmation dialogs
- Any system-chrome element

**Italic usage:** New York Italic is permitted only where the design document explicitly specifies it (Chapter 3 floating phrases at 19pt, Chapter 4 closing at 20pt). Do not apply italic to New York text by default.

---

## SF Symbol Rendering Mode Decision Matrix

| Scenario                               | Mode         | Rationale                               |
| -------------------------------------- | ------------ | --------------------------------------- |
| Navigation bar button                  | Monochrome   | Uniform, unobtrusive chrome             |
| Tab bar icon                           | Monochrome   | System convention, weight-matched       |
| Inline with body text                  | Monochrome   | Typographic consistency                 |
| Status indicator (active/inactive)     | Hierarchical | Layer opacity conveys state depth       |
| Heart collectible indicator            | Monochrome   | Colored via `warm-gold` foreground tint |
| Confirmation checkmark                 | Monochrome   | Colored via `success-glow` foreground   |
| Interactive control with depth cue     | Hierarchical | Primary shape at 100%, secondary at 50% |
| Anything requiring two explicit colors | Palette      | Rare — justify per use                  |
| Real-world object representation       | Multicolor   | Extremely rare in Ottie Express context |

### Hierarchical Mode Opacity Levels

When using Hierarchical rendering, the system applies:

- Primary layer: 100% opacity of the applied color
- Secondary layer: ~50% opacity
- Tertiary layer: ~25% opacity

These are system-determined. Do not manually replicate with `.opacity()` on Monochrome symbols.

---

## Weight Matching Reference

| Text Context    | Text Weight | Symbol Weight | Symbol Scale |
| --------------- | ----------- | ------------- | ------------ |
| Display heading | Bold        | Bold          | .medium      |
| Chapter heading | Semibold    | Semibold      | .medium      |
| Body text       | Regular     | Regular       | .medium      |
| HUD overlay     | Semibold    | Semibold      | .small       |
| Caption         | Regular     | Regular       | .small       |

**Implementation pattern:**

```swift
// CORRECT — weight-matched symbol and text
Label {
    Text("Score")
        .font(.system(size: 15, weight: .semibold))
} icon: {
    Image(systemName: "star.fill")
        .font(.system(size: 15, weight: .semibold))
}

// WRONG — mismatched weights
Label {
    Text("Score")
        .font(.system(size: 15, weight: .semibold))
} icon: {
    Image(systemName: "star.fill")
        .font(.system(size: 18, weight: .regular))
}

// WRONG — frame-based sizing
Image(systemName: "star.fill")
    .frame(width: 20, height: 20)
```

---

## Typewriter Animation Typography

The design document specifies typewriter text across multiple chapters. Typography constants:

| Property     | Value         | Source                    |
| ------------ | ------------- | ------------------------- |
| Font         | New York      | §3.1 Typewriter text role |
| Weight       | Regular       | §3.1                      |
| Size         | 18pt          | §3.1                      |
| Line height  | 1.6× (28.8pt) | Design doc §2.4, §3.5     |
| Char delay   | 40ms default  | Design doc Chapter 2      |
| Punctuation  | 200ms         | Design doc Chapter 2      |
| Slow variant | 60ms per char | Design doc Chapter 3      |
| Word-by-word | 80ms per word | Design doc Chapter 6      |
| Color        | `parchment`   | Design doc Chapter 2      |
