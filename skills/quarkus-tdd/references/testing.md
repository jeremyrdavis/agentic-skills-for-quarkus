# Quarkus Test Tooling — Doubles, REST Assured, Dev Services, Integration Companions

Load this file when writing a `@QuarkusTest`, choosing a mocking mechanism, hitting
an endpoint with REST Assured, or adding the integration-test companion. The
mechanics are numbered in `quarkus-tdd/SKILL.md` (6–11); this file is the worked
form. The loop itself is `tdd-cycle`; the test's place in it is unchanged by
anything here.

**The principle behind every example:** if you mock the system under test, you
are not testing — you are scaffolding. A class named `XTest` that mocks `X`
asserts only what Mockito returned; the rules inside `X` go untested. Mock
*collaborators* (another application service, an external client, a clock);
never the class the test is named after, never a plain domain object.

## 1. Resource test — HTTP tier, `@InjectMock` on the service behind the resource

The resource's job is request parsing, status codes, and the exception mapper.
The use case behind it has its own tests; here it is a collaborator, so it is
mocked (mechanic 6), and the mock variable is named after the type.

```java
package com.example.orders.api;

import com.example.orders.application.CancelOrderApplicationService;
import com.example.orders.application.OrderDTO;
import com.example.orders.domain.OrderId;
import com.example.orders.domain.OrderNotFoundException;
import io.quarkus.test.InjectMock;
import io.quarkus.test.junit.QuarkusTest;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.mockito.Mockito;

import java.math.BigDecimal;
import java.util.UUID;

import static io.restassured.RestAssured.given;
import static org.hamcrest.Matchers.equalTo;

@QuarkusTest
class OrdersResourceTest {

    static final UUID KNOWN = UUID.fromString("0b5f6a3e-8d1c-4b7a-9c2e-1f3d5a7b9c0d");
    static final UUID UNKNOWN = UUID.fromString("ffffffff-ffff-ffff-ffff-ffffffffffff");

    @InjectMock
    CancelOrderApplicationService cancelOrder;                      // collaborator — mocked

    @BeforeEach
    void stubUseCase() {
        Mockito.when(cancelOrder.cancel(new OrderId(KNOWN)))
               .thenReturn(new OrderDTO(KNOWN, UUID.randomUUID(), "CANCELLED", new BigDecimal("19.98"), "USD"));
        Mockito.when(cancelOrder.cancel(new OrderId(UNKNOWN)))
               .thenThrow(new OrderNotFoundException(new OrderId(UNKNOWN)));
    }

    @Test
    void cancelling_a_known_order_returns_the_cancelled_order() {
        given()
            .when().post("/orders/{id}/cancel", KNOWN)              // path matches @Path on OrdersResource
            .then()
                .statusCode(200)                                    // status first
                .body("status", equalTo("CANCELLED"));              // then JSON paths, never whole-body strings
    }

    @Test
    void cancelling_an_unknown_order_returns_404() {
        given()
            .when().post("/orders/{id}/cancel", UNKNOWN)
            .then()
                .statusCode(404);                                   // the exception mapper's job
    }
}
```

Per-argument behavior (`KNOWN` succeeds, `UNKNOWN` throws) is ordinary
`Mockito.when(...)` stubbing in `@BeforeEach`. It never needs `QuarkusMock`.

## 2. Integration-test companion — black-box tests only

A `@QuarkusIntegrationTest` runs the packaged artifact in a separate process.
Nothing from the test JVM reaches it: `@Inject` fields stay null, `@InjectMock`
and `QuarkusMock` install nothing, `@TestTransaction` has no transaction to
join. So `OrdersResourceTest` above has **no** companion — inheriting it would
call `stubUseCase()` on a null mock. The companion belongs to a black-box test:
REST Assured against real endpoints, Dev Services behind them, state set up
through the API itself.

```java
package com.example.orders.api;

import io.quarkus.test.junit.QuarkusTest;
import org.junit.jupiter.api.Test;

import static io.restassured.RestAssured.given;
import static org.hamcrest.Matchers.equalTo;

@QuarkusTest
class OrdersFlowTest {                                                  // black-box: no @Inject, no mocks

    static final String PLACE_ORDER = """
        {"customerId": "c-1", "lines": [{"sku": "A", "quantity": 1, "unitPrice": 10.00}]}
        """;

    @Test
    void an_order_placed_through_the_api_can_be_cancelled() {
        String id = given().contentType("application/json").body(PLACE_ORDER)
            .when().post("/orders")
            .then().statusCode(201).extract().path("id");

        given().when().post("/orders/{id}/cancel", id)
            .then().statusCode(200).body("status", equalTo("CANCELLED"));
    }
}
```

