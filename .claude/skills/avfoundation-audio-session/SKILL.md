---
name: avfoundation-audio-session
description: Enforce production-grade AVFoundation audio session architecture for layered playback, interruption resilience, and deterministic resume behavior on iOS. Use when writing or reviewing AVAudioSession configuration, multi-player coordination, audio interruption handlers, route change observers, crossfade implementations, or background audio lifecycle code. Triggers on AVAudioSession setup, AVAudioPlayer/AVAudioEngine multi-track playback, audio interruption handling, volume crossfading, or audio route change management.
---

# AVFoundation Audio Session Management

Governing standard for all AVFoundation audio session and multi-track playback code in this repository. Every audio pipeline must survive interruptions deterministically, coordinate layered playback without drift, and resume without silent failures.

---

## 1. Audio Architecture Philosophy

Audio is infrastructure. It must be reliable, predictable, and resilient to every system event iOS can produce.

**Rules:**

- Audio sessions are configured once and activated at the moment of first playback. No session churn.
- Every interruption has a deterministic response. No unhandled notification paths.
- Multi-player coordination uses hardware clock synchronization, not wall-clock estimation.
- Volume relationships between layers are maintained through named constants, not ad-hoc floats.
- Silent failures are the worst class of audio bug. Every code path must either produce sound or produce a logged error.

---

## 2. AVAudioSession Configuration

### 2.1 Category Selection (BLOCKING)

| Scenario                               | Category         | Options                                | Background Audio |
| -------------------------------------- | ---------------- | -------------------------------------- | ---------------- |
| Primary content (sole audio source)    | `.playback`      | none                                   | Yes              |
| Layered playback alongside other apps  | `.playback`      | `.mixWithOthers`                       | Yes              |
| Layered playback that ducks other apps | `.playback`      | `.duckOthers`                          | Yes              |
| Non-essential ambient sounds           | `.ambient`       | none                                   | No               |
| Voice + playback (recording context)   | `.playAndRecord` | `.defaultToSpeaker`, `.allowBluetooth` | Yes              |

**For Ottie Express Chapter 3 (voice note + instrumental + tape hiss):**

```swift
try session.setCategory(.playback, mode: .default, options: [])
```

No `.mixWithOthers` — this app owns the audio output. Multiple `AVAudioPlayer` instances within the same session coexist automatically without options.

### 2.2 Mode Selection

| Mode             | Use When                                                                    |
| ---------------- | --------------------------------------------------------------------------- |
| `.default`       | General playback, mixed content                                             |
| `.spokenAudio`   | Primary content is speech (enables interruption by other spoken-audio apps) |
| `.moviePlayback` | Video content with audio                                                    |

Use `.default` unless the content is exclusively spoken word. `.spokenAudio` allows other apps with `.interruptSpokenAudioAndMixWithOthers` to interrupt your playback.

### 2.3 Activation Timing (BLOCKING)

```
App launch     → setCategory(.playback, mode: .default)    [configure only]
User taps play → setActive(true)                            [activate]
All audio done → setActive(false, options: [.notifyOthersOnDeactivation])
```

**Rules:**

- Configure category at initialization. Activate at first playback. Deactivate only when genuinely done.
- Never toggle `setActive` between short playback events. One activation per playback session.
- Always pass `.notifyOthersOnDeactivation` when deactivating so interrupted apps can resume.
- Handle `setActive(true)` failure — it throws `cannotInterruptOthers` (code 560557684). Implement retry with backoff.

### 2.4 Background Audio Requirements (BLOCKING)

Both mandatory. Missing either causes silent suspension:

1. `Info.plist`: `UIBackgroundModes` array contains `audio`.
2. `AVAudioSession` category is `.playback`.

### 2.5 Prohibited Configuration Patterns

- Repeatedly toggling `setActive(true)` / `setActive(false)` between tracks.
- Changing category while the session is active without deactivating first.
- Using `.soloAmbient` (system default) in an audio-focused app.
- Setting `.mixWithOthers` when the app is the primary audio source (loses Now Playing integration).

---

## 3. Interruption Handling

See [references/interruption-lifecycle.md](references/interruption-lifecycle.md) for full handler implementation.

### 3.1 Notification Registration (BLOCKING)

