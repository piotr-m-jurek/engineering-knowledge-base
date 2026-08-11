---
title: Full-Stack Layer Options — Technology Choices
type: architecture
created: 2026-08-10
tags: [architecture, full-stack, infrastructure, databases, tradeoffs]
---

# Full-Stack Layer Options — Technology Choices

For each layer in [[full-stack-layers]], there are multiple technology options. The right choice depends on **what the business does**, not personal preference or hype.

The decision model per layer:

```
What does this layer need to do?
    ↓
What are the access patterns?
    ↓
What does the data look like?
    ↓
What are the consistency / latency / scale requirements?
    ↓
Pick the technology that fits those constraints
```

---

## Layer: Database

The most consequential choice. Wrong database = rewrite in 2 years.

### Relational (Postgres, MySQL)

**Fits when:**
- Data has clear relationships and structure
- You need ACID transactions across multiple entities
- Query patterns are varied and ad-hoc
- Correctness > raw speed

**Avoid when:**
- Write throughput is in the millions/sec and schema is fixed (→ Cassandra)
- All queries are aggregate scans over billions of rows (→ ClickHouse)
- The core feature is graph traversal at depth > 3 (→ Neo4j)

**Example business:** e-commerce orders, banking, SaaS user/billing data

```
Orders ─── OrderLines ─── Products
  └── Customers ──── Invoices
```

*Postgres is the default. Start here unless you have a specific reason not to.*

---

### Document (MongoDB, DynamoDB, Firestore)

**Fits when:**
- Data is naturally hierarchical and read as a unit (e.g. a blog post + comments)
- Schema evolves frequently (early product, lots of unknowns)
- Access pattern is almost always "give me this one document by ID"
- You need horizontal write scaling from day one

**Avoid when:**
- You need multi-document transactions (painful to add later)
- Query patterns are ad-hoc or unknown upfront — DynamoDB especially punishes this
- Your data is relational: many small entities joined in different combinations

**Example business:** CMS, user profiles, product catalog with highly variable attributes

```typescript
// document — the whole order is one unit, read together always
{
  _id: "order-123",
  customer: { id: "cust-1", name: "Alice" },
  lines: [
    { product: "prod-A", qty: 2, unitPrice: 10.00 },
    { product: "prod-B", qty: 1, unitPrice: 25.00 },
  ],
  status: "fulfilled",
  fulfilledAt: "2026-08-10"
}
```

**Tradeoff:** no joins, no multi-document transactions (without extra complexity), eventual consistency by default in distributed setups. DynamoDB additionally requires you to know all access patterns upfront (single-table design).

---

### Column-family / Wide-column (Cassandra, ScyllaDB)

**Fits when:**
- Massive write throughput is the primary constraint (millions of writes/sec)
- Data is time-series or append-only (logs, metrics, IoT sensor data)
- Reads are always by a known partition key + time range
- You can sacrifice strong consistency

**Avoid when:**
- You need ad-hoc queries or joins — there are none
- Your team is small and ops burden matters — Cassandra clusters are hard to run
- Correctness is critical — tunable consistency means you can misconfigure it silently
- You have < 1M writes/day — Postgres handles this fine

**Example business:** real-time analytics ingestion, IoT telemetry, activity feeds

```
partition key: device_id
clustering key: recorded_at DESC
→ "give me the last 1000 readings for device X" is O(1)
```

**Tradeoff:** no joins, no ad-hoc queries, schema must match access patterns exactly. Multi-row transactions are complex. Bad for anything relational.

---

### Columnar / Analytical (ClickHouse, BigQuery, Redshift, DuckDB)

**Fits when:**
- Queries aggregate over millions/billions of rows (`GROUP BY`, `SUM`, `AVG`, window functions)
- Data is written in bulk (ETL, event streams) not row-by-row
- Latency of seconds is fine, throughput is everything
- Workload is analytical (OLAP), not transactional (OLTP)

**Avoid when:**
- You need point lookups by ID — columnar layout makes these slow
- Data is written row-by-row at high frequency — columnar ingestion is batch-oriented
- You need to update or delete individual rows frequently
- Your dataset fits in Postgres with a materialized view — don't add infra you don't need

**Example business:** business intelligence, dashboards, data warehouse, log analytics

