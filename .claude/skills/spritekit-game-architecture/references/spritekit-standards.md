# SpriteKit Technical Standards Reference

Exhaustive technical rationale for every constraint in the SpriteKit Game Architecture skill. This document is the authoritative source for "why" behind each rule.

## Frame Cycle Execution Order

Every frame executes in this exact sequence (per Apple SpriteKit Programming Guide, WWDC 2014/608):

1. `update(_ currentTime: TimeInterval)` — game logic entry point
2. Scene evaluates all running `SKAction` instances on the node tree
3. `didEvaluateActions()` — post-action hook
4. Physics simulation runs; `SKPhysicsContactDelegate` callbacks fire
5. `didSimulatePhysics()` — post-physics hook
6. Scene applies `SKConstraint` objects
7. `didApplyConstraints()` — post-constraint hook
8. `didFinishUpdate()` — final pre-render hook
9. Scene renders to the `SKView`

Each method has a corresponding `SKSceneDelegate` method, enabling composition without subclassing. `didFinishUpdate()` is the safe point for deferred operations (node removal, state synchronization) because no further simulation occurs before render.

## Scene Initialization Timing

### `init(size:)` / `init(coder:)` State

| Property            | Value        | Notes                        |
| ------------------- | ------------ | ---------------------------- |
| `self.view`         | `nil`        | Not attached to any `SKView` |
| `self.size`         | Set          | Passed via initializer       |
| `self.physicsWorld` | Initialized  | But no rendering context     |
| `self.scene`        | `self`       | Root scene reference valid   |
| `self.camera`       | `nil` or set | If assigned in init          |
| `self.children`     | Empty        | Unless populated in init     |

### `didMove(to view:)` State

| Property            | Value                  | Notes                     |
| ------------------- | ---------------------- | ------------------------- |
| `self.view`         | Non-nil, valid         | Guaranteed by framework   |
| `self.size`         | Final (post-scaleMode) | May differ from init size |
| Safe area insets    | Available              | Via `view.safeAreaInsets` |
| Gesture recognizers | Attachable             | View exists for targets   |

### `willMove(from view:)` State

Fires just before scene detachment. `self.view` is still valid during this call. After this method returns, `self.view` becomes `nil`. This is the last opportunity for view-dependent cleanup.

## Texture Cache Lifecycle

SpriteKit maintains an opaque internal texture cache on the `SKView`. Cache behavior:

- **Lazy loading**: `SKTexture` objects load image data on first render demand, not on construction.
- **Preloading**: `preload(completionHandler:)` forces disk I/O, decompression, and GPU upload on a background thread. The completion handler fires on an **arbitrary queue** — dispatch to main for SpriteKit operations.
- **Cache persistence**: Textures remain cached until `SKView` deallocation or system low-memory pressure. There is no public API to flush the cache.
- **Preload memory cost**: During preloading, both the compressed source and decompressed GPU-ready copy exist simultaneously, roughly doubling peak memory for the atlas.

### Name Resolution Cost

`SKTexture(imageNamed:)` performs a multi-stage lookup on every call:

1. Checks Asset Catalog
2. Checks `.spriteatlas` bundles
3. Checks loose PNG/JPEG files in the main bundle
4. Checks `.atlas` folders

Even with internal caching, this lookup has measurable overhead when called per-frame. Pre-resolving via `SKTextureAtlas(named:).textureNamed(_:)` eliminates the search path.

### Atlas Sizing

| Device Generation | Max Atlas Texture Size |
| ----------------- | ---------------------- |
| A6 and earlier    | 2048x2048              |
| A7–A10            | 4096x4096              |
| A11+              | 8192x8192              |

Exceeding the max causes Xcode to split into multiple atlas pages, breaking single-draw-call batching for sprites within the atlas.

## Physics Body Cost Hierarchy

From cheapest to most expensive per-frame evaluation (WWDC 2014/608):

1. **Circle** (`init(circleOfRadius:)`) — cheapest; 3-4x faster than rectangles
2. **Rectangle** (`init(rectangleOf:)`)
3. **Polygon** (`init(polygonFrom:)`) — cost scales with vertex count
4. **Compound** (`init(bodies:)`) — union of multiple shapes
5. **Alpha mask** (`init(texture:size:)`) — pixel-level; avoid in production unless required

### `isDynamic` Optimization

Setting `isDynamic = false` removes a body from the velocity/force integration step. The body still participates as a collision boundary. For static geometry (ground planes, walls, platforms), this is a substantial per-frame cost reduction.

