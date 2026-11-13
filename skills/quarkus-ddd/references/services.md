# Canonical Example — The Refund Triangle (Java / Quarkus)

Load this file when creating a new service or splitting a mixed one in a
Quarkus project. For placement and review decisions, the Core Rules and Quick
Reference tables in `ddd-services/SKILL.md` are sufficient on their own.

The "process refund" use case: three classes, three roles. The `Order`
aggregate is the one from `ddd-aggregates`' canonical example, extended with a
`refund` transition. Comments cite rule numbers.

## 1. The aggregate's domain method (in `<bc>/domain`; full design in `ddd-aggregates`)

```java
// com.example.orders.domain.Order — sketch of the relevant method only
public void refund(Money amount) {
    if (status != Status.PLACED && status != Status.FULFILLED) {
        throw new OrderRefundNotAllowedException(new OrderId(id));
    }
    if (amount.amount().compareTo(total.amount()) > 0) {
        throw new IllegalArgumentException("refund amount exceeds order total");
    }
    this.status = Status.REFUNDED;
    this.refundedAmount = amount;
    this.pendingEvents.add(new OrderRefundedEvent(new OrderId(id), amount));
}
```

Enforces the *invariant* (Rule 7): the order is in a refundable state, the
amount doesn't exceed the total. The aggregate **does not compute** the amount;
it accepts the value and verifies the resulting state is legal.

## 2. The domain service (in `<bc>/domain`)

```java
package com.example.orders.domain;

import jakarta.enterprise.context.ApplicationScoped;
import java.time.Instant;
import java.util.List;

@ApplicationScoped        // CDI-managed instance (Rule 9) — but no @Transactional, no injected I/O
public class RefundCalculatorDomainService {

    private final RefundPolicy policy;      // configured DOMAIN value — allowed (Rule 5)

    RefundCalculatorDomainService(RefundPolicy policy) { this.policy = policy; }

    public Money compute(Order order, Instant now) {          // pure: domain in, domain out (Rule 4)
        Money base = order.total();
        Money prorated = applyTimeProration(base, order.placedAt(), now);
        Money afterRestocking = applyRestockingFees(prorated, order.lines());
        return roundToCurrencyFractionDigits(afterRestocking);
        // ... 80 lines of pure rules: prorated within policy.windowDays(),
        // restocking fees on opened SKUs, HALF_EVEN currency rounding, etc.
    }

    private Money applyTimeProration(Money base, Instant placedAt, Instant now) { /* ... */ }
    private Money applyRestockingFees(Money prorated, List<OrderLine> lines)    { /* ... */ }
    private Money roundToCurrencyFractionDigits(Money m)                        { /* ... */ }
}
```

Pure computation over an `Order` and an `Instant`. No repository, no
`@Transactional`, no `Event<>` (Rule 5). Unit-testable on the plain JVM with a
real `Order` and a fixed `Instant` — no framework bootstrap.

## 3. The application service (in `<bc>/application`)

```java
package com.example.orders.application;

import com.example.orders.domain.*;     // domain types and interfaces — never the impls
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.Event;
import jakarta.transaction.Transactional;
import java.time.Clock;

@ApplicationScoped
public class ProcessRefundApplicationService {

    private final OrderRepository orders;                     // domain interface (Rule 3)
    private final RefundCalculatorDomainService calculator;
    private final Event<DomainEvent> events;                  // the event publisher
    private final Clock clock;                                // injected so tests can fix time

    ProcessRefundApplicationService(OrderRepository orders,
                                    RefundCalculatorDomainService calculator,
                                    Event<DomainEvent> events,
                                    Clock clock) {
        this.orders = orders;
        this.calculator = calculator;
        this.events = events;
        this.clock = clock;
    }

    @Transactional                                            // the transaction boundary — here, only here (Rule 8)
    public RefundDTO refund(OrderId orderId) {
        Order order = orders.findById(orderId)
                .orElseThrow(() -> new OrderNotFoundException(orderId));

        Money amount = calculator.compute(order, clock.instant());  // domain-service call (Rule 7)
        order.refund(amount);                                        // aggregate transition
        orders.persist(order);                                       // persist via the interface

        order.pullPendingEvents().forEach(events::fire);             // drain and hand off (Rule 11)

        return RefundDTO.of(order, amount);                          // DTO out — never the aggregate (Rule 10)
    }
}
```

Load → compute → mutate → persist → drain → map. The application service is the
*only* class in the triangle that mentions `@Transactional`, `Event<>`, or the
repository. There is no rule computation hidden as a private method — that
computation lives in the domain service.

**After-commit delivery (Rule 11):** `events.fire(...)` runs inside the
transaction, but observers declared with
`@Observes(during = TransactionPhase.AFTER_SUCCESS)` receive the events only
after commit. The rule lives in `ddd-domain-events`; the Quarkus observer and
outbox mechanics are in `references/domain-events.md` beside this file.

## Why this triangle works

The three classes have different dependency signatures, and the package each
lives in enforces them (per `ddd-foundations` and the ArchUnit rules). The
aggregate depends on nothing beyond domain types. The domain service has no
I/O dependencies. The application service is the layer where framework
concerns aggregate. If you find yourself adding `@Transactional` or
`@Inject OrderRepository` to a class in `domain/`, you've discovered the class
is actually an application service — move and rename it.

## Java idioms

- CDI is the composition root: both services are `@ApplicationScoped`;
  constructor injection keeps them testable without the container.
- The configured `RefundPolicy` is produced by a CDI producer (or
  `@ConfigProperty`-backed factory) in boot/infrastructure — the domain service
  receives a plain domain value, not a config API.
- Testing split: the domain service tests are pure JVM (no `@QuarkusTest`);
  the application service tests wire fakes for `OrderRepository` and the event
  publisher, or run under `@QuarkusTest` with Dev Services PostgreSQL and the
  real Panache-backed repository (`references/persistence.md`).
- The tier split above, the runner, and test doubles are `quarkus-tdd`'s (`references/testing.md` there); this file only fixes which layer each class is in.
- Events: past tense, `*Event` suffix (`OrderRefundedEvent`), per
  `ddd-foundations` and `ddd-domain-events`.
