---
name: spritekit-game-architecture
description: Enforce production-safe SpriteKit game architecture for iOS infinite runner games. Use when writing or reviewing SKScene subclasses, game loops, physics configurations, texture management, parallax scrolling, or SwiftUI-SpriteKit integration via SpriteView. Triggers on SpriteKit code generation, game scene architecture, collision detection setup, sprite animation, or game loop timing implementation.
---

# SpriteKit Game Architecture

Enforce deterministic, memory-safe, performance-correct SpriteKit architecture for production iOS games. All SpriteKit code in this repository must comply with the constraints below. Violations are blocking.

## Authority

This skill governs all SpriteKit-related code. Constraints are non-negotiable. If a pattern below conflicts with a developer's preference, this skill wins. Consult [references/spritekit-standards.md](references/spritekit-standards.md) for detailed technical rationale.

## SKScene Lifecycle

### Initialization Boundary

`init(size:)` and `init(coder:)` execute before the scene is attached to any `SKView`. At init time, `self.view` is `nil`.

**Permitted in init:**

- Static node hierarchy construction (sprites, labels, shape nodes)
- Physics body configuration (category, collision, contact bitmasks)
- Texture pre-resolution from atlas references
- Action sequence construction (not execution)

**Prohibited in init:**

- Any access to `self.view` (force-unwrap crash)
- Layout calculations referencing view bounds or safe area
- Camera constraint configuration dependent on viewport dimensions

**`didMove(to:)` is the activation boundary.** View-dependent configuration belongs here exclusively. Guard against duplicate execution if the scene instance is re-presented:

```swift
private var didSetup = false

override func didMove(to view: SKView) {
    guard !didSetup else { return }
    didSetup = true
    // View-dependent configuration here
}
```

### Teardown

Override `willMove(from:)` for deterministic cleanup:

```swift
override func willMove(from view: SKView) {
    removeAllActions()
    removeAllChildren()
    physicsWorld.contactDelegate = nil
}
```

Verify `deinit` fires. If it does not, a retain cycle exists. Common culprits: closures capturing `self` strongly in `SKAction.customAction`, `physicsWorld.contactDelegate` holding a strong reference externally, or SwiftUI `@State` retaining the scene.

## Texture Management

### Loading Rules

- **Pre-resolve all textures at load time.** Store `SKTexture` arrays as instance properties.
- **Never call `SKTexture(imageNamed:)` or `SKTextureAtlas.textureNamed(_:)` inside `update(_:)`.** Name resolution has measurable per-call overhead.
- **Preload atlases before scene presentation** using `SKTextureAtlas.preloadTextureAtlases(_:withCompletionHandler:)`. Dispatch to main queue inside the handler.
- **Group atlas textures by usage context** (per-scene, per-level), not by sprite type. A single mega-atlas forces unrelated textures into memory.

### Memory Discipline

- Atlas preloading temporarily doubles memory (compressed + decompressed copies coexist). Budget accordingly on memory-constrained devices.
- SpriteKit's internal texture cache persists until the `SKView` deallocates or a low-memory warning fires. There is no public cache-flush API.
- Set `texture.filteringMode = .nearest` for pixel-art to prevent bilinear interpolation artifacts at tile boundaries.

### Animation Pattern

```swift
// Load once, at scene init or preload phase
private let runAtlas = SKTextureAtlas(named: "player_run")
private lazy var runFrames: [SKTexture] = {
    (1...8).map { runAtlas.textureNamed("run_\($0)") }
}()

// Animate with pre-resolved frames
func startRunAnimation(on sprite: SKSpriteNode) {
    let action = SKAction.animate(with: runFrames, timePerFrame: 0.08)
    sprite.run(.repeatForever(action), withKey: "run")
}
```

## Physics and Collision Detection

### Bitmask Architecture

Define categories as a `UInt32` constant struct. Use explicit bit positions. Never rely on the default all-ones masks.

```swift
struct PhysicsCategory {
    static let none:     UInt32 = 0
    static let player:   UInt32 = 1 << 0
    static let ground:   UInt32 = 1 << 1
    static let obstacle: UInt32 = 1 << 2
    static let collectible: UInt32 = 1 << 3
}
```