```java
package com.example.orders.api;

import io.quarkus.test.junit.QuarkusIntegrationTest;

@QuarkusIntegrationTest
class OrdersFlowIT extends OrdersFlowTest {
    // Same tests, run against the packaged artifact. Nothing to add.
}
```

One line of value: inheritance — but only when the parent is black-box. On
Maven, check `pom.xml` for `<skipITs>true</skipITs>`; if set, `./mvnw verify`
skips these, so run `./mvnw verify -DskipITs=false` before a PR. On Gradle, ITs
run only through `./gradlew quarkusIntTest`.

## 3. Repository test — container tier, real Dev Services PostgreSQL

The mapping, the lock column, and the atomic update are what this tier is
for, so nothing is mocked. The test injects the repository **interface**; CDI
resolves it to the Panache implementation.

```java
package com.example.orders.infrastructure;

import com.example.orders.domain.*;
import io.quarkus.test.TestTransaction;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

import java.math.BigDecimal;
import java.time.Instant;
import java.util.Currency;
import java.util.List;
import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;

@QuarkusTest
class OrderRepositoryTest {

    @Inject
    OrderRepository orders;                                          // com.example.orders.domain.OrderRepository

    @Test
    @TestTransaction                                                 // rolled back after the test — no cleanup
    void persisting_then_loading_returns_an_equal_order() {
        Money unit = new Money(new BigDecimal("9.99"), Currency.getInstance("USD"));
        OrderId id = new OrderId(UUID.randomUUID());
        Order order = Order.place(id, new CustomerId(UUID.randomUUID()),
                                  List.of(new OrderLine("sku-1", 2, unit)),
                                  Instant.parse("2026-11-13T10:00:00Z"));

        orders.persist(order);
        Order found = orders.findById(id).orElseThrow();

        assertThat(found.status()).isEqualTo(Order.Status.PLACED);
        assertThat(found.total()).isEqualTo(new Money(new BigDecimal("19.98"), Currency.getInstance("USD")));
        assertThat(found.version()).isEqualTo(0L);                    // fresh row; the lock cursor starts at 0
    }
}
```

Dev Services starts PostgreSQL for the test JVM; a manual `@Testcontainers`
container beside it means two databases and flaky tests (mechanic 9).

## 4. `QuarkusMock` — the three cases, and the shape that genuinely needs it

`@InjectMock` is the default (mechanic 6). Reach for
`QuarkusMock.installMockForType` only when:

- **(a) you must construct the double yourself** — a hand-rolled stateful fake,
  a `Clock`, a fixture too involved for field injection; install in
  `@BeforeAll` (the class needs `@TestInstance(Lifecycle.PER_CLASS)` for a
  non-static one) or `@BeforeEach`;
- **(b) a normal-scoped bean must be replaced for every injection point**, not
  only the field in the test class;
- **(c) a test method needs a different whole instance** — then
  `QuarkusMock.installMockForInstance(perTest, bean)`, where `bean` is the
  `@Inject`ed normal-scoped bean (its CDI client proxy — a raw object you built
  yourself is not accepted). It overrides for that method only.

Only normal-scoped beans (`@ApplicationScoped`, `@RequestScoped`) can be
replaced; `@Singleton` cannot. Every mechanism here — `installMockForType`,
`installMockForInstance`, and `@InjectMock`, which is built on them — replaces
the bean globally for the running app, so `@QuarkusTest` classes that mock
beans must not run under `junit.jupiter.execution.parallel.enabled`. Mocking
is not an isolation mechanism; disable parallel execution for those classes.

The legitimate case: a `Clock` fixed to one time by default, swapped to a
different time for the one test that crosses an expiry. Nothing is stubbed
per call; the discriminator is the whole object.

```java
package com.example.orders.api;

import io.quarkus.test.junit.QuarkusMock;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.BeforeAll;
import org.junit.jupiter.api.Test;

import java.time.Clock;
import java.time.Instant;
import java.time.ZoneOffset;
import java.util.UUID;

import static io.restassured.RestAssured.given;

@QuarkusTest
class OrderExpiryResourceTest {

    static final UUID KNOWN = UUID.fromString("0b5f6a3e-8d1c-4b7a-9c2e-1f3d5a7b9c0d");

    @Inject Clock clock;                                             // the @ApplicationScoped Clock bean's client proxy

    @BeforeAll
    static void installDefault() {
        Clock defaultClock = Clock.fixed(Instant.parse("2026-11-13T10:00:00Z"), ZoneOffset.UTC);
        QuarkusMock.installMockForType(defaultClock, Clock.class);   // case (a): a real instance, built here
    }

    @Test
    void an_order_is_active_at_the_default_time() {
        given().when().get("/orders/{id}", KNOWN).then().statusCode(200);
    }

    @Test
    void an_order_has_expired_a_year_later() {
        Clock yearLater = Clock.fixed(Instant.parse("2027-11-13T10:00:00Z"), ZoneOffset.UTC);
        QuarkusMock.installMockForInstance(yearLater, clock);        // case (c): this method only; second arg is the injected proxy

        given().when().get("/orders/{id}", KNOWN).then().statusCode(410);
    }
}
```

