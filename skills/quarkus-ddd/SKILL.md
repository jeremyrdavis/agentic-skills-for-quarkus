---
name: quarkus-ddd
description: Quarkus mechanics for the ddd-* skills — use alongside them (never instead of them) when implementing DDD patterns in a Quarkus project. Supplies the Quarkus realization of decisions the ddd-* skills make: package layout and ArchUnit boundary enforcement, the extension set, transaction placement with jakarta.transaction.Transactional, Panache repositories implemented on the infrastructure repository class behind the hand-written domain interface, @Version compare-and-set and its mapping to 409, after-commit event delivery via CDI TransactionPhase observers or a transactional outbox to Kafka, and the test tiers (plain JUnit for domain, @QuarkusTest with Dev Services for application and api). Trigger on DDD vocabulary (aggregate, value object, repository, application service, domain event, bounded context) in a Quarkus/Jakarta codebase, and on PanacheEntity or @Entity on a domain type, a PanacheRepository used as the domain repository interface, QuarkusTransaction or @Transactional outside <bc>/application, Emitter or Event<> injected into <bc>/domain, or @QuarkusTest on a domain test. Defines no DDD concepts — layering is ddd-foundations, aggregate shape is ddd-aggregates, value objects are ddd-value-objects, service kinds are ddd-services, repository contract is ddd-persistence, event rules are ddd-domain-events. Excludes creating the project and managing extensions and versions (quarkus-app).
---

# Quarkus DDD Mechanics

The Quarkus tier under the `ddd-*` skills. Those skills decide **what** (layers, aggregate shape, repository contract, who owns the transaction, when events are delivered). This skill decides **how Quarkus does it**. If a rule here appears to conflict with a `ddd-*` rule, the `ddd-*` rule wins and this file has a bug.

**Load order.** `quarkus-app` first (the project exists, extensions are aligned), then the `ddd-*` skill for the concern at hand, then this file, then exactly one reference file from `references/` for the concern being implemented.

**Red Flags — STOP if you find yourself thinking:**

- "Panache is the Quarkus idiom, so the aggregate extends `PanacheEntity`." (The aggregate is a plain object; Panache belongs on the infrastructure side.)
- "A `PanacheRepository<OrderEntity>` *alone* is the repository." (The domain interface is hand-written and speaks domain types; the infrastructure class implements *both* it and `PanacheRepositoryBase`.)
- "Wrap just the `persist` call in `QuarkusTransaction.requiringNew()` and publish the event afterwards." (The transaction is the application-service method; delivery is after commit by mechanism, not by call ordering.)
- "`@Transactional` on the repository method, to be safe." (Repositories never demarcate — `ddd-persistence` Rule 12.)
- "A domain unit test needs `@QuarkusTest`." (If it does, the domain type has a framework dependency; fix the type.)
- "The value object gets its own `@Entity` with an `@Id`." (A value object has no identity — `@Embeddable` or columns on the mirror.)

---

## Stack defaults

| Concern | Default | Alternative (say why) |
| --- | --- | --- |
| Java | 25 (per `quarkus-app`) | — |
| HTTP | Quarkus REST + Jackson (`rest-jackson`) | Reactive routes |
| Persistence | Hibernate ORM with Panache (`hibernate-orm-panache`, `jdbc-postgresql`): the infrastructure repository class implements the domain interface **and** `PanacheRepositoryBase<Record, UUID>` | `getEntityManager()` for the one verb Panache lacks (`merge`); never a Panache-only repository as the domain contract |
| Events in-process | CDI `Event<T>` fired by the application service; handlers `@Observes(during = TransactionPhase.AFTER_SUCCESS)` | — |
| Events cross-service | Transactional outbox row → `@Scheduled` relay → Reactive Messaging `Emitter` (`messaging-kafka`, `scheduler`) | Debezium outbox extension |
| Tests | JUnit 5; `@QuarkusTest` + Dev Services; REST Assured | Testcontainers directly |
| Boundary enforcement | ArchUnit test per bounded context (`archunit-junit5`, test scope) | Maven multi-module, one module per layer |

Extension bundle for a DDD-shaped service — add via `quarkus ext add`, never by hand (`quarkus-app`):
`rest-jackson, hibernate-orm-panache, jdbc-postgresql`, plus `messaging-kafka, scheduler` when events leave the service. `hibernate-validator` is for api-layer request types only; value objects validate themselves (`ddd-value-objects`).

---

## Layout

Single Maven module; packages mirror the `ddd-foundations` layers exactly:

```
{base}.{bc}.domain          # aggregates, value objects, domain services, repository interfaces, events
{base}.{bc}.application     # application services, commands, DTOs
{base}.{bc}.infrastructure  # JPA mirrors, Panache-backed repository implementations, outbox, Kafka adapters
{base}.{bc}.api             # Quarkus REST resources, request/response types, exception mappers
```

No `persistence/` package and no `domain/services/` package: `ddd-foundations` has four layers and this skill does not add a fifth. Boundaries are enforced by an ArchUnit test (`references/architecture-tests.md`), not by review.

---

## Core mechanics

