# Ottie Express

**Status:** Under active development
**Platform:** iOS 18.0+ / SwiftUI / iPhone 17 Pro
**Distribution:** TestFlight (private)

## Overview

Ottie Express is a private, single-user iOS application structured as a linear narrative experience. The app delivers six sequential chapters, each containing a self-contained interactive experience: mini-games, puzzles, animated scenes, and voice notes. Chapters unlock progressively and build toward a final screen.

The application uses native Apple frameworks exclusively with zero third-party dependencies. SpriteKit handles game scenes via `SpriteView`, AVFoundation manages layered audio playback, CoreHaptics drives tactile feedback, and the Speech framework provides on-device voice recognition. State management uses the Observation framework (`@Observable`) with `UserDefaults` persistence.

## Documentation

Full project documentation, implementation roadmap, and epic specifications are available at the documentation portal:

<https://dinesh-git17.github.io/ottie-express/#home>

The product design document is located at [`docs/OTTIE_EXPRESS_DESIGN_DOC.md`](docs/OTTIE_EXPRESS_DESIGN_DOC.md).

## Repository Structure

```
docs/           Design document and implementation roadmap
.claude/        Engineering governance, skills, and automation
.github/        CI workflows, PR template, and CODEOWNERS
```

## Development

- **Language:** Swift 6.0 with strict concurrency
- **Target:** iPhone 17 Pro, iOS 18.0+, portrait only
- **Build:** Xcode 16+
- **Dependencies:** None (native Apple frameworks only)
