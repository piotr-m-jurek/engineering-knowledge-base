---
title: Ports and Adapters (Hexagonal Architecture)
type: pattern
created: 2026-08-05
tags: [architecture, hexagonal, ports-adapters, ddd]
---

# Ports and Adapters (Hexagonal Architecture)

## Intent

Isolate application core from external systems (DB, HTTP, message queues, UI) by defining explicit ports (interfaces) and adapters (implementations). The core depends on nothing outside itself.

## Structure

```
         HTTP        CLI       Tests
           ↓          ↓          ↓
      [ Driving Adapters (Primary) ]
               ↓
        [ Application Core ]
          (domain + app services)
               ↓
      [ Driven Adapters (Secondary) ]
           ↓          ↓          ↓
          DB        Email      Queue
```

- **Port** — an interface defined by the core (e.g. `UserRepository`, `EmailSender`)
- **Driving adapter** — calls into the core (HTTP controller, CLI command, test)
- **Driven adapter** — called by the core via a port (Drizzle repo, SendGrid mailer)

## Relation to DDD

DDD's repository pattern is a direct application of ports-and-adapters:
- `UserRepository` interface = port (in domain layer)
- `DrizzleUserRepository` = driven adapter (in infrastructure layer)

→ See [[ddd-repository-pattern]]

## Key benefit

The application core is testable without any infrastructure. Swap the driven adapter for an in-memory fake in tests.

## Related

- [[domain-driven-design]]
- [[ddd-repository-pattern]]

## References

- Cockburn, [Hexagonal Architecture](https://alistair.cockburn.us/hexagonal-architecture/) (2005)