### Bitmask Partitioning

Physics engine evaluates potential contacts as N-squared pairwise checks. Setting restrictive `collisionBitMask` and `contactTestBitMask` values allows the engine to skip pairs with no overlapping bits. In typical scenes, proper partitioning reduces comparisons by ~50%.

### `usesPreciseCollisionDetection`

Default collision detection evaluates at discrete per-frame positions. Fast-moving small bodies (bullets, projectiles) can tunnel through thin obstacles between frames. `usesPreciseCollisionDetection = true` switches to swept-volume (continuous) detection.

Cost: high. The algorithm traces the body's path between frames and checks for intersections along the sweep. Enable only on fast-moving bodies, never on static or slow-moving ones. Cost scales with the total number of precise bodies in the scene.

## Draw Call Batching Conditions

SpriteKit batches consecutive draw calls when all conditions are met for a group of sprites (WWDC 2014/608, WWDC 2016/610):

1. **Same texture** (same atlas page)
2. **Same z-position** (or equivalent depth when `ignoresSiblingOrder = true`)
3. **Same blend mode**

Breaking any condition inserts a new draw call. A blend mode change forces a GPU state change. A texture change forces a texture bind.

### `ignoresSiblingOrder`

Default is `false`, which forces SpriteKit to respect child array insertion order for draw ordering. This prevents optimal batching because sprites may be interleaved by texture in insertion order.

Setting `ignoresSiblingOrder = true` allows SpriteKit to sort sprites by z-position and texture for optimal batching. This is mandatory for production scenes with more than a handful of sprites.

### `blendMode = .replace`

Default `.alpha` blend mode reads the existing framebuffer pixel, composites with alpha, and writes back. For fully opaque sprites (no transparency), `.replace` skips the framebuffer read, saving one memory operation per pixel. Measurable on fill-rate-limited devices (older iPads, iPhone SE).

## SKEffectNode Cost

`SKEffectNode` renders its entire child subtree into a private offscreen framebuffer, optionally applies a `CIFilter`, then composites the result. This happens **every frame** by default.

- `shouldEnableEffects = false`: Disables the offscreen pass entirely; node behaves as regular `SKNode`.
- `shouldRasterize = true`: Caches the offscreen result as a texture. Subsequent frames reuse the cache without re-rendering the subtree. **Invalidated when any child changes** (position, texture, action state).

Production rule: use `shouldRasterize = true` exclusively on subtrees that change infrequently (static backgrounds, UI chrome). Running an action on any child of a rasterized effect node forces full re-render, negating the optimization.

Each `SKEffectNode` and `SKCropNode` costs at minimum 2 draw calls (offscreen render + composite).

## Parallax Drift Prevention

### Floating-Point Accumulation

Naive approaches using `fmod` or `truncatingRemainder(dividingBy:)` accumulate floating-point error over extended play sessions (30+ minutes). The error manifests as visible 1-2 pixel seam flicker between tiles.

The modular reset pattern (`position.x += tileWidth` when past threshold) avoids this because it performs a single addition against a known constant rather than a division-based remainder.

### Sub-Pixel Gap Mitigation

Sprite positions are stored as `CGFloat`. GPU rasterization snaps to physical pixels. When a position falls between pixels, bilinear filtering interpolates, creating a 1-pixel-wide semi-transparent seam between adjacent tiles.

Mitigations:

- Overlap tiles by 1 point
- Set `anchorPoint = .zero` (eliminates half-size offset arithmetic that introduces additional rounding)
- Use `.nearest` filtering for pixel-art (eliminates interpolation entirely)

## SwiftUI SpriteView Internals

### Recreation Bug

`SpriteView` captures its `scene` parameter at init time. SwiftUI may re-init `SpriteView` when:

- Any `@State` or `@Binding` that the parent view depends on changes
- The parent view's `body` is re-evaluated for any reason
- The `isPaused` parameter (if bound to state) changes

Each re-init creates a new internal `SKView`. The old `SKView` (and its scene) is not guaranteed to be released, causing memory leaks. The new `SpriteView` presents the scene to a fresh internal `SKView`, triggering `didMove(to:)` again on the same scene instance (if using `@StateObject`) or on a new scene instance (if creating inline).

### `@StateObject` vs `@State` for Scene Storage

`@StateObject` is initialized once per SwiftUI view identity lifetime. It survives view re-evaluations. This is the correct storage for `SKScene` instances.