The `Clock` producer must be `@ApplicationScoped`, not `@Singleton`, for
either call to work. If the per-test instance is a `Mockito.mock(...)` with
`when(...)` calls keyed on argument values, it is per-call stubbing wearing
the wrong tool — go back to `@InjectMock`.

## Anti-patterns

| Don't | Why it's wrong | Fix |
| --- | --- | --- |
| `@InjectMock CancelOrderApplicationService cancelOrder;` *and* `QuarkusMock.installMockForType(mock, CancelOrderApplicationService.class)` for the same bean | Two mechanisms install one mock; one always wins and the other is dead code that misleads the next reader. | Pick `@InjectMock`, stub in `@BeforeEach`, delete the `installMockForType` line (mechanic 6). |
| Per-test `installMockForInstance` to vary return values by argument | That is per-call stubbing, which `Mockito.when(...)` in `@BeforeEach` already does. | `@InjectMock`; reserve `installMockForInstance` for a wholesale instance swap such as a `Clock`. |
| `@QuarkusTest class CancelOrderApplicationServiceTest` with `@InjectMock CancelOrderApplicationService` | The test is named after the class it mocks; it asserts its own stubs. | Exercise the real service against Dev Services or a hand-rolled in-memory repository fake (mechanic 7). |
| Mocking `Order`, `Money`, or `OrderLine` | The rules inside the object are the thing under test; a mock bypasses them. | Real instances through `Order.place(...)` (mechanic 7). |
| `CancelOrderApplicationService mockRepository = Mockito.mock(...)` | The variable name contradicts the type; the layer confusion the names exist to prevent. | Name the variable after the type: `cancelOrder`. |
| Test hits `/api/orders` while the resource declares `@Path("/orders")` | The test asserts a route that doesn't exist; its failure is uninformative. | Match the resource's `@Path` (mechanic 8). |
| A `@QuarkusTest` asserting that a field was persisted by Hibernate or serialized by Jackson | Tests the framework, not your code. | Assert the behavior your service exposes (mechanic 11). |
| An `*IT` that copies the test methods instead of `extends *Test` | Duplicates every method; the two drift. | `class FooIT extends FooTest {}` (mechanic 10). |
| `*IT extends *Test` where the test uses `@Inject`, `@InjectMock`, or `@TestTransaction` | The IT runs the app in another process; injected fields are null and mocks install nothing. | Companions for black-box tests only; the mocked test stays a `@QuarkusTest` (mechanic 10). |
| `installMockForInstance(mock, Clock.fixed(...))` | The second argument must be the injected bean's client proxy, not an object you built. | `@Inject Clock clock;` then `installMockForInstance(mock, clock)` (mechanic 6). |
| Manual `@Testcontainers` PostgreSQL beside Dev Services | Two databases compete; slow and flaky. | Let Dev Services manage it (mechanic 9). |
| `body(is("{...}"))` on a JSON response | Whole-body string compare; brittle to whitespace and field order. | JSON paths: `body("status", equalTo("CANCELLED"))` (mechanic 8). |

## Excuse / Reality

| Excuse | Reality |
| --- | --- |
| "A senior reviewer told me to use both `@InjectMock` and `QuarkusMock.installMockForType` — that's authority." | Cargo-culted defenses against an unspecified race aren't authority; they're folklore. If a CDI race exists, name it and file the bug. Otherwise: one source of truth per mock. |
| "Belt-and-braces never hurts — install the mock both ways." | They install the same mock through different mechanisms. One always wins; the other is dead code the next author has to decode. Tests are read more than written. |
| "The dynamic-mocking example shows three test methods with different behaviors, so my three-argument-cases scenario fits it." | That example swaps a *whole instance* (a different `Clock`). Different return or throw values per argument is `Mockito.when(...)` in `@BeforeEach`. If the per-test mock has `when(...)` calls keyed on arguments, it is the wrong tool. |
| "The `@QuarkusTest` just verifies the wiring; the real rules get pure-JUnit tests later." | "Later" codifies the absence. A `@QuarkusTest` named `XTest` that mocks `X` is a tautology, and the rules in `X` stay untested until someone notices in production. If the rules deserve unit tests, write them now, on the plain JVM, as the next list item. |

---

Absorbed from the earlier standalone `quarkus-testing` skill
(jeremyrdavis/quarkus-skill-testing), updated to the current package layout
and the `tdd-*` loop.
