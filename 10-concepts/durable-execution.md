---
title: Durable Execution
type: concept
created: 2026-08-11
tags: [distributed-systems, workflows, temporal, effect, reliability, async]
effect-source: 80-resources/effect-source
---

# Durable Execution

## The problem

You have a multi-step process. Step 1 calls an external API. Step 2 waits for a user to approve something — could be hours. Step 3 writes results back. The process takes minutes, hours, or days.

Normal in-memory code can't do this safely:
- Process crashes mid-way → start over, possibly re-charging a card or re-sending an email
- Server deploys mid-execution → all in-flight work lost
- Step 2 waits hours → you're holding a thread/fiber the whole time

The naïve fix is a state machine stored in a database with a background worker polling it. That works but it's manual, error-prone, and the business logic gets buried in polling loops, status columns, and retry tables. This is exactly what [[process-manager-saga]] describes.

**Durable execution** solves the same problem at the framework level: it makes long-running processes look like normal sequential code that simply cannot lose state.

---

## Core idea

> Record every step's result. On restart, replay from the last stored result instead of re-executing.

```
Normal execution:
  step1() → step2() → step3()
  crash mid-step2 → start over from step1

Durable execution:
  step1() → [result stored] → step2() → [result stored] → step3()
  crash mid-step2 → restart → step1 returns stored result → step2 retries from stored input
```

The code looks identical. The framework intercepts calls at marked **activity** boundaries, stores the result, and replays it if the process is resumed.

---

## Mental model: [[event-sourcing]] for execution

The workflow engine maintains an **event log** of everything that happened. On restart it replays the log to reconstruct state:

```
Log:
  ActivityCompleted { name: "fetchUser", result: { id: 42, ... } }
  ActivityCompleted { name: "chargeCard", result: { transactionId: "ch_123" } }
  TimerFired       { name: "sendReminder" }
  ...
```

Replaying the log is deterministic: given the same log, you reach the same state. The workflow function itself must be **deterministic** — no `Date.now()`, no `Math.random()` directly in workflow code. Use framework-provided durable clock and random instead.

---

## Key concepts

### Workflow

A named, typed function that defines a long-running process. It orchestrates activities and durable waits. Looks like normal sequential code.

```typescript
// Temporal (TypeScript SDK)
import { proxyActivities, sleep } from "@temporalio/workflow"

const { fetchTransactions, callLLM } = proxyActivities({ startToCloseTimeout: "30s" })

export async function budgetReview(envelopeId: string) {
  const transactions = await fetchTransactions(envelopeId)  // stored if succeeds
  const analysis     = await callLLM(transactions)           // stored if succeeds
  await sleep("24 hours")                                    // durable — no fiber held
  return analysis
}
```

```typescript
// Effect v4 (effect/unstable/workflow)
import { Workflow, Activity, DurableClock } from "effect/unstable/workflow"

export const BudgetReviewWorkflow = Workflow.make("BudgetReview", {
  payload: { envelopeId: Schema.String },
  idempotencyKey: (p) => p.envelopeId,
  success: Schema.String,
})

export const BudgetReviewLayer = BudgetReviewWorkflow.toLayer(function* (payload) {
  const transactions = yield* fetchTransactionsActivity(payload.envelopeId)
  const analysis     = yield* callLLMActivity(transactions)
  yield* DurableClock.sleep(Duration.hours(24))
  return analysis
})
```

Both look like normal code. Both survive restarts.

### Activity

A single step with a stored result. Idempotent by design — if the activity already completed in a previous execution, its stored result is returned without re-running the effect.

**Rule:** anything with side effects belongs in an activity — DB writes, HTTP calls, LLM calls, email sends. Non-side-effectful computation can live directly in workflow code.

```typescript
// Temporal
export async function chargeCard(amount: number, customerId: string) {
  return stripe.charges.create({ amount, customer: customerId })
}
// If chargeCard already succeeded and result is in the log,
// Temporal returns the stored result — Stripe is NOT called again.
```

