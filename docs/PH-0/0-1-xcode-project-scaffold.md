# E0.1: Scaffold Xcode Project and Directory Structure

**Phase:** 0 - Project Scaffold
**Class:** Infrastructure
**Design Doc Reference:** SS1 Product Overview, SS4 Architecture Overview, SS5 Asset Inventory
**Dependencies:**

- None (Phase 0 is the root of the dependency graph)

---

## Goal

Build the Xcode project scaffold with iOS 18.0 deployment target, Swift 6 strict concurrency, the SS4 directory hierarchy, initialized asset catalogs, and required privacy usage descriptions.

---

## Scope

### File Inventory

| File                                         | Action | Responsibility                                                                                                                                                         |
| -------------------------------------------- | ------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `OttieExpress.xcodeproj/project.pbxproj`     | Create | Xcode project configuration: iOS 18.0 deployment target, Swift 6, strict concurrency, privacy usage descriptions, portrait-only orientation, iPhone-only device family |
| `OttieExpress/App/OttieExpressApp.swift`     | Create | App entry point declaring `@main` struct conforming to SwiftUI `App` protocol with an empty root view                                                                  |
| `OttieExpress/Assets.xcassets/Contents.json` | Create | Root asset catalog manifest with development region metadata                                                                                                           |

### Integration Points

**OttieExpress.xcodeproj/project.pbxproj**

- Imports from: N/A (project definition file, not a Swift source file)
- Imported by: N/A (consumed by the Xcode build system)
- State reads: None
- State writes: None

**OttieExpressApp.swift**

- Imports from: None (no project source files exist at this phase)
- Imported by: None (app entry point, consumed by the build system via `@main`)
- State reads: None
- State writes: None

**Assets.xcassets/Contents.json**

- Imports from: N/A (asset catalog manifest, not a Swift source file)
- Imported by: N/A (resolved by the Xcode asset catalog compiler at build time)
- State reads: None
- State writes: None

---

## Out of Scope

- Design system token definitions (colors, typography, spacing, motion constants) (covered by E1.1 in Phase 1)
- `AppState` observable class and `UserDefaults` persistence layer (covered by E1.2 in Phase 1)
- Service interface implementations: `AudioService`, `HapticService`, `AnthropicService` (covered by E1.3, E1.4, E1.5 in Phase 1)
- Navigation router, chapter routing logic, and shared UI components (covered by Phase 2)
- Importing sprite, background, or audio assets into the asset catalog or filesystem directories (assets are added per-chapter in Phases 3 through 9)
- Build automation, CI/CD pipeline, Fastlane, or GitHub Actions configuration (separate chore scope if needed)
- Animation tuning, spring parameter calibration, or motion system implementation (covered by Phase 1 design tokens and Phase 2+ view epics)

---

## Definition of Done

- [ ] `xcodebuild build -scheme OttieExpress -destination 'platform=iOS Simulator,name=iPhone 17 Pro'` completes with zero errors and zero warnings
- [ ] App launches on iPhone 17 Pro simulator and renders an empty root view
- [ ] `IPHONEOS_DEPLOYMENT_TARGET` equals `18.0` in the project build settings
- [ ] `SWIFT_VERSION` equals `6` in the project build settings
- [ ] `SWIFT_STRICT_CONCURRENCY` equals `complete` in the project build settings
- [ ] `TARGETED_DEVICE_FAMILY` equals `1` (iPhone only) in the project build settings
- [ ] Supported interface orientations include only portrait in the project build settings
- [ ] Directory hierarchy under `OttieExpress/` contains all 15 directories from SS4: `App/`, `Chapters/Chapter0/`, `Chapters/Chapter1/`, `Chapters/Chapter2/`, `Chapters/Chapter3/`, `Chapters/Chapter4/`, `Chapters/Chapter5/`, `Chapters/Chapter6/`, `EndingScreen/`, `SharedUI/`, `Assets/Sprites/`, `Assets/Backgrounds/`, `Assets/Audio/`, `Services/`, `Extensions/`
- [ ] `Assets.xcassets` resolves at build time without catalog compilation errors
- [ ] Built app bundle `Info.plist` contains `NSMicrophoneUsageDescription` with a non-empty usage string
- [ ] Built app bundle `Info.plist` contains `NSSpeechRecognitionUsageDescription` with a non-empty usage string

