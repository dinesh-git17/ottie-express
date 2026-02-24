# Ottie Express — Client Proposal

**Prepared for:** Dinesh
**Date:** February 24, 2026
**Version:** 1.0
**Confidentiality:** This document is confidential and intended solely for the named recipient.

---

## Project Overview

Ottie Express is a premium iOS application designed as a deeply personal Valentine's Day gift. The app delivers a linear narrative experience across six interactive chapters, each offering a distinct interaction style: an infinite runner game, collaborative poetry, voice-activated audio playback, an animated comedic sequence, a top-down maze, and an interactive constellation map. The experience culminates in an ending screen that brings the journey to a quiet, emotional close.

The application is built exclusively for iPhone, distributed privately via TestFlight to a single recipient. There is no App Store release, no backend infrastructure, and no user account system. The entire experience runs on-device with one external service call for a personalized poetry response.

The design intent is uncompromising: every screen should feel like a finished product from a top-tier studio. No placeholder interfaces, no generic components, no shortcuts. Tactile feedback accompanies every meaningful interaction. Animations are spring-based and fluid. Audio is layered and responsive. The result should feel like something worth keeping on her phone permanently.

---

## Value of the Solution

What makes this project distinctive is not any single feature. It is the cumulative effect of six carefully crafted experiences, each with its own personality, each building emotional momentum toward a final reveal.

**For the recipient,** the app transforms a digital screen into something that feels handmade. The poems respond to her words. The cassette plays a real voice. The maze tells the story of your relationship through its structure. The constellations she draws form a shape that means something only to her. Every detail signals intentionality.

**For you as the client,** this is a finished product you hand someone once. It needs to work flawlessly on first launch. There is no second chance to make a first impression, no patch cycle, no user feedback loop. The quality bar is set by the nature of the gift itself: it must feel perfect the first time she opens it.

This proposal delivers that level of finish.

---

## Development Approach

The application is built entirely with native Apple frameworks. No third-party libraries or external dependencies. This ensures long-term stability, optimal performance on the target device, and zero supply chain risk.

**Platform:** iOS 18.0+, SwiftUI with SpriteKit for game chapters
**Target Device:** iPhone 17 Pro
**Orientation:** Portrait only
**Distribution:** TestFlight (private)

The development process follows a chapter-sequential build order. Each chapter is developed, tested, and polished as a self-contained unit before the next begins. Shared infrastructure (state persistence, audio management, haptic feedback, animation components) is established first, then each chapter builds on that foundation.

All character artwork, background illustrations, audio tracks, and the voice recording are provided by the client. Engineering effort covers implementation, interaction design, animation tuning, integration, and testing.

---

## Scope Breakdown

### Foundation

Establish the application architecture, state management system, and shared services that every chapter depends on.

- Application shell and navigation controller
- Chapter progression engine with persistent state (survives app termination and device restart)
- Audio service: background music playback, sound effects, crossfading between tracks, system interruption handling and recovery
- Haptic service: standard feedback patterns and advanced progressive haptic sequences
- Typewriter text component: character-by-character reveal with variable timing for punctuation, line breaks, and paragraph pauses
- Design system: color tokens, typography scale, spacing constants, motion curves

### Chapter 0: The Handshake

The entry gate. A biometric-themed fingerprint scan triggers a vault-opening animation, revealing a password screen.

- Fingerprint hold interaction with progressive fill animation and haptic escalation
- Vault split animation (top and bottom halves separate with spring physics)
- Password entry with shake-on-error feedback
- Sound design integration (vault open, scan tone)

### Chapter 1: The Runner

A side-scrolling infinite runner game set in a magical forest. The player collects 25 hearts while dodging obstacles, then receives a personal letter.

- SpriteKit game engine with three-layer parallax scrolling
- Four-frame character run cycle, jump mechanics, collision detection
- Heart collection system with HUD counter
- Progressive difficulty scaling across five tiers
- Obstacle spawning system (airborne and ground-level)
- Game over and retry flow
- Victory screen with celebration animation
- Full-screen letter with typewriter reveal

### Chapter 2: The Poems

Three fill-in-the-blank poems with a swipe-to-select word mechanic, followed by a collaborative poetry exchange using a live language service.

- Swipe-based word selector with curated word lists per blank
- Correct word detection with visual confirmation and haptic feedback
- Poem collection reveal with animated card transitions
- Free-form text input for the recipient's own poem
- Integration with external language service for personalized reply
- Two-stage response: a heartfelt reply followed by a poetic continuation
- Typewriter display for both responses with audio accompaniment
- Graceful fallback if the service is unavailable

### Chapter 3: The Cassette

A voice-activated chapter. The recipient speaks a phrase to unlock a cassette tape that plays a personal voice note.