```typescript
// Effect v4
const chargeCardActivity = Activity.make({
  name: "chargeCard",
  success: ChargeSchema,
  execute: Effect.fn("chargeCard")(function* (amount, customerId) {
    return yield* StripeClient.charge(amount, customerId)
  }),
})
```

### Signals / DurableDeferred

External events that wake up a waiting workflow. A webhook arriving, a user clicking "approve" — these become signals that the workflow can await durably.

```typescript
// Temporal
import { defineSignal, setHandler, condition } from "@temporalio/workflow"

const approvalSignal = defineSignal<[boolean]>("approval")
let approved = false

setHandler(approvalSignal, (value) => { approved = value })

export async function requiresApproval(data: string) {
  await condition(() => approved, "7 days")  // wait up to 7 days for signal
  if (!approved) throw new Error("Timed out waiting for approval")
  return process(data)
}
```

```typescript
// Effect v4
import { DurableDeferred } from "effect/unstable/workflow"

// DurableDeferred is a named, storable deferred value
// workflow awaits it; external code resolves it via engine.deferredDone()
```

### Idempotency

Workflows are identified by an **execution ID** derived from business keys. Starting the same workflow twice with the same execution ID returns the existing execution rather than starting a new one. This makes "at-least-once trigger, exactly-once execution" achievable.

```typescript
// Temporal
await client.workflow.start(budgetReview, {
  workflowId: `budget-review-${envelopeId}-${month}`,  // deterministic key
  args: [envelopeId],
})
// If this workflowId is already running/completed, returns existing handle
```

```typescript
// Effect v4
// idempotencyKey in Workflow.make derives executionId deterministically
// execute() is idempotent: same key = same execution
const executionId = yield* BudgetReviewWorkflow.execute({ envelopeId })
```

---

## Temporal specifically

Temporal is an open-source durable execution platform. You run a Temporal server (or use Temporal Cloud) and connect your workers to it.

```
┌─────────────────────┐     ┌──────────────────────────────┐
│  Your application   │────▶│  Temporal Server             │
│  (workflow client)  │     │  - stores event history      │
│                     │     │  - schedules workflow runs   │
└─────────────────────┘     │  - manages task queues       │
                            └──────────────────────────────┘
                                           │
                            ┌──────────────┴───────────────┐
                            │  Your workers                │
                            │  - execute workflow code     │
                            │  - execute activity code     │
                            └──────────────────────────────┘
```

**Temporal Server** is the source of truth for execution state. Workers poll for tasks, execute them, and report back. Workers are stateless — they can be scaled horizontally and crash freely.

**Key Temporal concepts:**

| Concept | Temporal term | What it is |
|---|---|---|
| Long-running process | Workflow | TypeScript function registered with a worker |
| Atomic step | Activity | Function with stored result |
| External event | Signal | Named message sent to a running workflow |
| Await external | `condition()` | Block until predicate is true (driven by signals) |
| Durable sleep | `sleep(duration)` | Timer stored server-side, no fiber held |
| Execution identity | Workflow ID | Business-meaningful string key |
| Execution instance | Run ID | Generated per-run |
| Query current state | Query | Read-only function callable on running workflow |

**What Temporal gives you that in-process doesn't:**
- UI for browsing, debugging, and replaying workflow executions
- Cross-process workflows: workflow on worker A calls activities on worker B
- Versioning: update workflow code without breaking in-flight executions
- Millions of concurrent executions without a single process owning state
- Temporal Cloud: fully managed, no server to run

**What Temporal costs:**
- Operational overhead: server to deploy, monitor, upgrade (or pay for Cloud)
- Workflow determinism constraint is strict — harder to debug when violated
- TypeScript SDK bundles workflow sandboxing, adds startup latency
- Separate debugging context: Temporal UI instead of local logs

---

## Effect Workflows vs Temporal

Effect v4 ships `unstable/workflow` — the same durable execution model in a `Layer`.

