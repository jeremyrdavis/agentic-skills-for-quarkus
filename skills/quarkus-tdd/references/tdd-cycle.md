# Canonical Example — The Loop in Java / Quarkus: Cancelling an Order

Load this file when running the `tdd-cycle` loop in a Quarkus project for the
first time, or when a cycle has gone wrong (a test passed without red, a tier
was run at the wrong time). The rules are `tdd-test-list` and `tdd-cycle`;
comments cite their numbers. The class under test is a plain `Order` with a
status, a factory `place(...)`, and `markFulfilled()`, shown here as if
`cancel()` were being added for the first time.

Setup: `quarkus test` is running in a second terminal (mechanic 2). Mode is
`auto` — no pauses; commits at every green and after each refactoring
(`tdd-cycle` Rule 12).

## 0. The list — `tdd-test-list` Rules 1–5, mechanic 4

Behavior only, ZOMBIES walked, simplest forcing case first, exceptions last.
It lives at the bottom of the test class and shrinks as tests are written.

```java
package com.example.orders.domain;                    // plain-JVM tier (mechanic 1)

import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.*;

class OrderTest {

    // [TEST] Cancelling a placed order marks it cancelled                 <- O
    // [TEST] Cancelling a placed order raises OrderCancelledEvent         <- O + I
    // [TEST] Cancelling a fulfilled order is refused                      <- E
    // [TEST] A refused cancellation leaves the order fulfilled            <- E (integrity)
    // [TEST] Cancelling a cancelled order is refused                      <- E
    // [TEST] Placing an order still raises OrderPlacedEvent               <- still works (tdd-test-list Rule 4)
}
```

## 1. First cycle — `cancelling_a_placed_order_marks_it_cancelled`

**🔴 Write the test** (`tdd-cycle` Rules 1–2): one item, one test, assertion first.

```java
    @Test
    void cancelling_a_placed_order_marks_it_cancelled() {
        Order order = placedOrder();                               // a test-local factory around Order.place(...)

        order.cancel();

        assertThat(order.status()).isEqualTo(Order.Status.CANCELLED);
    }
```

Expected value derived, not run (Rule 3): a placed order that is cancelled
has status `CANCELLED` — the enum constant doesn't exist yet either.

**Predict (Rule 4):** compile failure — `cancel()` and `Status.CANCELLED` don't
exist. `quarkus test` reports the `javac` error (Rule 5, first step; mechanic 3).

**Make it compile, nothing more** (mechanic 3):

```java
    public enum Status { PLACED, FULFILLED, CANCELLED }

    public void cancel() {
        throw new UnsupportedOperationException();              // a stub, not logic
    }
```

**Predict:** assertion failure — `UnsupportedOperationException` thrown
before the assertion. Run: red. ✔ Second step of the red (Rule 5).

**🟢 Simplest change that passes** (Rule 7):

```java
    public void cancel() {
        this.status = Status.CANCELLED;
    }
```

**Predict:** passes; every earlier `OrderTest` passes (plain-JVM tier, every
step — mechanic 2). Run: green.

**Simplify (Rule 9):** one line, required by the failing test. Nothing to
delete. **Refactor (Rule 10):** `🧹 none` — the next test is easy to write.

```
git commit -am "feat: cancelling a placed order marks it cancelled"
```

Delete the first `[TEST]` line (mechanic 4).

## 2. Second cycle — `cancelling_a_fulfilled_order_is_refused`

Taking the list in order would make the event test next. The event needs
`pullPendingEvents()`, which exists; the refusal test reaches the salient
design decision — what "refused" means — sooner (`tdd-test-list` Rule 5), so
it goes first and the list is reordered, not skipped.

**🔴 Test:**

```java
    @Test
    void cancelling_a_fulfilled_order_is_refused() {
        Order order = placedOrder();
        order.markFulfilled();

        assertThatThrownBy(order::cancel)
            .isInstanceOf(OrderCancellationNotAllowedException.class);
    }
```

Interface decision made in the test — the refusal is an exception thrown by
`cancel()`, the project's convention for a rule violation — `tdd-cycle` Rule 11.

**Predict:** compile failure — the exception class doesn't exist. Add it:

```java
package com.example.orders.domain;

public class OrderCancellationNotAllowedException extends RuntimeException {
    public OrderCancellationNotAllowedException(OrderId id) {
        super("order " + id.value() + " cannot be cancelled");
    }
}
```

**Predict:** assertion failure — `cancel()` sets the status and throws
nothing. Run: red. ✔

**🟢 Green:**

```java
    public void cancel() {
        if (status != Status.PLACED)                               // (see cycle 3)
            throw new OrderCancellationNotAllowedException(new OrderId(id));
        this.status = Status.CANCELLED;
    }
```

