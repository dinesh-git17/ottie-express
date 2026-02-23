# Pattern Architecture Reference

## CHHapticPattern Construction

Three initializer forms:

```swift
// Events + dynamic parameters
let pattern = try CHHapticPattern(events: [event1, event2], parameters: [dynamicParam])

// Events + parameter curves
let pattern = try CHHapticPattern(events: events, parameterCurves: [curve1, curve2])

// From AHAP data
let pattern = try CHHapticPattern(data: ahapData)
```

`pattern.duration` returns total pattern duration (read-only).
`try pattern.exportDictionary()` serializes to AHAP-format dictionary.

## CHHapticParameterCurve

Smooth interpolation between control points. Acts as a **multiplier** on base event values.

```swift
let intensityRamp = CHHapticParameterCurve(
    parameterID: .hapticIntensityControl,
    controlPoints: [
        CHHapticParameterCurve.ControlPoint(relativeTime: 0, value: 0.1),
        CHHapticParameterCurve.ControlPoint(relativeTime: 0.5, value: 1.0),
        CHHapticParameterCurve.ControlPoint(relativeTime: 1.5, value: 1.0),
        CHHapticParameterCurve.ControlPoint(relativeTime: 2.0, value: 0.0)
    ],
    relativeTime: 0
)
```

Available control IDs: `.hapticIntensityControl`, `.hapticSharpnessControl`, `.hapticAttackTimeControl`, `.hapticDecayTimeControl`, `.hapticReleaseTimeControl`.

**16-point limit per curve.** Control points beyond 16 in a single `CHHapticParameterCurve` are silently dropped. Chain multiple curves:

```swift
let curveA = CHHapticParameterCurve(
    parameterID: .hapticIntensityControl,
    controlPoints: first16Points,
    relativeTime: 0
)
let curveB = CHHapticParameterCurve(
    parameterID: .hapticIntensityControl,
    controlPoints: next16Points,
    relativeTime: curveA_endTime
)
```

## CHHapticDynamicParameter

Abrupt step-function change at a point in time. Also a **multiplier**.

```swift
let quietDown = CHHapticDynamicParameter(
    parameterID: .hapticIntensityControl,
    value: 0.3,
    relativeTime: 0.5
)
```

Runtime injection to a running player:

```swift
let param = CHHapticDynamicParameter(
    parameterID: .hapticIntensityControl,
    value: normalizedProgress,
    relativeTime: CHHapticTimeImmediate
)
try player.sendParameters([param], atTime: CHHapticTimeImmediate)
```

## Progressive Sequence Pattern

Concrete implementation for a fingerprint/biometric scan:

```swift
private enum ScanHaptic {
    static let initiationIntensity: Float = 0.3
    static let initiationSharpness: Float = 0.5
    static let processingBaseIntensity: Float = 0.6
    static let processingBaseSharpness: Float = 0.2
    static let confirmationIntensity: Float = 1.0
    static let confirmationSharpness: Float = 0.9
    static let decayDuration: TimeInterval = 0.4
    static let processingDuration: TimeInterval = 1.5
    static let rampUpDuration: TimeInterval = 0.5
}

func buildScanPattern() throws -> CHHapticPattern {
    // Phase 1: Initiation tap
    let initiation = CHHapticEvent(
        eventType: .hapticTransient,
        parameters: [
            CHHapticEventParameter(parameterID: .hapticIntensity, value: ScanHaptic.initiationIntensity),
            CHHapticEventParameter(parameterID: .hapticSharpness, value: ScanHaptic.initiationSharpness)
        ],
        relativeTime: 0
    )

    // Phase 2: Processing ramp
    let processing = CHHapticEvent(
        eventType: .hapticContinuous,
        parameters: [
            CHHapticEventParameter(parameterID: .hapticIntensity, value: ScanHaptic.processingBaseIntensity),
            CHHapticEventParameter(parameterID: .hapticSharpness, value: ScanHaptic.processingBaseSharpness)
        ],
        relativeTime: 0.1,
        duration: ScanHaptic.processingDuration
    )

    let intensityRamp = CHHapticParameterCurve(
        parameterID: .hapticIntensityControl,
        controlPoints: [
            CHHapticParameterCurve.ControlPoint(relativeTime: 0, value: 0.2),
            CHHapticParameterCurve.ControlPoint(relativeTime: ScanHaptic.rampUpDuration, value: 0.7),
            CHHapticParameterCurve.ControlPoint(relativeTime: ScanHaptic.processingDuration, value: 1.0)
        ],
        relativeTime: 0.1
    )

    let sharpnessModulation = CHHapticParameterCurve(
        parameterID: .hapticSharpnessControl,
        controlPoints: [
            CHHapticParameterCurve.ControlPoint(relativeTime: 0, value: 0.3),
            CHHapticParameterCurve.ControlPoint(relativeTime: ScanHaptic.processingDuration * 0.5, value: 0.7),
            CHHapticParameterCurve.ControlPoint(relativeTime: ScanHaptic.processingDuration, value: 1.0)
        ],
        relativeTime: 0.1
    )

    // Phase 3: Confirmation burst
    let confirmation = CHHapticEvent(
        eventType: .hapticTransient,
        parameters: [
            CHHapticEventParameter(parameterID: .hapticIntensity, value: ScanHaptic.confirmationIntensity),
            CHHapticEventParameter(parameterID: .hapticSharpness, value: ScanHaptic.confirmationSharpness)
        ],
        relativeTime: 0.1 + ScanHaptic.processingDuration
    )

    // Phase 4: Decay
    let decay = CHHapticEvent(
        eventType: .hapticContinuous,
        parameters: [
            CHHapticEventParameter(parameterID: .hapticIntensity, value: 0.4),
            CHHapticEventParameter(parameterID: .hapticSharpness, value: 0.3)
        ],
        relativeTime: 0.1 + ScanHaptic.processingDuration + 0.05,
        duration: ScanHaptic.decayDuration
    )

    let decayFade = CHHapticParameterCurve(
        parameterID: .hapticIntensityControl,
        controlPoints: [
            CHHapticParameterCurve.ControlPoint(relativeTime: 0, value: 1.0),
            CHHapticParameterCurve.ControlPoint(relativeTime: ScanHaptic.decayDuration, value: 0.0)
        ],
        relativeTime: 0.1 + ScanHaptic.processingDuration + 0.05
    )

    return try CHHapticPattern(
        events: [initiation, processing, confirmation, decay],
        parameterCurves: [intensityRamp, sharpnessModulation, decayFade]
    )
}
```

## AHAP File Format

```json
{
  "Version": 1.0,
  "Metadata": {
    "Project": "OttieExpress",
    "Created": "2025-01-15",
    "Description": "Biometric scan progressive feedback"
  },
  "Pattern": [
    {
      "Event": {
        "Time": 0.0,
        "EventType": "HapticTransient",
        "EventParameters": [
          { "ParameterID": "HapticIntensity", "ParameterValue": 0.3 },
          { "ParameterID": "HapticSharpness", "ParameterValue": 0.5 }
        ]
      }
    },
    {
      "Event": {
        "Time": 0.1,
        "EventType": "HapticContinuous",
        "EventDuration": 1.5,
        "EventParameters": [
          { "ParameterID": "HapticIntensity", "ParameterValue": 0.6 },
          { "ParameterID": "HapticSharpness", "ParameterValue": 0.2 }
        ]
      }
    },
    {
      "ParameterCurve": {
        "ParameterID": "HapticIntensityControl",
        "Time": 0.1,
        "ParameterCurveControlPoints": [
          { "Time": 0.0, "ParameterValue": 0.2 },
          { "Time": 0.5, "ParameterValue": 0.7 },
          { "Time": 1.5, "ParameterValue": 1.0 }
        ]
      }
    }
  ]
}
```

Loading:

```swift
guard let url = Bundle.main.url(forResource: "scan", withExtension: "ahap") else { return }
try engine.playPattern(from: url)
```

## Player Management

```swift
// Retained player for repeated patterns
let player = try engine.makePlayer(with: pattern)
try player.start(atTime: CHHapticTimeImmediate)

// Advanced player for transport control
let advancedPlayer = try engine.makeAdvancedPlayer(with: pattern)
advancedPlayer.loopEnabled = true
advancedPlayer.loopEnd = 2.0
advancedPlayer.completionHandler = { error in /* ... */ }
try advancedPlayer.start(atTime: CHHapticTimeImmediate)
```

Advanced player transport: `start`, `pause`, `resume`, `seek(toOffset:)`, `stop`. `playbackRate` controls speed (1.0 = normal).