```
effect/unstable/workflow/
  Workflow.ts       — Workflow.make(), .execute(), .poll(), .interrupt()
  Activity.ts       — Activity.make(), result stored by WorkflowEngine
  WorkflowEngine.ts — Context.Service; layerMemory for dev/test
  DurableClock.ts   — durable sleep
  DurableDeferred.ts — await external signal
  DurableQueue.ts   — durable message queue within a workflow
```

**`WorkflowEngine`** is a `Context.Service`. `WorkflowEngine.layerMemory` keeps state in-process (good for tests and weekends). A persistent engine (`WorkflowEngine.makeUnsafe(...)`) backed by a DB or Temporal is a `Layer` swap — workflow handler code is unchanged.

```typescript
// Dev / test
const AppLayer = Layer.mergeAll(
  WorkflowEngine.layerMemory,
  BudgetReviewWorkflowLayer,
  ...
)

// Production (hypothetical persistent engine)
const AppLayer = Layer.mergeAll(
  PersistentWorkflowEngineLayer,  // same WorkflowEngine interface
  BudgetReviewWorkflowLayer,      // unchanged
  ...
)
```

This is the same pattern as swapping `InMemoryBudgetRepository` for `SqliteBudgetRepository` — business logic is untouched, infrastructure is in the `Layer`.

**Practical split:**

| Scenario | Use |
|---|---|
| Weekend project, single service | `WorkflowEngine.layerMemory` |
| Production, single service, moderate load | Effect Workflows + persistent engine |
| Multi-service, large scale, need UI/versioning | Temporal |

---

## Relationship to Process Manager / Saga

[[process-manager-saga]] shows the manual pattern: a persisted state-machine aggregate + background worker + retry table. That pattern is correct and battle-tested.

Durable execution is the same idea automated:
- The "state machine" is the event log managed by the engine
- The "background worker" is the workflow worker polling for tasks
- The "retry table" is handled by activity retry policies

When to use which:

| | Process Manager (manual) | Durable Execution |
|---|---|---|
| Control | Full — you own every bit | Framework-managed |
| Complexity | High — state machine, worker, retries | Low — write sequential code |
| Visibility | SQL queries against your own tables | Temporal UI or engine-specific |
| Portability | No extra infra | Requires engine (in-process or Temporal) |

Neither replaces the other. A complex DDD process with rich invariants still benefits from modeling as a domain aggregate. Durable execution handles the *plumbing* around it — durability, retries, waits — not the domain rules themselves.

---

## Determinism constraint

Workflow code (not activity code) must be deterministic. The engine replays the log to reconstruct state — non-deterministic code breaks replay.

**Forbidden in workflow code (Temporal):**
```typescript
// ❌ non-deterministic
const id = crypto.randomUUID()        // different on each replay
const now = new Date()                // different on each replay
await fetch("https://api.example.com") // side effect — put in an activity
```

**Correct:**
```typescript
// ✅ framework-provided deterministic alternatives
import { uuid4, sleep } from "@temporalio/workflow"
const id = uuid4()                    // deterministic from event log
await sleep("1 hour")                 // stored as timer, not real sleep
```

Effect Workflows enforce this through types: the `WorkflowEngine` and `WorkflowInstance` services are only available inside workflow and activity handlers — any effect requiring them can't be called from outside.

---

## Related

- [[process-manager-saga]] — the manual version of the same problem
- [[event-sourcing]] — same replay mechanism; durable execution applies it to workflow position rather than domain state
- [[ddd-domain-events]] — workflows triggered by domain events
- [[ddd-finance-app-ai]] — Tier 4: durable AI workflows in the finance app
- [[ddd-finance-app]] — base architecture these workflows plug into
- [[ports-and-adapters]] — WorkflowEngine as a port; layerMemory vs persistent as adapters

## References

- Temporal docs: https://docs.temporal.io
- Effect v4 workflow source: `80-resources/effect-source/packages/effect/src/unstable/workflow/`
- Effect v4 workflow entry: `80-resources/effect-source/packages/effect/src/unstable/workflow/index.ts`
- Maxim Fateev, *Designing a Workflow Engine from First Principles* (Temporal blog)
