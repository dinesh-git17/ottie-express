# E2.3: Build ChapterCard Component

**Phase:** 2 - Navigation Shell and Shared UI
**Class:** Infrastructure
**Design Doc Reference:** SS4 Architecture Overview (SharedUI), SS7 Chapter 1 (Intro Card layout specification), SS2 Design System (Color Palette, Typography, Spacing and Layout)
**Dependencies:**

- Phase 1: Core Systems and Design Foundation (exit criteria met)
- E1.1: Define design system tokens (color tokens, font roles, spacing constants, and corner radii available as named Swift properties)

---

## Goal

Implement the reusable chapter intro card component with design-token-driven layout for chapter label, title, body text, optional asset image, and full-width action button consumable by all seven chapter entry screens.

---

## Scope

### File Inventory

| File                                      | Action | Responsibility                                                                                                             |
| ----------------------------------------- | ------ | -------------------------------------------------------------------------------------------------------------------------- |
| `OttieExpress/SharedUI/ChapterCard.swift` | Create | Chapter intro card view with parameterized label, title, body, optional image, and action button using design token values |

### Integration Points

**ChapterCard.swift**

- Imports from: `Color+DesignTokens.swift` (`Color.midnightNavy`, `Color.roseBlush`, `Color.softWhite`, `Color.mutedGray`), `Font+DesignTokens.swift` (`Font.displayTitle`, `Font.body`, `Font.hintCaption`), `DesignConstants.swift` (`CornerRadius.modal`, `Spacing.standard`, `Spacing.minTapTarget`, `CornerRadius.card`)
- Imported by: None within Phase 2 (consumed by Chapter 0-6 intro screens in Phase 3-9 feature epics)
- State reads: None
- State writes: None

---

## Out of Scope

- Chapter-specific body copy, asset image names, or button labels (each chapter provides these values as parameters when constructing a `ChapterCard` instance)
- Entry or appearance animation for the card itself (card transition animation is managed by `ChapterRouter` transitions from E2.1, not by `ChapterCard` internally)
- Card dismissal animation, swipe-to-dismiss, or drag-to-close gesture (chapters manage their own transition from intro card to chapter content via their flow coordinator)
- Landscape layout adaptation or iPad-specific sizing (SS1 specifies iPhone 17 Pro only in portrait orientation)
- Dynamic Type scaling or bold text accessibility overrides for card text elements (accessibility hardening deferred to Phase 11)
- Background blur, material, or vibrancy effects behind the card (SS7 specifies a solid `midnight-navy` fill)

---

## Definition of Done

- [ ] Card background renders in `Color.midnightNavy` (`#111827`) with 28pt corner radius applied to top-leading and top-trailing corners only
- [ ] Chapter label renders in SF Pro Text Semibold 13pt in `Color.roseBlush` (`#E8708A`), leading-aligned with 24pt horizontal padding
- [ ] Title renders in SF Pro Display Bold 32pt in `Color.softWhite` (`#F9F9F9`), leading-aligned below the chapter label
- [ ] Body text renders in SF Pro Text Regular 16pt in `Color.mutedGray` (`#8A8A9A`), leading-aligned below the title
- [ ] Action button renders full-width with `Color.roseBlush` fill, `Color.softWhite` label text, 56pt height, and 20pt corner radius
- [ ] Tapping the action button executes the provided action closure exactly once per tap
- [ ] When an image asset name is provided, the image renders right-aligned within the card layout
- [ ] When no image asset name is provided, the card renders without an image region and text fills the available width

---

## Implementation Notes

The card uses a `VStack(alignment: .leading)` as its primary layout container. Text elements stack vertically: chapter label, title, body. The action button pins to the bottom of the card with full-width layout. Horizontal padding is `DesignConstants.Spacing.standard` (24pt) on both sides. Reference: SS2 Spacing and Layout, SS7 Intro Card layout.

Top corner radius uses `UnevenRoundedRectangle(topLeadingRadius: DesignConstants.CornerRadius.modal, topTrailingRadius: DesignConstants.CornerRadius.modal)` as the clip shape. `DesignConstants.CornerRadius.modal` is 28pt per SS2. Bottom corners remain square since the card is designed as a full-screen bottom sheet. Reference: SS2 Spacing and Layout.

The component signature accepts these parameters:

- `chapterLabel: String` (e.g., "Chapter 1")
- `title: String` (e.g., "The Runner")
- `body: String` (e.g., game description text)
- `imageName: String?` (optional asset catalog name; `nil` omits the image)
- `buttonLabel: String` (e.g., "Play", "Begin", "View Files")
- `action: () -> Void` (tap handler for the action button)

All content is injected. `ChapterCard` owns no text content and no state. Reference: SS4 SharedUI definition ("Reusable components: TypewriterText, ChapterCard, HapticButton").

The optional image, when present, is positioned right-aligned and may overlap the card edge slightly per the SS7 Chapter 1 Intro Card specification ("`ottie-game-intro` asset, right-aligned, overlapping the card edge slightly"). Use a `ZStack` or `.overlay` to position the image relative to the card bounds. When absent, the `ZStack` or overlay renders empty and the text stack fills the card width. Reference: SS7 Screen: Chapter 1 Intro Card.

The action button uses `Color.roseBlush` as its background fill, `Color.softWhite` for the label text, a fixed height of 56pt, and `DesignConstants.CornerRadius.card` (20pt) for its corner radius. The button spans the full available width within the card's horizontal padding. Reference: SS7 Intro Card ("'Play' button, full-width, `rose-blush` fill, `soft-white` label, 56pt height, 20pt corner radius").

Typography roles map to `Font+DesignTokens.swift` properties. Chapter label: `.system(size: 13, weight: .semibold, design: .default)` per SS2 Hint/Caption role at semibold weight. Title: `Font.displayTitle` at 32pt. Body: `Font.body` at 16pt. These are defined in E1.1. Reference: SS2 Typography table.
