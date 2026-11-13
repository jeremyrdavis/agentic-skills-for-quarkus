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

## 2. In-process handler — `AFTER_SUCCESS`, own transaction

```java
package com.example.inventory.infrastructure;

import com.example.inventory.application.ReleaseReservationApplicationService;
import com.example.orders.domain.OrderCancelledEvent;             // the published domain event type
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.Observes;
import jakarta.enterprise.event.TransactionPhase;

@ApplicationScoped
public class OrderCancelledObserver {                            // inbound adapter — infrastructure

    private final ReleaseReservationApplicationService release;

    OrderCancelledObserver(ReleaseReservationApplicationService release) { this.release = release; }

    void on(@Observes(during = TransactionPhase.AFTER_SUCCESS) OrderCancelledEvent event) {
        release.forOrder(event.orderId(), event.eventId());       // its own @Transactional use case (Rule 7)
    }
}
```

The observer carries no `@Transactional` of its own: it is delivered after the
publisher's commit and *calls an application service*, which opens the handler's
transaction. The handler dedupes on `eventId` (Rule 8) — `AFTER_SUCCESS`
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
    String payload;                  // integration-event JSON
    Instant occurredAt;
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
        for (OutboxRecord r : outbox.unpublished(100)) {   // "publishedAt is null order by occurredAt"
            orders.send(r.payload);                        // at-least-once: a crash here re-sends on the next tick
            r.publishedAt = Instant.now();
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
not. A consumer in another bounded context deserializes `OrderCancelledV1`,
dedupes on `eventId`, and calls *its* application service — it never imports
`com.example.orders.domain`.

## What this demonstrates

- **After-commit by mechanism (Rule 6):** `AFTER_SUCCESS` observers in-process;
  outbox + relay across services. Neither relies on "publish after the call
  returns".
- **Own transaction per handler (Rule 7):** the observer delegates to an
  application service; the relay's `@Transactional` is infrastructure plumbing.
- **Idempotent handlers (Rule 8):** `eventId` is the outbox key and the
  consumer's dedupe key; at-least-once is the assumption.
- **Domain vs integration events (Rule 9):** the Kafka payload is `V1`, mapped
  in `infrastructure`; the domain type never leaves the service.
- **No broker in domain or application (Rule 10):** `Emitter` appears in one
  infrastructure class.

## Quarkus idioms

- Quarkus (ArC) honours `TransactionPhase` on observers when a JTA transaction
  is active; outside one, `AFTER_SUCCESS` observers fire immediately — keep the
  `@Transactional` on the application service.
- Use `Emitter<String>` with a JSON payload, or a typed `Emitter<OrderCancelledV1>`
  with the Jackson serializer, never `Emitter<OrderCancelledEvent>`.
- Dev Services starts Kafka for `@QuarkusTest`; test the relay against it and the
  observers with an in-memory `InMemoryConnector` when the broker is noise.
