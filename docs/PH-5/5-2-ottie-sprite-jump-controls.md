# E5.2: Build Ottie Sprite and Jump Controls

**Phase:** 5 - Chapter 1: The Runner
**Class:** Feature
**Design Doc Reference:** §7 (Ottie Sprite, Jump, Hit), §2 (Motion Principles)
**Dependencies:**

- Phase 2: Navigation Shell and Shared UI (exit criteria met)
- E1.1: Define design system tokens (motion constants available)
- E5.1: Scaffold SpriteKit scene and parallax world (`RunnerScene` subclass with update loop and ground plane, `Chapter1Constants` defined)
- Asset: `ottie-run-01` (available in asset catalog)
- Asset: `ottie-run-02` (available in asset catalog)
- Asset: `ottie-run-03` (available in asset catalog)
- Asset: `ottie-run-04` (available in asset catalog)
- Asset: `ottie-jump` (available in asset catalog)
- Asset: `ottie-hit` (available in asset catalog)

---

## Goal

Implement the Ottie player sprite with 4-frame run cycle at 12fps, tap-to-jump mechanic with physics arc and double-jump blocking, and hit state with knockback animation.

---

## Scope

### File Inventory

| File                                                | Action | Responsibility                                                                                                                                                                           |
| --------------------------------------------------- | ------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `OttieExpress/Chapters/Chapter1/OttieSprite.swift`  | Create | SKSpriteNode subclass managing Ottie's visual state machine (running, jumping, hit), 4-frame run cycle animation, jump impulse application, hit knockback sequence, and ground detection |
| `OttieExpress/Chapters/Chapter1/InputHandler.swift` | Create | Touch event processor that receives scene-level touch input, validates Ottie's grounded state, and issues jump commands to OttieSprite                                                   |

### Integration Points

**OttieSprite.swift**

- Imports from: `Chapter1Constants.swift` (E5.1)
- Imported by: `RunnerScene.swift` (E5.1 scene adds OttieSprite as child), `InputHandler.swift`, `HeartCollectible.swift` (E5.3 collision), `ObstacleNode.swift` (E5.3 collision)
- State reads: None
- State writes: None
- Public interface: `OttieSprite()` initializer; `jump()` triggers jump if grounded; `applyHit()` triggers knockback; `isGrounded: Bool` read-only property; physics body with category bitmask for collision detection

**InputHandler.swift**

- Imports from: `OttieSprite.swift`, `Chapter1Constants.swift` (E5.1)
- Imported by: `RunnerScene.swift` (E5.1 delegates touch events)
- State reads: `OttieSprite.isGrounded`
- State writes: None (calls `OttieSprite.jump()`)

---

## Out of Scope

- Parallax scrolling, world setup, and scene lifecycle (covered by E5.1)
- Heart collectible spawning, obstacle spawning, and collision response logic (covered by E5.3; this epic defines the physics body and bitmask on Ottie but does not handle contact callbacks)
- HUD display, life tracking, game over, and game complete state (covered by E5.4)
- `ottie-celebrate` animation and game complete visual sequence (covered by E5.4)
- Audio playback on hit or jump (§7 does not specify jump SFX; hit SFX is owned by E5.3 collision handling)
- Design token modifications or additions to the shared Extensions directory (owned by E1.1; this epic consumes existing tokens without modification)
- Frame rate optimization or texture atlas packing for run cycle sprites (deferred to Phase 12 final validation)

---

## Definition of Done

- [ ] Ottie sprite renders on the ground plane, positioned horizontally in the left third of the screen as defined by `Chapter1Constants.ottieScreenXRatio`
- [ ] Run cycle animates through `ottie-run-01`, `ottie-run-02`, `ottie-run-03`, `ottie-run-04` at 12fps continuously during the running state
- [ ] Tapping anywhere on the scene triggers a jump with a vertical impulse producing a physics-based parabolic arc
- [ ] Ottie sprite displays `ottie-jump` frame for the full duration of airtime, from impulse to ground landing
- [ ] A second tap during airtime is ignored and produces no jump impulse (double jump blocked)
- [ ] On ground landing after a jump, Ottie immediately returns to the running state and the 4-frame run cycle resumes
- [ ] `applyHit()` swaps the sprite to `ottie-hit` frame and plays a brief horizontal knockback displacement before returning to the running state
- [ ] `isGrounded` returns true when Ottie's physics body is in contact with the ground plane and false during airtime
- [ ] OttieSprite exposes a physics body with a category bitmask that E5.3 collision detection can reference

---

## Implementation Notes

OttieSprite manages three visual states via a private enum:

```swift
private enum OttieState {
    case running
    case jumping
    case hit
}
```

The run cycle uses `SKAction.animate(with:timePerFrame:)` with `timePerFrame = 1.0 / Chapter1Constants.ottieRunFPS` (1/12 = ~0.083s). Wrap in `SKAction.repeatForever` and run as a named action key (`"runCycle"`). On state transitions to jumping or hit, remove the run cycle action. On return to running, restart it.

Jump physics: Apply `dy` impulse via `physicsBody?.applyImpulse(CGVector(dx: 0, dy: jumpImpulse))`. The scene's gravity provides the downward arc. Tune `jumpImpulse` as a named constant so the arc peaks at approximately 40% of screen height, giving clearance over mid-air storm clouds.

Ground detection: The ground plane (E5.1) has a static physics body. OttieSprite's physics body uses `contactTestBitMask` to detect ground contact. On `didBegin` contact with the ground category, set state to `.running`. The `isGrounded` property derives from the current state: true when `.running`, false otherwise.

Knockback: `applyHit()` sets state to `.hit`, swaps texture to `ottie-hit`, runs an `SKAction.moveBy(x: -30, y: 0, duration: 0.2)` followed by `SKAction.moveBy(x: 30, y: 0, duration: 0.15)` to return Ottie to position, then transitions back to `.running`. During the hit state, `InputHandler` ignores tap input (Ottie cannot jump while in knockback).

`InputHandler` receives `touchesBegan` from `RunnerScene` and checks `ottieSprite.isGrounded` plus the current state is not `.hit`. If both conditions pass, it calls `ottieSprite.jump()`. This is the sole input path; the scene does not process touches directly.

Physics category bitmasks should be defined as static constants on a shared `PhysicsCategory` enum or struct within `Chapter1Constants`:

| Category | Bitmask Value |
| -------- | ------------- |
| ground   | 0x1 << 0      |
| ottie    | 0x1 << 1      |
| heart    | 0x1 << 2      |
| obstacle | 0x1 << 3      |
