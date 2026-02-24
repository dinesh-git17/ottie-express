# E7.3: Build Tape Playback UI

**Phase:** 7 - Chapter 3: The Cassette
**Class:** Feature
**Design Doc Reference:** SS9 (Tape Playing screen layout, VU meter, cassette reels, progress bar)
**Dependencies:**

- Phase 2: Navigation Shell and Shared UI (exit criteria met)
- E1.1: Define design system tokens (color and typography tokens available)
- E7.1: Build hint screen and permission flow (Chapter3Constants available)
- Asset: `vu-meter` (available in asset catalog)
- Asset: `cassette-tape` (available in asset catalog)
- Asset: `cassette-reel-l` (available in asset catalog)
- Asset: `cassette-reel-r` (available in asset catalog)

---

## Goal

Implement the tape playing screen layout with VU meter panel whose needles respond to audio level input, cassette with dual reel animation driven by playback progress, and a tape-style progress bar.

---

## Scope

### File Inventory

| File                                                 | Action | Responsibility                                                                  |
| ---------------------------------------------------- | ------ | ------------------------------------------------------------------------------- |
| `OttieExpress/Chapters/Chapter3/CassetteView.swift`  | Create | Cassette body with dual reel animation driven by playback progress (0.0 to 1.0) |
| `OttieExpress/Chapters/Chapter3/VUMeterView.swift`   | Create | VU meter panel with two needle gauges animated by audio level input             |
| `OttieExpress/Chapters/Chapter3/ReelAnimation.swift` | Create | Reel rotation and fill/deplete visual logic driven by normalized progress       |

### Integration Points

**CassetteView.swift**

- Imports from: `ReelAnimation.swift`, `Chapter3Constants.swift`, `Color+DesignTokens.swift`
- Imported by: `TapePlayerView.swift` (E7.4), `VoiceUnlockView.swift` (E7.2 reuses cassette layout)
- State reads: None (receives `progress: Double` and `audioLevel: Double` as parameters)
- State writes: None

**VUMeterView.swift**

- Imports from: `Chapter3Constants.swift`, `Color+DesignTokens.swift`
- Imported by: `TapePlayerView.swift` (E7.4)
- State reads: None (receives `audioLevel: Double` as parameter)
- State writes: None

**ReelAnimation.swift**

- Imports from: `Chapter3Constants.swift`
- Imported by: `CassetteView.swift`
- State reads: None (receives `progress: Double` as parameter)
- State writes: None

---

## Out of Scope

- Audio playback engine and `AVAudioPlayer` management (covered by E7.4)
- Floating phrase overlay during playback (covered by E7.4)
- BGM underlay fade-in logic (covered by E7.4)
- Cassette engagement slide-down transition animation (covered by E7.5)
- Audio level metering from `AVAudioPlayer` (E7.4 provides the level values; this epic consumes them)
- Screen color temperature transitions during engagement (covered by E7.5)
- Adding audio level metering or normalization utilities to AudioService (AudioService interface owned by E1.3)
- VU meter calibration against real audio output profiles (deferred to Phase 11 hardening)

---

## Definition of Done

- [ ] `VUMeterView` renders the `vu-meter` panel asset at top center of the tape playing screen
- [ ] Two needle gauges within `VUMeterView` rotate between a resting angle and a peak angle proportional to the `audioLevel` parameter (0.0 to 1.0)
- [ ] Needle rotation uses spring animation (stiffness 280, damping 22 baseline per SS2) for natural physical response
- [ ] VU needles settle to resting position (zero angle) when `audioLevel` drops to 0.0
- [ ] `CassetteView` renders below the VU meter with `cassette-reel-l` and `cassette-reel-r` overlaid on the cassette body
- [ ] Left reel visual fill starts at 1.0 and depletes to 0.0 as `progress` advances from 0.0 to 1.0
- [ ] Right reel visual fill starts at 0.0 and fills to 1.0 as `progress` advances from 0.0 to 1.0
- [ ] Reel rotation speed correlates with playback state: reels spin continuously during playback and stop when progress is static
- [ ] Progress bar renders below the cassette styled as a tape transfer indicator (thin horizontal bar), no timestamp labels
- [ ] Progress bar fill advances linearly from left to right matching the `progress` parameter
- [ ] All components accept external `progress` and `audioLevel` values, enabling the playback controller (E7.4) to drive them

---

## Implementation Notes

Per design doc SS9, the VU meter uses the `vu-meter` asset as the panel background. The needle gauges are drawn programmatically using `rotationEffect` on a needle shape or image, anchored at the bottom pivot point. Needle deflection maps `audioLevel` 0.0 to a resting angle (approximately -45 degrees from vertical) and `audioLevel` 1.0 to a peak angle (approximately +45 degrees from vertical). The exact resting and peak angles are defined as named constants in `Chapter3Constants`.

Reel fill/deplete animation uses scale or mask transforms on the reel assets. `cassette-reel-l` at progress 0.0 renders at full scale; at progress 1.0 renders at minimum scale (empty spool). `cassette-reel-r` inverts this relationship. Define the minimum and maximum reel scale factors as named constants in `Chapter3Constants`.

The progress bar has no text, no timestamps, and no percentage indicator per SS9 ("Timestamps not shown. Progress is visual only"). Use a simple `GeometryReader`-based fill bar with `warm-gold` fill color on a `muted-gray` track.

All three views are pure presentation components. They receive their driving values as parameters and perform no audio I/O or state mutation. This separation ensures E7.3 and E7.4 maintain clean boundaries per the skill's file boundary rule.
