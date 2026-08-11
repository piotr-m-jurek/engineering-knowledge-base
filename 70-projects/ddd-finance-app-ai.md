---
title: DDD Finance App — AI Extensions
type: project
created: 2026-08-11
tags: [ddd, effect, finance, ai, llm, project]
effect-version: v4
---

# DDD Finance App — AI Extensions

LLM integration points for [[ddd-finance-app]]. Ordered by value/complexity.

**Core principle:** LLM output is always untrusted input. Every result passes through normal domain validation. No special paths.

---

## Tier 1 — High value, low risk

### 1. Natural-language transaction entry

User types: *"coffee at Blue Bottle $4.50 yesterday"*
LLM extracts structured draft → domain validates → same invariants apply.

```
UserInput → TransactionParser (LLM) → ParsedDraft → Transaction.record() → TransactionRecorded
```

LLM never writes to state. It translates intent into a domain-shaped object. If parsing fails or the draft fails domain validation, same error paths as normal input.

```typescript
// packages/llm-acl/src/TransactionParser.ts
import { Context, Effect, Schema } from "effect"

export const ParsedTransactionDraft = Schema.Struct({
  amountCents:   Schema.Int,
  payee:         Schema.String,
  dateOffset:    Schema.Int,     // days from today: 0 = today, -1 = yesterday
  envelopeHint:  Schema.optional(Schema.String),
  type:          Schema.Literal("expense", "income"),
})
export type ParsedTransactionDraft = typeof ParsedTransactionDraft.Type

export class LLMParseError extends Data.TaggedError("LLMParseError")<{
  readonly input: string
  readonly reason: string
}> {}

export class TransactionParser extends Context.Service<TransactionParser, {
  parse(text: string): Effect.Effect<ParsedTransactionDraft, LLMParseError>
}>()("llm-acl/TransactionParser") {}
```

```typescript
// api route wires it:
const recordFromText = Effect.fn("recordFromText")(function* (text: string) {
  const parser = yield* TransactionParser
  const draft  = yield* parser.parse(text)

  const date = new Date()
  date.setDate(date.getDate() + draft.dateOffset)

  return yield* RecordTransactionService.record({
    envelopeRef:  draft.envelopeHint ?? defaultEnvelopeRef,
    amountCents:  draft.amountCents,
    payee:        draft.payee,
    date,
    type:         draft.type,
  })
})
```

**Why this tier:** highest UX payoff. `TransactionParser` is behind a `Context.Service` — tests mock it, production swaps in an OpenAI/Anthropic impl. Domain unchanged.

### 2. Envelope suggestion

After parsing, rank envelopes by relevance given merchant + description + past patterns.

```typescript
export class EnvelopeSuggester extends Context.Service<EnvelopeSuggester, {
  suggest(
    payee: string,
    description: string,
    history: ReadonlyArray<{ payee: string; envelopeRef: string }>,
  ): Effect.Effect<ReadonlyArray<{ envelopeRef: string; confidence: number }>, never>
}>()("llm-acl/EnvelopeSuggester") {}
```

Read-only. No domain mutation. Failure returns empty array → UI falls back to manual selection.

---

## Tier 2 — Medium value

### 3. Budget setup from income description

User: *"I make $4,200/month after tax, rent is $1,400, want to save 20%"*
LLM proposes envelope list → user reviews → `AddEnvelopeService` runs with normal invariant checks.

```
UserPrompt → BudgetDraftGenerator (LLM) → [{name, cents}] → UI confirmation → addEnvelope() × N
```

If LLM math is off, `AllocationExceedsBudgetIncome` fires on the first envelope that overflows. UI shows the error, user adjusts. LLM gets a second chance with the constraint surfaced.

```typescript
export class BudgetDraftGenerator extends Context.Service<BudgetDraftGenerator, {
  generate(
    description: string,
    incomeCents: number,
  ): Effect.Effect<ReadonlyArray<{ name: string; amountCents: number }>, LLMParseError>
}>()("llm-acl/BudgetDraftGenerator") {}
```

### 4. Overspend explanation

Reporting context (or a read model query) feeds data; LLM writes plain English.

```
EnvelopeBalanceUpdated (negative) → read model → OverspendExplainer → string
```