**Configuration rules:**

- Set `collisionBitMask` to only categories the body must physically bounce off.
- Set `contactTestBitMask` to only categories that require delegate callbacks.
- Set `categoryBitMask` to exactly one category per body type.
- Default `collisionBitMask` is `0xFFFFFFFF` (everything collides). Always override it explicitly.

### Contact Delegate

Body ordering in `SKPhysicsContact` is **not guaranteed**. Normalize before dispatch:

```swift
func didBegin(_ contact: SKPhysicsContact) {
    let mask = contact.bodyA.categoryBitMask | contact.bodyB.categoryBitMask
    switch mask {
    case PhysicsCategory.player | PhysicsCategory.obstacle:
        handlePlayerObstacleContact(contact)
    case PhysicsCategory.player | PhysicsCategory.collectible:
        handlePlayerCollectibleContact(contact)
    default:
        break
    }
}
```

### Deferred Node Removal

**Never call `removeFromParent()` inside `didBegin(_:)`.** Multiple contacts for the same body can fire in a single frame. Removing a node mid-callback causes subsequent callbacks to reference a detached node.

```swift
private var pendingRemovals: [SKNode] = []

func didBegin(_ contact: SKPhysicsContact) {
    guard let nodeA = contact.bodyA.node,
          let nodeB = contact.bodyB.node else { return }
    pendingRemovals.append(nodeB)
}

override func didFinishUpdate() {
    pendingRemovals.forEach { $0.removeFromParent() }
    pendingRemovals.removeAll()
}
```

### Performance Rules

- Prefer circle physics bodies (3-4x cheaper than rectangles).
- Set `isDynamic = false` on static geometry (ground, walls, platforms).
- Enable `usesPreciseCollisionDetection` only on fast, small projectiles. Never on static or slow bodies.
- Minimize `contactTestBitMask` coverage. Every set bit increases callback frequency.

## Parallax Scrolling

### Layer Architecture

Each parallax depth is a container `SKNode` holding tiled sprite children. Control z-ordering and speed independently per layer.

| Layer          | zPosition | Speed Factor | Content           |
| -------------- | --------- | ------------ | ----------------- |
| Far background | -40       | 0.0 (static) | Sky gradient      |
| Distant        | -30       | 0.25         | Mountains, clouds |
| Mid            | -20       | 0.5          | Trees, structures |
| Near           | -10       | 0.75         | Fences, props     |
| Gameplay       | 0         | 1.0          | Player, obstacles |
| Foreground     | 10        | 1.25         | Particle overlay  |

### Infinite Scroll Reset

Each layer requires `ceil(screenWidth / tileWidth) + 1` tile copies minimum. Reset position with modular arithmetic, not `fmod` (which accumulates floating-point drift over extended play sessions).

```swift
layer.position.x -= layer.speed * CGFloat(dt)
if layer.position.x <= -layer.tileWidth {
    layer.position.x += layer.tileWidth
}
```

### Seam Prevention

- Overlap tiles by 1 point to mask sub-pixel gaps from position rounding.
- Set `anchorPoint = .zero` on all tile sprites for predictable arithmetic.
- Ensure tile left/right edge pixels are identical in the source texture.
- Use `.nearest` filtering for pixel-art styles.

## Game Loop Timing

### Delta Time Calculation

SpriteKit does not provide delta time. Compute it from the absolute `currentTime` parameter:

```swift
private var lastUpdateTime: TimeInterval = 0

override func update(_ currentTime: TimeInterval) {
    let dt: TimeInterval
    if lastUpdateTime == 0 {
        dt = 0
    } else {
        dt = currentTime - lastUpdateTime
    }
    lastUpdateTime = currentTime

    updateGameState(deltaTime: dt)
}
```

### Pause Recovery

When the scene resumes from pause, `currentTime` has advanced by the entire pause duration. Without the `lastUpdateTime == 0` guard, `dt` becomes enormous, causing teleportation. Reset `lastUpdateTime` to zero on pause. Alternatively, clamp:

```swift
let dt = min(currentTime - lastUpdateTime, 1.0 / 60.0)
```

### Frame Independence

**All position updates must multiply by delta time.** Without this, movement speed is tied to frame rate. A 120Hz ProMotion iPad runs at 2x speed compared to a 60Hz device.

```swift
node.position.x += speed * CGFloat(dt)
```

### Pause Semantics

- `SKScene.isPaused = true`: Actions and physics pause. **`update(_:)` still fires.** CPU is not saved.
- `SKView.isPaused = true`: The entire rendering loop halts. `update(_:)` stops. Use this for true pause.

### Timing Source

Never use `Date()`, `DispatchTime`, or `Timer` for game timing. These clocks are not synchronized with the display refresh cycle. The `currentTime` parameter in `update(_:)` is derived from the display link timebase and is the only correct source for frame-coupled logic.

## SwiftUI Integration

### SpriteView Scene Retention

`SpriteView` creates an internal `SKView` on init. If SwiftUI re-evaluates the view body (any state change), and the `SpriteView` initializer is re-invoked, a new internal `SKView` is created. The old scene instance leaks.

**Mandatory pattern: retain the scene in a `@StateObject`.**

```swift
final class SceneStore: ObservableObject {
    let scene: GameScene

    init() {
        let s = GameScene(size: CGSize(width: 390, height: 844))
        s.scaleMode = .resizeFill
        self.scene = s
    }
}

struct GameView: View {
    @StateObject private var store = SceneStore()

    var body: some View {
        SpriteView(scene: store.scene)
    }
}
```

`@StateObject` survives SwiftUI view re-evaluations. The scene instance persists across body recomputation.

### Prohibited Patterns

- **Inline scene creation in `body`:** `SpriteView(scene: GameScene())` recreates the scene on every state change.
- **`@State` scene storage:** `@State` wraps value types. `SKScene` is a reference type; `@State` does not prevent re-init.
- **Binding `isPaused` to SwiftUI state via `SpriteView` init parameter:** Changing the binding recreates the `SpriteView`. Control pause state by setting `scene.isPaused` or `scene.view?.isPaused` directly.

### Communication Boundary

Scene-to-SwiftUI communication must use a coordinator with a `weak` back-reference:

```swift
final class GameCoordinator: ObservableObject {
    @Published var score: Int = 0
    let scene: GameScene

    init() {
        let s = GameScene(size: CGSize(width: 390, height: 844))
        s.scaleMode = .resizeFill
        self.scene = s
        s.coordinator = self
    }
}

class GameScene: SKScene {
    weak var coordinator: GameCoordinator?
}
```

The scene holds a `weak` reference to the coordinator. The coordinator holds a strong reference to the scene. No retain cycle.

### Closure Safety

All closures passed to `SKAction.customAction`, `SKAction.run`, or completion handlers that capture `self` must use `[weak self]`:

```swift
let action = SKAction.run { [weak self] in
    self?.handleEvent()
}
```

## Performance Checklist

Before shipping any SpriteKit scene:

- [ ] `view.ignoresSiblingOrder = true` (enables draw call batching)
- [ ] All textures pre-resolved from atlas references (no per-frame name lookup)
- [ ] Atlas textures grouped by scene/usage (not by sprite type)
- [ ] Draw calls under 20 per frame (`showsDrawCount`)
- [ ] Node count stable (off-screen nodes removed, not hidden)
- [ ] Circle physics bodies where precision is not required
- [ ] `isDynamic = false` on all static geometry
- [ ] Delta time applied to all movement calculations
- [ ] `lastUpdateTime` reset handled for pause/resume
- [ ] Scene retained via `@StateObject`, not inline construction
- [ ] `deinit` verified to fire on scene transitions
- [ ] No `SKEffectNode` without `shouldRasterize = true` on static content
- [ ] `blendMode = .replace` on fully opaque sprites

## References

- **Technical Standards**: See [references/spritekit-standards.md](references/spritekit-standards.md) for exhaustive technical rationale, frame cycle documentation, and edge case handling.
