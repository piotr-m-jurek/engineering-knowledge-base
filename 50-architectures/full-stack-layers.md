---
title: Full-Stack Application Layers
type: architecture
created: 2026-08-10
tags: [architecture, full-stack, infrastructure, caching, ddd]
---

# Full-Stack Application Layers

## Summary

A full-stack application is a vertical slice through multiple distinct layers. Each layer has a single responsibility. From a DDD perspective, only the domain and application layers matter architecturally — everything else is infrastructure.

---

## Diagram

```
┌──────────────────────────────────────────────┐
│  INFRASTRUCTURE / DEPLOYMENT                 │
│  (IaC, containers, DNS, TLS, networking)     │
├──────────────────────────────────────────────┤
│  FRONTEND  (Presentation)                    │
│  (UI components, routing, client state)      │
├──────────────────────────────────────────────┤
│  API GATEWAY / REVERSE PROXY                 │
│  (auth, rate limiting, routing, TLS offload) │
├──────────────────────────────────────────────┤
│  BACKEND HTTP  (Presentation boundary)       │
│  (request decoding, response encoding)       │
├──────────────────────────────────────────────┤
│  APPLICATION SERVICES                        │
│  (use-case orchestration, no business logic) │
├──────────────────────────────────────────────┤
│  DOMAIN  (pure business logic)               │
│  (entities, aggregates, domain services)     │
├──────────────────────────────────────────────┤
│  INFRASTRUCTURE  (adapters)                  │
│  ┌────────────┬──────────────────────────┐   │
│  │ DATABASE   │ BLOB STORAGE             │   │
│  │ SQL/NoSQL  │ S3, GCS, R2              │   │
│  ├────────────┼──────────────────────────┤   │
│  │ CACHE      │ MESSAGE BUS              │   │
│  │ Redis, CDN │ Kafka, SQS, RabbitMQ     │   │
│  ├────────────┴──────────────────────────┤   │
│  │ EXTERNAL HTTP CLIENTS                 │   │
│  │ Stripe, SendGrid, Twilio, etc.        │   │
│  └───────────────────────────────────────┘   │
├──────────────────────────────────────────────┤
│  OBSERVABILITY  (cross-cutting)              │
│  (logs, metrics, traces, alerts)             │
└──────────────────────────────────────────────┘
```

---

## Components

### Infrastructure / Deployment

Not a DDD layer — ops concern. Runs and connects everything else.

- Container orchestration (Docker, k8s, ECS)
- Infrastructure as Code (Terraform, Pulumi)
- Networking, DNS, TLS certificates
- CI/CD pipelines

### Frontend (Presentation)

Everything the user touches. Stateless relative to the server — all durable state lives in the backend.

- UI components and routing
- Client-side state management
- API communication (REST, GraphQL, tRPC)

### API Gateway / Reverse Proxy

The boundary between the public internet and your application fleet. Optional at small scale (nginx suffices), critical at scale.

- TLS termination
- Authentication and authorization enforcement
- Rate limiting and DDoS mitigation
- Request routing to multiple services
- Examples: nginx, Kong, AWS API Gateway, Cloudflare

### Backend HTTP (Presentation boundary)

The HTTP server. In DDD terms, this is still Presentation — it speaks HTTP, not domain. Its only job is decoding/encoding.

- Decode and validate incoming requests (route params, body, headers)
- Lift raw HTTP types → domain value objects
- Call application services
- Map domain errors → HTTP status codes
- Encode responses as JSON/Protobuf/etc.

→ See [[ddd-request-flow]] for the full handler-to-DB flow with Effect.

### Application Services

Orchestrate use cases. No business logic — only coordination: load aggregate, call domain method, save, publish events.

