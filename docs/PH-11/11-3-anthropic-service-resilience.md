# E11.3: Harden AnthropicService resilience

**Phase:** 11 - Cross-Cutting Hardening
**Class:** Integration
**Design Doc Reference:** SS16 API Integration (Anthropic API, Error Handling, Loading State), SS8 Chapter 2 (API: Haiku Stage 1, API: Haiku Stage 2)
**Dependencies:**

- Phase 1: Core Systems and Design Foundation (exit criteria met)
- E1.5: Build AnthropicService (AnthropicService created with typed errors, 10-second timeout, and Sendable conformance)
- Phase 6: Chapter 2 (exit criteria met, HaikuReplyView and PoemContinuationView consuming AnthropicService)

---

## Goal

Validate AnthropicService timeout, error handling, and Sendable safety under all network failure conditions to ensure Chapter 2 degrades gracefully with the correct fallback message.

---

## Scope

### File Inventory

| File                                           | Action | Responsibility                                                                                                                              |
| ---------------------------------------------- | ------ | ------------------------------------------------------------------------------------------------------------------------------------------- |
| `OttieExpress/Services/AnthropicService.swift` | Modify | Add named fallback string constant, verify 10-second timeout on both endpoints, validate all error paths produce catchable typed errors |

### Integration Points

**AnthropicService.swift**

- Imports from: `AnthropicModels.swift` (request/response DTO types)
- Imported by: Chapter 2 (`HaikuReplyView`, `PoemContinuationView` via Chapter2View flow coordinator)
- State reads: None
- State writes: None

---

## Out of Scope

- Chapter 2 view-layer fallback rendering logic (HaikuReplyView and PoemContinuationView catch `AnthropicService.Error`, extract the fallback string, and render it via TypewriterText; view code is not modified by this epic)
- Loading state cursor blink animation during the 10-second API wait window (view-layer animation owned by Chapter 2 feature epic E6.3)
- Retry logic, exponential backoff, or request queuing (SS16 specifies single-attempt with immediate fallback; no retry mechanism)
- API key rotation, refresh, or remote configuration (single key loaded from environment per CLAUDE.md SS10.4; key lifecycle is not a runtime concern)
- Rate limiting or request throttling (single-user private app with spend cap mitigation per SS16 Accepted Security Risks)
- API response caching or offline poem storage (not specified in design doc)

---

## Definition of Done

- [ ] `URLRequest` `timeoutInterval` is set to exactly 10 seconds on both `generateReply(to:)` and `generateContinuation(of:)` requests
- [ ] A network timeout (no server response within 10 seconds) causes both methods to throw `AnthropicService.Error` and the Chapter 2 view displays the fallback message in typewriter format
- [ ] An HTTP 4xx status code response causes both methods to throw `AnthropicService.Error.httpError` and the Chapter 2 view displays the same fallback message
- [ ] An HTTP 5xx status code response causes both methods to throw `AnthropicService.Error.httpError` and the Chapter 2 view displays the same fallback message
- [ ] A malformed JSON response body that fails `JSONDecoder` parsing causes both methods to throw `AnthropicService.Error.decodingFailed` and the Chapter 2 view displays the same fallback message
- [ ] The fallback string "bebe, your words were so beautiful I'm still processing them. 🥰" is defined as a named static constant `AnthropicService.fallbackMessage` accessible to consuming views
- [ ] `AnthropicService` compiles with zero concurrency warnings under Swift 6 strict concurrency

---

## Implementation Notes

E1.5 built AnthropicService with a typed `Error` enum (`missingAPIKey`, `httpError(statusCode:)`, `decodingFailed`, `emptyResponse`) and both endpoint methods configured with 10-second `timeoutInterval`. This epic validates those error paths produce the correct end-to-end fallback behavior and centralizes the fallback string. Reference: E1.5 Implementation Notes, SS16 API Integration.

**Fallback string centralization:** SS16 defines the exact fallback text:

```
bebe, your words were so beautiful I'm still processing them. 🥰
```

E1.5 placed fallback display responsibility in the Chapter 2 view layer. This epic adds a `static let fallbackMessage` constant to `AnthropicService` so the string has a single source of truth. Chapter 2 views reference `AnthropicService.fallbackMessage` in their error catch blocks instead of defining inline string literals. This satisfies CLAUDE.md SS3 (no magic numbers/strings). Reference: SS16 Error Handling, CLAUDE.md SS3.3 Forbidden Code Artifacts.

**Error path validation:** Each error type must be testable in isolation. Recommended validation approach:

- **Network timeout:** Disable network connectivity on the test device, invoke `generateReply(to:)`, confirm `Error` is thrown within 10 seconds (not before, not significantly after).
- **HTTP 4xx:** Send a request with an invalid API key; the Anthropic endpoint returns 401. Confirm `httpError(statusCode: 401)` is thrown.
- **HTTP 5xx:** Simulate by temporarily pointing `endpoint` to a URL that returns 500, or use a local proxy. Confirm `httpError(statusCode: 500)` is thrown.
- **Decode failure:** Simulate by temporarily corrupting the response parsing (e.g., expect a different key). Confirm `decodingFailed` is thrown.

Reference: SS16 API Integration, E1.5 DoD.

**Sendable verification:** `AnthropicService` is `Sendable` by construction (all stored properties are immutable `let` constants). Compile with Swift 6 strict concurrency mode enabled in build settings. Zero concurrency warnings on `AnthropicService.swift` and `AnthropicModels.swift`. `async throws` methods execute on the cooperative thread pool via `URLSession.shared.data(for:)`. Reference: SS4 Concurrency Model, E1.5 Implementation Notes.

**API key governance:** CLAUDE.md SS10.4 supersedes SS16 on key storage. The key loads from `ProcessInfo.processInfo.environment["ANTHROPIC_API_KEY"]`. If absent, the service stores an empty string and both methods throw `Error.missingAPIKey` at call time. Verify this path also results in the fallback message displaying in Chapter 2. Reference: CLAUDE.md SS10.4, PHASES.md Execution Notes (API key governance).