```sql
-- columnar stores compress and scan this in milliseconds across 1B rows
SELECT
  date_trunc('week', fulfilled_at) as week,
  sum(total_amount) as revenue,
  count(*) as orders
FROM orders
GROUP BY week
ORDER BY week DESC
```

**Tradeoff:** terrible for point lookups, high-frequency writes, or anything transactional. Not a replacement for Postgres — it's a complement.

DuckDB is notable: runs in-process (like SQLite), excellent for local analytics and data pipelines.

---

### Graph (Neo4j, Amazon Neptune)

**Fits when:**
- Relationships between entities *are* the data
- Queries traverse unknown depth (e.g. "friends of friends", "what depends on what")
- Highly connected data where relational joins become expensive at depth > 2–3

**Avoid when:**
- Traversal is not the core feature — a recursive CTE in Postgres usually suffices
- Your team doesn't know Cypher/Gremlin — unfamiliar query language is a real cost
- You need bulk analytics across nodes — graph DBs are poor at aggregations
- The graph fits in memory — a simple adjacency list in your app or a Postgres table is fine

**Example business:** social networks, fraud detection (rings of connected accounts), recommendation engines, dependency graphs, knowledge graphs

```cypher
// "Find all products bought by people who also bought product X"
// trivial in graph, painful join chain in SQL
MATCH (p:Product {id: "prod-A"})<-[:BOUGHT]-(u:User)-[:BOUGHT]->(rec:Product)
WHERE rec.id <> "prod-A"
RETURN rec.name, count(u) as score
ORDER BY score DESC
LIMIT 10
```

**Tradeoff:** poor fit for bulk analytics, unfamiliar query language (Cypher/Gremlin), smaller ecosystem than Postgres. Don't use it unless the traversal problem is central.

---

### Vector (pgvector, Qdrant, Weaviate, Pinecone)

**Fits when:**
- Similarity search is a core access pattern ("find items semantically similar to X")
- Data is embedded as high-dimensional vectors (ML embeddings from text, images, audio)
- You need approximate nearest-neighbour (ANN) search at scale

**Avoid when:**
- You need exact keyword match — full-text search is faster and simpler
- Your embedding quality is unknown or untested — bad embeddings make vector search useless regardless of the DB
- Dataset is < 100k rows — pgvector on Postgres is sufficient; no new infra needed

**Example business:** semantic search, recommendation (embedding-based), RAG pipelines, image similarity, duplicate detection

```typescript
// pgvector — Postgres extension, keeps vector search in your existing DB
// Store embedding alongside the row
await db.execute(sql`
  SELECT id, title, embedding <=> ${queryEmbedding} AS distance
  FROM documents
  ORDER BY distance
  LIMIT 10
`)
```

**pgvector** is often the right first choice: no new database, Postgres semantics, good enough for millions of rows. Move to Qdrant/Pinecone when you need billions of vectors or more index types.

**Tradeoff:** ANN search is approximate (tunable). Embedding quality matters more than index choice. Not a general-purpose DB.

---

### Time-series (TimescaleDB, InfluxDB, Prometheus)

**Fits when:**
- Data is always written with a timestamp and queried by time range
- Compression of time-series data matters (10–100x vs raw Postgres)
- Built-in downsampling, retention policies, and time-bucketing functions are needed

**Avoid when:**
- You have < 1M time-series rows — Postgres with a `recorded_at` index handles this fine
- You need to join time-series data with relational entities frequently — awkward across two DBs
- Your team already runs Postgres and the volume doesn't justify another system

**Example business:** infrastructure metrics, financial tick data, sensor readings, application performance monitoring

```sql
-- TimescaleDB (Postgres extension) — time_bucket is native
SELECT
  time_bucket('1 hour', time) AS hour,
  avg(cpu_percent) AS avg_cpu
FROM metrics
WHERE time > now() - interval '24 hours'
GROUP BY hour
ORDER BY hour
```

**Prometheus** is narrower: pull-based metrics scraping, not a general-purpose time-series store. Use it for infrastructure observability, not business data.

---

### Key-value (Redis, DynamoDB, etcd)

**Fits when:**
- Access is always `get(key)` / `set(key, value)` — no queries
- Latency must be sub-millisecond
- Data fits a simple structure (string, hash, list, set)