→ Defined in [[domain-driven-design#Application Service]]

### Domain Layer

The only layer that actually matters to the business. Pure logic — no I/O, no HTTP, no SQL. In Effect: zero requirements in `R`.

- [[ddd-aggregate-root]] — enforce invariants, emit events
- Value objects and entities
- Domain services (stateless operations spanning aggregates)
- Repository *interfaces* (ports)
- [[ddd-domain-events]] — facts that happened in the domain

### Infrastructure Layer (Adapters)

Implements the ports declared in the domain. The domain doesn't know these exist.

| Adapter | Purpose | Examples |
|---|---|---|
| Database | Durable relational or document storage | Postgres, MySQL, DynamoDB, MongoDB |
| Blob storage | Large binary objects (files, images, video) | S3, GCS, Cloudflare R2 |
| Cache | Fast read layer in front of DB or computation | Redis, Memcached |
| CDN | Edge cache for static assets and public reads | Cloudflare, CloudFront |
| Message bus | Async communication, event-driven workflows | Kafka, SQS, RabbitMQ, NATS |
| External HTTP | Third-party integrations | Stripe, SendGrid, Twilio |

All of these are wired via `Layer` in Effect — the domain never imports them.

### Cache

A cache is not a primary data store — it's a performance optimization. There are several distinct caching layers:

- **In-process cache** — `Map` or LRU in application memory; fastest, no network hop; lost on restart
- **Distributed cache** — Redis or Memcached; shared across multiple instances; survives restarts
- **CDN / edge cache** — HTTP response cache at geographic edge nodes; cuts origin load for public reads
- **DB query cache** — built into some databases (e.g., Postgres `pg_bouncer` + prepared statements)

Cache is always a trade-off: consistency vs. latency. Cache invalidation is notoriously hard.

### Message Bus

Decouples producers from consumers. Enables async processing, event sourcing, and cross-service integration without direct coupling.

- Producer emits an event (e.g., `OrderFulfilled`)
- Bus persists and delivers to all subscribers
- Consumer processes independently, at its own pace

→ Connects to [[ddd-domain-events]] for integration between [[ddd-bounded-context|bounded contexts]].

### Observability

Cross-cutting concern — spans all layers. Not optional.

- **Logs** — structured, queryable records of what happened
- **Metrics** — aggregated numbers over time (latency p99, error rate, throughput)
- **Traces** — distributed request paths across services
- **Alerts** — automated notification when thresholds breach

Examples: OpenTelemetry (standard), Datadog, Grafana + Prometheus, Honeycomb.

---

## Minimum viable full-stack app

| Layer | Required? | Notes |
|---|---|---|
| Frontend | Yes | Users need UI |
| Backend | Yes | Business logic must live somewhere |
| Database | Yes | Durable state |
| Infrastructure/deploy | Yes | Something must run the app |
| Blob storage | Situational | Only if you handle files or media |
| Cache | Situational | Only when DB reads are a bottleneck |
| Message bus | Situational | Only if you need async / event-driven |
| API Gateway | Situational | nginx at small scale; dedicated at scale |
| Observability | Always | Logs at minimum, from day one |

---

## DDD perspective

The key insight: from a DDD standpoint, **database, cache, message bus, blob storage, and external HTTP clients are all the same thing** — infrastructure adapters. The domain layer doesn't know which ones exist.

```
Domain defines: UserRepository interface
Infrastructure provides: DrizzleUserRepository (SQL), CachedUserRepository (Redis + SQL)
Application wires: Layer composition at startup
```

The only architectural layers DDD cares about:

```
Presentation → Application → Domain ← Infrastructure
```

Everything from "cache" to "S3" to "Kafka" lives in the bottom-right box.

---

## Related

- [[domain-driven-design]]
- [[ddd-request-flow]]
- [[ddd-bounded-context]]
- [[ddd-aggregate-root]]
- [[ddd-domain-events]]
- [[ports-and-adapters]]

## References

- Evans, *Domain-Driven Design* (2003) — ch. 4, Layered Architecture
- Vernon, *Implementing Domain-Driven Design* (2013) — ch. 4
- Fowler, [PresentationDomainDataLayering](https://martinfowler.com/bliki/PresentationDomainDataLayering.html)
- Richardson, [Microservice Architecture](https://microservices.io/patterns/index.html)
