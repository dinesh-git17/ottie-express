# E6.3: Build API Response Display Screens

**Phase:** 6 - Chapter 2: The Poems
**Class:** Feature
**Design Doc Reference:** §8 (Haiku Stage 1 Display, Haiku Stage 2 Display, API: Haiku Stage 1, API: Haiku Stage 2), §16 (API Integration, Error handling, Loading state)
**Dependencies:**

- Phase 2: Navigation Shell and Shared UI (exit criteria met)
- E1.1: Define design system tokens (color tokens `parchment`, `muted-gray`, `rose-blush` and typography roles available)
- E1.3: Build AudioService (`sfx-typewriter-tick` per-character playback available)
- E1.5: Build AnthropicService (`generateReply(to:)` and `generateContinuation(of:)` interfaces available)
- E2.2: Build TypewriterText component (progressive text reveal with per-character SFX callback available)
- E6.1: Build fill-in poem screens (Chapter2Constants available)
- Asset: `sfx-typewriter-tick` (available in app bundle)

---

## Goal

Implement the Stage 1 (haiku reply) and Stage 2 (poem continuation) display screens with API call execution, typewriter text animation, blinking cursor loading state, and fallback message on API failure within the 10-second timeout.

---

## Scope

### File Inventory

| File                                                        | Action | Responsibility                                                                                                                                                                               |
| ----------------------------------------------------------- | ------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `OttieExpress/Chapters/Chapter2/HaikuReplyView.swift`       | Create | Stage 1 display: her poem dimmed at top, API reply typewriters below, blinking cursor loading state, fallback message on failure, 2-second pause before transition                           |
| `OttieExpress/Chapters/Chapter2/PoemContinuationView.swift` | Create | Stage 2 display: her poem dimmed at top, API continuation typewriters below with spacing, blinking cursor loading state, fallback message on failure, "Continue" prompt after 3-second delay |

### Integration Points

**HaikuReplyView.swift**

- Imports from: `Chapter2Constants.swift` (E6.1), `TypewriterText` (E2.2)
- Imported by: `Chapter2View.swift` (E6.4)
- State reads: None (receives her poem text and API response text as input parameters)
- State writes: None (reports stage completion via callback closure)
- Service calls: AnthropicService.generateReply(to:) invoked within an async task on view appearance

**PoemContinuationView.swift**

- Imports from: `Chapter2Constants.swift` (E6.1), `TypewriterText` (E2.2)
- Imported by: `Chapter2View.swift` (E6.4)
- State reads: None (receives her poem text and API response text as input parameters)
- State writes: None (reports stage completion via callback closure)
- Service calls: AnthropicService.generateContinuation(of:) invoked within an async task on view appearance

---

## Out of Scope

- Fill-in poem screens, swipe mechanic, and word list data (covered by E6.1)
- Poem collection reveal and her-turn text input (covered by E6.2)
- AnthropicService implementation, DTO construction, and network layer (covered by E1.5; this epic consumes the interface)
- TypewriterText component implementation (covered by E2.2; this epic consumes the component)
- Flow coordination, screen sequencing, and chapter completion (covered by E6.4)
- System prompt authoring or API model selection (defined in §8 and §16; this epic passes parameters, does not define them)
- Retry logic or user-initiated retry on API failure (§16 specifies a single fallback message, not retry)
- Network reachability checks before API call (§16 relies on the 10-second timeout as the sole failure detection mechanism)

---

## Definition of Done

- [ ] HaikuReplyView displays her poem text in the top half in New York Regular 17pt with `muted-gray` color at reduced opacity
- [ ] HaikuReplyView shows a blinking cursor on an otherwise empty lower half while awaiting the API response
- [ ] HaikuReplyView calls AnthropicService.generateReply(to:) with her poem text on appearance
- [ ] API reply text typewriters in below her poem in New York Regular 18pt with `parchment` color using the TypewriterText component
- [ ] `sfx-typewriter-tick` plays per character during the reply typewriter animation via the TypewriterText audio callback
- [ ] HaikuReplyView transitions to Stage 2 after a 2-second pause following reply typewriter completion
- [ ] PoemContinuationView displays her poem text at the top, dimmed, in New York Regular 17pt with `muted-gray` color
- [ ] PoemContinuationView calls AnthropicService.generateContinuation(of:) with her poem text on appearance
- [ ] Continuation text typewriters in below her poem with vertical spacing (no line divider) in New York Regular 18pt with `parchment` color
- [ ] After Stage 2 typewriter completion, both halves remain visible and a "Continue" prompt fades in after a 3-second delay
- [ ] API failure on either stage displays the fallback message "bebe, your words were so beautiful I'm still processing them. 🥰" via typewriter animation
- [ ] Blinking cursor loading state persists for up to 10 seconds before the fallback triggers on API timeout
- [ ] Both views cancel in-flight API tasks and typewriter animations on view disappearance

---

## Implementation Notes

Per §8 Haiku Stage 1 Display, the loading state is a blinking cursor with no spinner and no loading text. The cursor blinks on an empty area below her poem. The 10-second timeout is configured on the URLRequest in AnthropicService (E1.5); this epic's responsibility is to detect the thrown error and trigger the fallback path.

Per §8 Haiku Stage 2 Display, the visual distinction between her poem and the continuation is spacing only ("a subtle dividing breath, no line, just spacing"). This is vertical padding between the two text blocks, not a visual separator element.

Per §16 Error handling, the fallback message is identical for both stages: "bebe, your words were so beautiful I'm still processing them. 🥰". This message typewriters in using the same TypewriterText component and `sfx-typewriter-tick` audio callback.

The API call is initiated within a `.task {}` modifier scoped to the view lifecycle. This provides automatic cancellation on view disappear per Swift structured concurrency. The view manages three states: loading (blinking cursor), success (typewriter), and failure (fallback typewriter). Use an enum to model these states explicitly.

Per §8, the Stage 1 system prompt and Stage 2 system prompt are defined verbatim in the design doc. These prompts are parameters passed to AnthropicService, not constructed by the display view. The view calls the service method, which encapsulates the prompt. The model is `claude-haiku-4-5-20251001` per §8 and §16.

Both views receive the API response as an async result and feed the text into the TypewriterText component. The TypewriterText component from E2.2 accepts an optional per-character audio callback, which this epic wires to AudioService.playSFX for `sfx-typewriter-tick`.
