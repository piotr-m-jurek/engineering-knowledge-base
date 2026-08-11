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

## Tier 4 — Durable AI workflows

### 7. Multi-step AI processes that survive restarts

Some AI tasks aren't a single LLM call — they span multiple steps, external waits, and retry loops that must survive process crashes. Examples in this app:

- **Budget review**: fetch 3 months of transactions → call LLM for analysis → wait for user approval → apply reallocation. The wait can be hours.
- **Anomaly investigation**: detect anomaly → call LLM for explanation → send notification → wait for user acknowledgement → update model.

The naïve approach (chain of `Effect.flatMap` in memory) loses all state on restart. You need **durable execution**.

**Two options at different scale points:**

| | Effect Workflows (`unstable/workflow`) | Temporal |
|---|---|---|
| Deployment | In-process, no extra infra | Separate Temporal server |
| State persistence | Pluggable (memory for dev, custom for prod) | Built-in, event-sourced |
| Scale | Single service, moderate load | Distributed, millions of executions |
| When | Fits this weekend project | When you outgrow in-process |

See [[durable-execution]] for the full concept.

### Effect Workflows — what it gives you

Effect v4 ships `unstable/workflow` — a durable execution engine embedded in the Effect runtime. The key types:

- **`Workflow.make`** — defines a named, typed workflow with payload/success/error schemas and an idempotency key
- **`Activity.make`** — wraps an effect as a named step whose result is stored; on replay, the stored result is returned instead of re-executing
- **`WorkflowEngine`** — the service that registers workflows, tracks execution state, and resumes suspended runs
- **`DurableClock`** — sleep for hours/days without holding a fiber open
- **`DurableDeferred`** — await an external signal (user approval, webhook) durably

```typescript
// packages/llm-acl/src/BudgetReviewWorkflow.ts
import { Workflow, Activity, DurableClock } from "effect/unstable/workflow"
import { Duration, Schema } from "effect"

// 1. Define activities — each result is stored durably
const fetchThreeMonths = Activity.make({
  name: "BudgetReview/fetchTransactions",
  success: Schema.Array(TransactionSchema),
  execute: Effect.fn("fetchTransactions")(function* (envelopeId: string) {
    return yield* TransactionRepository.findLastNMonths(envelopeId, 3)
  }),
})

const callLLMAnalysis = Activity.make({
  name: "BudgetReview/llmAnalysis",
  success: Schema.String,
  execute: Effect.fn("llmAnalysis")(function* (transactions: readonly Transaction[]) {
    return yield* OverspendExplainer.explain(transactions)
  }),
})

// 2. Define the workflow
export const BudgetReviewWorkflow = Workflow.make("BudgetReview", {
  payload: { envelopeId: Schema.String, userId: Schema.String },
  idempotencyKey: (p) => `${p.userId}-${p.envelopeId}`,
  success: Schema.String,
})

// 3. Register the handler
export const BudgetReviewWorkflowLayer = BudgetReviewWorkflow.toLayer(
  function* (payload) {
    // Each activity result is stored — crash here and resume picks up from last stored result
    const transactions = yield* fetchThreeMonths(payload.envelopeId)
    const analysis    = yield* callLLMAnalysis(transactions)

    // Wait up to 24h for user approval — no fiber held open
    yield* DurableClock.sleep(Duration.hours(24))

    return analysis
  }
)
```

```typescript
// Execute from api layer — discard = true means fire and poll
const executionId = yield* BudgetReviewWorkflow.execute(
  { envelopeId, userId },
  { discard: true }
)

// Later — poll for result
const result = yield* BudgetReviewWorkflow.poll(executionId)
```

**Why this matters architecturally:** The workflow handler looks like a normal `Effect.gen` function. The `WorkflowEngine` is the `Context.Service` that injects durability — swap `WorkflowEngine.layerMemory` for a persistent engine and the handler is unchanged. Same DDD payoff: behaviour in domain/application code, infrastructure concern in the `Layer`.

### When to reach for Temporal instead

- Multiple services need to participate in one workflow (cross-process coordination)
- You need Temporal's UI for workflow inspection and debugging
- Execution count is large enough that in-process state becomes a constraint

The programming model is identical in concept — activities, workflows, durable timers, signals for external events. The difference is operational: Temporal is infrastructure you run separately. [[durable-execution]] covers both.

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

- [[ddd-finance-app]] — base architecture
- [[durable-execution]] — Tier 4 concept: Temporal, Effect Workflows, durable activities
- [[process-manager-saga]] — the manual pattern that durable execution automates
- [[domain-driven-design]]
- [[ddd-bounded-context]]
- [[ports-and-adapters]]
- [[effect-context-and-layers]]