`@State` is designed for value types. While it technically works with reference types, it does not prevent the scene from being re-created if the SwiftUI view struct is re-initialized (which happens independently of `@State` storage).

### Retain Cycle Patterns

| Pattern                                                                  | Cycle? | Reason                                                            |
| ------------------------------------------------------------------------ | ------ | ----------------------------------------------------------------- |
| Scene holds strong ref to coordinator                                    | Yes    | Coordinator holds scene via `SceneStore`                          |
| Scene holds `weak` ref to coordinator                                    | No     | Weak breaks the cycle                                             |
| `SKAction.run { self.doThing() }`                                        | Yes    | Closure captures `self` strongly; action is retained by node tree |
| `SKAction.run { [weak self] in self?.doThing() }`                        | No     | Weak capture                                                      |
| `Timer.scheduledTimer(withTimeInterval:repeats:block:)` capturing `self` | Yes    | Timer retained by run loop; closure retains scene                 |

## Pause Semantics

| Mechanism               | `update(_:)`    | Actions | Physics | Rendering |
| ----------------------- | --------------- | ------- | ------- | --------- |
| `scene.isPaused = true` | **Still fires** | Paused  | Paused  | Active    |
| `view.isPaused = true`  | Stopped         | Stopped | Stopped | Stopped   |
| App backgrounded        | Stopped         | Stopped | Stopped | Stopped   |

Setting `scene.isPaused` does not save CPU on the update loop. For true pause (menu screens, app interruption), pause the `SKView`.

## Frame Budget Reference

| Refresh Rate       | Frame Budget | Use Case                                       |
| ------------------ | ------------ | ---------------------------------------------- |
| 120 Hz (ProMotion) | 8.33 ms      | iPad Pro, iPhone 13 Pro+                       |
| 60 Hz              | 16.67 ms     | Standard devices                               |
| 30 Hz              | 33.33 ms     | Menu/idle (set via `preferredFramesPerSecond`) |

Since iOS 10, SpriteKit uses dirty-scene rendering: if nothing changes, no draw calls occur. Set `preferredFramesPerSecond = 30` on menu scenes to save battery.

## Particle Emitter Guidelines

`SKEmitterNode` is the most GPU-intensive standard node type. Guidelines:

- Keep `particleBirthRate` under 500
- Prefer larger particles at lower birth rates over fine particles at high rates
- Remove emitters that leave the viewport; `isHidden = true` does not stop particle simulation
- For one-shot effects (explosions), use `numParticlesToEmit` and remove the emitter after completion via `SKAction.sequence([.wait(forDuration:), .removeFromParent()])`

## Lighting Constraints

- Maximum 8 `SKLightNode` instances affect any single sprite
- Ambient lighting (`ambientColor` on `SKLightNode`) is free
- Shadow casting cost scales linearly with light count
- `SKLightNode.categoryBitMask` controls which sprites a light affects; use it to limit light scope

## Source Cross-Reference

| Topic              | Apple Documentation                                                                                                                   | WWDC Session       |
| ------------------ | ------------------------------------------------------------------------------------------------------------------------------------- | ------------------ |
| Frame cycle        | [Responding to Frame-Cycle Events](https://developer.apple.com/documentation/spritekit/responding-to-frame-cycle-events)              | 2014/608           |
| Texture preloading | [Preloading Textures into Memory](https://developer.apple.com/documentation/spritekit/sktexture/preloading_textures_into_memory)      | 2014/608           |
| Atlas batching     | [SKTextureAtlas](https://developer.apple.com/documentation/spritekit/sktextureatlas)                                                  | 2014/608, 2016/610 |
| Physics bodies     | [Getting Started with Physics Bodies](https://developer.apple.com/documentation/spritekit/sknode/getting_started_with_physics_bodies) | 2014/608           |
| Contact delegate   | [SKPhysicsContactDelegate](https://developer.apple.com/documentation/spritekit/skphysicscontactdelegate)                              | —                  |
| SpriteView         | [SpriteView](https://developer.apple.com/documentation/spritekit/spriteview)                                                          | —                  |
| Performance        | [Maximizing Texture Performance](https://developer.apple.com/documentation/spritekit/maximizing-texture-performance)                  | 2014/608, 2016/610 |
| SKEffectNode       | [SKEffectNode](https://developer.apple.com/documentation/spritekit/skeffectnode)                                                      | 2014/608           |