1. **Transaction = application-service method.** `@jakarta.transaction.Transactional` on the public method of a `*ApplicationService` in `application`. Nowhere else: not on resources, repositories, domain services, or aggregates. Prefer the annotation over `QuarkusTransaction`; use the programmatic API only for an explicit `requiresNew` case, and say why in a comment.
2. **CDI is the composition root.** `@ApplicationScoped` on application services, domain services, and repository implementations; constructor injection. Domain types other than domain services carry no CDI annotations.
3. **The mirror carries the framework.** `@Entity`, `@Table`, `@Id UUID`, `@Version long`, column mappings go on `OrderRecord` in `infrastructure`; fields private, with `static fromDomain(Order)` and `toDomain()` beside them. Value objects map to `@Embeddable` types or to columns — never to their own `@Entity` with an `@Id` (`ddd-value-objects` Rule 7).
4. **The infrastructure repository is a Panache repository behind the domain interface.** `class OrderRepository implements com.example.orders.domain.OrderRepository, PanacheRepositoryBase<OrderRecord, UUID>` (same simple name as the interface — `ddd-persistence` Rule 3). Domain methods are implemented with Panache verbs: `findByIdOptional`, `deleteById`, `findAll(Sort)…page(...).list()`, `count()`, `update("… where id = ?1", id)`. `toDomain()` calls the aggregate's `rehydrate(...)`. The two `findById`/`persist` overloads (domain-typed vs Panache-typed) coexist on the class; only the domain-typed ones are on the interface.
5. **Upsert is `merge` + `flush`.** `persist(Order)` maps to a detached mirror carrying the aggregate's `version`, then `getEntityManager().merge(record)` (Panache has no merge verb) and `flush()`. Flushing inside the repository makes the compare-and-set fail *here*, in the application service's call, rather than wrapped at commit.
6. **Lock conflict → 409.** Catch `jakarta.persistence.OptimisticLockException` in the repository implementation and rethrow the bounded context's domain `ConcurrencyConflictException` (`ddd-persistence` Rule 7); an `@ServerExceptionMapper` in `api` maps it to `409 Conflict`. The domain exception is the only thing that crosses layers.
7. **Events deliver after commit, by mechanism.** The application service drains `pullPendingEvents()` and fires each through CDI `Event<DomainEvent>` inside its transaction. In-process handlers observe with `during = TransactionPhase.AFTER_SUCCESS` and delegate to an application service with its own `@Transactional` — right timing, but in-memory: a crash after commit loses the event, so use it only where that is tolerable. Cross-service delivery, and any event that must not be lost, writes an outbox row in the same transaction; a relay publishes via `Emitter`. Never `Emitter.send(...)` from `application`.
8. **Kafka channel per bounded context**, configured in `application.properties` (`mp.messaging.outgoing.{bc}.*`). Topic payloads are integration events (`ddd-domain-events` Rule 9), mapped from domain events in `infrastructure` — the domain event type is not the wire format.
9. **Test tiers follow the layers.** `domain`: plain JUnit 5, no Quarkus. `application`: `@QuarkusTest` with Dev Services PostgreSQL and the real repository implementation. `api`: `@QuarkusTest` + REST Assured. Test names are snake_case sentences (`placing_an_order_raises_order_placed`).
10. **Logging lives in `api` and `infrastructure`.** `io.quarkus.logging.Log` at DEBUG at adapter boundaries; nothing in `domain` or `application`.

---

## Reference files

Load exactly one, for the concern being implemented. Each exercises the numbered Core Rules of the matching `ddd-*` skill and cites them.

| Implementing… | Read | Pairs with |
| --- | --- | --- |
| A new bounded context's package tree, resource, application service, repository wiring | `references/foundations.md` | `ddd-foundations` |
| An application-service / domain-service split, transaction and event hand-off | `references/services.md` | `ddd-services` |
| A repository, mirror, `@Version`, 409 mapping, use-case-optimal query | `references/persistence.md` | `ddd-persistence` |
| Event delivery: `AFTER_SUCCESS` observers, outbox, Kafka config, integration events | `references/domain-events.md` | `ddd-domain-events` |
| The ArchUnit layer test for a bounded context | `references/architecture-tests.md` | `ddd-foundations` |

Aggregate and value-object code is stack-neutral Java and stays in `ddd-aggregates/references/canonical-example-java.md` and `ddd-value-objects/references/canonical-example-java.md`.

---

## Quick Reference

- [ ] `@Transactional` appears only on `*ApplicationService` methods in `application`.
- [ ] No `jakarta.persistence`, `io.quarkus`, `org.eclipse.microprofile`, `jakarta.ws.rs`, or `jakarta.transaction` import under `domain` (ArchUnit enforces).
- [ ] The domain repository interface is hand-written; `PanacheRepositoryBase` appears only on the infrastructure class that implements it, under the same simple name.
- [ ] Value objects persist as `@Embeddable` or columns; no `@Entity` with its own `@Id` for a value object.
- [ ] `persist` = `merge` + `flush`; `OptimisticLockException` is translated in `infrastructure`; `api` maps the domain exception to 409.
- [ ] Events go through `Event<T>` with `AFTER_SUCCESS` observers or an outbox; no `Emitter.send` in `application`.
- [ ] Domain tests have no `@QuarkusTest`.

### Companion skills

- **`ddd-foundations`, `ddd-aggregates`, `ddd-value-objects`, `ddd-services`, `ddd-persistence`, `ddd-domain-events`** — the design rules this skill implements. Written against `ddd-skills` 1.x; Core Rule numbers cited in `references/` refer to that version.
- **`quarkus-app`** — creating the project, choosing the platform version, adding extensions.