Redis doubles as cache + primary store for ephemeral data (sessions, rate limit counters, leaderboards, pub/sub).

**Avoid when:**
- You need to query by anything other than the key — use Postgres
- Data must be durable and loss is unacceptable — Redis AOF/RDB persistence is not as reliable as a proper DB
- You're using it as a primary data store for business data — it's designed for ephemeral/cached data

**Tradeoff:** no query language, no joins, limited data modeling. Redis data is in-memory by default — persistence requires explicit configuration.

---

### Database decision tree

```
Is correctness + transactions the primary constraint?
└─ YES → Postgres (start here, always)

Is the query pattern "aggregate millions of rows"?
└─ YES → ClickHouse / BigQuery alongside Postgres

Is relationship traversal the core feature?
└─ YES → Neo4j / Neptune

Is the primary access pattern similarity search?
└─ YES → pgvector (start) → Qdrant at scale

Is the write pattern time-series append-only?
└─ YES → TimescaleDB or InfluxDB

Is write throughput the bottleneck (millions/sec)?
└─ YES → Cassandra / ScyllaDB

Is schema highly variable and data read as whole units?
└─ YES → MongoDB / DynamoDB

Is latency the only constraint and data is simple?
└─ YES → Redis
```

---

## Layer: Cache

### In-process (Map / LRU)

**Fits when:** data is immutable or changes rarely, single instance, sub-microsecond access needed. Config values, compiled templates, feature flags.

**Avoid when:** you run multiple app instances (cache is not shared), data changes at runtime (cache goes stale silently), or you need TTL/eviction (→ Redis).

```typescript
// Effect: cache at the Layer level — initialized once, lives for app lifetime
export const FeatureFlagsLive = Layer.effect(
  FeatureFlags,
  Effect.gen(function* () {
    const db = yield* ConfigRepository
    const flags = yield* db.loadAll()  // load once at startup
    const cache = new Map(flags.map(f => [f.key, f.value]))
    return FeatureFlags.of({ get: (key) => Effect.succeed(cache.get(key) ?? false) })
  })
)
```

**Tradeoff:** lost on restart, stale across multiple instances.

---

### Distributed (Redis)

**Fits when:** multiple app instances share cache, cache must survive restart, you need TTL + eviction policies.

**Avoid when:** you have a single instance and data rarely changes — in-process cache is faster and simpler. Don't add Redis just for caching if you don't already run it.

Patterns:
- **Read-through:** check cache → miss → load from DB → populate cache
- **Write-through:** write to DB + cache atomically
- **Write-behind:** write to cache only, async flush to DB (risky)
- **Cache-aside:** application manages all cache logic explicitly (most common, most control)

**Eviction policies matter:**
- `allkeys-lru` — evict least recently used across all keys (good for caches)
- `volatile-lru` — evict LRU only among keys with TTL set
- `noeviction` — never evict, return errors when full (good for queues/sessions, bad for caches)

---

### CDN (Cloudflare, CloudFront, Fastly)

**Fits when:** responses are public, identical for all users, and geographically distributed users matter.