Register all five audio notifications at initialization. Missing any creates a silent failure path:

1. `AVAudioSession.interruptionNotification`
2. `AVAudioSession.routeChangeNotification`
3. `AVAudioSession.mediaServicesWereResetNotification`
4. `AVAudioSession.mediaServicesWereLostNotification`
5. `AVAudioSession.silenceSecondaryAudioHintNotification`

For `AVAudioEngine` users, also register `NSNotification.Name.AVAudioEngineConfigurationChange`.

### 3.2 Interruption State Machine

```
PLAYING ──[.began]──→ INTERRUPTED ──[.ended + shouldResume]──→ RESUMING ──→ PLAYING
                                   └─[.ended, no resume]──→ PAUSED
                                   └─[no .ended received]──→ detect via sceneDidBecomeActive
```

- On `.began`: Save playback positions for all active players. Update UI to paused. Do NOT restart.
- On `.ended` with `.shouldResume`: Reactivate session, restore positions, call `play()` on all layers.
- On `.ended` without `.shouldResume`: Stay paused. User must explicitly resume.
- No `.ended` received: Detect via `sceneDidBecomeActive`, inspect session state.

### 3.3 InterruptionReason (iOS 14.5+)

- `.default` — Standard (phone call, alarm, Siri). Auto-resume safe.
- `.appWasSuspended` — System suspended app. Do NOT auto-resume.
- `.builtInMicInUse` — Mic claimed. Relevant for `.playAndRecord` only.
- `.routeChange` — Route change caused interruption.

### 3.4 Race Condition: Engine Configuration Change

`AVAudioEngineConfigurationChange` can fire after `.ended`. If the interrupting app used a different sample rate, the engine reconfigures and a restart immediately after `.ended` fails.

**Solution:** Track both notifications with shared state. Do not restart until interruption has ended AND no configuration change is pending.

### 3.5 Media Server Reset

On `mediaServicesWereResetNotification`:

1. Dispose ALL audio objects. Ignore disposal errors.
2. Reset all internal playback state.
3. Reconfigure `AVAudioSession` from scratch.
4. Recreate all audio objects.
5. Do NOT reactivate until user action.

---

## 4. Multi-Player Coordination

See [references/multi-player-coordination.md](references/multi-player-coordination.md) for architecture patterns.

### 4.1 Synchronized Start (BLOCKING)

```swift
let layers: [AVAudioPlayer] = [voiceNote, instrumental, tapeHiss]
layers.forEach { $0.prepareToPlay() }

let startTime = layers[0].deviceCurrentTime + 0.25
layers.forEach { $0.play(atTime: startTime) }
```

- `prepareToPlay()` is mandatory. Preloads buffers, acquires hardware.
- `deviceCurrentTime` is the hardware clock shared by all players. This is the synchronization primitive.
- The delay provides buffer for hardware readiness.

### 4.2 Layer Volume Architecture

All volume values are named constants:

```
Voice Note      — primary content    — 1.0
Instrumental    — underlay           — 0.35
Tape Hiss       — ambient texture    — 0.15
```

- Retain all `AVAudioPlayer` instances as strong properties. Deallocation stops playback silently.
- Set `delegate = nil` before releasing any player.
- For infinite loops: `numberOfLoops = -1`. Use WAV/CAF for seamless loops — MP3/AAC encoder padding creates gaps.

### 4.3 AVAudioEngine Threshold

Use `AVAudioEngine` when any of: per-track effects needed, submixer grouping required, more than 3 concurrent sources, or runtime gain control via `AVAudioUnitEQ`.

### 4.4 Prohibited Coordination Patterns

- `DispatchQueue.main.asyncAfter` for playback synchronization.
- Creating new `AVAudioPlayer` instances per playback cycle.
- Session-level `.duckOthers` for intra-app volume management.

---

## 5. Volume and Crossfading

See [references/crossfade-patterns.md](references/crossfade-patterns.md) for implementation detail.

### 5.1 Built-in Fade

```swift
DispatchQueue.main.async {
    player.setVolume(targetVolume, fadeDuration: duration)
}
```

Must be called on main thread. Some iOS versions execute as instant jump otherwise.

