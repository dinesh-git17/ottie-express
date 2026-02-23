# Character Class Timing Reference

## 1. Character Classification Tree

```
Character
├── Whitespace
│   ├── Space (U+0020)           → spaceDelay
│   ├── Newline (U+000A)         → newlineDelay (check next char for paragraph break)
│   └── Tab (U+0009)             → spaceDelay
├── Punctuation — Sentence-Terminal
│   ├── Period (U+002E)          → periodDelay (check next chars for ellipsis)
│   ├── Exclamation (U+0021)     → exclamationDelay
│   ├── Question (U+003F)        → questionDelay
│   └── Interrobang (U+203D)     → exclamationDelay
├── Punctuation — Clause-Internal
│   ├── Comma (U+002C)           → commaDelay
│   ├── Semicolon (U+003B)       → semicolonDelay
│   ├── Colon (U+003A)           → colonDelay
│   └── Em Dash (U+2014)         → emDashDelay
├── Punctuation — Paired Delimiters
│   ├── Open Quotes (" ' " ')    → openQuoteDelay
│   ├── Close Quotes (" ' " ')   → closeQuoteDelay
│   ├── Open Paren/Bracket       → openQuoteDelay
│   └── Close Paren/Bracket      → closeQuoteDelay
├── Special Tokens
│   ├── Ellipsis (U+2026)        → ellipsisDelay (single character)
│   ├── Three-dot (... sequence) → ellipsisDelay (advance by 3)
│   └── Paragraph (\n\n)         → paragraphBreakDelay (advance by 2)
└── Default (letters, digits, symbols) → baseLetterDelay
```

## 2. Multi-Character Token Detection

Multi-character tokens must be detected via lookahead before single-character classification:

| Token Pattern | Detection Logic                            | Advance Count | Delay Applied         |
| ------------- | ------------------------------------------ | ------------- | --------------------- |
| `\n\n`        | `current == \n && next == \n`              | 2             | `paragraphBreakDelay` |
| `...`         | `current == . && next == . && next+1 == .` | 3             | `ellipsisDelay`       |
| `--`          | `current == - && next == -`                | 2             | `emDashDelay`         |

Check multi-character tokens first. If none match, fall through to single-character classification.

## 3. Delay Value Derivation

### Design Rationale

Timing values derive from reading rhythm research and typographic convention:

- **Base letter delay (30–50ms)**: At 40ms, produces ~25 characters/second. Average reading speed is ~250 words/minute (~20 characters/second). Slightly faster than reading to maintain forward momentum.
- **Space delay (20–35ms)**: Shorter than letters because word boundaries are processed subconsciously. Faster spaces maintain word-level rhythm.
- **Comma delay (120–180ms)**: Simulates the natural breath pause at a clause boundary. Equal to 3–4x the base letter delay.
- **Period delay (200–350ms)**: Marks sentence completion. The reader's internal voice pauses here. Equal to 5–8x the base letter delay.
- **Paragraph break (400–700ms)**: The longest pause. Signals a shift in thought, scene, or speaker. Provides visual breathing room before the next block of text begins.
- **Ellipsis delay (300–500ms)**: Signals suspension, trailing off, or dramatic pause. Longer than a period because the ellipsis implies continuation that hasn't arrived yet.

### Preset Comparison

| Character Class | Standard | Fast  | Dramatic |
| --------------- | -------- | ----- | -------- |
| Base letter     | 40ms     | 15ms  | 65ms     |
| Space           | 25ms     | 10ms  | 40ms     |
| Comma           | 150ms    | 60ms  | 250ms    |
| Period          | 280ms    | 100ms | 450ms    |
| Exclamation     | 280ms    | 100ms | 450ms    |
| Question        | 280ms    | 100ms | 450ms    |
| Semicolon       | 170ms    | 70ms  | 280ms    |
| Colon           | 150ms    | 60ms  | 250ms    |
| Ellipsis        | 400ms    | 150ms | 600ms    |
| Em dash         | 200ms    | 80ms  | 350ms    |
| Newline         | 350ms    | 120ms | 500ms    |
| Paragraph break | 550ms    | 200ms | 750ms    |
| Open quote      | 80ms     | 30ms  | 120ms    |
| Close quote     | 100ms    | 40ms  | 150ms    |

### Scale Factor Between Presets

- Fast ≈ 0.35–0.40x Standard
- Dramatic ≈ 1.4–1.6x Standard

Custom presets should maintain these proportional relationships between character classes.

## 4. Jitter Implementation

Jitter adds bounded randomness to prevent mechanical cadence:

```
jitteredDelay = baseDelay × randomFactor
randomFactor ∈ [1.0 - maxJitter, 1.0 + maxJitter]
```

Default `maxJitter = 0.15` (±15%).

### Jitter Distribution

Use uniform distribution within the range. Normal distribution concentrates values around the mean, which weakens the anti-mechanical effect.

### Jitter Scope

Jitter applies per-character, not per-word or per-sentence. Each character gets an independent random factor.

### Reproducible Jitter for Testing

For deterministic test assertions, disable jitter (`jitterEnabled = false`). The standard configuration disables jitter by default. Enable it only in production or when specifically testing jitter behavior.

## 5. Context-Sensitive Timing Adjustments

Beyond the character class table, certain narrative contexts warrant timing overrides:

| Context                     | Adjustment                                                    |
| --------------------------- | ------------------------------------------------------------- |
| Dialogue (inside quotes)    | Reduce base letter delay by 15–20%. Dialogue reads faster.    |
| Emphasis (all caps or bold) | Increase base letter delay by 20–30%. Weight draws attention. |
| Short sentence (< 5 words)  | No adjustment. Short sentences carry inherent punch.          |
| List items                  | Reduce newline delay by 30%. Items flow as a sequence.        |

These adjustments require the caller to mark text regions with metadata. The base timing engine does not infer context from content — it receives explicit timing overrides.