Run: green, all of `OrderTest`. **Simplify (Rule 9):** "does a failing test
require `!= PLACED`?" — the failing test only requires *fulfilled* to be
refused. This is the moment to write `== FULFILLED`. The example keeps the
broader guard deliberately, to show what happens next.

`🧹 none`. `git commit -am "feat: cancelling a fulfilled order is refused"`.

## 3. Third cycle — a test that passes without red (`tdd-cycle` Rule 6)

```java
    @Test
    void cancelling_a_cancelled_order_is_refused() {
        Order order = placedOrder();
        order.cancel();

        assertThatThrownBy(order::cancel)
            .isInstanceOf(OrderCancellationNotAllowedException.class);
    }
```

**Predict:** assertion failure — nothing has been written for this case.
Run: **green.** Stop.

> passed without red: the `status != Status.PLACED` guard added in cycle 2
> covered the cancelled case before this test required it. Rule 9 should have
> narrowed it to `== FULFILLED`. Keeping the test as a regression test;
> tightening simplify on the remaining items.

```
git commit -am "test: cancelling a cancelled order is refused (passed without red: cycle-2 guard was broader than its test)"
```

The commit message carries the record so the drift is visible in the history.

## 4. Fourth cycle — the application tier, `@QuarkusTest`

`[TEST] Cancelling an order through the use case persists the cancelled state`
exercises `CancelOrderApplicationService` and the repository, which need the
container (mechanic 1): `@QuarkusTest`, Dev Services PostgreSQL, the real
Panache repository, `@TestTransaction` (mechanic 9). It runs at this
green and after refactorings, not at every step (mechanic 2).

```java
package com.example.orders.application;

import com.example.orders.domain.*;
import io.quarkus.test.TestTransaction;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

@QuarkusTest
class CancelOrderApplicationServiceTest {

    @Inject CancelOrderApplicationService cancelOrder;             // the real one — never mocked here (mechanic 7)
    @Inject OrderRepository orders;                                // domain interface → Panache implementation

    @Test
    @TestTransaction
    void cancelling_an_order_through_the_use_case_persists_the_cancelled_state() {
        Order order = placedOrder();
        orders.persist(order);

        OrderDTO result = cancelOrder.cancel(order.id());

        assertThat(result.status()).isEqualTo("CANCELLED");
        assertThat(orders.findById(order.id()).orElseThrow().status()).isEqualTo(Order.Status.CANCELLED);
    }

    // [TEST] Cancelling an unknown order throws OrderNotFoundException
    // [TEST] Cancelling an order hands OrderCancelledEvent to the publisher
}
```

Same loop: `javac` red (`CancelOrderApplicationService.cancel` doesn't exist),
stub throws, assertion red, then the simplest green —

```java
    @Transactional                                                 // the use case owns the transaction
    public OrderDTO cancel(OrderId id) {
        Order order = orders.findById(id).orElseThrow();           // the not-found case is a later list item
        order.cancel();
        orders.persist(order);
        return OrderDTO.of(order);
    }
```

`orElseThrow()` without the domain exception is the simplest green; the
`[TEST] Cancelling an unknown order…` item is what will force
`OrderNotFoundException` (`tdd-cycle` Rule 7). The event hand-off is likewise
not written until its test is red — writing `pullPendingEvents().forEach(events::fire)`
now would make that test pass without red.

```
git commit -am "feat: cancelling an order through the use case persists the cancelled state"
```

## 5. Refactoring with its own hat — `tdd-cycle` Rule 10

After the event item is green, `OrderTest` has three tests that build a placed
order and fulfil or cancel it. That is the third repetition, not the first:

```
🧹 extract fulfilledOrder() beside placedOrder() in OrderTest; no production change
```

Run (plain-JVM tier) → green. `git commit -am "refactor: fulfilledOrder() test factory"`.
Nothing further: the next item is easy to write.

## 6. Closing — `tdd-test-list` Rule 9

The list is empty. "What would I still not bet on?" — that a cancelled order's
`total()` is unchanged. One more item, one more cycle (it will likely pass
without red, and say so). Then done.

## What this demonstrates

- **Tiers and cost (mechanics 1–2):** plain-JVM cycles ran the suite at
  every step; the `@QuarkusTest` ran at its own green.
- **Two-step red in Java (mechanic 3; `tdd-cycle` Rule 5):** `javac` first,
  stub throwing `UnsupportedOperationException`, then the assertion.
- **Derived expectations and predictions (Rules 3–4):** every run was preceded
  by what would fail and why.
- **The list is alive (`tdd-test-list` Rules 5–7; mechanic 4):** reordered in
  cycle 2, shrinking at every green, appended at the end.
- **Passed without red is a stop (Rule 6):** named, recorded in the commit,
  traced to a simplify step that was skipped.
- **Interface decisions in the test (Rule 11):**
  the exception type and the transition method; the implementation behind
  them grew one guard at a time.
- **Commits at green and after refactoring (Rule 12; mechanic 12).**
