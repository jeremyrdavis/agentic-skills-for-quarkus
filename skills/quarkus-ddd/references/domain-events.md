# Canonical Example — After-Commit Delivery, Outbox, and Integration Events (Java / Quarkus)

Load this file when wiring event delivery in a Quarkus project. The rules —
what an event is, what it carries, when it is delivered — are in
`ddd-domain-events/SKILL.md`; comments cite its rule numbers. The aggregate
raises events (`ddd-aggregates` Rule 12) and the application service drains
them (`ddd-services` Rule 11); this file starts where the drain ends.

## 1. The hand-off — application service, inside the transaction

```java
// com.example.orders.application.CancelOrderApplicationService — the relevant lines
@Transactional
public OrderDTO cancel(OrderId id) {
    Order order = orders.findById(id).orElseThrow(() -> new OrderNotFoundException(id));
    order.cancel();                                              // raises OrderCancelledEvent inside the aggregate
    orders.persist(order);
    order.pullPendingEvents().forEach(events::fire);             // Event<DomainEvent> — drain and hand off
    return OrderDTO.of(order);
}
```

`events.fire(...)` runs inside the transaction. Nothing is delivered yet: CDI
holds transactional observers until the phase they declared (Rule 6).

## 2. In-process handler — `AFTER_SUCCESS`, own transaction, same bounded context

A domain event is observed only inside the context that raised it (Rule 9);
here a second aggregate of the orders context, `Reservation`, reacts. Another
context never imports `orders.domain` — it consumes the integration event in
section 4.

```java
package com.example.orders.infrastructure;

import com.example.orders.application.ReleaseReservationApplicationService;
import com.example.orders.domain.OrderCancelledEvent;             // same bounded context
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.Observes;
import jakarta.enterprise.event.TransactionPhase;

@ApplicationScoped
public class OrderCancelledObserver {                            // inbound adapter — infrastructure

    private final ReleaseReservationApplicationService release;

    OrderCancelledObserver(ReleaseReservationApplicationService release) { this.release = release; }

    void on(@Observes(during = TransactionPhase.AFTER_SUCCESS) OrderCancelledEvent event) {
        release.forOrder(event.orderId(), event.eventId());       // its own transaction (Rule 7)
    }
}
```

```java
// com.example.orders.application.ReleaseReservationApplicationService — the relevant lines
@Transactional(Transactional.TxType.REQUIRES_NEW)                // not REQUIRED — see below
public void forOrder(OrderId orderId, UUID eventId) {
    if (handled.contains(eventId)) return;                       // idempotent (Rule 8)
    Reservation reservation = reservations.findByOrder(orderId).orElseThrow(...);
    reservation.release();                                       // the second aggregate, its own transaction
    reservations.persist(reservation);
    handled.record(eventId);
}
```

The observer carries no `@Transactional` of its own: it is delivered after the
publisher's commit and *calls an application service*, whose method opens the
handler's transaction with `REQUIRES_NEW`. During `AFTER_SUCCESS` the completed
publisher transaction is still associated with the thread, so a plain
`@Transactional` (`REQUIRED`) would join that finished transaction instead of
starting one. The handler dedupes on `eventId` (Rule 8) — `AFTER_SUCCESS`
delivery is in-memory, so a crash between commit and observer loses the event
unless the handler can be re-driven; use the outbox below when that matters.

## 3. Cross-service — transactional outbox to Kafka

**Outbox row, written in the publisher's transaction** (an `IN_PROGRESS`
observer runs synchronously inside the firing transaction):

```java
package com.example.orders.infrastructure;

@Entity @Table(name = "orders_outbox")
public class OutboxRecord {
    @Id UUID id;                     // = eventId — the consumer's idempotency key
    String type;                     // "OrderCancelled", version in the payload
    @Column(columnDefinition = "text")
    String payload;                  // integration-event JSON — never the default varchar(255)
    Instant occurredAt;              // when it happened — metadata, not the relay's ordering key
    long seq;                        // DB identity/sequence column — the relay orders on this
    Instant publishedAt;             // null until the relay sends it
    // constructors/factory elided
}

@ApplicationScoped
public class OutboxWriter {
    private final OutboxRepository outbox;            // PanacheRepositoryBase<OutboxRecord, UUID>
    private final IntegrationEventMapper mapper;      // domain event → integration event (Rule 9)
    OutboxWriter(OutboxRepository outbox, IntegrationEventMapper mapper) { this.outbox = outbox; this.mapper = mapper; }

    void on(@Observes(during = TransactionPhase.IN_PROGRESS) DomainEvent event) {
        outbox.persist(OutboxRecord.of(mapper.toIntegrationEvent(event)));   // same transaction as the aggregate
    }
}
```