---

## Implementation Notes

**Project creation:** Use Xcode 16+ "App" template. Select SwiftUI for the interface and Swift for the language. Set product name to `OttieExpress`, organization identifier to the owner's reverse-domain. The template generates a default `ContentView.swift` and `OttieExpressApp.swift`. Reorganize the output to place the app entry point under `App/` per SS4 Architecture Overview. Delete any template-generated files not required by the SS4 structure (e.g., default `ContentView.swift` if replaced by the entry point's inline body). Reference: SS4 Architecture Overview, project structure diagram.

**Build settings:**

| Setting                                          | Value                            | Source                                             |
| ------------------------------------------------ | -------------------------------- | -------------------------------------------------- |
| `IPHONEOS_DEPLOYMENT_TARGET`                     | `18.0`                           | SS4 Technology Stack: "iOS 18.0+"                  |
| `SWIFT_VERSION`                                  | `6`                              | SS4 Technology Stack: "Swift 6 language mode"      |
| `SWIFT_STRICT_CONCURRENCY`                       | `complete`                       | SS4 Technology Stack: "strict concurrency enabled" |
| `TARGETED_DEVICE_FAMILY`                         | `1` (iPhone only)                | SS1 Product Overview: iPhone 17 Pro target device  |
| `INFOPLIST_KEY_UISupportedInterfaceOrientations` | `UIInterfaceOrientationPortrait` | Design doc header: "Orientation: Portrait only"    |

**Directory hierarchy:** Create the full directory tree from SS4 Architecture Overview as filesystem directories under the `OttieExpress/` source root. In Xcode 16+ with file system-backed project structure, these map directly to on-disk directories. Empty directories require a `.gitkeep` file to persist in Git.

```
OttieExpress/
  App/
  Chapters/
    Chapter0/
    Chapter1/
    Chapter2/
    Chapter3/
    Chapter4/
    Chapter5/
    Chapter6/
  EndingScreen/
  SharedUI/
  Assets/
    Sprites/
    Backgrounds/
    Audio/
  Services/
  Extensions/
```

Reference: SS4 Architecture Overview, project structure specification.

**Privacy usage descriptions:** Add both keys via the target's Info tab (Custom iOS Target Properties) in Xcode, which writes entries to the project build settings in `project.pbxproj`. These keys are required for later chapters but must be present at the project level from the start.

| Key                                   | Required By                                              | Design Doc Reference                    |
| ------------------------------------- | -------------------------------------------------------- | --------------------------------------- |
| `NSMicrophoneUsageDescription`        | Chapter 3: `AVAudioEngine` input tap for voice recording | SS4 Dependencies (AVFoundation, Speech) |
| `NSSpeechRecognitionUsageDescription` | Chapter 3: `SFSpeechRecognizer` on-device recognition    | SS4 Dependencies (Speech framework)     |

The usage description strings appear in iOS system permission dialogs. Write them in the app's voice, addressed to Carolina as the sole user. Reference: SS1 Product Overview (single user, private gift app).

**Third-party dependencies:** None permitted. The project must not include any Swift Package Manager, CocoaPods, or Carthage dependencies. Reference: SS4 Dependencies: "No third-party package manager dependencies. Native Apple frameworks only. Zero external packages." SS4 Technology Stack: "Third-Party Packages: None."

**App entry point pattern:**

```swift
import SwiftUI

@main
struct OttieExpressApp: App {
    var body: some Scene {
        WindowGroup {
            Color.clear
        }
    }
}
```

The root view is intentionally empty. `AppState` injection, navigation routing, and themed content are added in Phases 1 and 2. Reference: SS4 State Management (injection pattern), SS4 Architecture Overview.
