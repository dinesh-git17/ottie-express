# Multi-Player Coordination Reference

Architecture patterns for running multiple concurrent audio sources with deterministic synchronization.

## AVAudioPlayer Multi-Layer Architecture

### Layer Definition

Define layers as a structured type, not loose arrays:

```swift
struct AudioLayer {
    let player: AVAudioPlayer
    let role: LayerRole
    let baseVolume: Float

    enum LayerRole {
        case voiceNote
        case instrumental
        case ambientTexture
    }
}
```

### Volume Constants

```swift
enum AudioMixConstants {
    static let voiceNoteVolume: Float = 1.0
    static let instrumentalVolume: Float = 0.35
    static let ambientTextureVolume: Float = 0.15
    static let fadeInDuration: TimeInterval = 0.8
    static let fadeOutDuration: TimeInterval = 0.5
    static let crossfadeDuration: TimeInterval = 1.2
    static let crossfadeSteps: Int = 60
    static let synchronizedStartDelay: TimeInterval = 0.25
}
```

### Synchronized Start Sequence

```swift
func startAllLayers() throws {
    let session = AVAudioSession.sharedInstance()
    try session.setActive(true)

    // 1. Prepare all players (preload buffers, acquire hardware)
    for layer in layers {
        guard layer.player.prepareToPlay() else {
            throw AudioLayerError.preparationFailed(layer.role)
        }
        layer.player.volume = layer.baseVolume
    }

    // 2. Schedule synchronized start on hardware clock
    guard let referencePlayer = layers.first?.player else { return }
    let startTime = referencePlayer.deviceCurrentTime + AudioMixConstants.synchronizedStartDelay

    for layer in layers {
        layer.player.play(atTime: startTime)
    }
}
```

### Pause and Resume All Layers

```swift
private var savedPositions: [LayerRole: TimeInterval] = [:]

func pauseAllLayers() {
    for layer in layers {
        savedPositions[layer.role] = layer.player.currentTime
        layer.player.pause()
    }
}

func resumeAllLayers() throws {
    let session = AVAudioSession.sharedInstance()
    try session.setCategory(.playback, mode: .default)
    try session.setActive(true)

    for layer in layers {
        if let position = savedPositions[layer.role] {
            layer.player.currentTime = position
        }
        guard layer.player.prepareToPlay() else {
            rebuildPlayer(for: layer.role)
            continue
        }
    }

    // Synchronized resume
    guard let ref = layers.first?.player else { return }
    let startTime = ref.deviceCurrentTime + AudioMixConstants.synchronizedStartDelay
    for layer in layers {
        layer.player.play(atTime: startTime)
    }

    savedPositions.removeAll()
}
```

### Player Lifecycle Rules

- Store `AVAudioPlayer` instances as strong properties on the coordinating object. Local variables are deallocated, silently stopping playback.
- Set `delegate = nil` on any player before releasing it. The delegate property uses `assign`/`unowned` semantics.
- For looping layers (instrumental, tape hiss): `player.numberOfLoops = -1`.
- Use WAV or CAF format for looping layers. MP3 and AAC codecs add encoder padding that creates audible gaps at loop boundaries.

### Player Rebuild

When `prepareToPlay()` returns `false` after interruption, the player failed to acquire hardware. Rebuild:

```swift
func rebuildPlayer(for role: LayerRole) {
    guard let url = audioURL(for: role) else { return }
    do {
        let newPlayer = try AVAudioPlayer(contentsOf: url)
        newPlayer.volume = baseVolume(for: role)
        newPlayer.numberOfLoops = loopCount(for: role)
        newPlayer.delegate = self
        replaceLayer(role: role, player: newPlayer)
    } catch {
        // Log structured error
    }
}
```

## AVAudioEngine Multi-Node Architecture

Use when per-track effects, submixers, or more than 3 sources are needed.

### Node Graph

```
[Voice PlayerNode] ─── [Voice EQ] ─── [Voice Submixer] ──┐
[Music PlayerNode] ─── [Music EQ] ─── [Music Submixer] ──┼── [Main Mixer] ── [Output]
[Hiss PlayerNode]  ─── [Hiss EQ]  ─── [Ambience Submixer]┘
```

Insert `AVAudioUnitEQ` between each player and its submixer. Use `globalGain` for volume control — `AVAudioPlayerNode.volume` has a known iOS bug where it has no effect in some versions.

### Engine Setup

```swift
func buildAudioGraph() {
    engine = AVAudioEngine()
    let mainMixer = engine.mainMixerNode

    for source in audioSources {
        let playerNode = AVAudioPlayerNode()
        let eq = AVAudioUnitEQ()

        engine.attach(playerNode)
        engine.attach(eq)

        engine.connect(playerNode, to: eq, format: source.format)
        engine.connect(eq, to: mainMixer, format: source.format)

        eq.globalGain = decibelGain(for: source.role)
    }
}
```

### Decibel Gain Mapping

Linear volume to dB for `AVAudioUnitEQ.globalGain`:

```swift
func decibelGain(linearVolume: Float) -> Float {
    guard linearVolume > 0 else { return -96.0 } // effective silence
    return 20.0 * log10(linearVolume)
}
```

| Layer        | Linear Volume | dB Gain |
| ------------ | ------------- | ------- |
| Voice Note   | 1.0           | 0.0     |
| Instrumental | 0.35          | -9.1    |
| Tape Hiss    | 0.15          | -16.5   |

### Synchronized Scheduling with AVAudioTime

For frame-accurate sync:

```swift
let sampleRate = audioFile.processingFormat.sampleRate
let futureHostTime = mach_absolute_time() + UInt64(0.1 * Double(NSEC_PER_SEC))
let startTime = AVAudioTime(hostTime: futureHostTime)

for (playerNode, audioFile) in playerFiles {
    playerNode.scheduleFile(audioFile, at: startTime)
    playerNode.play()
}
```

### Engine Interruption Recovery

```swift
func rebuildAndRestart() throws {
    engine.stop()

    // Detach all nodes
    for node in attachedNodes {
        engine.detach(node)
    }

    // Rebuild graph
    buildAudioGraph()

    // Restart
    try engine.start()

    // Reschedule audio
    for (playerNode, audioFile) in playerFiles {
        playerNode.scheduleFile(audioFile, at: nil)
        playerNode.play()
    }
}
```

### Engine vs Player Decision Matrix

| Criterion             | AVAudioPlayer      | AVAudioEngine            |
| --------------------- | ------------------ | ------------------------ |
| Concurrent sources    | 1–3                | 3+                       |
| Per-track effects     | No                 | Yes                      |
| Submixer grouping     | No                 | Yes                      |
| Realtime gain control | Limited            | Full (EQ node)           |
| Complexity            | Low                | Moderate                 |
| Interruption recovery | Simpler            | Requires graph rebuild   |
| Memory footprint      | Lower per instance | Higher (engine overhead) |