**Relay** — the only class that touches the broker:

```java
package com.example.orders.infrastructure;

import io.quarkus.scheduler.Scheduled;
import org.eclipse.microprofile.reactive.messaging.Channel;
import org.eclipse.microprofile.reactive.messaging.Emitter;

@ApplicationScoped
public class OutboxRelay {
    @Channel("orders") Emitter<String> orders;        // channel = bounded context (quarkus-ddd mechanic 8)
    private final OutboxRepository outbox;
    OutboxRelay(OutboxRepository outbox) { this.outbox = outbox; }

    @Scheduled(every = "1s")
    @Transactional                                    // infrastructure job, not a use case — the one tolerated exception
    void relay() {
        for (OutboxRecord r : outbox.unpublished(100)) {   // "publishedAt is null order by seq"
            orders.send(r.payload).toCompletableFuture().join();   // wait for the broker's ack: a failure throws,
            r.publishedAt = Instant.now();                         // publishedAt rolls back, the next tick retries
            // at-least-once: a crash after the ack and before commit re-sends the row — consumers dedupe on eventId
        }
    }
}
```

`application.properties` (carried over from the previous `quarkus-ddd`):

```properties
mp.messaging.outgoing.orders.connector=smallrye-kafka
mp.messaging.outgoing.orders.topic=orders
mp.messaging.outgoing.orders.value.serializer=org.apache.kafka.common.serialization.StringSerializer
```

Alternative: the Debezium Quarkus outbox extension writes the same table and
lets Debezium tail it, removing the relay. Same contract, no scheduler.

## 4. The integration event — a published, versioned contract (Rule 9)

```java
package com.example.orders.infrastructure.contracts;              // what other services depend on

public record OrderCancelledV1(UUID eventId, Instant occurredAt,
                               UUID orderId, UUID customerId, String reason) {}
```

Mapped from `OrderCancelledEvent` by `IntegrationEventMapper` in
`infrastructure`. The domain event is free to change with the model; `V1` is
not. The consumer in another bounded context deserializes `OrderCancelledV1`,
dedupes on `eventId`, and calls *its* application service — it never imports
`com.example.orders.domain`:

```java
package com.example.inventory.infrastructure;                     // the other context's inbound adapter

@ApplicationScoped
public class OrdersConsumer {
    private final ReleaseStockApplicationService release;         // inventory's own use case, own @Transactional
    OrdersConsumer(ReleaseStockApplicationService release) { this.release = release; }

    @Incoming("orders")                                           // mp.messaging.incoming.orders.* in this service
    @Blocking                                                     // the use case blocks on the database
    public void on(OrderCancelledV1 event) {                      // Jackson deserializer for the contract type
        release.forOrder(event.orderId(), event.eventId());       // dedupes on eventId inside
    }
}
```

## What this demonstrates

- **After-commit by mechanism (Rule 6):** `AFTER_SUCCESS` observers in-process
  (right timing, best-effort — in-memory); outbox + relay wherever the event
  must not be lost, and always across services. Neither relies on "publish
  after the call returns".
- **Own transaction per handler (Rule 7):** the observer delegates to an
  application service running `REQUIRES_NEW`; the relay's `@Transactional` is
  infrastructure plumbing.
- **Idempotent handlers (Rule 8):** `eventId` is the outbox key and the
  consumer's dedupe key; at-least-once is the assumption.
- **Domain vs integration events (Rule 9):** the Kafka payload is `V1`, mapped
  in `infrastructure`; the domain type never leaves the service, and the other
  context's consumer depends on the `V1` contract only.
- **No broker in domain or application (Rule 10):** `Emitter` appears in one
  infrastructure class.

## Quarkus idioms

- Quarkus (ArC) honours `TransactionPhase` on observers when a JTA transaction
  is active; outside one, `AFTER_SUCCESS` observers fire immediately — keep the
  `@Transactional` on the application service.
- Use `Emitter<String>` with a JSON payload, or a typed `Emitter<OrderCancelledV1>`
  with the Jackson serializer, never `Emitter<OrderCancelledEvent>`.
- `Emitter.send` returns a `CompletionStage` that completes on the broker's
  ack; the relay joins it before marking the row published, so a send failure
  rolls `publishedAt` back and the row is retried.
- Dev Services starts Kafka for `@QuarkusTest`; test the relay against it and the
  observers with an in-memory `InMemoryConnector` when the broker is noise.
- The tier split above, the runner, and test doubles are `quarkus-tdd`'s (`references/testing.md` there); this file only fixes which layer each class is in.
