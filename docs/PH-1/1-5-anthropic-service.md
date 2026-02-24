# E1.5: Build AnthropicService

**Phase:** 1 - Core Systems and Design Foundation
**Class:** Infrastructure
**Design Doc Reference:** SS16 API Integration (Anthropic API), SS8 Chapter 2 (API: Haiku Stage 1, API: Haiku Stage 2), SS4 Architecture Overview (Dependencies, Technology Stack, Concurrency Model)
**Dependencies:**

- Phase 0: Project Scaffold (exit criteria met)
- E0.1: Scaffold Xcode project and directory structure (`Services/` directory exists)

---

## Goal

Implement the Anthropic API client with `URLSession` async/await, `Sendable` DTOs, environment-based API key loading, and both poem exchange endpoints defined in SS8 and SS16.

---

## Scope

### File Inventory

| File                                           | Action | Responsibility                                                                                                                                                                                           |
| ---------------------------------------------- | ------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `OttieExpress/Services/AnthropicService.swift` | Create | Define `Sendable` AnthropicService struct with `generateReply(to:)` and `generateContinuation(of:)` methods, environment-based API key loading, URLSession async/await networking, and 10-second timeout |
| `OttieExpress/Services/AnthropicModels.swift`  | Create | Define `Codable` and `Sendable` request/response DTOs for the Anthropic `/v1/messages` endpoint                                                                                                          |

### Integration Points

**AnthropicService.swift**

- Imports from: `AnthropicModels.swift` (request/response DTO types)
- Imported by: None within Phase 1 (consumed by Chapter 2 poem exchange views in Phase 5)
- State reads: None
- State writes: None

**AnthropicModels.swift**

- Imports from: None (standalone DTO definitions using Foundation only)
- Imported by: `AnthropicService.swift` (encodes request DTOs, decodes response DTOs)
- State reads: None
- State writes: None

---

## Out of Scope

- Chapter 2 UI screens that consume the API responses (Her Turn Screen, Haiku Stage 1 Display, Haiku Stage 2 Display are feature epics in Phase 5)
- Fallback message display logic when API calls fail (UI concern handled by Chapter 2 view code; this epic returns errors, the view decides what to show)
- Typewriter animation of API response text (covered by `TypewriterText` component in E2.2)
- Loading state UI (cursor blink on empty screen for up to 10 seconds) described in SS16 (view-layer concern in Chapter 2 feature epic)
- Rate limiting, retry logic, or request queuing (SS16 specifies a single-user app with no concurrency concerns; a 10-second timeout with error propagation is sufficient)
- API key rotation, refresh, or remote configuration (single hardcoded-equivalent key loaded from environment; spend cap is the mitigation per SS16 Accepted Security Risks)

---

## Definition of Done

- [ ] `AnthropicService` struct conforms to `Sendable` by construction (all stored properties are immutable `let` constants)
- [ ] API key loads from `ProcessInfo.processInfo.environment["ANTHROPIC_API_KEY"]` at initialization, not hardcoded in source
- [ ] `generateReply(to:)` sends a POST request to `https://api.anthropic.com/v1/messages` with the SS8 Stage 1 system prompt and the user's poem text as the user message
- [ ] `generateReply(to:)` uses model `claude-haiku-4-5-20251001` and max tokens `300`
- [ ] `generateContinuation(of:)` sends a POST request to the same endpoint with the SS8 Stage 2 system prompt and the user's poem text as the user message
- [ ] `generateContinuation(of:)` uses model `claude-haiku-4-5-20251001` and max tokens `300`
- [ ] Both methods set a 10-second `timeoutInterval` on the `URLRequest`
- [ ] Both methods include `anthropic-version` header set to `2023-06-01` and `x-api-key` header set to the loaded API key
- [ ] Both methods include `Content-Type: application/json` header
- [ ] Both methods return `String` (the extracted text content from the first content block of the API response)
- [ ] Request DTO (`AnthropicRequest`) conforms to `Codable` and `Sendable`
- [ ] Response DTO (`AnthropicResponse`) conforms to `Codable` and `Sendable`, with nested `Content` type for the content array
- [ ] Both methods throw a typed error when the API returns a non-2xx status code, when JSON decoding fails, or when the response contains no text content blocks
- [ ] Project compiles with zero errors after adding both files