### 5.2 Equal-Power Crossfade (BLOCKING for Track Transitions)

Linear crossfades produce a -6dB perceived dip at midpoint. Use equal-power:

```
outGain = cos(progress * pi / 2)
inGain  = sin(progress * pi / 2)
```

Since `setVolume(_:fadeDuration:)` is linear internally, implement via timer-driven approach at ~60 steps per transition for true equal-power curves.

### 5.3 Volume Rules

- `volume` is linear 0.0–1.0. UI sliders must apply `pow(position, 2.0)` or logarithmic mapping.
- Never set `.volume` directly while `setVolume(_:fadeDuration:)` is in progress.
- For `AVAudioEngine`: use `AVAudioUnitEQ.globalGain` instead of `playerNode.volume` (known iOS bug where node volume has no effect).

### 5.4 Prohibited Volume Patterns

- Hardcoded float literals for volume. Use named constants.
- Abrupt `.volume` assignment during active playback without fade.
- Relying on `AVAudioPlayerNode.volume` without the `AVAudioUnitEQ` workaround.

---

## 6. Route Change Handling

### 6.1 Mandatory Response Matrix

| Reason                      | Action                                                          |
| --------------------------- | --------------------------------------------------------------- |
| `.oldDeviceUnavailable`     | Pause all playback. Check previous route for headphone/BT loss. |
| `.newDeviceAvailable`       | Do NOT auto-resume. Update UI only.                             |
| `.override`                 | No action. Programmatic change.                                 |
| `.categoryChange`           | Verify session configuration.                                   |
| `.routeConfigurationChange` | Check sample rate and channel count. Reconfigure if changed.    |

### 6.2 Headphone Disconnect Detection

Inspect `AVAudioSessionRouteChangePreviousRouteKey` for port types `.headphones`, `.bluetoothA2DP`, `.bluetoothLE`. Pause on disconnect. Never auto-resume on connect.

---

## 7. Playback Resilience

### 7.1 Resume After Interruption (BLOCKING)

1. Reconfigure session category (defensive).
2. Call `setActive(true)`.
3. Call `prepareToPlay()` on each player. If it returns `false`, rebuild that player.
4. Call `play()` on all layers.

### 7.2 Silent Failure Prevention

| Cause                                    | Fix                              |
| ---------------------------------------- | -------------------------------- |
| Player deallocation                      | Retain as strong property        |
| Missing `prepareToPlay()`                | Always call before `play()`      |
| Category not `.playback`                 | Set before activation            |
| `setActive` not called post-interruption | Reactivate in resume path        |
| `.mixWithOthers` on primary content      | Omit for Now Playing integration |
| Pause near track end                     | Detect and seek before replaying |

### 7.3 Now Playing Integration

Register `MPRemoteCommandCenter` play/pause targets. Update `MPNowPlayingInfoCenter` with track metadata and elapsed time. Without this, no lock screen controls appear and the system may suspend background audio.

---

## 8. Code Review Checklist

Reject audio code if ANY are true:

- [ ] Category is not `.playback` for audio-focused app
- [ ] `setActive(true)` called at launch instead of first playback
- [ ] Missing any of the five mandatory notification observers
- [ ] Interruption handler lacks `shouldResume` check
- [ ] `appWasSuspended` triggers auto-resume
- [ ] Multi-player start uses `asyncAfter` instead of `deviceCurrentTime`
- [ ] `prepareToPlay()` omitted before synchronized start
- [ ] Volume values are hardcoded float literals
- [ ] Track transitions use linear crossfade
- [ ] Route change handler missing `.oldDeviceUnavailable` response
- [ ] `setVolume(_:fadeDuration:)` called off main thread
- [ ] `AVAudioPlayer` instances stored as local variables
- [ ] Missing `UIBackgroundModes: audio` in Info.plist
- [ ] No `MPRemoteCommandCenter` for background control
- [ ] `mediaServicesWereResetNotification` handler not implemented

---

## References

- `references/interruption-lifecycle.md` — Full interruption handler, state machine, edge cases.
- `references/multi-player-coordination.md` — Synchronized start, AVAudioEngine graph, layer management.
- `references/crossfade-patterns.md` — Equal-power crossfade, volume curves, fade coordination.