- Microphone and speech recognition permission handling with Settings redirect fallback
- On-device speech recognition listening for a trigger phrase
- Live audio waveform visualizer responding to microphone input
- Cassette insertion animation with mechanical sound design
- Voice note playback with animated VU meter needles synced to audio levels
- Dual-reel cassette animation (left depletes, right fills)
- Timed floating key phrases synced to audio playback position
- Instrumental underlay that fades in mid-playback at reduced volume
- Final line reveal after playback completes

### Chapter 4: The Dossier

A comedic interlude. An otter in a business suit presents ten classified "evidence files" about the recipient in deadpan surveillance language.

- Scene composition with layered character and environment assets
- Briefcase-to-dossier transformation animation with sound effects
- Speech bubble system displaying ten sequential evidence entries
- Character expression changes timed to narrative beats
- Tap-to-advance navigation through all ten files
- Closing scene with expression shift and final romantic line

### Chapter 5: The Maze

A top-down maze across three visually distinct sections, progressing from cold gray to warm gold. The journey is a metaphor for long-distance relationship.

- SpriteKit maze engine with swipe-based directional controls
- Three themed maze sections with distinct tilesets and background music
- Labeled walls (humorous obstacles) and labeled paths (relationship milestones)
- Section transitions with camera fade, tileset swap, and music crossfade
- Four-directional character walking animation
- Wall collision feedback (sound and haptic, no punishment)
- Companion character visible in the final section
- Reunion cutscene: auto-walk, hug animation, heart particle burst, camera pull-back revealing the full maze

### Chapter 6: The Constellations

The finale. A dark sky filled with stars. The recipient draws six constellations by connecting star dots. The sixth forms the shape of an otter. The sky illuminates.

- Star field with staggered fade-in entry animation
- Drag-to-connect constellation drawing mechanic
- Six constellations with distinct star layouts
- Memory card overlay after each constellation (personal narrative text)
- Otter silhouette reveal on the sixth constellation
- Full-sky illumination event: all constellations flash, orchestral audio swell, strong haptic, camera zoom-out
- Final message with slow word-by-word typewriter reveal

### Ending Screen

A quiet landing. Background music. Faint constellations. A small waving otter. No buttons. The experience is complete.

### Quality Assurance

- Full end-to-end playthrough on physical iPhone 17 Pro
- Chapter state persistence verification (force-quit, relaunch, device restart)
- Audio interruption testing (incoming call during voice note playback)
- Speech recognition testing in quiet environment
- External service timeout and fallback verification
- Animation performance validation (sustained 60fps)
- Bundle size verification against target constraints

---

## Timeline Overview

| Phase                    | Duration    | Deliverables                                                                      |
| ------------------------ | ----------- | --------------------------------------------------------------------------------- |
| **Foundation**           | Week 1      | Architecture, state system, shared services, design tokens                        |
| **Chapters 0 and 1**     | Weeks 2 - 3 | Fingerprint gate, vault animation, password entry, full runner game, letter       |
| **Chapters 2 and 3**     | Weeks 3 - 5 | Poetry interaction, language service integration, voice unlock, cassette playback |
| **Chapters 4 and 5**     | Weeks 5 - 6 | Dossier scene, maze engine with three sections, reunion cutscene                  |
| **Chapter 6 and Ending** | Weeks 6 - 7 | Constellation system, sky illuminate, ending screen                               |
| **Testing and Polish**   | Week 8      | Full regression, animation tuning, performance validation, TestFlight delivery    |

**Total estimated duration: 8 weeks**

Timeline assumes client-provided assets (character artwork, backgrounds, audio tracks, voice recording) are delivered by the start of Week 2. Delays in asset delivery may extend the schedule proportionally.

---

## Cost Breakdown

| Deliverable                                                        |   Cost |
| ------------------------------------------------------------------ | -----: |
| Foundation (architecture, state, services, design system)          | $2,500 |
| Shared components (typewriter text, haptic patterns, audio engine) | $3,000 |
| Chapter 0: The Handshake                                           | $3,500 |
| Chapter 1: The Runner (SpriteKit game + letter)                    | $7,500 |
| Chapter 2: The Poems (interaction + service integration)           | $5,500 |
| Chapter 3: The Cassette (voice recognition + audio playback)       | $7,000 |
| Chapter 4: The Dossier (scene composition + animation)             | $3,500 |
| Chapter 5: The Maze (SpriteKit maze + 3 sections + cutscene)       | $6,500 |
| Chapter 6: The Constellations (drawing system + sky event)         | $5,000 |
| Ending Screen                                                      |   $500 |
| Quality assurance and device testing                               | $3,000 |
| Final polish and TestFlight delivery                               | $2,000 |

---

## Total

|                   |                 |
| ----------------- | --------------: |
| **Project Total** | **$49,500 USD** |

Payment terms, asset delivery schedule, and revision policy to be agreed upon prior to project start.

---

_This proposal is valid for 30 days from the date of issue._