---

## Implementation Notes

SS16 API Integration provides the canonical interface:

```swift
struct AnthropicService: Sendable {
    private let apiKey: String
    private let endpoint = "https://api.anthropic.com/v1/messages"
    private let model = "claude-haiku-4-5-20251001"

    func generateReply(to poem: String) async throws -> String
    func generateContinuation(of poem: String) async throws -> String
}
```

**API key storage divergence:** SS16 specifies a hardcoded API key. CLAUDE.md SS10.4 supersedes: "Secrets loaded from environment variables or `.env` files (gitignored)." Load the key via `ProcessInfo.processInfo.environment["ANTHROPIC_API_KEY"]`. If the key is absent, the initializer should store an empty string; the methods will fail at call time with a clear error. This preserves `Sendable` by keeping stored properties immutable. Reference: CLAUDE.md SS10.4, SS16 API Integration.

**SS8 Stage 1 system prompt** (used by `generateReply`):

```
You are replying as Dinesh, a loving and soft-spoken boyfriend who is a nerdy engineer.
You are deeply in love and your tone is warm, gentle, and sincere.
You call her baby, bebe, mi amor, and my love.
You use soft and loving language throughout.
You do not use em dashes anywhere.
You may use these emojis naturally: 🥰 ♥️ 🤓
Write a short, heartfelt reply to her poem.
3 to 5 lines.
Sound like a real person responding to someone they love, not a formal poem critique.
Match her emotional register and respond with warmth and recognition.
```

Reference: SS8 Chapter 2, API: Haiku Stage 1 (Reply).

**SS8 Stage 2 system prompt** (used by `generateContinuation`):

```
You are Dinesh, a loving and soft-spoken boyfriend who is a nerdy engineer.
You are continuing her poem as if it is one continuous poem written by both of you together.
Pick up from her last line and extend it.
Match her rhythm, her imagery, and her tone exactly.
Write 3 to 5 additional lines that feel like a natural extension of what she wrote.
It should read as one poem, not two separate pieces.
Do not use em dashes anywhere.
Use soft, loving, warm language.
You may use these emojis naturally where appropriate: 🥰 ♥️ 🤓
```

Reference: SS8 Chapter 2, API: Haiku Stage 2 (Continuation).

**Request body structure** per the Anthropic Messages API:

```json
{
  "model": "claude-haiku-4-5-20251001",
  "max_tokens": 300,
  "system": "<system prompt>",
  "messages": [{ "role": "user", "content": "<poem text>" }]
}
```

**Response parsing:** The response JSON contains a `content` array. Extract the `text` field from the first element where `type == "text"`. If no text content block exists, throw an error. Reference: SS16 API Integration, SS8 error handling fallback rule.

**Required headers:**

| Header              | Value                                               |
| ------------------- | --------------------------------------------------- |
| `Content-Type`      | `application/json`                                  |
| `x-api-key`         | Value from `ANTHROPIC_API_KEY` environment variable |
| `anthropic-version` | `2023-06-01`                                        |

**Error type:** Define a nested `AnthropicService.Error` enum conforming to `Swift.Error` with cases for `missingAPIKey`, `httpError(statusCode: Int)`, `decodingFailed`, and `emptyResponse`. This keeps error handling explicit and typed. Reference: SS4 Architecture Overview, error handling at system boundaries.

`AnthropicService` is not annotated `@MainActor` because it performs no UI work. Its `Sendable` conformance is structural (immutable stored properties). The `async throws` methods execute on the cooperative thread pool via `URLSession.shared.data(for:)`. Callers in view code invoke these methods within `.task {}` modifiers for automatic cancellation on view disappear. Reference: SS4 Concurrency Model.