**Avoid when:** responses are authenticated or user-specific (CDN can't safely cache them), your users are all in one region (latency gain is marginal), or you need dynamic personalisation on every request.

```
Without CDN: user in Tokyo → origin in Frankfurt → 200ms RTT
With CDN:    user in Tokyo → Cloudflare Tokyo edge → 5ms RTT
```

Cache-Control headers control CDN behaviour:
```
Cache-Control: public, max-age=3600, stale-while-revalidate=60
```

**Tradeoff:** only works for public responses. Authenticated, user-specific responses can't be CDN-cached without careful Vary header usage.

---

## Layer: Message Bus / Event Delivery

| Technology | Model | Fits when |
|---|---|---|
| BullMQ | Job queue (Redis) | Background jobs, retries, single service |
| RabbitMQ | AMQP broker | Fan-out to multiple consumers, complex routing |
| Kafka | Persistent log | Event sourcing, audit trail, replay, high throughput |
| SQS | Managed queue | AWS-native, simple at-least-once delivery |
| NATS | Lightweight pub/sub | Low latency, IoT, internal microservice mesh |
| Postgres LISTEN/NOTIFY | In-DB pub/sub | Low volume, already on Postgres, no extra infra |
| In-process (forkDaemon) | Fiber | Non-critical fire-and-forget, no retries needed |

**Kafka vs RabbitMQ vs BullMQ** — the key distinction:

- **BullMQ:** jobs are consumed once, removed after success. No replay. Good for task queues.
- **RabbitMQ:** messages are consumed once per queue. Multiple queues = fan-out. No replay.
- **Kafka:** messages are a **persistent log**. Consumers track their own offset. You can replay from any point. Mandatory when you need event sourcing or audit trails.

### Postgres LISTEN/NOTIFY — underused option

```typescript
// No extra broker. Postgres notifies subscribers when a row changes.
// Good for low-volume internal events within one service.

// Publisher (after insert):
await db.execute(sql`NOTIFY order_events, ${JSON.stringify(event)}`)

// Subscriber:
const client = await pool.connect()
await client.query("LISTEN order_events")
client.on("notification", (msg) => {
  const event = JSON.parse(msg.payload!)
  // handle event
})
```

**Tradeoff:** max payload 8KB, no durability (if subscriber is down, notification is lost), not for high throughput. Use the outbox pattern on top to get durability.

---

## Layer: Backend HTTP / API style

The API contract shape affects client coupling, versioning, and what tooling is available.

| Style | Fits when | Avoid when |
|---|---|---|
| REST | Public API, stable resources, many consumers, HTTP caching matters | Tight type-safety needed across TS stack — codegen is painful |
| GraphQL | Frontend-driven, many clients with different data needs, avoid over-fetching | Simple API with one client; N+1 queries and schema complexity bite hard |
| tRPC | Full-stack TypeScript, internal API, type safety end-to-end without codegen | Non-TS clients, public API, or you need HTTP caching semantics |
| gRPC | Service-to-service, binary protocol, streaming, polyglot | Browser clients (gRPC-Web adds complexity); small teams unfamiliar with protobufs |
| WebSocket | Real-time bidirectional (chat, collaborative, live data) | One-way push only — SSE is simpler; request/response — HTTP is simpler |
| SSE | Server → client push only (notifications, progress, live feed) | Client needs to send messages back — use WebSocket instead |

**In Effect:** `HttpApi` handles REST natively with Schema validation. tRPC and gRPC require separate adapters. WebSocket and SSE are supported via `HttpRouter` streaming responses.

---

## Layer: Frontend state

| Approach | Fits when | Avoid when |
|---|---|---|
| Server state only (TanStack Query / SWR) | Most apps — server is source of truth, client just displays | Offline-first or heavy client-side computation |
| Client state (Zustand, Jotai) | UI-only state (modals, tabs, form wizard steps) | Data that should survive refresh or be shareable — put it in URL or server |
| URL state | Shareable filters, pagination, search — survives refresh | Sensitive data; complex nested structures (URLs get ugly fast) |
| Global store (Redux) | Complex client-side business logic — rare, usually a smell | Almost always — reach for server state + URL first |

**Rule:** keep as little state client-side as possible. If it can live in the URL or be re-fetched, it should be.

In the Effect stack: `@effect/react` `useQuery`-style hooks for server state + URL params for shareable state covers 90% of cases.

---

## Layer: Search

When your database's `LIKE` or full-text search isn't enough.

| Technology | Fits when | Avoid when |
|---|---|---|
| Postgres full-text (`tsvector`) | Simple keyword search, small-medium dataset, already on Postgres | Typo tolerance, faceted search, or relevance tuning needed |
| Elasticsearch / OpenSearch | Complex full-text, facets, relevance tuning, large dataset | Small team or small dataset — operational cost is high |
| Typesense | Fast typo-tolerant search, simpler than ES, good for product/docs search | Billions of documents or custom relevance models |
| Meilisearch | Developer-friendly, good defaults, small-medium scale | Very large datasets or complex ranking requirements |
| pgvector | Semantic / embedding-based similarity search | Exact keyword match — it's slower and less precise than full-text for that |

**Access pattern drives the choice:**
- Exact keyword match → Postgres full-text
- Fuzzy, typo-tolerant, faceted → Typesense / Meilisearch
- Semantic meaning ("show me articles about dogs" matches "canine") → pgvector + embeddings
- Billions of documents with complex relevance tuning → Elasticsearch

---

## Layer: Observability

| Signal | Technology | Fits when |
|---|---|---|
| Logs | stdout + Loki / Datadog / CloudWatch | Always. Structured JSON logs from day one. |
| Metrics | Prometheus + Grafana | Self-hosted infrastructure metrics |
| Metrics | Datadog / New Relic | Managed, full-stack, willing to pay |
| Traces | OpenTelemetry → Jaeger / Tempo | Distributed tracing across services |
| Error tracking | Sentry | Exceptions with stack traces + user context |
| Uptime | Betterstack / UptimeRobot | External black-box availability monitoring |

**Effect + OpenTelemetry:** Effect has first-class OTel support. Spans follow fiber boundaries automatically.

```typescript
import { NodeSdk } from "@effect/opentelemetry"
import { OTLPTraceExporter } from "@opentelemetry/exporter-trace-otlp-http"

const OtelLive = NodeSdk.layer(() => ({
  resource: { serviceName: "orders-service" },
  spanProcessor: new BatchSpanProcessor(new OTLPTraceExporter()),
}))
```

Every Effect operation becomes a traceable span. No manual instrumentation needed.

---

## Layer: Infrastructure / Deployment

| Choice | Fits when | Avoid when |
|---|---|---|
| Single VPS (Railway, Render, Fly.io) | Early stage, small team, fast iteration | Traffic justifies horizontal scaling; stateful workloads need more control |
| Managed containers (ECS, Cloud Run) | Medium scale, don't want to manage k8s | You need custom scheduling, multi-tenancy, or have a platform team for k8s |
| Kubernetes | Large scale, many services, fine-grained control needed | Small team — ops overhead is enormous; start simpler |
| Serverless (Lambda, Vercel functions) | Bursty traffic, pay-per-request, stateless | Long-running workers, persistent DB connections, cold start sensitivity |
| Edge (Cloudflare Workers) | Latency-critical, globally distributed, stateless | Heavy compute, large dependencies, stateful operations |

**Serverless tradeoffs:** cold starts, 15min max duration, no persistent connections (bad for Postgres without a connection pooler like PgBouncer or RDS Proxy), stateless. Good for API routes, bad for long-running workers.

**Effect + serverless:** works, but `Layer.launch` assumes a long-lived process. For Lambda, use `Effect.runPromise` per invocation without a persistent layer.

---

## The real decision process

Wrong question: *"What's the best database?"*  
Right question: *"What are my access patterns, data shape, consistency needs, and scale?"*

```
Per layer, ask:
1. What does this layer need to DO? (access pattern)
2. What does the data LOOK LIKE? (structure)
3. How often does it CHANGE vs READ? (read/write ratio)
4. What happens if it's WRONG or SLOW? (consistency, latency SLA)
5. What can we AFFORD to operate? (team size, budget)
```

**Default stack for most B2B SaaS** (covers 80% of cases):

| Layer | Default choice | Upgrade when |
|---|---|---|
| Database | Postgres | Need analytics → add ClickHouse; need vectors → add pgvector |
| Cache | Redis | — |
| Search | Postgres full-text | Dataset > 1M rows or need facets → Typesense |
| Message bus | BullMQ | Need fan-out across services → RabbitMQ; need replay → Kafka |
| API | REST (Effect HttpApi) | Type-safe internal → tRPC; real-time → WebSocket/SSE |
| Frontend state | TanStack Query + URL | — |
| Observability | OTel + Sentry + Prometheus | — |
| Deploy | Railway / Render | Traffic justifies k8s → ECS or GKE |

Start with the default. Add complexity only when you have a measured problem, not a hypothetical one.

---

## Related

- [[full-stack-layers]]
- [[full-stack-layers-example]]
- [[domain-driven-design]]
- [[ddd-bounded-context]]
- [[ports-and-adapters]]

## References

- Kleppmann, *Designing Data-Intensive Applications* (2017) — the definitive guide to database tradeoffs
- Fowler, [CQRS](https://martinfowler.com/bliki/CQRS.html)
- Fowler, [EventSourcing](https://martinfowler.com/eaaDev/EventSourcing.html)
- [Use The Index, Luke](https://use-the-index-luke.com) — SQL indexing and access patterns
- [pgvector](https://github.com/pgvector/pgvector)
- [TimescaleDB](https://www.timescale.com)