```typescript
export class OverspendExplainer extends Context.Service<OverspendExplainer, {
  explain(
    envelopeName: string,
    allocationCents: number,
    transactions: ReadonlyArray<{ payee: string; amountCents: number; date: string }>,
  ): Effect.Effect<string, never>
}>()("llm-acl/OverspendExplainer") {}
```

Zero domain mutation. Pure read + generate. Safe to add at any point.

---

## Tier 3 — High value, more work

### 5. Conversational budget assistant

Chat interface: *"move $50 from dining to entertainment"* → intent classification → confirmation → `reallocate()`.

Requires:
- Intent classifier: free text → `{ intent, params }`
- Confirmation loop before any mutation
- Tool-calling pattern: LLM calls `reallocate` as a tool, app executes it

```
UserMessage → IntentClassifier → { intent: "reallocate", from, to, amount }
            → ConfirmationStep (UI)
            → reallocate(budgetId, envelopeId, newAllocation)
```

**Risk:** LLM hallucinating envelope names. Mitigation: always resolve names to IDs before confirmation; show resolved names back to user.

### 6. Receipt OCR → transaction

Image → text extraction (vision model) → same `TransactionParser` pipeline as Tier 1.

```
Image → VisionModel.extract() → raw text → TransactionParser.parse() → ParsedDraft → Transaction.record()
```

`VisionModel` is another `Context.Service` behind the same ACL boundary. Tier 1 must exist first.

---

## Tier 4 — Structural / infrastructure

### 7. Anomaly detection as a background fiber

Runs as a long-lived `Effect.forkScoped` fiber alongside the event bus. Subscribes to `TransactionRecorded` events, maintains a rolling window of spend per envelope, calls LLM only when a statistical threshold is crossed.

```typescript
// Not an LLM call on every transaction — only when z-score > threshold
Effect.forkScoped(
  Stream.fromPubSub(hub).pipe(
    Stream.filter(e => e._tag === "TransactionRecorded"),
    Stream.mapEffect(e => AnomalyDetector.check(e)),
    Stream.runForEach(alert => Notifications.send(alert)),
  )
)
```

Fits the existing `InProcessEventBusLayer` composition point. Add `AnomalyDetector` to `Layer.mergeAll` — nothing else changes.

This is the DDD payoff extended to AI: the architecture absorbs a new capability by adding a `Layer`, not by touching domain code.

---

## Package structure

```
packages/
└── llm-acl/             # all LLM services live here
    ├── src/
    │   ├── TransactionParser.ts
    │   ├── EnvelopeSuggester.ts
    │   ├── BudgetDraftGenerator.ts
    │   ├── OverspendExplainer.ts
    │   ├── IntentClassifier.ts    # Tier 3
    │   ├── AnomalyDetector.ts     # Tier 4
    │   └── errors.ts
    └── test/
        └── TransactionParser.test.ts  # mock impl, no real API calls
```

**Rule:** `api` imports from `llm-acl`. Domain never imports from `llm-acl`. `llm-acl` imports from `shared-kernel` for `Money`, `BudgetPeriod`. No other cross-context imports.

---

## Effect patterns for LLM services

All LLM services follow the same `Context.Service` + `Layer` pattern as repositories:

```typescript
// Production impl
const OpenAITransactionParserLayer = Layer.effect(
  TransactionParser,
  Effect.gen(function* () {
    const client = yield* OpenAIClient  // another Context.Service
    return TransactionParser.of({
      parse: Effect.fn("TransactionParser.parse")(function* (text) {
        const result = yield* client.complete({
          model: "gpt-4o-mini",
          response_format: { type: "json_object" },
          messages: [{ role: "user", content: buildPrompt(text) }],
        })
        return yield* Schema.decodeUnknown(ParsedTransactionDraft)(JSON.parse(result))
          .pipe(Effect.mapError(e => new LLMParseError({ input: text, reason: String(e) })))
      }),
    })
  })
)

// Test impl — no network, no API key
const MockTransactionParserLayer = Layer.succeed(
  TransactionParser,
  TransactionParser.of({
    parse: (_text) => Effect.succeed({
      amountCents: 450,
      payee: "Blue Bottle",
      dateOffset: -1,
      type: "expense",
    }),
  })
)
```

Swap layer in `main.ts`. Tests use mock. CI uses mock. Production uses real. Same domain code throughout.

---

## Related

- [[ddd-finance-app]]
- [[domain-driven-design]]
- [[ddd-bounded-context]]
- [[ports-and-adapters]]
- [[effect-context-and-layers]]
